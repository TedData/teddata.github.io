---
layout: post
title: "How to Create a SQL Server Login and User with Schema-Level Permissions"
subtitle: 'Grant Schema Permissions in SQL'
date: 2025-06-23
author: "Ted"
header-style: text
tags:
  - TSQL
  - SSMS
---

# How to Create a SQL Server Login and User with Schema-Level Permissions

When setting up access for applications or users in SQL Server, a common requirement is to create a login at the server level and a user at the database level — then grant the appropriate permissions to that user. In this post, I will walk through a simple example that does exactly that: creating a login named `BoostJuice`, mapping it to a user in the `[bi]` database, and granting it full permissions on the `fact` schema.

## Step 1: Create a Login at the Server Level

The first step is to create a SQL Server login. A login is a server-level security principal that allows the user to connect to the SQL Server instance.

You can execute the following command in the `master` database:

```sql
-- Execute in the master database
CREATE LOGIN [BoostJuice] WITH PASSWORD = N'ILuvBIT!';

ALTER LOGIN [BoostJuice] ENABLE;
```

This creates a SQL authenticated login named `BoostJuice` with the password `ILuvBIT!`. The `ALTER LOGIN` command ensures that the login is enabled.

---

## Step 2: Create a User in the Target Database

Once the login is created, you need to create a corresponding user in the database where you want to grant access. In this case, it’s the `[bi]` database.

```sql
-- Execute in the [bi] database
CREATE USER [BoostJuice] FOR LOGIN [BoostJuice];
```

This command maps the server login `BoostJuice` to a user inside the `[bi]` database.

---

## Step 3: Grant Permissions on the Schema

Finally, you grant the required permissions to the user. In this example, we want the user to have full read and write access — `SELECT`, `INSERT`, `UPDATE`, and `DELETE` — on all objects within the `fact` schema:

```sql
-- Grant permissions on the [fact] schema
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::fact TO [BoostJuice];
```

If your database uses multiple schemas, you would need to write separate `GRANT` statements for each schema where access is needed.

---

## Summary

In just a few simple steps, we have:

1. Created a SQL Server login
2. Created a corresponding database user
3. Granted schema-level permissions

This is a useful pattern for setting up access for an application, a reporting tool, or an external partner — providing them with controlled access to specific parts of your database.
