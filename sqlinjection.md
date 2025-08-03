# What is SQL Injection?

SQL Injection happens when a hacker sends input that changes a database query. 
Instead of just treating the input as normal text, the app runs it as part of the SQL command. This lets the attacker access, change, or delete data.

# Some MySQL Commands:

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
