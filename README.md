# SQLi_cheatsheet

## SQLi examples

### Retrieving Hidden Data

<p>Consider a shopping application that displays products in different categories. When the user clicks on the Gifts category, their browser requests the URL:</p>

````url
https://insecure-website.com/products?category=Gifts
````

<p>This causes the application to make a SQL query to retrieve details of the relevant products from the database:</p>

````sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
````
<p>This SQL query asks the database to return:</p>

- all details (*)
- from the products table
- where the category is Gifts
- and released is 1.

<p> The restriction released = 1 is being used to hide products that are not released. For unreleased products, presumably released = 0.</p>

<p>The application doesn't implement any defenses against SQL injection attacks, so an attacker can construct an attack like:</p>

````url
https://insecure-website.com/products?category=Gifts'--
````

<p>This results in the SQL query:</p>

````sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
````

<p>The key thing here is that the double-dash sequence -- is a comment indicator in SQL, and means that the rest of the query is interpreted as a comment. This effectively removes the remainder of the query, so it no longer includes AND released = 1. This means that all products are displayed, including unreleased products.</p>

<p>Going further, an attacker can cause the application to display all the products in any category, including categories that they don't know about:
</p>

````url
https://insecure-website.com/products?category=Gifts'+OR+1=1--
````

<p>This results in the SQL Query:</p>

````sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
````

### Subverting application logic

<p>Consider an application that lets users log in with a username and password. If a user submits the username wiener and the password bluecheese, the application checks the credentials by performing the following SQL query:</p>

````sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
````

<p>If the query returns the details of a user, then the login is successful. Otherwise, it is rejected.

Here, an attacker can log in as any user without a password simply by using the SQL comment sequence -- to remove the password check from the WHERE clause of the query. For example, submitting the username administrator'-- and a blank password results in the following query:</p>

````sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
````


### Retrieving data from other database tables

<p>In cases where the results of a SQL query are returned within the application's responses, an attacker can leverage a SQL injection vulnerability to retrieve data from other tables within the database. This is done using the UNION keyword, which lets you execute an additional SELECT query and append the results to the original query.

For example, if an application executes the following query containing the user input "Gifts":</p>

````sql
SELECT name, description FROM products WHERE category = 'Gifts'
````

<p>then an attacker can submit the input:</p>

```sql
' UNION SELECT username, password FROM users--
````

<p>This will cause the application to return all usernames and passwords along with the names and descriptions of products.</p>

<p>The UNION keyword lets you execute one or more additional SELECT queries and append the results to the original query. For example:</p>


````sql
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
````

<p>This SQL query will return a single result set with two columns, containing values from columns a and b in table1 and columns c and d in table2.

For a UNION query to work, two key requirements must be met:</p>

- The individual queries must return the same number of columns.
- The data types in each column must be compatible between the individual queries.


<p> To carry out a SQL injection UNION attack, you need to ensure that your attack meets these two requirements. This generally involves figuring out:</p>

- How many columns are being returned from the original query?
- Which columns returned from the original query are of a suitable data type to hold the results from the injected query?



#### Determining the number of columns required in a SQL injection UNION attack

<p>When performing a SQL injection UNION attack, there are two effective methods to determine how many columns are being returned from the original query.</p>

<p>The first method involves injecting a series of ORDER BY clauses and incrementing the specified column index until an error occurs. For example, assuming the injection point is a quoted string within the WHERE clause of the original query, you would submit:</p>

````sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
etc.
````


<p>This series of payloads modifies the original query to order the results by different columns in the result set. The column in an ORDER BY clause can be specified by its index, so you don't need to know the names of any columns. When the specified column index exceeds the number of actual columns in the result set, the database returns an error, such as:</p>

````
The ORDER BY position number 3 is out of range of the number of items in the select list.
````
<p>
The application might actually return the database error in its HTTP response, or it might return a generic error, or simply return no results. Provided you can detect some difference in the application's response, you can infer how many columns are being returned from the query.</p>

<p>The second method involves submitting a series of UNION SELECT payloads specifying a different number of null values:</p>

````sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
etc.
````

<p>If the number of nulls does not match the number of columns, the database returns an error, such as:</p>

````
All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.
````

<p>Again, the application might actually return this error message, or might just return a generic error or no results. When the number of nulls matches the number of columns, the database returns an additional row in the result set, containing null values in each column. The effect on the resulting HTTP response depends on the application's code. If you are lucky, you will see some additional content within the response, such as an extra row on an HTML table. Otherwise, the null values might trigger a different error, such as a NullPointerException. Worst case, the response might be indistinguishable from that which is caused by an incorrect number of nulls, making this method of determining the column count ineffective.</p>

#### Finding columns with a useful data type in a SQL injection UNION attack

<p>The reason for performing a SQL injection UNION attack is to be able to retrieve the results from an injected query. Generally, the interesting data that you want to retrieve will be in string form, so you need to find one or more columns in the original query results whose data type is, or is compatible with, string data.

Having already determined the number of required columns, you can probe each column to test whether it can hold string data by submitting a series of UNION SELECT payloads that place a string value into each column in turn. For example, if the query returns four columns, you would submit:</p>

<p>If the data type of a column is not compatible with string data, the injected query will cause a database error, such as:</p>

````
Conversion failed when converting the varchar value 'a' to data type int.
````
<p>If an error does not occur, and the application's response contains some additional content including the injected string value, then the relevant column is suitable for retrieving string data.</p>




