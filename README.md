# Guide: How to Encrypt the Home Folder in Linux with a FIDO2 Device Using gocryptfs (Debian)

This guide is adapted from [Lukeus_Maximus's guide][1] on encrypting a home folder with gocryptfs on Arch Linux. However, I wanted to achieve *passwordless* encryption at login on Debian, which is not covered in their guide. A [GitHub issue](https://github.com/rfjakob/gocryptfs/issues/281) raised by [xelra](https://github.com/xelra) concluded that a static password on a Yubico device was sufficient for them, but this approach did not work for me. Therefore, I created this guide.

It is recommended to follow this guide after having installed Debian.

#### Install Dependencies
```bash
apt install gocryptfs fido2-tools libu2f-host0 libpam-mount
```

### 1. Creating the New Encrypted Filesystem. [source][1]
1. As root, create the directory `/home/$user.cipher`:
    ```bash
    user="your_username_here"
    mkdir /home/$user.cipher
    ```

2. Change its permissions, owner, and group to match those of the user's existing home directory:
    ```bash
    chown $user /home/$user.cipher
    chgrp $user /home/$user.cipher
    chmod 700 /home/$user.cipher
    ```

3. Now we need to determine what FIDO2 device we can use to create the filesystem: [source][2]
    ```bash
    fido2-token -L
    ```

4. Then set up the encrypted filesystem using gocryptfs on the `/home/$user.cipher` directory with the FIDO2 token device path:
    ```bash
    gocryptfs -fido2 /dev/hidraw* -init /home/$user.cipher
    ```
    Replace `/dev/hidraw*` with your FIDO2 device path, e.g., `/dev/hidraw2`. You may need to touch your token.

    > **Note:** Remember to store the master key somewhere secure. It can be used to recover the filesystem if the FIDO2 device is lost.

    If the user's home directory is empty, you can skip to configuring PAM for auto-mounting at login.

### 2. Move Existing Home Directory Files to the Encrypted Filesystem. [source][1]
You will need to mount the newly created encrypted filesystem and copy the entirety of the user's home directory into it. As each file is copied in, it is encrypted by gocryptfs. To avoid any programs changing files in the user's home directory in the middle of the copy operation, the user *must* be logged out. Then, *as root*: 

1. Create the user's new home directory:
    ```bash
    mv /home/$user /home/$user.old
    mkdir -m 700 /home/$user
    chown $user /home/$user
    chgrp $user /home/$user
    ```

2. Mount the encrypted filesystem at the new home directory using your FIDO2 token:  
    You may have to touch your token.
    ```bash
    gocryptfs -fido2 /dev/hidraw* /home/$user.cipher /home/$user
    ```

3. Copy all home directory files into the mounted filesystem using rsync:
    ```bash
    rsync -av /home/$user.old/ /home/$user
    ```
    Install rsync if necessary:
    ```bash
    apt install rsync
    ```

4. Unmount the filesystem:
    ```bash
    fusermount -u /home/$user
    ```

### 3. Configure PAM for Auto-Mounting at Login
If the user logs in without their home directory mounted, their session will not benefit from any shell profile files or any programs configured to run at login. Until the directory is mounted with `gocryptfs`, the user's home directory will be empty. Generally then, it is highly desirable to have the encrypted home directory mount itself when the user logs in so that those things happen properly.

1. Configure FUSE:  
    Uncomment `user_allow_other` in `/etc/fuse.conf`:
    ```
    /etc/fuse.conf

    # The file /etc/fuse.conf allows for the following parameters:
    #
    # user_allow_other - Using the allow_other mount option works fine as root, in
    # order to have it work as user you need user_allow_other in /etc/fuse.conf as
    # well. (This option allows users to use the allow_other option.) You need
    # allow_other if you want users other than the owner to access a mounted fuse.
    # This option must appear on a line by itself. There is no value, just the
    # presence of the option.

    user_allow_other


    # mount_max = n - this option sets the maximum number of mounts.
    # Currently (2014) it must be typed exactly as shown
    # (with a single space before and after the equals sign).

    #mount_max = 1000
    ```

2. Configure `pam_mount.conf`: [source][1]  
    In `/etc/security/pam_mount.conf.xml`, add a new volume tag at the end of the file. Configure the logout and umount tags. Replace `your_username_here` with the actual username:
    ```xml
    ...

    <!-- requires ofl from hxtools to be present -->
    <logout wait="1000000" hup="no" term="yes" kill="yes" />
    <umount>umount -l %(MNTPT)</umount>

            <!-- pam_mount parameters: Volume-related -->

    <mkmountpoint enable="1" remove="true" />

    <volume
      user="your_username_here"
      fstype="gocryptfs"
      cipher="cipher"
      path="/home/%(USER).cipher"
      mountpoint="/home/%(USER)" />

    </pam_mount>
    ```

3. Configure pam_mount for mounting gocryptfs:
    Download the mounting script from GitHub:
    ```bash
    curl -o /usr/sbin/mount.gocryptfs https://raw.githubusercontent.com/Mikkel-Coder/gocryptfs_u2f_pammount/main/mount.gocryptfs
    ```
    Edit `/usr/sbin/mount.gocryptfs` to set the home directory and cipher paths:
    ```
    MOUNT_POINT="/home/your_username_here"
    ENCRYPTED_PATH="/home/your_username_here.cipher"
    ...
    ```
    And make it executable so that pam_mount can run it.
    ```bash
    chmod +x /usr/sbin/mount.gocryptfs
    ```

4. Configure PAM:  
    Edit `/etc/pam.d/common-auth` and append the arguments `disable_pam_password` and `disable_interactive` to `pam_mount.so`:
    ```
    ...

    # and here are more per-package modules (the "Additional" block)
    auth	optional			pam_mount.so  disable_pam_password disable_interactive
    # end of pam-auth-update config
    ```

    Make the same edit to `/etc/pam.d/common-session`:
    ```
    # and here are more per-package modules (the "Additional" block)
    session	required	pam_unix.so 
    session	optional	pam_mount.so disable_pam_password disable_interactive 
    session	optional	pam_systemd.so 
    # end of pam-auth-update config
    ```

    Logging in as the user will now mount the encrypted filesystem transparently. Logging out will unmount it.

    Remember to disable the root user if necessary.
    ```bash
    sudo usermod --expiredate 1 root
    ```

### 4 Securely Delete the Unencrypted Home Directory. [source][1]
If the unencrypted home folder was not empty, the user's home directory was then moved to *$user*.old earlier. These unencrypted files need removing securely; otherwise the encryption protecting the data can be easily avoided by just looking in the other folder. These older files need deleting securely as well - simply performing `rm -rf $user.old` will not remove the file from disk completely, it will just remove the reference to it.  

Multiple tools exist that claim to delete files securely (most notably shred) but these come into conflict with the journaling functions of jornaling filesystems (such as Ext4). The secure deletion tool is trying to make it so that you can't recover your files whilst the journaled filesystem is trying to make sure that you can recover them. 

### 5. (Optional) PAM authentication via u2f only [source][3]
One can go completely passwordless by using the `pam_u2f` module. Install it by:

1. Install `libpam-u2f`:
    ```bash
    sudo apt install libpam-u2f
    ```

2. Configure U2F keys:  
    Insert your U2F key and run the following command as the user:
    ```bash
    pamu2fcfg > u2f_keys
    ```
    You may have to touch your key.
    Add additional keys, such as backups keys by:
    ```bash
    pamu2fcfg -n >> u2f_keys
    ```

    As the users home directory is encrypted, the u2f_keys must be placed elsewhere such as `/etc/Yubico/`:
    ```bash
    sudo mv u2f_keys /etc/Yubico/
    ```
    > **Warning:** Please note that once you modify the /etc/pam.d/sudo file to require the YubiKey if you were to lose or misplace the YubiKey you will not be able to modify or change the file to remove the YubiKey requirement. [source][3]

    > **Warning:** By enabling using this process if the files are not readable by users it will cause you to be locked out of your system. The most common cause is encrypted /home/ folder which will not be readable by root. This will cause you to be locked out once you reset the machine. [source][3]

3. Configuring PAM  
    Edit `/etc/pam.d/common-auth` to use `pam_u2f.so` instead of `pam_unix.so`:
    ```
    ....
    # here are the per-package modules (the "Primary" block)
    auth	[success=1 default=ignore]	pam_u2f.so authfile=/etc/Yubico/u2f_keys cue 
    ....
    ```

    To force PIN usage, append `pinverification=1`:
    ```
    ....
    auth	[success=1 default=ignore]	pam_u2f.so authfile=/etc/Yubico/u2f_keys cue pinverification=1
    ....
    ```

    For more options, refer to the `pam_u2f` man page: `man pam_u2f`.

    You can add more authentication methods, such as fingerprints, by changing the order. Then update the `success`. It is used to tell pam to skip the next *X* `auth` modules if authentication was successful. For example:
    ```
    ...
    auth	[success=2 default=ignore]	pam_fprintd.so max-tries=1 timeout=10
    auth	[success=1 default=ignore]	pam_u2f.so authfile=/etc/Yubico/u2f_keys cue pinverification=1
    ...
    ``` 
    To require U2F authentication for changing user secrets, such as passwords for other users, edit `/etc/pam.d/common-passwd` to be the same as the *Primary* block from `/etc/pam.d/common-auth`.

[1]: https://wiki.archlinux.org/title/User:Lukeus_Maximus "Home directory encryption"
[2]: https://developers.yubico.com/libfido2/Manuals/fido2-token.html "Yubico Docs, fido2-token"
[3]: https://support.yubico.com/hc/en-us/articles/360016649099-Ubuntu-Linux-Login-Guide-U2F "Ubuntu Linux Login Guide - U2F"
