**Welcome to the DevOps interview homework for LucidLink.**

**Task1: Create an init/systemd config that starts the LucidLink client on boot and stops it on shutdown. Ensure that the password is not visible on: ps ax**


I went ahead with a systemd config to manage the lucidlink client. Systemd is a (relatively new) system and service manager for Linux operating systems. It is designed to be backwards compatible with SysV init scripts, and provides several features such as parallel startup of system services at boot time, on-demand activation of daemons, or dependency-based service control logic.

The first thing I did was to create the config file in ```/etc/systemd/system``` I named it ```joro.service```.

The service looks the following:
```
root@ubuntu-task-georgi:/etc/systemd/system# cat joro.service
[Unit]
Description=Startup of the lucid client service
After=network.target

[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/usr/local/sbin/start.sh
ExecStop=lucid exit


[Install]
WantedBy=multi-user.target
```

From top to bottom, Description is self explanatory, After=network.target entry was added as a good measure to run the service once, we have network connectivity. I did find an interesting read regarding this here. In the [Service] portion type=simple It is generally recommended to use Type= simple for long-running services whenever possible, as it is the simplest and fastest option. RemainAfterExit=yes if this option is used without RemainAfterExit= the service will never enter "active" unit state, but directly transition from "activating" to "deactivating" or "dead" since no process is configured that shall run continuously. ExecStart and ExecStop are used to call the start script and stop the service gracefully during a stop command, restart or shutdown.

All the scripts I used to control the lucid client are in ```/usr/local/sbin/start```.

In there we have three files:
```
pass.txt
lucid.sh
start.sh
```

```pass.txt``` - I did hit a few issues here, in the end, I decided to go with a plain text password that’s only readable by the root user (chmod 400). Ideally, this can be managed in a few ways the password can be parsed from an ansible secret or hashicorp vault. There’s also another way to do this natively with systemd credentials (https://systemd.io/CREDENTIALS/) but I couldn’t do that in the time I had. For production-ready environment I would ideally use the systemd credentials way.

```lucid.sh``` - Here we have the command you provided to start the lucid client.

```start.sh```  - is the wrapper that’s being called by the systemd service. Its function is to pass the password file to the script to fill in the password prompt. This is the workaround I found for the process not to include the password in plain text when you do a ps ax:
```
uzunov@ubuntu-task-georgi:~$ ps ax | grep 'lucid'
    728 ?        S      0:00 /bin/bash /usr/local/sbin/lucid.sh
    737 ?        S      0:00 /bin/bash /opt/lucidapp/resources/Lucid daemon --fs test1.lucid --user testuser
    769 ?        Sl     0:01 ./Lucid.bin daemon --fs test1.lucid --user testuser
   1311 pts/1    S+     0:00 grep --color=auto lucid
```

After both scripts were created they had to be made executable, but only by the root user (chmod 744).

With all this I think I have met all the requirements, the service starts when the VM is restarted on its own and it stops gracefully. Password is not visible in processes.



**Task2: Export the REST interface to be accessible from the outside in a secure way. You can choose the needed tools.**

To make the ```localhost:7778``` api accessible over the internet I went with Nginx and a reverse proxy.

The webserver can be installed with apt install nginx, after that we need to make sure that the service is up and running, which we can do with systemctl status nginx.service.

Once we confirm everything is operational, we can go ahead and add a config for the reverse proxy. I found a really good video on YouTube regarding this. By default, we don’t have a config file present in /etc/nginx/conf.d/, so I went and created one called lucid_api.conf. I also had to install apt install apache2-utils. With the help of apatche2-utils I created an encrypted password and username that’s passed in the config below. The code can be found below:
```
root@ubuntu-task-georgi:/etc/nginx/conf.d# cat lucid_api.conf
server {
  listen 80;
  listen [::]:80;

  server_name 164.92.237.87;

  auth_basic "Admin only";
  auth_basic_user_file /etc/htpasswd/niki;

  location / {
      proxy_pass http://localhost:7778/;
  }
}
```

Once the config is in place we need to do a reload with service nginx reload and a restart of the service to reload the settings with systemctl restart nginx. Once everything is set, we can do a curl to the machine with the following command:

```curl -u niki:niki 164.92.237.87/app/status```

I can confirm we did receive the same information we had when we execute the command on the VM itself.

The information can be also accessed by a web browser going to ```http://164.92.237.87/app/status```

You will require a username and password, both of them are niki.

Ideally, I wanted to add HTTPS and an SSL certificate, however, I couldn’t find a way to do this for a bare IP address, as certificates are bound to the domain, and buying a domain wasn’t part of the tasks. In the production environment I don’t think anyone would be using their public IP, so this can be handled then. We can use something like certbot.

**Task3: Secure the VM.
**
To secure the VM I decided to go with ufw (uncomplicated firewall). It comes pre-installed, so all we had to do is set some rules. By default, UFW is set to deny all incoming connections and allow all outgoing connections. This means anyone trying to reach your server would not be able to connect, while any application within the server would be able to reach the outside world. We need to enable ssh otherwise we might get locked out of the machine if we miss this. This is well written here.
```
ufw allow ssh
```

We also need to ufw allow Nginx HTTP to enable Nginx connections.

To make this setup production ready, I would ideally have a list of IPs that I would like to enter in the settings here:
```
root@ubuntu-task-georgi:~# ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22                         ALLOW IN    Anywhere
[ 2] 80                         ALLOW IN    Anywhere
[ 3] 22 (v6)                    ALLOW IN    Anywhere (v6)
[ 4] 80 (v6)                    ALLOW IN    Anywhere (v6)
```

For example, we can enable ssh only for a specific set of machines that will manage this VM, the HTTP connections would ideally be accessible to only a list of machines, not Anywhere, how it’s the current setup. However, I don’t have a list of IPs that would like to enable this machine currently, so I left all the settings open for the review of the task to be completed. However, this is not a good practice. All other connections on the rest of the ports should be handled and denied by default on the machine.UFW is configured to deny all incoming connections by default.


**The best way to replicate this setup is to have an ansible playbook.
**

That's why after I finished manually setting the steps above I decided to create a playbook, called playbook. yaml found in the ansible folder of this repository.

This folder contains the following:
- All the scripts and files we need to create to make the above setup automated.
- The actual playbook.
- An inventory file.

I've set the same files in ~/ansible (root home folder /ansible). The ansible playbook can be run from there.

I would not call this production ready, but this is what I've come up with for the time I had to spend on the task.

ToDo list can be found [HERE](https://docs.google.com/document/d/1jjqd08yTLuAO7PlEWstrWnCDLa9KcLaNFaTHsTI35WQ/edit?usp=sharing "HERE").
