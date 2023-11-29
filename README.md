# FUSEless-Mount-macOS
Workaround method for mounting Borg or Restic repos in macOS without FUSE.

# Preliminary notes.
Restic and BorgBackup are probably the most stable and well-developed FOSS backup tools for macOS that have a mounting capability for backup browsing. However, both use macFUSE for macOS mounting, which requires reducing security due to its system kext requirement. Until Borg and restic implement an kextless alternative like FUSE-T, or implement an alternative like webDAV a-la Cryptomator, Kopia is a great alternative if you are willing to use something that is still in active development (but by most accounts, completely fine, and being commercially supported/deployed), as it supports webDAV mounting of snapshots. 

For those that do still want to use Restic or Borg and do not want to change the required security settings needed to make macFUSE work on apple silicon (M1, etc.), the easiest ways to do so are using things like ResticBrowser and Borg-Repo-Explorer, which are viable ways to browse backups with minimal fuss, but are not a full solution to be able to search quickly and preview files yet. 

This hacky workaround uses a lightweight Linux VM to handle mounting and browsing backups virtually, then transferring files using a VM-Host shared directory.

# Method

## Step 1: Multipass
1. Download and install [Multipass](https://multipass.run/download/macos) package. Multipass is a super simple CLI tool by Canonical (makers of Ubuntu) that lets you spin up really lightweight ubuntu VMs on macOS.
2. Open Terminal, and create an ubunutu VM using `multipass launch --disk 10G --name name-here`.`10G` is the disk size you are alloting for the VM, and can be changed to anything. I suggest 10G just in case, but it will not take 10G of space immediately and just sets the limit for it to grow up to 10G in size.  `name=here` is whatever you want to name the VM. Multipass will download a minimal Ubuntu image, and set up the VM.
3. Once done run `multipass shell name-here` where `name-here` is whatever you named the VM like `borg-vm`. This command opens up a command-line instance for the VM. Terminal should say something like `ubuntu@name-here` or whatever you named the VM now.
4. run `sudo apt update && sudo apt upgrade -y`. This will update all the included packages and repo sources for the VM and allow you to properly install some more packages. enter `y` and press enter if you are prompted to. Once this is done, the VM may prompt you to restart some services, just hit enter to any prompts until you see the black terminal window again.
5. exit out of the shell using `exit`. You should now be back into the macOS terminal (Terminal should be showing something like `yourmacusername@yourmacname`. Run `multipass restart name-here` to restart your VM.
6. once it is done restarting, open another shell with `multipass shell name-here`.
7. run `sudo passwd root`, and change root's password to something you will remember. Ubuntu disables root's password by default, but we are going to log in as root later to avoid issues with accessing directories and getting GUIs like Vorta to execute borg commands correctly, so this is necessary.
8. We're now going to set up a GUI for the Ubuntu VM per [Multipass's docs](https://multipass.run/docs/set-up-a-graphical-interface). run `sudo apt-get install lxde`. enter `y` if prompted. LXDE is the most minimal desktop environment for ubuntu I could find, but if you have a preference on DE's, install whatever you want instead.
9. once done, run `sudo apt install xrdp` next. This is so we can use an app like Microsoft Remote Desktop to actually be able to see our VM's desktop.
10. Download and install [Microsoft Remote Desktop](https://apps.apple.com/us/app/microsoft-remote-desktop/id1295203466?mt=12) from the App Store.
11. will continue updating
