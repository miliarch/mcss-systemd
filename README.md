# Minecraft Server Scripts for systemd

This project draws inspiration from one of [justinjahn](https://gist.github.com/justinjahn)'s Gists: [Minecraft server(s) using systemd and screen](https://gist.github.com/justinjahn/4fe65b552b0622662420928cc8ffc7c0). I learned much from this Gist, and used the foundation to build a framework of sorts.

This project utilizes tmux instead of screen, provides a helper script for creation of new instances, and leverages a PID file for parent tmux session tracking by systemd services. I expect many improvements to the helper script to come, such as specification of JVM memory allocation and server port. For now though, it meets my needs, and additional features likely won't be an immediate focus.

# Requirements

## Minecraft Server Scripts for systemd
1. Administrator level access to a linux operating system that utilizes systemd for service management (CentOS 7 tested)
2. Local service user with name *minecraft* belonging to group *minecraft*
3. The project directory must be owned by the local service user
4. A copy of minecraft_server.jar in the project root directory ([download](https://www.minecraft.net/en-us/download/server))

## Prerequisites
In addition to the requirements above, the following programs must be installed and available to the *minecraft* user:
1. `java` installed and linked in `/usr/bin/java` (openjdk 1.8.0 JRE tested) 
2. `tmux` installed and linked in `/usr/bin/tmux` (tmux 1.8 tested)

# Installation and Configuration

This section was written to be modular. If you've already completed a particular step, feel free to skip to more relevant steps.

For the sake of simplicity, the project root directory in all steps is specified as `/opt/minecraft` (partial or complete path), and the server instance name is specified as `server1`. These references can be changed to suit your requirements.

## Ensure systemd is running and managing services

Check if systemd is installed on the system:
```
$ systemctl --version
systemd 219
```

If installed, check that it's running:
```
$ sudo systemctl status
● minecraft
    State: running
```

As far as advice on installing systemd ([what is systemd?](https://www.freedesktop.org/wiki/Software/systemd/)): I'd recommend using an operating system that employs it out of the box, such as CentOS 7 or Ubuntu 16.04+ if yours doesn't. I have no clue what it would take to switch to systemd from a different system daemon, but it strikes me as being rather involved.

## Ensure tmux is installed and available

Check if tmux is installed and available at `/usr/bin/tmux`:
```
$ /usr/bin/tmux -V
tmux 1.8
```

If the above command is not found, you need to install tmux ([what is tmux?](https://github.com/tmux/tmux)) to use these scripts.

On CentOS 7, the following command will install the tmux package:
```
$ sudo yum install tmux
```

## Ensure a Java 8 JRE is installed and available

Check if java is installed and available at `/usr/bin/java`:
```
$ /usr/bin/java -version
openjdk version "1.8.0_232"
OpenJDK Runtime Environment (build 1.8.0_232-b09)
OpenJDK 64-Bit Server VM (build 25.232-b09, mixed mode)
```

If the above command is not found, or the version is below [Minecraft system requirements](https://help.mojang.com/customer/portal/articles/325948-minecraft-system-requirements) (Java 8, equivalent to openjdk-1.8.0), you need to install Java to run Minecraft server.

On CentOS 7, the following command will install the OpenJDK 1.8.0 headless JRE package:
```
$ sudo yum install java-1.8.0-openjdk-headless
```

For other distributions, see the Minecraft Wiki page [Tutorials/Setting up a server](https://minecraft.gamepedia.com/Tutorials/Setting_up_a_server#Linux_instructions)

## Create service user

The Minecraft server will run as a particular user specific to the task. To create the user `minecraft` in group `minecraft`, issue:
```
$ sudo useradd -r -m -U -s /bin/bash -d /opt/minecraft minecraft
```

## Clone this project to your local filesystem

If you have `git` installed and available:
```
$ sudo -u minecraft git clone https://github.com/miliarch/mcss-systemd.git /opt/minecraft
```

On CentOS 7, the following command will install the git package:
```
$ sudo yum install git
```

If you don't have `git` installed, and can't install, you'll need to save [the repository archive](https://github.com/miliarch/mcss-systemd/archive/master.zip) to your filesystem and extract the contents to the project directory. If this concept is new to you, this [google search](https://www.google.com/search?q=linux+how+to+extract+zip) might help.

## Add the Minecraft server jar file to the project directory

You can find download information for the Java Edition Minecraft Server jar file on the official Minecraft site:
https://www.minecraft.net/en-us/download/server

Download the jar file to your local system, move it to the project root directory, and ensure it is named "minecraft_server.jar"

Note: The name "minecraft_server.jar" is **necessary** for scripts to function

## Install systemd service file

You'll need to install a systemd service script in order to run the Minecraft server instances as systemd services. The following commands will copy the example script in this repository to the correct system directory and reload the systemd daemon to ensure the service can be used:
```
$ sudo cp /opt/minecraft/minecraft@.service.example /etc/systemd/system/minecraft@.service
$ sudo systemctl daemon-reload
```

## Ensure service user permissions are correct in project directory

It's important that your service user is able to read and write to the project directory. To ensure the files are owned by the user as intended, run the following command:
```
$ sudo chown -R minecraft:minecraft /opt/minecraft
```

Note: You'll see `sudo -u minecraft` used for many commands in this installation and configuration guide. This command prefix causes the rest of the command to run as the `minecraft` user. This should result in correct permissions on actions that create new files on the filesystem. I know I've made the mistake of simply using `sudo` when issuing such commands more often than I'd like to admit, and in that instance, it's necessary to correct permissions. Hopefully this section helps when the inevitable permission issue pops up.

## Configure your first server instance

Run the create_new_instance script as the `minecraft` user to create a new server instances. This script requires that only 1 argument (the instance name) be passed. An example:
```
$ sudo -u minecraft /opt/minecraft/create_new_instance server1
```

You will be prompted to accept the Mojang Minecraft EULA as part of configuration. The helper script generates the eula.txt file as part of server instance creation, specifying "eula=true" as the last line of the file, and as such requires that the user be aware of the EULA and submit acceptance of the EULA terms as part of the run operation. This project is in no way affiliated with Mojang, but I recognize that acceptance of their terms is important.

## Run your server

To start your server, issue:
```
$ sudo systemctl start minecraft@server1
```

You should see no output after running this command; the server will be running in the background.

## Access your server console

Your server instance runs in a tmux session until the process ends, at which point the tmux session is closed. This is a benefit, as you can peek in on the server console and interact with it as necessary while the server process is running.

To access your server console, issue:
```
$ sudo -u minecraft tmux -a -t mc-server1
```

To detach from the session and leave it running, simply press "ctrl+b" and then "d" when your terminal is open.

## Stop your server

To stop your server, issue:
```
$ sudo systemctl stop minecraft@server1
```

## Configure your server to run at boot

To run your Minecraft server at boot time, issue:
```
$ sudo systemctl enable minecraft@server1
```

If you decide you'd rather not start your Minecraft server at boot time, simply disable it:
```
$ sudo systemctl disable minecraft@server1
```

# Important paths

```
- /etc/systemd/system/minecraft@.service    # systemd service file
- /opt/minecraft                            # minecraft user home dir, houses server instances and server jar files
- /opt/minecraft/create_new_instance	    # script to create a new minecraft instance; configures eula and instance scripts
- /opt/minecraft/instances                  # houses server instances / worlds
- /opt/minecraft/instances/<i>/scripts	    # houses instance specific scripts used by systemd (start, stop)
```

# Notes

- If you would like to modify start/stop scripts, focus on changing the create_new_instances script instead of individual start/stop scripts, as this script generates both
- All server instances reference a local symbolic link that points to /opt/minecraft/minecraft_server.jar by default. If you intend to use separate minecraft_server.jar versions for different server instances, make sure to link the exact jar file in each separate instance using the ln command (e.g.: `ln -s /opt/minecraft/minecraft_server.1.15.1.jar /opt/minecraft/instances/myinstance/minecraft_server.jar`)

# Change log
- 2019-01-05 - v1.0 - Initial version with many automatic convenience behaviors, without many dynamic configuration features (server instance name is about it)

