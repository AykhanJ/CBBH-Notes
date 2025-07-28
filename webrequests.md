# Web Requests

Curl Command:

| Commands      | Purpose       |
|---------------|---------------|
| `curl inlanefreight.com` | prints the site in its raw format | 
| `curl -O inlanefreight.com/index.html` | downloading a file| 
| `curl -s -O http://example.com` | silent mode, doesn't show when a file is downloading| 


if we contact a website with an invalid SSL certificate, then cURL by default would not proceed with the communication. To bypass this: 
`curl -k https://inlanefreight.com`

# GET Request

| Commands      | Purpose       |
|---------------|---------------|
| `curl inlanefreight.com -v` | verbose mode| 
| `curl -i http://<SERVER_IP>:<PORT>/` | Method and raw codes of the site| 
| `curl -u admin:admin http://<SERVER_IP>:<PORT>/` | Using credentials to authenticate in a site| 
| `curl -H 'Authorization: Basic YWRtaW46YWRtaW4=' http://<SERVER_IP>:<PORT>/` | Basic Auth used in Base64 encoding|   

# POST Request

| Commands      | Purpose       |
|---------------|---------------|
| `curl -X POST -d 'username=admin&password=admin' http://<SERVER_IP>:<PORT>/` | POST Authentication with data|
| `curl -X POST -d 'username=admin&password=admin' http://<SERVER_IP>:<PORT>/ -i` | Getting cookie| 
| `curl -b 'PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1' http://<SERVER_IP>:<PORT>/` | Using cookie to not using credentials every time| 
| `curl -X POST -d '{"search": "london"}' -b 'PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1' -H 'Content-Type: application/json' http://<SERVER_IP>:<PORT>/search.php` | searching for the word ' london ' in search | 
