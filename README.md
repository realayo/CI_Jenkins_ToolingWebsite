# Jenkins CI
We are going to enhance the architecture prepared in our [Load Balancer Solution With Apache](https://github.com/realayo/Apache_LoadBalancer) by adding a Jenkins server. We would configure a job to automatically deploy source codes
changes from Git to NFS server.

## Step 1 - Install Jenkins server
1. Create an AWS EC2 server based on Ubuntu Server 20.04 LTS and name it `Jenkins`
2. Install JDK 
```
sudo apt update
sudo apt install default-jdk-headless
```
3. Install Jenkins
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'

sudo apt update

sudo apt-get install jenkins
```
4. Make sure Jenkins is up and running
```
sudo systemctl status jenkins
```
![](https://user-images.githubusercontent.com/18899718/120899739-d15b9980-c5f6-11eb-8c5d-62a39384158f.png)
>Jenkins server uses TCP port 8080 - open it by creating a new Inbound Rule in your EC2 Security Group
5. Perform initial Jenkins setup from your browser `http://<Jenkins-Server-Public-IP-Address>:8080`
![](https://user-images.githubusercontent.com/18899718/120899901-96a63100-c5f7-11eb-8162-e106d3835622.png)
6. Retrive admin password
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
7. Copy the password on input it in the `Unlock Jenkins` page. 
8. Next, you would be asked to customise jenkins, select `Install suggested plugins`
which will immediately begin the installation process.
![](https://user-images.githubusercontent.com/18899718/120900128-c6a20400-c5f8-11eb-9843-3a81fa321f7a.png)
9. When the installation is complete, you’ll be prompted to set up the first administrative user, create a username and password and continue.


## Enable webhooks by going to your 
1. Go to prepository setting.
2. On the left pane, click on webhooks
3. Click on `add webhook`
4. Under payload URL, input `http://<Jenkins-Server-Public-IP-Address>:8080/github-webhook/
5. Under `content type` choose `application/json`
6. Click on `Add webhook` to save your settings.

## Configure Jenkins to retrieve source codes from GitHub using Webhooks

1. Go to Jenkins web console, click “New Item” and create a `Freestyle project`. Enter a name for the `freestyle project`, in our case we use `tooling`
2. Under `source code management`, select `git`. Go to the GitHub repository, click on `code` and copy the repository.
3. Save the configuration and let us try to run the build manually. Click on `Build Now` button, the build will be successfull and you will see it under `#1`
4. Our aim is to trigger the build automatically which we would fix in the next step.
5. Click on `Configure` job/project and add these two configurations
6. under `build trigger`, select `Github hook trigger for GITSCM polling`
7. Continue to `post-build actions` to archive all the files. Click on `add post build actions` and `archive the artifacts`, for `files to archive` add `**`
8. click on save.
9. Make some changes in the github repository and push the changes to `Main branch`
10. You will see that a new build has been automatically launched by webhook and you can see its results - artifacts, saved on Jenkins server.
> You will find the artifacts on the server if you navigate to `
ls /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/`

##  Configure Jenkins to copy files to NFS server via SSH
> Now we have our artifacts saved locally on Jenkins server, the next step is to copy them to our NFS server to /mnt/apps directory.

### Install `Publish Over SSH` plugin.
1. On main dashboard select `Manage Jenkins` and choose `Manage Plugins` menu item.
2. In `Available` tab search for `Publish Over SSH` plugin and install it. Click on `install without restart`

### Configure the project to copy artifacts over to NFS server.
1. On main dashboard select “Manage Jenkins” and choose “Configure System” menu item.
1. Scroll down to Publish over SSH plugin configuration section and configure it to be able to connect to your NFS server:
1. Provide the private key use to connect to your NFS server.

- Hostname: private IP address of your NFS server.

- Remote directory: `/mnt/apps` since our Web Servers use it as a mount point to retrieve files from the NFS server

- Test the configuration and make sure the connection returns Success. 

![](https://user-images.githubusercontent.com/18899718/120906336-45f5fe80-c61e-11eb-8d21-c1e1350c35bb.png)
> Remember, that TCP port 22 on NFS server must be open to receive SSH connections.

4. Save the configuration, open Jenkins job/project configuration page and add another `Post-build Action`
5. Click on `send build over SSH`
6. Under `Name` choose the NFS server address from the drop down.
7. For `source files` add `**`, which signifies we're copying all our files to the NFS server.
8. Save and trigger another job by editing a file in our repository. 
> You might experience an error like this: 
`ERROR: Exception when publishing, exception message [Permission denied]
Build step 'Send build artifacts over SSH' changed build result to UNSTABLE
Finished: UNSTABLE`
![](https://user-images.githubusercontent.com/18899718/120906661-ec430380-c620-11eb-9352-7e9a4f1555af.png)

9. You will have to change ownership of `/mnt/app` since it was intially set to be owned by nobody.
```
sudo chown -R ec2-user: /mnt/apps 
```
10. Trigger another job by editing a file in our repository and this time around we would have a succesful deployment. 
![](https://user-images.githubusercontent.com/18899718/120906676-067ce180-c621-11eb-8931-fcc29f75fb20.png)

11. To make sure that the files in `/mnt/apps` have been updated SSH to the NFS server and check README.MD file
```
cat /mnt/apps/README.md
```
![](https://user-images.githubusercontent.com/18899718/120906637-b9007480-c620-11eb-8709-61ce02ac8b68.png)
