# SGI Net Installation Server tool 'love' by TruHobbyist

To help preserve this amazing tool, I have decided to make my own mirror
and any additional notes or documentation that may help me use this 
software in the future. Love is a tool written in C++ which acts as 
three servers (bootpd, tftpd, rshd) in a single program, allowing you
to perform remote IRIX installations over your local network. Be sure
to run the love server on the same LAN as your SGI machine.

Love uses a `LABELS.<PLATFORM>.TXT` file (found in each platform's folder) to define
aliases for folders/files, making it simple to enter over on your SGI
machine. Instead of entering `192.168.1.60:/Very/Long/Folder/Path` on your SGI machine, love
instead uses these aliases to make entry concise: `192.168.1.60:love.53`

A guide is provided in the form of `love_guide.txt` originally written
by the author, TruHobbyist. Some of these instructions may be unclear
or grammatically incorrect, as TruHobbyist is a native German speaker.
I will provide a basic outline of what I did to install IRIX on my
SGI Indy. Note that the proceeding instructions are focused on creating
a Linux Mint VM for this task, but feel free to follow the guide as 
well. You can also find a separate guide provided by TruHobbyist on 
[their forum post at SGI User Group Forums](https://forums.sgi.sh/index.php?threads/love-install-irix-from-irix-linux-or-windows.949/). Their original
post details additional steps/dependencies necessary for running this
on Windows. Use Wayback Machine if the post disappears.

## Creating an IRIX Rescue VM

Personally I ran this server (love) in a Linux Mint VM using the pre-built ubuntu
binary found in `/debian-11.6.0`. I chose mint because it's light weight,
debian based, and has a simple setup. This brief guide will focus on this
setup, as it provides you with a completely isolated server that you can
spin up/down as needed as an IRIX recovery/rescue server.

### Virtual Box Steps

Using Virtual Box, create a new Virtual Machine (I just chose 'Ubuntu' in 
the menu) and select a mint ISO file. Make sure to provide ample disk 
space for your VM, as the IRIX distributions all take up roughly 10GB on 
disk. If you're prompted that media is missing when you boot it, just use
the toolbar menu to reselect the ISO and proceed with installing mint. 

Once installed, shut down the machine and edit your VM's settings. 
Go to 'Network' > 'Adapter 1' > and change `Attached to` to `Bridged Mode`,
selecting your machine's Ethernet port/Wi-Fi card. Change `Promiscuous Mode`
to `Allow All` (please educate yourself on what this mode does).

Next, you will need to install the VirtualBox guest additions CD for Ubuntu.
I recommend following a guide provided by VirtualBox, but as a quick 
reminder you can use the toolbar of your VM's window and select 'Devices' >
Insert Guest Additions CD or something along those lines. This step is 
technically optional if you expect to transfer the IRIX distribution files
over the internet. I needed guest additions to create a shared folder 
between my Windows host and the Mint guest VM. Follow the appropriate guide
for setting up a shared folder, it's very straightforward.

### Retrieve Necessary Files

You'll need to retrieve this repository over on your VM and place the
appropriate love binary somewhere familiar to you. Alternatively you
can compile the `love.cxx` file using the flags provided in the `ABOUT.txt`
file for your particular platform. Optionally if you'd like to execute love from 
any working directory in your terminal, then you'll of course need to 
add the location of your love binary to your `PATH` (please seek the
appropriate resources to educate yourself on this). I'm making the
assumption that you're likely educated using Linux/Unix given the
nature of this project. If not, it's only ten minutes worth of reading
and well worth your time.

In addition to the love binary, you will also need to retrieve the unpacked
versions of the IRIX installer discs per your desired version. TruHobbyist
provides a convenient tar file containing all versions mentioned in their
originally written `LABELS` file.  [You can find that tar file here.](https://drive.google.com/file/d/1GSsT4J_go8dH_ZrWFQlaLKdWtSsG6qyw/view) Alternatively you may source your
own copy of SGI CD's, but be aware that any ISO files you may find will 
need to be opened using an atypical procedure, as SGI was notorious for
packing their images in an unconventional manner. _Instructions for this 
will proceed this section if needed._

Once you've acquired your IRIX diststribution files, place them in `/mnt/IRIX/<VERSION_NUM>`
respectively according to the `LABELS.UNIX.TXT` file. See that each line in
the labels file is simply an alias defition in the convention of `<LONG_ALIAS> <FILE_PATH> <SHORT_ALIAS>`.
Use tab characters between each part if you want to create your own aliases in 
this file. I recommend taking some time and just reviewing what the labels file
looks like to give you a better understanding of what you'll be using or needing
to set up, it's very easy to follow.

### Unpacking an IRIX ISO

If you need to unpack an IRIX ISO file, you're probably already familiar
with most (if not all) programs recognizing these files as corrupt. _This simply
is not the case._ SGI merely imaged their discs in an unconventional way,
with the beginning of the image starting at a different point within the disk.
For this, you will need some tools only found on GNU/Linux machines. You may
be able to find these for other platforms, I'm just taking the easy route here
and using my Linux Mint VM as my toolbox. Windows does support WSL, so that's 
probably a fairly easy work around for Windows users.

Open a terminal window and use the following command to locate the start block of your ISO:
```bash
fdisk -l /path/to/file/irix_file.iso
```

In my case, I performed the command using an IRIX 5.3 iso file and received the following output:
```
Disk /mnt/shared/IRIX_5.3.iso: 511.59 MiB, 536444928 bytes, 1047744 sectors
Geometry: 255 heads, 63 sectors/track, 65 cylinders
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: sgi

Device                     Start     End Sectors   Size Id Type       Attrs
/mnt/shared/IRIX_5.3.iso8  60768 1047159  986392 481.6M  5 SGI sysv   
/mnt/shared/IRIX_5.3.iso9      0   60767   60768  29.7M  0 SGI volhdr 
/mnt/shared/IRIX_5.3.iso11     0 1047167 1047168 511.3M  6 SGI volume 

Partition table entries are not in disk order.
```

Take note of your sector size. If I'm not mistaken you will find
them all to be 512 bytes. This is indicated by:

```
Units: sectors of 1 * 512 = 512 bytes
```

Next, the line we're looking for in the table is (the largest part):
```
/mnt/shared/IRIX_5.3.iso8  60768 1047159  986392 481.6M  5 SGI sysv   
```

Using the `Start` value found for this line (in my case, 60768) and our sector size, run the following:
```bash
sudo mount -t efs -o loop,offset=$((60768*512)) /mnt/IRIX/irix_file.iso /mnt/IRIX/5.3
```

You may have to use the `mkdir` command (make directory) if you receive 
an error about a directory not existing. Create the directory and try
again. Once the `mount` command succeeds, you will have opened your 
IRIX ISO file at the `/mnt/` directory specified at the end of the 
`mount` command I demonstrated.

### Configure Ports

Make sure to allow network traffic across the following ports:
 - UDP 67 - Incoming
 - UDP 69 - Incoming
 - TCP 514 - Incoming
 - Any outgoing TCP connection from love.exe

On Ubuntu/Mint this can be acheived easily with:
```bash
sudo ufw enable
sudo ufw allow 67
sudo ufw allow 69
sudo ufw allow 514
```

### Run love

First be sure to give execution permissions to the love binary:
```bash
# Change directory to your directory containing love
cd /<YOUR_PATH_CONTAINING_LOVE>

# Grant execution permission to the love binary
sudo chmod +x ./love
```

Assuming you have the IRIX distribution you'd like to install at a 
directory aliased by `LABELS.UNIX.TXT` (in this case, `/mnt/IRIX/5.3/dist` for `love.53`)
you should be ready to run:

```bash
cd /<YOUR_PATH_CONTAINING_LOVE>
sudo ./love <VM_IP_ADDRESS> LABELS.UNIX.TXT
```

In my case, those commands looked like this:
```bash
cd ~/Desktop/love
sudo ./love 192.168.1.100 LABELS.UNIX.TXT
```

You should see lots of output about the label file being parsed, and finally
at the end you will see:
```
INFO: Listening for BOOTP packets
```

You are now ready to power on your SGI machine.

### SGI Machine Setup

If you used the aforementioned download link of TruHobbyist's IRIX tar file,
then your directory `/mnt/IRIX/` will contain the following folders: `5.3`, `6.0`,
`6.1`, `6.2`, `6.3`, `6.4`, `6.5`, `6.5.7`, `6.5.22`, `6.5.30`.

Power on your SGI machine. You may continously press the `ESC` key to enter 
the stop for maintenance menu, or click the button when it appears. Some CRT
monitors are sluggish, so `ESC` can help you stop the boot process and pause
for you to take action before it shows up on the monitor.

Use the option to enter the Command Monitor, and enter the following command:

```
unsetenv srvaddr
```

Next, enter the following and replace <IP_ADDR> with your desired IP address for your SGI:

```
setenv netaddr <IP_ADDR>
```

Finally, we should be able to boot the `fx.ARCS` command disk formatting utility
from the folder `/mnt/IRIX/6.5.30/stand/fx.ARCS` using the following command. You
will only be able to run `love.6530.fx` below if you have the folder 
`/mnt/IRIX/6.5.30` containing the folder `/stand` (standalone). If instead you 
would like to run it for instance with IRIX 5.3, you would enter `love.53.fx` instead.
It is recommended to use the `6.5.30` version of `fx`, as it is the latest one.
Keep in mind that the IP Address I am using is the IP Address of my Linux Mint VM
which is running love. I am passing the `--x` flag to `fx.ARCS` to enter extended
mode (recommended):

```
boot -f bootp()192.168.1.60:love.6530.fx --x
```

You will be asked a series of questions to select the SCSI disk you are going to 
format. Please be warned that the following instructions may take an incredibly 
long time, especially on disks above 8 Gigabytes. For better instructions on 
running this formatting tool, please see `love_guide.txt` and refer to the
instructions on formatting. Also be aware that the following command is 
destructive! You will lose ALL data on the drive we are formatting.

Again, this will delete ALL of your data. You have been warned!

In the following screen, `fx` prompts us to enter an option. Among these options
is the easiest, and least likely for you to mess up, which is the `[a]uto` option.
I prefer using `[a]uto`, as I'm working with smaller drives (2GB to 4GB in size).

To run the command, simply enter either `a` or `auto`, and follow the instructions.

After formatting is complete (roughly 30 minutes to 2 hours), you may exit the
command monitor by clicking `Done` in the bottom right, or type `exit` and press enter.

You are now ready to install IRIX. At the main screen of Stop for maintenance, you
should see an option to install software. Click this option. Choose to install from
a remote directory. You will be asked for an IP Address. Enter the IP Address of
your Linux Mint VM. You will now be prompted to enter the directory containing your
IRIX distribution. Enter one of the following depending on your desired version:\

**For IRIX 6.5+, you must follow the instructions found in `love_guide.txt` to 
better understand the process of installing** I will make some effort here to
briefly go over this, but please read the guide's section on installing 6.5+
as well in order to better understand what's going on here.

| IRIX Version    | Directory to Enter |
| --------------- | ------------------ |
| IRIX 5.3        | love.53            |
| IRIX 6.0        | love.60            |
| IRIX 6.1        | love.61            |
| IRIX 6.2        | love.62            |
| IRIX 6.3        | love.63            |
| IRIX 6.4        | love.64            |
| IRIX 6.5.7      | love.657.1         |
| IRIX 6.5.22     | love.6522.1        |
| IRIX 6.5.30     | love.6530.1        |

**Additionally:** I attempted installation of IRIX 6.0, 6.1, 6.2, 6.3, and 6.4,
only being able to install 6.2. I believe the files provided by TruHobbyist may
not have included the installer utility in 6.0, 6.1, 6.3, and 6.4. After 
installing 6.2, I did not have a desktop environment. For this reason, I formatted
my disk and moved to IRIX 6.5.22.

Enter the directory you would like to install, press enter, then Next.

You should now see your installation files being downloaded onto your SGI. There
will also be output coming from your terminal window on your Linux VM, showing
the commands that are being received and handled. Wait for this process to complete.

Once your installer has loaded, you will be provided with a step-by-step installer
that is fairly easy to follow. Miniroot will run, asking you to create a filesystem.
You will want to answer YES to this. If asked, use EFS for IRIX 5.3, XFS for 6.0+.
You may use EFS for 6.0+, but XFS was new at the time. If you are knowledgeable 
enough to disagree with me, then you do not need these steps. Suffice it to say,
I have heard that while XFS was new and offered improvements, it had some draw
backs. This is likely only going to affect the most hardcore users who already 
know these circumstances.

When asked about 512 byte or 4096 byte block sizing, you may choose either. Smaller
block sizes will offer better performance and reduced disk fragmentation for 
systems with many small files. For systems containing many large files, you may
choose 4096. Again, if you are aware of these performance implications, you likely
do not need my help here.

Lastly, you will be provided with a prompt that will list many commands:
- from
- open
- list
- go
- conflicts
- keep
..etc etc.

*For IRIX 5.3, 6.0, 6.1, 6.2, 6.3, 6.4:*
Simply enter `go` and press enter.

*For IRIX 6.5+:*
You must load all overlay CD's and applications. For IRIX 6.5.22, I performed the following:

__If at any point you are asked to pick between Maintenance and Feature streams, pick Feature Stream.
Likewise, if you are knowledgeable enough to know the difference, be my guest and pick Maintenance.__

After each `open` command, I am pressing enter. Load all of the following CDs:

`open 192.168.1.60:love.6522.1`\
`open 192.168.1.60:love.6522.2`\
`open 192.168.1.60:love.6522.3`\
`open 192.168.1.60:love.65.found1`\
`open 192.168.1.60:love.65.found2`\
`open 192.168.1.60:love.65.devfound`\
`open 192.168.1.60:love.65.devlib`\
`open 192.168.1.60:love.65.appsjune1998`\
`open 192.168.1.60:love.65.nfs3`

You may receive some scrolling prompts during this process. Use the spacebar to continue through
these prompts.

Lastly, execute the `go` command to begin installing. If you receive conflicts, please use the 
`conflicts` command to list the conflicts. You may enter your choices to conflicts with the following
example:

```
conflicts 1b 2b 3a 4a
```

Please keep in mind that this is merely an example of entering your options to resolve conflicts, 
and does not pertain to the 6.5.22 installation. For 6.5.22, I received no conflicts when loading
the disks in that order. For 6.5.30, `love_guide.txt` provides the necessary conflict resolutions
you will need.

After resolving conflicts, you may proceed again with the `go` command. Please be somewhat
attentitive to the long installation process, checking it every few minutes to ensure everything 
is working smoothly. The only hitch I ran into was while the installer was installing the 
software package `appletalk.sw` which I chose to `Skip`. After over 1 hour, the installation 
completed. When you choose to restart, ELF Requickstarting will be performed and will take an
additional 5 to 15 minutes to optimize binaries for your system.

That's all! If you require any additional help, feel free to report a problem with this repo to
contact me personally. Also please refer to `love_guide.txt` of course and the forums, as you'll
likely find what you need. 

I hope this guide provides you with the help you need to get your antique SGI computer back to 
life. Thanks for following along. ❤️

Thank you to TruHobbyist for this amazing tool. Without you, this would not have been possible.
