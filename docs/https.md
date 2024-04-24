
# HTTPS guide
 Key and Cert files are required for HTTPS, and they are not detailed, but explain how to generate them and where LoxiLB can read and use user-generated Key and Cert files.

```
      --tls              enable TLS  [$TLS]
      --tls-host=        the IP to listen on for tls, when not specified it's the same as --host [$TLS_HOST]
      --tls-port=        the port to listen on for secure connections (default: 8091) [$TLS_PORT]
      --tls-certificate= the certificate to use for secure connections (default:
                         /opt/loxilb/cert/server.crt) [$TLS_CERTIFICATE]
      --tls-key=         the private key to use for secure connections (default:
                         /opt/loxilb/cert/server.key) [$TLS_PRIVATE_KEY]
```
To enable https on LoxiLB, we changed it to enable it using the  `--tls`option. 

Tls-host and tls-port are the contents of deciding which IP to listen to. The default IP address used as tls-host is 0.0.0.0, which is everywhere, but for future security, we recommend doing only certain values. The port is 8091 as the default. You can also find and change this from a value that does not overlap with the service you use.

LoxiLB reads the key by default as /opt/loxilb/cert/path with server.key and the Cert file as server.crt in the same path. In this article, we will learn how to create the server.key and server.crt files.

You can enable and run HTTLS (TLS) with the following commands.
```
./loxilb --tls
```

## Preparation
First of all, the simplest way is to create it using *openssl*. To install openssl, you can install it using the command below.
```
apt install openssl
```
The LoxiLB team confirmed that it operates on 1.1.1f version of openssl.
```
openssl version
OpenSSL 1.1.1f  31 Mar 2020
```
### 1. Create server.key 

```
openssl genrsa -out server.key 2048
```

The way to generate server.key is simple. You can create a new key by typing the command above. In fact, if you type in the command, you can see that the process is output and the server.key is generated.
```bash
openssl genrsa -out server.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
..............................................+++++
...........................................+++++
e is 65537 (0x010001)
```

### 2. Create server.csr 

```
openssl req -new -key server.key -out server.csr
```

Create a csr file by putting the desired value in the corresponding item. This file is not used directly for https, but it is necessary to create a Cert file to be created later. When you type in the command above, a long sentence appears asking you to enter information, and you can fill in the corresponding value according to your situation.
```bash
openssl req -new -key server.key -out server.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

### 3. Create server.crt
```
openssl x509 -req -days 365 -in server.csr -signkey server.key  -out server.crt
```
This is the process of creating server.crt using server.key and server.csr generated above. You can issue a certificate with a limited deadline by setting the expiration date of the certificate well and putting a value after -day. The server.crt file is created with the following output.
```bash
openssl x509 -req -days 365 -in server.csr -signkey server.key  -out server.crt
Signature ok
subject=C = AU, ST = Some-State, O = Internet Widgits Pty Ltd
Getting Private key
```

### 4. Validation
You can enable https with the server.key and server.cert files generated through the above process.

If you move all of these files to the `/opt/loxilb` path and check them, you can see that they work well.

```bash
sudo cp server.key /opt/loxilb/cert/.
sudo cp server.crt /opt/loxilb/cert/.
./loxilb --tls
```

```bash
 curl http://0.0.0.0:11111/netlox/v1/config/loadbalancer/all
{"lbAttr":[]}

 curl -k https://0.0.0.0:8091/netlox/v1/config/loadbalancer/all
{"lbAttr":[]}
```

It should appear in the log as follows.

```bash
2024/04/12 16:19:48 Serving loxilb rest API at http://[::]:11111
2024/04/12 16:19:48 Serving loxilb rest API at https://[::]:8091
```
