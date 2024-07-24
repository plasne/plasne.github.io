---
layout: post
title: Managed Identity in Azure SQL DB
---

This guide will share the basic information needed to use a Managed Identity for Azure SQL DB.

## Code

Below is some sample code to create a connection using a token derived from DefaultAzureCredential. This allows you to support not only Managed Identity but also Azure CLI, Visual Studio, and other authentication methods.

```csharp
var context = new Azure.Core.TokenRequestContext(["https://database.windows.net/.default"]);
var connection = new SqlConnection(config.SQL_SERVER_HISTORY_SERVICE_CONNSTRING)
{
    AccessToken = this.defaultAzureCredential.GetToken(context).Token
};
```

## Connection String

The connection string will look like this...

```text
Server=tcp:my-sqlserver.database.windows.net,1433;Initial Catalog=my-db;Persist Security Info=False;Encrypt=True;TrustServerCertificate=False;
```

## Permissions

Adding the account to Azure SQL DB will be something like this...

```sql
-- create the user
CREATE USER [the-name-of-managed-identity] FROM EXTERNAL PROVIDER;

-- verify the user was created and has the right Client ID
SELECT name, type, type_desc, CAST(CAST(sid as varbinary(16)) as uniqueidentifier) as ClientId
FROM sys.database_principals
WHERE name = 'the-name-of-managed-identity'
GO;

-- assign permissions
ALTER ROLE db_ddladmin ADD MEMBER [the-name-of-managed-identity];
ALTER ROLE db_datareader ADD MEMBER [the-name-of-managed-identity];
ALTER ROLE db_datawriter ADD MEMBER [the-name-of-managed-identity];
```
