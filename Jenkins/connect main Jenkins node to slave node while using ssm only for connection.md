---
tags:
  - Jenkins
  - Linux
  - AWS
  - SSM
---
# Part 1: On the Jenkins Controller (Master)

  
  

   1. Create the Node:

       * Go to Manage Jenkins > Nodes.

       * Click New Node.

       * Name it (e.g., private-agent-1), select Permanent Agent, and click Create.

  
  

   2. Configure the Node:

       * Remote root directory: Enter /home/ssm-user/jenkins-agent.

           * Where is this created? This directory will be created by you on the new agent EC2 instance in the next part. It is the dedicated

             folder where the agent will do its work (e.g., checkout code, run builds).

       * Launch method: Select Launch agent by connecting it to the controller.

       * Click Save.

  
  

   3. Get the Command:

       * Click on the name of the new agent you just created.

       * Jenkins will display a command that includes a secret key. It will look like:

          java -jar agent.jar -jnlpUrl http://... -secret <YOUR_SECRET_KEY> ...

       * Copy this entire command. You will need it in the next step.

4. In Manage Jenkins Security—TCP port for inbound agents—change to fixed and input 50000, then save

  

  ---

# Part 2: On the Agent EC2 Instance (Slave)

  

   5. Connect to the new agent EC2 instance using SSM.

  
  

   6. Install Java: The agent needs Java to run.

  

 sudo yum install -y java-21-amazon-corretto

  
  

   7. Create the Directory & Get Agent Software:

  
  

   1     # Create the folder you defined in Part 1

   2     mkdir /home/ssm-user/jenkins-agent

   3

   4     # Enter the new folder

   5     cd /home/ssm-user/jenkins-agent

   6

   7     # Download the agent software from your Jenkins controller

   8     curl -o agent.jar http://<JENKINS_CONTROLLER_PRIVATE_IP>:8080/jnlpJars/agent.jar

  

       * Important: Replace <JENKINS_CONTROLLER_PRIVATE_IP> with the actual private IP address of your main Jenkins controller instance.

  
  

   4. Connect to Controller:

       * Paste and run the full java -jar agent.jar... command you copied from the Jenkins UI.

to make the connection as a service that starts automatically 

sudo systemctl start jenkins-agent.service

  

     [Unit]

     Description=Jenkins Agent

     After=network.target

  

     [Service]

     User=ssm-user

     WorkingDirectory=/home/ssm-user/jenkins-agent

     ExecStart = /usr/bin/java -jar agent.jar -jnlpUrl [http://10.0.3.119:8080/computer/agent-1/slave-agent.jnlp](http://10.0.3.119:8080/computer/agent-1/slave-agent.jnlp) -secret 76925943

      c97afdcb41cb2267cdd1dfbf738025152c46019c21794b75c4409173 -workDir "/home/ssm-user/jenkins-agent"

     Restart=always

     RestartSec=10

  

     [Install]

     WantedBy=multi-user.target

  

sudo systemctl daemon-reload

sudo systemctl start jenkins-agent.service

sudo systemctl status jenkins-agent.service