# CyberTunnel
In this tutorial, I will guide you through the process of securely exposing a Local Area Network hosted website using reverse SSH tunneling with a cloud server.<br><br> This will work as an alternative to services like ngrok, and the cool part is that you'll create this on your own!<br><br> Reverse SSH tunneling is a powerful technique that enables devices behind NATs or firewalls to make their services available on the internet by leveraging a server with a public IP address. One of the key advantages of this approach is that it allows you to bring your website online even if you don't have port forwarding enabled on your local router. Additionally, this method provides a secure means of accessing local resources remotely and offers an effective solution for bypassing network restrictions. Follow along to learn how to set up and configure a reverse SSH tunnel for your LAN-hosted website.

## Requirements and Preparation
Before getting started with this tutorial, ensure that you have the following resources ready:
* A PC, laptop, or a Raspberry Pi running a website on the LAN
* A Cloud Server with a public IP address
* A domain/subdomain pointing to the Ubuntu Server's IP address

**Note:** I will be using a Raspberry Pi which is running a website on the LAN, and an Ubuntu Server with a public IP Address

## Setting up basic configuration
### Install OpenSSH server on Raspberry Pi and Ubuntu Server:
```
sudo apt update
sudo apt install openssh-server
```
### Create a new user on Ubuntu Server:
```
sudo adduser pi
```
### Create passwordless ssh-keygen on Raspberry Pi and Ubuntu Server:
```
sudo -u pi ssh-keygen -t rsa -b 4096 -f /home/pi/.ssh/id_rsa_passwordless -N ''
```
### Copy the key from Raspberry Pi to Ubuntu Server:
```
sudo -u pi ssh-copy-id -i /home/pi/.ssh/id_rsa_passwordless.pub pi@<SERVER-IP-ADDRESS>
```
### Copy the key from Ubuntu Server to Raspberry Pi:
On Ubuntu Server, copy the contents of "id_rsa_passwordless.pub"
```
sudo nano /home/pi/.ssh/id_rsa_passwordless.pub
```
Paste the content into "/home/pi/.ssh/authorized_keys" on the Raspberry Pi
## Setup startup service on Raspberry Pi
**Note:** I don't recommended using systemctl service for this, becauses I personally encountered a lot of problems using it, so we will be using Supervisor for this tutorial
### Install supervisor on Raspberry Pi:
```
sudo apt install supervisor
```
### Create configuration files for the services:
I'm creating two services, first one is "autom.conf" which is for my LAN-hosted website, and the second one is "reverse.conf" which is for my reverse-SSH-tunneling service
```
sudo nano /etc/supervisor/conf.d/autom.conf
sudo nano /etc/supervisor/conf.d/reverse.conf
```
**Note:** You will have to change these files according to your needs
### autom.conf
```
[program:autom]
command=/usr/bin/python3.9 /home/pi/automation/auto_gesture.py
user=pi
autostart=true
autorestart=true
stderr_logfile=/var/log/autom.service.err.log
stdout_logfile=/var/log/autom.service.out.log
priority=10
```
### reverse.conf
```
[program:reverse]
command=/usr/bin/ssh -NT -i /home/pi/.ssh/id_rsa_passwordless -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes -R 8082:localhost:5069 pi@<SERVER-IP-ADDRESS>
user=pi
autostart=true
autorestart=true
stderr_logfile=/var/log/reverse.err.log
stdout_logfile=/var/log/reverse.out.log
priority=20
```
### Start the services:
```
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all
```
## Setup Nginx on Ubuntu Server
### Install Install nginx on Ubuntu Server:
```
sudo apt install nginx
```
### Ensure no other web server (e.g., Apache) or service is running on port 80:
```
sudo lsof -i :80
sudo kill [PID]
sudo systemctl stop apache2
sudo systemctl disable apache2
```
### Create configuration file for nginx:
```
sudo nano /etc/nginx/sites-available/rpi
```
### Paste the following content in "rpi" file:
```
server {
    listen 80;
    server_name <DOMAIN-NAME>;

    location / {
        proxy_pass http://localhost:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
### Create a symbolic link to enable the site:
```
sudo ln -s /etc/nginx/sites-available/rpi /etc/nginx/sites-enabled/
```
### Restart nginx:
```
sudo systemctl restart nginx
```
## Make the website run on HTTPS
### Install Certbot:
```
sudo apt install -y certbot python3-certbot-nginx
```
### Obtain and install SSL/TLS certificate:
```
sudo certbot --nginx -d <DOMAIN-NAME>
```
### Certbot automatically sets up a renewal process via a systemd timer, test the renewal process by running:
```
sudo certbot renew --dry-run
```
## Troubleshooting
If the website is not accessible on https://<DOMAIN-NAME>, follow these steps:
### 1. Update SSH configuration on Ubuntu Server
Open the "sshd_config" file:
```
sudo nano /etc/ssh/sshd_config
```
Uncomment or change the following lines from no to yes:
```
AllowTcpForwarding yes
GatewayPorts yes
```
Restart SSH server to apply changes:
```
sudo systemctl restart ssh
```
### 2. Check if any firewall is blocking port 8082 on Ubuntu Server
```
sudo ufw allow 8082
```
### 3. Check if Raspberry Pi's web server is listening on port 5069 and bound to all available IP addresses (0.0.0.0)
```
sudo netstat -tuln | grep 5069
```
If the output is:
```
tcp        0      0 192.168.1.3:5069        0.0.0.0:*               LISTEN 
```
Change the IP address of your LAN-hosted website on Raspberry Pi to 0.0.0.0 (this will vary depending on how you are running your web application). If changed successfully, the output should be:
```
tcp        0      0 0.0.0.0:5069            0.0.0.0:*               LISTEN
```

## Conclusion
By following this tutorial, you have successfully set up a secure reverse SSH tunnel using a Raspberry Pi and an Ubuntu server. This configuration enables you to expose your LAN-hosted website on the internet even without port forwarding enabled on your local router. 
