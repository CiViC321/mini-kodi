# Minimal Kodi Installation

Instructions for installing and configuring a minimal Kodi media center machine, to fill the gap left after the XBMC/Kodi-buntu distributions were retired. 

The Ubuntu-Desktop distribution is just too bloated for a "setup-once and forget about it" machine.


## Install a Minimal Ubuntu 18 Dist.
The goal is to install only what's absolutely necessary for kodi operation. 

[Ubuntu MinimalCD](https://help.ubuntu.com/community/Installation/MinimalCD) was chosen initially as it allows us to specify packages or package groups manually during installation but it doesn't support EFI out of the box and requires an active internet connection during installation as the CD doesn't contain any.

[Ubuntu Server](https://www.ubuntu.com/download/server) seemed like the next logical choice. It has very minimal defaults, this should allow us to only add the packages we need.

### 1. Download ISO

Download the ISO that matches your architecture (ex 64-bit) from [Ubuntu](https://www.ubuntu.com/download/server).
I find the torrent option is usually the fastest.

Note: Chose the LTS version for longest lifetime.
​    
### 2. Boot from ISO

Burn the ISO to a DVD or use a tool like [UNetBootIn](https://unetbootin.github.io/) or [Rufus](https://rufus.akeo.ie/) to create a boot USB.
​    
### 3. Go through the installation

  - Select Language Option(s)

  - Select Keyboard Layout

  - Select `Install Ubuntu`

  - Use DHCP assigned IP or configure your own IP

  - Proxy if you need it

  - Select Ubuntu Mirror Options

  - Filesystem setup

    Use `Use an entire disk` unless you want to manually set it up

  - Profile Setup

    Note: Something logical (kodi, media-pc, etc...) that doesn't conflic with existing hosts on LAN.

    | Entry              | Value |
    | ------------------ | ----- |
    | Your name          | kodi  |
    | Your server's name | kodi  |
    | Pick a username    | kodi  |
    | Choose a password  | ***   |


  - Skip the Snaps unless you need one
  
  - Finished Reboot Now



## Ubuntu 18 OS Configuration

Some basic operating system configuration and fixup before installing kodi.

### Fix sudo

If your hostname can't be resolved by DNS (or you have a duplicate hostname on the network), sudo will hang for up to a minute, this can get aggravating fast.

Fix it by adding the hostname to `/etc/hosts`

```bash
sudo sed -i "/^127.0.0.1/ s/$/ $(hostname)/" /etc/hosts
```

### Remove cloud-init

Seems Ubuntu is assuming we're all *in the cloud* and introduced (jammed down our throat) a *"a set of **python** scripts and utilities to make your cloud images be all they can be!"*

How hard would it be to add this as an option during installation?

To remove this we'll need to

- Remove all cloud-init datasources

  ```bash
  echo 'datasource_list: [ None ]' | sudo -s tee /etc/cloud/cloud.cfg.d/90_dpkg.cfg
  ```

- Uninstall all it's packages

  ```bash
  sudo apt-get purge cloud-init
  ```

- Delete all it's configuration files

  ```bash
  sudo rm -rf /etc/cloud/; sudo rm -rf /var/lib/cloud/
  ```

- Remove open-iscsi

  The iSCSI daemon will wait indefinitely and hang the boot process if it's not removed when cloud-init is removed

  ```bash
  sudo apt remove open-iscsi
  ```


### Update OS

Install the latest patches for the currently installed packages.

- Update apt and initiate an upgrade 

  ```bash
  sudo bash -c 'apt update && apt upgrade -y'
  ```

- Reboot

  ```bash
  sudo reboot
  ```


### Install xubuntu-core

The xbuntu-core is a lightweight desktop environment, provides us with the least amount of packages needed for a functioning X11 (graphical) environment to support kodi.

- Enable [Universe Repository](https://help.ubuntu.com/community/Repositories/Ubuntu#The_Four_Main_Repositories)

    ```bash
    sudo add-apt-repository universe
    ```
- Install xubuntu-core
    Note: The carat in the command below is needed, it [specifies a tasksel task](https://help.ubuntu.com/community/Tasksel#Usage_.28alternative.29).

    ```bash
    sudo apt install xubuntu-core^
    ```

### Configure Time

Configure time synchronization with chrony and set the timezone - it might not seem like a big deal but it's good practice and will save you from a possible headache later.

- Install chrony

  ```bash
  sudo apt install chrony
  ```

- Get a list of timezones

  ```bash
  timedatectl list-timezones
  ```

- Set your timezone (ex. Chicago)

  ```bash
  sudo timedatectl set-timezone 'America/Chicago'
  ```

### Create Users & Groups

We'll create separate users and a common group for all the services (kodi, sonarr, radarr, sabnzbd) to allow them to share access to the media store while still maintaining process separation.

- Common Group
  - Create group

    ```bash
    sudo groupadd media
    ```

  - Add kodi to the group

    ```bash
    sudo usermod --append --groups media ${USER}
    ```

- **Users**

  - SABnzbd+

    ```bash
    sudo useradd                     \
     --comment  "SABnzbd+ Daemon"    \
     --shell    /usr/sbin/nologin    \
     --home-dir /var/opt/sabnzbdplus \
     --groups   media                \
     --system   --user-group         \
     sabnzbd 
    ```

  - Sonarr

    ```bash
    sudo useradd                  \
     --comment  "Sonarr Daemon"   \
     --shell    /usr/sbin/nologin \
     --home-dir /opt/Sonarr       \
     --groups   media             \
     --system   --user-group      \
     sonarr
    ```

  - Radarr

    ```bash
    sudo useradd                  \
     --comment  "Radarr Daemon"   \
     --shell    /usr/sbin/nologin \
     --home-dir /opt/Ronarr       \
     --groups   media             \
     --system   --user-group      \
     radarr
    ```

  - Plex Media Server

    ```bash
    sudo useradd                         \
     --comment  "Plex User"              \
     --shell    /usr/sbin/nologin        \
     --home-dir /var/lib/plexmediaserver \
     --groups   media                    \
     --system   --user-group             \
     plex
    ```


### Clone mini-kodi Repo

The repository contains various config files and scripts that will be used throughout. We'll need to clone it to ensure we have the latest files available to us.

- Clone the repo

  ```bash
  cd $HOME; git clone https://github.com/OnceUponALoop/mini-kodi.git
  ```

- Create a runtime copy of the configuration files

  ```bash
  cp -r $HOME/mini-kodi/conf $HOME/mini-kodi/runconf
  ```

- Replace user entries with desired users

  Change the usernames below to match the desired usernames

  - Kodi

    ```bash
    find $HOME/mini-kodi/runconf  -type f \
    | xargs -I {} sed -i 's/KODI_USER/kodi/g; s/KODI_GROUP/kodi/g' {}
    ```

  - Common media group

    ```bash
    find $HOME/mini-kodi/runconf  -type f \
    | xargs -I {} sed -i 's/COMMON_GROUP/media/g' {}
    ```

  - SABnzbd+

    ```bash
    find $HOME/mini-kodi/runconf  -type f \
    | xargs -I {} sed -i 's/SABN_USER/sabnzbd/g; s/SABN_GROUP/sabnzbd/g' {}
    ```

  - Sonarr

    ```bash
    find $HOME/mini-kodi/runconf  -type f \
    | xargs -I {} sed -i 's/SONR_USER/sonarr/g; s/SONR_GROUP/sonarr/g' {}
    ```

  - Radarr

    ```bash
    find $HOME/mini-kodi/runconf  -type f \
    | xargs -I {} sed -i 's/RADR_USER/radarr/g; s/RADR_GROUP/radarr/g' {}
    ```

  - Plex Media Server

    ```bash
    find $HOME/mini-kodi/runconf  -type f \
    | xargs -I {} sed -i 's/PLEX_USER/plex/g; s/PLEX_GROUP/plex/g' {}
    ```


## Install Kodi

- Add Kodi PPA Repository

  Set the right PPA depending on which release of Kodi you want. I recommend sticking with stable unless you have a good reason not to.

  - Stable

    ```bash
    sudo add-apt-repository ppa:team-xbmc/ppa
    ```

  - Nightly

    ```bash
    sudo add-apt-repository ppa:team-xbmc/unstable
    ```

- Update apt and install kodi

  Note we're also installing the inputstream addons, these are required by various plugins but require installation through apt.

  ```bash
  sudo bash -c 'apt update && apt install kodi kodi-inputstream-adaptive kodi-inputstream-rtmp'
  ```

- Configure lightdm to start kodi on bootup

  Create file `/etc/lightdm/lightdm.conf.d/kodi.conf`

  ```
  sudo cp $HOME/mini-kodi/runconf/lightdm-kodi.conf /etc/lightdm/lightdm.conf.d/kodi.conf
  ```

- Test it by launching lightdm
  ```bash
  sudo systemctl start lightdm
  ```
  If it has already started, test it by killing lightdm, it should restart with Kodi
  ```bash
  sudo killall lightdm
  ```

## Configure Remote

I spent a bunch of time over the past 8 years massaging lirc into behaving correctly, it didn't help that my IR receiver of choice was always an OG xbox adapter.

I've finally given up and taken the easy route - I bought a newfangled [flirc](https://flirc.tv) and will be using that with this configuration. I plan on maintaining config files and instructions for the various remotes i encounter, might be helpful to other people out there.



## Install SABNzbd

Reference: [SABnzbd Documentation](https://sabnzbd.org/wiki/installation/install-ubuntu-repo)

- Enable [Multiverse Repository](https://help.ubuntu.com/community/Repositories/Ubuntu#The_Four_Main_Repositories)

  ```bash
  sudo add-apt-repository multiverse
  ```

- Install sabnzbd repositories

  ``` bash
  sudo add-apt-repository ppa:jcfp/nobetas
  sudo add-apt-repository ppa:jcfp/sab-addons
  ```

- Install sabnzbd and dependencies
  ```bash
  sudo apt update && sudo apt install sabnzbdplus software-properties-common python-sabyenc par2-tbb
  ```

- Enable auto-startup
  ```bash
  sudo systemctl enable sabnzbdplus
  ```

- Replace the configuration file `/etc/default/sabnzbdplus`
  ```bash
  sudo cp $HOME/runconf/systemd/sabnzbd-dafaults.init /etc/default/sabnzbdplus
  ```

- Start it
  ```bash
  sudo systemctl start sabnzbdplus
  ```

## Install Sonarr

Reference: [Mono](https://www.mono-project.com/download/stable/#download-lin) & [Sonarr](https://github.com/Sonarr/Sonarr/wiki/Installation#linux) documentation.

- Install Mono Repositories

  ```bash
  sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
  echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
  ```

- Install Sonarr Repository
  ```bash
  sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 0xa236c58f409091a18aca53cbebff6b99d9b78493
  echo "deb http://apt.sonarr.tv/ master main" | sudo tee /etc/apt/sources.list.d/sonarr.list
  ```

- Install Sonarr

  ```bash
  sudo apt update && sudo apt install nzbdrone
  ```

- Fix Permissions
  The default radarr package file permissions need to be adjusted (Windows developers perhaps)

  ```bash
  sudo find /opt/NzbDrone -type f -exec chmod 644 {} \;
  sudo chown sonarr:sonarr -R /opt/NzbDrone
  ```

- Create a systemd service for Sonarr

  **Note:** The user and pathing in the service file might need to be adjusted to match your system/user info.

  ```bash
  sudo cp $HOME/mini-kodi/runconf/systemd/sonarr.service /etc/systemd/system/
  ```

- Enable and start it
  ```bash
  sudo systemctl enable sonarr.service
  sudo systemctl start  sonarr.service
  ```

- Change the port

  Sonarr defaults to port 8089 if available. change the `Port` directive in the configuration file `/var/opt/sonarr/config.xml` to adjust the port.

  ```xml
  <Config>
    <LogLevel>Info</LogLevel>
    <Port>8082</Port>
    <UrlBase></UrlBase>
    <BindAddress>*</BindAddress>
    <SslPort>9898</SslPort>
    <EnableSsl>False</EnableSsl>
    <ApiKey>-------------------------</ApiKey>
    <AuthenticationMethod>None</AuthenticationMethod>
    <LaunchBrowser>False</LaunchBrowser>
  </Config>
  ```

- Restart the service to use the new port

  ```
  sudo systemctl restart sonarr.service
  ```

## Install Radarr

There's still no repo packages available for radarr so we'll have to install it manually. This isn't too bad especially since it auto-updates anyway.

- Install dependencies

  ```bash
  sudo apt install libmono-cil-dev curl mediainfo
  ```

- Install Radarr

  ```bash
  cd /opt
  sudo wget $( curl -s https://api.github.com/repos/Radarr/Radarr/releases | grep linux.tar.gz | grep browser_download_url | head -1 | cut -d \" -f 4 )
  sudo tar -xvzf Radarr.*.linux.tar.gz
  sudo rm -f /opt/Radarr.*.linux.tar.gz
  ```

- Fix Permissions
  The default radarr package file permissions need to be adjusted (Windows developers perhaps)

  ```bash
  sudo find /opt/Radarr -type f -exec chmod 644 {} \;
  sudo chown radarr:radarr -R /opt/radarr
  ```

- Create a systemd service for Sonarr

  **Note:** The user and pathing in the service file might need to be adjusted to match your system/user info.

  ```bash
  sudo cp $HOME/mini-kodi/runconf/systemd/radarr.service /etc/systemd/system/
  ```

- Enable and start it

  ```bash
  sudo systemctl enable radarr.service
  sudo systemctl start  radarr.service
  ```

- Change the port

  Radarr defaults to port 7878 if available. change the `Port` directive in the configuration file `/var/opt/Radarr/config.xml` to adjust the port.
  ``` xml
  <Config>
    <LogLevel>Info</LogLevel>
    <Port>8083</Port>
    <UrlBase></UrlBase>
    <BindAddress>*</BindAddress>
    <SslPort>9898</SslPort>
    <EnableSsl>False</EnableSsl>
    <ApiKey>--------------------------------</ApiKey>
    <AuthenticationMethod>None</AuthenticationMethod>
    <Branch>develop</Branch>
    <LaunchBrowser>False</LaunchBrowser>
  </Config>
  ```

- Restart the service to use the new port

  ```bash
  sudo systemctl restart radarr.service
  ```

## Install Plex Media Server

- Trust Plex repository key

  ```bash
  curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
  ```

- Add Plex repository

  ```bash
  echo deb https://downloads.plex.tv/repo/deb public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
  ```

- Install Plex Media Server

  ```bash
  sudo bash -c 'apt update && apt install plexmediaserver'
  ```

  Ignore the installer prompt to replace the apt repo file.


## OS Customization

- **Audio Configuration**

  If you're using an NVidia card or an NVidia ION box (ex Zotac Zbox) you'll soon realize that there's no audio in Kodi. 

  This is due to the fact that NVidia presents the wrong default device for audio.
  Reference: [NVidia HDMI Audio (Under-Enumeration of Logical Devices)](http://http.download.nvidia.com/XFree86/gpu-hdmi-audio-document/index.html#under_enumeration_of_logical_devices) & [ArchiWiki](https://wiki.archlinux.org/index.php/PulseAudio/Examples#HDMI_output_configuration)

  - List all audio devices
    ```
    kodi@kodi:~$ sudo aplay -l
    **** List of PLAYBACK Hardware Devices ****
    Home directory not accessible: Permission denied
    card 0: Intel [HDA Intel], device 0: ALC888 Analog [ALC888 Analog]
      Subdevices: 1/1
      Subdevice #0: subdevice #0
    card 0: Intel [HDA Intel], device 1: ALC888 Digital [ALC888 Digital]
      Subdevices: 1/1
      Subdevice #0: subdevice #0
    card 1: NVidia [HDA NVidia], device 3: HDMI 0 [HDMI 0]
      Subdevices: 1/1
      Subdevice #0: subdevice #0
    card 1: NVidia [HDA NVidia], device 7: HDMI 0 [HDMI 0]
      Subdevices: 1/1
      Subdevice #0: subdevice #0
    card 1: NVidia [HDA NVidia], device 8: HDMI 0 [HDMI 0]
      Subdevices: 1/1
      Subdevice #0: subdevice #0
    card 1: NVidia [HDA NVidia], device 9: HDMI 0 [HDMI 0]
      Subdevices: 1/1
      Subdevice #0: subdevice #0
    ```
    
  - Get a list of all the devices with their `hw` name
    ```
    # sudo aplay -l | grep card | awk -F"card |device |:" '{print "hw:" $2 "," $4}'
    hw:0,0
    hw:0,1
    hw:1,3
    hw:1,7
    hw:1,8
    hw:1,9
    ```
    
  - Test each device to figure out which one is actually producing audio
    ```
    sudo speaker-test -c 2 -r 48000 -D <HW-ID>
    ```
    
  - Once we determine which device is our audio device we can reconfigure pulseaudio to load it
    ```
    sudo echo "load-module module-alsa-sink device=<HW-ID>" >> /etc/pulse/default.pa
    ```
    
    The Zotac ZBOX ID40 uses hw:1,7
    ```
    sudo echo "load-module module-alsa-sink device=hw:1,7" >> /etc/pulse/default.pa
    ```
    
  - Reboot to apply changes and test
    Restarting pulse and Kodi should be enough but I haven't tested that.

- **Enable Kodi Power Control**

  TODO make a file in project

  By default Kodi lacks permissions to Suspend/Wake/Shutdown/Poweroff.
  Reference: [Kodi Wiki](http://kodi.wiki/view/HOW-TO:Suspend_and_wake_in_Ubuntu)

  - Install dependencies, they should all already be installed but better safe than sorry!
    ```bash
    sudo apt install policykit-1 upower acpi-support
    ```

  - Create file `/var/lib/polkit-1/localauthority/50-local.d/kodi.pkla` 

    ```bash
    sudo cp $HOME/mini-kodi/runconf/polkit-kodi.pkla /var/lib/polkit-1/localauthority/50-local.d/kodi.pkla
    ```

  - Restart polkit to apply the settings
    ```
    sudo systemctl restart polkit
    ```

- **Enable USB automount**

  Configure it to use the USB label as the mount name.
  - Create file `/etc/udev/rules.d/11-media-by-label-auto-mount.rules` with the following content
    ```
    # Start at sdb to avoid system hard drive
    KERNEL!="sd[b-z][0-9]", GOTO="media_by_label_auto_mount_end"
     
    # Import FS info
    IMPORT{program}="/sbin/blkid -o udev -p %N"
     
    # Get a label if present, otherwise specify one
    ENV{ID_FS_LABEL}!="", ENV{dir_name}="%E{ID_FS_LABEL}"
    ENV{ID_FS_LABEL}=="", ENV{dir_name}="usbhd-%k"
     
    # Global mount options
    ACTION=="add", ENV{mount_options}="realtime"
     
    # Filesystem-specific mount options
    ACTION=="add", ENV{ID_FS_TYPE}=="vfat|ntfs",
    ENV{mount_options}="$env{mount_options},utf8,gid=100,umask=002"
     
    # Mount the device
    ACTION=="add", RUN+="/bin/mkdir -p /media/%E{dir_name}", RUN+="/bin/mount -o $env{mount_options} /dev/%k /media/%E{dir_name}"
     
    # Clean up after removal
    ACTION=="remove", ENV{dir_name}!="", RUN+="/bin/umount -l /media/%E{dir_name}", RUN+="/bin/rmdir /media/%E{dir_name}"
     
    # Exit
    LABEL="media_by_label_auto_mount_end"
    ```

- **Hide grub menu**

  We can hide the grub menu while still retaining the ability to use it by using the HIDDEN options
  - Create a new file `/etc/default/grub.d/99-kodi-splash.cfg`
    ```bash
    sudo cp $HOME/mini-kodi/runconf/grub-kodi-splash.cfg /etc/default/grub.d/99-kodi-splash.cfg
    ```

  - Recreate it to apply settings
    ```
    sudo update-grub
    ```

- **Install Kodi Plymouth Theme**

  Plymouth is a Fedora project that enables graphical diplay during bootup. We're going to change the default theme to a Kodi one to make bootup look nicer.
  - Install dependencies
    ```bash
    sudo apt install fakeroot
    ```
  - Clone the git repo to a temp location
    ```bash
    git clone https://github.com/solbero/plymouth-theme-kodi-animated-logo.git /tmp/plymouth-theme-kodi
    ```
  - Navigate to repo and build the deb package
    ```bash
    cd /tmp/plymouth-theme-kodi
    ./build.sh
    ```

  - Install the package
    ```bash
    sudo dpkg -i /tmp/plymouth-theme-kodi/plymouth-theme-kodi-animated-logo.deb
    ```

  - Update grub
    ```bash
    sudo update-grub
    ```

  - Delete the source as we no longer need it
    ```bash
    cd ~/; rm -rf /tmp/plymouth-theme-kodi
    ```

  - Reboot to test
    ```
    sudo reboot
    ```

- **Disable WiFi**

  If you're not using the wireless interface then you might want to disable it.

  The best way to ensure it's completely disabled is to block loading of the wireless driver using.



  - Find the driver used in your wireless card

    ```bash
    sudo lshw -C network | grep -B 1 -A 12 'Wireless interface'
    ```

    Look for the driver=<drivername> (ex driver=iwlwifi)



  - Create the file `/etc/modprobe.d/disable-wifi.conf`

    ```bash
    # Disable Wifi Driver
    install iwlwifi /bin/true
    ```

    Reference: [Blacklisting a module](https://wiki.archlinux.org/index.php/Kernel_module#Using_files_in_/etc/modprobe.d/)

- **Enable Auto-Update**

  - Reference: [Ubuntu AutomaticSecurityUpdates](https://help.ubuntu.com/community/AutomaticSecurityUpdates)
  - Edit `/etc/apt/apt.conf.d/50unattended-upgrades`
  - Enable `"${distro_id}:${distro_codename}-updates";`
  - Add kodi PPA `"LP-PPA-team-xbmc:${distro_codename}";`
  - Add SabNzbd `PPA "LP-PPA-jcfp-sab-addons:${distro_codename}";`
  - Enable cleanup `Unattended-Upgrade::Remove-Unused-Dependencies "true";`
  - Enable Reboot `Unattended-Upgrade::Automatic-Reboot "true";`
  - Set Reboot time to 4am `Unattended-Upgrade::Automatic-Reboot-Time "04:00";`

  Final result should look something like this
  ```c++
  // Automatically upgrade packages from these (origin:archive) pairs
  Unattended-Upgrade::Allowed-Origins {
          "${distro_id}:${distro_codename}";
          "${distro_id}:${distro_codename}-security";
          // Extended Security Maintenance; doesn't necessarily exist for
          // every release and this system may not have it installed, but if
          // available, the policy for updates is such that unattended-upgrades
          // should also install from here by default.
          "${distro_id}ESM:${distro_codename}";
          "${distro_id}:${distro_codename}-updates";
  //      "${distro_id}:${distro_codename}-proposed";
  //      "${distro_id}:${distro_codename}-backports";
          "LP-PPA-team-xbmc:${distro_codename}";
          "LP-PPA-jcfp-sab-addons:${distro_codename}";
  };
  
  // List of packages to not update (regexp are supported)
  Unattended-Upgrade::Package-Blacklist {
  //      "vim";
  //      "libc6";
  //      "libc6-dev";
  //      "libc6-i686";
  };
  
  // This option allows you to control if on a unclean dpkg exit
  // unattended-upgrades will automatically run
  //   dpkg --force-confold --configure -a
  // The default is true, to ensure updates keep getting installed
  //Unattended-Upgrade::AutoFixInterruptedDpkg "false";
  
  // Split the upgrade into the smallest possible chunks so that
  // they can be interrupted with SIGUSR1. This makes the upgrade
  // a bit slower but it has the benefit that shutdown while a upgrade
  // is running is possible (with a small delay)
  //Unattended-Upgrade::MinimalSteps "true";
  
  // Install all unattended-upgrades when the machine is shuting down
  // instead of doing it in the background while the machine is running
  // This will (obviously) make shutdown slower
  //Unattended-Upgrade::InstallOnShutdown "true";
  
  // Send email to this address for problems or packages upgrades
  // If empty or unset then no email is sent, make sure that you
  // have a working mail setup on your system. A package that provides
  // 'mailx' must be installed. E.g. "user@example.com"
  //Unattended-Upgrade::Mail "root";
  
  // Set this value to "true" to get emails only on errors. Default
  // is to always send a mail if Unattended-Upgrade::Mail is set
  //Unattended-Upgrade::MailOnlyOnError "true";
  
  // Do automatic removal of new unused dependencies after the upgrade
  // (equivalent to apt-get autoremove)
  Unattended-Upgrade::Remove-Unused-Dependencies "true";
  
  // Automatically reboot *WITHOUT CONFIRMATION*
  //  if the file /var/run/reboot-required is found after the upgrade
  Unattended-Upgrade::Automatic-Reboot "true";
  
  // If automatic reboot is enabled and needed, reboot at the specific
  // time instead of immediately
  //  Default: "now"
  Unattended-Upgrade::Automatic-Reboot-Time "04:00";
  
  // Use apt bandwidth limit feature, this example limits the download
  // speed to 70kb/sec
  //Acquire::http::Dl-Limit "70";
  ```
