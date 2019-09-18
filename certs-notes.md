openssl genrsa -aes256 -passout pass:password -out ca.pass.key 4096
openssl rsa -passin pass:password -in ca.pass.key -out ca.key
rm ca.pass.key



openssl req -new -x509 -days 3650 -key ca.key -out ca.pem

# client_id is *only* for the output filenames
# incrementing the serial number is important
CLIENT_ID="01-alice"
CLIENT_SERIAL=01

###### PICK ONE OF THE TWO FOLLOWING ######
###### (instrux in the CA section above) ######
# rsa
openssl genrsa -aes256 -passout pass:password -out ${CLIENT_ID}.pass.key 4096
openssl rsa -passin pass:password -in ${CLIENT_ID}.pass.key -out ${CLIENT_ID}.key
rm ${CLIENT_ID}.pass.key

# generate the CSR
# i think the Common Name is the only important thing here. think of it like
# a display name or login.
openssl req -new -key ${CLIENT_ID}.key -out ${CLIENT_ID}.csr

# issue this certificate, signed by the CA root we made in the previous section
openssl x509 -req -days 3650 -in ${CLIENT_ID}.csr -CA ca.pem -CAkey ca.key -set_serial ${CLIENT_SERIAL} -out ${CLIENT_ID}.pem


cat ${CLIENT_ID}.key ${CLIENT_ID}.pem ca.pem > ${CLIENT_ID}.full.pem

openssl pkcs12 -export -out ${CLIENT_ID}.full.pfx -inkey ${CLIENT_ID}.key -in ${CLIENT_ID}.pem -certfile ca.pem


# Create the Server Key, CSR, and Certificate
openssl genrsa -des3 -out server.key 1024
openssl req -new -key server.key -out server.csr

# We're self signing our own server cert here.  This is a no-no in production.
openssl x509 -req -days 365 -in server.csr -CA ca.pem -CAkey ca.key -set_serial 01 -out server.crt





######### on NGINX

    ssl_client_certificate /path/to/ca.pem;
    ssl_verify_client optional; # or `on` if you require client key


