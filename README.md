# MTS-Tls-mTLS
# mTLS Activation for HTTP2 Scenarios
This part will showcase the different steps and actions done in order to activate mTLS in HTTP2 scenarios in my environment and UDM as a target example.
 
## Generate the necessary certificates


* Generate a private key:  
```
openssl genrsa -passout http2JKS -out mtspkey.key 2048
```
* Generate a certificate generation conf file `certconf.cnf` and insert the values (cert fields, client's IP, FQDN) summed below:
```
vi 

[ req ]
prompt = no
utf8 = yes
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
C = FR
L = ND1
O = 5gc.mnc001.mcc208.3gppnetwork.org
CN = hostname.5gc.mncXXX.mccYYY.3gppnetwork.org


[v3_req]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
subjectAltName = @alt_names
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth

[alt_names]
DNS.1 = hostname.5gc.mncxxx.mccyyy.3gppnetwork.org

IP.1 = 1XX.253.X.Y
```
* Generate a CSR file `mtscsr.csr`:  
```
openssl req -new -config certconf.cnf -key mtspkey.key -out mtscsr.csr
```
And verify the output:
```
openssl req -text -noout -verify -in mtscsr.csr
```
* Go to your  [PK ](https://pki/certificates/request.do) and load the `mtscsr.csr` in order to generate the certificate as an output. Steps are defined in confluence (link to be added) > Now our client certificate is ready : `hostname.5gc.mncxx.mccyy.3gppnetwork.org.pem` which we will deposite on our MTS environment.
* Retrieve the  certificate chain : root CA and intermediate CA from confluence (link to be added)  
## Install certificates in the keystores:

In order to facilitate the keystore management, [KSE (Keystore Explorer)](https://keystore-explorer.org/downloads.html) is installed.
* Open KSE and load the keystore`./conf/certficate/http2.keystore` (no password set by default):
```shell
kse
```
* Delete the simple-http-server installed  
![Image](./img/mtstls1.png)  

* Rclick and import a Pair Key > Type: **OpenSSL** > Select the private key and client certificate :  
![Image](./img/mtstls2.png)

* A prompt will ask to define a passkey for the imported pairkey, the value of *$PASS* to be inserted
* Define a password for the KeyStore with the same value of *$PASS* and proceed to save the KeyStore:  
![Image](./img/mtstls3.png) 
* Open the `./conf/certficate/http2.keystore` in KSE and install the intermediary CA and root CA through "Import Trusted Certificate"  
![Image](./img/mtstls4.png) 
* Same as for `http2.keystore`, set *$PASS* as password for this keystore and save.
* Optionnaly apply same client/server keystore setup on `./conf/certficate/client.JKS` and `./conf/certficate/server.JKS` to globalize configuration for other scenarios.

## Define TLS properties

Go to the properties file in MTS directory:`./conf/tls.properties` and update the lines including:
- SSL Version
- Two way SSL
- Keystores passwords (replace *$PASS* must **the same password** as the generated certificates passkey, and no empty password)
```
# Flag to activate the SSL Two Way (the client sends a certificate to the server)
cert.TWO_WAY=TRUE

# SSL version
cert.SSL_VERSION=TLSv1.2

# Store for HTTP2 test
cert.KEYSTORE.DIRECTORY=../conf/certificate/http2.keystore
cert.KEYSTORE.PASSWORD=$PASS

cert.TRUSTSTORE.DIRECTORY=../conf/certificate/http2.truststore
cert.TRUSTSTORE.PASSWORD=$PASS

# Path to the certificates on the SERVER side
cert.SERVER.DIRECTORY=../conf/certificate/server.jks
# Password of the certificates and keystores on the SERVER side (empty = no password)
cert.SERVER.KEYSTORE_PASSWORD=$PASS
cert.SERVER.KEY_PASSWORD=$PASS

# Path to the certificates on the CLIENT side
cert.CLIENT.DIRECTORY=../conf/certificate/client.jks
# Password of the certificates and keystores on the CLIENT side (empty = no password)
# Used only in the two way mode (cert.TWO_WAY=TRUE)
cert.CLIENT.KEYSTORE_PASSWORD=$PASS
cert.CLIENT.KEY_PASSWORD=$PASS
```
## Update HTTP2 scenario to use TLS 
* In the scenario location `./udm_ausf/Functions/http2_request_function.xml` we update the openChannel to use https in order to trigger TLS :
```XML
<openChannelHTTP2 name="[nameChannel]" remoteURL="https://[remoteHTTP]"/>
```
# Decrypt TLS messages in HTTP2 traces in Wireshark:
* Once the TLS handshake is successful between client (MTS) and server (UDM), the HTTP2 Data messages are encrypted :  
![Image](./img/mtstls5.png)
* We can use [jSSLKeyLog](https://jsslkeylog.github.io/) to log the TLS session keys in a logfile that we will load on wireshark on decrypt the packets.
* After downloading and putting `jSSLKeyLog.jar` in `./bin/` we update the `./bin/java_arguments` to include the argument `-javaagent:jSSLKeyLog.jar=./mtsTLSkey.log`:
```
-Dfile.encoding=ISO-8859-15 -Xss1m -XX:+UseConcMarkSweepGC -XX:ParallelGCThreads=1 -Djava.library.path=../lib/native -noverify -cp ../lib/*;../lib/ext1/*;../lib/ext2/*;../tutorial/core/076_parameters_pluggable/HelloWorldPluggableComponent.jar -javaagent:jSSLKeyLog.jar=./mtsTLSkey.log
```
* Once we launch the scenario in MTS, the TLS session keys will be parsed into `./bin/mtsTLSkey.log`
* In Wireshark `Edit/Preferences/Protocols/TLS` we load the `./bin/mtsTLSkey.log` in *(Pre)-Master-Secret log filename*:  
![Image](./img/mtstls6.png)
* The packets will be decrypted and HTTP2 messages become visible:  
![Image](./img/mtstls7.png)
