# Active Directory Red/Blue Team Lab with Splunk Integration

## Project Overview

This project guides you through building a fully functional, on-premises Active Directory domain environment integrated with Splunk for Security Information and Event Management (SIEM). The lab is designed from scratch and serves as an excellent hands-on learning platform for:

* **IT Administration:** Understanding AD setup, configuration, and management (users, OUs, domain joining).
* **Blue Team (Defense & Detection):** Learning how to configure logging (Sysmon), forward logs (Splunk Universal Forwarder), ingest data into a SIEM (Splunk), and analyze telemetry to detect potentially malicious activity.
* **Red Team (Offense):** Practicing common attack techniques against an AD environment (e.g., Brute Force, Atomic Red Team tests) using tools like Kali Linux.

Completing this project provides practical experience, enhances technical skills valuable for various IT and cybersecurity roles, and offers a tangible project to discuss during job interviews.

**Acknowledgement:** This lab setup is heavily based on the fantastic 5-part "Active Directory Project (Home Lab)" series by MyDFIR on YouTube. Please refer to their channel for detailed video walkthroughs.

## Features

* **Complete AD Domain:** Windows Server 2022 configured as a Domain Controller (`mydfir.local`).
* **SIEM Integration:** Ubuntu Server hosting Splunk Enterprise for log collection and analysis.
* **Enhanced Endpoint Logging:** Sysmon installation and configuration for detailed process monitoring.
* **Log Forwarding:** Splunk Universal Forwarders configured on Windows endpoints to send logs to Splunk.
* **Client Workstation:** Windows 10 machine joined to the domain, acting as a target.
* **Attacker Machine:** Kali Linux VM pre-configured for penetration testing activities.
* **Attack Simulation:** Tools and frameworks (Crowbar, Atomic Red Team) for simulating attacks.
* **Detection Practice:** Use Splunk to query and analyze logs generated during simulated attacks.

## Lab Architecture

### Virtualization Software

* **Primary:** Oracle VirtualBox (Free)
* **Alternative (for M1/M2/M3 Macs):** Cloud providers like Vultr or Microsoft Azure, as VirtualBox may have limitations.

### Virtual Machines (VMs)

1.  **Domain Controller (`addc01`)**
    * **OS:** Windows Server 2022 (Standard Desktop Experience)
    * **Purpose:** Hosts Active Directory Domain Services.
    * **RAM:** 4GB (Install), 2GB+ (Runtime)
    * **Disk:** 50GB+
    * **IP:** `192.168.10.7` (Static)
2.  **Splunk Server (`Splunk`)**
    * **OS:** Ubuntu Server 22.04.1 LTS
    * **Purpose:** Hosts Splunk Enterprise SIEM.
    * **RAM:** 4GB - 8GB+ (More is better for Splunk performance)
    * **Disk:** 100GB+
    * **IP:** `192.168.10.10` (Static)
3.  **Target Workstation (`Target-PC`)**
    * **OS:** Windows 10
    * **Purpose:** Domain-joined client for testing and targeting.
    * **RAM:** 4GB (Install), 2GB+ (Runtime)
    * **Disk:** 50GB+
    * **IP:** `192.168.10.100` (Static Recommended)
4.  **Attacker Machine (`Kali`)**
    * **OS:** Kali Linux (Pre-built VirtualBox Image Recommended)
    * **Purpose:** Platform for launching attacks.
    * **RAM:** 2GB+
    * **Disk:** 20GB+ (Based on image)
    * **IP:** `192.168.10.250` (Static)

### Network Configuration

* **VirtualBox Network Type:** NAT Network (e.g., named `ad-project`)
* **Network Address:** `192.168.10.0/24`
* **Gateway:** Typically `192.168.10.1` (Managed by VirtualBox NAT Network)
* **DNS:**
    * Windows machines initially use `8.8.8.8` for internet access during setup.
    * **Crucially,** after AD DC promotion, domain-joined machines (`Target-PC`, potentially `addc01`) MUST point their DNS solely to the AD DC's IP (`192.168.10.7`).
    * Splunk Server & Kali can use `8.8.8.8` or `192.168.10.7` (once AD DC DNS is running).

### Diagram

It is highly recommended to create a visual diagram (e.g., using draw.io) representing the VMs, network connections, IP addresses, and log flow (endpoints -> Splunk).

## Prerequisites

### Hardware

* **Host Machine:** Minimum 16GB RAM, 250GB free disk space recommended. Less may cause performance issues.

### Software & Images

* Oracle VirtualBox installed.
* Windows Server 2022 Evaluation ISO (from Microsoft Evaluation Center).
* Windows 10 ISO (using Microsoft Media Creation Tool).
* Ubuntu Server 22.04.1 LTS ISO.
* Kali Linux VirtualBox Image (from kali.org).
* Splunk Enterprise Debian package (`.deb`) (requires free Splunk.com account).
* Splunk Universal Forwarder MSI installer (from Splunk.com).
* Sysmon (from Microsoft Sysinternals).
* Sysmon configuration file (e.g., Olaf Hartong's `sysmonconfig.xml` from GitHub).

### Knowledge

* Basic familiarity with VirtualBox, Windows/Linux command lines, and networking concepts.

## Installation & Setup Steps

**(Follow the MyDFIR video series for detailed visual guidance)**

1.  **Planning & Diagram:** Design your lab layout and IP scheme.
2.  **VirtualBox Setup:**
    * Install VirtualBox.
    * Create a **NAT Network** (File > Tools > Network Manager > NAT Networks):
        * Name: `ad-project` (or similar)
        * Network CIDR: `192.168.10.0/24`
        * Enable DHCP: Optional (we'll use static IPs mostly)
        * Supports IPv6: No (unless needed)
        * Enable Port Forwarding: Not required for this core setup.
3.  **VM Creation & OS Installation:**
    * Create VMs in VirtualBox for Server 2022, Ubuntu, and Windows 10 using their respective ISOs. Specify RAM/Disk/CPU.
        * **Windows Server:** Choose "Standard (Desktop Experience)". Set Administrator password.
        * **Ubuntu:** Create a user/password during setup. Install OpenSSH server (optional).
        * **Windows 10:** Standard install.
    * Import the Kali Linux VirtualBox image. Default credentials are `kali`/`kali`.
    * **Tip:** During VM creation, check "Skip Unattended Installation" if prompted, to avoid potential issues.
4.  **Initial Network Config & Updates:**
    * Power on all VMs.
    * Change the **Network Adapter** for **each VM** to attach to your created **NAT Network** (`ad-project`).
    * **Configure Static IPs:**
        * **Ubuntu (`Splunk`):** Edit `/etc/netplan/00-installer-config.yaml` (or similar) and run `sudo netplan apply`. Set IP `192.168.10.10`, Gateway `192.168.10.1`, DNS `8.8.8.8`.
        * **Windows Server (`addc01`):** Use Network Adapter properties. Set IP `192.168.10.7`, Mask `255.255.255.0`, Gateway `192.168.10.1`, DNS `8.8.8.8` (temporarily). Rename computer to `addc01`.
        * **Kali:** Use network settings GUI or `nmtui`. Set IP `192.168.10.250`, Mask `255.255.255.0`, Gateway `192.168.10.1`, DNS `8.8.8.8`.
        * **Windows 10 (`Target-PC`):** Use Network Adapter properties. Set IP `192.168.10.100`, Mask `255.255.255.0`, Gateway `192.168.10.1`, DNS `8.8.8.8` (temporarily). Rename computer to `Target-PC`.
    * **Update Systems:**
        * Ubuntu/Kali: `sudo apt update && sudo apt upgrade -y`
        * Windows: Run Windows Update.
    * **Install Guest Additions:** Install on Ubuntu (`sudo apt install virtualbox-guest-utils`) and Windows VMs for better integration (clipboard, shared folders). Create a shared folder in VirtualBox settings to easily transfer files (e.g., Splunk installer) to Ubuntu. Add your Ubuntu user to the `vboxsf` group (`sudo adduser $USER vboxsf`) and reboot Ubuntu. Mount the shared folder.
5.  **Splunk Server Setup (`Splunk` VM):**
    * Transfer the Splunk Enterprise `.deb` file to the Ubuntu VM (e.g., via shared folder).
    * Install Splunk: `sudo dpkg -i /path/to/splunk-installer.deb`
    * Start Splunk & Accept License: `sudo /opt/splunk/bin/splunk start --accept-license`. Create an admin user/password when prompted.
    * Enable Splunk to start on boot: `sudo /opt/splunk/bin/splunk enable boot-start -user splunk`
    * Access Splunk Web UI: `http://192.168.10.10:8000`
    * **Configure Receiving:** Settings > Forwarding and Receiving > Configure receiving > New Receiving Port. Enter `9997`. Save.
    * **Create Index:** Settings > Indexes > New Index.
        * Name: `endpoint`
        * Index Data Type: Events
        * Keep default settings for now. Save.
6.  **Endpoint Configuration (Repeat for `Target-PC` and `addc01`):**
    * **Install Sysmon:**
        * Transfer `Sysmon64.exe` and `sysmonconfig.xml` to the Windows VM.
        * Open **Admin PowerShell/CMD**: `.\Sysmon64.exe -accepteula -i sysmonconfig.xml`
    * **Install Splunk Universal Forwarder:**
        * Run the MSI installer.
        * Accept license.
        * Customize Options: Set **Deployment Server** blank for now. Set **Receiving Indexer** to `192.168.10.10:9997`.
        * Choose **Local System** user during installation.
    * **Configure Log Forwarding (`inputs.conf`):**
        * Create/Edit the file: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`
        * **Important:** Do NOT edit files in the `default` directory.
        * Add the following content:
          ```ini
         [WinEventLog://Application]
         index = endpoint
         disabled = false

         [WinEventLog://Security]
         index = endpoint
         disabled = false

         [WinEventLog://System]
         index = endpoint
         disabled = false

         [WinEventLog://Microsoft-Windows-Sysmon/Operational]
         index = endpoint
         disabled = false
         renderXml = true
         source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
          ```
    * **Restart Splunk Forwarder Service:** Open `services.msc`, find `SplunkForwarder Service`, and **Restart** it. Ensure it's running as **Local System**.
    * **Verify Logs in Splunk:** Go to Splunk Search & Reporting app (`http://192.168.10.10:8000`). Search `index=endpoint host=<hostname>`. You should see logs arriving after a minute or two. Repeat for the other Windows host.
7.  **Active Directory Setup (`addc01` VM):**
    * Open Server Manager. Manage > Add Roles and Features.
    * Select Role-based installation. Select the server.
    * Check **Active Directory Domain Services**. Add required features. Complete installation.
    * Click the notification flag in Server Manager, select **Promote this server to a domain controller**.
    * Select **Add a new forest**. Enter a **Root domain name:** `mydfir.local` (or your choice).
    * Set Forest/Domain functional levels (keep defaults if unsure). Enter a DSRM password.
    * Complete the wizard (ignore DNS delegation warning for lab). The server will restart.
    * Log in using the domain administrator account (e.g., `MYDFIR\Administrator`).
    * Open **Active Directory Users and Computers** (Server Manager > Tools).
    * Create Organizational Units (OUs) (e.g., `IT`, `HR`, `Servers`, `Workstations`).
    * Create domain users (e.g., `JSmith`, `TSmith`) within OUs. Set passwords (uncheck "User must change password..." for lab ease).
8.  **Domain Join (`Target-PC` VM):**
    * **CRITICAL:** Change the `Target-PC`'s network adapter **DNS settings** to point **ONLY** to the AD DC IP: `192.168.10.7`. Remove `8.8.8.8`.
    * Verify connectivity: `ping mydfir.local` (should resolve to `192.168.10.7`).
    * Go to System Properties > Computer Name tab > Change... > Member of: **Domain**.
    * Enter the domain name: `mydfir.local`.
    * Provide domain admin credentials (e.g., `mydfir\Administrator` and password) when prompted.
    * Restart the `Target-PC` when prompted.
    * Log in using a domain account (e.g., `MYDFIR\JSmith` or select the domain and type `JSmith`).
9.  **Kali Setup (`Kali` VM):**
    * Ensure static IP (`192.168.10.250`) is set and networking works.
    * Update/Upgrade: `sudo apt update && sudo apt upgrade -y`
    * Install Crowbar: `sudo apt install -y crowbar`
    * Prepare password list:
        * `sudo gunzip /usr/share/wordlists/rockyou.txt.gz`
        * Create a smaller test list (e.g., `password.txt`) on your Desktop: `head -n 50 /usr/share/wordlists/rockyou.txt > ~/Desktop/password.txt`
        * **Add the actual password** for one of your test domain users (e.g., TSmith) to `~/Desktop/password.txt`.
10. **Take Snapshots!** Now is a great time to snapshot all VMs in VirtualBox to save a clean, configured state.

## Usage: Attack & Detect Scenarios

**Disclaimer:** Only perform these actions within your isolated lab environment. Do not target systems you do not own or have explicit permission to test.

### 1. RDP Brute Force Attack

* **On `Target-PC`:**
    * Enable Remote Desktop (System Properties > Remote tab > Allow remote connections).
    * Ensure the users you want to target (e.g., `MYDFIR\TSmith`) are in the Remote Desktop Users group or Administrators group.
* **On `Kali`:**
    * Open terminal.
    * Run Crowbar:
        ```bash
        crowbar -b rdp -s 192.168.10.100/32 -u TSmith -C ~/Desktop/password.txt
        ```
        (Replace user `TSmith` and IP if needed).
    * Observe output for successful password discovery.
* **Detection (`Splunk`):**
    * Search: `index=endpoint host="Target-PC" (EventCode=4625 OR EventCode=4624)`
    * Look for numerous EventCode 4625 (Failed Logon) from the Kali IP (`192.168.10.250`).
    * Identify EventCode 4624 (Successful Logon) from the Kali IP for the targeted user. Analyze logon type (should be 10 for RDP) and timestamps.

### 2. Atomic Red Team Simulation

* **On `Target-PC`:**
    * Open **Admin PowerShell**.
    * Bypass Execution Policy for current user: `Set-ExecutionPolicy Bypass -Scope CurrentUser -Force`
    * **Crucial:** Add **Defender Exclusion** for `C:\` to prevent Atomic files from being quarantined (Windows Security > Virus & threat protection > Manage settings > Add or remove exclusions > Add an exclusion > Folder > `C:\`). *Note: This significantly reduces security and is for lab purposes only.*
    * Install Atomic Red Team framework (Command usually found in MyDFIR video description or Atomic Red Team docs):
        ```powershell
        IEX (IWR '[https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1](https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1)' -UseBasicParsing); Install-AtomicRedTeam -getAtomics
        ```
* **On `Target-PC` (Admin PowerShell):**
    * Navigate to the atomics directory: `cd C:\AtomicRedTeam\atomics\`
    * List tests for a technique: `Invoke-AtomicTest T1059.001 -ShowDetailsBrief` (PowerShell)
    * Run a specific test: `Invoke-AtomicTest T1059.001 -TestNumbers 1`
    * Try another test: `Invoke-AtomicTest T1136.001 -TestNumbers 1` (Create Account: Local)
* **Detection (`Splunk`):**
    * Wait a few minutes for logs to ingest.
    * Search for indicators related to the test:
        * `index=endpoint host="Target-PC" "*invoke-expression*"`
        * `index=endpoint host="Target-PC" EventCode=4688 "powershell.exe"` (Look for suspicious command lines)
        * `index=endpoint host="Target-PC" EventCode=4720` (A user account was created - related to T1136.001)
        * Search for specific commands or strings used in the Atomic tests.
    * Correlate findings with the MITRE ATT&CK framework (Technique IDs T1059.001, T1136.001, etc.).

## Troubleshooting

* **VM Network Issues:** Check IPs/Masks/Gateway. Ensure VMs are on the same VirtualBox NAT Network. Ping between VMs (Windows Firewall might block ping by default, test other connectivity like RDP or Splunk forwarder connection). If DHCP fails, set a static IP.
* **Splunk Forwarding:**
    * Verify network connectivity from Windows host to Splunk server (`192.168.10.10` on port `9997`). Use PowerShell: `Test-NetConnection 192.168.10.10 -Port 9997`.
    * Check `inputs.conf` path (`...\etc\system\local\`) and syntax.
    * Ensure Splunk Forwarder Service is **running** and using **Local System account**.
    * **Restart** Splunk Forwarder Service after any `inputs.conf` changes.
* **Splunk Receiving:**
    * Ensure the `endpoint` index exists in Splunk (Settings > Indexes).
    * Verify Splunk is listening on port `9997` (Settings > Forwarding and Receiving > Configure receiving).
    * Restart Splunk service on Ubuntu: `sudo /opt/splunk/bin/splunk restart`.
* **Domain Join Issues:**
    * **DNS is key!** Ensure the joining machine's ONLY DNS server is the AD DC IP (`192.168.10.7`).
    * Verify network connectivity to the DC (`ping mydfir.local`, `ping 192.168.10.7`).
    * Use the full domain name (`mydfir.local`).
    * Use correct domain admin credentials.
* **Crowbar Fails:** Ensure RDP is enabled on the target and the target user is allowed to RDP. Double-check the target IP and username. Make sure the correct password IS in the password list file (`password.txt`).
* **Atomic Red Team Issues:** Ensure Defender exclusion is properly set for `C:\`. Allow time for logs to appear in Splunk. Some tests might be blocked even with exclusions.

## Key Learning Outcomes & Next Steps

* Practical understanding of AD DS installation and basic administration.
* Experience configuring Sysmon and Splunk Universal Forwarder for enhanced logging.
* Hands-on experience with Splunk for data ingestion and searching.
* Understanding of the red team attack lifecycle (recon, execution) and corresponding blue team detection using SIEM.
* Familiarity with tools like Crowbar and frameworks like Atomic Red Team & MITRE ATT&CK.
* Develop troubleshooting skills for network, OS, and application issues.

**Next Steps:**

* **Use Snapshots:** Leverage VirtualBox snapshots to revert to clean states before testing new attacks or configurations.
* **Explore Splunk:** Dive deeper into Splunk Search Processing Language (SPL), create alerts, build dashboards, and install security-focused Splunk apps (e.g., Splunk Security Essentials). Check out Splunk's free training.
* **Explore MITRE ATT&CK:** Research different techniques and try simulating them with Atomic Red Team or other tools. Map your Splunk searches to ATT&CK techniques.
* **Expand the Lab:** Add more clients, implement Group Policies, set up a Certificate Authority, add vulnerability scanning, or integrate other security tools.
* **Document Your Work:** Keep notes on your setup, commands used, and findings. Consider adding this project to your GitHub portfolio.

Good luck, and have fun exploring!
