# How-to : Crashplan & Freenas

Pre-requisites

* [Crashplan account (free)](http://www.crashplan.com/)
* FreeNAS-9.2.1-RELEASE-x64

## Install

### Step 1: Install the Crashplan plugin version 3.5.3_1

Plugins --> Install Crashplan
![Crashplan plugin](p1.png)

![Installing plugin](p2.png)

### Step 2 : Accept TOS
![TOS](p3.png)

### Step 3 : Enable Crashplan plugin
![Turn on service](p4.png)

### Step 4 : Create a sshd user for the Crashplan jail, enable TCP forwarding

To configure the Crashplan running inside a FreeNAS jail we need to use another machine that has graphical user interface. The jail has no user interface, only text console. Below is an overview of the connection that lets us use another machines GUI to configure the server running inside headless jail.

![Overview of the connection](overview.png)

To login to jail when no sshd is available in it you must either a) use jexec command on FreeNAS console or b) use the jail console in FreeNAS WebUI.

a) Connect to jail. If you are on FreeNAS login. Note that the JID may be different.
```
[root@freenas] ~# jls
   JID  IP Address      Hostname                      Path
     1  -               crashplan_1                   /mnt/zpool/jails_2/crashplan_1
[root@freenas] ~# jexec 1 /bin/tcsh
```

b) Open terminal to _jail_ from WebUI
![Edit jail:/etc/rc.conf](p4.png)

Enable the sshd. The instructions below are taken from [the FreeNAS wiki](http://doc.freenas.org/index.php/Adding_Jails#Accessing_the_Command_Line_of_a_Jail)

Edit /etc/rc.conf
```
...
sshd_enable="YES"
...
```
Create User for ssh-access

Create a new user, note that the user needs to be in group `wheel`.
```
root@crashplan_1:/ # adduser
Username: crashplan
Full name: 
Uid (Leave empty for default): 1001
Login group [crashplan]: 
Login group is crashplan. Invite crashplan into other groups? []: wheel
Login class [default]: 
Shell (sh csh tcsh nologin) [sh]: tcsh
Home directory [/home/crashplan]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: 
Use an empty password? (yes/no) [no]: 
Use a random password? (yes/no) [no]: 
Enter password: 
Enter password again: 
Lock out the account after creation? [no]: 
Username   : crashplan
Password   : *****
Full Name  : 
Uid        : 1001
Class      : 
Groups     : crashplan wheel
Home       : /home/crashplan
Home Mode  : 
Shell      : /bin/tcsh
Locked     : no
OK? (yes/no): yes
pw: mkdir(/home/crashplan): No such file or directory
adduser: INFO: Successfully added (crashplan) to the user database.
Add another user? (yes/no): no
Goodbye!
```

Start the sshd manually this once:

```
root@crashplan_1:/ # service sshd start
...
```

At this point, I like to copy my pub key to make things easier on me. The 192.168.1.103 here is the IP of the crashplan jail.
You must first find the IP of the jail itself (not the FreeNAS box).  You can find this in the Jails menu.  In this case the instructions show it as `192.168.1.103`:

```
[me@mylaptop] ~$ brew install ssh-copy-id
[me@mylaptop] ~$ ssh-copy-id crashplan@192.168.1.103
```

Now, let's create a tunnel. This will redirect my laptop port 4200 to port 4243 on the crashplan jail. The console will stay open, you may close it when you are done with the step 7.

```
[me@mylaptop] ~$ ssh -L 4200:127.0.0.1:4243 crashplan@192.168.1.103 -N -v -v
```

I prefer to use a script that I can run.  Add a file called `/usr/local/bin/crashplan` with the following content:

```sh
#!/bin/bash

# Port forward to FreeNAS for Crashplan plugin

SERVICE_PORT=`cat /Applications/CrashPlan.app/Contents/Resources/Java/conf/ui.properties | grep servicePort | egrep -v '#' | awk -F "=" '{print $2}'`

CRASHPLAN_USER="crashplan"
CRASHPLAN_JAIL=192.168.1.2
echo "Connecting on Service Port: $SERVICE_PORT"
echo "Expecting Crashplan FreeNAS Jail at: $CRASHPLAN_USER@$CRASHPLAN_JAIL"

# Connect to crashplan
ssh -L $SERVICE_PORT:127.0.0.1:4243 $CRASHPLAN_USER@$CRASHPLAN_JAIL -N
```


### Step 5 : Configure Crashplan UI to connect to remote host (through ssh-tunnel)

This is done _on your desktop machine_ (for me it was my laptop), that you use to configure the crashplan service running in the FreeNAS jail.

See [crashplan's documentation](http://support.crashplan.com/doku.php/how_to/configure_a_headless_client)

Configure the desktop (my laptop) to use the ssh tunnel by editing the `ui.properties` file.
Make a backup of the file, if you use the Crashplan on the desktop machine too. This change will divert the
connections to port 4200, so it will not work without the ssh-tunnel and it will not connec to local crashplan
any more.


You may find the `ui.properties` file in the following location:

```
Linux (if installed as root): /usr/local/crashplan/conf/ui.properties
Mac: /Applications/CrashPlan.app/Contents/Resources/Java/conf/ui.properties
Solaris (if installed as root): /opt/sfw/crashplan/conf/ui.properties
Windows: C:\Program Files\CrashPlan\conf\ui.properties
```

Change the service port to 4200, which we will use to tunnel to the remote connection.

```
servicePort=4200
```

### Step 6 : Verify Crashplan is running and listening

```
[root@freenas] ~# jexec crashplan_1 sockstat -4
USER     COMMAND    PID   FD PROTO  LOCAL ADDRESS         FOREIGN ADDRESS
crashplan sshd      4149  5  tcp4   192.168.1.103:22      192.168.1.83:53226
root     sshd       4147  5  tcp4   192.168.1.103:22      192.168.1.83:53226
root     java       3952  56 tcp4   127.0.0.1:4243        *:*
root     java       3952  57 tcp4   *:4242                *:*
root     java       3951  56 tcp4   127.0.0.1:4243        *:*
...
```

### Step 7: Mount storage directories

From the FreeNAS GUI you must configure the storage for the jail so that you can back up the filesystem.  This can be done if you go to:

```
Jails -> crashplan_1 -> Storage -> Add Storage
```

You must put in the root `Source` and `Destination` directory and set it to `Read-Only`.  The `Source` is the directory you wish to back up.  The `Destination` is the directory you want to mount it on so Crashplan can read it.  You select `Read-Only` to secure the data against anything in the jail trying to modify your files.  Here is an example:

![Crashplan Storage](p5.png)

### Step 8: Connect with Crashplan UI

First download and install the crashplan desktop application.  Make sure the client (desktop application) is the same version as the plugin.  In our case we want both to be 3.6.X.

https://www.code42.com/crashplan/download/

Launch the modified Crashplan UI on the desktop (my laptop). Ssh-tunnel must be open. Login and configure. Quit UI and enjoy versioned backups to and from your FreeNAS.

You may close the ssh-tunnel at this point when the Crashplan UI is closed.

## Common Problems

### Update to 3.6.3

After update to 3.6.3 (happens automatically, pushed from Crashplan) the service fails to start. Thanks to mstinaff thread: http://forums.freenas.org/index.php?threads/crashplan-3-6-3.18416/ we now have a working solution to this.

You need to manually update a file.  First make a backup:

```sh
root@crashplan_1:/ # cd /usr/pbi/crashplan-amd64/share/crashplan/bin
root@crashplan_1:/usr/pbi/crashplan-amd64/share/crashplan/bin # cp -v run.conf run.conf.bkp
```

The original file for `/usr/pbi/crashplan-amd64/share/crashplan/bin/run.conf` looks like this:

```
SRV_JAVA_OPTS="-Dfile.encoding=UTF-8 -Dapp=CrashPlanService -DappBaseName=CrashPlan -Xms20m -Xmx1024m -Djava.net.preferIPv4Stack=true -Dsun.net.inetaddr.ttl=300 -Dnetworkaddress.cache.ttl=300 -Dsun.net.inetaddr.negative.ttl=0 -Dnetworkaddress.cache.negative.ttl=0 -Dc42.native.md5.enabled=false"
GUI_JAVA_OPTS="-Dfile.encoding=UTF-8 -Dapp=CrashPlanDesktop -DappBaseName=CrashPlan -Xms20m -Xmx512m -Djava.net.preferIPv4Stack=true -Dsun.net.inetaddr.ttl=300 -Dnetworkaddress.cache.ttl=300 -Dsun.net.inetaddr.negative.ttl=0 -Dnetworkaddress.cache.negative.ttl=0 -Dc42.native.md5.enabled=false"
```

You will need to update it to look like this:

```
SRV_JAVA_OPTS="-Djava.nio.channels.spi.SelectorProvider=sun.nio.ch.PollSelectorProvider -Dfile.encoding=UTF-8 -Dapp=CrashPlanService -DappBaseName=CrashPlan -Xms20m -Xmx1024m -Djava.net.preferIPv4Stack=true -Dsun.net.inetaddr.ttl=300 -Dnetworkaddress.cache.ttl=300 -Dsun.net.inetaddr.negative.ttl=0 -Dnetworkaddress.cache.negative.ttl=0 -Dc42.native.md5.enabled=false"
GUI_JAVA_OPTS="-Djava.nio.channels.spi.SelectorProvider=sun.nio.ch.PollSelectorProvider -Dfile.encoding=UTF-8 -Dapp=CrashPlanDesktop -DappBaseName=CrashPlan -Xms20m -Xmx512m -Djava.net.preferIPv4Stack=true -Dsun.net.inetaddr.ttl=300 -Dnetworkaddress.cache.ttl=300 -Dsun.net.inetaddr.negative.ttl=0 -Dnetworkaddress.cache.negative.ttl=0 -Dc42.native.md5.enabled=false"
```

Finally, you need to stop and start the jail.  Without doing this the plugin will not start correctly.

### Cannot connect to crashplan jail over ssh

Sometimes the `sshd` service needs to be restarted.  This is easy:

```sh
[root@freenas] ~# jls
   JID  IP Address      Hostname                      Path
     1  -               crashplan_1                   /mnt/zpool/jails_2/crashplan_1
[root@freenas] ~# jexec 1 /bin/tcsh
root@crashplan_1:/ # service sshd restart
```
