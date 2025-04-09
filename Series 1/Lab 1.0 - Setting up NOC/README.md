
# Objective
Using the tool zabbix to send agents for monitoring. setup, deployment and troubleshooting for this initial project.



## Software
Oracle Virtual Box (V-7.1.6)
Zabbix (V-7.2)

## OS
Ubuntu server and Linux GUI Machines. We'll need 2 machines (at least for now) one for the zabbix server and the other for the agent to send logs. For the server we are going to be using Ubuntu Server 24.10 ISO and for the client Kali 2025.1a (The decision of Kali as a client has nothing to do, at least for now, with security things or pentesting realated, but the need of a client with GUI)

---

# Contents

| Project | Description |
|---------|-------------|
| [Virtualbox](https://github.com/DanielMunozParedes/NOC/blob/main/Series%201/Lab%201.0%20-%20Setting%20up%20NOC/README.md#virtualbox) | General overview, tips and internal network.|
| [Context](https://github.com/DanielMunozParedes/NOC/blob/main/Series%201/Lab%201.0%20-%20Setting%20up%20NOC/README.md#context) | Whare we going to do and choosing the correct component | 
| [Zabbix server](https://github.com/DanielMunozParedes/NOC/blob/main/Series%201/Lab%201.0%20-%20Setting%20up%20NOC/README.md#zabbix-server) | configuration of the zabbix server in ubuntu |
| [Dashboard](https://github.com/DanielMunozParedes/NOC/blob/main/Series%201/Lab%201.0%20-%20Setting%20up%20NOC/README.md#dashboard) | dashboard setup| 
| [Zabbix agent](https://github.com/DanielMunozParedes/NOC/blob/main/Series%201/Lab%201.0%20-%20Setting%20up%20NOC/README.md#zabbix-agent) | Configuration of the zabbix agent in kali| 
| [troubleshooting](https://github.com/DanielMunozParedes/NOC/blob/main/Series%201/Lab%201.0%20-%20Setting%20up%20NOC/README.md#troubleshooting) | Problems and solutions you might encounter during the realization of this lab| 

---

## virtualbox


[![zabbix3.png](https://i.postimg.cc/C5SCG651/zabbix3.png)](https://postimg.cc/KkpgFfnS)



the 2 virtual machines we'll be using
[![zabbix2.png](https://i.postimg.cc/C1mC4sqs/zabbix2.png)](https://postimg.cc/LYgZ9fvJ)



make sure to have bridge mode for both of them since we need internet connection, tho if you like to have an isolated net you can download the deb package and copy via usb to the server and the agent and do the apt update since that command runs from the deb repo you add it will work as well.



## Context


[![zabbix1.png](https://i.postimg.cc/Ssy78vJK/zabbix1.png)](https://postimg.cc/gwTZb4y9)


this is the link of the page: https://www.zabbix.com/download?zabbix=7.2&os_distribution=debian&os_version=12&components=agent_2&db=&ws=

before we install this tool we need to understand the concept. Again we are going to install a tool that has 2 primary modes: server and client (agent). the server is where the info is going to be gathering, collection, dashboard and the ip where the agents will be looking to send the data. Agent is the device where the logs,health,data will be collected to be send to the server. so we need to choose what mode to install obviously for each device. And one zabbix server. That does not means that a zabbix server cannot be an agent, IT CAN. but for practical reasons and organization we will be using oone server. You will see the server will come with and agent, because the zabbix will need also collect info from the server of course. I'm going to be using the mysql for the DB and apache for the web becasue are the msot common and so far there is no problems doing it.

before starting make sure to have both machines updated and upgraded


## Zabbix server

We will begin with the server using the virtual machine ubuntu server. we select the correct distro for our machine, the version we can check it with 
```
cat /etc/os-release
```
we will use the server, frontend and agent2. The frontend will allow us to have a gui dashboard , the web based interface that runs in apache, and the agent2 is the plugin for an agent for the server itself.

tho they (zabbix team) recommend to do all of this with root user we can do this with sudo, the only part we will need root is to acces the mysql database query part

*
*
*

download the package, once downloaded unarchive and update. The download will take some minutes so dont panic if the command line stays still. 

```
sudo wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
sudo apt update 
```

what we are doing here is to download and unarchive that package. then after "adding" the deb pacakge repo we do update so the system can see now the new deb added



nexy we install the three components: server(sql files and scripts to initialize the DB), frontend (web gui) and agent2

```
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent2 
```




now install the plugings for common database , these are recommended in case we have alredy have monitoring on them. Obligatory for mysql if you are using it.

```
sudo apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql 
```

*

here we need to log in as root  for the mysql part, and here it comes as well the part were we need to install "mysql-server" package. the tutorial don't specify , but it is necesary.

```
mysql -uroot -p
password
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@localhost identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit; 
```
we create the database zabbix, then we create a user zabbix, give all privileges for that user on that zabbix DB and we allow stored functions to be created for the schema

*


still in root we unpack the schema structure to the DB zabbix. It will ask us for the password we used in the earlier step.

```
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```


*

next we came back to mysql with root and the option where allow stored mysql fuctions for the schema? we disable with 0 for security reasons


```
mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit; 
```

*


next we need to modify the zabbix_server.config file. This file is the hearth for the server zabbix. For this step we are going to specify the file what is the password of the mysql db. you should be able to see the files such as this:

[![zabbix6.png](https://i.postimg.cc/N0585dL9/zabbix6.png)](https://postimg.cc/5HWQrqBb)


search for the line that contains 
```
DBPassword=password 
```
it will be commented. You can do it by showing the lines in the grep filtering like

```
nl zabbix_server.config | grep DBP
```

[![zabbix-7.png](https://i.postimg.cc/zBH5xHX2/zabbix-7.png)](https://postimg.cc/hznHjjmV)


found that line and change for the password of the mysql.


*

now we restart and enable the services

```
systemctl restart zabbix-server zabbix-agent2 apache2
systemctl enable zabbix-server zabbix-agent2 apache2 
```
*

## Dashboard

now go the a web browser and acces with the ip of the ubuntu server/zabbix
http://ip/zabbix/


[![1.png](https://i.postimg.cc/brKS5sMN/1.png)](https://postimg.cc/JH5hD4M9)

make sure all the prerequisites are "OK"

[![2.png](https://i.postimg.cc/ZK750tnJ/2.png)](https://postimg.cc/JGXLdFjF)


use the password for the db

[![3.png](https://i.postimg.cc/tgPqrs5g/3.png)](https://postimg.cc/SXSbRNH0)


select a name for the zabbix server and the time zone


[![4.png](https://i.postimg.cc/DZT2R7jP/4.png)](https://postimg.cc/d75PycKh)


at the end it will present to you the log in page. Remember user Admin and password zabbix (this are different from the DB)


[![5.png](https://i.postimg.cc/0Q08shhR/5.png)](https://postimg.cc/qhgfcDkj)



dashboard

[![6.png](https://i.postimg.cc/FKtmJ94X/6.png)](https://postimg.cc/CR493p5c)



make sure your agent server is green


[![7.png](https://i.postimg.cc/Nj2BMKZx/7.png)](https://postimg.cc/NKtWCjsy)



we almost dome, after this lets prepare the host for the agent, so when we create it will listen to it.
Go to hosts and in the upper right corner click on create host. You have to add the host to certain "tags" so the zabbix tool can recognize it as an agent. also you need to know the ip of the agent.

[![8.png](https://i.postimg.cc/YS176JrG/8.png)](https://postimg.cc/yJY2BpSs)


dont worry if it shows red, we havent install the agent on the kali



*
*
*

## zabbix agent


on kali download the platform for agent, and choose the distro version for your machine


[![9.png](https://i.postimg.cc/Y28pyK1Q/9.png)](https://postimg.cc/dDkPLfN0)



unarchive and update
```
sudo wget https://repo.zabbix.com/zabbix/7.2/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.2+debian12_all.deb
sudo dpkg -i zabbix-release_latest_7.2+debian12_all.deb
sudo apt update 

```



install the agent component and plugins
```
sudo apt install zabbix-agent2
sudo apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql 

```
*

Important step!!! go to the zabix config files in /etc/zabbix
and the file "zabbix_agent.config" file , we are going to change 2 lines "Server" (again you can filter with grep) that line is going to be poiting to localhost (127.0.0.1), which is saying "the zabbix server is the agent" and that is wrong. That ip should be the zabbix server ip, tha is the ubuntu server. This step is really important becasue if not the zabbix agent will be sendidng data to itselft and that does not makes sense becase the agent kali is not the server.


[![10.png](https://i.postimg.cc/Nf5wJyGw/10.png)](https://postimg.cc/N5qVLjrN)



this line, change for the ip of the ubuntu server( or your zabbix server)


[![11.png](https://i.postimg.cc/k5Fr6KsW/11.png)](https://postimg.cc/PLxV0LBq)







restart the services and enable
```
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2 
```


and we are done!! look:

[![12.png](https://i.postimg.cc/VktPw3bG/12.png)](https://postimg.cc/FYhBgCp3)


[![15.png](https://i.postimg.cc/43GqgnGy/15.png)](https://postimg.cc/Wd9XmNhc)

[![16.png](https://i.postimg.cc/rp9bFcyJ/16.png)](https://postimg.cc/nsCT0yDj)


*
*
*
---
## Troubleshooting

some troubleshootings are to make sure for the agent the agent config file the line "Server" is pointing to the zabbix server. If the agent is the one from the server itselft and is showing red taht means its config file Server is not pointing localhost and/or the plugins and services for the Mysql server arent installed


```
zabbix-server-mysql
zabbix-sql-scripts 
zabbix-agent2 
zabbix-agent2-plugin-mssql
```

another thing you can make sure if your port 10050 is open with netstat

[![17.png](https://i.postimg.cc/7Z0dd0pj/17.png)](https://postimg.cc/qNB1ChBG)

if not restart the services and if the problem persist go to the config file (if agent or server) and specify the line to listen in that port

---
also if you need to make a clean installation you can do with this 2 lines (either agent or server)
```
sudo apt remove "zabbix*" -y
sudo apt purge "zabbix*"

sudo apt remove "mysql-server*" -y

```
if you need to delete the zabbix DB, go to mysql cli root and show the list of the dabases

[![18.png](https://i.postimg.cc/9FvsLJNG/18.png)](https://postimg.cc/Xpk2JcWJ)


and do a:

```
DROP DATABASE zabbix

```

then you can recreate the zabbix DB
