# Full Disk Encryption Home Folder (WIP)

"Why yes, I wasted 5 hours just to tell you how to make soup."

So I decided one day that I wanted to encrypt my drives after I wanted to move my home folder to a entirely different drive. This is neat and cool, but there's some hiccups that can happen in the process that might make you want to rip your hair out!

I've already ripped my hair out, thankfully, so that you won't have to.

This guide assumes you are making encrypted disks for use on Linux systems only. I don't know how to make encrypted Windows drives (and much less of encrypted Windows boot drives) just yet.

## Downloading and set-up

To begin, we must first create an encrypted drive. This can be done through any number of utilities... Perosnally I like using `Gnome-disks`

Debian: `apt install gnome-disk-utility cryptsetup`

Fedora: `dnf install gnome-disk-utility cryptsetup`

Arch: `pacman -Sy gnome-disk-utility cryptsetup`

Find your disk and create a new partition with it. Name it whatever but be sure to select the "Internal disk for use with Linux systems only". Select the "password protect volume" option. Give it a password and, there you go, your drive is encrypted!

That's just the easy part though, here's where it can get confusing if you want to auto-mount it.

## Moving /home to a new drive

There's a plethora of reasons as to why you'd want to do this. For instance:
- It can save time with installing new operating system
- It's easier to move the home drive to a new system
- You save time on not needing to back up everything from your boot drive

That being said, your new home folder is encrypted and you need to get it all set up. Here's how.

First, if you haven't, decrypt the drive so you can access it. Make it mountable at someplace like `/mnt/tmp`. I'll write down instructions here on how to do it from CLI but you can just as easily do this through a GUI between gnome-disk-utility and other applications.

```bash
sudo mkdir /mnt/tmp
# give your encrypted volume a name where the brackets are
# if you don't know which drive holds your encrypted volume, do a lsblk to find it
# or peek through gnome disk utility for the drive
sudo cryptsetup luksOpen /dev/sdX1 <name-of-encrypted-volume>
sudo mount /dev/mapper/<name-of-encrypted-volume> /mnt/tmp
```

Great, now the drive is decrypted and is mounted. Now we copy our home folder into it.

```bash
sudo rsync -avx /home/ /mnt/tmp
```

The rsync command will help to preserve our file permissions, as to prevent issues upon trying to remount it in the future

Once these are done, we can test out to see if it worked. Take your drive and unmount it, then mount it again but at `/home`
```bash
sudo umount /mnt/tmp
sudo mount /dev/mapper/<name-of-encrypted-volume> /home
```

From here you can check and see if everything looks good. Everything should function just fine, work the same way, read and write well. If that's all good, then we can move onto the next step.

### "But what if I don't want to encrypt it?"

Tbh, if you don't want to bother with encrypting the drive, you could just leave it here as it is but don't encrypt the drive from the get-go. The steps here would work with any non-encrypted drive, sans the fact that mounting it would be different

```bash
sudo mount /dev/sdX1 /mnt/tmp
```

I write this, assuming you know how to make new partitions on disks already, be it from CLI or from a GUI like `gnome-disk-utility` or even `gparted`.

## Auto-mounting and auto-decrypting

The hard part.

For this, we'll need to edit our `fstab` file and our `crypttab` files, both located in `/etc`.

To prevent future headache, I recommend finding your disk's UUID within gnome-disk-utility. It's simple to find, you select your desired disk and click on it from the list. The information in the window should label it clearly. We'll need this for the aformentioned files.

Open up your `fstab` file. Here's an example of what mine looks like:
```
UUID=4bf3001c-0fcf-47ae-9f08-7ef49fc42d36 /                       btrfs   subvol=root,compress=zstd:1,x-systemd.device-timeout=0 0 0
/dev/disk/by-uuid/398b6a5f-783a-4862-a286-2e501dc231be /home	 ext4    auto nofail,users,default     0 2
```

Mine may look a little odd due to the fact that my boot drive is already encrypted, but here we have two drives: my boot drive and my /home drive.

The home drive will need to be denoted with it's UUID, as to prevent future headache. This can be done with `/dev/disk/by-uuid/<UUID-of-disk>`
Afterwards, follow it with these labels following after it (with spaces!):
```
/home      auto nofail,users,default   0 2
```

This tells the system:
- Mount the drive at `/home`
- Mount it automatically
- Don't print out errors at startup, make available to users, default options
- disable dump, filesystem check to put it second in-line to first option (which is the boot drive usually)

If your home partition was also already listed here in the fstab, just comment it out for the time being.

Now we open `crypttab`.
```
home UUID=55951cfd-002a-41cd-ae38-6cab43903815 /etc/luks-keys/homekey nofail
```

Crypttab is a file that a daemon process can call to in order to decrypt drives at start-up. In this order, the crypttab reads that:
- The decrypted volume will be called "home" in the dev mapper
- The UUID of the encrypted volume, which is **DIFFERENT** from the volume's UUID.
- The file of which the key to decrypt it is kept
- Option to not print errors (but more-so to hide the password screen on start-up)

For the UUID here, we need to get the UUID from the actual ENCRYPTED portion of the drive. In gnome-disk-utility, your drive should have two volumes in it, with one stacked on top of the other. The top one should read out `LUKS` while the other reads `Ext4`. We want to select the `LUKS` partition and read out it's UUID. This is what will be put into the `crypttab` file here.

Now for automatically decrypting the drive, we'll need to put the key somewhere where the root file system can access it. The key is the password that was made earlier for the drive's encryption. If no such place exists, you can create it in either /root or in `/etc/<dir-name-of-your-choice>`

If you're making the dir in /etc, make sure to:
- `chown -R root:root` the dir
- `chmod 700` the dir
- `chmod 600` the keyfile

Additionally, you may want to run this on the keyfile (as a root user or in a root shell):
```
perl -pi -e 'chomp if eof' /path/to/file
```
I include this because I had an odd issue where my drive wouldn't auto-decrypt despite the key being correct. This is because text editors would create a new line beyond the key, which crypt accepts as a value and will make the drive fail to decrypt. This command here will help trim out that empty space, leaving only the key.

Source: https://serverfault.com/questions/742675/cryptsetup-luksopen-key-file-does-not-work

Alright, so now we should be set up and ready to have our encrypted /home drive auto-mount at start-up and also decrypt itself.

## BEFORE YOU REBOOT (and general troubleshooting)
I advise, with a bit of caution, that you have a Linux livedisk at the ready. If someting goes wrong and you can't decrypt your home drive automatically, you won't be able to log into your system without doing it in TTY.

If issues arise after rebooting, don't panic! Your files are still fine. If you followed my instructions, you should still have your old home folder + your new one. Maybe something got messed up somewhere. Nonetheless, we'll fix it!

- _Did you run the perl command?_ That newline thing messed me up when I was doing this myself. If that's not ran, your disk will fail to decrypt!
- _Are your UUID's correct?_ If you can't access gnome-disk-utility from your desktop anymore, you can do it from a livedisk and make sure that you have your UUID's correct. You can also list them out in TTY with `sudo blkid`
- _Is the formatting in fstab/crypttab correct?_ Make sure that your entries into fstab/crypttab match similar ones that may already exist within those files (or that they match similarly to mine)

### For Fedora/RedHat users
SELinux comes installed on RedHat distros by default and you may find yourself unable to login still despite ensureing that you did the above. This is a SELinux issue. To resolve this, log into your system via TTY and run this command:
```
restorecon -R /home
```
It may take a while to run, depending on how large your filesystem is, but afterwards, it should resolve your issues.


## TO-DO:
- Formatting
- Proper, dedicated section for if you just want to do this with an unencrypted drive
- Security cautions and advice
- MFA decryption with something like a hardware key (Yubikey)
- Ridding of old home folder if everything is all Good and Cool
- How-to's but with CLI (if running a minimal system or a system that can't run GDU)
