---
layout: post
title: Soft Delete in SQL Server MERGE
---

Recently I needed to handle create, edit, and delete via a SQL MERGE but the delete was a soft delete. While this is not a problem in some SQL implementations, SQL Server only allows for conditional match statements that support a single UPDATE clause.

For instance, this will fail:

```sql
MERGE INTO [dbo].[example] AS E
    USING @input AS I
    ON E.[id] = I.[id]
WHEN MATCHED AND I.[action] = 'DELETE' THEN
    UPDATE SET [isDeleted] = 1
WHEN MATCHED AND I.[action] = 'EDIT' THEN
    UPDATE SET [color] = I.[color]
WHEN NOT MATCHED AND I.[action] = 'CREATE' THEN
    INSERT ([color]) VALUES (I.[color])
OUTPUT I.[action] AS requested, $action AS taken, INSERTED.[Id];
```

Instead, we have to create some conditional statements like this:

```sql
MERGE INTO [dbo].[example] AS E
    USING @input AS I
    ON E.[id] = I.[id]
WHEN MATCHED AND I.[action] = 'DELETE' OR I.[action] = 'EDIT' THEN
    UPDATE SET
        [color] = CASE WHEN I.[action] = 'EDIT' THEN I.[color] ELSE E.[color] END
        [isDeleted] = CASE WHEN I.[action] = 'DELETE' THEN 1 ELSE E.[isDeleted] END
WHEN NOT MATCHED AND I.[action] = 'CREATE' THEN
    INSERT ([color]) VALUES (I.[color])
OUTPUT I.[action] AS requested, $action AS taken, INSERTED.[Id];
```

Unfortunately you need a CASE statement per field, but this works perfectly.
