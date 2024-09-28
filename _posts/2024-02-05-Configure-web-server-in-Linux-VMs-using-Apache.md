---
layout: post
title: "Configure Linux VM as a web server using Apache"
author: Sriram H. Iyer
---

## Overview

[Apache](https://httpd.apache.org/) is an open-source web server for modern operating systems including UNIX, Linux, and Windows.

In this blog, we will configure an Azure VM with Ubuntu OS as a web server using Apache.

## Network Architecture

We will deploy an Azure Virtual Machine with Ubuntu OS for configuring it as a web server using Apache.

![Network Diagram](https://raw.githubusercontent.com/hisriram96/blog/2e6581f4269388ff1f98ee8d413dbebc4b4ae6e4/_pictures/azure-linux-virtual-machine-network-diagram.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisriram96%2Fblog%2Fmain%2F_arm-templates%2Fazure-virtual-machine-ubuntu-deployment.json)

## Configure HTTP server using Apache

1. Install Apache package.

   ```bash
   sudo apt-get update
   sudo apt-get install apache2
   ```

2. Create directory for custom website.

   ```bash
   sudo mkdir /var/www/www.example.com
   ```

3. Create a landing webpage.

   ```bash
   sudo vi /var/www/www.example.com/index.html
   ```

   Example of the landing page.

   ```bash
   <h1>Hello World!<h1>
   ```

   Example:

   <img width="437" alt="index" src="https://github.com/hisriram96/blog/assets/56336513/7fc834db-f110-48a4-8da7-23c5b51e7c2a">

4. Create a virtual host file for your website.

   ```bash
   sudo vi /etc/apache2/sites-available/www.example.com.conf
   ```

   Configuration of virtual host file:

   ```bash
   <VirtualHost *:80>
         DocumentRoot /var/www/www.example.com
         ServerName www.example.com
      
	     ErrorLog ${APACHE_LOG_DIR}/error.log
	     CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   ```

   Example:

   ![http-conf](https://github.com/hisriram96/blog/assets/56336513/7ff01598-be0c-47cd-8c45-aa5f26a3c796)

5. Enable the vitual host file.

   ```bash
   sudo a2ensite www.example.com
   ```

6. Run an Apache configuration file syntax test. It parses the configuration files and either reports Syntax Ok or detailed information about the particular syntax error.

   ```bash
   sudo apache2ctl configtest
   ```

7. Restart Apache service.

   ```bash
   sudo systemctl restart apache2
   ```

   Example:

   ![status](https://github.com/hisriram96/blog/assets/56336513/ffda72f5-674f-4c4f-872d-efc362641c74)

8. Verify accessing the web site.

   ```bash
   curl -v http://www.example.com --resolve www.example.com:80:<Public IP of the VM>
   ```

   Example:

   <img width="642" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/3e90d2a0-5ec6-4723-b204-66673d614acf">

## Securing web service using TLS/SSL certificates

In the previous sections, we configured Apache as a web service using HTTP protocol. In order to encrypt the traffic, we need to create TLS/SSL certificate and configure Apache to use the certificate for enabling HTTPS.

We could either create a self-signed certificate or issue a trusted CA (Certificate Authority) certificate. 

Please note that self-signed certificates are not recommended for production or in customer/public facing websites because the custom domain for which the certificate is issued is not verified. 

When a truested CA certificate is issued, the custom domain in verified by a third-party CA and is more secure.

### Generate a self-signed certificate and update Apache configuration

In this section, we will create a self-signed SSL certificate and `.pfx` file using the [OpenSSL](https://www.openssl.org/) commands and configure them in Apache.

> Self-signed certificates are digital certificates that are not signed by a trusted third-party CA. Self-signed certificates are created, issued, and signed by the company or developer who is responsible for the website or software being signed. This is why self-signed certificates are considered unsafe for public-facing websites and applications.

Since we will create self-signed SSL certificate with custom root CA using OpenSSL commands, we need to install OpenSSL package.

```bash
sudo apt-get update
sudo apt-get install openssl
```

We will create a self-signed certificate chain with own custom root CA.

1. Create a key for root certificate

   ```bash
   openssl ecparam -out root.key -name prime256v1 -genkey
   ```

2. Create a CSR (Certificate Signing Request) for root certificate and self-sign it

   Generate the Certificate Signing Request (CSR). The CSR is a public key that is given to a CA when requesting a certificate. The CA issues the certificate for this specific request.

   We will be prompted for the password for the root key, and the organizational information for the custom CA such as Country/Region, State, Org, OU, and the fully qualified domain name (this is the domain of the issuer).

   ```bash
   openssl req -new -sha256 -key root.key -out root.csr
   ```

3. Create the root certificate using the root CSR. We will use this to sign your server certificate.

   ```bash
   openssl x509 -req -sha256 -days 365 -in root.csr -signkey root.key -out root.crt
   ```

4. Generate the key for the server certificate.

   ```bash
   openssl ecparam -out server.key -name prime256v1 -genkey
   ```

5. Create the CSR for server certificate.

   > Please note that the CN (Common Name) for the server certificate must be different from the issuer's domain. For example, in this case, the CN for the issuer is `example.com` and the server certificate's CN is `www.example.com`.

   ```bash
   openssl req -new -sha256 -key server.key -out server.csr
   ```

6. Create the server certificate signing it using root key

   ```bash
   openssl x509 -req -in server.csr -CA root.crt -CAkey root.key -CAcreateserial -out server.crt -days 365 -sha256
   ```

7. Create a full chain certificate bundling root and server certificates.

   ```bash
   cat server.crt > bundle.crt
   cat root.crt >> bundle.crt
   ```

   Example:

   <img width="721" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/39c1f863-78bd-4926-96d2-6402b007d7dd">

8. Install root certificate in the [CA trust store](https://ubuntu.com/server/docs/security-trust-store) of Ubuntu. 

   ```bash
   sudo apt-get install -y ca-certificates
   sudo cp root.crt /usr/local/share/ca-certificates
   sudo update-ca-certificates
   ```

   Example:

   <img width="543" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/e4ee3891-9504-4f38-b936-9598928ba846">

### Issue a trusted CA certificate using Let's Encrypt

In this section, we will issue a trusted CA certificate using [Let's Encrypt](https://letsencrypt.org/about/). Let’s Encrypt is an open certificate authority (CA) which provides trusted CA certificate for free.

> Please note that Let's Encrypt or any CA would perform domain validation. Since `example.com` is not a valid domain, this section requires that you have a valid domain. In this example, `tcpip.digital` domain is used and the Apache is configured with a virtual host file for `tcpip.digital`.

1. Snap package manager is needed to install the Certbot tool so first install the snapd.

   ```bash
   sudo apt-get install snapd
   sudo snap install snap-store
   ```

2. Install Certbot in Ubuntu VM.

   ```bash
   sudo snap install --classic certbot
   ```

3. Configure the Certbot with Apache and request SSL certificate from Let’s Encrypt.

   ```bash
   sudo certbot --apache -d example.com
   ```

   Example:

   ![encrypt](https://github.com/hisriram96/blog/assets/56336513/5f3042d8-e981-48a4-9e0a-66918a03234c)

Generated certificate files coud be found at `/etc/letsencrypt/live`.

```bash
sudo ls -l /etc/letsencrypt/live/example.com
```

Let’s Encrypt will generate following files:

```bash
README		   	--> Contains information about generated certificate files by Let’s Encrypt
cert.pem	      	--> Contains leaf certificate generated by Let’s Encrypt for your domain
chain.pem	   	--> Contains intermediate and root certificates of Let’s Encrypt
fullchain.pem		--> Contains entire chain of leaf, intermediate and root certificates. This must be configured in your Apache virtual host file
privkey.pem 		--> Contains private key generated by Let’s Encrypt for your domain
```

Example:

<img width="607" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/c4661c52-e7f7-4cb9-b614-5c24858e2011">

You must use the "fullchain.pem" file as certificate for your Apache server.

Please note that Certbot automatically configures the virtual host file of Apache so that the web service uses HTTPS protocol and redirects HTTP requests to HTTPS.

Example:

![encrypt-config](https://github.com/hisriram96/blog/assets/56336513/e84b873f-8979-4545-b7e2-119e95b456f5)

### Configure virtual host file of Apache for HTTPS

In case you want to use self-signed certificate or the virtual host file is not managed by Certbot, you would need to update the virtual host file `/etc/apache2/sites-available/www.example.com.conf` to use SSL certificate.

Configuration of virtual host file:

```bash
<VirtualHost *:443>
      DocumentRoot /var/www/www.example.com
      ServerName www.example.com
      SSLEngine on
      SSLCertificateFile /home/user/bundle.crt
      SSLCertificateKeyFile /home/user/server.key
	  
      ErrorLog ${APACHE_LOG_DIR}/error.log
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

The Apache server will only respond to HTTPS requests on port 443 with the above configuration in virtual host file. It is good practice to also respond on port 80, even if you want to force all traffic to be encrypted by using permanent redirection from 80 (HTTP) to 443 (HTTPS).

Configuration of virtual host file:

```bash
<VirtualHost *:80>
      ServerName www.example.com
      Redirect permanent / https://www.example.com/
</VirtualHost>

<VirtualHost *:443>
      DocumentRoot /var/www/www.example.com
      ServerName www.example.com
      SSLEngine on
      SSLCertificateFile /home/user/bundle.crt
      SSLCertificateKeyFile /home/user/server.key

      ErrorLog ${APACHE_LOG_DIR}/error.log
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Enable SSL module within the Apache configuration.

```bash
sudo a2enmod ssl
```

After the SSL module is enabled, enable the website's virtual host file.

```bash
sudo a2ensite www.example.com
```

Run an Apache configuration file syntax test. It parses the configuration files and either reports Syntax Ok or detailed information about the particular syntax error.

```bash
sudo apache2ctl configtest
```

Restart Apache service.

```bash
sudo systemctl restart apache2
```

Example:

<img width="646" alt="ssl-mini" src="https://github.com/hisriram96/blog/assets/56336513/360d9bf2-e9f1-437a-98c0-0047717670ff">

## Verify accessing the Apache

In case of using self-signed certificate, you would see "Not Secure" as the self-signed certificate is not secure unless the root certificate is stored in the trusted CA store of the client machine. When using `curl` command to access the web site over HTTPS, use `-k` option to proceed with TLS handshake even if not secure.

<img width="651" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/a2dba2de-e164-4db0-8c18-1ae0080d1848">

In case the trusted CA certificate is used, you could access the custom domain without using `-k` option.

<img width="415" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/872c96f3-539f-4510-8b99-0a2635beedd2">

<link rel="alternate" type="application/rss+xml"  href="{{ site.url }}/feed.xml" title="{{ site.title }}">
