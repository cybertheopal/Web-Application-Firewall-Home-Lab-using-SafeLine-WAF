# Web Application Firewall Home Lab using SafeLine WAF

<a name="1-lab-context"></a>
##  1. Lab Context

In this lab, I set up an isolated virtual environment to simulate web application attacks against the Damn Vulnerable Web Application (DVWA) hosted on an Ubuntu VM. The primary goal was to deploy, configure, and test the SafeLine Web Application Firewall (WAF) to evaluate its effectiveness in detecting and blocking common web-based threats.

---

#  Table of Contents 

-  [1. Lab Context](#1-lab-context)
- 🖥 [2. Lab Environment Setup](#2-lab-environment-setup)
-  [3. Lab Networking Setup](#3-lab-networking-setup)
-  [4. Ubuntu Server Configuration](#4-ubuntu-server-configuration)
  - [4.1. Initial System Updates and Installations](#41-initial-system-updates-and-installations)
  - [4.2. Installing and Configuring LAMP Stack](#42-installing-and-configuring-lamp-stack)
  - [4.3. Installing and Configuring Damn Vulnerable Web App (DVWA)](#43-installing-and-configuring-damn-vulnerable-web-app-dvwa)
  - [4.4. Changing the DVWA Listening Port to 8080](#44-changing-the-dvwa-listening-port-to-8080)
  - [4.5. Adding Custom Values to the DVWA Database](#45-adding-custom-values-to-the-dvwa-database)
-  [5. DNS Resolution Setup](#5-dns-resolution-setup)
-  [6. Creating a Self-Signed SSL Certificate](#6-creating-a-self-signed-ssl-certificate)
-  [7. Installing and Configuring SafeLine WAF](#7-installing-and-configuring-safeline-waf)
  - [7.1. Importing the Self-Signed Certificate](#71-importing-the-self-signed-certificate)
  - [7.2. Onboarding the DVWA Application](#72-onboarding-the-dvwa-application)
-  [8. Demonstrating SQL Injection from Kali Linux](#8-demonstrating-sql-injection-from-kali-linux)
  - [8.1. Launching the Attack](#81-launching-the-attack)
  - [8.2. Observing SafeLine WAF Protection](#82-observing-safeline-waf-protection)
-  [9. SafeLine WAF Advanced Configurations](#9-safeline-waf-advanced-configurations)
  - [9.1. HTTP Flood Defense](#91-http-flood-defense)
  - [9.2. Authentication Sign-In](#92-authentication-sign-in)
  - [9.3. Custom Deny Rules (Blocking Kali IP)](#93-custom-deny-rules-blocking-kali-ip)
-  [10. Conclusion and Lessons Learnt](#10-conclusion-and-lessons-learnt)

---
<a name="2-lab-environment-setup"></a>
##  2. Lab Environment Setup 

•	VMWare Workstation Pro<br>
•	Kali Linux<br>
•	Ubuntu Server<br>
•	SafeLine Web Application Firewall<br>
•	OpenSSL<br>
•	Apache2, PHP, and MySQL (LAMP Stack)<br>
•	Damn Vulnerable Web App (DVWA)<br>

<a name="3-lab-networking-setup"></a>
## 3. Lab Networking Setup

I initially set the Ubuntu VM’s network adapter to Bridged mode, giving it direct internet access through my physical network. This allowed me to easily download and install required packages like Apache, PHP, and DVWA. Bridged networking simplified the setup by providing full connectivity.

Once installation and configuration were complete, I shut down the Ubuntu VM and switched its network adapter to Host-Only mode. The Kali Linux VM was also set to Host-Only, creating an isolated virtual network. This setup ensured both VMs could communicate with each other, while the vulnerable Ubuntu VM remained cut off from the internet and local network, reducing exposure to real-world threats.

Inside this isolated environment, I positioned the Safeline firewall between the two VMs to monitor and control traffic. Kali was used to launch simulated attacks against the DVWA server, allowing me to test the firewall’s rules and confirm they could detect and block malicious activity.

This hybrid approach combined the convenience of bridged networking for setup with the security of host-only networking for controlled attack testing, enabling a safe and effective evaluation of the Safeline firewall.

<a name="4-ubuntu-server-configuration"></a>
##  4. Ubuntu Server Configuration 

<a name="41-initial-system-updates-and-installations"></a>
### 4.1. Initial System Updates and Installations 

<img src="https://i.imgur.com/oibxSVB.png">

I started by updating the package list and upgrading existing packages on the Ubuntu VM using the following commands:

```
sudo apt-get update
sudo apt-get upgrade -y
```

This ensured the system was up to date before installing any additional software.

<img src="https://i.imgur.com/Q14YgcN.png">

Next, I installed the net-tools package to access networking commands like ifconfig:

```
sudo apt-get install -y net-tools
```

This helped me check and manage network settings during the lab setup.

<img src="https://i.imgur.com/dBnB8jK.png">

After that, I installed OpenSSL to enable secure communication and certificate management:

```
sudo apt-get install -y openssl
```

This was useful for testing HTTPS connections and encryption-related tasks in the lab.

<a name="42-installing-and-configuring-lamp-stack"></a>
### 4.2. Installing and Configuring LAMP Stack 

#### Installing Apache2, PHP, and MySQL:

<img src="https://i.imgur.com/0KEOoYl.png">

Next, I installed the core components of the LAMP stack – Apache2, PHP, and MySQL – using the following command:

```
sudo apt-get install -y apache2 php php-mysql mysql-server
```

This set up the web server environment needed to host DVWA on the Ubuntu VM.

<a name="43-installing-and-configuring-damn-vulnerable-web-app-dvwa"></a>
### 4.3. Installing and Configuring Damn Vulnerable Web App (DVWA)

<img src="https://i.imgur.com/lSMntTp.png">

After setting up the LAMP stack, I installed Damn Vulnerable Web Application (DVWA). I navigated to the web server’s root directory and cloned the DVWA repository using:

```
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
```

This downloaded the DVWA files into the Apache web directory, making them accessible through the browser.

#### Set File Permissions

<img src="https://i.imgur.com/Z6ibsIP.png">

Next, I set the correct file permissions for the DVWA directory to ensure Apache could access and serve the files properly:

```
sudo chown -R www-data:www-data DVWA
sudo chmod -R 755 DVWA
```

This assigned ownership to the Apache user (www-data) and set appropriate read, write, and execute permissions.

#### Configure DVWA Database

<img src="https://i.imgur.com/ErlXDmi.png">

Then, I configured the DVWA database settings by editing the configuration file:

```
sudo nano /var/www/html/DVWA/config/config.inc.php
```

Inside the file, I updated the database parameters as needed:

```
$DBMS = 'MySQL';
$db   = 'dvwa';
$user = 'dvwa_user';
$pass = 'p@ssw0rd';
$host = 'localhost';
```

These settings ensure DVWA can connect to the MySQL database using the correct credentials.

#### Created a new database and user in MySQL

<img src="https://i.imgur.com/8Q6wO0y.png">

Next, I created the DVWA database and a dedicated user in MySQL. I accessed the MySQL console as root:

```
sudo mysql -u root -p
```

Then, I ran the following commands to set up the database and user:

```
CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL ON dvwa.* TO 'dvwa_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

This created a database named dvwa, a user with the necessary permissions, and applied the changes.

#### Initialised DVWA

<img src="https://i.imgur.com/VEkEYRq.png">

<img src="https://i.imgur.com/SDknVfi.png">

Then, I opened a browser and navigated to:

```
http://<Ubuntu IP>/DVWA/setup.php
```

On the setup page, I clicked Create / Reset Database to initialise DVWA and prepare it for use.

<a name="44-changing-the-dvwa-listening-port-to-8080"></a>
### 4.4. Changing the DVWA Listening Port to 8080

<img src="https://i.imgur.com/DXjbYcD.png">

Next, I changed the DVWA listening port from the default port 80 to port 8080. To do this, I edited the Apache ports configuration file:

```
sudo nano /etc/apache2/ports.conf
```

Inside the file, I changed the line:

```
Listen 80
```

To

```
Listen 8080
```

This told Apache to listen on port 8080 instead of the default.


<img src="https://i.imgur.com/haKm9aD.png">

Next, I updated the default Apache virtual host to match the new port. I opened the config file:

```
sudo nano /etc/apache2/sites-available/000-default.conf
```

Then, I changed

```
<VirtualHost *:80>
```

To

```
<VirtualHost *:8080>
```

After saving, I restarted Apache to apply the changes:

```
sudo systemctl restart apache2
```

Now, DVWA was accessible at:

```
http://<Ubuntu IP>:8080/DVWA
```

<a name="45-adding-custom-values-to-the-dvwa-database"></a>
### 4.5. Adding Custom Values to the DVWA Database 

<img src="https://i.imgur.com/Bsufxky.png">

To prepare for SQL injection testing, I added some custom data to the DVWA database. First, I logged into MySQL:

```
sudo mysql -u root -p
```

Then, I selected the dvwa database and created a sample table with some entries:

```
USE dvwa;

CREATE TABLE test_users (
  id INT NOT NULL AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  password VARCHAR(50) NOT NULL,
  PRIMARY KEY (id)
);

INSERT INTO test_users (username, password) VALUES
  ('alice', 'alice123'),
  ('bob', 'bob123'),
  ('admin', 'admin123');

EXIT;
```

I created a new table called test_users in the dvwa database with columns for id, username, and password. Then, I added three users—alice, bob, and admin—with their passwords. After that, I exited the MySQL shell.

These entries gave me targets for practicing SQL injection attacks within DVWA.

<a name="5-dns-resolution-setup"></a>
##  5. DNS Resolution Setup

<img src="https://i.imgur.com/fufequl.png">

<img src="https://i.imgur.com/xuUxQWl.png">

To simplify access, I edited the /etc/hosts file on both the Ubuntu and Kali VMs:

```
sudo nano /etc/hosts
```

I added this line (replacing <Ubuntu IP> with the actual IP address):

```
<Ubuntu IP> dvwa.local
```

Now, from the Kali VM, I could access DVWA using the URL:

```
http://dvwa.local:8080/DVWA
```

Making it easier to remember and connect to the app.

<a name="6-creating-a-self-signed-ssl-certificate"></a>
##  6. Creating a Self-Signed SSL Certificate

<img src="https://i.imgur.com/Bc2o7tF.png">

On the Ubuntu VM, I created a self-signed SSL certificate to enable HTTPS for DVWA. First, I made a directory to store the certificate files:

```
sudo mkdir /etc/ssl/dvwa
```

Then, I generated the certificate and private key with this command:

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/dvwa/dvwa.key \
  -out /etc/ssl/dvwa/dvwa.crt
```

This created a 1 year valid certificate and key for securing the web app.

<a name="7-installing-and-configuring-safeline-waf"></a>
##  7. Installing and Configuring SafeLine WAF

<img src="https://i.imgur.com/l5GTRQD.png">

SafeLine offers an automatic install script, so on the Ubuntu VM, I ran this command in the terminal:

```
bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```

I then followed the on-screen prompts to complete the installation.<br>
At the end, I received an admin username, password, and the URL to access SafeLine, on port 9443.<br>
Once logged in, I went ahead and upgraded to the PRO license to unlock the full set of features.<br>

<a name="71-importing-the-self-signed-certificate"></a>
### 7.1. Importing the Self-Signed Certificate

<img src="https://i.imgur.com/yJdbfzU.png">

Next, in the SafeLine WAF interface, I imported the self-signed certificate I created earlier. 

Under the SSL certificate section, I uploaded these files:

```
Certificate File: /etc/ssl/dvwa/dvwa.crt
Private Key File: /etc/ssl/dvwa/dvwa.key
```

<a name="72-onboarding-the-dvwa-application"></a>
### 7.2. Onboarding the DVWA Application 

<img src="https://i.imgur.com/njbAzRD.png">

Then, I used the SafeLine WAF management console to add a new application:

1.	Set the DNS name to www.dvwa.local<br>
2.	Set the Backend URL (reverse proxy) to http://<UbuntuIP>:8080.<br>
3.	Removed port 80 and kept only port 443.<br>
4.	Attached the SSL certificate so the WAF could serve HTTPS traffic.<br>

This configured SafeLine to protect and securely proxy traffic to DVWA.

<a name="8-demonstrating-sql-injection-from-kali-linux"></a>
##  8. Demonstrating SQL Injection from Kali Linux

<a name="81-launching-the-attack"></a>
### 8.1. Launching the Attack 

<img src="https://i.imgur.com/SXiUlg6.png">

Next, I opened Kali Linux and navigated to the DVWA site at: http://dvwa.local/

This automatically redirected to HTTPS through the SafeLine WAF.

Then, I tried the classic SQL injection string ' OR '1=1 on the DVWA login form to see if the WAF would detect and block the attack.

<a name="82-observing-safeline-waf-protection"></a>
### 8.2. Observing SafeLine WAF Protection

<img src="https://i.imgur.com/nDfuBIn.png">

<img src="https://i.imgur.com/KftQnps.png">

SafeLine WAF successfully detected and blocked the malicious SQL injection attempts. I checked the WAF logs to confirm that the injection tries were caught. When the WAF was set to blocking mode, I also observed error or block pages displayed instead of the normal DVWA response.

<a name="9-safeline-waf-advanced-configurations"></a>
##  9.SafeLine WAF Advanced Configurations

<a name="91-http-flood-defense"></a>
### 9.1. HTTP Flood Defense

<img src="https://i.imgur.com/MyibhHO.png">

In SafeLine WAF, I enabled HTTP Flood prevention by restricting excessive access requests. For this lab, I configured it to ban any user for 1 minute if they make more than 3 requests within 10 seconds. This setup demonstrates how the WAF can help prevent potential DoS attacks.

<img src="https://i.imgur.com/4LBzXQa.png">

On Kali, I rapidly clicked through the application and soon got a blocked page—confirming that the HTTP flood protection was working exactly as intended.

<img src="https://i.imgur.com/1sScCU7.png">

The blocked attempt was captured and recorded in the SafeLine WAF logs for analysis.

<a name="92-authentication-sign-in"></a>
### 9.2. Authentication Sign-In

<img src="https://i.imgur.com/rMu6hlY.png">

Next, I enabled the Auth Sign-In feature in the SafeLine WAF policy for DVWA. I set up a username and password for authentication. Then, when I tried to access DVWA from Kali, the WAF prompted me to enter credentials before allowing any traffic through to the server.

<img src="https://i.imgur.com/Bjfl5WM.png">

<a name="93-custom-deny-rules-blocking-kali-ip"></a>
### 9.3. Custom Deny Rules (Blocking Kali IP)

<img src="https://i.imgur.com/jHG2r6d.png">

I identified the IP address of my Kali VM. Then, in SafeLine WAF, I added a custom deny rule to block that IP.

<img src="https://i.imgur.com/xFlyIVk.png">

When I tested from Kali, I received a blocked response, confirming the rule worked as expected.

<a name="10-conclusion-and-lessons-learnt"></a>
##  10. Conclusion and Lessons Learnt

The main goal of this lab was to set up and configure the SafeLine Web Application Firewall (WAF) to protect a vulnerable web application (DVWA) within an isolated virtual environment. Through this process, I gained valuable hands-on experience deploying a firewall specifically designed to monitor and control web traffic at the application layer.

By installing SafeLine WAF between the attacker (Kali Linux) and the target (Ubuntu DVWA server), I was able to configure multiple firewall features—from basic traffic filtering to advanced protections like SQL injection detection, HTTP flood prevention, authentication gating, and custom deny rules. Each step reinforced the critical role a WAF plays in defending web applications against common and sophisticated threats.

The lab demonstrated that simply installing a firewall is not enough. Careful configuration is essential to tailor protections to the specific application and threat environment. For example, changing DVWA’s listening port, importing SSL certificates into the WAF, and setting authentication controls all required attention to detail to ensure proper firewall function without disrupting legitimate access.

Additionally, the custom deny rules showed how granular firewall policies can be used to block specific IPs, further strengthening security beyond generic rule sets.

### Lessons Learnt:

- **Firewall Deployment Requires Layered Configuration:** Installing SafeLine WAF was straightforward, but effective protection required configuring SSL, reverse proxy settings, authentication, and custom rules.<br>

- **Application-Layer Firewalling Is Crucial:** The lab highlighted how a WAF inspects and filters HTTP/HTTPS traffic in ways that traditional network firewalls cannot, offering vital protection against web-specific attacks.<br>

- **Testing in an Isolated Network Is Best Practice:** Using a host-only virtual network ensured a safe environment to simulate attacks and tune firewall responses without risking unintended impacts.<br>

- **Fine-Tuning Policies Balances Security and Access:** Setting thresholds for HTTP flood protection and enabling authentication before allowing traffic helped balance user experience with strong security controls.<br>

- **Logging and Monitoring Support Ongoing Security:** Accessing detailed firewall logs confirmed which attack attempts were blocked, underscoring the importance of visibility for maintaining security over time.<br>

In summary, this lab deepened my practical understanding of how to deploy, configure, and validate a web application firewall. It reinforced that a WAF is a powerful frontline defence tool but only as effective as its configuration and ongoing management. This experience lays a solid foundation for securing web applications with tailored firewall policies in real-world environments.
