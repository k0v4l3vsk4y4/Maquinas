# Kenobi - THM

NMAP de la máquina
```bash
nmap -sC -sV -n -v 10.10.228.42

PORT     STATE SERVICE      VERSION
21/tcp   open  ftp          ProFTPD 1.3.5
22/tcp   open  ssh          OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|_  256 5a:06:ed:eb:b6:56:7e:4c:01:dd:ea:bc:ba:fa:33:79 (ED25519)
80/tcp   open  http         Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
111/tcp  open  rpcbind      2-4 (RPC #100000)
139/tcp  open  netbios-ssn?
445/tcp  open  netbios-ssn  Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
2049/tcp open  nfs          2-4 (RPC #100003)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.00: 
|_    Message signing enabled but not required
|_smb2-time: Protocol negotiation failed (SMB2)

NSE: Script Post-scanning.
Initiating NSE at 09:56
Completed NSE at 09:56, 0.00s elapsed
Initiating NSE at 09:56
Completed NSE at 09:56, 0.00s elapsed
Initiating NSE at 09:56
Completed NSE at 09:56, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 147.59 seconds
```
Using nmap we can enumerate a machine for SMB shares.

```bash
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.228.42

Starting Nmap 7.91 ( https://nmap.org ) at 2021-12-03 10:13 EST
Nmap scan report for 10.10.228.42
Host is up (0.050s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.228.42\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 2
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.228.42\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.228.42\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>

Nmap done: 1 IP address (1 host up) scanned in 9.99 seconds
```
Using your machine, connect to the machines network share.
Once you're connected, list the files on the share. What is the file can you see?
```bash
smbclient //10.10.228.42/anonymous -N    

1 ⨯
Try "help" to get a list of possible commands.
smb: \> ls
NT_STATUS_IO_TIMEOUT listing \*
smb: \> SMBecho failed (NT_STATUS_INVALID_NETWORK_RESPONSE). The connection is disconnected now
NT_STATUS_CONNECTION_DISCONNECTED listing \*
                                                                                                                                                                                                                           
┌──(kali㉿kali)-[~]
└─$ smbclient //10.10.228.42/anonymous   
Enter WORKGROUP\kali's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 06:49:09 2019
  ..                                  D        0  Wed Sep  4 06:56:07 2019
  log.txt                             N    12237  Wed Sep  4 06:49:09 2019

                9204224 blocks of size 1024. 6877108 blocks available
smb: \> 
```
Download the file log.txt
```bash
smbclient //10.10.228.42/anonymous -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 06:49:09 2019
  ..                                  D        0  Wed Sep  4 06:56:07 2019
  log.txt                             N    12237  Wed Sep  4 06:49:09 2019

                9204224 blocks of size 1024. 6877108 blocks available
smb: \> cat log.txt
cat: command not found
smb: \> help
?              allinfo        altname        archive        backup         
blocksize      cancel         case_sensitive cd             chmod          
chown          close          del            deltree        dir            
du             echo           exit           get            getfacl        
geteas         hardlink       help           history        iosize         
lcd            link           lock           lowercase      ls             
l              mask           md             mget           mkdir          
more           mput           newer          notify         open           
posix          posix_encrypt  posix_open     posix_mkdir    posix_rmdir    
posix_unlink   posix_whoami   print          prompt         put            
pwd            q              queue          quit           readlink       
rd             recurse        reget          rename         reput          
rm             rmdir          showacls       setea          setmode        
scopy          stat           symlink        tar            tarmode        
timeout        translate      unlock         volume         vuid           
wdel           logon          listconnect    showconnect    tcon           
tdis           tid            utimes         logoff         ..             
!              
smb: \> get log.txt
getting file \log.txt of size 12237 as log.txt (58.0 KiloBytes/sec) (average 58.0 KiloBytes/sec)
smb: \> exit
                                                                                                                                                                                                                           
┌──(kali㉿kali)-[~]
└─$ cat log.txt 
```

Your earlier nmap port scan will have shown port 111 running the service rpcbind. This is just a server that converts remote procedure call (RPC) program number into universal addresses. When an RPC service is started, it tells rpcbind the address at which it is listening and the RPC program number its prepared to serve. 

In our case, port 111 is access to a network file system. Lets use nmap to enumerate this.

´´´bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.228.42
Starting Nmap 7.91 ( https://nmap.org ) at 2021-12-03 10:46 EST
Nmap scan report for 10.10.228.42
Host is up (0.066s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *

Nmap done: 1 IP address (1 host up) scanned in 1.13 seconds
                                                
```

