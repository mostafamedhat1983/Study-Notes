# install specific version of ansible using pip3

sudo pip3 install ansible==4.9.0

## verify ansible location

command -v ansible

 If you need every user on the system to be able to run the ansible command, you should create a file in
  /etc/profile.d/.

  Create a new shell script in `/etc/profile.d/`. The tee command is used to write the file with sudo.


   1     echo 'export PATH="/usr/local/bin:$PATH"' | sudo tee /etc/profile.d/[ansible.sh](http://ansible.sh/)

      Note: This assumes Ansible is in `/usr/local/bin`. Adjust the path if you found it elsewhere.
   2. Make the script executable.

  

   1     sudo chmod +x /etc/profile.d/[ansible.sh](http://ansible.sh/)

  

   3. The changes will be applied for all users the next time they log in.

  

  Step 3: Verify It Works

  

   4. Open a NEW terminal to ensure the startup files have been loaded.

   5. Run the ansible --version command from any directory.

  

   1     ansible --version

  

  If it prints the Ansible version information, the binary is now globally available in your path.