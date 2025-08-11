# Client-Side Validation

Many websites check file types (like images only) using JavaScript on your browser before uploading. This means if you try to upload a file that’s not an image, the website blocks it right away.

But this check happens only in your browser, so you can easily bypass it by:

Ignoring or changing the file upload request sent to the server, or
Editing the website’s front-end code in your browser to remove or disable the file type check.

**How This Works in Practice:**

- When you try to upload a profile image, the site only lets you pick .jpg, .png, etc.
- If you try to upload a .php file (like a web shell), the upload button gets disabled, and you see an error.
- However, the website never actually sends your file to the server until you press upload, so the server might not even check file types.

**Bypassing Validation by Modifying the Request**

- Using tools like Burp Suite, you can capture the file upload request.
- Change the file name from image.png to shell.php and replace the file content with your malicious script.
- Send this modified request directly to the server.
- If the server doesn’t check the file type, your PHP web shell uploads successfully.

**Bypassing Validation by Editing Front-End Code**

- Open your browser’s developer tools.
- Find the file input element and the JavaScript function that checks file types.
- Remove or disable that JavaScript validation.
- Now you can select and upload any file type (like your PHP web shell) normally.


# Blacklist Filters 

Sometimes, websites try to stop dangerous files from uploading by blocking certain file extensions (like .php) on the server side. This is called a blacklist—a list of banned file types.

**What’s the Problem?**

- The blacklist only blocks specific extensions (e.g., .php, .php7).
- It doesn’t block all possible extensions that can run code.
- The blacklist might be case-sensitive, so pHp or Php might get through.
- If the blacklist is weak, attackers can bypass it and upload harmful scripts.

**How to Bypass a Blacklist?**

- Try different file extensions that are not on the blacklist but can still run code (like .phtml).
- Use tools like Burp Suite Intruder to automatically test many extensions and find which ones the server allows.
- When you find an allowed extension, upload your malicious script with that extension.
- Then visit the uploaded file URL to run your code.

**Example**

- The server blocks .php but not .phtml.
- You upload shell.phtml with your web shell code.
- The upload works, and your script runs on the server.



# When are whitelists used?

- When uploads should be limited to a few safe types (like profile images).
- Sometimes used together with blacklists.

**How whitelist validation can fail:**

Many whitelists check if the filename contains an allowed extension instead of ends with it.

Example code with weak regex:


<pre>$fileName = basename($_FILES["uploadFile"]["name"]);

if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}</pre>

This only checks if the filename has .jpg, .jpeg, .png, or .gif anywhere in the name, not necessarily at the end.

**Double Extensions Bypass**

You can upload a file named like this:

`shell.jpg.php`

It passes the above regex check (because it contains .jpg), but actually ends with .php. So, it uploads a PHP script disguised as an image.

Try fuzzing file names with this payload to find allowed extensions:

Modify a normal upload request to:

`shell.jpg.php`

With PHP shell content and send it.

If successful, visit:

`http://SERVER_IP:PORT/profile_images/shell.jpg.php?cmd=id`

You get remote code execution.

**Strict Regex Example**

Some apps use stricter regex:

<pre>if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) {
    echo "Only images are allowed";
    die();
}</pre>

This forces the filename to end with an allowed extension. The previous double extension trick won't work here.

**Web Server Misconfiguration Bypass (Reverse Double Extension)**

If the Apache server has this config:

<pre><FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch></pre>

It treats any file containing .php, .phtml, or .phar as PHP — even if it doesn't end with those.

**Example: Upload a file named `shell.php.jpg`**

which passes the strict whitelist but will execute as PHP.

Try uploading:

`shell.php.jpg`

Visit:

`http://SERVER_IP:PORT/profile_images/shell.php.jpg?cmd=id`

And you get code execution.

**Character Injection Bypass**

You can insert special characters to trick filename checks:

Common payload characters:

`%20, %0a, %00, %0d0a, /, .\, ., …, :`

Example trick:

`shell.php%00.jpg`

On some PHP versions, the server treats this as shell.php, bypassing the whitelist.

On Windows servers:

`shell.aspx:.jpg`

will be treated as shell.aspx.

Bash script to generate these payloads for fuzzing:

<pre>for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
    for ext in '.php' '.phps'; do
        echo "shell$char$ext.jpg" >> wordlist.txt
        echo "shell$ext$char.jpg" >> wordlist.txt
        echo "shell.jpg$char$ext" >> wordlist.txt
        echo "shell.jpg$ext$char" >> wordlist.txt
    done
done</pre>

Use this wordlist.txt with Burp Intruder to fuzz the upload and find bypasses.


# Type Filters

To improve security, many web servers and apps also check the actual content of the uploaded file to make sure it matches the expected type (like images, videos, documents). These checks usually don’t use blacklists or whitelists but look at the file content instead.

There are two main ways to validate file content:

**1. Content-Type Header**

This is a part of the upload request that tells the server the file type. For example, a PHP app might check it like this:

<pre>$type = $_FILES['uploadFile']['type'];

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}</pre>

Browsers usually set this header based on the file extension, but since this happens on the client side, attackers can change it to bypass the filter.

You can test allowed content types by fuzzing with lists like SecLists’ Content-Type wordlist:

`wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Web-Content/web-all-content-types.txt`
`cat web-all-content-types.txt | grep 'image/' > image-content-types.txt`

By intercepting an upload request and changing the Content-Type header to an allowed image type (e.g., image/jpg), you may be able to upload a malicious file disguised as an image.

**2. MIME-Type (File Content)**

This method checks the actual file bytes (called "magic bytes" or file signature) to identify the file type, not just the name or header.

For example, GIF images start with GIF87a or GIF89a. If you change the first bytes of any file to these, it will look like a GIF regardless of its extension.

Example:

`echo "this is a text file" > text.jpg`
`file text.jpg`
**Output: ASCII text**

`echo "GIF8" > text.jpg`
`file text.jpg`
**Output: GIF image data**

PHP can check MIME type like this:

<pre>$type = mime_content_type($_FILES['uploadFile']['tmp_name']);

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}</pre>

If the server tests both the Content-Type header and MIME type, you can try prepending GIF magic bytes (GIF8) to your PHP script and keep the .php extension. This can trick the server into accepting and uploading your file:

POST request to /upload.php with file 'shell.php' starting with GIF8 header

Visiting the uploaded file will run your PHP commands:

`http://SERVER_IP/profile_images/shell.php?cmd=id`

Tips:

- Sometimes a combination of tricks works best — try mixing allowed MIME types with disallowed Content-Type headers or vice versa.
- Experiment with different payloads to find what the server accepts.
