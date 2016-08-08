#Docker certificates howto

##How to generate docker certificates for private registry and api

###Generating root certificates.

Root certificates are used as signing authority.

Generate key:
`openssl genrsa -aes256  -out ca-key.pem 4096`

Sign the key:
`openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem`


###Generating the keys for docker daemon

Generate key:
`openssl genrsa -out server-key.pem 4096`

Generate certificate signing request. Replace IP with your host ip
`openssl req -subj "/CN=$IP" -sha256 -new -key server-key.pem -out server.csr`

Use extension when signing the certificate. Again, replace the $IP
`echo subjectAltName = IP:$IP,IP:127.0.0.1 > extfile.cnf`

Generate the certificate sign it with the certificate authority.
```
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
-CAcreateserial -out server-cert.pem -extfile extfile.cnf
```


### Generating the keys for docker client

`openssl genrsa -out key.pem 4096`

`openssl req -subj '/CN=client' -new -key key.pem -out client.csr`

`echo extendedKeyUsage = clientAuth > extfile.cnf`

`openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile.cnf`


`chmod -v 0400 ca-key.pem key.pem server-key.pem`
`chmod -v 0444 ca.pem server-cert.pem cert.pem`


You can verify if it works. Run daemon with:

```
docker daemon --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem \
  -H=0.0.0.0:2376
```  

Display docker version:

```
docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem \
  -H=$IP:2376 version
```


### Generating the keys for docker private registry

This is pretty much the same as generating the server keys for api. We will reusing
previously generated certificate authority

Generate key:
`openssl genrsa -out domainkey 4096`

Generate certificate signing request. Replace IP with your host ip
`openssl req -subj "/CN=$IP:5000" -sha256 -new -key domain.key -out domain.csr`

Use extension when signing the certificate. Again, replace the $IP
`echo subjectAltName = IP:$IP,IP:127.0.0.1 > extfile.cnf`

Generate the certificate sign it with the certificate authority.
```
openssl x509 -req -days 365 -sha256 -in dimain.csr -CA ca.pem -CAkey ca-key.pem \
-CAcreateserial -out domain.crt -extfile extfile.cnf
```

### Generate username and password

```
mkdir auth
docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > auth/htpasswd
```

### Copy the certificates to docker certs.d

To avoid registry complaining at the certificates signed y the unknown authority we will
copy them to docker certs directory.

On Linux docker host.

```
mkdir  /etc/docker/certs.d/$IP:5000
# Example:  mkdir -p /etc/docker/certs.d/192.168.1.10:5000
```

On Mac it gets interesting. You have to attach to mobby using sreen first

```
screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

Use root as username. It should log you in without password.

Follow the steps for linux. When you are done:

`service docker restart`

### Run the registry

```
mkdir certs
mv domain.crt domain.key certs
```

```
docker run -d -p 5000:5000 --restart=always --name registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```
