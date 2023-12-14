# Mounting Borg backups or Restic repos without FUSE in macOS
Workaround method for mounting Borg or Restic repos in macOS without FUSE. Specifically aimed for browsing backups to preview files within or restoring a portion of a backup rather than everything.

# Preliminary Notes
[Restic](https://restic.net/) and [BorgBackup](https://www.borgbackup.org/) are probably the most stable and well-developed FOSS backup tools for macOS that have a mounting capability for backup browsing. However, both use macFUSE for mounting on macOS, which requires reducing security due to its system kext requirement. 

Until Borg and restic implement an kextless alternative like [FUSE-T](https://www.fuse-t.org/), or implement an alternative method of mounting like using webDAV (example: Cryptomator), [Kopia](https://kopia.io/) is a great alternative if you are willing to use something that is still in active development (but by most accounts, completely fine, and being commercially supported/deployed), as it supports [WebDAV mounting](https://kopia.io/docs/reference/command-line/common/mount/) of snapshots. 

For those that do still want to use Restic or Borg and do not want to change the required security settings needed to make macFUSE work on apple silicon (M1, etc.), the easiest ways to do so are using things like [ResticBrowser](https://github.com/emuell/restic-browser) and [Borg-Repo-Explorer](https://github.com/Netruk44/borg-repository-explorer), which are viable ways to browse backups with minimal effort, but are not a full solution to be able to search quickly and preview files yet. 

This hacky workaround uses a lightweight Alpine Linux VM to handle to handle mounting backups, and Cyberduck to browse them. Transferring files out of backups is done either with SSHFS/SFTP using Cyberduck, or via a VM-Host shared directory. 


**refer to [new-lima-templates](new-lima-templates) folder for correct .yaml's** 


# Part 1) First, Follow one of the methods below: 

## Method For macOS 13 and Above (Ventura, Sonoma)
This method uses Lima with Apple's Virtualization.framework (VZ), Alpine Linux and VirtioFS along with Cyberduck. Offers the best combination of convenience, file copying performance and low resource usage I could come up with. 

### Step 1: Download and configure Lima VM template
1. Download one of the  `alpine-virtiofs-*.yaml` config file from this repository to a directory of your choice.
   - This is a configuration file will install a minimal Alpine Linux instance with rsync. 
   - BorgBackup users should download  `alpine-virtiofs-borg.yaml`. Restic users should download  `alpine-virtiofs-restic.yaml`.
2. Open up the .yaml file in a text editor. We need to configure a shared directory location between VM and the host to be able extract files from your backups to.
   - go to the "mounts" section in the file
   - <img width="392" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/f6f40ba4-73ff-46f8-8370-491974f0b846">
   - change /path/to/location/on/host to the filepath to the directory where you want files to show up in macOS. This filepath has to be the full filepath. For example, for a directory called 'testdir' on the Desktop in macOS, you would put `/Users/yourmacusername/Desktop/testdir`. **Make sure to create this directory before moving on.**
   - change /path/to/location/on/vm to the filepath where you want the directory to show up in the Linux VM. This is where you will copy files from your backups to in the VM. A convenient location is /mnt. For example, `/mnt/HostMount`.
   - if you perhaps store backups locally, you can mount them here to a folder by creating a new `- location` entry. Just follow the same "location, mountPoint, writeable" pattern you see in the template already. 
3. Scroll to the `provision` section, and under the `script` portion, **under** `sudo cp -r /home/*/.ssh /root` (Do not change), make any additional folders you want by adding in lines of `sudo mkdir /path/to/directory/in/vm`.
  - Add a dedicated folder to temporarily mount backups to, for example `sudo mkdir /mnt/BackupMountPoint`.
  - If you use a NAS on your network to store your backups using an SMB share, or a remote location that you mount with rclone, you can make a folder in which to mount them in, like `sudo mkdir /mnt/NASMountPoint`.
4. Feel free to add/delete packages as you wish from the `provision` > `script` section of the yaml file you download or just leave it as is.
   - For example, you may want to add rclone if you store and mount backups from a remote location.
   - If you are mounting network storage of some kind, you may need to add some packages. For example, included already are the tools necessary for mounting SMB/CIFS shares, but you may need to mount an NFS share, so you would need to add a line to install the `nfs-utils` package using `sudo apk add nfs-utils`.
5. under the `ssh` section, a `localPort` has been defined that will allow us to access the VM via Cyberduck. I chose port `54551` because it is usually a free port, but you can change it to some other number if you wish.
6.  Save and close.

### Step 2: Create VM using Lima and Cyberduck Configuration
1. Open Terminal and download/install [Lima](https://github.com/lima-vm/lima) via [homebrew](https://brew.sh) using `brew install lima`. Lima is a lightweight virtualizer for macOS that lets you easily spin up linux VMs. We are going to be spinning up a really small Alpine Linux VM to achieve what we want/
2.  `cd` to the the directory you downloaded the .yaml file to and then run `limactl create --cpus=2 --memory=2 --disk=5 --name=name-here /path/to/alpine-*.yaml`. Hit enter to select Proceed with current configuration, and Lima will create an Alpine Linux VM based on the .yaml file.
   - Tweak the number of cpu cores, memory and disk size how you see fit. See [Lima's documentation](https://lima-vm.io/docs/reference/limactl_create/) for notes.
   - Set `--name` to what you want the VM instance to be called, like `alp`
   - `/path/to/alpine-*.yaml` is the filepath to where you downloaded whichever .yaml file you chose
3. Download and install [Cyberduck](https://cyberduck.io). Cyberduck is a server/storage browser that is very similar to Finder. It will allow you to connect to the Alpine Linux VM and browse/transfer files from it to your Mac.
4. run `limactl start name-here` where name-here is whatever you named the instance. This will launch your instance.  
   - The initial setup may take a bit as your VM instance downloads all the packages and sets everything up. Once it's done, you can verify it's running by using `limactl list`.
5. Once your newly spun up VM is running, open Cyberduck. In the menu bar, go to Bookmark > New Bookmark. In the window that pops up:
   - Click on the dropdown that currently says FTP, and change it to SFTP (SSH File Transfer Protocol)
   - Change the Nickname field to whatever you wish to call the bookmark.
   - In the Server field, type `127.0.0.1`. In the Port field type the ssh port number you assigned in the .yaml file from Part 1, Step 1 (the default is port `54551`.
   - In the User field, type `root`
   - Click the SSH Private Key dropdown, and click Choose. A finder window should open, and by default it should show your Mac's User folder. Turn on "Show Hidden Files" by hitting `Cmd + Shift + .`. A few new folders should be showing now. Go to .lima > _config > and click the `user` file (NOT the user.pub file) and then hit Choose. The field should now say `~/.lima/_config/user`.
   - Close the window, and there should be a bookmark in the main Cyberduck window now.
6. Either continue to Part 2 and configure an auto-start script or skip to the Manually Mounting section.

## Method For macOS 12 and Below (Monterey/Big Sur/etc.)
This method uses Lima, QEMU, Alpine Linux, and VirtFS-9P.

1. Download one of the  `alpine-virt9p-*.yaml` config file from this repository to a directory of your choice.
   - BorgBackup users should download  `alpine-virt9p-borg.yaml`. Restic users should download  `alpine-virt9p-restic.yaml`.
2. Follow the rest of the steps in the Method for macOS13 and Above.

If you are having issues, use one of the Alpine GUI methods in [AlpineDeprecratedMethods](AlpineDeprecatedMethods.md) or Ubuntu GUI methods in [UbuntuDeprecatedMethods](UbuntuDeprecatedMethods.md).

# Part 2)  (optional) Configure a one-click startup 'button' and auto-mount script. 

By now, you have a really lightweight VM that has all the tools you need to mount backups. The next step is to make everything start up easily.   

Below is an example of a small "one-click" startup button made using the built-in macOS application Automator. This script starts the VM, opens Cyberduck and starts an SSH session via the bookmark you made, and (optionally), runs a mount script inside the VM that will mount any network locations where you may store your backups, and mounts your Borg/Restic Backups to a directory in the VM (such as the one you created in Part 1, Step 1). **This assumes you only have one bookmark in Cyberduck**.

## Startup script using Automator
1. Open Automator, and create a new Application. Add a "Run Applescript" to the flow. Below is an example of what you can have in the startup script:
   - <img width="1051" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/71ce4eeb-4f7f-4227-8a40-fda2c21480a0">
2. If you are not mounting a network location, or do not want to use mounting script:
   - delete the `do script "limactl copy ....`,  `do script "chmod ...`, and `do script "./mountingscript.sh` lines.
   - change all instances of `name-here` to whatever you named your VM instance
4. If you are:
   - change `/path/to/mounting/mountingscript.sh` to the directory of where your mounting script is on macOS
   - change all instances of `name-here` to whatever you named your VM instance
   - change `mountingscript.sh` to whatever you named your mount script if you have one
5. Save it to your Applications folder and name it whatever you want.
6. Running this script will open a terminal window and run the commands in the terminal window automatically. You may be prompted for credentials within the terminal window depending on if you choose to use mounting script in conjunction. It will also open Cyberduck and start a
  
## Mounting Script or Manually Mounting

### Mounting Script
This is a bash script you can make that automounts any network storages and your borg/restic repo. You can create it using a text editor. Name it something like `mountingscript.sh`. Below is an example that mounts a NAS with an SMB share and a borg backup stored on that SMB share:

<img width="903" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/bad026f6-04dc-4ffd-b19b-553bad3d6646">

You can change the paths to whatever you like, and the mountpoints to the folders you may have created in Part 1, Step 1. 

This script will be called copied over by the automator script to the VM if you have configured it to do so, and executed automatically.  you will be prompted for your credentials to mount your network share and your repository. Configure this script however you need to. 

For restic, see their docs for `restic mount` [here](https://restic.readthedocs.io/en/latest/050_restore.html).

For Borg, see their docs for `borg mount` [here](https://borgbackup.readthedocs.io/en/stable/usage/mount.html). 

### Manually Mounting
If you choose not to make a mount script, you can just mount via command line as you normally would. While the VM is running, in a terminal window, run `limactl shell name-here`. This will open terminal instance inside the VM where you can run commands. See the docs linked above. **Make sure to run all commands as sudo, or you may get permissions errors.**


# Part 3) Browsing Backups, Previewing files in the backup, Copying files to the host

If you configured an auto-start script/mounting script earlier then it will automatically open the VM, open Cyberduck and start a connection using the bookmark you made earlier, and mount your Borg/restic backup. 

If you chose to manually mount, make sure you have mounted your Borg/Restic backups within the VM **before** opening Cyberduck and starting a connection to the VM by double clicking the bookmark you made earlier. 

## Browsing

Either case may open a window in Cyberduck saying Changed fingerprint and Allow or Deny. Tick Always if you do not want to see this window when you connect, and then hit Allow. By default, Cyberduck starts the connect in the user you are connecting as's home folder. Since we are connecting as `root`, it drops us in `/root`. You can now use Cyberduck similarly to how you would Finder. Browse to whereever you mounted your backup earlier, and browse it like you normally would. 

## Previewing
To preview files, you can use Cyberduck's Quick Look feature. This works especially well with files of smaller sizes. Just browse to the file in the backup/repo you want to preview, right click on them and hit Quick Look. This will simply open them within macOS in a program that can open them. For example, if you Quick Look a pdf file, it will just open in Preview like normal. Quick Look may take a few seconds longer with larger files like movies in my testing, so it may be more convenient to just copy them over quickly to view them.

## Copying

### For small amounts of files/Files with smaller sizes:

Simply drag them from Cyberduck to whereever you want to copy the files to on macOS. This will transfer the files out of the VM via SFTP/SSHFS. This method is fast enough for most cases. 

### For larger amounts of files/files with larger sizes:

In a terminal window, run `limactl shell name-here`, which will start a terminal instance to your VM (if you have configured an auto-start script similar to above, then there will already be one open). In Cyberduck, right click on the file/directory you wish to copy to the host and hit Info (or just use Cmd + I). In the window that pops up, under General, copy the filepath next to Where. 

Go back to the terminal window and run an rsync command similar to `sudo rsync -r --info=progress2 /pasted/file/path/from/Cyberduck/ /path/to/shared/VM/host/directory`
- `/pasted/file/path/from/Cyberduck/` is where you can just paste the filepath you copied from Cyberduck (path to the files within your mounted backup)
- `/path/to/shared/VM/host/directory` is the path to the shared VM-Host directory within the VM that you configured in the .yaml file in Part 1, Step 1. 

This command uses rsync to copy the desired file/directory to the VirtioFS/Virt-9P shared VM-Host directory, and show progress. It will be much faster than using SSHFS/SFTP. The file/directory will appear on macOS where you configured them to in the .yaml file from Part 1, Step 1. 

### For complete restores:

- see `restic restore` documentation [here](https://restic.readthedocs.io/en/latest/050_restore.html) or borg's restore options documentation [here](https://docs.borgbase.com/restore/borg/). and copy to the shared directory.

- But honestly, if you plan on extracting most, if not all files within a backup, it is much easier to just do this **natively** on macOS using Vorta for Borg/borg extract or restic restore rather than within the VM. 


# Closing Notes
This was the most straightforward way I could come up with to achieve what I wanted with on macOS. This was inspired by this [post](https://forum.restic.net/t/restic-mount-on-osx-and-windows-in-2021/4487/3). Things that could be done to improve this include setting up a BASH script to do much of the setup autimatically, or some of the alternatives/tweaks listed below.
- [Macpine](https://github.com/beringresearch/macpine), [OrbStack](https://orbstack.dev), and [Tart](https://tart.run) may be viable alternatives to Lima incase it is giving you a headache. Further, if you would be okay using Ubuntu specificially instead of Alpine, [Multipass](https://multipass.run) is an extremely easy way to get Ubuntu VMs running. It does not support VirtioFS yet, but I think they are actively working on implementing this and support Apple's Virtualization.framework instead of QEMU. See [UbuntuDeprecatedMethods](UbuntuDeprecatedMethods.md). 
- Docker using bind mounts(?) may be another way to achieve this. See [restic](https://forum.restic.net/t/cannot-mount-on-macos-with-macfuse-4-x/3338/5). Could be adapted for Borg by someone motivated.
