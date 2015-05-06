# pxeBoot

A collection of scripts and config files that makes installing
Debian based distributions easy and fun using preseeding!

Maybe that's overselling it a bit - if you need to install or
test the install procedure for multiple varieties of Debian based
distributions, or if you have to provision the same distribution
on multiple computers on your network, this framework is worth looking
at.

I developed it for automating the install of VirtualBox VMs on a 
single host using VirtualBox's built-in pxeboot and TFTP server, but
it will work on a real network just as well.

The scripts will probably not work on Windows or Mac machines, feel
free to update them and send me pull requests for these platforms.

Setting up a Debian based distribution for hands-free network installs
is a n step process.

1. Create the `netboot`able installer files
1. Create the `netboot` compatible boot scripts
1. Mount the installer, boot scripts, and ISO install media where 
   your web server will serve them up
1. Configure your TFTP server and web server
1. Optionally create the VM that is the target machine
1. Fire up your target machine

You'll need the following Linux skills:

- Basic command line familiarity
- Able to set up and configure a simple webserver with virtual hosts
- Able to set up a TFTP server and DHCP server for pxeboot or ...
- Some familiarity with VirtualBox and the built-in TFTP and DHCP servers

I'm assuming that if you are doing this, that you'll know where to
get the ISO images for the Debian based distribution you want to install.

Here's what a typical session looks like after you have chosen your
distribution. These first steps are done once for each distribution variant.

```
sudo -u pxeboot ./makePxeBootDirs

./makePxeBootInstaller ubuntu trusty amd64
./makePxeBootScripts   ubuntu trusty amd64
```

For each VM you want to create after that, you do this:

```
./mountPxeBootMedia    ubuntu trusty amd64

./makePxeBootableVM    ubuntu trusty amd64 template-trusty64-amd64
./installPxeBootableVM ubuntu trusty amd64 template-trusty64-amd64
```

NOTE: Not all of the distribution specific folders are fully up to date. For
now, only `ubuntu/trusty/amd64` has been fully tested!

## Getting The Lay Of The Land

The convention for Debian is to put locally served content under
`/srv/` and we're going to do the same thing. To make the process
a bit more general and to allow automation later, we'll create a 
new system user called `pxeboot` and add ourselves to the group:

```
sudo adduser --system --disabled-password --disabled-login \
             --home /srv/pxeboot \
             --group pxeboot
sudo adduser username pxeboot
```

You'll need to log out of your desktop session and back in to make the
group membership change take effect.

Once that's done, we can run another script to set up the
folder structure that will be used for the rest of the scripts
in the `pxeBoot` family. The folders are created under `/srv/pxeboot`
and will be writeable by members of the `pxeboot` group - and
we just made ourselves part of that group.

```
sudo -u pxeboot ./makePxeBootDirs
```

Now we can go ahead and run the rest of the scripts as ourselves -
no need to become the `pxeboot` user.

Each Debian based distribution has a slightly different
`netboot`able kernel file, a special release based folder
naming convention for packages, a set of supported architectures,
and so on. Rather than design a script to work around all these
differences, the `pxeboot` scripts use config files so that
you can customize your installs within some guidelines.

The example here is for an Ubuntu Trusty amd64 install that will be
used to develop the [ev3dev][ev3dev] kernel and SD card image for
the [LEGO MINDSTORMS EV3][LegoEV3].

You'll note that the previous sentence has 4 components that uniquely
identify the installation you want to perform:

1. Distribution - `Ubuntu`
1. Release - `Trusty`
1. Architecture - `amd64`
1. Machine - `template-trusty-amd64`

Your machine is the unique purpose for the installation - the framework
makes it relatively easy to configure a target machine for multiple 
distributions or architectures. You can also focus on one distribution,
release, and architecture and then create multiple custom machines.

The `pxeboot` user's default `HOME` directory is set to `/srv/pxeboot` so
all of the files we generate and serve up to boot the computer we are
installing Debian on will live there.

Your user's local `pxeBoot` folder is where the configuration files
for your Debian installations will live.

The structure is consistent among the different high level
folders, it goes:

```
path/to/folder/distribution/release/architecture/machine
```

The high level folders include:

- `/path/to/pxeboot` - your local config files
- `/srv/pxeboot/download` - the downloaded `netboot.tar.gz` files
- `/srv/pxeboot/pxeboot` - the expanded `netboot.tar.gz`filess
- `/srv/pxeboot/iso` - the ISO images for the distro
- `/srv/pxeboot/tftp` - the mount point for the tftp netboot files
- `/srv/pxeboot/www` - the mount point for the netboot installer files

The folder structure is also used as the parameter list 
for the scripts. All of the examples here assume that
we're installing Ubuntu Trusty amd64 - change the parameters
as needed for your install.

## Create The netbootable Installer Files

The first step is to download the appropriate `netboot.tar.gz` file 
for your distribution, release, and architecture. Then expand
the `netboot.tar.gz` file into a corresponding folder.

This script uses the `pax` utility, so if you don;t have it yet
then:

```
sudo apt-get --no-install-recommends install pax
```

The folder containing these files will later be mounted at another
location so that your TFTP server can find them.

```
./makePxeBootInstaller ubuntu trusty amd64 
```

With a typical broadband connection, this script will take
about a minute to downlod the installer, and then it will expand
the file.

## Create The Customized Boot Scripts 

Next we modify some of the files in the `installer/` folder that
we just created. The files will be backed up first if backups
don't already exist. That makes it safe to run this
script multiple times. The files are modified each time the
script is run.

```
./makePxeBootScripts ubuntu trusty amd64
```

This takes only a few seconds.

## Mount the Installer, Boot Scripts, and ISO Image(s)

As mentioned earlier, I'm assuming that you've somehow
downloaded the installer ISO image for your distro. While
you can download more CD/DVD images to get a more complete
set of packages, they get out of date pretty
quickly. It's better to create a partial repository
mirror with the [`growrepo`][growrepo] scripts.

The original plan was to use `libguestfs-tools` to allow
users to mount the ISO files without needing `sudo` - we
even managed to get things working so that other users could
view the mounted files. But `guestmount` is a Hypervisor
only one Hypervisor could be running at a time on my
host machine, which means VirtualBox would not start.

So, long story short, we're back to needing `sudo` but we
only activate it for the `mount` and `umount` operations.

Each installer ISO should be saved to the `/srv/pxeboot/iso/`
folder in the appropriate sub folder before running the script.

For Ubuntu Trusty amd64, the iso would be located at:

```
/srv/pxeboot/iso/ubuntu/trusty/amd64/ubuntu-14.04-server-amd64.iso
```

Note that you should not have more than one ISO in this
folder - if you do it's not at all clear which one will get
mounted.

```
./mountPxeBootMedia ubuntu trusty amd64
```

The default setup scripts give members of the `pxeboot` group
write access to `/srv/pxeboot/www` so that we can easily create
mount points there.

Give it a few seconds to mount the file, the console messages
will tell you when it's done.

## Configure Your TFTP Server and Web Server

I'm not going to explain how to configure your TFTP server here, there
are just too many options. But I will describe how VirtualBox serves
up pxeboot files from its internal NAT networking DHCP and TFTP servers.

VirtualBox has a neat feature that most folks don't even know about, so
let me congratulate the engineers there for putting DHCP and
TFTP servers into the product so cleverly that they don't interfere with
your own network.

These two services work on the NAT interface that is set up on all the VMs
that I create. The stanza in the `createVM` script looks like this:
 
```
vmNATPrefix="/srv/pxeboot/pxeboot/ubuntu/trusty/amd64/"
vmNATBootFile="pxelinux.0"

VBoxManage modifyvm "$vmName" --nic4 nat                    \
                              --nicbootprio4 1              \
                              --natnet4 10.0.2/24           \
                              --nattftpprefix4 $vmNATPrefix \
                              --nattftpfile4 $vmNATBootFile
```

This is where we tell the VM where it wants to look for the
pxeboot executable.

We'll go over how to run the script to create the VM shortly.

The web server I like best is [Hiawatha][Hiawatha] - it's got a small
footprint, it's secure, and it's easy to configure.

It's not needed for the TFTP boot portion of the process, but it is
needed after the pxeboot kernel loads so that it knows where to get the
`preseed.cfg` file. Remember the boot scripts we modified earlier? That's
where we tell the pxeboot kernel where to load the `preseed.cfg` file
from. The modified pxeboot script has a kernel boot config string
with the following added to it:

```
url=http://192.168.56.1:8080/preseed.cfg`
```

FIXME: Figure out if this can be fudged with a name we can resolve.

## Virtual Machine Setup

The nice thing about VirtualBox is that it can be scripted, so you'll
never have to point/click to assemble a VM ever again. To avoid
having to back up the VMs by default in the current user's `$HOME`
directory, all VMs will be created in the `/srv/vboxusers/` directory.

```
sudo mkdir                /srv/vboxusers
sudo chown root:vboxusers /srv/vboxusers
sudo chmod 775            /srv/vboxusers
```

We'll update it with `775` permissions and give ownership to
`root:vboxusers` so that any member of `vboxusers` can create
their own directory structure and store VMs there.

That way we can back up only the VMs that we really want to.

I use this script to set up DHCP on the two hostonly network
interfaces I generally configure whenever I set up VirtualBox:

```
VBoxManage hostonlyif create ipconfig "vboxnet0" --ip 192.168.56.1
VBoxManage dhcpserver remove --ifname "vboxnet0"
VBoxManage dhcpserver add    --ifname "vboxnet0" --ip 192.168.56.2       \
                                                 --netmask 255.255.255.0 \
                                                 --lowerip 192.168.56.100\
                                                 --upperip 192.168.56.115
VBoxManage dhcpserver modify --ifname "vboxnet0" --enable

VBoxManage hostonlyif create ipconfig "vboxnet1" --ip 10.0.2.1
VBoxManage dhcpserver remove --ifname "vboxnet1" 
VBoxManage dhcpserver add    --ifname "vboxnet1" --ip 10.0.2.2           \
                                                 --netmask 255.255.255.0 \
                                                 --lowerip 10.0.2.100    \
                                                 --upperip 10.0.2.115
VBoxManage dhcpserver modify --ifname "vboxnet1" --enable
```

Now let's create our first VM!

That's as simple as doing this from where your `pxeBoot` scripts live:

```
./makePxeBootableVM ubuntu trusty amd64 template-trusty-amd64
```

Followed by this to actually run the self-installing VM:

```
./installPxeVM ubuntu trusty amd64 template-trusty-amd64
```

That's all there is to it!

