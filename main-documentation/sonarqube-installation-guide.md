# How to Install and Setup SonarQube
A complete documentation for installing, configuring, and managing a self-hosted SonarQube instance on a Linux VM.

*Note: This Documentation uses Bash commands for installation.*

## Step 1: SSh to your Linux Machine
First step is to conect to your linux machine. If you are using physical linux machine, then simply open the terminal and start following the below steps.

But If you are working in cloud environment, then you have to connect to your Virtual Machine using secure shell (ssh). Below is the command to connect using ssh:

```ssh [username]@[hostname or IP address]```

## Step 2: Install Java 11
It is necessary to install Java before installing SonarQube because SonarQube, both the server and the scanners, require a Java Runtime Environment (JRE) to function. Specifically, SonarQube documentation indicates a requirement for Oracle JRE 11 or OpenJDK 11 (or later versions like Java 17 for newer SonarQube versions). Without a compatible Java installation, SonarQube will not be able to run.
Here are the step to install java on your linux machine:

Update the packages.

```$ sudo apt update```

Install dependencies.

```$ sudo apt install wget unzip curl gnupg2 ca-certificates lsb-release socat -y```

Install Java 11.

```$ sudo apt-get install openjdk-11-jre -y```


## Step 3. Install PostgreSQL
Installing a supported database like PostgreSQL is a necessary prerequisite for installing and running SonarQube, especially for production environments.

SonarQube requires a database to store all its data, including analysis results, project configurations, user information, and more. While SonarQube comes with an embedded H2 database for testing or demonstration purposes, it is not recommended for production use.

Add PostgreSQL repository.

```$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'```

Add the PostgreSQL signing key.

```$ wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -```

Update the system.

```$ sudo apt update```

Install PostgreSQL.

```$ sudo apt-get install postgresql postgresql-contrib -y```

Enable PostgreSQL service to start on system boot.

```$ sudo systemctl enable postgresql```

Start PostgreSQL service.

```$ sudo systemctl start postgresql```

## Step 4. Create SonarQube Database

Change postgres default user password.

```$ sudo passwd postgres```

Log in with user postgres.

```$ su - postgres```

Create sonarqube user.

```$ createuser sonarqube```

Enter the PostgreSQL interactive shell.

```$ psql```

Set password for user sonarqube. Change SecurePassword with your secure password.

```ALTER USER sonarqube WITH ENCRYPTED password 'SecurePassword';```

Create database named sonarqube.

```CREATE DATABASE sonarqube OWNER sonarqube;```

Grant all the privileges on the sonarqube database to the sonarqube user.

G```RANT ALL PRIVILEGES ON DATABASE sonarqube to sonarqube;```

Exit the PostgreSQL shell.

```\q```

Return to your non-root account.

```$ exit```

### Pre-Requisites on Linux Systems 
Configuring the maximum number of open files and other limits

You must ensure that:

•	The maximum number of memory map areas a process may have (vm.max_map_count) is greater than or equal to 524288.

•	The maximum number of open file descriptors (fs.file-max) is greater than or equal to 131072.

•	The user running SonarQube Server can open at least 131072 file descriptors.

•	The user running SonarQube Server can open at least 8192 threads.

To check and change these limits, login as the user used to run SonarQube Server and proceed as described below depending on the type of this user.

## Step 5. Install and Configure SonarQube
After installing and setting up the pre-requisites, its time to install the SonarQube Community Edition on our Ubuntu/Debian Machine. Download the latest version of SonarQube. To find the latest version, visit the download page: https://www.sonarqube.org/downloads/

This document installs the community version. To install Enterprise (which requires licence) or other edition, visit this link: 
https://www.sonarsource.com/products/sonarqube/downloads/

And for more information and guidance about editions other than Community Edition, visit this official document: 
https://docs.sonarsource.com/sonarqube-server/server-installation

Let's start installing our SonarQure:

```$ wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.2.2.50622.zip```

Extract the downloaded archive.

```$ sudo unzip sonarqube-9.2.2.50622.zip```

Create the installation directory /opt/sonarqube.

```$ sudo mkdir /opt/sonarqube```

Move the extracted files into the installation directory.

```$ sudo mv sonarqube-*/* /opt/sonarqube```

Create a System User account for SonarQube.

```$ sudo useradd -M -d /opt/sonarqube/ -r -s /bin/bash sonarqube```

Change the ownership of the installation directory.

```$ sudo chown -R sonarqube:sonarqube -R /opt/sonarqube```

Edit the properties file to update the database credential.

```$ sudo nano /opt/sonarqube/conf/sonar.properties```

The final file should have the following changes. Save and close the file.

***sonar.jdbc.username=sonarqube***

***sonar.jdbc.password=SecurePassword***

***sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube***

***sonar.web.host=0.0.0.0***

## Step 6. Add systemd Services
Create a systemd service file.

```$ sudo nano /etc/systemd/system/sonarqube.service```

Add the bellow code to the file. Save and close the file.
```
[Unit]
Description=SonarQube Service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonarqube
Group=sonarqube
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

Edit the sonar script file.

```$ sudo nano /opt/sonarqube/bin/linux-x86-64/sonar.sh```

Find this line. To search for it, use CONTROL+W, enter search phrase then press ENTER.

```#RUN_AS_USER=```

Now uncomment the line and change it. Finally, save and exit the file.

```RUN_AS_USER=sonarqube```

Reload the system daemon.

```$ sudo systemctl daemon-reload```

Enable SonarQube service to start on system boot.

```$ sudo systemctl enable sonarqube```

Start SonarQube service.

```$ sudo systemctl start sonarqube```

Check the service status.

```$ sudo systemctl status sonarqube```

Allow SonarQube default port 9000 through the system's firewall.

```$ sudo ufw allow 9000/tcp```

## Step 7. Change Kernel Limits
As given above the pre-requisites for the linux systems, where there was a specified limits in order to install an enterprise level or other edition of SonarQube. So, here is the tutorial on how to setup the limits.
Edit the sysctl configuration file to change some system defaults.

```$ sudo nano /etc/sysctl.conf```

Add the below code to the file. Then, save and exit the file.
```
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```

Reload the sysctl configurations for changes to take effect.

```$ sudo sysctl --system```

## Step 8. Access SonarQube
Go to your browser and go to the URL http://Server_IP:Port. For example:
http://192.0.2.11:9000

## Conclusion
You have installed SonarQube on Linux server. Login with the default credential with your username as admin and your password as admin. You can now continue and begin creating accounts for code analysis.



The credit humbly goes to the websites, from where I got help: 

Official Documentation: https://docs.sonarqube.org/latest/

Other Documentation: https://docs.vultr.com/how-to-install-sonarqube-on-debian-11


