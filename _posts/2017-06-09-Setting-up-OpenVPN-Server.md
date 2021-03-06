---
title: "Part 1-Setting up OpenVPN Server"
layout: post
author: "saravanan"
og_image_url: "https://devcenter.megam.io/res/gotalk-intro.png"
description: "How to setup OpenVPN server in ubuntu"
---


### Introduction

A Virtual Private Network (VPN) allows you to traverse untrusted networks privately and securely as if you were on a private network. The traffic emerges from the VPN server and continues its journey to the destination.

When combined with HTTPS connections, this setup allows you to secure your wireless logins and transactions.

OpenVPN is a full-featured open source Secure Socket Layer (SSL) VPN solution that accommodates a wide range of configurations. This tutorial will keep the installation and configuration steps as simple as possible.

[![img](https://s3-ap-southeast-1.amazonaws.com/megampub/images/vertice/DEPLOY-TO-MEGAM-VERTICE-BIG.png)](https://docs.megam.io/installation/prequisites/)

### Prerequisites

To follow this tutorial :

* You will need access to an Ubuntu 16.04 server.

### Step 1 - Install OpenVPN

* To start off we need to install OpenVPN on our server

      $ sudo apt-get update

      $ sudo apt-get install openvpn easy-rsa

### Step 2: Set Up the CA Directory

* OpenVPN is an TLS/SSL VPN. So it utilizes certificates in order to encrypt traffic between the server and clients. In order to issue trusted certificates, we will need to set up our own simple certificate authority (CA).

      $ make-cadir ~/openvpn-ca

* Move into the newly created directory to begin configuring the CA.

      $ cd ~/openvpn-ca

### Step 3: Configure the CA Variables

* To configure the values our CA will use, we need to edit the vars file within the directory. Open that file now in your text editor.

      $ nano vars

* You will find some variables that can be adjusted to determine how your certificates will be created. Towards the bottom of the file, find the settings that set field defaults for new certificates.

      . . .

      export KEY_COUNTRY="US"
      export KEY_PROVINCE="CA"
      export KEY_CITY="SanFrancisco"
      export KEY_ORG="Fort-Funston"
      export KEY_EMAIL="me@myhost.mydomain"
      export KEY_OU="MyOrganizationalUnit"

      . . .

* Edit the variables do not left them blank.

* we should also edit the KEY_NAME value just below this section, which populates the subject field. To keep this simple, we'll call it server in this guide:

      export KEY_NAME="server"

 * When you are finished, save and close the file.

### Step 4: Build the Certificate Authority

* We can use the variables we set and the easy-rsa utilities to build our certificate authority. Ensure you are in your CA directory, and then source the vars file you just edited.

      $ cd ~/openvpn-ca
      $ source vars

* You should see the following if it was sourced correctly.

      Output
      NOTE: If you run ./clean-all, I will be doing a rm -rf on /openvpn-ca/keys

* Make sure we're operating in a clean environment by typing.

      $ ./clean-all

 * Now, we can build our root CA by typing.

        $ ./build-ca

* It will initiate the process of creating the root certificate authority key and certificate. Since we filled out the vars file, all of the values should be populated automatically. Just press ENTER through the prompts to confirm the selections.

      Output
      Generating a 2048 bit RSA private key
      ..........................................................................................+++
      ...............................+++
      writing new private key to 'ca.key'
      -----
      You are about to be asked to enter information that will be incorporated
      into your certificate request.
      What you are about to enter is what is called a Distinguished Name or a DN.
      There are quite a few fields but you can leave some blank
      For some fields there will be a default value,
      If you enter '.', the field will be left blank.
      -----
      Country Name (2 letter code) [IN]:
      State or Province Name (full name) [TN]:
      Locality Name (eg, city) [Chennai]:
      Organization Name (eg, company) [Megam systems]:
      Organizational Unit Name (eg, section) [Community]:
      Common Name (eg, your name or your server's hostname) [Megam]:
      Name [server]:
      Email Address [admin@email.com]:

* We now have a CA that can be used to create the rest of the files we need.

### Step 5: Create the Server Certificate, Key, and Encryption Files

* Next, we will generate our server certificate and key pair, as well as some additional files used during the encryption process.

* Note: If you choose a name other than server here, you will have to adjust some of the instructions below. For instance, when copying the generated files to the /etc/openvpn directroy, you will have to substitute the correct names. You will also have to modify the /etc/openvpn/server.conf file later to point to the correct .crt and .key files.

* To start generating the OpenVPN server certificate and key pair. We can do this by typing.

      $ ./build-key-server server

* Once again, it prompts for default values based on the argument we just passed in (server) and the contents of our vars file we sourced. Accept the default values by pressing ENTER. Do not enter a challenge password for this setup. At the end, you will have to enter y to two questions to sign and commit the certificate:

      Output
      . . .

      Certificate is to be certified until May  1 17:51:16 2026 GMT (3650 days)
      Sign the certificate? [y/n]:y


      1 out of 1 certificate requests certified, commit? [y/n]y
      Write out database with 1 new entries
      Data Base Updated

* Next, we'll generate a few other items. We can generate a strong Diffie-Hellman keys to use during key exchange by typing.

      $ ./build-dh

* we can generate an HMAC signature to strengthen the server's TLS integrity verification capabilities.

      $ openvpn --genkey --secret keys/ta.key

### Step 6: Generate a Client Certificate and Key Pair

* Next, we can generate a client certificate and key pair. Although this can be done on the client machine and then signed by the server/CA for security purposes, for this guide we will generate the signed key on the server for the sake of simplicity.

* We will generate a single client key/certificate for this guide, but if you have more than one client, you can repeat this process as many times as you'd like. Pass in a unique value to the script for each client.

* To produce credentials without a password, to aid in automated connections, use the build-key command.

      $ cd ~/openvpn-ca
      $ source vars
      $ ./build-key client1

* If you wish to create a password-protected set of credentials, use the build-key-pass command.

      $ cd ~/openvpn-ca
      $ source vars
      $ ./build-key-pass client1

* Again, the defaults should be populated, so you can just hit ENTER to continue. Leave the challenge password blank and make sure to enter y for the prompts that ask whether to sign and commit the certificate.
### Step 7: Configure the OpenVPN Service

* we can begin configuring the OpenVPN service using the credentials and files we've generated.

### Copy the Files to the OpenVPN Directory

* We need to copy the files we need to the /etc/openvpn configuration directory.

* We can start with all of the files that we just generated. These were placed within the ~/openvpn-ca/keys directory as they were created. We need to move our CA cert and key, our server cert and key, the HMAC signature, and the Diffie-Hellman file.

      $ cd ~/openvpn-ca/keys
      $ sudo cp ca.crt ca.key server.crt server.key ta.key dh2048.pem /etc/openvpn

* Next, we need to copy and unzip a sample OpenVPN configuration file into configuration directory so that we can use it as a basis for our setup.

      $ gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf

### Adjust the OpenVPN Configuration

Now our files are in place, we can modify the server configuration file.

      $ sudo nano /etc/openvpn/server.conf

* Find the HMAC section by looking for the tls-auth directive. Remove the ";" to uncomment the tls-auth line. Below this, add the key-direction parameter set to "0":

      tls-auth ta.key 0 # This file is secret
      key-direction 0

* Find the section on cryptographic ciphers by looking for the commented out cipher lines. The AES-128-CBC cipher offers a good level of encryption and is well supported. Remove the ";" to uncomment the cipher AES-128-CBC line.

      cipher AES-128-CBC

* Below this, add an auth line to select the HMAC message digest algorithm. For this, SHA256 is a better choice.

      auth SHA256

* Find the user and group settings and remove the ";" at the beginning of to uncomment those lines.

      user nobody
      group nogroup

* When you are finished, save and close the file.

### Step 8: Adjust the Server Networking Configuration

* We need to adjust some aspects of the server's networking so that OpenVPN can correctly route traffic.

* Allow IP Forwarding

* First, we need to allow the server to forward traffic. This is fairly essential to the functionality we want our VPN server to provide.

* We can adjust this setting by modifying the /etc/sysctl.conf file.

      $ sudo nano /etc/sysctl.conf

 * Inside, look for the line that sets net.ipv4.ip_forward. Remove the "#" character from the beginning of the line to uncomment that setting.

    net.ipv4.ip_forward=1

* Save and close the file when you are finished.

* To read the file and adjust the values for the current session, type.

      $ sudo sysctl -p

### Step 9: Start and Enable the OpenVPN Service

* We're finally ready to start the OpenVPN service on our server.

* We need to start the OpenVPN server by specifying our configuration file name as an instance variable after the systemd unit file name. Our configuration file for our server is called /etc/openvpn/server.conf, so we will add @server to end of our unit file when calling it.

      $ sudo systemctl start openvpn@server

* Double-check that the service has started successfully by typing

      $ sudo systemctl status openvpn@server

* If everything went well, your output should look something that looks like this.

      Output
      ● openvpn@server.service - OpenVPN connection to server
      Loaded: loaded (/lib/systemd/system/openvpn@.service; disabled; vendor preset: enabled)
      Active: active (running) since Tue 2016-05-03 15:30:05 EDT; 47s ago
      Docs: man:openvpn(8)
      https://community.openvpn.net/openvpn/wiki/Openvpn23ManPage
      https://community.openvpn.net/openvpn/wiki/HOWTO
      Process: 5852 ExecStart=/usr/sbin/openvpn --daemon ovpn-%i --status /run/openvpn/%i.status 10 --cd /etc/openvpn --script-security 2 --config /etc/openvpn/%i.conf --writepid /run/openvpn/%i.pid (code=exited, sta
        Main PID: 5856 (openvpn)
        Tasks: 1 (limit: 512)
        CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
        └─5856 /usr/sbin/openvpn --daemon ovpn-server --status /run/openvpn/server.status 10 --cd /etc/openvpn --script-security 2 --config /etc/openvpn/server.conf --writepid /run/openvpn/server.pid

        May 03 15:30:05 openvpn2 ovpn-server[5856]: /sbin/ip addr add dev tun0 local 10.8.0.1 peer 10.8.0.2
        May 03 15:30:05 openvpn2 ovpn-server[5856]: /sbin/ip route add 10.8.0.0/24 via 10.8.0.2
        May 03 15:30:05 openvpn2 ovpn-server[5856]: GID set to nogroup
        May 03 15:30:05 openvpn2 ovpn-server[5856]: UID set to nobody
        May 03 15:30:05 openvpn2 ovpn-server[5856]: UDPv4 link local (bound): [undef]
        May 03 15:30:05 openvpn2 ovpn-server[5856]: UDPv4 link remote: [undef]
        May 03 15:30:05 openvpn2 ovpn-server[5856]: MULTI: multi_init called, r=256 v=256
        May 03 15:30:05 openvpn2 ovpn-server[5856]: IFCONFIG POOL: base=10.8.0.4 size=62, ipv6=0
        May 03 15:30:05 openvpn2 ovpn-server[5856]: IFCONFIG POOL LIST
        May 03 15:30:05 openvpn2 ovpn-server[5856]: Initialization Sequence Completed

* You can also check that the OpenVPN tun0 interface is available by typing.

      $ ip addr show tun0

* You should see a configured interface.

      Output
      4: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 100
      link/none
      inet 10.8.0.1 peer 10.8.0.2/32 scope global tun0
      valid_lft forever preferred_lft forever

* If everything went well, enable the service so  it can starts automatically at boot.

      $ sudo systemctl enable openvpn@server

### Step 10: Create Client Configuration Infrastructure

* Next, we need to set up a system that will allow us to create client configuration files easily.
Creating the Client Config Directory Structure

* Create a directory structure within your home directory to store the files.

      $ mkdir -p ~/client-configs/files

 * Since our client configuration files will have the client keys embedded, we should lock down permissions on our inner directory.

      $ chmod 700 ~/client-configs/files

* Creating a Base Configuration

* Next copy an example client configuration into our directory to use as our base configuration.

      $ cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf

 * Open this new file in your text editor.

      $ nano ~/client-configs/base.conf

  * Inside, we need to make a few adjustments.

  * First, locate the remote directive. This points the client to our OpenVPN server address. This should be the public IP address of your OpenVPN server. If you changed the port that the OpenVPN server is listening on, change 1194 to the port you selected.

      . . .
      # The hostname/IP and port of the server.
      # You can have multiple remote entries
      # to load balance between the servers.
      remote server_IP_address 1194
      . . .


* Be sure that the protocol matches the value you are using in the server configuration.

      proto udp

* Next, uncomment the user and group directives by removing the ";".

      # Downgrade privileges after initialization (non-Windows only)
      user nobody
      group nogroup

* Find the directives that set the ca, cert, and key. Comment out these directives since we will be adding the certs and keys within the file itself.

      # SSL/TLS parms.
      # See the server config file for more
      # description.  It's best to use
      # a separate .crt/.key file pair
      # for each client.  A single ca
      # file can be used for all clients.
      #ca ca.crt
      #cert client.crt
      #key client.key

* Mirror the cipher and auth settings that we set in the /etc/openvpn/server.conf file.

      cipher AES-128-CBC
      auth SHA256

* Next, add the key-direction directive somewhere in the file. This must be set to "1" to work with the server.

      key-direction 1

 * Finally, add a few commented out lines. We want to include these with every config, but should only enable them for Linux clients that ship with a /etc/openvpn/update-resolv-conf file. This script uses the resolvconf utility to update DNS information for Linux clients.

      # script-security 2
      # up /etc/openvpn/update-resolv-conf
      # down /etc/openvpn/update-resolv-conf

 * If your client is running Linux and has an /etc/openvpn/update-resolv-conf file, you should uncomment these lines from the generated OpenVPN client configuration file.

 * Save the file when you are finished.

* Creating a Configuration Generation Script

* Next, we will create a simple script to compile our base configuration with the relevant certificate, key, and encryption files. This will place the generated configuration in the ~/client-configs/files directory.

* Create and open a file called make_config.sh within the ~/client-configs directory.

      $ nano ~/client-configs/make_config.sh

 * Inside, paste the following script.

    #!/bin/bash

    # First argument: Client identifier

    KEY_DIR=~/openvpn-ca/keys
    OUTPUT_DIR=~/client-configs/files
    BASE_CONFIG=~/client-configs/base.conf

    cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn

* Save and close the file when you are finished.

* Mark the file as executable by typing.

      $ chmod 700 ~/client-configs/make_config.sh

### Step 11: Generate Client Configurations

* Now, we can easily generate client configuration files.

* If you followed along with the guide, you created a client certificate and key called client1.crt and client1.key respectively by running the ./build-key client1 command in step 6. We can generate a config for these credentials by moving into our ~/client-configs directory and using the script we made.

      $ cd ~/client-configs
      $ ./make_config.sh client1

* If everything went well, we should have a client1.ovpn file in our ~/client-configs/files directory.

      $ ls ~/client-configs/files

      Output
      client1.ovpn

* Transferring Configuration to Client Devices

* We need to transfer the client configuration file to the relevant device. For instance, this could be your local computer or a mobile device.

* While the exact applications used to accomplish this transfer will depend on your choice and device's operating system, you want the application to use SFTP (SSH file transfer protocol) or SCP (Secure Copy) on the backend. This will transport your client's VPN authentication files over an encrypted connection.

* Here is an example SFTP command using our client1.ovpn example. This command can be run from your local computer (OS X or Linux). It places the .ovpn file in your home directory.

 local $ sftp user@openvpn_server_ip:client-configs/files/client1.ovpn ~/

### Verify server and client configuration files

* verify server configuration file `server.conf`

      port 1194
      proto udp
      dev tun
      ca ca.crt
      cert server.crt
      key server.key  # This file should be kept secret
      dh dh2048.pem
      server 10.8.0.0 255.255.255.0
      ifconfig-pool-persist ipp.txt
      keepalive 10 120
      tls-auth ta.key 0 # This file is secret
      key-direction 0
      cipher AES-128-CBC   # AES
      auth SHA256
      comp-lzo
      user nobody
      group nogroup
      persist-key
      persist-tun
      status openvpn-status.log
      log-append  /var/log/openvpn.log
      verb 3

* verify client configuration file `client.ovpn`

        client
        dev tun
        proto udp
        remote 185.88.29.231 1194
        resolv-retry infinite
        nobind
        user nobody
        group nogroup
        persist-key
        persist-tun
        remote-cert-tls server
        cipher AES-128-CBC
        auth SHA256
        key-direction 1
        comp-lzo
        verb 3
        <ca>
        -----Displays show ca.crt contents here. -----
        </ca>
        <cert>
        -----Displays client.crt contents here. -----
        </cert>
        <key>
        -----Displays client.key contents here. -----
        </key>
        <tls-auth>
        -----Displays ta.key contents here. -----
        </tls-auth>

### Conclusion

These are the steps to configure the OpenVPN in a server. Now you access the Internet safely and securely from your computer or laptop when connected to an untrusted network.
