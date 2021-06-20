
# Docker-Compose Deployment

This document details a deployment of a stack of Docker Containers providing a production backend that serves a WordPress Application via the fastcgi Application Server to a Web Server with built-in fastcgi cache & lossless compression. 

Reverse Proxy performs secure Domain Name Validation, Auto TLS Management, Web Application Firewall & is paired with CloudFlare CDN. 

End-to-End-Encrypted Email Server via ProtonMail Secure Email. 

Automation & Orchestration by Docker-Compose including a Portainer Admin UI.

This deployment is comprised of the following technologies: 
```env
I)    Alpine Linux OS : Container Operating System
II)   Debian Linux OS : Container Operating System
III)  Maria DB        : Relational Database Management System ( RDBMS )
IV)   Redis DB Cache  : In-Memory, Schema-Less Database Object Cache
V)    PHP 7.4         : Hypertext Preprocessor ( General-Purpose Scripting Language )
VI)   WordPress 5.7   : Content Management System ( CMS )
VII)  Php-Fpm         : Application Server ( FastCGI Process Manager )
VIII) Nginx           : Web Server ( FastCGI_CACHE & Lossless Compression )
IX)   ProtonMail      : Mail Server ( E2EE-SMTP Server )
X)    Caddy V2        : Reverse Proxy ( HTTP/3 | TLS Management | Application Firewall  )
XI)   CloudFlare      : Content Delivery Network & Cloud Firewall ( CDN )
XII)  Portainer       : Container Management UI 
```

### Table of Content

* Architecture
* Requirements
* Preparation
* Clone Repository
* Define Environment Variables
* Create Required Directories
* Configure `Caddyfile`
* Deployment with Docker-Compose
* Proton Bridge Initialization
* WordPress Configuration
* Update `wp-config.php`
* Activate Plugins
* Stoppage of Deployment
* Removal of Containers & Images

### Architecture

![tsuki_architecture_image](https://user-images.githubusercontent.com/75030055/100444533-db329c00-30ab-11eb-8cde-68061e918231.png)

### Requirements

* KVM, LXC or Similiar Cloud Hypervisor
* OS: Linux (Debian 10) with Root Privileges
* Docker and Docker-Compose installed on the host machine
* A Fully Qualified Domain Name (FQDN)
* Cloudflare "Orange-Cloud" DNS Record with your FQDN pointing to your server’s public IP address

### Preparation

* [Register a Fully Qualified Domain Name (FQDN)](https://www.namecheap.com/support/knowledgebase/article.aspx/10072/35/how-to-register-a-domain-name)

* [Create an "Orange-Cloud" DNS Records](https://www.namecheap.com/support/knowledgebase/article.aspx/9607/2210/how-to-set-up-dns-records-for-your-domain-in-cloudflare-account/)

* [Ensure the `cloudflare_api_token` has Zone Edit Permissions for DNS.](https://sammckenzie.be/en/blog/using-caddy-with-cloudflare/) 

* [Sign-Up for a Paid ProtonMail account](https://protonmail.com/signup)

SSH into the host machine : 

    ssh root@PUBLIC_IP_ADDRESS


* [Create a secondary sudo user](https://linuxhint.com/create_new_sudo_user_debian10/)

* [Enable user public-key SSH access](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-debian-10)

* [Disable root SSH login](https://www.tecmint.com/disable-or-enable-ssh-root-login-and-limit-ssh-access-in-linux/)
    
Confirm Seconday User has SSH access from a new shell : 

    ssh NEW_USER@PUBLIC_IP_ADDRESS

* [Install Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10)

* [Install Docker-Compose](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-debian-10)

Install Uncomplicated Firewall (UFW) & git : 

    sudo apt install -y ufw git

Enable UFW :

    sudo ufw allow ssh
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow http
    sudo ufw allow https
    sudo systemctl enable ufw
    sudo ufw enable
    sudo ufw status verbose

### STEP 1: Clone Repository

    git clone https://github.com/techwise-technologies/tsuki-digital.git my-webshop
    cd my-webshop/

### STEP 2: Define Environment Variables

The `.env` file, stored as a hidden file in the main directory, requires your input. There is a `.env_example` that you can copy to start.

    cp .env_example .env

You now have a `.env` file. This file contains insecure default values for configuration options. 

During deployment this `.env` file is used by Your WordPress Application to initialize the configuration files. 

The values you input are used for secure authentication between the Services running within the Docker Containers. It is important to keep a copy of the values you input as only you have them.

    nano .env

This opens the `.env` file with the nano editor and you are now required to define certain values.

Replace all occurances of 'yourdomain_com', 'strong_password', 'very_strong_password' & 'very_very_strong_password' as desired and all occurances of 'email@yourdomain.com' & 'cloudflare_api_token' with your own email and token. 

Example `.env` file (default values):

```env
# Caddy v2
CADDY_CONF_DIR=./caddy/config
CADDY_DATA_DIR=./caddy/data
CADDYFILE=./caddy/Caddyfile
CADDY_VERSION=2.3.0

# ProtonMail
PROTON_BRIDGE_VERSION=stable

# CloudFlare 
CLOUDFLARE_EMAIL=email@yourdomain.com       <------------ EDIT THIS
CLOUDFLARE_AUTH_TOKEN=cloudflare_api_token  <------------ EDIT THIS

# Nginx
NGINX_CONF_DIR=./nginx/conf.d
NGINX_LOG_DIR=./logs/nginx
FASTCGI_CACHE_DIR=./cache
NGINX_VERSION=stable

# PHP Configs
TSUKI_PHP_CONF=./tsuki.ini

# WordPress
WORDPRESS_DB_PASSWORD=strong_password        <------------ EDIT THIS
WORDPRESS_VERSION=php7.4           
WORDPRESS_DB_NAME=yourdomain_com_wp          <------------ EDIT THIS   ## leave the trailing _wp 
WORDPRESS_DB_USER=yourdomain_com             <------------ EDIT THIS      
WORDPRESS_DATA_DIR=./wordpress
WORDPRESS_TABLE_PREFIX=wp_
WORDPRESS_DB_HOST=db

# Redis
REDIS_PASSWORD=very_very_strong_password      <------------ EDIT THIS
REDIS_CONF=./redis/redis.conf
REDIS_DATA_DIR=./redis/data/
REDIS_VAR_LIB=./redis/lib/
REDIS_VERSION=alpine

# MariaDB
MYSQL_ROOT_PASSWORD=very_strong_password      <------------ EDIT THIS 
MYSQL_DATABASE=yourdomain_com_wp              <------------ EDIT THIS   ## leave the trailing _wp 
MYSQL_PASSWORD=strong_password                <------------ EDIT THIS
MYSQL_USER=yourdomain_com                     <------------ EDIT THIS
DB_DATA_DIR=./mariadb
MARIADB_VERSION=10.5

```
Exit the file by pressing and holding `ctrl` + `x`. 

This will initiate a prompt that will ask if you wish to save the changes, press `Y` and `Enter`. 

You now have a `.env` file ready to use for deployment.

### STEP 3: Create Required Directories

    mkdir -p cache/ mariadb/ wordpress/ logs/nginx/

### STEP 4: Configure `Caddyfile`
   
    cd caddy/ && nano Caddyfile

This opens the  Caddyfile with the nano editor and you are now required to input your domain name.

Replace all occurances of 'yourdomain.com' with your desired Domain Name and all occurances of 'email@yourdomain.com' & 'cloudflare_api_token' with your own email and token.

```env
yourdomain.com, www.yourdomain.com {          <------------ EDIT THIS
  reverse_proxy nginx
  tls email@yourdomain.com                    <------------ EDIT THIS
  
  tls {
    dns cloudflare cloudflare_api_token       <------------ EDIT THIS
  }
}

cpanel.yourdomain.com {                     <------------ EDIT THIS
  reverse_proxy portainer:9000
  tls email@yourdomain.com                    <------------ EDIT THIS
  
  tls {
    dns cloudflare cloudflare_api_token       <------------ EDIT THIS
  }
}
```

Exit the file by pressing and holding `ctrl` + `x`.

This will initiate a prompt that will ask if you wish to save the changes, press `Y` and `Enter`. 

### STEP 5: Deployment with Docker-Compose

Time to deploy, run this command and watch the code execute. It may take a few minutes, be patient.

    cd .. && docker-compose up -d 

After a few moments your WordPress Application & Admin UI is ready to be configured.

    WordPress Application:  https://www.yourdomain.com     <------------ Complete the 1 minute WordPress installation
    Portainer Admin UI :    https://cpanel.yourdomain.com    
    

    [ Note : Keep a Copy of the Username & Password, this shall be the Admin credentials for the WordPress Application. ] 

Navigate to Your Admin Panel URL and Folow the Instructions to Create an Admin User. Keep a Copy of the Credentials. Log into the Admin UI.

### STEP 6 : Proton Bridge Initialization

In the Admin UI, go to the `Mail_Server` container and stop it temporarily. This is required to initialize and add your account to the bridge. In the shell run the following command. 

    docker run --rm -it -v my-webshop_protonmail:/root techwise-technologies/proton-bridge:stable init

Wait for the bridge to startup, use `login` command and follow the instructions to add your account into the bridge. Then use `info` to see the configuration information (take note of the username and password). After that, use `exit` to exit the bridge. You may need `CTRL`+`C` to exit the docker entirely.

The initialization step exposes the bridge CLI so you can do things like switch between combined and split mode, change proxy, etc. The [official guide](https://protonmail.com/support/knowledge-base/bridge-cli-guide/) gives more information on to use the CLI.

Once initialized, go back in the Admin UI & start the `Mail_Server` container.

### STEP 7: WordPress Configuration

Log into the WordPress Admin Panel.

Go To Plugins on the Left Menu & delete the 2 default options (Hello Dolly & Akismet). 

Add 3 New Plugins :

      I)    Redis Object Cache   | By Till Krüss 
     II)    Nginx Cache          | By Till Krüss 
    III)    WP Mail SMTP         | By WPForms

### STEP 8: Update `wp-config.php`

In the command prompt, navigate to the wordpress folder and open the `wp-config.php` in nano

    cd wordpress/ && sudo nano wp-config.php

Add The Following Settings just above the MYSQL Settings :

    // ** REDIS settings ** //
    define( 'WP_CACHE_KEY_SALT', 'wp-docker-redis');
    define( 'WP_REDIS_HOST', 'redis');
    define( 'WP_REDIS_PASSWORD', 'very_very_strong_password');      <------------ EDIT THIS

Replace `very_very_strong_password` with the SAME value you used previously for this key. 

Add This at the very Bottom of the file:

    /** Filesystem API Update Method */
    define('FS_METHOD','direct');

Exit the file by pressing and holding `ctrl` + `x`. 

This will initiate a prompt that will ask if you wish to save the changes, press `Y` and `Enter`. 

### STEP 9: Activate Plugins

In The WordPress Admin UI, Navigate To Plugins and Activate SMTP, Redis & Nginx Cache.

* Navigate to WP Mail SMTP Settings and fill in the required SMTP info under "Other SMTP"
    
    ```env
    SMTP Host:      protonmail
    Encryption:     None
    SMTP Port:      25
    AUTO TLS:       OFF
    Authenticaton:  ON
    SMTP Username:  "As per Proton-Bridge cli"
    SMTP Password:  "As per Proton-Bridge cli"
    ```
    `Save Settings`
    
* Navigate to Redis Settings and "Enable Object Caching"

* Navigate to Nginx Settings and Set the Path to "/tmp/cache"

    `Purge Cache`

### Stoppage of Deployment

    docker-compose down

### Removal of Containers & Images

WARNING : THIS REMOVES ALL CONTAINERS & NETWORKS & IMAGES !!!

    docker container prune
    docker image prune -a
    docker network prune
    docker volume prune

