# PXE Boot: Configuring a PXE Server  

## **Why is PXE Boot Needed?**  
Preboot Execution Environment (PXE) boot is essential for network-based OS installation. It allows computers to boot over a network without needing physical installation media (USB/DVD). PXE is widely used in enterprise environments, data centers, and cloud infrastructure for rapid and automated OS deployment.  

### **Key Reasons for Using PXE:**  
✔ **Automated Deployment**: Enables large-scale OS installations without manual intervention.  
✔ **No Physical Media Required**: Eliminates the need for USB/DVDs, reducing logistics overhead.  
✔ **Remote Installation**: Useful for environments where physical access to machines is limited.  
✔ **Consistent Configuration**: Ensures uniform OS deployment with predefined configurations.  
✔ **Time and Cost Efficiency**: Reduces setup time and minimizes IT maintenance efforts.  

---

## **Benefits of PXE Boot**  
🔹 **Centralized OS Deployment**: A single PXE server can install OS on multiple client machines.  
🔹 **Scalability**: Supports large-scale environments like data centers and corporate networks.  
🔹 **Fast and Reliable**: Reduces the risk of manual errors and speeds up system provisioning.  
🔹 **Supports Various OSs**: Can be used to deploy Linux, Windows, or other OS images.  
🔹 **Custom Configuration**: Can integrate with Kickstart (Linux) or Unattended Setup (Windows) for automated installation.  

---

## **Use Cases of PXE Boot**  
📌 **Enterprise IT Deployment**: Automating OS installations in large organizations.  
📌 **Data Centers**: Setting up thousands of servers quickly with minimal effort.  
📌 **Cloud Environments**: Provisioning virtual machines and containers at scale.  
📌 **Disaster Recovery**: Quickly restoring OS environments in case of system failures.  
📌 **Educational Institutions**: Deploying OS across multiple lab computers efficiently.  

---

## **Implementation: Setting Up a PXE Server on CentOS 7**  

### **Step 1: Install Required Packages**  
```bash
yum -y install syslinux xinetd tftp-server
```
✔ `syslinux`: Provides PXE bootloader (`pxelinux.0`).  
✔ `xinetd`: Manages network services like TFTP.  
✔ `tftp-server`: Transfers boot files to PXE clients.  

---

### **Step 2: Configure DHCP Server**  
Edit the DHCP configuration file:  
```bash
vi /etc/dhcp/dhcpd.conf
```
Add the following configuration:  
```bash
option domain-name "master";
default-lease-time 600;
max-lease-time 7200;

subnet 192.168.100.0 netmask 255.255.255.0 {
    range 192.168.100.101 192.168.100.120;
    option routers 192.168.100.7;
    filename "pxelinux.0";
    next-server 192.168.100.7;
}
```
✔ `filename "pxelinux.0";` → Specifies the bootloader file.  
✔ `next-server 192.168.100.7;` → Defines the PXE server's IP address.  

**Restart DHCP Service**  
```bash
systemctl restart dhcpd.service
```

---

### **Step 3: Configure TFTP Server**  
Modify the TFTP configuration file:  
```bash
vi /etc/xinetd.d/tftp
```
Ensure the following settings:  
```bash
{
    socket_type     = dgram
    protocol        = udp
    wait            = yes
    user            = root
    server          = /usr/sbin/in.tftpd
    server_args     = -s /var/lib/tftpboot
    disable         = no
    per_source      = 11
    cps             = 100 2
    flags           = IPv4
}
```
**Start and Enable TFTP Service**  
```bash
systemctl start xinetd
systemctl enable xinetd
systemctl status xinetd
```

---

### **Step 4: Prepare PXE Boot Files**  
Create required directories and copy PXE bootloader files:  
```bash
mkdir -p /var/lib/tftpboot/pxelinux.cfg
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot
cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot
```

---

### **Step 5: Mount and Copy OS Installation Files**  
Mount the CentOS 7 installation ISO/CD:  
```bash
mount -t auto -o loop /dev/cdrom /var/pxe/centos7
```
Copy PXE boot files:  
```bash
cp /var/pxe/centos/images/pxeboot/initrd.img /var/lib/tftpboot/centos7/
cp /var/pxe/centos/images/pxeboot/vmlinuz /var/lib/tftpboot/centos7/
```

---

### **Step 6: Create PXE Boot Menu**  
Edit the PXE boot configuration file:  
```bash
vi /var/lib/tftpboot/pxelinux.cfg/default
```
Add the following configuration:  
```bash
timeout 100
default menu.c32
menu title ########## PXE BOOT MENU ##########

label 1
    menu label ^1) Install CentOS 7
    kernel centos7/vmlinuz
    append initrd=centos7/initrd.img method=http://192.168.100.7/centos7 devfs=nomount

label 2
    menu label ^2) Boot from local drive
    localboot 0
```

---

### **Step 7: Configure HTTP Server for OS Installation**  
Install Apache (`httpd`) to serve OS installation files:  
```bash
yum install httpd
systemctl start httpd
systemctl enable httpd
```
Configure HTTP alias for CentOS installation files:  
```bash
vi /etc/httpd/conf.d/pxeboot.conf
```
Add:  
```bash
Alias /centos7 /var/pxe/centos7
<Directory /var/pxe/centos7>
    Options Indexes FollowSymLinks
    Require ip 127.0.0.1 192.168.100.0/24
</Directory>
```
**Restart Apache**  
```bash
systemctl restart httpd
```

---

### **Step 8: Configure Kickstart for Automated Installation**  
Create a Kickstart configuration file:  
```bash
mkdir /var/www/html/ks
vi /var/www/html/ks/centos7-ks.cfg
```
Example Kickstart Configuration:  
```bash
install
autostep
reboot
auth --enableshadow --passalgo=sha512
url --url=http://192.168.100.7/centos7/
ignoredisk --only-use=sda
lang en_US.UTF-8
keyboard --vckeymap=us
network --bootproto=dhcp --activate
rootpw --iscrypted $6$EC1T.oKN5f3seb20$y1WlMQ...
timezone America/New_York --isUtc
bootloader --location=mbr --boot-drive=sda
zerombr
clearpart --all --initlabel
part /boot --fstype="xfs" --size=500
part pv.10 --fstype="lvmpv" --ondisk=sda --size=51200
volgroup VolGroup --pesize=4096 pv.10
logvol / --fstype="xfs" --size=20480 --name=root --vgname=VolGroup
logvol swap --fstype="swap" --size=4096 --name=swap --vgname=VolGroup
%packages
@core
%end
```
Change permissions:  
```bash
chmod 644 /var/www/html/ks/centos7-ks.cfg
```

---

## **Final Steps: Booting the PXE Client**  
1️⃣ Set the PXE client to boot from network in BIOS.  
2️⃣ The client will request an IP from the DHCP server.  
3️⃣ It will download the PXE bootloader from the TFTP server.  
4️⃣ The PXE boot menu appears, allowing OS installation.  
5️⃣ If Kickstart is configured, installation proceeds automatically.  

---

<br>
<br>




# **PXE Boot: Configuring a PXE Client**  

## **What is a PXE Client?**  
A PXE client is a computer configured to boot using the network instead of local storage (HDD/SSD). The client retrieves the OS installation files from a PXE server via **TFTP** and starts the installation process.  

### **How PXE Boot Works for Clients**  
1️⃣ **PXE Client Requests an IP** → The client machine sends a DHCP request to obtain an IP address and PXE boot parameters.  
2️⃣ **PXE Server Responds** → The server provides a bootloader (`pxelinux.0`) and installation files via **TFTP**.  
3️⃣ **Client Loads the Boot Menu** → The client displays a menu for selecting an OS installation or booting from local storage.  
4️⃣ **OS Installation Begins** → If a Kickstart file is available, the installation proceeds automatically.  

---

## **Setting Up the PXE Client**  

### **Step 1: Configure the Client's BIOS/UEFI**  
1. **Turn on the client machine** and enter BIOS/UEFI settings (`F2`, `F12`, `DEL`, or `ESC` based on manufacturer).  
2. Navigate to **Boot Order / Boot Options**.  
3. Enable **PXE Boot (Network Boot)** and set it as the first boot device.  
4. Save changes and reboot.  

---

### **Step 2: Network Boot the PXE Client**  
Once the PXE client starts:  
🔹 The **DHCP server** assigns an IP address and provides boot options.  
🔹 The client downloads the **PXE bootloader (pxelinux.0)** from the TFTP server.  
🔹 The PXE **boot menu** appears, allowing OS selection or local boot.  

---

### **Step 3: Selecting OS Installation via PXE**  
On the PXE boot menu, choose:  
✔ **Install CentOS 7** → Installs CentOS from the PXE server.  
✔ **Boot from Local Drive** → Skips PXE boot and loads the OS from the local HDD/SSD.  

---

### **Step 4: Automatic Installation via Kickstart (Optional)**  
If the PXE server has a **Kickstart file** (`ks.cfg`), the installation is fully automated:  
1️⃣ The client downloads the OS installation files via HTTP.  
2️⃣ Kickstart automatically partitions the disk, sets up users, and installs necessary packages.  
3️⃣ The system installs and reboots without manual intervention.  

---

## **Troubleshooting PXE Boot Issues**  
📌 **PXE-E53: No Boot Filename Received**  
✔ Ensure the DHCP server provides the correct `pxelinux.0` filename.  
✔ Verify `next-server` in DHCP configuration points to the PXE server.  

📌 **PXE-E32: TFTP Open Timeout**  
✔ Check if the **TFTP server** is running:  
```bash
systemctl status xinetd
```
✔ Ensure the firewall allows TFTP:  
```bash
firewall-cmd --add-service=tftp --permanent
firewall-cmd --reload
```

📌 **Boot Menu Not Appearing?**  
✔ Verify `/var/lib/tftpboot/pxelinux.cfg/default` is correctly configured.  

📌 **Client Fails to Load Installation Files**  
✔ Ensure the HTTP server is running:  
```bash
systemctl status httpd
```
✔ Check if the installation files are accessible:  
```bash
curl http://192.168.100.7/centos7/
```

<br>


## **Summary**  
PXE Boot simplifies OS installation in large-scale environments. It eliminates the need for manual installations, reduces setup time, and ensures consistency across multiple systems. With PXE and Kickstart, organizations can automate deployment processes efficiently. 🚀





<br>
<br>
<br>

<br>
<br>
<br>


## Precise Implementation: Setting Up a PXE Server on CentOS 7



<br>
<br>




# PXE Server Setup Guide

## 1. Set Up the Server VM
### 1.1 Create the Server VM
- Create a new VM in your virtualization software (e.g., VirtualBox, VMware).
- Allocate resources (e.g., 2 CPUs, 2 GB RAM).
- Install a Linux distribution (e.g., Ubuntu Server) on the VM.

### 1.2 Configure Network Settings
- Set the network adapter to **Bridged Adapter** or **NAT with port forwarding**, ensuring communication with clients.
- Assign a **static IP address** to the server (e.g., `192.168.1.10`).

### 1.3 Update the System
```bash
sudo apt update && sudo apt upgrade -y
```
**Explanation:** Ensures the latest package list and updates installed packages.

## 2. Install Required Packages
```bash
sudo apt install isc-dhcp-server tftpd-hpa syslinux -y
```
**Explanation:** Installs necessary services:
- **isc-dhcp-server:** Assigns IP addresses to clients.
- **tftpd-hpa:** Serves boot files to clients.
- **syslinux:** Provides PXELINUX bootloader.

## 3. Configure the DHCP Server
### 3.1 Edit the DHCP Configuration File
```bash
sudo nano /etc/dhcp/dhcpd.conf
```
**Explanation:** Opens the DHCP configuration file for editing.

### 3.2 Add the Following Configuration:
```plaintext
allow booting;
allow bootp;

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;  # IP range for clients
    option broadcast-address 192.168.1.255;  # Broadcast address
    option routers 192.168.1.1;  # Default gateway
    option domain-name-servers 8.8.8.8;  # DNS server
    filename "pxelinux.0";  # Boot file
    next-server 192.168.1.10;  # PXE server IP
}
```
**Explanation:** Configures DHCP to assign IPs and provide PXE boot details.

### 3.3 Restart the DHCP Service
```bash
sudo systemctl restart isc-dhcp-server
```
**Explanation:** Applies the new configuration.

## 4. Configure the TFTP Server
### 4.1 Edit the TFTP Configuration File
```bash
sudo nano /etc/default/tftpd-hpa
```
**Explanation:** Opens the TFTP configuration file.

### 4.2 Update the Configuration:
```plaintext
RUN_DAEMON="yes"
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure"
```
**Explanation:** Configures TFTP to run as a daemon and serve boot files.

### 4.3 Create TFTP Boot Directory
```bash
sudo mkdir -p /var/lib/tftpboot/pxelinux.cfg
```
**Explanation:** Creates necessary directories for PXE boot files.

### 4.4 Copy PXE Bootloader
```bash
sudo cp /usr/lib/syslinux/pxelinux.0 /var/lib/tftpboot/
```
**Explanation:** Copies the PXELINUX bootloader to the TFTP directory.

## 5. Create PXE Configuration File
### 5.1 Create the Default Configuration File
```bash
sudo nano /var/lib/tftpboot/pxelinux.cfg/default
```
**Explanation:** Opens the PXE configuration file for editing.

### 5.2 Add the Following Configuration:
```plaintext
DEFAULT menu.c32
PROMPT 0
TIMEOUT 300
LABEL Linux
    KERNEL vmlinuz
    APPEND initrd=initrd.img root=/dev/nfs nfsroot=192.168.1.10:/nfsroot ip=dhcp rw
```
**Explanation:** Defines the default boot option and kernel parameters.

## 6. Restart the TFTP Service
```bash
sudo systemctl restart tftpd-hpa
```
**Explanation:** Applies the TFTP configuration.

---

# Client Configuration

## 7. Set Up the Client VMs
### 7.1 Create Client VMs
- Create two new VMs in your virtualization software.
- Allocate resources (e.g., 1 CPU, 1 GB RAM).
- Install a minimal Linux OS (e.g., Ubuntu Server).

### 7.2 Configure Network Settings
- Set the network adapter to **Bridged Adapter** or **NAT with port forwarding**.
- Enable **PXE Boot** in BIOS/UEFI and prioritize it over the hard drive.

## 8. Boot the Clients
### 8.1 Start the First Client VM
- Power on the VM.
- It will request an IP from the DHCP server and download PXE bootloader.

### 8.2 Monitor the Boot Process
- Check if the client successfully loads the PXE boot menu.
- If properly configured, the menu should display the boot options.

### 8.3 Select a Boot Option
- Use the keyboard to select a boot option.
- The client loads the specified kernel and `initrd.img`, starting the boot process.

### 8.4 Repeat for Second Client
- Power on the second client VM and follow the same steps.

## 9. Verify Network Boot
- Ensure both clients successfully boot and communicate with the PXE server.
- Check network settings to confirm they receive the correct IPs from DHCP.

---

## Summary
This guide provides step-by-step instructions to set up a PXE server using **Ubuntu**, **isc-dhcp-server**, **TFTP**, and **PXELINUX**, allowing multiple clients to boot over the network. By following these steps, you ensure that clients can seamlessly retrieve and boot OS images from the PXE server.







<br>
<br>
<br>












**👨‍💻 𝓒𝓻𝓪𝓯𝓽𝓮𝓭 𝓫𝔂**: [Suraj Kumar Choudhary](https://github.com/Surajkumar4-source) | 📩 **𝓕𝓮𝓮𝓵 𝓯𝓻𝓮𝓮 𝓽𝓸 𝓓𝓜 𝓯𝓸𝓻 𝓪𝓷𝔂 𝓱𝓮𝓵𝓹**: [csuraj982@gmail.com](mailto:csuraj982@gmail.com)





<br>
