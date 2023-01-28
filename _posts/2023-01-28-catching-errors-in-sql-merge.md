---
layout: post
title: Catching Errors in SQL MERGE
---

## Why use MERGE?

Using the MERGE command with a User-Defined Table Type can greatly increase performance. On a C# project I was working on where we needed to create/edit/delete up to 10,000 rows of data we say the following results:

- METHOD 1: Create a transaction, do INSERT, UPDATE, or DELETE command, loop until done, commit transaction. This took 10 minutes.

- METHOD 2: Put all rows into a User-Defined Table Type, do MERGE command. This took 2 seconds.

That is a 300x increase in performance!

## Catching Errors

Is there a downside to using the MERGE method? Well, it does make catching errors more complex. But first we need to separate the types of errors into 3 buckets: (a) datatype constraints and (b) errors related to a single row, and (c) errors that relate to the entire MERGE statement.

## DataType Constaints

Imagine our table looks like this:

```sql
CREATE TABLE [dbo].[colors] (
    [id]    int IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [color] varchar(20)
)
```

Then imagine our user-defined table looks like this:

```sql
CREATE TYPE [dbo].[batch] AS TABLE
(
    [action]    varchar(10) NOT NULL,
    [ref]       varchar(255) NOT NULL,
    [id]        int,
    [color]     varchar(20)
```

Imagine our input data looks like this:

```text
CREATE,ext-row-A,-1,red
EDIT,ext-row-B,10,green-is-not-a-long-enough-name-for-a-color
```

In C#, the code to put this into our parameter, might look something like this:

```csharp
using var inputTable = new DataTable();
inputTable.Columns.Add(new DataColumn("action", typeof(string)));
inputTable.Columns.Add(new DataColumn("ref", typeof(string)));
inputTable.Columns.Add(new DataColumn("id", typeof(int)));
inputTable.Columns.Add(new DataColumn("color", typeof(string)));

foreach (var dataRow in data)
{
    DataRow row = inputTable.NewRow();
    row["action"] = dataRow.Action;
    row["ref"] = dataRow.ReferenceId;
    row["id"] = dataRow.Id;
    row["color"] = dataRow.Color;
    inputTable.Rows.Add(row);
}

command.Parameters.Add(new SqlParameter("input", SqlDbType.Structured)
{
    TypeName = $"[dbo].[batch]",
    Value = inputTable,
});
```

For completeness, imagine our MERGE statement looks like this:

```sql
MERGE INTO [dbo].[colors] AS C
    USING @input AS I
    ON C.[id] = I.[id]
WHEN MATCHED AND I.[action] = 'EDIT' THEN
    UPDATE SET C.[color] = I.[color]
WHEN MATCHED AND I.[action] = 'DELETE' THEN
    DELETE
WHEN NOT MATCHED AND I.[action] = 'CREATE' THEN
    INSERT ([color]) VALUES (I.[color])
OUTPUT I.[ref], I.[action] AS requested, $action AS taken, INSERTED.[Id];
```

Now we can see that this input data is going to have a problem because the color is too long to fit in the "colors" table. But because it is also too big to fit in our "batch" table type we never even get to the MERGE statement. We fail as soon as the data gets to the SQL Server because there is no way for it to map the DataTable to the User-Defined Table Type and not violate the constraints.

The error message from SQL does not contain any helpful information about which row was a problem. To discern which row had a problem on the SQL Server would be complex, but here is an approach that might work (not fully tested):

1. Serialize all the input data into a JSON string.
1. Create your User-Defined Table Type with a `VARCHAR(MAX)` field to hold your JSON data.
1. In your MERGE statement you will use `JSON_VALUE` to extract individual fields.

This approach should move the problem from your input table to the MERGE statement.

IMPORTANT! I do not recommend this approach. Rather than moving the problem into SQL and the MERGE statement, it is trivial to write some validation logic in C# to ensure the data is of the right type, not null, and within the size constaints. This should be the preferred way to handle this problem.

## Errors Related To a Single Row

This would be an error raised because applying an INSERT, UPDATE, or DELETE with a particular set of data throws an exception. If you do validation on the input records as suggested above then there are 2 things I can think of that would cause this kind of error:

- A constraint violation.
- A trigger.

If your application controls the data store then these errors can also be handled by C# validation and I would recommend doing that.

The MERGE statement completes or errors as a whole. If you are taking the approach of catching things in C# validation, then I would recommend leaving that behavior intact. When an unexpected error condition is found, then modify your validation logic to cover that case.

However, should you want to proceed to catching row-level problems and raising those to users, here is an approach that works (fully tested):

Wrap your MERGE statement in a SQL TRY...CATCH:

```sql
BEGIN TRY
    MERGE...
    OUTPUT I.[ref], I.[action] AS requested, $action AS taken, INSERTED.[Id];
END TRY
BEGIN CATCH
    SELECT ERROR_MESSAGE()
END CATCH
```

Running this SQL statement will not longer throw an error on a failed row. Instead, it will complete as successful and you will have 2 result sets:

- The first result set contains all the OUTPUT data of the rows that contained valid data up to the row right before the row that failed. These are in order.

- The second result set contains the error message for the row that had bad data.

Then you can simply run the batch again starting with the next record beyond the bad record.

Here are some thoughts on how this should be implemented:

- You could wrap the MERGE inside a transaction. That way once a batch completes successfully, if any of the batches failed, you can roll back the entire set of batches.

- If you can stay on the SQL Server instead of coming back to C#, you will get better performance. To that end, you could wrap the whole TRY..MERGE..CATCH block inside a WHILE loop that iterated until all batches were completed. The error messages could be collected into a table variable that is flushed out at the end.

- If you expected a lot of errors, you could break the batch into smaller batches that you ran in parallel. You would wrap the whole thing in a transaction so that you could fail the entire set.

## Errors Related to the Entire MERGE Statement

If you have this type of error, you need to notify a developer (coding issue) or administrator (infrastructure issue) to fix it. This is not a condition that a user could be expected to fix by changing the input data.
