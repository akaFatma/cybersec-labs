#!/bin/bash

# =========================================================
# OPENSSL PKI LAB - RHEL SERVER
# This server will act as:
# 1) Certification Authority (CA)
# 2) Apache Web Server (HTTP + HTTPS)
# 3) DNS Server
# =========================================================

CA_HOME="/root/ca"

############################################################
# STEP 1 - Install required packages
############################################################
echo "STEP 1: Installing required packages"

# Install Apache, SSL support, DNS tools, OpenSSL, firewall, and VM tools
yum --disablerepo="*" --enablerepo="rhel7-local" install -y \
httpd mod_ssl bind bind-utils openssl firewalld open-vm-tools


############################################################
# STEP 2 - Prepare the CA directory structure
############################################################
echo "STEP 2: Preparing CA directory structure"

# Create folders used by the OpenSSL CA
mkdir -p $CA_HOME/{certs,crl,newcerts,private}

# Protect the private key folder
chmod 700 $CA_HOME/private

# Copy the default OpenSSL configuration file
cp /etc/pki/tls/openssl.cnf $CA_HOME/

# Change the CA directory path in the config file
sed -i 's|^dir[[:space:]]*=.*|dir             = /root/ca|' $CA_HOME/openssl.cnf

# Create CA database files
touch $CA_HOME/index.txt
echo 01 > $CA_HOME/serial


############################################################
# STEP 3 - Add SAN settings to v3_req in openssl.cnf
############################################################
echo "STEP 3: Configuring v3_req with SAN"

# Add SAN values directly inside the OpenSSL config
# Modern browsers check SAN, not  CN
if ! grep -q "subjectAltName = @alt_names" $CA_HOME/openssl.cnf; then
cat >> $CA_HOME/openssl.cnf <<EOF

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = www.esi.dz
DNS.2 = *.esi.dz
EOF
fi


############################################################
# STEP 4 - Create the Root CA
############################################################
echo "STEP 4: Creating Root CA"

# Generate the CA private key and self-signed certificate
openssl req -new -x509 -nodes \
-extensions v3_ca \
-keyout $CA_HOME/private/cakey.pem \
-out $CA_HOME/cacert.pem \
-days 3650 \
-config $CA_HOME/openssl.cnf \
-subj "/C=DZ/ST=ALGIERS/L=ALGIERS/O=ESI/OU=IT/CN=ESI CA"

# Protect the CA private key and allow reading the CA certificate
chmod 600 $CA_HOME/private/cakey.pem
chmod 644 $CA_HOME/cacert.pem


############################################################
# STEP 5 - Backup CA key and certificate
############################################################
echo "STEP 5: Backing up CA files"

# Create a backup archive of CA files
tar -czf $CA_HOME/rootca.tar.gz -C $CA_HOME private/cakey.pem cacert.pem


############################################################
# STEP 6 - Generate website private key and CSR
############################################################
echo "STEP 6: Generating website key and CSR"

# Create the website private key and CSR
# v3_req is used so SAN is included in the request
openssl req -new -nodes \
-keyout $CA_HOME/private/webkey.pem \
-out $CA_HOME/certs/newreq.pem \
-config $CA_HOME/openssl.cnf \
-reqexts v3_req \
-subj "/C=DZ/ST=ALGIERS/L=ALGIERS/O=ESI/OU=IT/CN=www.esi.dz"

# Protect the website private key
chmod 600 $CA_HOME/private/webkey.pem


############################################################
# STEP 7 - Sign the server certificate with the CA
############################################################
echo "STEP 7: Signing server certificate"

# Sign the CSR with the CA and apply v3_req extensions
openssl ca -batch \
-config $CA_HOME/openssl.cnf \
-policy policy_anything \
-extensions v3_req \
-out $CA_HOME/certs/webcert.pem \
-infiles $CA_HOME/certs/newreq.pem

# Allow reading the final certificate
chmod 644 $CA_HOME/certs/webcert.pem


############################################################
# STEP 8 - Verify the generated certificate
############################################################
echo "STEP 8: Verifying certificate"

# Verify that the certificate was signed by the CA
openssl verify -CAfile $CA_HOME/cacert.pem \
$CA_HOME/certs/webcert.pem

# Show SAN values inside the certificate
openssl x509 -in $CA_HOME/certs/webcert.pem -text -noout | grep -A1 "Subject Alternative Name"


############################################################
# STEP 9 - Install certificates for Apache
############################################################
echo "STEP 9: Installing certificates for Apache"

# Copy certificate and private key to Apache TLS directories
install -m 644 $CA_HOME/certs/webcert.pem \
/etc/pki/tls/certs/webcert.pem

install -m 600 $CA_HOME/private/webkey.pem \
/etc/pki/tls/private/webkey-clear.pem

# Restore SELinux context if needed
restorecon -Rv /etc/pki/tls || true


############################################################
# STEP 10 - Create website directories and pages
############################################################
echo "STEP 10: Creating website content"

# Create directories for HTTP and HTTPS content
mkdir -p /var/www/html/esi/site80
mkdir -p /var/www/html/esi/site443

# Create HTTP test page
cat > /var/www/html/esi/site80/index.html <<EOF
<h1>HTTP OK - www.esi.dz</h1>
EOF

# Create HTTPS test page
cat > /var/www/html/esi/site443/index.html <<EOF
<h1>HTTPS OK - www.esi.dz</h1>
EOF

# Publish the CA certificate so the client can download it
cp $CA_HOME/cacert.pem /var/www/html/esi/site80/cacert.pem

# Set correct ownership for Apache
chown -R apache:apache /var/www/html/esi

# Restore SELinux context on web files
restorecon -Rv /var/www/html/esi || true


############################################################
# STEP 11 - Configure Apache HTTP virtual host
############################################################
echo "STEP 11: Configuring HTTP virtual host"

# Configure Apache for normal HTTP
cat > /etc/httpd/conf.d/vhost.conf <<EOF
<VirtualHost *:80>
ServerAdmin webmaster@esi.dz
DocumentRoot /var/www/html/esi/site80
ServerName www.esi.dz
</VirtualHost>
EOF


############################################################
# STEP 12 - Configure Apache HTTPS virtual host
############################################################
echo "STEP 12: Configuring HTTPS virtual host"

# Configure Apache for HTTPS using the signed certificate
cat > /etc/httpd/conf.d/esi-ssl.conf <<EOF
Listen 443 https

<VirtualHost *:443>
ServerName www.esi.dz
ServerAlias esi.dz
DocumentRoot /var/www/html/esi/site443

SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/webcert.pem
SSLCertificateKeyFile /etc/pki/tls/private/webkey-clear.pem
</VirtualHost>
EOF


############################################################
# STEP 13 - Start Apache
############################################################
echo "STEP 13: Starting Apache"

# Check Apache configuration syntax
httpd -t

# Enable Apache at boot and restart it now
systemctl enable httpd
systemctl restart httpd


############################################################
# STEP 14 - Configure DNS server
############################################################
echo "STEP 14: Configuring DNS"

# Backup the original DNS config
cp /etc/named.conf /etc/named.conf.bak

# Create the DNS server configuration
cat > /etc/named.conf <<EOF
options {
directory "/var/named";
listen-on port 53 { any; };
allow-query { any; };
recursion yes;
dnssec-validation no;
};

zone "esi.dz" IN {
type master;
file "esi.dz.zone";
};
EOF

# Create the DNS zone file
cat > /var/named/esi.dz.zone <<EOF
\$TTL 86400
@ IN SOA ns1.esi.dz. root.esi.dz. (
2026030801 3600 900 604800 86400 )
IN NS ns1.esi.dz.
ns1 IN A 10.10.0.3
webserver IN A 10.10.0.3
www IN CNAME webserver.esi.dz.
EOF

# Set correct permissions on the zone file
chown root:named /var/named/esi.dz.zone
chmod 640 /var/named/esi.dz.zone

# Validate DNS configuration
named-checkconf
named-checkzone esi.dz /var/named/esi.dz.zone

# Enable and restart the DNS service
systemctl enable named
systemctl restart named


############################################################
# STEP 15 - Configure firewall
############################################################
echo "STEP 15: Configuring firewall"

# Start and enable the firewall
systemctl enable firewalld
systemctl start firewalld

# Open HTTP, HTTPS, and DNS services
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=dns

# Apply firewall changes
firewall-cmd --reload

echo "PKI LAB SERVER SETUP COMPLETED"