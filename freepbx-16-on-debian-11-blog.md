## FreePBX 16 and Debian 11: An Apt Combination

There are many ways to install a FreePBX system; the easiest way is to install it through a FreePBX Distro ISO. But if you want more choices, you have several--FreePBX is an open-source project and works with all supported versions of Asterisk and on various flavors of Linux.

My Linux of choice is Debian, and this past weekend version 11 of the OS was released. Let's try it out!

In this blog post, we are going to simplify the installation process as much as possible through the use of Debian's "apt" package system. Some benefits of using packages are that they are maintained by the OS maintainers--there's no need to rebuild software from source when there's an update; the command lines are simple; and installations are predictable--software is installed in /usr/bin, /usr/lib, with configuration files in /etc, and so on. Moreover, the software you install from the package system tends to be well-vetted and stable. Some drawbacks to using the package system are that you are limited to the version(s) chosen by the package maintainers and customization may be more difficult. As an example, with Debian 11's apt repository, we are limited to using Asterisk version 16.

Let's walk through the steps to install FreePBX using Debian packages. At the end, you will have a functional FreePBX 16 system with Asterisk 16 at the core. I am including a link to the Gist so that you can copy and paste, and so that any errors can be corrected after this post has been published.

<script src="https://gist.github.com/billsimon/f66636f83de8e1162ea318ccd7a9b576.js"></script>

Start with a base Debian 11 installation (however you get it from your cloud/VPS provider or Debian install disc/ISO) and log in as root or `sudo su -` to become root. Then update to current with `apt update && apt upgrade` before continuing. 

The main `apt install` command brings in all the prerequisite packages including Asterisk, PHP 7.4, Apache, MariaDB, NodeJS, and other supporting software. This is the bulk of the installation process.

Because we are installing Asterisk from Debian's package system, it comes with its own sample configurations and startup scripts. We will disable the startup scripts because FreePBX manages Asterisk. Also, we'll clear out the sample configs and leave only a bare-bones config for FreePBX to build upon.

The next steps are a few adjustments to the default Apache web server configuration, required by FreePBX, and configuration of the ODBC connection that FreePBX uses to work with the CDR database.

Now we can install FreePBX, downloaded from freepbx.org and deployed using the included install script. 

As a final step, we create a systemd unit that starts FreePBX on boot.

In a matter of minutes, FreePBX 16 with Asterisk 16 is now installed and you are ready to move on to configuration from the web interface. Browse to http://<the IP address of the server> to continue. 
  
There are still a few clean-up items remaining. Don't put off this task: add some network security. The Sangoma firewall module is not available on this installation, so you will probably want to manually configure iptables or ufw to control acccess, unless you have another firewall in front of your server.

Also, UCP's node service doesn't work with the packaged version of nodejs, but [there is a ticket open to solve this](https://issues.freepbx.org/browse/FREEPBX-22742). Keep an eye on module updates to see when this fix comes through.

