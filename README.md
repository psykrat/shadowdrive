# Shadowdrive
Your silent guardian in the shadows. Always there. Never seen.

#### **Requirements:**
1. Raspberry Pi Zero W/WH (because you will need wireless capability)
2. MicroSD card (preferably 16GB or more)
3. PiSugar Battery or a similar portable power source
4. USB drive or external HDD/SSD for storage (optional if your SD card is large enough)
5. Raspbian OS (or Raspberry Pi OS as it's now known)
6. A computer to access the Samba share

#### **Steps:**

1. **Setup the Raspberry Pi Zero:**
   - Flash the Raspberry Pi OS onto the MicroSD card using a tool like [Raspberry Pi Imager](https://www.raspberrypi.org/software/).
   - Insert the MicroSD card into the Raspberry Pi Zero.

2. **Setting up Pi as an Access Point:**
   - Boot up your Raspberry Pi and connect to it either directly via keyboard/mouse/display or SSH into it.
   - Update your system: 
     ```
     sudo apt update && sudo apt upgrade
     ```
   - Install the necessary packages:
     ```
     sudo apt install dnsmasq hostapd
     ```
   - Stop the new services:
     ```
     sudo systemctl stop dnsmasq
     sudo systemctl stop hostapd
     ```
   - Configure static IP for the wlan0 interface by editing the `dhcpcd` configuration:
     ```
     sudo nano /etc/dhcpcd.conf
     ```
     Add the following lines:
     ```
     interface wlan0
     static ip_address=192.168.4.1/24
     nohook wpa_supplicant
     ```
     Save and close the file.

   - Restart `dhcpcd`:
     ```
     sudo service dhcpcd restart
     ```
   - Configure `dnsmasq`:
     ```
     sudo nano /etc/dnsmasq.conf
     ```
     Add these lines:
     ```
     interface=wlan0
     dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
     ```
   - Set up `hostapd`:
     ```
     sudo nano /etc/hostapd/hostapd.conf
     ```
     Add these lines:
     ```
     interface=wlan0
     driver=nl80211
     ssid=YourNetworkName
     hw_mode=g
     channel=7
     wmm_enabled=0
     macaddr_acl=0
     auth_algs=1
     ignore_broadcast_ssid=0
     wpa=2
     wpa_passphrase=YourPassword
     wpa_key_mgmt=WPA-PSK
     wpa_pairwise=TKIP
     rsn_pairwise=CCMP
     ```
   - Point `hostapd` to the config file:
     ```
     sudo nano /etc/default/hostapd
     ```
     Find the line `#DAEMON_CONF=""` and replace it with:
     ```
     DAEMON_CONF="/etc/hostapd/hostapd.conf"
     ```

   - Start `hostapd` and `dnsmasq`:
     ```
     sudo systemctl start hostapd
     sudo systemctl start dnsmasq
     ```

3. **Setting up Samba:**
   - Install Samba:
     ```
     sudo apt install samba samba-common-bin
     ```
   - Once installed, backup the original Samba configuration file and create a new one:
     ```
     sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.original
     sudo nano /etc/samba/smb.conf
     ```
   - Add a new share definition to this file, for example:
     ```
     [PiDrive]
     path = /media/pi/your_drive_name
     writeable = yes
     guest ok = no
     create mask = 0777
     directory mask = 0777
     ```
   - Secure your Samba server by adding a password:
     ```
     sudo smbpasswd -a pi
     ```

4. **Encrypting the USB drive:**
   - Install `cryptsetup`:
     ```
     sudo apt install cryptsetup
     ```
   - Set up encryption on your drive:
     ```
     sudo cryptsetup luksFormat /dev/sda1
     ```
     This will erase all data on the drive. Make sure to back up any important data beforehand.
   - Unlock the drive to use it:
     ```
     sudo cryptsetup luksOpen /dev/sda1 my_encrypted_drive
     ```
   - To auto-mount it on boot, you'll need to add it to `/etc/fstab`. This can be tricky, so make sure to follow a thorough guide specifically on this step if you want it.

4. **Encrypting an Internal Storage Partition (Instead of using and encrypting an external storage device for a more portable option.):**

   - #### 1. **Partition the SD Card:**

      **Note:** You'll need to repartition your SD card, which can be dangerous if done incorrectly. Always back up your data before performing these steps.

    - First, check the current partition layout with:
      ```bash
      sudo fdisk -l
      ```
    
    Look for your SD card, usually named `/dev/mmcblk0`. Note down the partition names and sizes.
    
    - Start the partition tool:
      ```bash
      sudo fdisk /dev/mmcblk0
      ```
    
    - Delete a partition if necessary (be careful with this!). You can use the `d` command in `fdisk` to delete and then the `n` command to create a new partition. Ensure you leave enough space for the OS, and the remaining can be used for storage.
    
    - Write the changes and exit by pressing `w`.
    
    - Reboot the Raspberry Pi for the changes to take effect.
    
    #### 2. **Set Up Encryption on the New Partition:**
    
    - Install the necessary encryption tools:
      ```bash
      sudo apt install cryptsetup
      ```
    
    - Initialize LUKS encryption on the new partition. Replace `/dev/mmcblk0pX` with your specific partition number:
      ```bash
      sudo cryptsetup luksFormat /dev/mmcblk0pX
      ```
    You'll be warned that this will erase all data on the partition. Confirm and set a strong passphrase when prompted.
    
    #### 3. **Open and Map the Encrypted Partition:**
    
    - Open the encrypted partition, creating a mapped device named `encrypted_storage`:
      ```bash
      sudo cryptsetup luksOpen /dev/mmcblk0pX encrypted_storage
      ```
    
    - Check that the mapped device has been created:
      ```bash
      ls /dev/mapper/
      ```
    
    You should see `encrypted_storage` in the list.
    
    #### 4. **Format the Mapped Device:**
    
    - Create a filesystem on the mapped device:
      ```bash
      sudo mkfs.ext4 /dev/mapper/encrypted_storage
      ```
    
    #### 5. **Mounting the Encrypted Partition:**
    
    - Create a mount point:
      ```bash
      sudo mkdir /media/encrypted_storage
      ```
    
    - Mount the encrypted storage:
      ```bash
      sudo mount /dev/mapper/encrypted_storage /media/encrypted_storage
      ```
    
    #### 6. **Configure Auto-Mounting:**
    
    - First, we need the UUID of the encrypted partition. Get it with:
      ```bash
      sudo blkid | grep mmcblk0pX
      ```
    Copy the UUID value (e.g., UUID="1234-abcd-...").
    
    - Open the `/etc/crypttab` file to add an entry for the encrypted partition:
      ```bash
      sudo nano /etc/crypttab
      ```
    
    - Add the following line:
      ```
      encrypted_storage UUID=YOUR_UUID none luks
      ```
    
    Replace `YOUR_UUID` with the actual UUID value you got from the `blkid` command.
    
    - Add an entry in `/etc/fstab` to mount the mapped device on boot:
      ```bash
      sudo nano /etc/fstab
      ```
    
    - Add the following line:
      ```
      /dev/mapper/encrypted_storage /media/encrypted_storage ext4 defaults 0 2
      ```
    
    #### 7. **Reboot:**
    
    Finally, reboot your Raspberry Pi. The encrypted partition should automatically be decrypted and mounted on boot, prompting you for the passphrase.
    
    ---
    
    Remember, if you ever forget the encryption passphrase, the data on the encrypted partition will be lost, so be sure to store the passphrase securely.

5. **Accessing your Personal Cloud:**
   - From a computer, search for available networks and connect to the SSID you set up earlier (`YourNetworkName`).
   - Once connected, you should be able to access the Samba share using the Pi's static IP address (e.g., `\\192.168.4.1\PiDrive` on Windows or `smb://192.168.4.1/PiDrive` on macOS/Linux).

6. **If you want to create a human readable name instead of accessing via the IP then follow the steps below:**

    ### **Windows:**
    
    1. Open Notepad as Administrator.
    2. Go to `File` -> `Open` and navigate to `C:\Windows\System32\drivers\etc\`.
    3. Change the file type dropdown from `Text Documents` to `All Files`.
    4. Open the `hosts` file.
    5. Add the IP address and desired hostname at the end of the file:
       ```
       192.168.4.1 shadowdrive
       ```
    6. Save and close Notepad.
    7. You can now access the Samba share with `\\shadowdrive\PiDrive`.
    
    ### **macOS/Linux:**
    
    1. Open Terminal.
    2. Type `sudo nano /etc/hosts` and press enter.
    3. Enter your password when prompted.
    4. Scroll to the bottom and add the following line:
       ```
       192.168.4.1 shadowdrive
       ```
    5. Press `CTRL + O` to save and `CTRL + X` to exit.
    6. You can now access the Samba share with `smb://shadowdrive/PiDrive`.

    ---
   
    Remember, this change is local to the device where you modified the hosts file. If you want other devices on your network to also use this human-readable hostname, you'd need to edit the hosts file on each of those devices or set up a local DNS server to handle the resolution.

**Security Note:** Ensure that you use strong, unique passwords and regularly backup your data. While this setup provides a level of privacy since it's not connected to the internet, physical access to the device will still pose a potential risk, even with encryption, if someone has enough time and resources.

You are now in the shadow realm.
