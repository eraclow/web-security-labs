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
