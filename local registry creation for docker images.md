**Self-hosted Docker registry with self-signed certificates**

When you need a Docker registry hosted on your LAN and you don't want to
go through the trouble of purchasing certificates from a Certificate
Authority, what do you do? You deploy a registry using self-signed
certificates.

**What you'll need**

To make this work, you'll need at least two machines, both of which have
Docker installed. I'm going to demonstrate on Ubuntu Server 20.04 and
Pop!\_OS desktop. If you're using a different operating system, you'll
need to alter the process accordingly.

**How to create your directories**

The first thing we're going to do is create some directories to house
the repository and the necessary certificates. I'm going to demonstrate
this on my users' home directory, but you can place them in any
directory to which your user has access.

Create the base directory with:

**mkdir \~/registry**

Create the two subdirectories with:

**mkdir \~/registry/certs**

**mkdir \~/registry/auth**

Change in the certs directory with:

**cd \~/registry/certs**

Generate a private key with:

**openssl genrsa 1024 \> domain.key**

Change the permissions for the new key with:

**chmod 400 domain.key**

Next, we need to generate our certificate. However, because of the way
the authorization process now works, we must first create a san.cnf file
with:

**nano san.cnf**

In that file, paste the following contents (making sure to edit
accordingly):

**\[req\]**

**default_bits  = 2048**

**distinguished_name = req_distinguished_name**

**req_extensions = req_ext**

**x509_extensions = v3_req**

**prompt = no**

**\[req_distinguished_name\]**

**countryName = XX**

**stateOrProvinceName = N/A**

**localityName = N/A**

**organizationName = Self-signed certificate**

**commonName = 120.0.0.1: Self-signed certificate**

**\[req_ext\]**

**subjectAltName = \@alt_names**

**\[v3_req\]**

**subjectAltName = \@alt_names**

**\[alt_names\]**

**IP.1 = 192.168.1.191**

Make sure to change (at least) IP.1 = to match the IP address of your
hosting server.

Save and close the file.

Generate the key with:

**openssl req -new -x509 -nodes -sha1 -days 365 -key domain.key -out
domain.crt -config san.cnf**

Change into the auth directory with:

**cd ../auth**

We now must pull down the registry container and have it generate an
htpasswd file. This is done with the command:

**docker run \--rm \--entrypoint htpasswd registry:2.7.0 -Bbn USERNAME
PASSWORD \> htpasswd**

Where USERNAME is a unique username and PASSWORD is a unique/strong
password.

**How to deploy the registry server**

It's now time to deploy the registry server. Change back to the base
registry directory with:

**cd \~/registry**

Deploy the registry container with the command:

**docker run -d \\**

**\--restart=always \\**

**\--name registry \\**

**-v \`pwd\`/auth:/auth \\**

**-v \`pwd\`/certs:/certs \\**

**-v \`pwd\`/certs:/certs \\**

**-e REGISTRY_AUTH=htpasswd \\**

**-e REGISTRY_AUTH_HTPASSWD_REALM=\"Registry Realm\" \\**

**-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \\**

**-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \\**

**-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \\**

**-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \\**

**-p 443:443 \\**

**registry:2.7.0**

Your registry should now be running and accessible from the local
machine. If, however, you want to access it from a remote system, we
need to add a ca.crt file. You need to copy the contents of the
\~/registry/certs/domain.crt file.

Log into your second machine and create a new directory with:

**sudo mkdir -p /etc/docker/certs.d/SERVER:443**

Where SERVER is the IP address of the machine hosting the registry.

Create the new file with:

**sudo nano /etc/docker/certs.d/SERVER:443/ca.crt**

**Copy the contents of domain.crt file in ca.crt**

Where SERVER is t he IP address of the machine hosting the registry.

Paste the contents from the domain.crt file (from the hosting server)
into this new file. Save and close the file.

**How to login to the new registry**

From the second machine, open a terminal window and log into your new
Docker registry with the command:

**docker login -u user -p password https://SERVER:443**

Where USER is the user you added when you generated the htpasswd file
above and SERVER is the IP address of the machine hosting the registry.

You should be prompted for a password. Upon successful authentication,
you'll see Login Succeeded.

Copy the ca.crt file to ca-certificates 

**cp ca.crt /usr/local/share/ca-certificates/**

**update-ca-certificates**

**sudo reboot**

Congratulations, you're now able to use that self-hosted Docker registry
for your container images.

After creating the local registry:

Access your registry using

![](./image1.png){width="5.250038276465442in"
height="2.32293416447944in"}

To push image to your local registry:

-   First we need to tag our docker image using:

**sudo docker tag \<dockerimage\>:\<tag\> \<IP address\>/\<new image
name to store inside ur registry\>:\<tag or withour tag\>**

-   **For eg:** sudo docker tag nginx_image:latest
    3.27.212.183:443/venkatkalla11/ajay11

Then, proceed to push your tagged image.

-   **For eg:** sudo docker push
    3.27.212.183:443/venkatkalla11/ajay11:latest

To pull image from our local registry:

**For eg:** sudo docker pull
3.27.212.183:443/venkatkalla11/ajay11:latest
