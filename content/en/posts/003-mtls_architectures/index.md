---
title: "Improving on-premise architectures with mTLS"
date: 2025-07-06
author: Fiti
description: Example of mutual tls implementation and explanation
tags:
  - mtls
  - security
  - containers
---

## Introduction

We usually connect to most websites using HTTPS (which uses TLS). This is preferred over plain HTTP for two main reasons:

- Traffic does not travel in plain text, and
- We can ensure that the server we're connecting to is the one it claims to be.

The first point is thanks to TLS (Transport Layer Security), and the second comes from certificate validation and the role of the CA (Certificate Authority). However, in this connection, the client (us) needs to check the second point, but the server doesn't care who is connecting (unless it has configured some whitelists or firewall rules).

mTLS (mutual TLS) performs the validation done by the client in a typical HTTPS connection, but also adds validation in the opposite direction. That means the client will validate the server, and the server will also validate the client.

In other words:
- TLS: Client validates the server.
- mTLS: Client validates the server + Server validates the client.

## How mTLS works

mTLS just adds a new message in the TLS communication from the server that requests the client to authenticate, instead of simply providing the response.

- TLS:
    - The client starts the connection with the server.
    - The server provides the TLS certificate.
    - The client validates the certificate.
    - The client and the server are able to communicate.
- mTLS:
    - The client starts the connection with the server.
    - The server provides the TLS certificate **and requests the certificate of the client**.
    - The client validates the certificate **and sends its certificate**.
    - The server validates the certificate of the client and provides the response if authenticated.
    - The client and the server are able to talk to each other.
### Example of scenario

mTLS can significantly improve security by providing a secure connection that **does not rely on** credentials. Instead, it relies on certificates: certificates that are validated by a CA.

In our example, we're going to set up a server that receives internet connections, and we want to protect another server that is installed on-site, making it accessible only to the HTTP server. In fact, our HTTP server will act as a proxy for the second one. We aim to protect access to the second server not only from external sources but also from someone on the local network.

{{< mermaid >}}
sequenceDiagram
    actor Client as External Actor
    Client->>Server1: Connect Request
    box "Container A"
        participant Server1 as Server1 (exposed)
    end
    box "Container B"
        participant Server2 as Server2 (local)
    end
    Note right of Client: External access to public API
    activate Server1
    Server1->>Server2: Initiate mTLS Handshake
    Server2->>Server1: Server Certificate
    Server1->>Server2: Client Certificate
    Server2->>Server2: Validate Client Certificate
    Server1->>Server1: Validate Server Certificate
    Note over Server1, Server2: mTLS handshake (secure channel)
    Server1->>Server2: Forward request
    Server2->>Server1: Response
    Server1->>Client: Response
    Note over Server1, Server2: Same physical location
{{< /mermaid >}}

We are going to configure the HTTP server on server2 to use mTLS. The advantages of this are:

- No one (even if an attacker has gained access to the local network), except server1 (or anyone with a valid certificate), can make requests to server2.
- Access is granted using certificates instead of passwords, which is more secure and easier to manage.

### Steps

There are multiple elements in mTLS: keys, certificates, servers... Here's a summary:

- **CA**:
    - The private key is used to sign every certificate.
    - The public certificate is provided to the client and server so they can validate certificates signed by the CA.
- **Server**:
    - The private key.
    - The certificate signed by the CA, previously requested through a CSR (Certificate Signing Request).
- **Client**:
    - The private key.
    - The certificate signed by the CA, also requested via a CSR.

Since both client and server must validate each other, and both hold certificates signed by the same CA, the validation will succeed.

### Explanation (RSA, and why all this thing actually works)


- Two large *prime* numbers are chosen: $$p, q \in \mathbb{P}$$
- Calculates $n=p \cdot q$ . This will be **part of the public key**.
- Calculates (computing the Euler's Totient Function)
  $$\phi(n) = (p-1) \cdot (q-1)$$
- Choose the public exponent $e$ as:
  $$1 < e < \phi(n), gcd(e, \phi(n)) = 1$$
- Compute the private exponent $d$ as:
$$  d\equiv e^{-1} \mod \phi(n)$$
- So that: 
$$e\cdot d \equiv 1 \mod \phi(n)$$
In other words, it is the inverse multiplicative.
We have now the elements of the keys:
- Public key: $(e,n)$
- Private key: $(d,n)$

#### 1. The CA has its RSA Key pair

CA sets its private key $d=7$ and public $e=3$, $n=33$, because $n=p \cdot q$ where $p=3$ and $q=11$ are primes (these small numbers are just for example!)

$$d=7$$
$$e=3$$
$$n=33$$
#### 2. The server generates a `CSR`.

Let's say that the CSR hash is:

$$m=5$$
#### 3. The CA signs the `CSR`:

The CA generates the signature (the signed certificate):

$$ certificate = m ^d \mod n \equiv 5 ^7 \mod 33 \equiv 14$$
#### 4. The client validates the certificate provided by the server

When the client receives the certificate, it checks agains the certificate of the CA:

$$m\prime = certificate^e\mod n \equiv 14^3 \mod 33 \equiv 5$$
$$m=m\prime$$

And since the client has in the public key (the certificate) the value of the `CSR` (5), it can conclude that the certificate has been signed by the CA so its valid.

## Real example

In order to practice this, I am going to create it in a virtual environment using containers.

First, our directories will look like:

```shell
.
├── docker-compose.yml
├── server1/
│   └── Dockerfile
├── server2/
│   ├── Dockerfile
│   └── index.html
└── requirements/
    ├── ca.crt
    ├── server1.crt
    ├── server1.key
    ├── server2.crt
    └── server2.key

```

- **Server 1**: The server exposed to the internet. It will be the only one able to connect to `server2`.
- **Server 2**: The protected server that will allow access only to clients with signed certificates.

It is time to generate the certificates. The steps are:

1. Generate the CA private key and self-signed certificate. This will be our root of trust.
2. Generate private keys for server1 and server2.
3. Generate the CSR for each server.
4. Sign these CSRs with the CA and generate certificates for both servers.

This script performs these steps:

```shell
# Create CA, Server and Client keys.

openssl genrsa -out server1.key 2048
openssl genrsa -out server2.key 2048
openssl genrsa -out ca.key 2048

# Create self-signed certificate for CA only.

openssl req -x509 -new -key ca.key -out ca.crt -nodes -days 365 -subj "/C=ES/ST=Spain/L=Spain/O=ORG/OU=TEST/CN=ca.com"

# Create CSR.

openssl req -new -key server1.key -out server1.csr -nodes -subj "/C=ES/ST=Spain/L=Spain/O=ORG/OU=TEST/CN=server1"
openssl req -new -key server2.key -out server2.csr -nodes -subj "/C=ES/ST=Spain/L=Spain/O=ORG/OU=TEST/CN=server2"

# Sign CSR with CA cert and priv key.

openssl x509 -req -in server1.csr -CA ca.crt -CAkey ca.key -out server1.crt -days 365
openssl x509 -req -in server2.csr -CA ca.crt -CAkey ca.key -out server2.crt -days 365
```

Now the following files will help us to generate containers:

The server1 `Dockerfile` will do nothing:

```dockerfile
FROM ubuntu:22.04
```

The server2 `Dockerfile` will install the `nginx` and dependencies. Then, configure it to serve the web with `nginx`:

```dockerfile
FROM ubuntu:22.04

RUN apt update && apt install -y nginx openssl

COPY server2/index.html /var/www/html/index.html
COPY ./requirements /etc/nginx/certs/

RUN mkdir -p /etc/nginx/snippets

COPY server2/nginx.conf /etc/nginx/sites-available/default

CMD ["nginx", "-g", "daemon off;"]
```

In the `index.html` we place an example:

```html
<!DOCTYPE html>
<html>
  <body><h1>Hello, mutual TLS is working</h1></body>
</html>
```

And in the `nginx.conf` file:

```nginx
server {
    listen 443 ssl;
    server_name server2;

    ssl_certificate /etc/nginx/certs/server2.crt;
    ssl_certificate_key /etc/nginx/certs/server2.key;
    ssl_client_certificate /etc/nginx/certs/ca.crt;
    ssl_verify_client on;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

> Note that this is where we're forcing the server to request and validate the client's certificate as well.

And finally in the `docker-compose.yml` file:

```yaml
services:
  server2:
    build:
      context: .
      dockerfile: server2/Dockerfile
    ports:
      - "8443:443"
    volumes:
      - ./requirements:/etc/nginx/certs:ro

  server1:
    build:
      context: .
      dockerfile: server1/Dockerfile
    depends_on:
      - server2
    volumes:
      - ./requirements:/certs:ro
    command: >
      sh -c "apt update && apt install -y curl ca-certificates \
      && curl --cert /certs/server1.crt --key /certs/server1.key --cacert /certs/ca.crt https://server2:443 &&\
      tail -f /dev/null"
```

> Note that in the server 1 we automatically request the webpage providing the valid certificates.


After a `docker compose up` command, we will see the following output:

```shell
➜  003-mtls docker compose up
[+] Running 2/2
 ✔ Container 003-mtls-server2-1  Created                                                                                                                                                                                     0.0s
 ✔ Container 003-mtls-server1-1  Created                                                                                                                                                                                     0.0s
Attaching to server1-1, server2-1
server1-1  |
server1-1  | WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
server1-1  |
server1-1  | Hit:1 http://ports.ubuntu.com/ubuntu-ports jammy InRelease
server1-1  | Hit:2 http://ports.ubuntu.com/ubuntu-ports jammy-updates InRelease
server1-1  | Hit:3 http://ports.ubuntu.com/ubuntu-ports jammy-backports InRelease
server1-1  | Hit:4 http://ports.ubuntu.com/ubuntu-ports jammy-security InRelease
server1-1  | Reading package lists...
server1-1  | Building dependency tree...
server1-1  | Reading state information...
server1-1  | 6 packages can be upgraded. Run 'apt list --upgradable' to see them.
server1-1  |
server1-1  | WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
server1-1  |
server1-1  | Reading package lists...
server1-1  | Building dependency tree...
server1-1  | Reading state information...
server1-1  | ca-certificates is already the newest version (20240203~22.04.1).
server1-1  | curl is already the newest version (7.81.0-1ubuntu1.20).
server1-1  | 0 upgraded, 0 newly installed, 0 to remove and 6 not upgraded.
server1-1  |   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
server1-1  |                                  Dload  Upload   Total   Spent    Left  Speed
server1-1  | <!DOCTYPE html>
server1-1  | <html>
server1-1  |   <body><h1>Hello, mutual TLS is working</h1></body>
server1-1  | </html>
100    84  100    84    0     0  16122      0 --:--:-- --:--:-- --:--:-- 16800
Gracefully stopping... (press Ctrl+C again to force)
```

As you can see, the response is being received correctly. If we want to be sure of what is happening, we can enter in the container manually and try some commands.

```shell
root@b9f1035b1511:/# curl https://server2:443
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
root@b9f1035b1511:/#
```

The client is blocking the connection because it does not trust the server (the certificate is not "valid"). Here we have two alternatives to solve this:
- Provide the `ca.crt` for verification. Since server2's certificate was signed by the CA, it will be accepted as valid.

```shell
root@b9f1035b1511:/# curl https://server2:443 --cacert /certs/ca.crt
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
  ```

- Include the `--insecure` flag:
```shell
root@b9f1035b1511:/# curl https://server2:443 --insecure
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
  ```

In both cases, the new response is the same: **The server is refusing to reply because we did not send the certificate**.

```shell
root@b9f1035b1511:/# openssl req -x509 -new -key /certs/server1.key -out selfsigned.crt -nodes -days 365 -subj "/C=ES/ST=Spain/L=Spain/O=ORG/OU=TEST/CN=server1.com"
```

And try to use it:

```shell
root@b9f1035b1511:/# curl https://server2:443 --cacert /certs/ca.crt --key /certs/server1.key --cert selfsigned.crt
<html>
<head><title>400 The SSL certificate error</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>The SSL certificate error</center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
```

The server is now **receiving our certificate, but it isn't valid**.

If we use the certificate that we signed at the beggining:
```shell
root@b9f1035b1511:/# curl https://server2:443 --cacert /certs/ca.crt --key /certs/server1.key --cert /certs/server1.crt
<!DOCTYPE html>
<html>
  <body><h1>Hello, mutual TLS is working</h1></body>
</html>
```

So, in conclusion, only clients with certificates signed by the CA will be able to establish a connection to the server.
