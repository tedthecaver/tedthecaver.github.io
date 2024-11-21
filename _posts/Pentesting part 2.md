# Kioptrix 
---
![[Pasted image 20241109192312.png]]
- we are logged into the Kioptrix machine 
![[Pasted image 20241109192711.png]]
- a simple ping command shows us our ip address 

Now going onto our kali machine we determine our ip with `ifconfig` and then perform `netdiscover -r x.y.z.0/24` which uses address resolution protocol to discover everything on the network and we are sweeping everything on the subnet of 24.
![[Pasted image 20241109194505.png]]
- we can see that the ip is the same as we get from pinging with the Kioptrix machine

## Nmap #Nmap 
---
SYN (hey port are you open) --> SYNACK (yeah I'm open lets make the connection) --> ACK (*makes connection*)
- we can use Nmap which identifies open ports similar to the three way handshake
- Nmap follows the same SYN --> SYNACK but it goes to SYNACK --> RST 
- this means instead of connecting to the port it simply shows us which ports are open which is why it is considered a stealthy scan

We can do `nmap -T4 -p- -A <ip>`, where `-T4` represents the speed, 1 being the slowest and 5 being the fastest. Slower is better for detection but it is slow. `-p-` means i want to scan all ports, without -p- we will scan the 1000 most common ports however that leaves 64535 ports left unscanned. `-A` stands for everything, version info, operating system, etc.

![[Pasted image 20241109202226.png]]
The Nmap scan shows us a lot of important information such as the different ports and the services running on them as well as the version.

![[Pasted image 20241109202602.png]]
- is the rest of the information 

## Enumerating HTTP/HTTPS #http #https
---
![[Pasted image 20241109204818.png]]
- it is interesting to note that https and http are different, our target machine does not have any encryption algorithms to support https connection it seems 
- however port 443 is open and running https so....
- try to add a security exception

![[Pasted image 20241109210750.png]]
- not working....
![[Pasted image 20241109211240.png]]
- still not working....
![[Pasted image 20241109211452.png]]
- I'm not even sure if it is possible to work 

![[Pasted image 20241109212236.png]]
- even another browser does not work i give up

![[Pasted image 20241109205119.png]]
- at least http works

The default webpage hints at poor hygiene. Are there hidden directories behind this?, is there another host?, if not why is this page still up, signals the admin is lazy and potentially left more vulnerabilities.   

![[Pasted image 20241109213736.png]]
404 page showing us a bit more information than it should

We can run `nikto` which is a web vulnerability scanning tool 
![[Pasted image 20241109214359.png]]
![[Pasted image 20241109214434.png]]
We can see the Apache version is outdated and open to exploits as well as OpenSSL.
![[Pasted image 20241109214859.png]]
![[Pasted image 20241110225752.png]]
file extensions help us narrow down the search, Apache usually runs php and Microsoft usually runs asp, this info is important to gather during enumeration
.txt, .zip, .rar are other examples of common files 

Viewing the source code of the website is important, we should be looking for comments, keys, passwords, etc. 

![[Pasted image 20241110231343.png]]
in the responses section, 200s usually mean its ok, 300s is a redirect, 400s are response errors and 500s are server errors 

![[Pasted image 20241110231600.png]]
might be an interesting file 

![[Pasted image 20241110231907.png]]
always scroll to the bottom of the page, we can see we have Webalizer version 2.01 which might have known exploits 

![[Pasted image 20241110232608.png]]
seems to be a list of folders and directories, could be interesting 

## Enumerating SMB #smb
---
SMB is a file share, consider the scans folder, when you scan something on a printer and it magically appears in your scans folder that is an example of SMB. A lot of internal exploits such as MS17010.

![[Pasted image 20241109202226.png]]
- refresher of the nmap scan which shows SMB open on port 139

![[Pasted image 20241109202602.png]]
- `-A` shows us extra info with Host scrip results
- we can see the SMB version (SMB2) which may have known exploits

In Metasploit we can search for smb
![[Pasted image 20241113174505.png]]
- shows us important info on the service as well as the exploit info
- if we want to use 389 we can enter `use 389` or `use auxiliary/scanner/smb/smb_version` 

![[Pasted image 20241113175131.png]]
- `info` shows us what we are doing in the module 
- RHOSTS and RPORT refers to the target hosts and port  

![[Pasted image 20241113175616.png]]
- we have set the RHOSTS with the target IP and found the Samba version 

`smbclient` allows us to connect to the file share possibly with anonymous access. A list of files on the server and the information inside will be huge in the exploitation phase of the mission.
 ![[Pasted image 20241113180138.png]]
 - anonymous login is successful and we are able to see filenames

![[Pasted image 20241113180418.png]]
- we cannot access the ADMIN$ file 

![[Pasted image 20241113180602.png]]
- we're in! :)

![[Pasted image 20241113180711.png]]
- and now we've hit a dead end :/

## Enumerating SSH #ssh
---
![[Pasted image 20241109202226.png]]
- refresh of nmap scan which shows ssh open on port 22

![[Pasted image 20241113181631.png]]
- we have simply used ssh to connect 

![[Pasted image 20241113181741.png]]
- some machines have difficulty with ssh so we can attempt to login like this 
- as soon as we enter a password we are moving from the enumeration phase to the exploitation phase 
- sometimes we can see a banner when sshing which gives us info on the server, version etc. 
- not a lot of remote code execution availabilities with ssh

## Researching Potential Vulns
---
80/443 and 139/445 are the most vulnerable usually so we should research those first. Remember the `nikto` scan which showed us that Apache mod_ssl 2.8.4 was potentially vulnerable. 

![[Pasted image 20241113185424.png]]

- https://www.cvedetails.com/ is a good place to look at scorings of potential vulnerabilities 

![[Pasted image 20241113190919.png]]
- that link is popping up again

![[Pasted image 20241113191303.png]]
- now we are looking for samba 2.2.1a exploit 
- Rapid7 is always nice to see since it makes Metasploit

![[Pasted image 20241113191707.png]]
- the version number fits
- we did get anonymous access to IPC$

![[Pasted image 20241113192007.png]]
- shows us how to exploit within the frameworks of Metasploit

![[Pasted image 20241113193757.png]]
- `seachsploit` allows us to search for the same exploits as on ExploitDB if we don't have internet connection or are in a pinch

![[Pasted image 20241113194228.png]]
- `searchsploit mod ssl 2` shows us exploits for mod_ssl 2 
- we can see on the RHS that remote is a big look for us

## Scanning with Nessus #nessus
---
After downloading Nessus from the terminal we can go to the Downloads folder and run `dpkg -i <Nessus_filename>`. Run the given command then access the https page, accessing the page without running the command will not give you the option to bypass the security certificate.

![[Pasted image 20241115152632.png]]
- running a basic Nessus scan with the Kioptrix ip yields us many possible vulnerabilities

![[Pasted image 20241115153738.png]]
- it seems that SSL itself is vulnerable and we can exploit it
- important to note that Nessus doesn't tell us how to exploit it directly we need to do our own research

## Reverse shells vs. Bind shells
---
![[Pasted image 20241115154905.png]]
- A shell is simply access to a machine
- A reverse shell means the target connects to us
- In the picture we have setup a listener with netcat on our machine, `-lvnp 4444` means listening on port 4444 with verbose results 
- the target machine is connecting to our ip on port 4444 and then execute `/bin/sh` which is a Linux machine
- if it was a windows machine `/bin/sh` ---> `cmd.exe` 

![[Pasted image 20241115155713.png]]
- in a bind shell we open a port on the machine then connect to it
- the target machine deploys the listener this time
- the execution, `-e /bin/sh` still happens on the target 

## Staged vs. Nonstaged #payloads
---
A payload is what we run as an exploit. This is what we send to the victim and attempt to get a shell on their machine.

![[Pasted image 20241115161556.png]]
  - we can see the stages are separated by a slash 
  - not every payload works we need to try out different combos, e.g. bind shell + Staged or reverse shell + Non-staged

## Gaining root w/ #Metasploit
---
![[Pasted image 20241115162204.png]]
- remember we searched samba 2.2 vulns (smb on port 139) and a few came up so let's target samba 
- we had the IPC$ anon connection so this is a good place to start
- we also had found trans2open on Rapid7 in our enumeration process

![[Pasted image 20241115163207.png]]
- our enumeration showed us that the target is a Linux machine so we know which module to choose

![[Pasted image 20241115163759.png]]
- after doing `use 1` we are given the option to set the RHOST so we go ahead and do that

![[Pasted image 20241115164202.png]]
- what is happening here?
- Metasploit is trying brute force attacks on different return addresses until it finds one it can send the payload/stage to
- However the stage keeps dying  

![[Pasted image 20241115165203.png]]
- cancelling the operation and going back to options we see we have a new option 
- we are running a staged payload as seen by the `linux/x86/meterpreter/reverse_tcp` 
- we can do `set payload linux/x86` and then hit TAB twice to see the payload options

![[Pasted image 20241115170021.png]]
- we have switched to a non-staged payload
- let's run this payload now

![[Pasted image 20241115172002.png]]
- and it worked! we are root :)  






