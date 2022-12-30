## Example: Use secrets with a Nginx service

### Generate CA certificates


1.  Generate a root key.

    ```
    $ openssl genrsa -out "root-ca.key" 4096

    ```

2.  Generate a CSR using the root key.

    ```
    $ openssl req\
              -new -key "root-ca.key"\
              -out "root-ca.csr" -sha256\
              -subj '/C=UK/ST=Midlothian/L=Edinburgh/O=Starx/CN=Swarm Secret Example CA'

    ```

3.  Configure the root CA. Edit a new file called `root-ca.cnf` and paste the following contents into it. This constrains the root CA to signing leaf certificates and not intermediate CAs.

    ```
    [root_ca]
    basicConstraints = critical,CA:TRUE,pathlen:1
    keyUsage = critical, nonRepudiation, cRLSign, keyCertSign
    subjectKeyIdentifier=hash

    ```

4.  Sign the certificate.

    ```
    $ openssl x509 -req  -days 3650  -in "root-ca.csr"\
                   -signkey "root-ca.key" -sha256 -out "root-ca.crt"\
                   -extfile "root-ca.cnf" -extensions\
                   root_ca

    ```

5.  Generate the site key.

    ```
    $ openssl genrsa -out "site.key" 4096

    ```

6.  Generate the site certificate and sign it with the site key.

    ```
    $ openssl req -new -key "site.key" -out "site.csr" -sha256\
              -subj '/C=UK/ST=Midlothian/L=Edinburgh/O=Starx/CN=localhost'

    ```

7.  Configure the site certificate. Edit a new file called `site.cnf` and paste the following contents into it. This constrains the site certificate so that it can only be used to authenticate a server and can't be used to sign certificates.

    ```
    [server]
    authorityKeyIdentifier=keyid,issuer
    basicConstraints = critical,CA:FALSE
    extendedKeyUsage=serverAuth
    keyUsage = critical, digitalSignature, keyEncipherment
    subjectAltName = DNS:localhost, IP:127.0.0.1
    subjectKeyIdentifier=hash

    ```

8.  Sign the site certificate.

    ```
    $ openssl x509 -req -days 750 -in "site.csr" -sha256\
        -CA "root-ca.crt" -CAkey "root-ca.key"  -CAcreateserial\
        -out "site.crt" -extfile "site.cnf" -extensions server
    ```

**Protect root-ca.key**

### Nginx conf

File: site.conf

```
server {
    listen                443 ssl;
    server_name           localhost;
    ssl_certificate       /run/secrets/site.crt;
    ssl_certificate_key   /run/secrets/site.key;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

### Create a nginx service

```
docker service create \
     --name nginx \
     --secret site.key \
     --secret site.crt \
     --secret source=site.conf,target=/etc/nginx/conf.d/site.conf \
     --publish published=3000,target=443 \
     nginx:latest \
     sh -c "exec nginx -g 'daemon off;'"
```

Verify you can reach the Nginx server

```
curl --cacert root-ca.crt https://localhost:3000
```

Verify TLS Server

```
openssl s_client -connect localhost:3000 -CAfile root-ca.crt
```

## Remove the service

```
docker service rm nginx
```
