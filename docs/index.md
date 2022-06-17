# Metabase migration

VPS Provider: Amazon (AWS Lightsail)

Linux Distribution: Ubuntu 20.04 LTS

Machine Type: 1 vCPU, 1 GB Memory, 40 GB SSD (amd64)

Metabase Version: 0.41.7

## Prepare OS

    sudo apt-get update -y
    sudo apt-get upgrade -y

## Creating A Swap Partition

For general information on swap partitions check out the [Ubuntu Documentation](https://help.ubuntu.com/community/SwapFaq)

The following steps will create a 1 gigabyte (1024MB) Swap partition. It is recommended to allocate 1 gigabyte per concurrent session you expect to run at any given time. Please adjust according to your needs.

    sudo dd if=/dev/zero bs=1M count=1024 of=/mnt/1GiB.swap
    sudo chmod 600 /mnt/1GiB.swap
    sudo mkswap /mnt/1GiB.swap
    sudo swapon /mnt/1GiB.swap

Verify swap file exists

    cat /proc/swaps

To make the swap file available on boot

    echo '/mnt/1GiB.swap swap swap defaults 0 0' | sudo tee -a /etc/fstab

## Install Java JRE

You may already have Java installed. To check the version, open a terminal and run:

    java -version

If Java isn’t installed, follow the steps to install the recommended latest LTS version of JRE from [Eclipse Temurin](https://adoptium.net/) with HotSpot JVM and x64 architecture.

1.  Ensure the necessary packages are present:

    ```
    apt-get install -y wget apt-transport-https
    ```

1.  Download the Eclipse Adoptium GPG key:

    ```
    wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /usr/share/keyrings/adoptium.asc
    ```

1.  Configure the Eclipse Adoptium apt repository:

    ```
    echo "deb [signed-by=/usr/share/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
    ```

1.  Install the Temurin version 11:

    ```
    apt-get update # update if you haven't already
    apt-get install temurin-11-jdk
    ```

## Install MariaDB & Setup Database

### Step 1 — Installing MariaDB

Before you install MariaDB, update the package index on your server with `apt`:

    sudo apt update

Then install the package:

    sudo apt install mariadb-server

When installed from the default repositories, MariaDB will start running automatically. To test this, check its status.

    sudo systemctl status mariadb

If MariaDB isn’t running, you can start it with the command `sudo systemctl start mariadb`.

### Step 2 — Configuring MariaDB

Run the security script that came installed with MariaDB. This will take you through a series of prompts where you can make some changes to your MariaDB installation’s security options:

    sudo mysql_secure_installation

The first prompt will ask you to enter the current database `root` password. Since you have not set one up yet, press `ENTER` to indicate “none”.

    Output
    . . .
    Enter current password for root (enter for none):

The next prompt asks you whether you’d like to set up a database `root` password. On Ubuntu, the `root` account for MariaDB is tied closely to automated system maintenance, so we should not change the configured authentication methods for that account. Type `N` and then press `ENTER`.

    Output
    . . .

    Set root password? [Y/n] N

From there, you can press `Y` and then `ENTER` to accept the defaults for all the subsequent questions. This will remove some anonymous users and the test database, disable remote `root` logins, and then load these new rules.

### Step 3 — Creating an Administrative User that Employs Password Authentication

On Ubuntu systems running MariaDB 10.3, the `root` MariaDB user is set to authenticate using the `unix_socket` plugin by default rather than with a password. Because the server uses the `root` account for tasks like log rotation and starting and stopping the server, it is best not to change the `root` account’s authentication details. Instead, the package maintainers recommend creating a separate administrative account for password-based access.

To this end, open up the MariaDB prompt from your terminal:

    sudo mariadb

Then create a new user with `root` privileges and password-based access. Be sure to change the username and password to match your preferences:

    GRANT ALL ON *.* TO 'admin'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;

Flush the privileges to ensure that they are saved and available in the current session:

    FLUSH PRIVILEGES;

Create a database for Metabase instance

    CREATE DATABASE metabase_db;

Following this, exit the MariaDB shell:

    exit

You can test out this new user with the `mysqladmin` tool, a client that lets you run administrative commands. The following `mysqladmin` command connects to MariaDB as the `admin` user and returns the version number after prompting for the user’s password:

    mysqladmin -u admin -p version

## Migrate Metabase database from existing server

Run the following command in the existing Metabase server to backup Metabase MySQL database:

    mysqldump --user=user_name --password='password' --quick metabase_db | gzip > /tmp/metabase_db.sql.gz

Copy the backup file to new Metabase server:

    scp -P 22 -r /tmp/metabase_db.sql.gz ubuntu@0.0.0.0:/home/ubuntu/

Import MySQL DB on the new server:

    gunzip < /home/ubuntu/metabase_db.sql.gz | mysql -u admin -p'password' metabase_db

## Create an unprivileged user to run Metabase and give him acces to app and logs

For security reasons we want to have Metabase run as an unprivileged user. We will call the user simply `metabase`. Further we will create the files we will need later for logging and configuration of Metabase, and apply the correct security settings for our unprivileged user.

    sudo groupadd -r metabase
    sudo useradd -r -m -d /home/metabase -s /bin/false -g metabase metabase
    sudo chown -R metabase:metabase /home/metabase
    sudo touch /var/log/metabase.log
    sudo chown syslog:adm /var/log/metabase.log
    sudo touch /etc/default/metabase
    sudo chmod 640 /etc/default/metabase

## Download Metabase

Download the Metabase JAR file:

    wget -O /home/metabase/metabase.jar https://downloads.metabase.com/v0.41.7/metabase.jar

## Create a Metabase Service

Create the `/etc/systemd/system/metabase.service` service file and open it in your editor:

```
sudo touch /etc/systemd/system/metabase.service
sudo nano /etc/systemd/system/metabase.service
```

```
[Unit]
Description=Metabase server
After=syslog.target
After=network.target

[Service]
WorkingDirectory=/home/metabase
ExecStart=/usr/bin/java -jar /home/metabase/metabase.jar
EnvironmentFile=/etc/default/metabase
User=metabase
Type=simple
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=metabase
SuccessExitStatus=143
TimeoutStopSec=120
Restart=always

[Install]
WantedBy=multi-user.target
```

## Create syslog conf

Next we need to create a syslog conf to make sure systemd can handle the logs properly.

    sudo touch /etc/rsyslog.d/metabase.conf
    sudo nano /etc/rsyslog.d/metabase.conf
        
    if $programname == 'metabase' then /var/log/metabase.log
    & stop

Restart the syslog service to load the new config.

    sudo systemctl restart rsyslog.service

## Environment Variables for Metabase

Open your `/etc/default/metabase` environment config file in your editor:

```
sudo nano /etc/default/metabase
```

```
MB_DB_TYPE=mysql
MB_DB_DBNAME=metabase_db
MB_DB_PORT=3306
MB_DB_USER=admin
MB_DB_PASS=password
MB_DB_HOST=127.0.0.1

JAVA_VERSION=jdk-11.0.15+10
JAVA_HOME=/usr/lib/jvm/temurin-11-jdk-amd64

JAVA_OPTS=-Xmx512m
```

## Register & Start Metabase service

    sudo systemctl daemon-reload
    sudo systemctl start metabase.service
    sudo systemctl status metabase.service

Enable the service to startup during boot.

    sudo systemctl enable metabase.service

Now, whenever you need to start, stop, or restart Metabase:

    sudo systemctl start metabase.service
    sudo systemctl stop metabase.service
    sudo systemctl restart metabase.service

## (Optional) Nginx proxy & Let’s Encrypt SSL Certificate

Install Nginx and Let’s Encrypt Certbot client

    sudo apt-get install nginx
    sudo apt-get install certbot
    sudo apt-get install python3-certbot-nginx

Assuming you’re starting with a fresh NGINX install, use a text editor to create a file in the `/etc/nginx/conf.d` directory named `domain-name.conf` (replace domain-name with the name of your domain).

    sudo nano /etc/nginx/conf.d/domain-name.conf

Specify your domain name (and variants, if any) with the server_name directive:

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/html;
        server_name domain-name.com www.domain-name.com;
        location / {
            proxy_pass http://127.0.0.1:3000;
        }
    }

Save the file, then run this command to verify the syntax of your configuration and restart NGINX:

    nginx -t && nginx -s reload

The NGINX plug‑in for certbot takes care of reconfiguring NGINX and reloading its configuration whenever necessary.
Run the following command to generate certificates with the NGINX plug‑in:

    sudo certbot --nginx -d domain-name.com -d www.domain-name.com

Respond to prompts from certbot to configure your HTTPS settings, which involves entering your email address and agreeing to the Let’s Encrypt terms of service.

When certificate generation completes, NGINX reloads with the new settings. certbot generates a message indicating that certificate generation was successful and specifying the location of the certificate on your server.

Let’s Encrypt certificates expire after 90 days. The `certbot` package we installed takes care of this for us by adding a systemd timer that will run twice a day and automatically renew any certificate that’s within thirty days of expiration.

You can query the status of the timer with `systemctl`:

    sudo systemctl status certbot.timer

To test the renewal process, you can do a dry run with `certbot`:

    sudo certbot renew --dry-run

If you see no errors, you’re all set. When necessary, Certbot will renew your certificates and reload Nginx to pick up the changes. If the automated renewal process ever fails, Let’s Encrypt will send a message to the email you specified, warning you when your certificate is about to expire.

Auto-Renewal using cron job. Here we add a cron job to an existing crontab file to renew your certificates automatically.

Open the crontab file.

    crontab -e

Add the certbot command to run daily. The command checks to see if the certificate on the server will expire within the next 30 days, and renews it if so. The --quiet directive tells certbot not to generate output. All installed certificates will be automatically renewed and reloaded.

    0 12 * * * /usr/bin/certbot renew --quiet

## (Optional) Configuring Firewall Rules with UFW

This section covers enabling and configuring UFW (UncomplicatedFirewall). [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall) is the default firewall management tool on Ubuntu and is also available on Debian and Fedora. It operates as a easy to use front-end for iptables.

If UFW is not installed, install it now using `apt` or `apt-get`.

    sudo apt update
    sudo apt install ufw

Add firewall rules to allow ssh (port 22) connections as well as http (port 80) and https (port 443) traffic.

    sudo ufw allow ssh
    sudo ufw allow http
    sudo ufw allow https

Enable UFW if its not already enabled.

    sudo ufw enable

Verify that UFW is enabled and properly configured for ssh and web traffic.

    sudo ufw status

# Official Installation Guide
  - [Running Metabase on Debian as a service with nginx](https://www.metabase.com/docs/latest/operations-guide/running-metabase-on-debian.html)
  - [Running the Metabase JAR file](https://www.metabase.com/docs/latest/operations-guide/running-the-metabase-jar-file.html)
