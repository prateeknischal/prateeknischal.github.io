---
title: "Trust the Certs"
date: 2021-08-12T01:05:22+05:30
description: "What are x509 certs made of"
---

## Why do you remember certificates
As an IT professional, you might have come across this error while browsing sites of setting up your local development environment.
![bad ssl](https://i.imgur.com/aowiN2p.png)

As a fix, you would have been recommended to go to the advanced option in this error and accept the risk and allow the website to work. Your browser may still warn you by displaying the lock on the ominbar of your website as red or unsafe, but you choose to ignore it and not deal with it.

Why do you see this error? Browsers don't do a very good job of explaining the error upfront. Only if you go on the [Learn more](https://www.youtube.com/watch?vdQw4w9WgXcQ) link, you might hope to get some more information about the error. Sometimes it may confuse you more.

## What does the above error mean
The error message says `NET::ERR_CERT_INVALID`, but what does that mean. It's doesn't tell what's invalid and why. If you [venture more](https://self-signed.badssl.com/) and click on the error, you will be presented with this information.
```
Subject: *.badssl.com
Issuer: *.badssl.com
Expires on: 9 Oct 2021
Current date: 8 Aug 2021

PEM encoded chain:
-----BEGIN CERTIFICATE-----
MIIDeTCCAmGgAwIBAgIJAPziuikCTox4MA0GCSqGSIb3DQEBCwUAMGIxCzAJBgNV
    ---SNIPPED---
EVA0pmzIzgBg+JIe3PdRy27T0asgQW/F4TY61Yk=
-----END CERTIFICATE-----
```
What are these terms and what is a `CERTIFICATE`. Before we understand what the error is, let's take a look at what are the things and what are they supposed to be.

## History of the internet
In the early days, internet used to be an academic place. One of the earliest implementation of the "internet" (inter connected networks) was the [ARPANET](https://en.wikipedia.org/wiki/ARPANET) which was a Defense project, i.e. the US military. A few campuses across the US were connected to this network and it allowed them to talk to each other over the "network".

Once this technology was made public, people started experimenting with it to share information. This slowly developed into information served in pretty pages with the development of World Wide Web. There is a long history between how internet transformed and how WWW was developed by [Sir Tim Berners-Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee).

WWW (loosely referred as internet in this post) became a place to share information. Due to the nature of communication, anyone who was connected to the network could talk to other people on the network, it became a ground for moving sensitive information. People started moving money on the network in the form of goods. Now, you wouldn't want anyone to "intercept" that payment intended for someone else after all the information is just a bunch of electrical signals moving around wires. You could just tap into it and evesdrop.

There was a requirement of providing a secure channel between 2 parties exchanging sensitive information over an inherently insecure channel.

## What did we do to protect the internet

A very simple way to protect the data flowing on the networks is to encrypt it. Encrypting something implies that the data, using a secret or **key**, is reversibly transformed into a different form (gibberish) which does not make sense to anyone who is not involved in the process (It isn't so simple, but let's not diverge).

The earliest attempt to provide a secure conduit over an insecure channel was [SSL 1.0](https://security.stackexchange.com/questions/41658/what-was-ssl-1-0) which didn't even see the light of day. Let's just say it wasn't very secure.

Then came multiple revisions of the technology which tried to improve upon it and prove stronger guarantees for protecting the data from an evesdropper. They were [SSL v2](https://www-archive.mozilla.org/projects/security/pki/nss/ssl/draft02.html), [SSL v3](https://datatracker.ietf.org/doc/html/rfc6101) and then finally SSL was retired in favour of [TLS v1](https://www.ietf.org/rfc/rfc2246.txt) which was prevalent for a very long time. As time went by, flaws were found in the TLS v1 protocol was updated to [TLS v1.1](https://datatracker.ietf.org/doc/html/rfc4346) and then [TLS v1.2](https://datatracker.ietf.org/doc/html/rfc5246) which is the most used protocol. [TLS v1.3](https://datatracker.ietf.org/doc/html/rfc8446) offers stronger security and faster performance over the older version and is being adopted by the general public.

What do these protocols mean? They are standards on how to protect the data on the network. They mainly rely on encrypting the data using algorithms like [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard).

There is a problem though. AES is a **symmetric** encryption algorithm. This means, the same **key** is used for encrypting and decrypting the data. Between 2 parties that are thousands of miles apart, how do we agree on a key and what happens if one of them leaks the key. Turns out that encrypting the data safely is not that big a problem than arriving at a common key to do so.

## Enters Asymmetric encryption

Asymmetric encryption is a scheme involving a key which consists of 2 parts, a public and a private part. The private part should be protected by the owner and the public part can be distributed to everyone. A public key can be used to encrypt information which can only be decrypted by the corresponding private key.

Now everyone can have their own key pairs and use it to share information on an insecure channel. If Alice wants to share information with Bob, Alice can encrypt is using Bob's public key and send it over the wire and only Bob could use his private key to decrypt it and make sense of it.

This constitues [Public key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography) which provided 2 features
* Public key encryption - Encryption happens using a public key (available to all) and decryption happens using a private key (owned by a single entity).
* Digital signature - A methematically proven method to prove ownership of the private key without exposing anything.

There is still a problem, they are not authenticated, which means, Alice is sharing information securely with _someone_. There is no way to prove that it's Bob unless they are aware of each other's public keys before hand which is not practical when there are so many people to talk to.

This problem is _managed_ by the [Public Key Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure). It allows to put identity to public keys.

## Public Key Infrastructure

Public key infrastructure is a system of vetting a public keys by a trusted third party. For example, between Alice, Bob and Charlie, if both Alice and Bob trust Charlie, they are going to trust Charlie's word on which public keys belong to whom. Charlie here will be termed as a [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority).

Technically it's a chicken an egg problem, how do you trust Charlie without meeting them. But that's a topic for another day.

## Back to the original problem
Notice the error message on the image, It says,
```
Subject: *.badssl.com
Issuer: *.badssl.com
```
Chrome does not **trust** `*.badssl.com`, so it's not going to trust anything vetted by it and hence is rejecting the connection.

## What's that strange blob of text below it
PKI puts identity to a public key. But how does it do that? Remember how the is a common trusted third party. The third party _digitally signs_ the identity of the private key holder and their public key and distributes it to everyone who wants it.

The `CERTIFICATE` is to **certify** the identity of the owner of the private part of the public key. You may choose to trust or not trust the third party who is vouching for this certification.

Into the weeds, how do we actually make sense of the strange blob. Here's how
```
echo "-----BEGIN CERTIFICATE-----
MIIDeTCCAmGgAwIBAgIJAPziuikCTox4MA0GCSqGSIb3DQEBCwUAMGIxCzAJBgNV
BAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRYwFAYDVQQHDA1TYW4gRnJhbmNp
c2NvMQ8wDQYDVQQKDAZCYWRTU0wxFTATBgNVBAMMDCouYmFkc3NsLmNvbTAeFw0x
OTEwMDkyMzQxNTJaFw0yMTEwMDgyMzQxNTJaMGIxCzAJBgNVBAYTAlVTMRMwEQYD
VQQIDApDYWxpZm9ybmlhMRYwFAYDVQQHDA1TYW4gRnJhbmNpc2NvMQ8wDQYDVQQK
DAZCYWRTU0wxFTATBgNVBAMMDCouYmFkc3NsLmNvbTCCASIwDQYJKoZIhvcNAQEB
BQADggEPADCCAQoCggEBAMIE7PiM7gTCs9hQ1XBYzJMY61yoaEmwIrX5lZ6xKyx2
PmzAS2BMTOqytMAPgLaw+XLJhgL5XEFdEyt/ccRLvOmULlA3pmccYYz2QULFRtMW
hyefdOsKnRFSJiFzbIRMeVXk0WvoBj1IFVKtsyjbqv9u/2CVSndrOfEk0TG23U3A
xPxTuW1CrbV8/q71FdIzSOciccfCFHpsKOo3St/qbLVytH5aohbcabFXRNsKEqve
ww9HdFxBIuGa+RuT5q0iBikusbpJHAwnnqP7i/dAcgCskgjZjFeEU4EFy+b+a1SY
QCeFxxC7c3DvaRhBB0VVfPlkPz0sw6l865MaTIbRyoUCAwEAAaMyMDAwCQYDVR0T
BAIwADAjBgNVHREEHDAaggwqLmJhZHNzbC5jb22CCmJhZHNzbC5jb20wDQYJKoZI
hvcNAQELBQADggEBAGlwCdbPxflZfYOaukZGCaxYK6gpincX4Lla4Ui2WdeQxE95
w7fChXvP3YkE3UYUE7mupZ0eg4ZILr/A0e7JQDsgIu/SRTUE0domCKgPZ8v99k3A
vka4LpLK51jHJJK7EFgo3ca2nldd97GM0MU41xHFk8qaK1tWJkfrrfcGwDJ4GQPI
iLlm6i0yHq1Qg1RypAXJy5dTlRXlCLd8ufWhhiwW0W75Va5AEnJuqpQrKwl3KQVe
wGj67WWRgLfSr+4QG1mNvCZb2CkjZWmxkGPuoP40/y7Yu5OFqxP5tAjj4YixCYTW
EVA0pmzIzgBg+JIe3PdRy27T0asgQW/F4TY61Yk=
-----END CERTIFICATE-----" | openssl x509 -noout -text
```
The above command dumps the following information
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 18222331727589575800 (0xfce2ba29024e8c78)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=California, L=San Francisco, O=BadSSL, CN=*.badssl.com
        Validity
            Not Before: Oct  9 23:41:52 2019 GMT
            Not After : Oct  8 23:41:52 2021 GMT
        Subject: C=US, ST=California, L=San Francisco, O=BadSSL, CN=*.badssl.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c2:04:ec:f8:8c:ee:04:c2:b3:d8:50:d5:70:58:
                        ---SNIPPED---
                    7c:f9:64:3f:3d:2c:c3:a9:7c:eb:93:1a:4c:86:d1:
                    ca:85
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Subject Alternative Name:
                DNS:*.badssl.com, DNS:badssl.com
    Signature Algorithm: sha256WithRSAEncryption
         69:70:09:d6:cf:c5:f9:59:7d:83:9a:ba:46:46:09:ac:58:2b:
             ---SNIPPED---
         00:60:f8:92:1e:dc:f7:51:cb:6e:d3:d1:ab:20:41:6f:c5:e1:
         36:3a:d5:89
```
Notice the following things

* Serial Number: The unique identifier for the certificate with the issuer
* Issuer: The full name of who issued (vetted) the public key
* Subject: The identity of how owns the private key
* Subject Public key: The actual public key which is vetted
* Signature Algorithm: The algorithm used to sign the identity and public key information
* Validity: The dates for which this identity is considered valid

This is an [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) style keypair. So the public key has a Modulus and an Exponent. These values can be use to encrypt a message.

The signature Algorithm is the algorithm used to _digitally sign_ the information in the certificate so that anyone can verify the validity of the certificate. This verification is done by the public key of the Issuer or the trusted third party who we already trust as posses their public key.

## How does a trusted certificate look like

This is a certificate that I don't trust, but how does a certificate look like that I trust.
```
echo | openssl s_client -connect www.google.com:443 2>/dev/null \
     | openssl x509 -noout -text
```
This looks something like
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            3a:6a:c0:d8:53:c9:8f:f1:0a:00:00:00:00:f2:c8:bb
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Google Trust Services LLC, CN=GTS CA 1C3
        Validity
            Not Before: Jul 12 03:48:05 2021 GMT
            Not After : Oct  4 03:48:04 2021 GMT
        Subject: CN=www.google.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b5:87:a3:12:1a:60:3d:6f:74:f1:33:0f:67:60:
                        ---SNIPPED--
                    11:3e:1f:ba:8b:59:d9:e8:6c:a0:6c:3e:07:31:e9:
                    56:bb
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                2D:42:0A:99:66:0A:65:08:C7:B4:59:B8:23:55:13:FA:A2:D1:A2:01
            X509v3 Authority Key Identifier:
                keyid:8A:74:7F:AF:85:CD:EE:95:CD:3D:9C:D0:E2:46:14:F3:71:35:1D:27

            Authority Information Access:
                OCSP - URI:http://ocsp.pki.goog/gts1c3
                CA Issuers - URI:http://pki.goog/repo/certs/gts1c3.der

            X509v3 Subject Alternative Name:
                DNS:www.google.com
            X509v3 Certificate Policies:
                Policy: 2.23.140.1.2.1
                Policy: 1.3.6.1.4.1.11129.2.5.3

            X509v3 CRL Distribution Points:

                Full Name:
                  URI:http://crls.pki.goog/gts1c3/fVJxbV-Ktmk.crl

            1.3.6.1.4.1.11129.2.4.2:
                ......u.\.C....ED.^..V..7...G..s..^........z..4=.....F0D..
..4.8...xK.].rk.....o.R
..^.T%.S..vt11..v..\./.w0".T..0.V..M..3.../ ..N.d....z..3......G0E. ...|..Y...R2.......k...+.....8.p.!..SF...+.o..').D.........mc....(A
    Signature Algorithm: sha256WithRSAEncryption
         d7:9f:97:e8:2a:5c:bc:04:9d:51:9d:ab:d4:9b:b0:cd:71:b5:
            ---SNIPPED---
         45:b0:52:39
```

This certificate has a lot more information about the identity of the `Subject` and what this key is allowed to do.

## A scary view of the certificate

Try the following commands
```bash=
echo | openssl s_client -connect www.google.com:443 2>/dev/null \
     | openssl x509 \
     | openssl asn1parse -i

echo | openssl s_client -connect www.google.com:443 2>/dev/null \
     | openssl x509 \
     | openssl asn1parse -i -strparse 196
```
Look for interesting stuff in the output!

Now that we know how the certificate look and what does it contain, let's see how to use it for a local development environment.

## Using certificates in a local development environment

Using self signed certificates can be a pain for 2 reasons
- There can be a lot depding on what domains you are issuing them for
- You will have to deal with pesky errors from browsers and clients setting to ignore SSL verification everywhere which isn't ideal.

Since, you control your development environment, why not just become the **Certificate Authority** and use it to issue all certificates. You trust just one certificate and any certificate issued by that third part (which is you) is automatically trusted by your clients.

Ok, easier said than. How do I issue my own Certificate Authority.

>As they say, Hold my beer!

```bash=
# Assuming you have Golang installed

git clone https://github.com/square/certstrap
cd certstrap
go build

# create a directory for your binaries, useful for other stuff as well
mkdir "$HOME/.bin"
mv certstrap "$HOME/.bin/certstrap"

# put certstrap in the PATH
export PATH="$PATH:$HOME/.bin"

certstrap --version
```
If your package manager has certstrap, it's even simpler. And that's it, you are good to issue certificates

## Generate your Certificate authority
```bash=
certstrap init --common-name domain.com \
    --organization Domain \
    --organizational-unit AllThingsSecure \
    --country IN \
    --province KA
```

Enter the passphrase prompted. You always want to protect your CAs because it will essentially become a trusted certificate and if this leaks, bad things can happen to you, like impersonating Gmail or your banks! Be careful!!

The above command should dump out 3 files
```
Created out/domain.com.key
Created out/domain.com.crt
Created out/domain.com.crl
```
These are your Key, Certificate and Certificate Revocation List (used to revoke certificates issued by this CA if you choose to, basically to say that I no longer vet a particular certificate).

## Issue your first certificate

Once your CA is issued, you can use it issue a certificate.

```bash=
certstrap request-cert \
    --common-name foo.domain.com \
    --domain foo.domain.com,bar.domain.com \
    --country IN --organization Domain --organizational-unit NewTeam
```
**Note**: Make sure to have the Common Name in the `--domain` flag since Chrome will reject if the CN is not part of the SAN attributes as [browsers no longer trust the CN field alone](https://myarch.com/mandatory-certificate-extensions/).

The above command is going to generate 2 files
```
Created out/foo.domain.com.key
Created out/foo.domain.com.csr
```
One of them is an RSA private key and the other is a CSR or a Certificate Signing Request. This is like a form you fill up and ask the Certificate Authority to certify. The CA may do it's own additional checks to make sure the information is correct. In this case, since we are the CA, we know it's trustworthy. So, we certify by signing it using the CA key like below

```bash=
certstrap sign foo.domain.com \
    --CA domain.com \
    --expires "3 months"
    --csr out/foo.domain.com.csr
```
Answer all the password prompts, and there you have it, your certificate. You ideally want to issue short lived certificates for dev usage but you can issue for a longer duration but not longer than [397 days](https://support.globalsign.com/ssl/general-ssl/397-day-maximum-tls-certificate-validity). You generate the CA once and then use the `request-cert` and `sign` step to generate as many certs you want.

### What if I need more info in the certficiate
This is a very simple way to generate certificates. This is likely sufficient for a local web development environment but if you have complicated needs which require different attributes, encoding addition and custom information, you are likely going to need a better way. [openssl certificate authority](https://jamielinux.com/docs/openssl-certificate-authority/) is an excellent resource to build you certificates with all the knobs exposed.

## Make your system trust your CA

All operating systems have a default list of certificates that it trusts, they are usually very well known and trusted CAs. The browser and clients can either refer to the Operating system list for trusting CAs or have their own list. For example,
* Chrome uses the system default list
* Firefox has its own list
* NodeJS has it's own list and for any new CAs it uses the file pointed by `NODE_EXTRA_CA_CERTS`.

You can refer to your platform documentation, on how to import/add a new trusted certificate. For MacOS, it's pretty straight forward, just add it in the `Keychain Access` application in the `System` list. For Linux, refer to the man page for [`update-ca-certificates`](https://manpages.ubuntu.com/manpages/xenial/man8/update-ca-certificates.8.html) utility.

Windows users are on their own!

Once you have the CA added, all of your applications should trust all certificates issued by your CA.

## The issuer heirarchy

As mentioned, in general CA keys are kept in a highly secure location with no access to the network. They are usually valid for more than 10 years which means loosing them would cause a huge chaos. How do we then use it to issue certificates. To solve this problem, other certificates are issued called **Issuers** which are valid for a smaller time duration and are allowed to issue other **Issuers** or certificates called **leaf** certificates as they are no longer allowed to issue other certs, so last in the tree of certs or the leaf.

```bash=
echo | openssl s_client -connect google.com:443 >/dev/null
```
produces,
```
depth=3 C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
verify return:1
depth=2 C = US, O = Google Trust Services LLC, CN = GTS Root R1
verify return:1
depth=1 C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
verify return:1
depth=0 CN = *.google.com
verify return:1
DONE
```
Notice how the server presents 4 certificates,
* `C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA` The Root CA
* `C = US, O = Google Trust Services LLC, CN = GTS Root R1` an intermediate issuer CA
* `C = US, O = Google Trust Services LLC, CN = GTS CA 1C3` another layer of intermediate issuer CA
* `CN = *.google.com` the final leaf certificate for the server

## What are certificates made of

Now that you have seen how to make a simple certificate chain, let's see what each item in a sample certificate mean.
Looking at the google.com certificate,

* **Version** denotes the certificate structure, version 3 is the [`x509v3`](https://en.wikipedia.org/wiki/X.509) type of certificate.
* **Serial Number** notes the unique identifier for the certificate. This can be same for 2 different issuers but can't be same for any 2 certificate for a single issuer.
* **Signature Algorithm** defines what algorithm is used to digitally sign the data in the certificate by the Issuer. `sha256WithRSAEncryption` means that the data was hashed using `SHA256` and then encrypted using the private key of the issuer (the signature scheme uses private key for "encryption" instead of the public key like in the standard encryption scheme).
* **Issuer** defines the fully qualified name of the Issuer of the certificate. Usually a CA does not use it's root private key for any operation, it will issue a group of `Issuer` certificates which are allowed to sign other certificates for practical considerations. The CA private keys are kept under the highest security off the network.
* **Validity** defines the time period for which this certificate is valid, beyond this, the certificate should not be trusted.
* **Subject** defines the fully qualified descriptor of the identity who is claiming to be the owner of the private part of the public key in the certificate.
* **Subject Public Key Info** defines what algorithm and respective values are representing the key. The algorithm is `rsaEncryption` which contains of a modulus and an exponent. Refer the Wiki article about RSA key pair.
* **X509v3** extensions represent what a particular certificate is allowed to do. For example
    * **X509v3 Key Usage**, a critical extension which should be strictly adhered to. It's allowed to do `Digital Signature` (allowed to sign some data digitally), `Key Encipherment` (encrypt other keys not part of the application, like in TLS)
    * **X509v3 Extended Key Usage** allows this certificate to be used as a `TLS Web Server Authentication`, which is this certificate can be used to represent a client's identity during a TLS connection
    * **X509v3 Basic Constraints**, a critical extension, says, this certificate **should not** be considered as a `CA` or a certificate authority.
    * **X509v3 Subject Key Identifier** defines identity of the private key. Certificates expire, which means, a new key pair can represent the same identity, so the private keys are differentiated by this value.
    * **Authority Key Identifier** defines the identity of the private key of the signer which vetted this certificate.
    * **Authority Information Access** tells locations on the internet where to find more information about the issuer.
        * **OCSP** or Online Certificate Status Protocol is a protocol that allows a client to check if the certificate which is presented is actually valid and not revoked
        * **CA Issuers** has the information about the issuer certificate if it's not bundled with this certificate. This can be handy to verify the certificate to the root CA
            ```bash=
            curl http://pki.goog/repo/certs/gts1c3.der -o - \
                | openssl x509 -inform DER -noout -text
            ```
    * **X509v3 Subject Alternative Name**, in case this certificate is used to serve a TLS request against a domain or DNS name, what DNS names can be represented by this certificate. So, if the DNS names contain foo.google.com and bar.google.com, the certificate will be allowed to represent those domains. But, if this certificate is presented during a request with facebook.com, the client will reject it because it's not part of the DNS names.
    * **X509v3 Certificate Policies** is a set of addition (possibly custom) policies which may be enforced on this certificate. Note that it doesn't have an "readable" name. This could signify that it's a non-standard attribute not known to the generic public client and may be ignored.
        * [`2.23.140.1.2.1`](https://oidref.com/2.23.140.1.2.1) denotes
            >Use of this OID is appropriate when the certificate lacks information about the certificate holder's legal identity.
        * `1.3.6.1.4.1.11129.2.5.3` seems to be a custom value, still part of google(11129) though.
    *  **X509v3 CRL distribution Points**, a CA may want to say that these certificate (issuers or leaf certificates) are no longer valid because of some issue other than expiry. That information is encoded in a _Certificate Revocation List_ which is published by each CA and should be checked by a client if possible to make sure it's trusting a valid certificate.
        ```bash=
        curl http://crls.pki.goog/gts1c3/fVJxbV-Ktmk.crl -o - \
            | openssl crl -inform DER -noout -text
        ```
        Notice how this literally has a list of Serial Numbers of the certs that will be no longer valid after the `Revocation Date`.

    * [`1.3.6.1.4.1.11129.2.4.2`](https://oidref.com/1.3.6.1.4.1.11129.2.4.2), Eh? what's this. This is a custom (non-standard) attribute represnted in an OID format. This OID refers to
        ```
        {
            iso(1)
            identified-organization(3)
            dod(6)
            internet(1)
            private(4)
            enterprise(1)
            google(11129)
            2(2) 4(4) 2(2)
        }
        ```
* **Signature Algorithm** this time defines what signature algorithm is used and what is it's value.
=
