# Harbor: How


## Introduction

[Harbor](https://github.com/goharbor/harbor) is an cloud native registry project. It enables developers to create their 
own container image registry.


## Create Your Own Image Registry with Harbor
**Make Sure you have Docker, OpenSSL on your Machine**

### Get the Offline [Harbor Installer](https://github.com/goharbor/harbor/releases)

Choose the corresponding release based on your machine

```shell
wget https://github.com/goharbor/harbor/releases/download/v2.11.1/harbor-offline-installer-v2.11.1.tgz
tar -zxvf harbor-offline-installer-v2.11.1.tgz -C ~/harbor
```
### Configure HTTPS access to Harbor

Implement this following script `certs_gen.sh`, reference [here](https://goharbor.io/docs/2.11.0/install-config/configure-https/)

Change the domain name using the following command:
```shell
sed -i 's/yourdomain.com/harbor.kubecenter.com/g' certs_gen.sh
```

Change `harbor.kubecenter.com` to your own domain name


```shell
#Generate a Certificate Authority Certificate
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=MyPersonal Root CA" \
 -key ca.key \
 -out ca.crt
 
#Generate a Server Certificate
openssl genrsa -out yourdomain.com.key 4096
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key yourdomain.com.key \
    -out yourdomain.com.csr
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=yourdomain.com
DNS.2=yourdomain
DNS.3=hostname
EOF


openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in yourdomain.com.csr \
    -out yourdomain.com.crt
#Provide the Certificates to Harbor and Docker

cp yourdomain.com.crt /data/cert/
cp yourdomain.com.key /data/cert/


openssl x509 -inform PEM -in yourdomain.com.crt -out yourdomain.com.cert

cp yourdomain.com.cert /etc/docker/certs.d/yourdomain.com/
cp yourdomain.com.key /etc/docker/certs.d/yourdomain.com/
cp ca.crt /etc/docker/certs.d/yourdomain.com/

systemctl restart docker
```

### Go to the path where harbor offline installer was saved

In the example above, the path is `~/harbor/harbor_offline`

Edit the `harbor.yml`. Optionally, change host name, port, and admin password. Find `install.sh`
under the same path, run `./install.sh`, which will install Harbor


### Push image to the registry in another machine

Go to `/etc/hosts` and add the machine address to `hosts`. In my machine, it's 
```shell
[put_machine_address_here] harbor.kubecenter.com
```

### Copy over the certs 
```shell
scp -r root@[domain]:/etc/docker/certs.d /etc/docker
```

### Restart Docker
```shell
systemctl restart docker
```

### Docker Login
```shell
docker login harbor.kubecenter.com
```


Now you should be able to push the images to the registry using docker commands.





