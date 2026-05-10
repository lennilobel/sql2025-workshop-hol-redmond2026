# T-SQL Enhancements

In this section, you will explore the most exciting new T-SQL features introduced in SQL Server 2025. These enhancements expand the language in powerful ways—adding support for Unicode string handling, fuzzy and pattern-based matching, and a brand-new aggregate function.

Labs in this section cover the following T-SQL enhancements:

* **UNISTR**: Returns a Unicode string constructed from a string of Unicode code points.

* **ANSI String Concatenation (`||`)**: Introduces the ANSI standard operator for string concatenation, which offers consistent behavior with automatic type conversion and NULL handling.

* **Base64 Encoding and Decoding**: Adds `BASE64_ENCODE` and `BASE64_DECODE` functions for encoding binary data to Base64 strings and decoding Base64 strings back to binary data, including support for URL-safe Base64 encoding.

* **PRODUCT**: A new aggregate function that returns the product of all values (or distinct values) in a group or window of rows.

* **Fuzzy Matching Functions**: Includes `EDIT_DISTANCE`, `EDIT_DISTANCE_SIMILARITY`, `JARO_WINKLER_DISTANCE`, and `JARO_WINKLER_SIMILARITY` for comparing string similarity and supporting string matching scenarios.

* **Regular Expression Functions**: Adds a full suite of `REGEXP_...` functions including `REGEXP_LIKE`, `REGEXP_INSTR`, `REGEXP_SUBSTR`, `REGEXP_REPLACE`, `REGEXP_COUNT`, `REGEXP_SPLIT_TO_TABLE`, and `REGEXP_MATCHES`—bringing powerful pattern-matching capabilities to T-SQL.

Time to jump in!

___

▶ [Lab: UNISTR](https://github.com/lennilobel/sql2025-workshop-hol-orlando2025/blob/main/HOL/1.%20T-SQL%20Enhancements/1.%20UNISTR.md)
