# URL Encoding

| Character      | Encoding       |
|---------------|---------------|
| `space` | 	%20 | 
| `!` | 	%21 | 
| `"` | 	%22 | 
| `#` | 	%23 | 
| `$` | 	%24 | 
| `%` | 	%25 | 
| `&` | 	%26 | 
| `'` | 	%27 | 
| `(` | 	%28 | 
| `)` | 	%29 | 

# Always check source codes!

# What is HTML Injection?

HTML Injection happens when a website shows user input without checking or cleaning it.

🔄 Where It Happens
From backend (e.g., showing user comments saved in the database)
From frontend (e.g., input displayed immediately using JavaScript)

`<style> body { background-image: url('https://academy.hackthebox.com/images/logo.svg'); } </style>` - The page background changes, meaning HTML Injection
