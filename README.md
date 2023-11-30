# Mounting Borg backups or Restic repos without FUSE in macOS
Workaround method for mounting Borg or Restic repos in macOS without FUSE.

# Preliminary Notes
[Restic](https://restic.net/) and [BorgBackup](https://www.borgbackup.org/) are probably the most stable and well-developed FOSS backup tools for macOS that have a mounting capability for backup browsing. However, both use macFUSE for mounting on macOS, which requires reducing security due to its system kext requirement. 

Until Borg and restic implement an kextless alternative like [FUSE-T](https://www.fuse-t.org/), or implement an alternative method of mounting like usingwebDAV (example: Cryptomator), [Kopia](https://kopia.io/) is a great alternative if you are willing to use something that is still in active development (but by most accounts, completely fine, and being commercially supported/deployed), as it supports [WebDAV mounting](https://kopia.io/docs/reference/command-line/common/mount/) of snapshots. 

For those that do still want to use Restic or Borg and do not want to change the required security settings needed to make macFUSE work on apple silicon (M1, etc.), the easiest ways to do so are using things like [ResticBrowser](https://github.com/emuell/restic-browser) and [Borg-Repo-Explorer](https://github.com/Netruk44/borg-repository-explorer), which are viable ways to browse backups with minimal effort, but are not a full solution to be able to search quickly and preview files yet. 

This hacky workaround uses a lightweight Linux VM to handle mounting and browsing backups virtually, then transferring files using a VM-Host shared directory.

# Method

## Setup

### Step 1: Create and configure Multipass VM
1. Download and install the [Multipass](https://multipass.run/download/macos) package. Multipass is a super simple CLI tool by Canonical (makers of Ubuntu) that lets you spin up really lightweight ubuntu VMs on macOS.
2. Open Terminal, and create an ubunutu VM using `multipass launch --disk 10G --name name-here`.
   - `10G` is the disk size you are alloting for the VM, and can be changed to anything. I suggest 10G just in case, but it will not take 10G of space immediately and just sets the limit for it to grow up to 10G in size.
   -  `name-here` is whatever you want to name the VM. Multipass will download a minimal Ubuntu image, and set up the VM.
3. Once done run `multipass shell name-here`. This command opens up a command-line instance for the Ubuntu VM. Terminal should say something like `ubuntu@name-here` or whatever you named the VM now.
4. run `sudo apt update && sudo apt upgrade -y`. This will update all the included packages and repo sources for the VM and allow you to properly install some more packages.
   - enter `y` and press enter if you are prompted to. Once this is done, the VM may prompt you to restart some services, just hit enter to any prompts until you see the black terminal window again.
5. exit out of the Ubuntu shell using `exit`. You should now be back into the macOS terminal (Terminal should be showing something like `yourmacusername@yourmacname`.
6. Run `multipass restart name-here` to restart your VM.
7. once it is done restarting, open another shell with `multipass shell name-here`.
8. run `sudo passwd root`, and change root's password to something you will remember.
   - Ubuntu disables root's password by default, but we are going to log in as root later to avoid issues with accessing directories and getting GUIs like Vorta to execute borg commands correctly, so this is necessary.
9. We're now going to set up a GUI for the Ubuntu VM per [Multipass's docs](https://multipass.run/docs/set-up-a-graphical-interface).
    - Run `sudo apt-get install lxde`. enter `y` if prompted.
    - [LXDE](https://help.ubuntu.com/community/LXDE) is the most minimal desktop environment for ubuntu I could find, but if you have a preference on DE's, install whatever you want instead.
10. once LXDE is finished installing, run `sudo apt install xrdp`.
    - XRDP lets ususe an app like Microsoft Remote Desktop to view our VM's desktop.
11. Exit out of the Ubuntu shell again using `exit`. You should be at the mac terminal again.
12. Run `multipass restart name-here` to restart your VM again.

### Step 2: Microsoft Remote Desktop Setup
1. Download and install [Microsoft Remote Desktop](https://apps.apple.com/us/app/microsoft-remote-desktop/id1295203466?mt=12) from the App Store.
2. Once done, open it up and add a New PC from the menu bar by going to Connections > Add PC. This should bring up a dialog box of some info we need to fill out.
3. in Terminal, run `multipass list`.
   - This lists all the VMs you have created. Copy what you see under the IPv4 column for the instance you created earlier.
   - Paste this IP in the `PC Name` field in the dialog box.
4. In the `User account` dropdown, hit Add User Account
   - in the Username field, put `root`
   - in the Password field, put the password you set earlier.
   - Hit Add
5. (optional) Under Display, untick Start session in full screen
6. Hit add
7. Double click the new box for the PC you added, and if any prompts come up just hit okay/connect.
   - A window should open up and you should see the Ubuntu LXDE desktop we configured.

It's important to use root to login just to avoid any weird permission issues with accessing directories, mounting, using GUIs like Vorta, etc. 

# Mounting and Browsing Backups
We now have an really minimal Linux VM to use as a GUI to browse Restic/Borg repos. It starts and stops really quickly, and is light on memory, CPU and disk space usage. 

## Configuring a directory share
We are using a shared directory between the VM and macOS to transfer data out of the Borg/Restic repo. By default, Multipass uses SSHFS for this, but you can configure it to use different protocols by looking [here](https://multipass.run/docs/improve-mount-performance). 

Following the [Multipass Share Data docs](https://multipass.run/docs/share-data-with-an-instance):

In macOS, open Terminal and run `multipass mount /path/to/folder name-here:/some/path/`
   - `/path/to/folder` is a the path to a directory on macOS that you want to transfer files to. Example: a folder on your macOS desktop called 'testdir', then /path/to/folder would be `Desktop/testdir`
   - `name-here:/some/path` is the path *inside* the VM that you want the folder to show up. example: folder on root's desktop called 'testdir', then /some/path/ would be `/root/Desktop/testdir`
   - so for example: I would use `multipass mount Desktop/testdir name-here:/root/Desktop/testdir` to make a directory share between the VM and macOS using the folders given as examples above.
  
This directory will automatically be shared between macOS and the VM every time the VM is started now. You can stop this by it using `multipass unmount name-here` (removes all shared directories) or add more directories using the same method described above. See [Multipass Share Data docs](https://multipass.run/docs/share-data-with-an-instance) for more info.

## Mounting and Browsing backups
This will be done within the VM. 

For restic, see their docs for `restic mount` [here](https://restic.readthedocs.io/en/latest/050_restore.html).

For Borg, see their docs for `borg mount` [here](https://borgbackup.readthedocs.io/en/stable/usage/mount.html). You can also use Vorta (see [here](https://vorta.borgbase.com/).

## Restoring/Moving files
If you are trying to just retrieve a few files out of your restic/borg backup, mounting within the VM and moving your desired files to the shared directory works fine.

For large/complete restores, see `restic restore` documentation [here](https://restic.readthedocs.io/en/latest/050_restore.html) or borg's restore options documentation [here](https://docs.borgbase.com/restore/borg/). 

For either case, point the output path to the shared directory you set up earlier. These files will appear in the macOS path you specified earlier, and you can copy them to whereever you want after they are done populating.

## Starting and Stopping the VM Manually or using Automator 
### Manually
- start the VM from terminal using `multipass start name-here`.
- stop the VM from terminal using `multipass stop name-here`.
### Using Automator to make an Application
Automator is a built in macOS application that allows you to make 'applications' that automate a bunch of things. You can take advantage of this by creating one that automatically runs the start command and opens up Microsoft Remote Desktop. 
Example for starting VM:
<img width="608" alt="image" src="https://github.com/fleetwoodmac/FUSEless-Mount-macOS/assets/69140036/0d2e65b8-0cfe-4486-a87f-eed4100fdc4b">

You can go further and create a separate application for stopping the VM and closing Microsoft Remote Desktop, and assign both to keyboard shortcuts, see [here](https://appleinsider.com/articles/18/03/14/how-to-create-keyboard-shortcuts-to-launch-apps-in-macos-using-automator) for an example.


# Closing Notes
This was the most straightforward way I could come up with to achieve what I wanted with on macOS. This was inspired by this [post](https://forum.restic.net/t/restic-mount-on-osx-and-windows-in-2021/4487/3). Things that could be done to improve this include setting up a BASH script to do much of the setup autimatically, or some of the alternatives/tweaks listed below.
- This could potentially be even lighter-weight if you are more comfortable doing everything without a GUI. Skip the LXDE and RDP stuff, and just do everything from CLI!
- [Macpine](https://github.com/beringresearch/macpine) is another alternative that uses alpine linux VMs instead of Ubuntu, which has the potential to be even lighter on CPU, RAM, and disk usage.
- Docker using bind mounts(?) may be another way to achieve this. See [restic](https://forum.restic.net/t/cannot-mount-on-macos-with-macfuse-4-x/3338/5). Could be adapted for Borg by someone motivated.
