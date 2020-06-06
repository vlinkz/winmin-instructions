# Winmin Prerelease Setup

## Background

Winmin (working title) is a set of tools and scripts that facilitate using Windows applications on Linux using a kvm virtual machines and libvirt. Winmin was originally inspired by a [tweet](https://twitter.com/unixterminal/status/1255919797692440578) from [Hayden Barnes](https://twitter.com/unixterminal) in which he demonstraited running Microsoft Word on Ubuntu 20.04. Since then I have worked to replicate this setup and make various tools that make using it easier.

## Preparation

Install packages and setup groups

```
$ sudo apt -y install spice-client-gtk libvirt-clients libvirt-daemon-system virtinst python3 python-is-python3 python3-magic samba

$ sudo usermod -aG kvm,tty $USER
```
You may need to logout or reboot in order for group changes to take effect. You can check your current groups with the `groups` command.

In order to transfer files between the host and guest, a samba server must be set up. To set this up, add the following to the end of `/etc/samba/smb.conf`.

```
[winmin]
  comment = VM Filesystem
  path = /
  browseable = yes
  read only = no
```

After changing the samba config, add an smb account for your user.
```
$ sudo smbpasswd -a $USER
```

Then restart the samba service.

```
$ sudo systemctl restart smbd
```
Enable the service if not automatically enabled
```
$ sudo systemctl enable smbd
```
Set default libvirt uri to system.
```
$ echo "export LIBVIRT_DEFAULT_URI=\"qemu:///system\"" >> $HOME/.bashrc
$ source $HOME/.bashrc
```


# Setting up Winmin

Download the latest deb packages for each winmin component in the releases section of its repository.

- [winmin-viewer](https://github.com/vlinkz/winmin-viewer/releases)
- [winmin-tools](https://github.com/vlinkz/winmin-tools/releases)
- [winmin-setup](https://github.com/vlinkz/winmin-setup/releases)

Once downloaded, install them with dpkg

```
$ sudo dpkg -i winmin-viewer_0.1-1_amd64.deb winmin-tools_0.1-1_amd64.deb winmin-setup_0.1-1_amd64.deb pywinminsetup_0.1-1_amd64.deb
```
If you are not using a Ubuntu based distribution, you can build and install the packages manually as described in each repository's readme.

Run the winmin setup application from the command line. You will need to input a Windows 10 iso (May 2020 update or later only). 
```
$ winmin-setup
```
After selecting the ISO and clicking next, the application will appear to freeze. It is NOT frozen, it is downloading the virtio-win guest iso (I'll fix the loading screen soon).

Follow the instructions in the application. Make sure you install the virtio-win guest tools and run the script specified in the instruction. Also make sure to reboot at least once before finishing (it says when to do it in the instructions).

Once finished, head over to the [winmin-pkgs](https://github.com/vlinkz/winmin-pkgs) repo to find yaml packages. Feel free to submit PRs with new packages or request them in the issues section.

# Installing Packages

To install a package, use the `winmin-yml-install` command or the `winmin-install` command. See the [winmin-tools](https://github.com/vlinkz/winmin-tools) repo for details on how to install packages.

__IMPORTANT__: When installing an application, before closing the installation window, RUN THE PROGRAM(S) YOU INTEND TO USE AT LEAST ONCE. Currently both the ms-office and visualstudio packages are set to destroy the vm (revert back to the previous save state) on exit, this means any configuration changes made to the application will not be saved between uses. This can be changed by setting the `--save` flag in each application's `.desktop` file, however it is not set by default as closing an application will take around 10-15 seconds. Also, before exiting the installer CLOSE ALL WINDOWS INCLUDING THE INSTALLER. This will probably be automated later when I get around to it.

__IMPORTANT__: Currently, running two applications from the same package (or VM) at once is "impossible". For example, if I have Word open, and a launch Excel, the Word window will freeze and the Excel will be launched on top of Word in the new window. When you close the frozen Word window, the VM will be saved or destroyed and the Excel window will freeze. I'm working on a fix using multiple monitors, stay tuned (see: [winmin-viewer-multimonitor](https://github.com/vlinkz/winmin-viewer/blob/master/winmin-viewer-multimonitor.c)).

# Issues

Please report an issues encountered in the repository of the respective tool that did not work.

If you have questions about the project as a whole, or any suggestions, you can create an issue on this repository or contact me on [twitter](https://twitter.com/VlinkZ3).

Note: Outlook sometimes complains about `C:\Users\VM\AppData\Local\Microsoft\Outlook\email@address.com.ost` being outdated. After installing ms-office, before closing the window, delete this file. I will probably add a prefix command that deletes this file soon.

# Development Setup

## Tools

To work on developing this set of tools, I highly recommend you install `virt-manager`. This tool allows you to manage and edit libvirt virtual machines.

If you shutdown or reboot a Winmin virtual machine, after booting, open the terminal and run 
```
$ virsh console winmin-{vm}

SAC> cmd

SAC> ch -sn Cmd001

C:\Windows\system32> ^]
```

This sets up the virtual machine in a state where it can receive commands.

# How Winmin works

## Basics

Winmin works by installing Windows applications into virtual machines, and then starting a virtual machine, executing the application, and then displaying it using a remote desktop viewer.

## In depth (WIP)

### Initial Setup

When running `winmin-setup`, a [python script](TODO) executes commands to create a libvirt virtual machine named `winmin-base`. After the user proceeds through the installation, this virtual is shut down and never directly used again. This machine works as a base image where all other virtual machine with applications installed are derived from. It is important to make this image as small and lightweight as possible in order to have good performance.

When setting up the vm, the [Windows Emergency Management Services and Serial Console](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc787940(v=ws.10)?redirectedfrom=MSDN) is installed. This allows Winmin tools to interact directly with the Windows guest's serial interface and execute commands quickly.

### Installing Applications

When an application is installed with either `winmin-yml-install` or `winmin-install`, a new virtual machine is created. A new virtual hard drive is created which uses the `winmin-base` image as its [backing file](https://libvirt.org/kbase/backing_chains.html). Using this technique, when an application is installed, the new disk only takes up as much additional space as the size of the application.

Once the user installs the application, the state of the vm is [saved](https://libvirt.org/docs/libvirt-appdev-guide-python/en-US/html/libvirt_application_development_guide_using_python-Guest_Domains-Lifecycle-Save.html). Every time a user launches the application, the save state is resumed and the application is executed. 

### Running Applications

When an application is started, it's virtual machine is started from the last save state. Commands are sent to the serial port and are executed in the guest. The SPICE socket is located and `winmin-viewer` is executed to display the application. Any file arguments are converted to their respective addresses on the host's samba server so that the guest can access and edit them. When an application window is closed, the virtual machine is either saved or destroyed (reverted back to its previous save state).
