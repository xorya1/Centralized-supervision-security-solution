
# Zabbix, Wazuh, Graylog, and Grafana Integration Project for security and supervision

## Overview
This repository contains the configuration and integration steps for setting up a comprehensive monitoring and logging solution using Zabbix, Wazuh, Graylog, and Grafana. This guide includes securing HTTP with HTTPS, adding custom items, and integrating FortiAnalyzer and FortiManager.

## Table of Contents
1. [Installation Guide](#installation-guide)
    - [Prerequisites](#prerequisites)
    - [Install Zabbix](#install-zabbix)
    - [Configure Zabbix with VMware](#configure-zabbix-with-vmware)
    - [Install Wazuh](#install-wazuh)
    - [Install Graylog](#install-graylog)
    - [Install Grafana](#install-grafana)
    - [Integrate Components](#integrate-components)
2. [Securing HTTP with HTTPS](#securing-http-with-https)
3. [Custom Items and Scripts](#custom-items-and-scripts)
4. [Conclusion](#conclusion)

## Installation Guide

### Prerequisites
- A server or virtual machine with a supported Linux distribution (e.g., Ubuntu 20.04).
- Root or sudo access to the server.
- Internet access to download necessary packages and dependencies.

### Install Zabbix

#### Add Zabbix Repository
```bash
wget https://repo.zabbix.com/zabbix/5.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_5.0-1+ubuntu20.04_all.deb
sudo dpkg -i zabbix-release_5.0-1+ubuntu20.04_all.deb
sudo apt update
```

#### Install Zabbix Server, Frontend, and Agent
```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-agent
```

#### Create Initial Database
```bash
sudo mysql -uroot -p
CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
exit
```

#### Configure Zabbix Server
Edit the Zabbix server configuration file:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Set the database configuration:
```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=password
```

#### Start Zabbix Server and Agent
```bash
sudo systemctl restart zabbix-server zabbix-agent
sudo systemctl enable zabbix-server zabbix-agent
```

#### Configure PHP for Zabbix Frontend
Edit the PHP configuration file:
```bash
sudo nano /etc/zabbix/php-fpm.conf
```
Set the timezone:
```
php_value[date.timezone] = Europe/London
```

#### Restart Apache Server
```bash
sudo systemctl restart apache2
```

### Configure Zabbix with VMware

#### Add a Client on Zabbix's Web Interface
1. Log in to Zabbix's web interface.
2. Navigate to "Configuration" -> "Hosts" -> "Create host".
3. Specify the IP address of vCenter and important macros such as URL, password, and username for the system administrator.

#### Add a Template to Import Hosts from vSphere Client
1. Navigate to "Configuration" -> "Templates".
2. Add a template that will take care of adding all different hosts and importing them from the vSphere client.

#### Create a Trigger Prototype for Datastore
1. Navigate to "Configuration" -> "Hosts".
2. Select the host and create a new trigger prototype for the datastore when the space is less than 5%.

### Install Wazuh

#### Add Wazuh Repository
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

#### Install Wazuh Manager
```bash
sudo apt install wazuh-manager
```

#### Start Wazuh Manager
```bash
sudo systemctl start wazuh-manager
sudo systemctl enable wazuh-manager
```

#### Install Wazuh API
```bash
sudo apt install wazuh-api
sudo systemctl start wazuh-api
sudo systemctl enable wazuh-api
```

### Install Graylog

#### Install Java
```bash
sudo apt install openjdk-11-jre-headless
```

#### Add Graylog Repository
```bash
wget https://packages.graylog2.org/repo/packages/graylog-4.0-repository_latest.deb
sudo dpkg -i graylog-4.0-repository_latest.deb
sudo apt update
```

#### Install Graylog Server
```bash
sudo apt install graylog-server
```

#### Configure Graylog Server
Edit the server configuration file:
```bash
sudo nano /etc/graylog/server/server.conf
```
Set the password secret and root password hash:
```
password_secret = somepasswordpepper
root_password_sha2 = <your-root-password-hash>
```

#### Start Graylog Server
```bash
sudo systemctl start graylog-server
sudo systemctl enable graylog-server
```

### Install Grafana

#### Add Grafana Repository
```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
```

#### Install Grafana
```bash
sudo apt install grafana
```

#### Start Grafana Server
```bash
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

## Securing HTTP with HTTPS

### Generate SSL Certificates
1. Generate SSL certificates using Let's Encrypt or a self-signed certificate.
2. For Let's Encrypt:
```bash
sudo apt install certbot
sudo certbot certonly --standalone -d yourdomain.com
```

### Configure Apache for HTTPS (Zabbix)
1. Edit the Apache configuration file for Zabbix:
```bash
sudo nano /etc/apache2/sites-available/zabbix.conf
```
2. Update the configuration to include SSL:
```apache
<VirtualHost *:443>
    ServerAdmin webmaster@localhost
    DocumentRoot /usr/share/zabbix
    ServerName yourdomain.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem

    <Directory /usr/share/zabbix>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/zabbix_error.log
    CustomLog ${APACHE_LOG_DIR}/zabbix_access.log combined
</VirtualHost>
```
3. Enable the SSL module and site, then restart Apache:
```bash
sudo a2enmod ssl
sudo a2ensite zabbix.conf
sudo systemctl restart apache2
```

## Integrate Components

### Integrate Zabbix with Grafana
1. In Grafana, go to Configuration -> Data Sources.
2. Add a new data source and select "Zabbix".
3. Configure the data source with the Zabbix API URL and credentials.

### Integrate Wazuh with Graylog
1. Configure Wazuh to send logs to Graylog by editing the Wazuh configuration file (`/var/ossec/etc/ossec.conf`):
```xml
<output>
  <graylog_output>
    <server>graylog-server-ip</server>
    <port>12201</port>
  </graylog_output>
</output>
```
2. In Graylog, create a new input to receive logs from Wazuh:
    - Navigate to "System -> Inputs" and select "GELF UDP".
    - Configure the input to match the settings in the Wazuh configuration.

### Configure Graylog to Forward Logs to Wazuh Indexer
1. In Graylog, go to "System -> Outputs" and create a new output to forward logs to Wazuh Indexer.
2. Configure the output with the appropriate Wazuh Indexer details.

### Integrate FortiAnalyzer and FortiManager with Graylog
1. Configure FortiAnalyzer and FortiManager to send syslog data to Graylog:
    - In FortiAnalyzer/FortiManager, navigate to Log Settings and configure syslog settings to point to the Graylog server IP and port.

### Display Data in Grafana
1. In Grafana, create new dashboards using data from Zabbix, Graylog, and Wazuh.
2. Add panels to the dashboards to display metrics such as security alerts, logs, server status, and network traffic.

## Custom Items and Scripts

### Custom Zabbix Items
1. Create custom Zabbix items to monitor specific metrics:
    - Go to "Configuration -> Hosts" in Zabbix.
    - Select a host and create new items with custom parameters and key values.

### Custom Grafana Dashboards
1. In Grafana, create dashboards tailored to specific monitoring needs:
    - Use the built-in query editor to fetch and display data from Zabbix, Graylog, and Wazuh.
    - Customize panels with different visualization options like graphs, tables, and alerts.

## Configuration Examples

### SSL Configuration for Zabbix
1. Edit the Apache configuration file for Zabbix to enforce HTTPS:
    ```bash
    sudo nano /etc/apache2/sites-available/zabbix.conf
    ```
2. Example SSL configuration:
    ```apache
    <VirtualHost *:80>
        ServerName yourdomain.com
        Redirect permanent / https://yourdomain.com/
    </VirtualHost>

    <VirtualHost *:443>
        ServerAdmin webmaster@localhost
        DocumentRoot /usr/share/zabbix
        ServerName yourdomain.com

        SSLEngine on
        SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem

        <Directory /usr/share/zabbix>
            Options FollowSymLinks
            AllowOverride None
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/zabbix_error.log
        CustomLog ${APACHE_LOG_DIR}/zabbix_access.log combined
    </VirtualHost>
    ```
3. Enable the SSL module and site, then restart Apache:
    ```bash
    sudo a2enmod ssl
    sudo a2ensite zabbix.conf
    sudo systemctl restart apache2
    ```

### Configure FortiGate with Zabbix
1. Download the FortiGate template:
    ```bash
    wget https://github.com/mbdraks/fortinet-zabbix/archive/master.zip
    unzip master.zip
    ```
2. Import the template into Zabbix:
    - Go to "Configuration -> Templates" and import the template XML file.
3. Add FortiGate as a host in Zabbix and link the imported template.

### Add Email Notifications in Zabbix
1. Configure the Zabbix server to use an SMTP server:
    - Go to "Administration -> Media types" and create a new media type for email.
2. Create an action to send email notifications:
    - Go to "Configuration -> Actions" and create a new action that triggers email notifications based on specific conditions.

### Create a Map in Zabbix
1. Go to "Monitoring -> Maps" and create a new map.
2. Add hosts to the map and define links between them.
3. Customize the map with icons and connection types.

### Wazuh Installation and Integration

#### Install Wazuh Indexer
1. Add the Wazuh repository and install the Wazuh indexer:
    ```bash
    sudo apt install wazuh-indexer
    ```
2. Configure the indexer by editing the configuration file:
    ```bash
    sudo nano /etc/wazuh-indexer/wazuh.yml
    ```

#### Install Wazuh Dashboard
1. Install the Wazuh dashboard:
    ```bash
    sudo apt install wazuh-dashboard
    ```
2. Start the Wazuh dashboard service:
    ```bash
    sudo systemctl start wazuh-dashboard
    sudo systemctl enable wazuh-dashboard
    ```

#### Deploy SSL Certificates for Wazuh
1. Generate or obtain SSL certificates for Wazuh components.
2. Configure Wazuh components to use the SSL certificates by editing their configuration files.

### Integration of Wazuh with Graylog and Grafana
1. Configure Wazuh to forward logs to Graylog.
2. Add Wazuh as a data source in Grafana to visualize security alerts and logs.

### Custom Items and Prototypes in Zabbix
1. Create custom items for monitoring specific metrics in Zabbix.
2. Create trigger prototypes to define conditions for alerts, such as datastore space thresholds.

### Index Management in Graylog
1. Manage indexes in Graylog to optimize log storage and retrieval.
2. Configure retention policies and index rotation settings.

## Conclusion
Following these detailed steps will set up a robust and integrated network monitoring and security solution using Zabbix, Wazuh, Graylog, and Grafana. This guide includes securing HTTP with HTTPS, custom items, and integration specifics tailored to your project's requirements. For more advanced configurations and troubleshooting, refer to the official documentation of each tool.
