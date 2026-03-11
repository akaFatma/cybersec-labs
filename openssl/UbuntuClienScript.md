#!/bin/bash

# =========================================================
# OPENSSL PKI LAB - UBUNTU CLIENT
# This machine will act as:
# 1) Client test machine
# =========================================================

CLIENT_IP="10.10.0.100"
DNS_IP="10.10.0.3"
IFACE="netplan-ens33"

############################################################
# STEP 1 - Configure static IP and DNS
############################################################
echo "STEP 1: Configuring static IP and DNS"

# Configure static IP and DNS using NetworkManager
nmcli con mod "$IFACE" ipv4.method manual ipv4.addresses ${CLIENT_IP}/24 ipv4.dns ${DNS_IP}

# Restart the network connection
nmcli con down "$IFACE"
nmcli con up "$IFACE"


############################################################
# STEP 2 - Verify IP configuration
############################################################
echo "STEP 2: Checking IP configuration"

# Show IP address configuration
ip addr show


############################################################
# STEP 3 - Test connectivity
############################################################
echo "STEP 3: Testing connectivity"

# Ping server by IP
ping -c 4 10.10.0.3

# Ping server by hostname
ping -c 4 www.esi.dz

# Check DNS resolution
getent hosts www.esi.dz


############################################################
# STEP 4 - Download the CA certificate
############################################################
echo "STEP 4: Downloading CA certificate"

# Download CA certificate from the HTTP site
wget -O ~/cacert.pem http://www.esi.dz/cacert.pem

# Confirm the file exists
ls -l ~/cacert.pem

# Display first lines of the certificate
head -5 ~/cacert.pem


############################################################
# STEP 5 - Verify the CA certificate
############################################################
echo "STEP 5: Verifying CA certificate"

openssl x509 -in ~/cacert.pem -noout -subject -issuer


############################################################
# STEP 6 - Verify HTTPS certificate chain
############################################################
echo "STEP 6: Verifying HTTPS certificate chain"

openssl s_client \
-connect www.esi.dz:443 \
-servername www.esi.dz \
-CAfile ~/cacert.pem </dev/null | egrep 'Verify return code|subject=|issuer='


############################################################
# STEP 7 - Import CA certificate in Firefox
############################################################
echo "STEP 7: Import CA certificate into Firefox"

echo "Open Firefox and import: ~/cacert.pem"
echo "Path: Settings -> Privacy & Security -> Certificates -> Authorities -> Import"
echo "Check: Trust this CA to identify websites"


echo "UBUNTU CLIENT SETUP COMPLETED"