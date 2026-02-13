**Building an IT Helpdesk Lab: From Active Directory to Patch Management**

This document outlines the complete setup of my on-premises IT lab. The goal was to simulate a real-world corporate environment to practice core IT support and system administration skills, including user management, file sharing, and patch management.

**Lab Foundation & Virtual Networking**

I used Oracle VirtualBox as the hypervisor on my main PC(laptop). The first decision was the network configuration. I needed a setup where virtual machines (VMs) could communicate with each other and access the internet independently.

- **Network Type:** I chose **NAT Network** (named `NatNetwork`). This creates a private subnet (`10.0.2.0/24`) and a virtual router (`10.0.2.1`) within VirtualBox. All VMs attached to this network can see each other, and their internet traffic is routed through my host PC.  
    ![[Pasted image 20260213215132.png]]
    
- **DHCP:** The NAT Network has a built-in DHCP server. I later created my own DHCP server on the Domain Controller for more control, but this initial automatic assignment gets the VMs online quickly.  
    ![[Pasted image 20260213215253.png]]
    

**Domain Controller Setup (Windows Server 2022)**

I created a VM named "AD-VM" and installed Windows Server 2022.  
![[Pasted image 20260213215346.png]]

- **Static IP Configuration:** After installation, the first task was to set a static IP address. I opened Network Settings, went to IPv4 properties, and entered:
    
    - **IP Address:** `10.0.2.10`
        
    - **Subnet Mask:** `255.255.255.0`
        
    - **Default Gateway:** `10.0.2.1` (The virtual router for the NAT Network)
        ![[Pasted image 20260213220625.png]]
    - **Preferred DNS Server:** `10.0.2.10` (The server will host its own DNS)  
        
        
- **Installing AD DS and DNS:** I used Server Manager to add the **Active Directory Domain Services (AD DS)** and **DNS Server** roles. After the installation, I promoted the server to a Domain Controller for a new forest named `kaden.com`. This process automatically configures DNS to host the `kaden.com` zone.  
    ![[Pasted image 20260213220801.png]]
    
- **DNS Forwarders:** After promotion, the server could resolve internal names but not external ones (like `google.com`). I opened the DNS Manager console, right-clicked the server name, went to Properties > Forwarders, and added `8.8.8.8`. This allows the DNS server to forward requests for external domains.  
    ![[Pasted image 20260213220847.png]]
    
- **DHCP Server Role:** To automatically assign IPs to client machines, I installed the **DHCP Server** role. After installation, I created and activated a new scope:
    
    - **Scope Name:** Lab Clients
        
    - **IP Range:** `10.0.2.100` - `10.0.2.200`
        
    - **Subnet Mask:** `255.255.255.0`
        
    - **Default Gateway:** `10.0.2.1`
        
    - **DNS Servers:** `10.0.2.10`
        
    - **Authorize:** I had to authorize the DHCP server in Active Directory for it to start issuing leases.
        

**Client Setup (Windows 11 Pro) & Initial AD Tasks**

I created a second VM named "Win11" for the client.

- **Installation & Network:** I installed Windows 11 Pro. During setup, I connected it to the same `NatNetwork`. After setting up the server's DHCP scope, the client automatically received an IP from my defined range (`10.0.2.100-200`).  
    ![[Pasted image 20260213220943.png]]
    
- **Creating User Accounts in AD:** Before joining the domain, I created test user accounts (e.g., `John Doe`, `Jane Smith`) in Active Directory Users and Computers (ADUC) on the DC.
    
- **Joining the Domain:** With the client on the same network as the DC and receiving the correct DNS server (`10.0.2.10`), I went to **Settings > Accounts > Access work or school** and clicked "Connect". I chose the option to join a local Active Directory domain, entered `kaden.com`, and provided the domain administrator credentials. The join was successful after a reboot. I then logged in as one of the test domain users.
    

**Implementing Patch Management with Action1**

I set up a patch management solution to keep systems updated.

- **Action1 Account:** I signed up for a free Action1 account, which is designed for patch management and includes features for reporting and audits.
    ![[Pasted image 20260213223222.png]]
- **Deploying the Agent:** On both the AD-VM and Win11, I downloaded and installed the Action1 agent.
    
- **Creating a Patch Policy:** In the Action1 cloud console, I created a patch management policy to automatically approve and install critical security updates for both endpoints on a schedule.
    ![[Pasted image 20260213223450.png]]
- **Running Reports:** After the first patch cycle, I ran a report to verify installed updates and generate documentation for the lab's patch compliance status.
    

**Configuring File Shares and Permissions**

I set up network file sharing with proper security.

- **Creating Security Groups:** In ADUC, I created security groups like `Sales_Team` and `HR_Team`.
    
- **Adding Users to Groups:** I added my test user accounts to the relevant groups (e.g., `John.Doe` to `Sales_Team`).
    
- **Creating Shared Folder on DC:** On the AD-VM, I created a folder named `CompanyData`. I then created subfolders like `Sales` and `HR`.
    ![[Pasted image 20260213223859.png]]
- **Setting NTFS and Share Permissions:**
    ![[Pasted image 20260213224016.png]]
    - Folder above is only accessible by Administrators for example.
      If I wanted to grant access to more team members this would pop up:
      ![[Pasted image 20260213224237.png]]
	- For the `Sales` folder, I removed default permissions and granted `Sales_Team` the "Read/Write" share permission and "Modify" NTFS permission.
        
    - For the `HR` folder, I granted `HR_Team` similar permissions and explicitly denied access to the `Sales_Team` for testing.
        
- **Testing:** From the Win11 client logged in as a Sales user, I successfully accessed the `Sales` share via `\\AD-VM\CompanyData\Sales` but was correctly denied access to the `HR` folder.
    

**Advanced User and Security Policies**

I configured more granular user and device management.

- **Logon Hours:** In ADUC on the user account `John.Doe`, I configured **Logon Hours** to restrict login access to business hours only (e.g., Monday-Friday, 8 AM - 6 PM). An attempt to log in outside these hours would be blocked.
    ![[Pasted image 20260213224638.png]]
- **Account Lockout Policies:** I configured a **Group Policy Object (GPO)** linked to the domain to enforce an account lockout policy after a certain number of failed logon attempts, enhancing security against brute-force attacks.
    
- **Asset Management with Action1:** I used Action1's dashboard to view detailed asset inventories of both VMs, check for CVEs (Common Vulnerabilities and Exposures) affecting installed software, and ensure all systems were patched against known vulnerabilities.
    

**Troubleshooting Notes**

- **Ping Not Working:** Initially, I couldn't ping the DC from other VMs. This was due to the Windows Firewall on the server. The solution was to enable the "File and Printer Sharing (Echo Request - ICMPv4-In)" rule in Windows Defender Firewall with Advanced Security.
    
- **DHCP Not Authorized:** The DHCP server would not start until I right-clicked the server name in the DHCP console and selected "Authorize". This is an Active Directory security requirement.
    

**Conclusion**

The lab is now fully functional and mimics a small business IT environment. I have a working Active Directory domain, automated IP address assignment via DHCP, internal and external DNS resolution, and have implemented practical IT tasks including patch management with Action1, secure file sharing with permission testing, and user access policies like logon hours and account lockout. This environment provides a safe and realistic space to practice the skills required for a helpdesk or junior system administrator role.