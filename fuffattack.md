# Basic directory fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ`

# Extension Fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://SERVER_IP:PORT/blog/indexFUZZ`

# Page fuzzing after determining the extension:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php`

# Recursive Scan:

- Recursive Fuzzing
So far, we’ve been fuzzing directories, then going inside them to find files. But if there are many folders with subfolders, this process becomes very slow. To fix this, we can use recursive fuzzing, which automates the process.

- What Is Recursive Fuzzing?
Recursive fuzzing means that once a tool finds a folder, it will automatically scan inside it, and keep doing this for every new folder it finds.

Some websites have deep folder paths like /login/user/content/uploads/..., which can make scans very long. To avoid wasting time, we can set a depth limit—this tells the tool how many levels deep it should scan.

# Using ffuf for Recursive Fuzzing:
- Use -recursion to turn on recursive scanning.

- Use -recursion-depth 1 to scan only main folders and their direct subfolders.

For example: it will scan /login and /login/user, but not /login/user/content.

- Use -e .php to tell ffuf to look for .php pages.

- Add -v to show full URLs in the output (helpful for seeing where each file is).

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v`
