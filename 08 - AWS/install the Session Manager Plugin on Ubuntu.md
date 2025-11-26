---
tags:
  - AWS
  - SSM
---

# Step 1: Download the Installer

  This command uses curl to download the official Debian (.deb) package for the plugin from AWS.
  
curl "[https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb](https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb)" -o "session-manager-plugin.deb"
 ---

#   Step 2: Install the Plugin

  This command uses the dpkg package manager to install the file you just downloaded. sudo is required because it installs the software for all users on your system.
 sudo dpkg -i session-manager-plugin.deb 
 ---

# Step 3: Verfiy the Installation 

 session-manager-plugin
  If it's successful, you will see this output, confirming the plugin is ready:
   The Session Manager plugin is installed successfully. Use the AWS CLI to start a session.