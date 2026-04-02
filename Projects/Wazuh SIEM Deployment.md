
#### Wazuh Deployment Goals:
- Deploy a self hosted enterprise-grade SIEM
- Monitor and collect logs from Virtual Machines and external devices LAN Devices
- Practice system administration, device hardening, compliance, etc.
- Understand the SIEM environment for transferable experience

#### Current Home Lab Architecture:
- **Host Machine:** Windows 10 Desktop PC
- **Virtualization Software:** VMware Workstation Pro (Free Version)
- **Server :** Ubuntu Server 24.04 VM
- **Endpoints:** Windows 10 VM, Kali Linux VM, Chromebook (Running Debian Velvet-Os

### Technical Steps to Complete Goal:
#### 1) Installing the Ubuntu Server Virtual Machine:

1. Opened VMware Workstation Pro, and created a new virtual machine
2. Configured the virtual machine to have:
    - 8 Gigs Ram
    - 4 Processing Cores
    - 100 Gig HDD
3. Originally created two network adapters, the first being configured for Network Address Translation and the second being Host-Only. (This becomes an issue later)
4. Selected the Ubuntu Server ISO and launched the Virtual Machine and updated the server's dependencies

#### 2) Installing Wazuh:

(The Wazuh Documentation was my best friend)

1. Downloaded the Wazuh installation assistant with: `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh`
2. Ran the Wazuh installation assistant
3. Opened Brave and connected to the Wazuh Dashboard through the server's IP address

#### 3) Installing the Windows 10 VM Agent:

1. Followed the same process as downloading and creating the Ubuntu Server VM
2. Configured the Windows 10 Virtual Machine to have:
    - 4 Gigs of Ram
    - 4 Processing Cores
    - 80 Gig HDD
3. Configured the Windows 10 VM to have two network adapters, one for NAT and the other for Host-Only. Allowing for internet access and direct communication with the host and other Virtual Machines on the host.
4. Updated the system's dependencies
5. Using the Agent deployment section of the Wazuh Dashboard I filled out the information pertaining to the Agent name, server IP address, etc. and copied the installation command with the correct information.
6. Ran the installation script
7. Started the Wazuh service using: `Start-Service wazuhsvc`
8. Refreshed Wazuh Dashboard and it appeared as an active agent

#### 4) Installing the Kali Linux VM Agent:

1. Followed the same process as downloading and creating the Ubuntu Server VM and Windows 10 VM
2. Configured the Kali Linux Virtual Machine to have:
    - 2 Gigs of Ram
    - 4 Processing Cores
    - 80 Gig HDD
3. Configured the network settings the same as the Windows 10 Virtual Machine
4. Updated the system's dependencies
5. Again, using the Agent deployment section of the Wazuh Dashboard I created the installation script
6. Ran the installation script in the Kali terminal
7. Ran the following commands to enable and start the Wazuh-agent: `sudo systemctl daemon-reload` sudo systemctl enable wazuh-agent systemctl start wazuh-agent`
8. The Wazuh-agent is now successfully installed

#### 5) Adding External Chromebook Wazuh-Agent:

1. After booting Velvet-Os on the Chromebook s330 via USB I did the same process of creating a new agent to deploy tailored to the Chromebook
2. I typed the script into the Velvet-Os terminal and it installed correctly
3. After running the script I had to manually start the Wazuh-agent because ChromeOs's firmware doesn't support any service managers with: `/var/ossec/bin/wazuh-control start`
4. After this I checked the Wazuh Dashboard and the Chromebook's agent was not registering, this is when I realized the network settings of the Ubuntu Server weren't allowing the Chromebook to reach the server.
5. The Ubuntu Server configured with NAT means that the server is only reachable by the host windows machine, not by LAN devices, which my Chromebook is
6. I tried switching the network settings on the Ubuntu Server VM to Bridged - Automatic, which resulted in DHCP not working, so the Ubuntu Server only had a Loopback Address, this also resulted in me messing up the Server IP of the deployed agents
7. I then tried forcing the bridged connection to my physical Wi-Fi card (Intel(r) Dual Band Wireless-AC 3168) via running the Virtual Network Editor with Admin privileges.
8. This worked and the other Virtual Machines and Chromebooks were able to successfully connect again after changing the Server IP Address in their configuration files to the new IP Address of the bridged server. I did have to restart each agent also to enforce the configuration change.

#### Some of the Obstacles I faced:

1. Ubuntu Server running Wazuh Manager originally unreachable from Chromebook
2. Switching Ubuntu Server settings broke deployed Agents due to Server IP change
3. Windows Wazuh agent did not install cleanly 1. `sc query WazuhSvc` would return nothing 2. `WazuhSvc` would return service name invalid (Due to incomplete/corrupted install from earlier attempts)
4. Ubuntu Server only showed loopback address after switching network adapter to bridged - automatic
5. Sleep deprivation (7 hours over a 48 hour + period)
6. Other random installation errors and hardware misconfigurations

### End Result:

1. Seamless integration between Wazuh-agents on VMs and External Hardware
2. Knowledge of SIEM installation and deployment processes
3. Knowledge of Virtual Machine networking and configuration processes and their interactions with actual LAN environments
4. Knowledge navigating GitHub Repos
5. Skills following official documentation and troubleshooting methods