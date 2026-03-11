# ali-wherescape - Template Language Reference

This reference provides detailed guidance on WhereScape's template scripting language for custom code generation.

---

## Template Structure

WhereScape templates use VBScript to generate SQL dynamically based on metadata.

```vbscript
' Template: Custom Stage Table Template
' Purpose: Generate SCD Type 2 MERGE with custom hash logic

Dim objTable, objColumn, strSQL

' Get table metadata
Set objTable = Context.Object

' Start SQL generation
strSQL = "MERGE INTO " & objTable.FullName & " tgt" & vbCrLf
strSQL = strSQL & "USING (" & vbCrLf

' Add source columns
For Each objColumn In objTable.Columns
    If objColumn.SourceColumn <> "" Then
        strSQL = strSQL & "    " & objColumn.SourceColumn & " AS " & objColumn.Name & "," & vbCrLf
    End If
Next

' Generate hash key
strSQL = strSQL & "    MD5("
For Each objColumn In objTable.Columns
    If objColumn.BusinessKey Then
        strSQL = strSQL & objColumn.Name & " || '|' || "
    End If
Next
strSQL = Left(strSQL, Len(strSQL) - 8) & ") AS hash_key" & vbCrLf
strSQL = strSQL & "FROM " & objTable.SourceTable & vbCrLf
strSQL = strSQL & ") src" & vbCrLf

' ON clause
strSQL = strSQL & "ON tgt.hash_key = src.hash_key AND tgt.is_current = TRUE" & vbCrLf

' WHEN MATCHED
strSQL = strSQL & "WHEN MATCHED AND tgt.hash_diff <> src.hash_diff THEN" & vbCrLf
strSQL = strSQL & "    UPDATE SET tgt.valid_to = CURRENT_DATE - 1, tgt.is_current = FALSE" & vbCrLf

' WHEN NOT MATCHED
strSQL = strSQL & "WHEN NOT MATCHED THEN" & vbCrLf
strSQL = strSQL & "    INSERT (" & vbCrLf
For Each objColumn In objTable.Columns
    strSQL = strSQL & "        " & objColumn.Name & "," & vbCrLf
Next
strSQL = Left(strSQL, Len(strSQL) - 3) & vbCrLf
strSQL = strSQL & "    ) VALUES (" & vbCrLf
For Each objColumn In objTable.Columns
    strSQL = strSQL & "        src." & objColumn.Name & "," & vbCrLf
Next
strSQL = Left(strSQL, Len(strSQL) - 3) & vbCrLf
strSQL = strSQL & "    );" & vbCrLf

' Return generated SQL
Template.SQL = strSQL
```

---

## Common Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| Context.Object | Current table/object | objTable |
| Context.Object.FullName | Fully qualified name | SCHEMA.TABLE_NAME |
| Context.Object.Columns | Column collection | For Each objColumn In ... |
| Column.Name | Column name | customer_id |
| Column.SourceColumn | Source column mapping | src.cust_id |
| Column.DataType | Data type | VARCHAR(50) |
| Column.BusinessKey | Is business key? | TRUE/FALSE |
| Template.SQL | Generated SQL output | strSQL |

---

## Template Best Practices

| Practice | Why | Example |
|----------|-----|---------|
| Use metadata loops | Avoid hardcoding column names | For Each objColumn In objTable.Columns |
| Parameterize table names | Support reuse across schemas | objTable.FullName |
| Add comments | Generated SQL is hard to debug | strSQL = strSQL & "-- Stage Customer Data" |
| Version templates | Track changes over time | Template name includes version: "Stage_SCD2_v2" |
| Test with small objects | Large objects generate huge SQL | Test on 3-5 column table first |

---

## Template Variable Cheat Sheet

```vbscript
' Table metadata
objTable.Name                    ' Table name
objTable.FullName                ' Schema.Table
objTable.SourceTable             ' Source table name
objTable.Columns                 ' Column collection

' Column metadata
objColumn.Name                   ' Column name
objColumn.SourceColumn           ' Source column mapping
objColumn.DataType               ' Data type
objColumn.BusinessKey            ' Is business key?
objColumn.IsPrimaryKey           ' Is primary key?

' Context
Context.Object                   ' Current object
Context.Environment              ' Environment (DEV, PROD)
Context.User                     ' Current user
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
