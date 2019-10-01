# Server_setup
Creating an RStudio/Shiny server using AWS Lightsail and Docker (on MacOS)
This is heavily inspired by the [script to install todo by mikegcoleman](https://github.com/mikegcoleman/todo/blob/master/lightsail-compose.sh), [Jordan Farrers manual to install Rstudio](https://jrfarrer.github.io/post/how-to-setup-rstudio-on-amazon-lightsail/) and [Mikkels guide to setup Rstudio an Shiny with a shared pakage directory] https://www.r-bloggers.com/setup-encrypted-rstudio-and-shiny-dashboard-solution-in-3-minutes/.

## Create a server
- Create an account with [Amazon Web Services (AWS) Lightsail](https://lightsail.aws.amazon.com)
- Create an instance. 
  - Leave the location as-is
  - Under instance image Choose Linux/Unix > OS Only > Ubuntu 18.04 LTS
- Do not use the Default SSH key, but upload a public key of your own (\*_rsa.pub). To do this, use the SSH Key pair manager. Click on the **+** and upload a public key.
  - SSH keys are located in ~/.ssh/. One can navigate there from Finder by pressing <kbd>Shift</kbd> + <kbd>Command</kbd> + <kbd>G</kbd> and typing the directory name (~/.ssh/.)
  - Copy your public key to a visible directory that can be accessed by the web-interface
  - Select the appropriate file 
  - Instructions to create a new SSH Key can be found [here](https://help.github.com/en/enterprise/2.16/user/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- Choose the free instance plan
- Name the server (e.g. *Rstudio*)
- Click *Create Instance*
  
## initial setup of the server
While the server is spinning up, a few parameters need to be set. 
1) To make sure that the instance can be reached from outside
  - Click on 'Networking' 
  - Create a Static-IP (note: this is free as long as it is coupled to an instance)
  - To couple the Static IP to the newly created instance click on the three dots next to the IP and select manage.
  - Select the newly created instance and click 'Attach'
  - Couple this to the newly created instance (note: this is also free)
2) To open the ports required for *Rstudio* and *Shiny*
  - Click on 'Instances' and select the newly created instance
  - Click on the three dots and select 'Manage'
  - Click on 'Networking and add to new entries under 'Firewall'
    - Custom, TCP, 8787 <-- for Rstudio
    - Custom, TCP, 3838 <-- for Shiny
    - Click 'save'
    
## Setting up a connection
Now it is time to configure Ubuntu Linux to run Rstudio and Shiny via an SSH session.
#To set up a local SSH session:
  - note down the IP address of the instance.
  - open the terminal and initise an SSH-connection by typing `ssh ubuntu@IPADDRESS` with IPADDRESS the ip address of the instance (e.g. ubuntu@12.241.147.112).
    - you may have to type the secret sentence for the SSH key that you are using. This can be found in the keychain. Open keychain (e.g. via Spotlight Search) and search for the key that you have uploaded. 
    - if you have previously used the same IP-address to connect to another instance you will get a warning. This can be solved by removing the relevant entry from the known hosts file (`~/.ssh/known_hosts`). Look for the line starting with the IPADDRESS that you are using and remove it. Don't forget to save prior to issuing the ssh command in the terminal.
  - You may see a message asking if the host can be trusted. Say 'yes' to add the server to the known_hosts file.
Now you should be seeing something like ubuntu@ip-212.141.47.12:~$. Note that the displayed IP adress is the *internal* rather than the 

## Further setup of the server 
As the amount of memory is limited to 512K it is helpful to assign swapspace:
```
sudo touch /var/swap.img
sudo chmod 600 /var/swap.img
sudo dd if=/dev/zero of=/var/swap.img bs=2048k count=1000
sudo mkswap /var/swap.img
sudo swapon /var/swap.img
```

Is it also a good idea to upgrade the packages on the server:
`apt-get upgrade`
You may get the question whether the SSH configuration file should be changed. Say no.

##installing packages 

To increase reproducability of analysis we will use [Docker](https://www.docker.com/) with [Rocker images](https://www.rocker-project.org/images/). This will guaruantee consistency between different installations of R/Rstudio/Shiny and associated packages.

# Install Docker and Docker Compose
  ```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# to enable running docker as user ubuntu
sudo usermod -aG docker ubuntu

# install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# copy the dockerfile into /srv/docker 
# if you change this, change the systemd service file to match
# WorkingDirectory=[whatever you have below]
mkdir /srv/docker
curl -o /srv/docker/docker-compose.yml https://raw.githubusercontent.com/jkeuskamp/Biont_server_setup/master/docker-compose.yml
```

#install Rstudio/Tidyverse and Shiny/tidyverse
```
# copy in systemd unit file and register it so our compose file runs 
# on system restart
curl -o /etc/systemd/system/docker-compose-app.service https://raw.githubusercontent.com/Biont_server_setup/master/docker-compose-app.service
systemctl enable docker-compose-app

# start up the application via docker-compose
docker-compose -f /srv/docker/docker-compose.yml up -d
```

#install (Headless) Dropbox
```
#Download Python 2.7 required for the Dropbox script
sudo apt install python2.7-minimal
#Download and start the deamon
cd ~ && wget -O - "https://www.dropbox.com/download?plat=lnx.x86_64" | tar xzf -
~/.dropbox-dist/dropboxd
```
You may see a line asking you to paste a link into your browsr to connect the dropbox to the server.
The process will continue after you do so
If your dropbox is large, you may be asked to run the following line prior to rerunning the above
```
echo fs.inotify.max_user_watches=100000 | sudo tee -a /etc/sysctl.conf; sudo sysctl -p
```

note that the steps above can also be performed as an installation script during setup of the instance. To do so paste the contents of the two blocks above in the 'inital script

##Using the packages
To use RStudio navigate to <AWS IP address>:8787 using a browser. Username = test and password = test. These passwords can be changed in the docker-compose.yml. To do this, type 'sudo nano /srv/docker/docker-compose.yml and change the lines starting with USER and PASSWORD.
  
To use Shiny navigate to <AWS IP address>:3838.
