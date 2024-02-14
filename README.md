# Self Hosting
In this practice I am going to set up a Web server.

I am going to use Vagrant to create the virtual machine.

## Table of contents
- [Self Hosting](#self-hosting)
  - [Table of contents](#table-of-contents)
  - [Description](#description)
  - [Domain Registration and DynDNS](#domain-registration-and-dyndns)
  - [Ports Forwarding](#ports-forwarding)
  - [Web server](#web-server)
    - [Installation and configuration](#installation-and-configuration)
    - [Access control and status](#access-control-and-status)
    - [Personalised errors](#personalised-errors)
    - [HTTPS](#https)
      - [Let's Encrypt SSL Certificate](#lets-encrypt-ssl-certificate)
    - [Testing](#testing)
      - [Performance testing](#performance-testing)
        - [Comparison](#comparison)
  - [Vagrant](#vagrant)

## Description
In a virtual machine called "web" I am going to install Apache and configure a web service. I am going to buy a domain and set up DynDNS so that it is always up to date.

## Domain Registration and DynDNS
For this practice I have bought the domain "xtabay00.es" from IONOS.
As I don't have a static IP, I have to make my web server automatically check if the public IP of my router is the one that points to my domain in IONOS.
To do this, in the same platform, I can create an API-key and send it in an HTTP request header, so that it authorises access to update the IP.
Now, making this request:
```
    curl -X 'POST' \
    'https://api.hosting.ionos.com/dns/v1/dyndns' \
    -H 'accept: application/json' \
    -H 'X-API-Key: private.public' \
    -H 'Content-Type: application/json' \
    -d '{
    "domains": [
        "xtabay00.es",
        "www.xtabay00.es"
    ],
    "description": "My DynamicDns"
    }'
```
*Change 'private.public' to the private and public part of your generated API key.*  
This generates this response:
```
    {
    "bulkId": "59a1ef9d-40be-4b1d-97cd-9c987cc390dd",
    "updateUrl": "https://ipv4.api.hosting.ionos.com/dns/v1/dyndns?q=NGM4YmIxNzA5MTEzNGQ4ZmE0ZmQ5MTFjZjA5YzY3NzUucWZDSi1VU0x4M29TVi1HVGxhTnpyZUtUaUlVb0t6bDNNZWwzWFFvcmhITDM0cU9WZGVoSE80T05vc0ZLTEVwOTlVR3lBX2pKNHdYMElrbFRfekFrdmc",
    "domains": [
        "www.xtabay00.es",
        "xtabay00.es"
    ],
    "description": "My DynamicDns"
    }
```
I get the update url.
All that's left is to create a task to update every so often.  
`echo "*/5 * * * * curl https://url" | crontab - `

## Ports Forwarding
In my case, with a HUAWEI EG8145V5 router, I had to enable the DMZ and map ports 80 and 443. For greater security, I put the machine in bridge mode and enabled the DMZ directly on the VM interface, without going through my host. So the port mapping from the router is to that same interface.

## Web server
### Installation and configuration
```
    # Apache Installation
    apt-get update
    apt-get install -y apache2

    # Copy the configuration files for Apache and my website
    cp -v /vagrant/config/misitio.es.conf /etc/apache2/sites-available/
    
    # Enable/disable virtual sites
    a2ensite misitio.es
    a2dissite 000-default
    a2dissite default-ssl
```
*Fragment of: provision.sh*  
To create the website, I create _misitio.es.conf_ file with the basic configuration of the website.
```
    <VirtualHost *:80>
        ServerAdmin *******@ieszaidinvergeles.org
        ServerName xtabay00.es
        DocumentRoot /var/www/misitio.es
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
```
*Fragment of: misitio.es*

The directory tree with all the resources of the website, you can copy it to /var/www or link it and modify it from the host. In my case I have created a synchronised folder.
```
    web.vm.synced_folder "misitio.es", "/var/www/misitio.es"
```
*Fragment of: Vagrantfile*

### Access control and status
I have created the admin folder where access is only possible with username and password using the digest authentication method, which is more secure.
This is done by creating the _.htpasswd_ file (using the command `htdigest -c htpasswd realm username`), I enable the `a2enmod auth_digest` module and adding in the site configuration file, the following directive:
```
    <Directory /var/www/misitio.es/admin>
        AuthType Digest
        AuthName "administracion"
        AuthUserFile /etc/apache2/.htpasswd
        Require user admin
    </Directory>
```
*Fragment of: misitio.es.conf*

Now I'm going to activate the Apache module to display information about server status, requests, memory usage, etc. And this will be accessible in _/status_, asking for authentication as well.
For this I enable the `a2enmod status` module and in the configuration file I do the same with /status, but instead of using <Directory/> I set it to <Location/> since it is not a real directory but a location inside the website.
```
    <Location /status>
        AuthType Digest
        AuthName "status"
        AuthUserFile /etc/apache2/.htpasswd
        Require user sysadmin
		SetHandler server-status
    </Location>
```
*Fragment of: misitio.es.conf*

### Personalised errors
To add a custom error page, I create such a page (error404.html) and add in the configuration file the directive `ErrorDocument 404 /error404.html`.

### HTTPS
To access my site via HTTPS is very simple:
1) Create the certificates with IONOS, Let's Encrypt or similar.
2) In the configuration file, create another VirtualHost exactly the same, but on port 443.
3) Add these lines that activate SSL and indicate the path to the certificates:
```
    SSLEngine on
	SSLCertificateFile /etc/ssl/certs/cert.pem
	SSLCertificateKeyFile /etc/ssl/private/privkey.pem
	SSLCertificateChainFile /etc/ssl/private/fullchain.pem
```
4) Activate the `a2enmod ssl` module.

#### Let's Encrypt SSL Certificate
To obtain the SSL certificate with Let's Encrypt follow these steps:
1) Install CertBot `apt install certbot python3-certbot-apache`.
2) Run `certbot --apache`.
3) Fill in the fields that you are asked for.
4) Save the keys that have been generated in _/etc/letsencrypt/live/domain/_.
5) Copy them to the path of your choice and make sure it is the path specified in the configuration file.
![letsEncrypt](https://github.com/xtabay00/Selfhosting-anto/assets/151829005/8849549b-f0b0-4e26-8227-89066910a881)


### Testing
Access the following URL's and check that it works as it should.  
Access via HTTPS: https://xtabay00.es  
Access to _admin_: https://xtabay00.es/admin  
Access to _status_: https://xtabay00.es/status  
Access to the logo: https://xtabay00.es/logo.png  
Error page: https://xtabay00.es/xyz

#### Performance testing
I will test with 100 users and 1000 requests, and with 1000 users and 1000 requests.

We will add to the request header `Accept-Encoding: gzip, deflate` with the **_-H " "_** modifier. When the server receives this request with the Accept-Encoding header, it has the option to compress the response before sending it to the client if the server supports compression and if the compression is configured correctly. Compressing the response can significantly reduce the amount of data transferred over the network, which is especially useful when performing performance tests to simulate a more realistic scenario where clients accept compressed responses.

 > ab -c 100 -n 1000 -H "Accept-Encoding: gzip, deflate" https://xtabay00.es/

    
        This is ApacheBench, Version 2.3 <$Revision: 1903618 $>
        Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
        Licensed to The Apache Software Foundation, http://www.apache.org/    
       ...
        Server Software:        Apache/2.4.56
        Server Hostname:        xtabay00.es
        Server Port:            443
        SSL/TLS Protocol:       TLSv1.3,TLS_AES_256_GCM_SHA384,2048,256
        Server Temp Key:        X25519 253 bits
        TLS Server Name:        xtabay00.es
        ...

        Concurrency Level:      100
        Time taken for tests:   5.959 seconds
        Complete requests:      1000
        Failed requests:        0
        Total transferred:      485000 bytes
        HTML transferred:       186000 bytes
        Requests per second:    167.83 [#/sec] (mean)
        Time per request:       595.852 [ms] (mean)
        Time per request:       5.959 [ms] (mean, across all concurrent requests)
        Transfer rate:          79.49 [Kbytes/sec] received
        ...

> ab -c 1000 -n 1000 -H "Accept-Encoding: gzip, deflate" https://xtabay00.es/

        This is ApacheBench, Version 2.3 <$Revision: 1903618 $>
        Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
        Licensed to The Apache Software Foundation, http://www.apache.org/
        ...
        
        Server Software:        Apache/2.4.56
        Server Hostname:        xtabay00.es
        Server Port:            443
        SSL/TLS Protocol:       TLSv1.3,TLS_AES_256_GCM_SHA384,2048,256
        Server Temp Key:        X25519 253 bits
        TLS Server Name:        xtabay00.es
        ...

        Concurrency Level:      1000
        Time taken for tests:   6.515 seconds
        Complete requests:      1000
        Failed requests:        27
        (Connect: 0, Receive: 0, Length: 27, Exceptions: 0)
        Total transferred:      478695 bytes
        HTML transferred:       183582 bytes
        Requests per second:    153.50 [#/sec] (mean)
        Time per request:       6514.633 [ms] (mean)
        Time per request:       6.515 [ms] (mean, across all concurrent requests)
        Transfer rate:          71.76 [Kbytes/sec] received

        ...
        
**Analysis:**  
Concurrency Level 100 test:
The server managed to handle approximately 168 requests per second, with an average time per request of around 596 milliseconds. The transfer rate indicates that about 79.49 kilobytes per second were received.

Concurrency Level 1000 test:
In this test with a concurrency level of 1000, the server managed to handle approximately 153.5 requests per second, with an average time per request of about 6515 milliseconds. The transfer rate indicates that about 71.76 kilobytes per second were received. In addition, 27 failed requests were observed, all related to the length of the response.

**Conclusions:**  
From 100 to 1000 the server continued to handle requests but the average time per request increased significantly, the transfer rate also decreased and 27 failed requests are observed in the test with a concurrency level of 1000, which may indicate that the server reached its maximum capacity or that there were resource issues.

Let's also use the **_-k_** option, which is used to enable the keep-alive functionality.
The persistent connection allows a single HTTP connection to be reused for multiple requests, rather than opening and closing a new connection for each individual request. This simulates behaviour more like that of a typical web browser, which uses persistent connections to load multiple resources on a web page without having to open and close a connection for each resource. This positively affects performance testing by reducing the time and resources required to establish new connections.  
> ab -c 1000 -n 1000 -H "Accept-Encoding: gzip, deflate" -k https://xtabay00.es/

        ...
        Concurrency Level:      1000
        Time taken for tests:   4.394 seconds
        Complete requests:      1000
        Failed requests:        292
        (Connect: 0, Receive: 0, Length: 292, Exceptions: 0)
        Keep-Alive requests:    708
        Total transferred:      369216 bytes
        HTML transferred:       131688 bytes
        Requests per second:    227.58 [#/sec] (mean)
        Time per request:       4394.051 [ms] (mean)
        Time per request:       4.394 [ms] (mean, across all concurrent requests)
        Transfer rate:          82.06 [Kbytes/sec] received
        ...

**Analysis:**  
Test with Concurrency Level 1000 and Keep-Alive:
In this test the server managed to handle about 227.58 requests per second, which is an increase compared to the previous test without persistent connections.
The average time per request was reduced to approximately 4394 milliseconds, indicating an improvement in performance compared to the previous test without persistent connections.
The transfer rate increased to 82.06 kilobytes per second received.
292 failed requests were observed, again indicating that the server is reaching its limits under higher load.

**Conclusion:**  
Overall, these results suggest that the server performs acceptably under moderate load, but its capacity is negatively affected when concurrency is increased to higher levels.

##### Comparison
Finally, let's make a comparison with a different server. That of the IES Zaidin Vergeles secondary school.
|                                         | xtabay00.es                  | www.ieszaidinvergeles.org    |
|-----------------------------------------|------------------------------|------------------------------|
| Requests per second (RPS) [#/sec] (mean)| 268.50                       | 268.18                       |
| Time per request [ms]             (mean)| 3724.392                     | 3728.871                     |
| Transfer rate [Kbytes/sec]              | 117.27                       | 619.79                       |
| Total transferred [bytes]               | 447256                       | 2366568                      |
| Failed requests                         | 151                          | 977                          |

They have more or less similar performance, the most noticeable difference is the data transfer rate.
Why does this happen?
If we look at the amount of data transferred we have that the IZV transfers 5,291 times more bytes than xtabay00, which explains why the transfer rate is 2,285 times higher.
So the performance of both is very similar.

## Vagrant
I create a virtual machine called 'web' with Vagrant.
```ruby
    Vagrant.configure("2") do |config|

    config.vm.box = "debian/bullseye64"
    config.vm.provider "virtualbox"


    config.vm.define "web" do |web|
        web.vm.hostname="web"
        web.vm.network "public_network", ip: "192.168.100.50"

        # Linking our web resources to the Document Root
        web.vm.synced_folder "misitio.es", "/var/www/misitio.es"

        # VM provision
        web.vm.provision "shell", path: "provision.sh"
    end

    end
```
*Fragment of: Vagrantfile*  
Apart from all the files, for automation with Vagrant, I have created a provision files. In it is everything explained above, the installation of the services, the copy of the relevant configuration files and the enabling of the services.

If you want to deploy this practice, download the whole folder, go to it from cmd and run `vagrant up`.
