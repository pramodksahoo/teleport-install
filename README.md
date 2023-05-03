Use the default branch jenkins_pipeline if you dont want to install prometheus and grafana. use prom_graphana branch if you wanna install prometheus and grafana.

# TELEPORT Install & Setup :-
Teleport is available in two editions: community and enterprise edition. We will be using the community edition in this example setup.

Teleport can help you to securely access the Linux servers via SSH : </br>
1. Server Access: Single Sign-On, short-lived certificates, and audit for SSH servers.

### Install Teleport on Linux:-
In this example tutorial, we are using an Amazon linux image for EC2 instance. Hence, to install Teleport on AWS Linux server;

### Step-1:-
1. - Spin-up EC2 instance on AWS console : Note the Private IP of the instance

2. Open an SSH port in addition port 443 in SG group for that EC2 instance

3. Login to EC2 Instance :

Teleport uses TLS to provide secure access to its Proxy Service and Auth Service, and this requires a domain name that clients can use to verify Teleport's certificate

### Step-2:- 
1. Set DNS resolvable hostnames for Teleport Server

Check Host Name

# hostname

Set Hostname for that teleport server

# hostnamectl set-hostname <dns name of that server(Ex:-teleport.eapi.nonprod.nb01.local)>

Set private IP in Hosts file with maping DNS name

# echo "10.42.55.21 teleport.eapi.nonprod.nb01.local teleport" >> /etc/hosts

Check the entry for verify the hosts file

# cat /etc/hosts
  If there is not available public SSL certificate, then need to Generate self-signed SSL/TLS certificates for Teleport Server

### NOTE: 
  **The certificate must have a subject that corresponds to the domain of your Teleport host, e.g., teleport.db2-serv.com. Replace the domain names accordingly.**

## Self-sign certificate:-

openssl req -x509 -nodes -newkey rsa:4096 \

-keyout /var/lib/teleport/teleport.key \

-out /var/lib/teleport/teleport.pem -sha256 -days 3650 \

-subj "/C=US/ST=N.virginia/L=N.virginia/O=FIS/OU=Org/CN=teleport.db2-serv.com"


### Step-3:-
Install and setup Teleport Configuration file

# yum update

Add repo for teleport

# yum-config-manager --add-repo https://rpm.releases.teleport.dev/teleport.repo

Install Teleport

# yum install teleport

Generate Teleport Configuration file

Once you have setup the domain name and generates the SSL certs, run the command below to generate Teleport configuration file.

# teleport configure -o /etc/teleport.yaml  \

     --cluster-name=teleport.eapi.nonprod.nb01.local \

     --public-addr=teleport.eapi.nonprod.nb01.local:443 \

     --cert-file=/var/lib/teleport/teleport.eapi.nonprod.nb01.local-chain.pem \

     --key-file=/var/lib/teleport/teleport.eapi.nonprod.nb01.local.key

Check the configuration file for teleport server & verify the key name and dns name.

# cat /etc/teleport.yaml

### Example of YAML file:

-----------------

version: v2

teleport:

  nodename: teleport.eapi.nonprod.nb01.local

  data_dir: /var/lib/teleport

  log:

    output: stderr

    severity: INFO

    format:

      output: text

  ca_pin: ""

  diag_addr: ""

auth_service:

  enabled: "yes"

  listen_addr: 0.0.0.0:3025

  cluster_name: teleport.eapi.nonprod.nb01.local

  proxy_listener_mode: multiplex

ssh_service:

  enabled: "yes"

  commands:

  - name: hostname

    command: [hostname]

    period: 1m0s

proxy_service:

  enabled: "yes"

  web_listen_addr: 0.0.0.0:443

  public_addr: teleport.eapi.nonprod.nb01.local:443

  https_keypairs:

  - key_file: /var/lib/teleport/teleport.eapi.nonprod.nb01.local.key

    cert_file: /var/lib/teleport/teleport.eapi.nonprod.nb01.local.pem

  acme: {}

---------------------------------------
you can test its validity using the --test option

# teleport configure --test /etc/teleport.yaml

Start Teleport Service

# systemctl enable --now teleport

Check the status of Teleport service

# systemctl status teleport

### Step-4:-

Create Teleport Admin User

# tctl users add teleport-admin --roles=editor,access --logins=root,ec2-user

Sample command output;

User "teleport-admin" has been created but requires a password. Share this URL with the user to complete user setup, link is valid for 1h:

https://teleport.eapi.nonprod.nb01.local:443/web/invite/1c2fd60cad32df99a65b75081f78bbda



NOTE: Make sure teleport.eapi.nonprod.nb01.local.com:443 points at a Teleport proxy which users can access.



Finalize Teleport Setup on Browser

You can now access the link provided, which is valid for one hour

Open in Browser : https://teleport.eapi.nonprod.nb01.local:443/web/invite/1c2fd60cad32df99a65b75081f78bbda

Click Get Started to create an account


### Step-5:-
You can now proceed to add servers for secure access to the Teleport access plane.

### Node join to Teleport SERVER:-

Copy certificate file from one ec2 instance to another ec2 instance

# scp -i xyz.pem ec2-user@10.42.55.51:/home/ec2-user/teleport.eapi.nonprod.nb01.local.zip .
# cp teleport.eapi.nonprod.nb01.local.zip /etc/pki/ca-trust/extracted/pem/
copy certificate file
# cp teleport.eapi.nonprod.nb01.local.zip /usr/share/pki/ca-trust-source/anchors
copy certificate file
# cd /etc/pki/ca-trust/extracted/pem/
copy .cert file content to tls-ca-bundle.pem

# update-ca-trust

single command automaticaly:

sudo bash -c "$(curl -kfsSL https://teleport.eapi.nonprod.nb01.local/scripts/0261d6444c72c9cd2fa468045c733ccc/install-node.sh)"

 ## Manualy:
 hostnamectl set-hostname db2-server-3
# hostname
# echo "10.42.55.21 teleport.eapi.nonprod.nb01.local teleport" >> /etc/hosts
# yum-config-manager --add-repo https://rpm.releases.teleport.dev/teleport.repo
# yum install teleport
Create teleport.yaml file in node system
# vi /etc/teleport.yaml

### replace the token_name & ca_pin.
---------------------------
version: v2
teleport:
nodename: db2-server-3
data_dir: /var/lib/teleport
join_params:
token_name: cbd3b85b1d4fd225b4c5b3cf6abb94fb
method: token
auth_servers:
- teleport.db2-serv.com:443
log:
output: stderr
severity: INFO
format:
output: text
ca_pin: sha256:7dd8b268553188b82b09cd00575e9e0e14f0b013ed1e62e0dd31e4a45aa4dd56
diag_addr: ""
auth_service:
enabled: "no"
ssh_service:
enabled: "yes"
labels:
environment: db2servers
commands:
- name: hostname
command: [hostname]
period: 1m0s
proxy_service:
enabled: "no"
https_keypairs: []
acme: {}
----------------------

you can test its validity using the --test option

# teleport configure --test /etc/teleport.yaml

Start Teleport Service

# systemctl enable --now teleport

Check the status of Teleport service

# systemctl status teleport

Now you can verify the added server in Teleport Server Dashboard.


Click on connect and select user to login the host with ssh.
