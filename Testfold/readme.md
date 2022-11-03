## Apache Login, Basic Security and Access to Files
Problem: When users are authorized through my own login-system, I want the users to get access to files **without** having Apache to prompt them for password (again)

Im having Apache, PHP and a database running om a windows server.

My login-system is home-made, using AJAX and PHP. PHP accessing the backend using PDO. 
My sites uses SSL in order for accepting password to be sendt between backend and browser in cleartext.

Creating a new user like this:
```PHP
$hashedPwd = password_hash($password, PASSWORD_DEFAULT);		
$sql = "INSERT INTO t_users (email, password) values(?, ?)";
```
Checking password:

```PHP
<?php declare(strict_types=1);
session_start();
header("Content-Type: application/json"); 
$jsonData = json_decode(file_get_contents("php://input"));
$password = $jsonData->pwd;
$sql = "SELECT id, password, email FROM t_users WHERE email = ?";
$stmt = $pdo->prepare($sql);
$stmt->bindParam(1, $mail);
$stmt->execute();
$data = $stmt->fetch(PDO::FETCH_ASSOC); 
$pwdCheck = password_verify($password, $data['password']);
if ($pwdCheck == false) {
etc etc
```
In my Apache VHOST I have this: (it is the **/play** and the **AuthDBDUserPWQuery**
```INI
Alias /play "E:\Music\Jukebox\Pop"
<Directory "E:\Music\Jukebox\Pop">
  Options +Indexes +FollowSymLinks +MultiViews
  IndexOptions IgnoreCase FancyIndexing FoldersFirst NameWidth=* DescriptionWidth=* SuppressHTMLPreamble
  IndexIgnore header.html footer.html favicon.ico .htaccess .ftpquota .DS_Store icons *.log *,v *,t .??* *~ *#
  IndexHeadInsert "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">"
  AuthType Basic
  AuthName "G12 Realm"
  AuthBasicProvider socache dbd
  AuthnCacheProvideFor dbd
  Require valid-user
  AuthDBDUserPWQuery "SELECT password FROM t_users WHERE email = %s"
</Directory>
```  
If users (who has *not* logged in to my server) hit the server with <www.myserver.tld>/**play** they will see the browsers login prompt for Basic Authentification, if they have not logged in using my own login system. Thats fine.
The server sends a 401 and the browser pops up the built-in login prompt.

On the other hand, if a users has been authenticated using my own system, my javascript will use this code:
```JAVASCRIPT
var request = new XMLHttpRequest();
request.open('GET', 'https://dev.g12.dk/inc/createBasicAutHeader.php', false, <email>, <password in cleartext>);
request.setRequestHeader('X-Requested-With', 'XMLHttpRequest');
request.send(null); 
```
This will make the browser to create a request header for all requests until the user closes the browser, like:
Authorization: Basic dGhv<bla bla bla>uZGs6SWhhd2N3NmM=

```PHP
<?php declare(strict_types=1);
$rc = 4; 
$headers = getallheaders();
foreach($headers as $key=>$val){
  if ($key == 'Authorization') {$rc = 0;}
}
echo $rc;
```

:+1:
:cook:

<img width="115" alt="image" src="https://user-images.githubusercontent.com/12120277/199652828-aa092046-2caa-4fa8-94ac-6fa8f1bf1651.png">
