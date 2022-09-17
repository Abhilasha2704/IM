# Certificate Authority Instructions

## Create a root CA

A root CA is the central authority to sign one or more intermediate CAs, must be kept safe after creation, and most likely will never be used afterward in our case.  

Download the directory structure and configuration files for creating CAs and certifications from [ca_structure.tgz](https://github.ibm.com/cloud-operations/scylla-db/raw/master/examples/cert-manager/ca_structure.tgz).  Extract the archive via `tar -zxvf ca_structure.tgz`.


Then change to the ca directory to perform the rest of the steps:
```
cd ca
```

### Create the CA key
`openssl genrsa -aes256 -out private/ca.key.pem 4096`

*Ensure to set a **strong passcode***

### Create the CA cert
`openssl req -config openssl.cnf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem`

Ensure to set a common name.  Example:
```
Country Name (2 letter code) [US]:
State or Province Name [NC]:
Locality Name []:
Organization Name [Digital]:
Organizational Unit Name []:
Common Name []:Digital Root CA
Email Address []:osenbach@us.ibm.com
```

Store `private/ca.key.pem` and `certs/ca.cert.pem` in a secure location and don't forget the passcode.  In my case, I used a Box folder to store the entire ca folder (finish these instructions first) in addition to a Box Note to document the passcode.




## Create an Intermediate CA

An intermediate CA is created per-environment and is used to sign additional certificates.  The certificate and key are set as secrets within your Kubernetes environments for downstream elements to make use of.  


### Create the Intermediate key

`openssl genrsa -out intermediate/private/intermediate.key.pem 4096`

### Create the Intermediate CSR

`openssl req -config intermediate/openssl.cnf -new -sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem`

Ensure to set a distinct common name while keeping the others similar to the root CA.  Example:
```
Country Name (2 letter code) [US]:
State or Province Name [NC]:
Locality Name []:
Organization Name [Digital]:
Organizational Unit Name []:
Common Name []:Digital Intermediate CA
Email Address []:osenbach@us.ibm.com
```

### Create the Intermediate Cert

`openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem`


## Create the Certificate Chain

A file with both the CA and Intermediate CA certificates is needed for verifications of the entire end-to-end authority:

`cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem`




## Create the ScyllaDB Server Certificate

This certificate and key will be used directly on each ScyllaDB node to secure the communication

### Create the Server Key

`openssl genrsa -out intermediate/private/scylla-im.key.pem 2048`


### Create the Server CSR

`openssl req -config intermediate/openssl.cnf -key intermediate/private/scylla-im.key.pem -new -sha256 -out intermediate/csr/scylla-im.csr.pem`

Ensure to set a distinct common name while keeping the others similar to the root and intermediate CAs.  Example:
```
Country Name (2 letter code) [US]:
State or Province Name [NC]:
Locality Name []:
Organization Name [Digital]:
Organizational Unit Name []:
Common Name []:Scylla DB Node
Email Address []:osenbach@us.ibm.com
```


### Create the Server Cert

`openssl ca -config intermediate/openssl.cnf -extensions server_cert -days 3650 -notext -md sha256 -in intermediate/csr/scylla-im.csr.pem -out intermediate/certs/scylla-im.cert.pem`