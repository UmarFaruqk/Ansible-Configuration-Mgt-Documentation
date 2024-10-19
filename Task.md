## Ansible-Configuration-Mgt (Automate Project 7-10)

In Projects 7 to 10 we had to perform a lot of manual operations to set up virtual servers, install and configure required software and deploy our web application.

This Project will us appreciate DevOps tools even more by making most of the routine tasks automated with Ansible Configuration Management, at the same time we will become confident with writing code using declarative languages such as YAML.

# Ansible Client as a Jump Server (Bastion Host)
A Jump Server (sometimes also referred as Bastion Host) is an intermediary server through which access to internal network can be provided. For instance, the current architecture i am working on, ideally, the webservers would be inside a secured network which cannot be reached directly from the Internet. That means, even DevOps engineers cannot SSH into the Web servers directly and can only access it through a Jump Server – it provide better security and reduces attack surface.

On the diagram below the Virtual Private Network (VPC) is divided into two subnets - Public subnet has public IP addresses and Private subnet is only reachable by private IP addresses. ![reference image](/Pictures/pic26.png)

# Step 1:  Install and Configure Ansible on EC2 Instance
1. We will update then *Name* tag onour *Jenkins* EC2 Instance to Jenkins-Ansible. We will use this server to run playbooks.
2. IN GitHub we will create a new repository and name it *ansible-config-mgt*. ![reference image](/Pictures/pic2.PNG)
3. Install Ansible but first we update our instance using *sudo apt update* then we run *sudo apt install ansible* ![reference image](/Pictures/pic1.PNG)
4.  Check your Ansible version by running *ansible --version* ![reference image](/Pictures/pic3.PNG)
5.  Configure Jenkins build job to archive your repository content every time you change it - this will solidify your Jenkins configuration skills acquired in Project 9.
1. Create a new Freestyle project *ansible* in Jenkins and point it to your 'ansible-config-mgt' repository. ![reference image](/Pictures/pic6.PNG)
2. Configure a webhook in GitHub and set the webhook to trigger *ansible* build. ![reference image](/Pictures/pic5.PNG)
3. Configure a Post-build job to save all (**) files, like you did it in Project 9. ![reference image](/Pictures/pic7.PNG)
4. Test your setup by making some change in README.md file in *main* branch and make sure that builds starts automatically and Jenkins saves the files (build artifacts) in following folder *cat /var/lib/jenkins/Ansible/README.md* ![reference image](/Pictures/pic8.PNG) ![reference image](/Pictures/pic9.PNG)
**NOTE**: Trigger Jenkins project execution only for main (or master) branch.
Now your setup will look like this: ![reference image](/Pictures/pic10.PNG)
**Tip**: Every time you stop/start your Jenkins-Ansible server - you have to reconfigure GitHub webhook to a new IP address, in order to avoid it, it makes sense to allocate an Elastic IP to your Jenkins-Ansible server (you have done it before to your LB server in Project 10). Note that Elastic IP is free only when it is being allocated to an EC2 Instance, so do not forget to release Elastic IP once you terminate your EC2 Instance. ![reference image](/Pictures/pic4.PNG)

# Step 2: Prepare your development environment using Visual Studio Code
1. First part of 'DevOps' is 'Dev', which means you will require to write some codes and you shall have proper tools that will make your coding and debugging comfortable - you need an Integrated development environment (IDE) or Source-code Editor. There is a plethora of different IDEs and source-code Editors for different languages with their own advantages and drawbacks, you can choose whichever you are comfortable with, but we recommend one free and universal editor that will fully satisfy your needs - Visual Studio Code (VSC) you can get it here <https://code.visualstudio.com/download>
2. After you have successfully installed VSC, configure it to connect to your newly created GitHub repository.
3. Clone down your ansible-config-mgt repo to your Jenkins-Ansible instance using *git clone <ansible-config-mgt repo link>* ![reference image](/Pictures/pic11.PNG)

# Step 3: Begin Ansible Development
1. In your *ansible-config-mgt* GitHub repository, create a new branch that will be used for development of a new feature.
**Tip**: Give your branches descriptive and comprehensive names, for example, if you use Jira or Trello as a project management tool - include ticket number (e.g. *PRJ-145*) in the name of your branch and add a topic and a brief description what this branch is about - a *bugfix, hotfix, feature, release (e.g. feature/prj-145-lvm)*
2. Checkout the newly created feature branch to your local machine and start building your code and directory structure
3. Create a directory and name it *playbooks* - it will be used to store all your playbook files.
4. Create a directory and name it *inventory* - it will be used to keep your hosts organised.
5. Within the playbooks folder, create your first playbook, and name it *common.yml*
6. Within the inventory folder, create an inventory file () for each environment (Development, Staging Testing and Production) *dev, staging, uat,* and *prod* respectively. These inventory files use *.ini* languages style to configure Ansible hosts. ![reference image](/Pictures/pic12.PNG)

# Step 4: Set up an Ansible Inventory
An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate. Since our intention is to execute Linux commands on remote hosts, and ensure that it is the intended configuration on a particular server that occurs. It is important to have a way to organize our hosts in such an Inventory.

Save the below inventory structure in the *inventory/dev* file to start configuring your development servers. Ensure to replace the IP addresses according to your own setup.

**NOTE**: Ansible uses TCP port 22 by default, which means it needs to ssh into target servers from Jenkins-Ansible host - for this you can implement the concept of ssh-agent. Now you need to import your key into ssh-agent.

To learn how to setup SSH agent and connect VS Code to your Jenkins-Ansible instance, please see this video:
1. For Windows users -<https://www.youtube.com/watch?v=OplGrY74qog>
2. For Linux users - <https://www.youtube.com/watch?v=OplGrY74qog>

1. run the following command *eval `ssh-agent -s`* *ssh-add <path-to-private-key>*
2. Confirm the key has been added with the command below, you should see the name of your key *ssh-add -l*
3. Now, ssh into your *Jenkins-Ansible* server using ssh-agent *ssh -A ubuntu@public-ip* ![reference image](/Pictures/pic13.PNG)

Also notice, that your Load Balancer user is ubuntu and user for RHEL-based servers is ec2-user.

4. Update your inventory/dev.yml file with this snippet of code ![reference image](/Pictures/pic14.PNG)
**NOTE**: All the IP addresses above are private IP addresses

# Step 5: Create a Common Playbook
It is time to start giving Ansible the instructions on what you need to be performed on all servers listed in *inventory/dev*.
In *common.yml* playbook you will write configuration for repeatable, re-usable, and multi-machine tasks that is common to systems within the infrastructure.

1. Update your *playbooks/common.yml* file with following code *---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  become: yes
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest
   

- name: update LB server
  hosts: lb
  become: yes
  tasks:
    - name: Update apt repo
      apt: 
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest*
Examine the code above and try to make sense out of it. This playbook is divided into two parts, each of them is intended to perform the same task: install wireshark utility (or make sure it is updated to the latest version) on your RHEL 8 and Ubuntu servers. It uses root user to perform this task and respective package manager: yum for RHEL 8 and apt for Ubuntu.
2. Feel free to update this playbook with following tasks:
1. Create a directory and a file inside it
2. Change timezone on all servers
3. Run some shell script
For a better understanding of Ansible playbooks watch this <https://www.youtube.com/watch?v=ZAdJ7CdN7DY> and read this <https://www.redhat.com/en/topics/automation/what-is-an-ansible-playbook>

# Step 6: Update GIT with the latest code
Now all of your directories and files live on your machine and you need to push changes made locally to GitHub.
1. We have a separate branch, there is need to raise a Pull Request (PR), get your branch peer reviewed and merged to the main branch.
2. Commit your code into GitHub, Use the commands below:
1. git status
2. git add <selected files>
3. git commit -m "commit message"
4. git pull origin prj-11a

3. Head to your github account, create a Pull request
4. Head back on your terminal, checkout from the feature branch into the main, and pull down the latest changes.
1. git checkout main
2. git pull origin prj-11a

Once your code changes appear in main branch – Jenkins will do its job and save all the files (build artifacts) to /var/lib/jenkins/Ansible/inventory/dev.yml  /var/lib/jenkins/Ansible/playbooks/common.yml directory on Jenkins-Ansible server. ![reference image](/Pictures/pic17.PNG)

# Step 7:  Run the first Ansible test
Now, it is time to execute ansible-playbook command and verify if your playbook actually works
1. Execute ansible-playbook command . On your jenkins-ansible server termianl, run *ansible-playbook -i /var/lib/jenkins/Ansible/inventory/dev.yml  /var/lib/jenkins/Ansible/playbooks/common.yml* ![reference image](/Pictures/pic18.PNG)
2. You can go to each of the servers and check if wireshark has been installed by running which *wireshark* or *wireshark --version* ![reference image](/Pictures/pic21.PNG) ![reference image](/Pictures/pic22.PNG) ![reference image](/Pictures/pic23.PNG)  ![reference image](/Pictures/pic24.PNG)  ![reference image](/Pictures/pic25.PNG)
3. The updated Ansible architecture now looks like this:
![reference image](/Pictures/pic28.png)
4. Update your ansible playbook with some new Ansible tasks and go through the full *checkout -> change codes -> commit -> PR -> merge -> build -> ansible-playbook*
5. cycle again to see how easily you can manage a servers fleet of any size with just one command!
1. Create a directory and a file inside it
2. Change timezone on all servers
3. Run some shell script
4. ***
![reference image](/Pictures/pic15.PNG) ![reference image](/Pictures/pic18.PNG) ![reference image](/Pictures/pic19.PNG)

**CONGRATULATIONS** : We just automated routine tasks by implementing our first ansible project.