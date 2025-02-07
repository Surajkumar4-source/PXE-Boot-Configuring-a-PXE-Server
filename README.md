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
<br>


## **Conclusion**  
PXE Boot simplifies OS installation in large-scale environments. It eliminates the need for manual installations, reduces setup time, and ensures consistency across multiple systems. With PXE and Kickstart, organizations can automate deployment processes efficiently. 🚀
