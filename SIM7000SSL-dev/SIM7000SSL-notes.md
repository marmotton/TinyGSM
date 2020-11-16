# SIM7000 SSL version dev notes

These are just notes used while writing the SIM7000 SSL version. This file shouldn't make its way into the main repo...

## Example of getting a file on a web server

### Connecting to the network

```
at+cnact=1,"dr.m2m.ch"
OK

+APP PDP: ACTIVE
at+cnact?
+CNACT: 1,"10.133.65.46"

OK
```

### Connecting with TCP and SSL

```
at+cacid=0
OK
at+csslcfg="sslversion",0,3
OK
at+casslcfg=0,"ssl",1
OK
at+casslcfg=0,cacert,"dstrootx3.crt" <-- Optional, certificate must be uploaded beforehand
at+caopen=0,"eende.net",443
+CAOPEN: 0,0
```

### Getting the file

Trying to get this small text file: https://eende.net/test.txt.
It should work if using SSL. When this file is accessed using plain http, a "301 Moved Permanently" error is returned by the server.

The value "10000" means that the timeout for entering the GET request is 10s. The line endings in the GET request must be CRLF (double-check your terminal configuration, I ended up using CuteCom and its send file function).

```
AT+CASEND=0,71,10000
>
GET /test.txt HTTP/1.0
Host: eende.net
Connection: keep-alive

OK

+CASEND: 0,0,71

+CADATAIND: 0 <-- Means that data is available

AT+CARECV=0,1024 <-- This command must be sent before +CASTATE:0,0 is shown (i.e. before the server closes the connection)
AT+CACLOSE=0
```

## Example using the integrated HTTPS client

This will not be used in the TinyGSM implementation, but shows that there is hope for SSL to work.

```
at+cnact=1,"dr.m2m.ch"
at+cnact?
at+cacid=0
at+csslcfg="sslversion",0,3
at+casslcfg=0,ssl,1
at+shconf="BODYLEN",350
at+shconf="HEADERLEN",350
at+shconf="URL","https://eende.net:443"
at+shconn
at+shbod="FOOBAR",6
at+shreq="https://eende.net/test.txt",1 --> should return +SHREQ: "GET",200,65
at+shread=0,65
at+shdisc
```

## Uploading a certificate

In my case I uploaded the DST Root CA X3 certificate for letsencrypt.
Note: It should be replaced by ISRG Root X1 during January 2021. See: https://letsencrypt.org/certificates/ (pem files)

This must be done only once, it is then stored in flash in the /customer/ folder.
```
AT+CFSINIT
AT+CFSWFILE=3,"dstrootx3.crt",0,1200,10000
--> Copy-paste the certificate within 10s (size of certificate file is 1200 bytes in this case)
AT+CFSRFILE=3,"dstrootx3.crt",0,2000,0 <-- for checking that the file exists and looks correct
AT+CFSTERM
```

It might be possible to upload a file containing many certificates and then use "csslcfg convert". Not tested yet.

## Using OpenSSL server for testing

On linux, launch this command: `openssl s_server -key privkey.pem -cert cert.pem -cert_chain chain.pem -accept 13000`
Make sure port 13000 is open (e.g. port forwarding on the router)

Setup SSL and connect:
```
at+cnact=1,"dr.m2m.ch"
at+cnact?

at+cacid=0
at+csslcfg="sslversion",0,3
at+casslcfg=0,ssl,1
at+casslcfg=0,cacert,"dstrootx3.crt" <-- optional
at+caopen=0,"eende.net",13000
AT+CASEND=0,13,10000
--> Copy-pase some message that is 13 characters long
--> the message should appear on the openssl console on the PC

--> Write some message on the PC and press enter
--> +CADATAIND: 0 should be displayed
AT+CARECV=0,100
--> +CARECV: 6,hello
AT+CARECV=0,100
--> +CARECV: 0
at+caclose=0
--> PC shows "connection closed"
```
