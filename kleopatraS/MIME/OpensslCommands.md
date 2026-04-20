#!/bin/bash
#SALHI FATMA _ SADOUN AMEL SIQ2
# View the original Kleopatra S/MIME certificate request file (.p10) in readable text form.
# The request is in DER format, so -inform DER is required.
openssl req -inform DER -in certs/eden_kleo.p10 -text -noout

# Convert the Kleopatra certificate request from DER (.p10) to PEM (.pem).
# This creates a text-based PEM version that is easier to inspect and sign.
openssl req -inform DER -in certs/eden_kleo.p10 -out certs/eden_kleo.pem

# View the converted Kleopatra S/MIME certificate request file (.pem) and verify it.
# -verify checks that the CSR signature is valid.
openssl req -in certs/eden_kleo.pem -text -noout -verify

# Sign the user's certificate request with the CA using the OpenSSL CA configuration.
# This issues the final user S/MIME certificate file (eden.pem).
openssl ca -config openssl.cnf -policy policy_anything -out certs/eden.pem -infiles certs/eden_kleo.pem

# View the issued user S/MIME certificate file (.pem) in readable text form.
# This lets you confirm the subject, issuer, serial number, dates, and extensions.
openssl x509 -in certs/eden.pem -text -noout

# Convert the CA certificate from PEM format to DER/CER format for Windows import.
# This creates cacert.cer, which can be imported into Windows/Kleopatra.
openssl x509 -outform DER -in cacert.pem -out cacert.cer

# View the converted CA certificate file (.cer) in readable text form.
# Because the .cer file is in DER format, -inform DER is required.
openssl x509 -inform DER -in cacert.cer -text -noout

# Verify that the issued user certificate chains correctly to the CA certificate.
# If successful, the output should end with: certs/eden.pem: OK
openssl verify -CAfile cacert.pem certs/eden.pem