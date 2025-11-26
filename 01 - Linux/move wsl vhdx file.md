---
tags:
  - Linux
  - WSL
---
wsl --manage <distro_name> --move <new_location>
distro_name is the distro you installed for example Ubuntu

if you get an error file is being used end any task related to wsl
then register the distro again 

wsl.exe -d <NewDistributionName>

NewDistributionName = Ubuntu

check the vhdx file and make sure it is not read only and your windows user have all permissions to edit this file

--------------------------------------------------

sudo echo -e "[user]\ndefault=your_user_name\n” >> /etc/wsl.conf

wsl --export "Ubuntu-22.04" "ubuntu22-export.tar"

wsl --unregister "Ubuntu-22.04"

wsl --import "custom_name" "path_to_new_location" "path_to_tar_file"

--------------------------------------------------

 1. List your installed distributions:

  

   1     wsl --list --all

  
  

   2. Shut down any running distributions:

  

   1     wsl --shutdown

  
  
  

   3. Move the distribution:

      Replace Ubuntu with your distribution's name and D:\WSL\Ubuntu with your desired new location.

  

   1     wsl --manage Ubuntu --move D:\WSL\Ubuntu

check the vhdx file and make sure it is not readonly and your windows user have all permissions to edit this file 

  

  Method 2: Using wsl --export and wsl --import

  

  This method is a bit more involved, but it's also useful for creating backups.

  

   1. List your installed distributions:

  

   1     wsl --list -v

  
  
  

   2. Shut down the distribution you want to move:

  

   1     wsl --terminate <distribution_name>

  

      For example:

  

   1     wsl --terminate Ubuntu-22.04

  
  
  

   3. Export the distribution to a TAR file on your other drive:

  

   1     wsl --export <distribution_name> "D:\wsl_export\ubuntu.tar"

  
  

   4. Unregister the original distribution (this deletes it from C:):

  
  

   1     wsl --unregister <distribution_name>

  
  

   5. Import the distribution to the new location:

  
  

   1     wsl --import <distribution_name> "D:\wsl_import\ubuntu" "D:\wsl_export\ubuntu.tar" --version 2