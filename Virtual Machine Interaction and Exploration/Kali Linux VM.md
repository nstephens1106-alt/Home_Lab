**README:** 
- This Kali Linux VM was created using VMware Workstation Pro as a core tool for exploring different offensive security techniques within my HomeLab environment. The sections below document different activities I performed.

### Remote Accessing Metasploitable 2 VM

1. I utilized nmap to scan the Metasploitable VM's ports, then used a variety of techniques to gain remote access as root into the Metasploitable VM.

	1a. After scanning with nmap I discovered ports 512, 513, and 514 were open, which are known as R-services or Remote Services. Utilizing the _rlogin_ command on the IP address of the Metasploitable VM I was able to gain root access.
    
    2a. After scanning the IP Address of the Metasploitable 2 VM again, I noticed that port 23 was open by default. Telnet runs on port 23, and Telnet is a unencrypted remote login protocol used for remote system administration. I then connected to the Metasploitable VM with _telnet 192.186.100.131_ to gain access.