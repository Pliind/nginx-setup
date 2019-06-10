# Guide on how to setup Nginx with Letsencrypt in Docker for intermediate users.
This guide is made with a Synology in mind, but will of course work on other systems / setups aswell.

If you are looking for encryption of Home assistant, Unifi Cloud Key or any other service you've come to the right place.

## Prerequisitions:
- Port 80 and 443 open and pointed to the IP Nginx will be using (this is set in the docker-compose file).
- Access to your admin-account on Synology
- A Synology NAS with an Intel-based CPU, (required for Docker)
- A domain you want to use* (i've also added an extra snipp of code further down if you are happy with only using duckdns)
- knowledge of using nano and basic directory navigation in linux

> *Domain-info.
> For every subdomain you need to make a CNAME pointer to your dynamic IP (for static IP use A pointer), don't forget to point subdomain @ also, orelse letencrypt will fail when validating domain.com (even if www.domain.com works)

Why Nginx? 
Nginx will place itself as an old-time telephone operator between the internet and your own network, simply speaking.
A request from hass.domain.com will make the operator (nginx) have a look inside its catalogue and find what internal IP it should direct you to, meanwhile keeping the connection encrypted.

>This is my first repo and I'm self taught in Linux, so please bear with me if some of my methods are the "slow way" 

## 1. Prep your NAS.
- Install Docker from the Package Center
- Enable SSH (Control Panel -> Terminal & SNMP -> Enable SSH Service.
- Download a suitable SSH-client to your PC, Putty for example
- Connect to your NAS (NASIP:22)
- login as admin and after login as root with commando `sudo -i` (enter your admin password again)
- I like keeping my docker-folder in root so change folder to root `cd /.` and create a directory for your docker-containers
. `mkdir docker`
- We also need to make folder for nginx, `cd docker`, `mkdir letsencrypt`, `cd letsencrypt`, `mkdir config`

```version: '2'

services:
  letsencrypt:
    container_name: letsencrypt
    image: linuxserver/letsencrypt
    restart: unless-stopped
    networks:
      nginx_network:
        ipv4_address: 192.168.1.33         #choose any IP of your choice
    ports:
     - 80:80          #open this port in your router and set it to the ip above
     - 443:443        #open this port in your router and 
    environment:
      - EMAIL=your@email.com        #optional, fill in if you want cert-notifications
      - TZ=Europe/Stockholm         #set your timezone
      - URL=example.com             #your url domain.com domain.duckdns.org etc.. 
      - SUBDOMAINS=,www,hass,unifi  #add any subdomains you are going to be using
      - VALIDATION=http              #make sure port 80 is open and clear 
    volumes:
     - /docker/letsencrypt/config:/config    #you will have to create this dir manually or you will get an error
    cap_add:
     - NET_ADMIN


networks:
  nginx_network:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24            # <-- Update to the same subnet your current network has.
```
> We create a new network and IP in the code above so we do not overlap any standard ports in the synology NAS or any other service that might need it. Placing your Letsencrypt container on its own IP makes it more flexible and enables you to have other services such as pihole on the same server without interference.

If you are not interested in using your own domain, and only using a duckdns.org domain instead is enough for you, use these environment paramters instead:
```
environment:
      - EMAIL=your@email.com        #optional, fill in if you want cert-notifications
      - TZ=Europe/Stockholm         #set your timezone
      - URL=example.com             #domain.duckdns.org  
      - SUBDOMAINS=wildcard  #add any subdomains you are going to be using
      - VALIDATION=duckdns             
      - DUCKDNSTOKEN=XXXXXXX #Insert your duckdnstoken here.
      
```
## 2. Create your container
- Next we want to give docker the commands for what to download and what settings to apply to the docker-container.
- `cd /docker` and create a file called docker-compose.yaml. `sudo nano docker-compose.yaml` paste the content from the file from this repo into it.
- Read all notes per setting and adjust to your own needs.
- When you're done, issue the command `docker-compose up -d` to create the container. you will also get feedback if something went wrong in creating the container in the terminal.
- Hopefully everything went well, you can also check the log in docker for any errors, the log will also feedback the success of creating cerfificates for your subdomains. `docker logs letsencrypt`

>Everytime you change something in your docker-compose, you need to issue the docker-compose command. `sudo docker-compose up -d`
```
server {
        listen 80 default_server;
        listen [::]:80;
        server_name _;
        return 301 https://$host$request_uri;
}

# main server block
server {
        listen 443 ssl default_server;

        root /config/www;
        index index.html index.htm index.php;

        server_name test.domain.com;   #no idea what this does?! 

        # enable subfolder method reverse proxy confs
        include /config/nginx/proxy-confs/*.subfolder.conf;

        # all ssl related config moved to ssl.conf
        include /config/nginx/ssl.conf;

        client_max_body_size 0;

        location / {
                try_files $uri $uri/ /index.html /index.php?$args =404;
        }
        location ~ \.php$ {
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                # With php7-cgi alone:
                fastcgi_pass 192.168.1.145:80;        #tbh not sure what this does yet :D
                # With php7-fpm:
                #fastcgi_pass unix:/var/run/php7-fpm.sock;
                fastcgi_index index.php;
                include /etc/nginx/fastcgi_params;
        }
}
# enable subdomain method reverse proxy confs
include /config/nginx/proxy-confs/*.domain.conf; ### Important! change domain to your own domain. 
# enable proxy cache for auth
proxy_cache_path cache/ keys_zone=auth_cache:10m;


```
## 3. Setting up Nginx
Now your server is up and running and all we need to do is tell nginx the where what and how.
- navigate to /docker/letsencrypt/config/nginx/site-confs and edit the file `sudo nano default`
- remove the contents and insert the contents of this repos default file.
- edit parameters that are marked up in the file.
- navigate to /docker/letsencrypt/config/nginx/proxy-confs
This folder is filled with examples to use, all you need to do is rename them and set them up to your  
- `ls` to list what's in the directory. 
- rename unifi for example.
- `mv unifi.subdomain.conf.sample unifi.domain.conf` #make sure to set domain to your domain
- edit the file: `sudo nano unifi.domain.conf`
```
# make sure that your dns has a cname set for unifi and that your unifi-controller container is not using a ba$

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name unifi.domain.com; #change to your domain

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    # enable for ldap auth, fill in ldap details in ldap.conf
    #include /config/nginx/ldap.conf;

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_unifi unifi-controller;
        proxy_pass https://192.168.1.4:8443; ## change to your internal IP
    }

    location /wss {
        # enable the next two lines for http auth
        #auth_basic "Restricted";
        #auth_basic_user_file /config/nginx/.htpasswd;

        # enable the next two lines for ldap auth
        #auth_request /auth;
        #error_page 401 =200 /login;

        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_unifi unifi-controller;
        proxy_pass https://192.168.1.4:8443; #change to local IP
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_ssl_verify off;
    }

}
```

Restart Letsencryt 
`docker container restart letsencrypt`

Test your setup by opening a incognito window.

>Disclaimer: this guide was made for me to empty my head, because i will have forgotten half this in a week.
