# Client-side SSL
For excessively paranoid client authentication.

---

Updated Apr 5 2019:

because this is a gist from 2011 that people stumble into and _maybe_ you should AES instead of 3DES in the year of our lord 2019.

some other notes:

I've noticed that across platforms, some browsers/devices like like PFX bundles, others like PEMs, some things will import ECC certs just fine but fail to list them in the "select certificate" menu when the server wants it. Server-side stuff seems good, with most things supporting ECC, but clients are a crapshoot. I'd say unless you've got some time to experiment, you may want to stick to RSA.

(In my own dev servers i just ended up configuring both an RSA CA and an ECC CA and using them both on the server, and provisioning one of each type for each client and trying them both. if, like nginx, your server only lets you use one CA cert root, **you can concatenate multiple CA PEMs together** and then [use that combined file](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_client_certificate).)

---

## Using self-signed certificate.
### Create a Certificate Authority root
This'll represent you / your org / your server -- basically the thing that vouches for the validity of a key.

```bash
###### PICK ONE OF THE TWO FOLLOWING ######

# OPTION ONE: RSA key. these are very well-supported around the internet.
# you can swap out 4096 for whatever RSA key size you want. this'll generate a key
# with password "xxxx" and then turn around and re-export it without a password,
# because genrsa doesn't work without a password of at least 4 characters.
#
# some appliance hardware only works w/2048 so if you're doing IOT keep that in
# mind as you generate CA and client keys. i've found that frirefox & chrome will
# happily work with stuff in the bigger 8192 ranges, but doing that vs sticking with
# 4096 doesn't buy you that much extra practical security anyway.

openssl genrsa -aes256 -passout pass:xxxx -out ca.pass.key 4096
openssl rsa -passin pass:xxxx -in ca.pass.key -out ca.key
rm ca.pass.key

# OPTION TWO: make an elliptic curve-based key.
# support for ECC varies widely, and support for the predefined curves also varies.
# it's "secp256r1" in this case, which is as well-supported as it gets but if you want to
# avoid NIST-provided things, or if you want to go with bigger/newer keys, you can
# swap that out:
#
# * check your openssl supported curves: `openssl ecparam -list_curves`
# * check client support for whatever browser/language/system/device you want to use:
#      https://en.wikipedia.org/wiki/Comparison_of_TLS_implementations#Supported_elliptic_curves

openssl ecparam -genkey -name secp256r1 | openssl ec -out ca.key

###### END  "PICK ONE" SECTION ######

# whichever you picked, you should now have a `ca.key` file.

# now generate the CA root cert
# when prompted, use whatever you'd like, but i'd recommend some human-readable Organization
# and Common Name.
openssl req -new -x509 -days 3650 -key ca.key -out ca.pem
```

### Create the Client Key and CSR

```bash
# client_id is *only* for the output filenames
# incrementing the serial number is important
CLIENT_ID="01-alice"
CLIENT_SERIAL=01

###### PICK ONE OF THE TWO FOLLOWING ######
###### (instrux in the CA section above) ######
# rsa
openssl genrsa -aes256 -passout pass:xxxx -out ${CLIENT_ID}.pass.key 4096
openssl rsa -passin pass:xxxx -in ${CLIENT_ID}.pass.key -out ${CLIENT_ID}.key
rm ${CLIENT_ID}.pass.key
# ec
openssl ecparam -genkey -name secp256r1 | openssl ec -out ${CLIENT_ID}.key
###### END  "PICK ONE" SECTION ######

# whichever you picked, you should now have a `client.key` file.

# generate the CSR
# i think the Common Name is the only important thing here. think of it like
# a display name or login.
openssl req -new -key ${CLIENT_ID}.key -out ${CLIENT_ID}.csr

# issue this certificate, signed by the CA root we made in the previous section
openssl x509 -req -days 3650 -in ${CLIENT_ID}.csr -CA ca.pem -CAkey ca.key -set_serial ${CLIENT_SERIAL} -out ${CLIENT_ID}.pem
```

#### Bundle the private key & cert for end-user client use

basically https://www.digicert.com/ssl-support/pem-ssl-creation.htm , with the entire trust chain

    cat ${CLIENT_ID}.key ${CLIENT_ID}.pem ca.pem > ${CLIENT_ID}.full.pem

#### Bundle client key into a PFX file
Most browsers will happily use this if they don't like the raw ascii PEM file. You'll _possibly_ need to set a password here, which you'll need on the browser/client end when you import the key+cert PFX bundle.

    openssl pkcs12 -export -out ${CLIENT_ID}.full.pfx -inkey ${CLIENT_ID}.key -in ${CLIENT_ID}.pem -certfile ca.pem

### Install Client Key on client device (OS or browser)
Use `client.full.pfx` (most commonly accepted in GUI apps) and/or `client.full.pem`. Actual instructions vary.

### Install CA cert on nginx
So that the Web server knows to ask for (and validate) a user's Client Key
against the internal CA certificate.

    ssl_client_certificate /path/to/ca.pem;
    ssl_verify_client optional; # or `on` if you require client key

Configure nginx to pass the authentication data to the backend application:

* [Client Side Certificate Auth in Nginx](http://blog.nategood.com/client-side-certificate-authentication-in-ngi), section “Passing to PHP.”
* [SSL module documentation](http://wiki.nginx.org/NginxHttpSslModule)

See also:

* my other gist, on doing the key / CSR dance for your HTTPS server: https://gist.github.com/mtigas/6177424
* mozilla's SSL configuration generator: https://mozilla.github.io/server-side-tls/ssl-config-generator/

## Using CACert Keys
(removed)
