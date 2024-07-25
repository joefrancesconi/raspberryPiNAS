
# Creating an NAS with a RaspberryPi

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Downloading Ubuntu Server](#downloading-ubuntu-server)
3. [Writing the Image to the SD Card](#writing-the-image-to-the-sd-card)
4. [Initial Setup](#initial-setup)
5. [Network Setup](#network-setup)
6. [Post-Installation Configuration](#post-installation-configuration)
7. [Setting up a hard drive for NAS storage](#Setting-up-a-hard-drive-for-NAS-storage)
8. [Setting up Samba for filesharing](#Setting-up-Samba-for-filesharing)
9. [Encrypting the home directory](#Encrypting-the-home-directory)
10. [Connecting to the NAS from Windows](#Connecting-to-the-NAS-from-Windows)

## Prerequisites

Before you start, ensure you have the following:

- A Raspberry Pi (preferably a Raspberry Pi 3, 4, or later).
- A microSD card (at least 8GB recommended).
- A computer with an SD card reader.
- A stable internet connection and ethernet cable.
- An HDMI cable and monitor
- A keyboard
- A storage device such as a thumb drive or external hard drive

## Downloading Ubuntu Server

1. Visit the [official Ubuntu website](https://ubuntu.com/download/raspberry-pi) and download the latest version of Ubuntu Server for Raspberry Pi.
2. Choose the correct version for your Raspberry Pi model (e.g., 32-bit or 64-bit).

## Writing the Image to the SD Card

1. Download and install a tool to write the image to the SD card, such as [Balena Etcher](https://www.balena.io/etcher/).
2. Insert the microSD card into your computer’s SD card reader.
3. Open Balena Etcher (or your chosen tool).
4. Select the Ubuntu Server image file you downloaded.
5. Select the microSD card as the target.
6. Click "Flash" to begin writing the image to the SD card.
7. Once the process is complete, safely eject the microSD card from your computer.

## Initial Setup

1. Insert the microSD card into your Raspberry Pi.
2. Plug in your keyboard and storage device to the Raspberry Pi.
3. Connect the Raspberry Pi to your network using an Ethernet cable.
4. Power on the Raspberry Pi by connecting it to a power source.

### Network Setup

1. Configure your network and to use a static IP:
    ```sh
   sudo nano /etc/netplan/50-cloud-init.yaml
   ```
   ```yaml
   network:
    version: 2
    renderer: networkd
    ethernets:
      eth0:
        dhcp4: false
        dhcp6: false
        addresses:
        - 192.168.1.166/24
        routes:
        - to: default
          via: 192.168.1.1
        nameservers:
          addresses: [8.8.8.8,8.8.4.4]
   ```
   Please replace 102.168.1.166/24 with your prefered IP and network mask and the 192.168.1.1 with your gateway address.
2. Then apply with:
   ```sh
   sudo netplan apply
   ```

## Post-Installation Configuration

1. Update the system:
   ```sh
   sudo apt update
   sudo apt upgrade -y
   ```
2. Install any additional packages you need:
   ```sh
   sudo apt install <package_name>
   ```
3. Configure the hostname:
   ```sh
   sudo hostnamectl set-hostname <new_hostname>
   ```
4. Set up any other services or configurations as needed.

## Setting up a hard drive for NAS storage

1. Create and format a partition on your storage device. If you are going to be connecting to the NAS with a windows machine, you should format you partition with ntfs.
   For more information on how to create and format partitions consult this [video](https://youtu.be/1UpZzqH6qeA?si=pb0qduVSSzw1nX9R).

2. Set your mount point.
   Get the UUID of your harddrive with
   ```sh
   sudo blkid
   ```
   Record the UUID of the hard drives you wish to mount.
   Next edit /etc/fstab and add the following lines at the bottom.
   ```sh
   sudo nano /etc/fstab
   ```
   ```sh
   # syntax
   # UUID="YOUR-UID-HERE" /mnt/ntfs/ ntfs nls-utf8,umask-0222,uid-1000,gid-1000,ro 0 0
   UUID="438A7D041A7D234C" /mnt/nas-hdd-1/ ntfs nls-utf8,umask-0222,uid-1000,gid-1000,ro 0 0
   ```
   **Make sure that your mount point is inside your user directory. This will be important when we encrypt the home directory later.**
   
## Setting up Samba for filesharing

1. Install Samba
   ```sh
   sudo apt install build-essential ssh samba samba-common samba-common-bin ntfs-3g fuse2fs
   ```
2. Check that Samba is installed and running properly
   ```sh
   sudo systemctl status smbd
   ```
   Look in the response to say that it is active.
3. Configure Samba shares
   ```sh
   sudo nano /etc/samba/smb.conf
   ```
   At the top under Authentification, add the line:
   ```sh
   smb encrypt = mandatory
   ```
   Next scroll to the very bottom of the file and add the following configuration, alter as needed to match your environment and file names.
   ```sh
   [nas-hd-1]
   comment = Nas Share
   # Below is the path to the share. In this case its the whole NAS Hard drive
   path = /home/<your username>/mnt/nas-hdd-1 
   guest ok = no
   browseable = yes
   create mask = 0777
   directory mask = 0777
   writable = yes
   read only = no
   write list = root, @lpadmin
   ```
   If you have more hard drives you need to repeat this step for each one.
   When you are done, save and exit.

4. Setting up Samba Users
   Now we need to add the current user to the samba group, set a password and allow it through the firewall.
   Run the following commands in terminal.
   ```sh
   sudo usermod -aG sambashare $USER

   sudo smbpasswd -a $USER

   sudo ufw allow samba

   sudo systemctl restart smbd
   ```

## Encrypting the home directory

1. Install eCryptfs
   ```sh
   sudo apt install ecryptfs-utils cryptsetup
   ```
2. Create a tmp admin account
   ```sh
   sudo addusr temp_user
   ```
   Add user to sudo group:
   ```sh
   sudo usermod -aG sudo temp_user
   ```
3. Log into the new admin account
   
4. Encrypt you main accounts home folder
   ```sh
   sudo ecryptfs-migrate-home -u <username>
   ```
5. Log back into the encrypted user account
   You can recover you randomly generated passphrase with
   ```sh
   ecryptfs-unwrap-passphrase
   ```
6. Clean up - Delete the temp account
   ```sh
   sudo userdel --remove temp_user
   ```
   During the process, a home folder backup was also created.
   You can remove this by:
   ```sh
   ls /home/
   sudo rm -rf /home/username.xxxxx
   ```

## Connecting to the NAS from Windows

1. On windows, open file explorer.
   
2. On the left pannel, right click the network category and then select Map Network Drive as shown:
   ![image](https://github.com/user-attachments/assets/1f87f885-d92a-4aa5-bad0-dad58703ba36)
   
4. Select a drive letter and put in the network path to your NAS share folder.
   ![image](https://github.com/user-attachments/assets/6d93ef39-1cba-41d1-876b-8d6aaca1d03e)
   
   You can click browse to find the folder you defined in the samba config file.

6. Click OK, then Finish.
   
7. Windows will now ask you for the username and password for your samba user that you created.
   Input those and now you should be connected to your NAS!



