## Enumeration
Starting off with this box we are provided a set of credentials.
tyler:LhKL1o9Nm3X2

Like all machines, I like to start off with a quick nmap -sV scan which returns that we have SSH and HTTP running on ports 22 and 80.

![nmap scan results](../../assets/Outbound-HTB/01-nmapscan.png "Nmap scan results")


I tried connecting to the SSH service with Tylers credentials, but no dice. A quick searchsploit for the http and ssh services confirmed 
no known exploits for their versions. 

I decided to look into what was running on the web server, trying to connect to http://10.129.232.158 returns a error saying "Firefox can't establish a connection to the server at mail.outbound.htb". So we add mail.outbound.htb to the /etc/hosts file.

![Connection error](../../assets/Outbound-HTB/02-connectionerror.png "Connection error")

Reconnecting lands us on a login page for "Roundcube Webmail". Naturally we try Tylers credentials here, and they work. Tyler has nothing in his inbox but checking the about button reveals that the server is running Roundcube Webmail 1.6.10. I ran searchsploit roundcube 1.6.10 and sure enough there is a known RCE exploit for this version and Kali has an exploit for it preinstalled. 

I did some research to where the exploitdb exploits are stored in Kali and discovered they are stored in /usr/share/exploitdb/exploits and the searchsploit command tells us that from there the specific exploit we want is in /multiple/webapps/52324.NA. I copied this file into my working directory for the box and did a little investigation to see what we were working with. running file 52324.NA tells us it's a Ruby script indicating it's likely intended for use in Metasploit, and running cat 52324.NA and looking at the top we see a comment noting that the module requires metasploit. 

![Exploit investigation](../../assets/Outbound-HTB/03-exploitinvestigation.png "Exploit investigation")

## Exploitation
I renamed the exploit to roundcubeexploit.rb and then moved it to /usr/share/metasploit-framework/modules/exploits/multi/ so the exploit is usable in Metasploit. 

![Exploit moved to metasploit](../../assets/Outbound-HTB/04-exploitmovedtometa.png "Exploit moved to metasploit")

Then we can open Metasploit and select the exploit. We need to set some options for it to work so I ran
set rhosts 10.129.232.158 - This sets the target machine
set lhost 10.10.14.128 - Sets the lhost to our local machine
set vhost mail.outbound.htb - Sets vhost option to our target (more on this later)
set username tyler - The exploit requires credentials so we give it the ones we are provided
set password LhKL1o9Nm3X2 - Same as above

![Metasploit options set](../../assets/Outbound-HTB/05-metasploitoptionsset.png "Metasploit options set")

## Foothold
With all the options set we are ready to run the exploit and see what we get. 

![Meterpreter shell](../../assets/Outbound-HTB/06-meterpretershell.png "Meterpreter shell")

Awesome the exploit worked and got us a meterpreter session which we can use to get a shell as www-data. 

At this point I got a little stuck trying to figure out how to progress, so I had to use a hint from ippsec's video. And the hint was, always look for the database when you get a shell on a web server. Duh.

So I ran cd ~ to start in www-data's home directory and worked my way through the folders. Landing in /home/html/Roundcube shows us there are many directories to look at. Pausing here and thinking led to me to check out the /config folder, which contains a file called "config.inc.php". Running cat on this shows us a bunch of config options, one of which includes the database config. This line tells us all we need to know, "$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube'"

![Config file](../../assets/Outbound-HTB/07-configfile.png "Config file")

So from here I knew we would be connecting to the DB. I knew I would need to do some brushing up on enumerating databases so I went back to the HTB Starting Point room "Sequel" that I had completed previously. The info in here was great but when trying to enumerate the DB in my shell I was getting a blank screen after connecting. This is when I learned about upgrading shells. I did some research into how and why to upgrade a shell as well as listened to ippsecs explanation which all boiled down to running the following 3 commands.

script -qc /bin/bash /dev/null
export TERM=xterm
stty rows 40 cols 120

More explanation on this below.

Now that we have the shell upgraded so it can display the DB properly we can connect to it using the command mysql -u roundcube -p. This will connect using the username roundcube and password authentication. We know from the config line we found that roundcube is the username and RCDBPass2025 is the password. Running this gets us in to the DB.

![Database connection](../../assets/Outbound-HTB/08-dbconnection.png "Database connection")

Once connected I began to enumerate the DB by running show databases; and selected the roundcube database. From there I ran show tables; which returned all of the tables, the ones of immediate interest to me were users and identities. I ran describe on both of these tables to see the fields they have. 

The main thing of interest I see is the username in the users table, so I ran select username from users; and we get back jacob, mel, and tyler. 

![Database enumeration](../../assets/Outbound-HTB/09-dbenumeration.png "Database enumeration")

We are making progress and I feel either mel or jacob is going to be the key to our user flag but we still don't see passwords stored anywere. I got stuck here for a while and wasn't sure how to progress so I decided to consult ippsecs video. 
From watching a bit of his video I now know that the session table is of interest and contains the next step of progression. More info below on the reasoning behind finding this. 

Running select vars from session; returns to huge blocks of what appears to be base64 encoded text. Using cyberchef to decode the first block we get what appears to be session info for the user jacob, which includes a line for password and has the value L7Rv00A8TuwJAr67kITxxcSgnIk25Am/. Doing the same for the second block returns the same but for the user tyler, and has the password v3yK8bHlPqYG4VcfUE2uaC2reDvy8WUp. It's obvious these are encrypted but decrypting them is the question. 

![Cyberchef decoding](../../assets/Outbound-HTB/10-cyberjacob.png "Cyberchef decoding")

Once again at this point I got stuck and had to reference ippsecs video which made me realize I missed 2 ciritcal things when exploring this box. First off, in the config file we got the DB credentials from there was a 3DES key in there as well, this tells us the encryption algorithm used and gives us the key to decrypt it. Overlooking reading the entire config file also caused me to miss the fact that it says right in it the users imap password is stored in the session file and then provides the decryption key. Using the passwords and the key we can decrypt them in Cyberchef.

I also should have fully checked the directories in the Roundcube folder. In the bin directory there is a script called decrypt.sh. When we run it and provide the passwords from the DB we are able to get the following credentials. jacob:595mO8DmwGeD and tyler:LhKL1o9Nm3X2. Tyler's password comes back as the same one we started with so it looks like the process worked. 

![Decrypting password](../../assets/Outbound-HTB/11-decrypt.png "Decrypting password")

With the credentials we now have for jacob I decided to attempt to log into the Roundcube app. Logging in with these credentials worked and we see a couple of emails, one of which is from Tyler and contains info about a password update telling Jacob that his new password is gY4Wr3a1evp4.

![Password in email](../../assets/Outbound-HTB/12-jacobpassword.png "Password in email")


We know this isn't Jacob's email password so I decided to try it out on the SSH service. The SSH connection works and we find the user flag in Jacob's home directory.

![User flag](../../assets/Outbound-HTB/13-userflag.png "User flag")

## Privelege Escalation
To begin the privelege escalation I always start with sudo -l. Immediately we notice that Jacob can run below as sudo, I knew this had to be something of interest but I got stuck here for a while and decided to consult ippsecs video on how to exploit this. I got so caught up in trying to find a specific version to search for exploits that I completely overlooked just googling "linux below exploit" which returned results indicating that there might be a local privelege escalation exploit.. 

![Sudo -l results](../../assets/Outbound-HTB/14-sudol.png "Sudo -l results")

Having read and watched how the attack works we are ready to try it on our box. In short, running below generates a file called error_root.log inside of /var/log/below, below automatically chmods this file to be 666 permissions. By removing this file, creating a symlink in it's place and running below again you can get it to chmod any file on the system to 666 allowing you to edit it, in this case we will do it on passwd. Once we do that we borrow the part of rvizx exploit found here (link to github) to create a user called pwn with root persmissions. Then we su to pwn, move to the home directory and cat our root.txt

![Root flag](../../assets/Outbound-HTB/15-rootflag.png "Root flag")

## Resources used
Exploit for below - https://github.com/rvizx/CVE-2025-27591
Explanation of below vulnerability - https://security.opensuse.org/2025/03/12/below-world-writable-log-dir.html
ippsec CTF walkthrough - https://www.youtube.com/watch?v=bDql3eTHgZ8&t=1253s

## Things I learned / Explanations
More coming.
