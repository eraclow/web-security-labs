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

### Logic

 - We need to break out of the string context

 - Add a condition that always evaluates to TRUE

 - Comment out the rest of the query

This how We bypass the released=1 filter and reveal the hidden data.

### Example request

The final injected request looked like:

?category=’ OR 1=1 --
