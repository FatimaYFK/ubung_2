# Kryptographische Methoden für die IT

### *Fadime Konuk*
### *c2210537019*
___
## Kurzübersicht
- [OpenSSL](#ueberschriften)
	- [Prüfung der Gültigkeit eines Zertifikates per CRL](#crl)
	- [Prüfung der Gültigkeit eines Zertifikates per OCSP](#ocsp)
  - [Erstellen eines selbstsignierten Zertifikates mit dem Subject C=AT/CN=foo
und einer Gültigkeit von 2 Jahren.](#at)
  - [Erstellen eines ECDSA Private Key (Kurve: SECP256r1) sowie eines
zugehörigen CSR (Subject s.o.)](#crs)
  - [Konvertieren eines Zertifikates vom textuellen PEM Format in ein binäres
DER Format](#der)
  - [Erstellen eines passwortgesicherten RSA Private Key (3072 bit)](#rsa)
  - [Konvertierung des oben erzeugten RSA Schlüssels in ein unverschlüsseltes
Format](#rsa1)

## Voraussetzungen
  - _openssl_ mit dem Kommandozeilentool
  - Kali oder Ubuntu

<a name="ueberschriften"></a>
## Openssl

The __openssl__ program is a command line tool for using the various cryptography functions of OpenSSL's crypto library from the shell.  It can be used for
  - Creation and management of private keys, public keys and parameters
  - Public key cryptographic operations
  - Creation of X.509 certificates, CSRs and CRLs
  - Calculation of Message Digests
  - Encryption and Decryption with Ciphers
  - SSL/TLS Client and Server Tests
  - Handling of S/MIME signed or encrypted mail
  - Time Stamp requests, generation and verification
  
_Reference_: [https://manpages.ubuntu.com/manpages/xenial/man1/openssl.1ssl.html]

<a name="crl"></a>
### Prüfung der Gültigkeit eines Zertifikates per CRL

```
openssl version
```
![image](https://user-images.githubusercontent.com/59235025/194764971-4d8d1a9a-001b-4706-9e1a-914e622cfef3.png)

```
# Zuerst benötigen wir ein Zertifikat von einer Website. Hier verwende ich Wikipedia als Beispiel. Wir können dies mit dem folgenden openssl-Befehl erhalten
openssl s_client -connect wikipedia.org:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p'
```
![image](https://user-images.githubusercontent.com/59235025/194765055-17247c8a-bea1-4b43-b495-c651746caee1.png)

```
# Dann diese Ausgabe in einer Datei, z. B. wikipedia.pem:
openssl s_client -connect wikipedia.org:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p' > wikipedia.pem
# Prüfen. ob dieses Zertifikat eine CRL-URI hat:
openssl x509 -noout -text -in wikipedia.pem | grep -A 4 'X509v3 CRL Distribution Points'
```
![image](https://user-images.githubusercontent.com/59235025/194765190-701d82e4-a04b-4ca5-b6f1-6fe45968b7a7.png)

```
# Download the CRL:
wget -O crl.der http://crl3.digicert.com/DigiCertTLSHybridECCSHA3842020CA1-1.crl

# Downloaded CRL is DER format , convert it to PEM format
openssl crl -inform DER -in crl.der -outform PEM -out crl.pem

# Mit der Option -showcerts mit dem openssl s_client können wir alle Zertifikate einschließlich der Kette sehen:
openssl s_client -connect wikipedia.org:443 -showcerts 2>&1 < /dev/null
```
![image](https://user-images.githubusercontent.com/59235025/194765618-852bc2d4-1a74-4682-8f06-8bc6d88bce61.png)

```
# Download google certificate chain
OLDIFS=$IFS; IFS=':' certificates=$(openssl s_client -connect wikipedia.org:443 -showcerts -tlsextdebug -tls1 2>&1 </dev/null | sed -n '/-----BEGIN/,/-----END/ {/-----BEGIN/ s/^/:/; p}'); for certificate in ${certificates#:}; do echo $certificate | tee -a chain.pem ; done; IFS=$OLDIFS

# Nachdem wir nun alle benötigten Daten haben, können wir das Zertifikat verifizieren.
openssl verify -crl_check -CAfile crl_chain.pem wikipedia.pem
```

<a name="ocsp"></a>
### Prüfung der Gültigkeit eines Zertifikates per OCSP

Hier nehme ich wieder Wikipedia als Beispiel.

```
# Zuerst benötigen wir ein Zertifikat von einer Website
openssl s_client -connect wikipedia.org:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p'
```

```
# Speichern Sie diese Ausgabe in einer Datei als wikipedia.pem:
openssl s_client -connect wikipedia.org:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p' > wikipedia.pem

# Prüfen, ob dieses Zertifikat einen OCSP-URI hat:
openssl x509 -noout -ocsp_uri -in wikipedia.pem

# Mit der Option -showcerts mit dem openssl s_client können wir alle Zertifikate einschließlich der Kette sehen:
openssl s_client -connect wikipedia.org:443 -showcerts 2>&1 < /dev/null

# Wir können eine OCSP-Anfrage senden und nur Text mit dem folgenden Opensl-Befehl ausgeben:
openssl ocsp -issuer chain.pem -cert wikipedia.pem -text -url http://ocsp.digicert.com

# So sieht ein guter Zertifikatsstatus aus
openssl ocsp -issuer chain.pem -cert wikipedia.pem -url http://ocsp.digicert.com
```

<a name="at"></a>
### Erstellen eines selbstsignierten Zertifikates mit dem Subject C=AT/CN=foo und einer Gültigkeit von 2 Jahren

```
openssl req -newkey rsa:2048 -new -nodes -x509 -days 730 -keyout key.pem -out cert.pem
```

![image](https://user-images.githubusercontent.com/59235025/194767620-a19452d1-6bee-4bb9-887e-cf26e239457e.png)
![image](https://user-images.githubusercontent.com/59235025/194767723-1c0e8b7e-db06-452a-8e6c-de2e5b78829f.png)

<a name="crs"></a>
### Erstellen eines ECDSA Private Key (Kurve: SECP256r1) sowie eines zugehörigen CSR (Subject s.o.)

```
# find your curve
openssl ecparam -list_curves
````
![image](https://user-images.githubusercontent.com/59235025/194767851-b686f56b-dd24-41eb-bc05-08b029b1cbf1.png)

```
# generate a private key for a curve ( SECP256r1 doesn’t exist in the list)
openssl ecparam -name secp256k1 -genkey -noout -out cert.pem
```
![image](https://user-images.githubusercontent.com/59235025/194767927-58059efe-2251-4db0-b127-8096bd820f17.png)

```
# Finally, generate the CSR as you have done:
openssl req -new -sha256 -key cert.pem -out cert.csr
```
![image](https://user-images.githubusercontent.com/59235025/194768002-209adc02-1975-4994-9224-54775e6990a3.png)














