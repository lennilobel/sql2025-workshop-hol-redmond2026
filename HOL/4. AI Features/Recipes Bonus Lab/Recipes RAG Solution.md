# Recipes RAG Solution  

This lab implements a primitive **Retrieval-Augmented Generation (RAG)** pattern. RAG is a technique that enhances AI-generated responses by retrieving relevant documents from a knowledge base before passing them into a language model. Instead of relying solely on the model’s pre-trained knowledge, RAG dynamically incorporates external data, allowing it to provide more contextually relevant and accurate responses. A full RAG solution doesn't just perform vector search—it takes the raw vector search results and transforms them into a natural language response using a pre-trained **chat completions model**. This means we are moving beyond merely retrieving semantically similar records, and into generating human-friendly answers based on the retrieved data.

The previous lab used a tiny dataset with a few movie titles in a single column of a single table. That simple data model was adequate for learning how to vectorize data and run vector searches for semantic similarity. This lab uses a **Recipes** database, which we will populate with 50 recipes. This is a much richer dataset than the previous lab, and has multiple related tables. Each recipe includes structured details such as ingredients, preparation steps, and metadata, which will be vectorized across all columns and related tables—**not just the recipe name**. This approach mirrors real-world RAG implementations, where entire knowledge bases (not just document titles) are transformed into vectors for more comprehensive retrieval.  

But again, we won't be stopping at vector search. Like before, we will use the `VectorizeText` stored procedure to generate vector embeddings for both recipes and user queries using an Azure OpenAI **text embedding model**. However, after retrieving the most relevant recipes using vector search, we will take the process a step further by leveraging an Azure OpenAI **chat completions model** to format the raw database results into a natural language response. This mimics how AI-powered assistants retrieve and present data in an interactive, human-friendly way. By the end of this lab, you'll have a working example of a simple RAG workflow that retrieves relevant structured data and presents it in a conversational format.  

### Prerequisites  

This lab is based on the previous lab. Before proceeding, you must have created your personal **MyLabs** database and created the `VectorizeText` stored procedure as explained in the previous lab.

Also, ensure you are still connected to your personal **MyLabs** database on `vslive-sql.database.windows.net` from the previous lab.

## Create the Tables  

First, let's define the database schema for storing recipes. The schema consists of multiple related tables that capture different aspects of a recipe, such as ingredients, preparation steps, meal types, and tags. At this stage, the database functions like any traditional relational database—it does not yet include any AI capabilities.  

Each recipe is represented as a single row in the `Recipe` table, which contains high-level details such as the recipe name, difficulty level, cuisine type, and nutritional information. However, recipes often consist of multiple components, so we also create related tables to break down different aspects of a recipe into structured data:  

- `Ingredient`: Stores the list of ingredients for each recipe. Each recipe can have multiple ingredients, and each ingredient has an index that determines its order within the recipe.  
- `Instruction`: Stores the step-by-step cooking instructions for each recipe. Each step is indexed to maintain its sequence.  
- `MealType`: Categorizes the recipe based on the type of meal it belongs to (e.g., breakfast, lunch, dinner). A recipe may be associated with multiple meal types.  
- `Tag`: Contains additional descriptive tags (such as "gluten-free," "spicy," or "quick meal") that help categorize and filter recipes.  

Each of these related tables has a foreign key reference to the `Recipe` table, resulting in a basic normalized relational data model.

<!--
```sql
DROP TABLE IF EXISTS Tag
DROP TABLE IF EXISTS MealType
DROP TABLE IF EXISTS Instruction
DROP TABLE IF EXISTS Ingredient
DROP TABLE IF EXISTS Recipe

IF EXISTS (SELECT * FROM sys.database_scoped_credentials WHERE name = 'BlobStorageCredential')
  DROP DATABASE SCOPED CREDENTIAL BlobStorageCredential

IF EXISTS (SELECT * FROM sys.external_data_sources WHERE name = 'BlobStorageContainer')
  DROP EXTERNAL DATA SOURCE BlobStorageContainer

IF EXISTS (SELECT * FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
  DROP MASTER KEY
```
-->

```sql
CREATE TABLE Recipe(
  RecipeId int NOT NULL,
  RecipeName varchar(50) NOT NULL,
  PrepTimeMinutes int,
  CookTimeMinutes int,
  Servings int,
  Difficulty varchar(50),
  Cuisine varchar(50),
  CaloriesPerService int,
  Rating decimal(18, 9),
  ReviewCount int,
  CONSTRAINT PK_Recipe PRIMARY KEY CLUSTERED (RecipeId),
)

CREATE TABLE Ingredient(
  IngredientId int NOT NULL IDENTITY,
  RecipeId int NOT NULL,
  IngredientIndex int NOT NULL,
  IngredientName varchar(50) NOT NULL,
  CONSTRAINT PK_Ingredient PRIMARY KEY CLUSTERED  (IngredientId),
  CONSTRAINT FK_Recipe_Ingredient FOREIGN KEY(RecipeId) REFERENCES Recipe (RecipeId),
)

CREATE TABLE Instruction(
  InstructionId int NOT NULL IDENTITY,
  RecipeId int NOT NULL,
  InstructionIndex int NOT NULL,
  InstructionText varchar(200) NOT NULL,
  CONSTRAINT PK_Instruction PRIMARY KEY CLUSTERED (InstructionId),
  CONSTRAINT FK_Recipe_Instruction FOREIGN KEY(RecipeId) REFERENCES Recipe (RecipeId),
)

CREATE TABLE MealType(
  MealTypeId int NOT NULL IDENTITY,
  RecipeId int NOT NULL,
  MealTypeName varchar(50) NOT NULL,
  CONSTRAINT PK_MealType PRIMARY KEY CLUSTERED (MealTypeId),
  CONSTRAINT FK_Recipe_MealType FOREIGN KEY(RecipeId) REFERENCES Recipe (RecipeId),
)

CREATE TABLE Tag(
  TagId int IDENTITY(1,1) NOT NULL,
  RecipeId int NOT NULL,
  TagName varchar(50) NOT NULL,
  CONSTRAINT PK_Tag PRIMARY KEY CLUSTERED (TagId),
  CONSTRAINT FK_Recipe_Tag FOREIGN KEY(RecipeId) REFERENCES Recipe (RecipeId),
)
```

## Populate the Tables  

Now we'll populate the database with 50 recipes from a raw JSON file stored in **Azure Blob Storage**. The JSON file contains structured recipe data, including ingredients, instructions, meal types, and tags. To access this file, you will be provided with a **shared access signature (SAS) token** for authentication.  

- **Note:** To examine the raw JSON source file, open the local copy of `recipes.json` provided in this lab's folder.

The first snippet of code below sets up the database credentials and external data source to connect to the Azure Blob Storage container. This allows us to retrieve the raw JSON file directly from cloud storage.  

### Define Credentials and Blob Storage Data Source

Paste the following T-SQL code into the query window, and replace the `SECRET` placeholder with the SAS token provided for the lab. Then execute the code to create the credentials and external data source needed to access the JSON file in Blob Storage:  

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ngP@$$w0rd'

CREATE DATABASE SCOPED CREDENTIAL BlobStorageCredential
  WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
  SECRET = '<provided by the instructor>'

CREATE EXTERNAL DATA SOURCE BlobStorageContainer
WITH (
  TYPE = BLOB_STORAGE,
  LOCATION = 'https://lennidemo.blob.core.windows.net/datasets',
  CREDENTIAL = BlobStorageCredential
)
```  

### Import the Raw JSON Source File 

Now we can retrieve the raw JSON file from Blob Storage and processes it into the relational tables. The script below uses `OPENROWSET` with the `BULK` provider to read the `Recipes.json` file and store it in a variable. Then, `OPENJSON` parses the JSON structure, and `JSON_VALUE` extracts individual property values from the JSON as columns for storing in the `Recipe` table.

Since each recipe contains multiple ingredients, instructions, meal types, and tags, these must also be stored in their respective related tables. So we also use `CROSS APPLY` with `OPENJSON` to extract and insert **multiple related rows per recipe** into each of the associated tables.  

```sql
-- Retrieve the raw JSON source file from blob storage
DECLARE @Json varchar(max) = (
  SELECT BulkColumn
  FROM OPENROWSET(BULK 'Recipes.json', DATA_SOURCE = 'BlobStorageContainer', SINGLE_CLOB) AS RecipesJson
)

-- Insert into the Recipe table
INSERT INTO Recipe (
  RecipeId,
  RecipeName,
  PrepTimeMinutes,
  CookTimeMinutes,
  Servings,
  Difficulty,
  Cuisine,
  CaloriesPerService,
  Rating,
  ReviewCount)
SELECT 
  JSON_VALUE(Recipe.value, '$.id') AS RecipeId,
  JSON_VALUE(Recipe.value, '$.name') AS RecipeName,
  JSON_VALUE(Recipe.value, '$.prepTimeMinutes') AS PrepTimeMinutes,
  JSON_VALUE(Recipe.value, '$.cookTimeMinutes') AS CookTimeMinutes,
  JSON_VALUE(Recipe.value, '$.servings') AS Servings,
  JSON_VALUE(Recipe.value, '$.difficulty') AS Difficulty,
  JSON_VALUE(Recipe.value, '$.cuisine') AS Cuisine,
  JSON_VALUE(Recipe.value, '$.caloriesPerServing') AS CaloriesPerService,
  JSON_VALUE(Recipe.value, '$.rating') AS Rating,
  JSON_VALUE(Recipe.value, '$.reviewCount') AS ReviewCount
FROM OPENJSON(@Json) AS Recipe

-- Insert into the Ingredient table
INSERT INTO Ingredient (RecipeId, IngredientIndex, IngredientName)
SELECT 
  JSON_VALUE(Recipe.value, '$.id') AS RecipeId,
  Ingredient.[key] AS IngredientIndex,
  Ingredient.value AS IngredientName
FROM OPENJSON(@Json) AS Recipe
CROSS APPLY OPENJSON(Recipe.value, '$.ingredients') AS Ingredient

-- Insert into the Instruction table
INSERT INTO Instruction (RecipeId, InstructionIndex, InstructionText)
SELECT 
  JSON_VALUE(Recipe.value, '$.id') AS RecipeId,
  Instruction.[key] AS InstructionIndex,
  Instruction.value AS InstructionText
FROM OPENJSON(@Json) AS Recipe
CROSS APPLY OPENJSON(Recipe.value, '$.instructions') AS Instruction

-- Insert into the MealType table
INSERT INTO MealType (RecipeId, MealTypeName)
SELECT 
  JSON_VALUE(Recipe.value, '$.id') AS RecipeId,
  MealType.value AS MealTypeName
FROM OPENJSON(@Json) AS Recipe
CROSS APPLY OPENJSON(Recipe.value, '$.mealType') AS MealType

-- Insert into the Tag table
INSERT INTO Tag (RecipeId, TagName)
SELECT 
  JSON_VALUE(Recipe.value, '$.id') AS RecipeId,
  Tag.value AS TagName
FROM OPENJSON(@Json) AS Recipe
CROSS APPLY OPENJSON(Recipe.value, '$.tags') AS Tag
```  


Now confirm that the tables are populated correctly:

```sql
SELECT * FROM Recipe
SELECT * FROM Ingredient
SELECT * FROM Instruction
SELECT * FROM MealType
SELECT * FROM Tag
```  

You have successfully imported 50 recipes into the database, with all related data stored in their respective tables. While the database is relatively simple for demo purposes, it does represent a typical real-world normalized relational data model.

## Create a View for Complete Recipe Entities  

To perform vectorization, we need to ensure that each recipe and all of its related data (ingredients, instructions, meal type, and tags) are combined into a **single text representation per recipe**. Since vector models process text input as a single unit, we will transform the structured relational data into a consolidated format where each recipe’s details appear in a single row for vectorization.  

This view accomplishes that by:  
- Using `STRING_AGG` to aggregate related rows (e.g., all ingredients, instructions, meal types, and tags) into a **single string** per recipe.  
- Using `ROW_NUMBER() OVER (ORDER BY)` to assign a sequence number to ingredients and instructions, ensuring they are concatenated in the correct sequence and numbered accordingly.  
- Maintaining **one row per recipe**, with all related data formatted as a structured text representation.  

Run the following T-SQL code to create the view:  

```sql
CREATE OR ALTER VIEW RecipeView AS
SELECT 
  r.RecipeId,
  r.RecipeName,
  r.PrepTimeMinutes,
  r.CookTimeMinutes,
  r.Servings,
  r.Difficulty,
  r.Cuisine,
  r.CaloriesPerService,
  r.Rating,
  r.ReviewCount,
  Ingredients = (
    SELECT STRING_AGG(CONCAT(i.RowNum, ') ', i.IngredientName), '. ')
    FROM (
      SELECT 
        IngredientName,
        ROW_NUMBER() OVER (ORDER BY IngredientId) AS RowNum
      FROM Ingredient
      WHERE RecipeId = r.RecipeId
    ) AS i
  ),
  Instructions = (
    SELECT STRING_AGG(CONCAT(i.RowNum, ') ', i.InstructionText), ' ')
    FROM (
      SELECT 
        InstructionText,
        ROW_NUMBER() OVER (ORDER BY InstructionId) AS RowNum
      FROM Instruction
      WHERE RecipeId = r.RecipeId
    ) AS i
  ),
  MealType = (
    SELECT STRING_AGG(mt.MealTypeName, '; ')
    FROM MealType AS mt
    WHERE mt.RecipeId = r.RecipeId
  ),
  Tags = (
    SELECT STRING_AGG(t.TagName, '; ')
    FROM Tag AS t
    WHERE t.RecipeId = r.RecipeId
  )
FROM Recipe AS r
```  

Now query the view for the first five recipes, and note how the related data is formatted into a **single row per recipe**:

```sql
SELECT TOP 5 * FROM RecipeView
```  

### What to Observe
- Each row represents **one complete recipe**, combining its ingredients, instructions, meal type, and tags into a structured text format.  
- **Ingredients and instructions** are concatenated and numbered in the correct order.  
- **Meal types and tags** are concatenated using a semicolon separator.  

## Create a Stored Procedure to Retrieve a Recipe as JSON  

To prepare recipes for vectorization, we need to retrieve each recipe and its related data as a single piece of text. Using a **structured JSON document** for this purpose is ideal because it:  
- Combines all recipe details, ingredients, instructions, meal types, and tags into a single text entity.  
- Serves as a clean input for the `VectorizeText` stored procedure, which will generate a semantic vector for each recipe.  
- Ensures that the entire context of a recipe is captured when embedding it for similarity search.  

To achieve this, we create a stored procedure that:  
1. Accepts a `@RecipeId` as an input parameter.  
2. Queries the `RecipeView` for the corresponding row of the complete recipe.  
3. Formats the row using `FOR JSON AUTO` to return a structured JSON object via the `@RecipeJson` output parameter.  

Run the following T-SQL to create or update the stored procedure:  

```sql
CREATE OR ALTER PROCEDURE GetRecipeJson
  @RecipeId int,
  @RecipeJson nvarchar(max) OUTPUT
AS
BEGIN

  SET @RecipeJson = (
    SELECT *
    FROM RecipeView
    WHERE RecipeId = @RecipeId
    FOR JSON AUTO, WITHOUT_ARRAY_WRAPPER
  )

END
```  

Test it out by retrieving JSON representation of recipe ID 1: 

```sql
DECLARE @RecipeJson nvarchar(max)
EXEC GetRecipeJson @RecipeId = 1, @RecipeJson = @RecipeJson OUTPUT
SELECT @RecipeJson
```  

### **What to Observe**  
- The output should be a **JSON object** containing the **complete recipe details**, including:  
  - Recipe name, difficulty, cuisine, and nutritional info.  
  - All ingredients in a **numbered list**.  
  - Step-by-step instructions in a **concatenated format**.  
  - Meal types and tags categorized properly.  
- This JSON will later be **vectorized** to generate embeddings for similarity search.  

This stored procedure will allow us to vectorize full recipe entities, rather than just names or short descriptions.  

## Vectorize the Recipes  

Now let's vectorize all the recipes in our database.

The first step is to modify the `Recipe` table by adding a `vector` column. This column will store **1536-dimensional vectors**, just like in the `Movie` table from the previous lab. And just as we did then, we will vectorize our data using the Azure OpenAI **text-embedding-3-large** model, capturing the semantic meaning of each complete recipe entity.  

### Add the Vector Column  

Run the following command to add the vector column:  

```sql
ALTER TABLE Recipe ADD Vector vector(1536)
```  

Now let's generate vectors for each recipe and store them in this new column.  

### Generate and Store Recipe Vectors

The following script processes each recipe in the database, fetches its JSON representation, generates a vector, and updates the `Recipe` table.  

This process works very similarly to the previous lab:
1. Uses a cursor to iterate over each `RecipeId` in the `Recipe` table.  
2. Calls `GetRecipeJson` to retrieve the complete JSON representation of each recipe.  
3. Calls `VectorizeText` to generate a 1536-dimensional vector for the JSON content.  
4. Updates the `Vector` column in the `Recipe` table with the generated vector.  

Run this script to vectorize all the recipes (allow a few seconds to process all 50 recipes):  

```sql
DECLARE @RecipeId int
DECLARE @RecipeJson varchar(max)
DECLARE @Vector vector(1536)

DECLARE curRecipes CURSOR FOR
  SELECT RecipeId FROM Recipe

OPEN curRecipes
FETCH NEXT FROM curRecipes INTO @RecipeId

WHILE @@FETCH_STATUS = 0
BEGIN
 -- Get the JSON content for the recipe
 EXEC GetRecipeJson @RecipeId, @RecipeJson OUTPUT

  -- Generate vector for the recipe
  EXEC VectorizeText @RecipeJson, @Vector OUTPUT

  -- Store vector in the Recipe table
  UPDATE Recipe
  SET Vector = @Vector
  WHERE RecipeId = @RecipeId

  FETCH NEXT FROM curRecipes INTO @RecipeId
END

CLOSE curRecipes
DEALLOCATE curRecipes
```  

When the vectorization process completes, run the following query to inspect the `Vector` column:  

```sql
SELECT * FROM Recipe
```  

### **What to Observe**  
- The `Vector` column should now be **populated with 1536-dimensional vectors**, each representing a complete recipe entity.  
- These vectors **capture the semantic meaning** of the entire recipe, including ingredients, instructions, meal type, and tags.  
- This step enables **semantic search**, allowing us to query the database based on **meaning**, rather than just exact text matches.  

Now that the recipes have been vectorized, we are ready to perform vector searches using natural language queries.

## Create a Stored Procedure to Run a Vector Search  

We need a way to search for recipes based on semantic similarity to natural language queries.  

This stored procedure functions very much like the one from the previous lab. But instead of returning only the single closest match with `TOP 1`, this procedure returns the `TOP 5` most similar recipes, sorted by **cosine distance** (from most similar to least).  

Run the following T-SQL to create the stored procedure:  

```sql
CREATE OR ALTER PROCEDURE RecipeVectorSearch
  @Question varchar(max)
AS
BEGIN
  -- Prepare a vector variable to capture the question vector components
  DECLARE @QuestionVector vector(1536)

  -- Vectorize the question using Azure OpenAI
  EXEC VectorizeText @Question, @QuestionVector OUTPUT

  -- Find the most similar recipes based on cosine similarity
  SELECT TOP 5
    rv.*,
    CosineDistance = VECTOR_DISTANCE('cosine', @QuestionVector, r.Vector)
  FROM
    Recipe AS r
    INNER JOIN RecipeView AS rv ON rv.RecipeId = r.RecipeId
  ORDER BY
    CosineDistance
END
```  

Now that we have our vector search procedure in place, let's test it with different natural language queries.  

Run the following queries, one at a time:  

```sql
EXEC RecipeVectorSearch 'How about some Italian appetizers?'
```  
```sql
EXEC RecipeVectorSearch 'Show me your pizza recipes'
```  
```sql
EXEC RecipeVectorSearch 'Pineapple in the ingredients'
```  
```sql
EXEC RecipeVectorSearch 'Please recommend delicious Italian desserts'
```  
```sql
EXEC RecipeVectorSearch 'Fresh Mediterranean salad options'
```  
```sql
EXEC RecipeVectorSearch 'I love chicken, kebab, and falafel'
```  
```sql
EXEC RecipeVectorSearch 'Got any Asian stir fry recipes?'
```  
```sql
EXEC RecipeVectorSearch 'Give me some soup choices'
```  
```sql
EXEC RecipeVectorSearch 'Traditional Korean breakfast'
```  

### What to Observe
- Each query returns **five recipes** that are most semantically similar to the input.  
- The **CosineDistance** column shows the similarity score, with **lower values** indicating a **closer match**.  
- The **vector model** understands **semantic relationships** rather than just exact keyword matches. For example, queries like `"Pineapple in the ingredients"` will also return recipes with mango in the ingredients as being only slightly less similar than pineapple, since they are both tropical fruits.
 
- The search handles **ingredient-based**, **cuisine-specific**, and **meal-type** queries effectively.  

This is an AI-powered recipe search that enables users to find recipes using natural language instead of rigid SQL queries.

## Create a JSON Version of the Vector Search Stored Procedure  

This stored procedure is almost identical to the `RecipeVectorSearch` procedure we just created, but with one key difference. Instead of returning tabular results, it returns the results as a raw JSON string representing an array of recipe objects.

### Why Do We Need a JSON Version?
- The chat completions model we will use in the next step requires a single piece of text, rather than separate tabular results.  
- This version packages all search results into a single JSON string, which can be passed as input to the Azure OpenAI chat model for processing into a natural language response.

Run the following T-SQL code to create the vector search stored procedure that returns a single JSON string for all thre search results:  

```sql
CREATE OR ALTER PROCEDURE RecipeVectorSearchJson
  @Question varchar(max)
AS
BEGIN
  -- Prepare a vector variable to capture the question vector components
  DECLARE @QuestionVector vector(1536)

  -- Vectorize the question using Azure OpenAI
  EXEC VectorizeText @Question, @QuestionVector OUTPUT

  -- Find the most similar recipes based on cosine similarity
  DECLARE @JsonResults nvarchar(max) = (
    SELECT TOP 10
      rv.*,
      CosineDistance = VECTOR_DISTANCE('cosine', @QuestionVector, r.Vector)  -- New T-SQL function
    FROM
      Recipe AS r
      INNER JOIN RecipeView AS rv ON rv.RecipeId = r.RecipeId
    WHERE
      VECTOR_DISTANCE('cosine', @QuestionVector, r.Vector) < .65
    ORDER BY CosineDistance
    FOR JSON AUTO
  )

  SELECT JsonResults = @JsonResults
END
```  

Now run the following queries using the same **question text** to compare the outputs:  

```sql
EXEC RecipeVectorSearch 'I love chicken, kebab, and falafel'
EXEC RecipeVectorSearchJson 'I love chicken, kebab, and falafel'
```  

### **What to Observe**  
- The first stored procedure (`RecipeVectorSearch`) **returns tabular results**, where each recipe appears as a **row**.  
- The second stored procedure (`RecipeVectorSearchJson`) **returns a single JSON string**, which holds an **array of objects**, each representing a recipe.  
- The JSON structure **matches the tabular output**, but in a **string format** that can be **sent to the chat model for further processing**.  

## Create a Stored Procedure for Chat Completions  

Now that we have vectorized our recipes and can retrieve the most relevant matches, the next step is to generate a natural language response to a natural language query using Azure OpenAI’s **GPT-4o** model.  

This `AskQuestion` stored procedure is very similar to the `VectorizeText` stored procedure from the previous lab, since they are both calling Azure OpenAI. The key differences in this stored procedure are:  
1. Uses the GPT-4o chat completions model instead of the text embedding model.  
2. Targets the `/chat/completions` API endpoint rather than `/embeddings`.  
3. Includes system and user messages in the API payload, shaping the assistant’s demeanor, behavior, and response style.  
4. Allows configuration of response settings such as `temperature`, `max_tokens`, and `penalties`, which affect the creativity and verbosity of the response.  

This stored procedure will allow us to pass in the raw JSON search results and transform them into a conversational AI response.  

### Create the Chat Completions Stored Procedure

Run the following T-SQL code to create the stored procedure:  

```sql
CREATE OR ALTER PROCEDURE AskQuestion
  @Question varchar(max),
  @ResponseText varchar(max) OUTPUT
AS
BEGIN
  -- Azure OpenAI endpoint
  DECLARE @OpenAIEndpoint varchar(max) = 'https://lenni-m6wi7gcd-eastus2.cognitiveservices.azure.com/'
  DECLARE @OpenAIDeploymentName varchar(max) = 'gpt-4o' -- Use the appropriate GPT model deployment name
  DECLARE @OpenAIApiVersionVersion varchar(max) = '2023-05-15'
  DECLARE @Url varchar(max) = CONCAT(@OpenAIEndpoint, 'openai/deployments/', @OpenAIDeploymentName, '/chat/completions?api-version=', @OpenAIApiVersionVersion)

  -- Azure OpenAI API key
  DECLARE @OpenAIApiKey varchar(max) = '<provided by the instructor>'
  DECLARE @Headers varchar(max) = JSON_OBJECT('api-key': @OpenAIApiKey)

  -- Payload for chat completions request
  DECLARE @Payload varchar(max) = JSON_OBJECT(
    'messages': JSON_ARRAY(
      JSON_OBJECT('role': 'system', 'content': 'You are an assistant that helps people find recipes from a database.'),
      JSON_OBJECT('role': 'user', 'content': @Question)
    ),
    'max_tokens': 1000,        -- Max number of tokens for the response; the more tokens you specify (spend), the more verbose the response
    'temperature': 1.0,        -- Range is 0.0 to 2.0; controls "apparent creativity"; higher = more random, lower = more deterministic
    'frequency_penalty': 0.0,  -- Range is -2.0 to 2.0; controls likelihood of repeating words; higher = less likely, lower = more likely
    'presence_penalty': 0.0,   -- Range is -2.0 to 2.0; controls likelihood of introducing new topics; higher = more likely, lower = less likely
    'top_p': 0.95              -- Range is 0.0 to 2.0; aka "Top P sampling"; temperature alternative; controls diversity of responses (1.0 is full random, lower values limit randomness)
  )

  -- Response and return value
  DECLARE @Response nvarchar(max)
  DECLARE @ReturnValue int

  -- Call Azure OpenAI to get chat response
  EXEC @ReturnValue = sp_invoke_external_rest_endpoint
    @url = @Url,
    @method = 'POST',
    @headers = @Headers,
    @payload = @Payload,
    @response = @Response OUTPUT

  -- Handle API errors
  IF @ReturnValue != 0
    THROW 50000, @Response, 1

  -- Extract the assistant's reply from the JSON response
  SET @ResponseText = JSON_VALUE(@Response, '$.result.choices[0].message.content')
END
```  

### Understanding the API Call

Notice the many similarities this stored procedure has with the `VectorizeText` stored procedure. Both call Azure OpenAI, but this one utilizes a chat completions model (to return a natural language response) rather than a text embedding model (which returns a vector).

Specificaly, note the following:

- **System Message:**  
  - `"You are an assistant that helps people find recipes from a database."`  
  - This sets the assistant’s role, instructing it to behave like a recipe assistant.  
- **User Message:**  
  - The input question (such as JSON results of a recipe vector search) is passed into the API.  
- **Response Settings:**  
  - `max_tokens`: Limits the length (and cost) of the response.  
  - `temperature`: Controls creativity (higher = more random, lower = more deterministic).  
  - `frequency_penalty`: Adjusts word repetition tendencies.  
  - `presence_penalty`: Encourages or discourages new topics.  
  - `top_p`: Alternative to temperature for controlling diversity.  

### Test the Stored Procedure

Now, let's run the stored procedure with a sample question:  

```sql
DECLARE @ResponseText varchar(max)
EXEC AskQuestion 'What is the capital of France?' , @ResponseText OUTPUT
SELECT @ResponseText
```  

### **What to Observe**  
- It takes noticeably longer to get an Azure OpenAI response from the chat completions model than it does from the text embedding model. Expect to wait between 5 and 15 seconds for each API call.
- The assistant should generate a human-like response. Based on the system message we provided, it explains that its purpose is to help recommend recipes. However, it still knows that Paris is the capital of France.  
- The model does not search the database itself—it only responds based on the system prompt and input question.  
- In the next step, we will leverage this cognitive ability by passing recipes from actual vector search results, allowing the assistant to return a contextually relevant response to the user based on the database search results.  

## Create the AI Assistant Stored Procedure  

This is the final step in building a chat-based AI assistant for finding recipes in the database.

The `AskRecipeQuestion` stored procedure ties everything together by:  
1. Accepting a natural language question as input.  
2. Calling `RecipeVectorSearchJson` (the JSON version of the vector search stored procedure) to retrieve the most relevant recipes as a JSON string.  
3. Constructing a user prompt that helps the GPT model generate a natural language response instead of raw search results.  
4. Calling `AskQuestion` which calls the chat completions model to format and present the results conversationally.  

### How the Prompt Works
The GPT model is instructed to:  
- Start the response with a friendly, human-like sentence related to the user’s question.  
- Format the retrieved recipes in a structured way, listing:  
   - Ingredients as a comma-separated list.  
   - Instructions as a numbered comma-separated list.  
- Use the retrieved JSON data as context for its response.  

Run the following T-SQL to create the stored procedure:  

```sql
CREATE OR ALTER PROCEDURE AskRecipeQuestion @Question nvarchar(max)
AS
BEGIN

  SET NOCOUNT ON

  -- Capture the vector search results as JSON
  DECLARE @JsonResultsTable table (JsonResults nvarchar(max))
  INSERT INTO @JsonResultsTable EXEC RecipeVectorSearchJson @Question
  DECLARE @JsonResults nvarchar(max) = (SELECT JsonResults FROM @JsonResultsTable)

  -- Construct the user prompt
  DECLARE @UserPrompt nvarchar(max) = CONCAT('
    The user asked the question "', @Question, '" and the database returned the recipes below.
    Generate a response that starts with a sentence or two related to the user''s question,
    followed by each recipe''s details. For the details, list the ingredients as a comma-separated
    string, and list the instructions as a numbered list:
    ', @JsonResults
  )

  -- Call the GPT model to generate the response
  DECLARE @ResponseText nvarchar(max)
  EXEC AskQuestion @UserPrompt, @ResponseText OUTPUT
  PRINT @ResponseText

END
```  

### Try It Out!

Run the following queries *one at a time* to interact with the AI-powered recipe assistant:  

```sql
EXEC AskRecipeQuestion 'How about some Italian appetizers?'
```  
```sql
EXEC AskRecipeQuestion 'Show me your pizza recipes'
```  
```sql
EXEC AskRecipeQuestion 'Please recommend delicious Italian desserts'
```  
```sql
EXEC AskRecipeQuestion 'Fresh Mediterranean salad options'
```  
```sql
EXEC AskRecipeQuestion 'I love chicken, kebab, and falafel'
```  

### **What to Observe**  
- The database understands the user’s query and retrieves the most relevant recipes.  
- It formats the response conversationally, rather than just returning raw JSON.  
- It structures the ingredients and instructions neatly, making it easy to read.  
- The assistant can handle a variety of queries, from cuisine types to specific ingredients.  

## Congratulations!
You have successfully built a **Retrieval-Augmented Generation (RAG) AI assistant** inside **Azure SQL Database**! Remember, everything we just implemented is still in *public preview*, and is therefore subject to change by the time these features are released for general availability. And everything will work the same on-premises with **SQL Server 2025** when it is released later this year.

This approach can be expanded to other domains (legal, medical, finance, etc.), enabling intelligent search experiences over any structured data you store in SQL Server.  
___

▶ [Lab: Vector Utility Functions](https://github.com/lennilobel/sql2025-workshop-hol-orlando2025/blob/main/HOL/4.%20AI%20Features/3.%20Vector%20Utility%20Functions.md)
