# Exploring New JSON Capabilities in SQL Server 2025

SQL Server 2025 introduces several powerful enhancements to its native JSON support, addressing long-standing gaps in performance, usability, and standards compliance. These features make JSON a first-class citizen alongside traditional relational data types.

This hands-on lab walks you through the new JSON features introduced in SQL Server 2025:

* Native `json` data type
* In-place updates using the `.modify()` method
* Native JSON indexes `CREATE JSON INDEX`
* JSON path expression array enhancements
* JSON aggregates for building JSON arrays and objects across rows
* The new `JSON_CONTAINS` function for containment checks

## JSON Formatting in SSMS 22

At long last, SQL Server Management Studio (SSMS) 22 allows you to view JSON data in a formatted, human-readable way. When you run a query that returns JSON data, SSMS renders the JSON as a hyperlink. Clicking the hyperlink opens a new tab within SSMS that displays the JSON in a nicely formatted expandable and collapsible structure, making it much easier to read and understand JSON results.

You'll get to experience this improved JSON formatting in SSMS 22 throughout the labs in this module. This enhancement significantly improves the developer experience when working with JSON data in SQL Server, without needing to copy and paste into external tools for formatting.

___

▶ [Lab: Native JSON Data Type](https://github.com/lennilobel/sql2025-workshop-hol-redmond2026/blob/main/HOL/2.%20JSON%20Support/1.%20Native%20JSON%20Data%20Type.md)
