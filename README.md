# Guide: How to encrypt home folder in Linux with a FIDO2 device with  gocryptfs (Debian)

Most of this guide is heavily  inspired from [Lukeus_Maximus's guide][1] on how to encrypted ones home folder with gocryptfs on Arch Linux, but I was not able to follow there guide as I wanted to do *passwordless* encryption at login. A [Github issus](https://github.com/rfjakob/gocryptfs/issues/281) opened by [xelra](https://github.com/xelra) discussed and concluded that a Yubicos *static password* was sufficent for them, but not for me! I have therefore made this guide.

It is recommended that this guide is followed when Debian have just been installed. 

#### Install dependencies
```
# apt install gocryptfs fido2-tools libu2f-host0 libpam-mount
```

### 1. Creating the new encrypted filesystem. [source][1]
1. As root, create the directory `/home/$user.cipher`:
    ```
    # user="your_username_here"
    # mkdir /home/$user.cipher
    ```

2. Change its permissions, owner, and group so that it matches those of the user's existing home directory: 
    ```
    # chown $user /home/$user.cipher
    # chgrp $user /home/$user.cipher
    # chmod 700 /home/$user.cipher
    ```

3. Now we need to determine what FIDO2 device we can use to create the filesystem: [source][2]
    ```
    # fido2-token -L
    ```

4. Then set up the encrypted filesystem using gocryptfs on the `/home/$user.cipher` directory using the FIDO2 token device path: 
    ```
    # gocryptfs -fido2 /dev/hidraw* -init /home/$user.cipher
    ```
    Replacing `/dev/hidraw*` with your FIDO2 device path like: `/dev/hidraw2` You may have to touch your token. 

    > **Remember to store the master key somewhere secure. It can be used to recover the filesystem if the FIDO2 device is lost.**

    If the user's home directory is empty, you can skip to configure PAM for auto-mounting at login.

### 2. Moving existing home directory files to encrypted filesystem. [source][1]
You will need to mount the newly created encrypted filesystem and copy the entirety of the user's home directory into it. As each file is copied in, it is encrypted by gocryptfs. To avoid any programs changing files in the user's home directory in the middle of the copy operation, the user *must* be logged out. Then, *as root*: 

1. Create the user's new home directory:
    ```
    # mv /home/$user /home/$user.old
    # mkdir -m 700 /home/$user
    # chown $user /home/$user
    # chgrp $user /home/$user
    ```

2. Mount the encrypted filesystem at the new home directory using your FIDO2 token:  
    You may have to touch your token.
    ```
    # gocryptfs -fido2 /dev/hidraw* /home/$user.cipher /home/$user
    ```

3. Copy all home directory files into the mounted filesystem using rsync:
    ```
    # rsync -av /home/$user.old/ /home/$user
    ```
    You can install rsync with:
    ```
    # apt install rsync
    ```

4. Unmount the filesystem:
    ```
    # fusermount -u /home/$user
    ```

### 3. Configure PAM for auto-mounting at login
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
    In `/etc/security/pam_mount.conf.xml` add a new volume tag at the end of the file, and then configure then logout and umount tags. Please see **[INSERT GITHUB EXAMPLE REF OF `pam_mount.conf.xml]**. Remember to change the username.
    ```
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

3. Configure pam_mount for mounting gocryptfs.  
    Download the mounting script from Github:
    ```
    # curl -o /usr/sbin/mount.gocryptfs http://github.com/INSERT_PATH_HERE
    ``` 
    Edit `mount.gocryptfs` with the users home directory and cipher.
    ```
    MOUNT_POINT="/home/your_username_here"
    ENCRYPTED_PATH="/home/your_username_here.cipher"
    ...
    ```
    And make it executable so that pam_mount can run it.
    ```
    chmod +x /usr/sbin/mount.gocryptfs
    ```

4. Configure PAM.  
    Edit `/etc/pam.d/common-auth` and append the arguments `disable_pam_password` and `disable_interactive` to `pam_mount.so`:
    ```
    ...

    # and here are more per-package modules (the "Additional" block)
    auth	optional			pam_mount.so  disable_pam_password disable_interactive
    # end of pam-auth-update config
    ```
    The do the same edit to `/etc/pam.d/common-session`:
    ```
    # and here are more per-package modules (the "Additional" block)
    session	required	pam_unix.so 
    session	optional	pam_mount.so disable_pam_password disable_interactive 
    session	optional	pam_systemd.so 
    # end of pam-auth-update config
    ```

    Logging in as the user will now cause the encrypted filesystem to be mounted transparently. Logging out will correspondingly unmount the encrypted filesystem. 

### 4 Securely delete the unencrypted home directory. [source][1]
If the unencrypted home folder was not empty, the user's home directory was then moved to *$user*.old earlier. These unencrypted files need removing securely; otherwise the encryption protecting the data can be easily avoided by just looking in the other folder. These older files need deleting securely as well - simply performing `rm -rf $user.old` will not remove the file from disk completely, it will just remove the reference to it.  

Multiple tools exist that claim to delete files securely (most notably shred) but these come into conflict with the journaling functions of jornaling filesystems (such as Ext4). The secure deletion tool is trying to make it so that you can't recover your files whilst the journaled filesystem is trying to make sure that you can recover them. 

[1]: https://wiki.archlinux.org/title/User:Lukeus_Maximus "Home directory encryption"
[2]: https://developers.yubico.com/libfido2/Manuals/fido2-token.html "Yubico Docs, fido2-token"
