# Mounting Borg backups or Restic repos without FUSE in macOS
Workaround method for mounting Borg or Restic repos in macOS without FUSE. Specifically aimed for browsing backups to preview files within or restoring a portion of a backup rather than everything.

# Preliminary Notes
[Restic](https://restic.net/) and [BorgBackup](https://www.borgbackup.org/) are probably the most stable and well-developed FOSS backup tools for macOS that have a mounting capability for backup browsing. However, both use macFUSE for mounting on macOS, which requires reducing security due to its system kext requirement. 

Until Borg and restic implement an kextless alternative like [FUSE-T](https://www.fuse-t.org/), or implement an alternative method of mounting like using webDAV (example: Cryptomator), [Kopia](https://kopia.io/) is a great alternative if you are willing to use something that is still in active development (but by most accounts, completely fine, and being commercially supported/deployed), as it supports [WebDAV mounting](https://kopia.io/docs/reference/command-line/common/mount/) of snapshots. 

For those that do still want to use Restic or Borg and do not want to change the required security settings needed to make macFUSE work on apple silicon (M1, etc.), the easiest ways to do so are using things like [ResticBrowser](https://github.com/emuell/restic-browser) and [Borg-Repo-Explorer](https://github.com/Netruk44/borg-repository-explorer), which are viable ways to browse backups with minimal effort, but are not a full solution to be able to search quickly and preview files yet. 

This hacky workaround uses a lightweight Linux VM to handle mounting and browsing backups virtually, then transferring files using a VM-Host shared directory.


# Part 1) First, Follow one of the methods below: 

## Method For macOS 13 and Above (Ventura, Sonoma)
This method uses Lima with Apple's Virtualization.framework (VZ), Alpine Linux and VirtioFS. Offers the best combination of file copying and low resource usage I could come up with. 

### Step 1: Download and configure Lima VM template
1. Download one of the  `alpine-virtiofs-*.yaml` config file from this repository to a directory of your choice.
   - This is a configuration file will install Alpine Linux with the LXQt desktop environment and a few packages like LibreOffice Writer/Calc, xpdf, GEdit and mpv so that you can look at files of different types in your backup. 
   - BorgBackup users should download  `alpine-virtiofs-borg.yaml`. Restic users should download  `alpine-virtiofs-restic.yaml`.
2. Open up the .yaml file in a text editor. We need to configure a shared directory location between VM and the host to be able extract files from your backups to.
   - go to the "mounts" section in the file
   - <img width="392" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/f6f40ba4-73ff-46f8-8370-491974f0b846">
   - change /path/to/location/on/host to the filepath to the directory where you want files to show up in macOS. This filepath has to be the full filepath. For example, for a directory called 'testdir' on the Desktop in macOS, you would put `/Users/yourmacusername/Desktop/testdir`. **Make sure to create this directory before moving on.**
   - change /path/to/location/on/vm to the filepath where you want the directory to show up in the Linux VM. This is where you will copy files from your backups to in the VM. A convenient location is /mnt. For example, `/mnt/HostMount`.
   - if you perhaps store backups locally, you can mount them here to a folder by creating a new `- location` entry. Just follow the same "location, mountPoint, writeable" pattern you see in the template already. 
3. Scroll to the `provision` section, and under the `script` portion, **under** `sudo rc-update add sddm` (Do not change), make any additional folders you want by adding in lines of `sudo mkdir /path/to/directory/in/vm`.
  - Add a dedicated folder to temporarily mount backups to, for example `sudo mkdir /mnt/BackupMountPoint`.
  - If you use a NAS on your network to store your backups using an SMB share, or a remote location that you mount with rclone, you can make a folder in which to mount them in, like `sudo mkdir /mnt/NASMountPoint`.
3. Feel free to add/delete packages as you wish from the `provision` > `script` section of the yaml file you download or just leave it as is. For example, you may want to add rclone if you store and mount backups from a remote location.
   - If you are mounting network storage of some kind, you may need to add some packages. See the "auto-mounting USB drives" and "network browsing" sections of the [LXQt Alpine Wiki page](https://wiki.alpinelinux.org/wiki/LXQt). Add any gvfs package that you need by adding a line like `sudo apk add ...`. For example, included already are the tools necessary for mounting SMB shares, `sudo apk add gvfs-smb`, but you may want to mount NFS shares so you may need to add `sudo apk add gvfs-nfs`.
4.  Save and close.

### Step 2: Create VM using Lima and Initial Configuration
1. Open Terminal and download/install [Lima](https://github.com/lima-vm/lima) via [homebrew](https://brew.sh) using `brew install lima`. Lima is a lightweight virtualizer for macOS that lets you easily spin up linux VMs..We are going to be spinning up a really small Alpine Linux VM to achieve what we want/
2.  `cd` to the the directory you downloaded the .yaml file to and then run `limactl create --cpus=2 --memory=2 --disk=10 --name=name-here /path/to/alpine-*.yaml`. Hit enter to select Proceed with current configuration, and Lima will create an Alpine Linux VM based on the .yaml file.
   - Tweak the number of cpu cores, memory and disk size how you see fit. See [Lima's documentation](https://lima-vm.io/docs/reference/limactl_create/) for notes.
   - Set `--name` to what you want the VM instance to be called, like `alp`
   - `/path/to/alpine-*.yaml` is the filepath to where you downloaded whichever .yaml file you chose
3. run `limactl start name-here` where name-here is whatever you named the instance. This will launch your instance and open a separate window with the linux VM. **Closing this window stops the VM, so do not close it unless you want to fully shut down the VM**. Leave the terminal window from which you are running commands open as well. 
   - The initial setup will take a bit as your VM instance downloads all the packages and sets up the GUI/desktop environment/applications. You will eventually see the LXQt login screen come up. Take note of the username on the login window. It should be your Mac's login username or something similar.
4. From the terminal window you left open, run `limactl shell name-here`. This opens an ssh instance to the VM from which you can run commands. you should see something like `lima-name-here:~$` in the terminal window.
    - Run `sudo passwd yourusername` where yourusername is the username you saw on the LXQt login page. This will prompt you to set a password. Set it to something.
     -  Run `sudo passwd root`. This is setting the root password just in case. Set it to something else.
5. Now go back to the window showing the Linux VM and enter the password you set for your username. This will log you into the VM. You should see a linux GUI desktop now.
6. We now have to just make sure icons show up properly.
   - First hit the menu button in the bottom left (like the windows start button) then go to Preferences > LXQt Settings > LXQt Configuration Center. Click appearance and go to Icons Them, and hit Adwaita and hit apply. This will make icons actually appear. Close these windows.
   - Next go the the menu button > Accessories > PCManFM-Qt. This will open up the File manager.
   - Go to Tools > Open Tab in Root Instance. This will open up a new window that should say "Root Instance" somewhere. Right Click on the top where the file path is show and hit "edit path". navigate to `/home/yourusername`.
   - Go to View > Show Hidden. This should show some new folders. Copy the `.config` one that pops up. Go to the address bar and navigate to `/root`. Paste the `.config` folder there. Tick Apply to all files and hit overwrite. Close the Root instance window, and reopen a window in Root Instance. Icons should now also be showing in the Root instance window.
- **PCManFM-QT Root Instance** windows are what you should use to browse your backup to avoid weird permission issues.
7. Shut off the VM by going to the menu button > Leave > Shutdown.

## Method For macOS 12 and Below (Monterey/Big Sur/etc.)
This method uses Lima, QEMU, Alpine Linux, and VirtFS-9P.

1. Download one of the  `alpine-virt9p-*.yaml` config file from this repository to a directory of your choice.
   - This is a configuration file will install Alpine Linux with the LXQt desktop environment and a few packages like LibreOffice Writer/Calc, xpdf, GEdit and mpv so that you can look at files of different types in your backup. 
   - BorgBackup users should download  `alpine-virt9p-borg.yaml`. Restic users should download  `alpine-virt9p-restic.yaml`.
2. Follow the rest of the steps in the method for macOS13 and above.

I had some issues with this, such as the mouse not showing up sometimes. If you are having issues, use one of the Ubuntu methods in [DeprecatedMethods](DeprecatedMethods.md)

# Part 2)  Next, configure a one-click startup 'button' and auto-mount script. 

By now, you have a really lightweight VM that has all the tools you need to mount backups and 

**This setup has a weird quirk that the password for the default user always gets reset for some reason**. 

Below is an example of a small "one-click" startup button made using the built-in macOS application Automator. This script starts the VM, does the manual step of changing this users' password, and (optionally), runs a mount script inside the VM that will mount any network locations where you may store your backups, and mounting your Borg/Restic Backups to a directory in the VM (such as the one you created in Part 1, Step 1.

## Startup script using Automator
1. Open Automator, and create a new Application. Add a "Run Applescript" to the flow. Below is an example of what you can have in the startup script:
   - <img width="614" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/9965c13e-8859-4e3c-a250-b18a11939050">
2. If you are not mounting a network location, or do not want to use mounting script, delete the `do script "limactl copy ....`,  `do script "chmod ...`, and `do script "./mountingscript.sh` lines.
3. If you are:
   - change `/path/to/mounting/mountingscript.sh` to the directory of where your mounting script is on macOS
   - change all instances of `name-here` to whatever you named your VM instance
   - change all instances of `yourlinuxusernamehere`  to whatever the default username on the VM is (the one you saw in the log in screen during Part 1, Step 2, **not root**
   - change `passwordhere` to whatever you want the user's password to be
   - change `mountingscript.sh` to whatever you named your mount script if you have one
4. Save and name it whatever you want. It will appear in your Applications on macOS.
5. Running this script will open a terminal window as well as a window with your VM and run the commands in the terminal window automatically. You may be prompted for credentials within the terminal window depending on if you choose to use mounting script in conjunction.
  
## Mounting Script or Manually Mounting and Copying files to the host

### Mounting Script
This is a bash script you can make that automounts any network storages and your borg/restic repo. You can create it using a text editor. Name it something like `mountingscript.sh`. Below is an example that mounts a NAS with an SMB share and a borg backup stored on that SMB share:

<img width="903" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/bad026f6-04dc-4ffd-b19b-553bad3d6646">

You can change the paths to whatever you like, and the mountpoints to the folders you may have created in Part 1, Step 1. 

This script will be called copied over by the automator script to the VM if you have configured it to do so, and executed automatically.  you will be prompted for your credentials to mount your network share and your repository. Configure this script however you need to. 

For restic, see their docs for `restic mount` [here](https://restic.readthedocs.io/en/latest/050_restore.html).

For Borg, see their docs for `borg mount` [here](https://borgbackup.readthedocs.io/en/stable/usage/mount.html). Vorta is unforunately not available for Alpine Linux officially. 

### Manually Mounting
If you choose not to make a mount script, you can just use the terminal inside the VM (QTerminal from the menu button) and mount via command line as you normally would. See the docs linked above. **Make sure to run all commands as sudo, or you may get permissions errors.**


### Copying files to the host

For small amounts of files:
Once you mount your backups within the VM, copy/store any files you want to shared VM-Host directory you configured in Part 1, Step 1, and they will appear in MacOS. You can also use restic/borg's restore functions. 

For large/complete restores:
- see `restic restore` documentation [here](https://restic.readthedocs.io/en/latest/050_restore.html) or borg's restore options documentation [here](https://docs.borgbase.com/restore/borg/).

- But honestly, if you plan on extracting most, if not all files within a backup, it is much easier to just do this **natively** on macOS using Vorta for Borg/borg extract or restic restore.



# Closing Notes
This was the most straightforward way I could come up with to achieve what I wanted with on macOS. This was inspired by this [post](https://forum.restic.net/t/restic-mount-on-osx-and-windows-in-2021/4487/3). Things that could be done to improve this include setting up a BASH script to do much of the setup autimatically, or some of the alternatives/tweaks listed below.
- This could potentially be even lighter-weight if you are more comfortable doing everything without a GUI. Skip the LXQt stuff by removing the provision section from the template and do everything from CLI. You just lose the ability to fully preview the content of files within a backup without first copying to the host machine. 
- [Macpine](https://github.com/beringresearch/macpine), [OrbStack](https://orbstack.dev), and [Tart][https://tart.run] may be viable alternatives to Lima incase it is giving you a headache. Further, if you would be okay using Ubuntu specificially instead of Alpine, [Multipass](https://multipass.run) is an extremely easy way to get Ubuntu VMs running. See [DeprecatedMethods](DeprecatedMethods.md). 
- Docker using bind mounts(?) may be another way to achieve this. See [restic](https://forum.restic.net/t/cannot-mount-on-macos-with-macfuse-4-x/3338/5). Could be adapted for Borg by someone motivated.
