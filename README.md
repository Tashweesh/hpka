# HPKA 0.1, Draft spec

----------------------------------------------------

Written by the Syrian watermelon

Email : [batikhsouri@gmail.com](mailto:batikhsouri@gmail.com)

Twitter : [@BatikhSouri](https://twitter.com/BatikhSouri)

Github : [https://github.com/BatikhSouri](https://github.com/BatikhSouri)

### Introduction

HPKA is an extension of the HTTP protocol that aims to authenticate users through public key authentication.

It has some features that are useful when you want to run a distributed, federated network.

It would allow adhoc user authentication, registration, backup (TODO), server transfer (TODO) and complete deletion.

### Technical overview

On each HTTP request, the client appends some headers :

* HPKA-Req: all the details about the action type, username, public key, as described by the protocol below.
* HPKA-Signature: the signature of the HPKA-Req field content

We should note that this solution as it is now is not safe from MITM attacks when not used over HTTPS. Except if the server keeps a PubKey blob and respective signature history for each user for a rather large period of time (1 month for example). Logging the signatures in that case would be enough, because the payload should stay constant.

If the headers mentioned above are not present in the HTTP request, then add a "HPKA-Available: 1" header when responding to the client.

If some error occured or some mistake was made in the request, the reponse will have it's status code == 445. In addition to that, it will also carry an additional "HPKA-Error" header; it's value will be just an error number according to the HPKA-Error protocol described below

### Security overview

A similar system (client pub key auth) has been implemented through TLS/SSL client certificates. However, the authentication there is done on TLS/SSL level and not HTTP. Furthermore, client certificates are delivered by the server/service and are signed by a CA on delivery. That last point means that you should be trusting the CA.

The difference here with HPKA is that the users generates his own key pair and signs his public key with his private key on registration. Then he appends dedicated HTTP headers on each request, each time with a new signature.

Furthermore, this technique brings some advantages over usual username/password authentication methods. For example, if a website using this method is "hacked", the hackers can't hack all users at once by dumping a passwords DB. An other example is that it is probably much harder to hack a specific user account because you would have to compromise the user's device first.

For improved security, the user's key file shoud use a passphrase, so in case the user's device is compromised in any way, the account would be compromised if and only if the attacker was able to decrypt the key file through an expensive password attack (assuming the user wouldn't give him the password...)

Note that the this module, as of now, uses the NIST-designed curves for ECDSA signatures. We are aware of the Dual-EC-DRBG NSA backdoor. And some specialists think that there is a good probability that these NIST-designed, NSA-approved curves may be backdoored as well. Aside these, Curve25519 and Ed25519, there aren't lots of other curves used broadly. I plan to extend HPKA for Ed25519/NaCl signatures in upcoming verisons.

**Side note:**  
Further on, I'll say in the small threat model below that a service using HPKA should be hosted somehow securely (HSTS or Tor hidden service). I know there is a contradiction between HSTS and the fact we maybe shouldn't trust CAs as much as we do. But, we can actually do without them in our case : a user will call the same server many times, so certificate pinning of a self-signed cert should be enough. Otherwise, for first time users there isn't a way as much accepted/used as TLS/SSL for authenticating a server.

### Threat model

We describe here our assumptions about the user's computer, and what an attacker can achieve :

* The user acts reasonably. He would not give the password of his key file to an attacker for example
* The user's computer
	* Uses a propeerly implemented HPKA client
	* Is not infected by malware
* The service uses [HSTS](http://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) or a [Tor hidden service](https://www.torproject.org/docs/hidden-services). Equivalently, it should be nearly "impossible" to eavesdrop on the connection between the server and the client
* We assume that the security level provided [DSA](http://en.wikipedia.org/wiki/Digital_Signature_Algorithm), [RSA](https://en.wikipedia.org/wiki/RSA_(algorithm\)) and [ECDSA](https://en.wikipedia.org/wiki/ECDSA) signature schemes is valid. Also we assume that the [most common, non-Koblitz curves](http://www.secg.org/collateral/sec2_final.pdf) are safe.
* Assumptions about the server:
	* The server has HPKA prorperly implemented
	* The server can refuse a new user registration (attacker could guess usernames then)

### HPKA-Req protocol

__NOTES :__

* Everything is stored in Big Endian
* No encoding is used for key's elements

__The Req payload is constructed as follows :__

* Version number : one byte. Only value for now: 0x01
* UTC Unix Epoch (timestamp since 1-1-1970 00:00:00 UTC, in seconds) (8 bytes long)
* Username.length (one byte)
* Username
* ActionType (one byte)
* Key type : one byte. Values possible are:
	* 0x01 for ECDSA
	* 0x02 for RSA
	* 0x04 for DSA
* Then, depending on the key type
	* If keyType == ECDSA (== 0x01)
		* publicPoint.x.length (unsigned 16-bit integer)
		* publicPoint.x
		* publicPoint.y.length (unsigned 16-bit integer)
		* publicPoint.y
		* curveID (one byte)
	* If keyType == RSA (== 0x02)
		* modulus.length (unsigned 16-bit integer)
		* modulus
		* publicExponent.length (unsigned 16-bit integer)
		* publicExponent
	* If keyType == DSA (== 0x04)
		* primeField.length (unsigned 16-bit integer)
		* primeField
		* divider.length (unsigned 16-bit integer)
		* divider
		* base.length (unsigned 16-bit integer)
		* base
		* publicElement.length (unsigned 16-bit integer)
		* publicElement
* Append 10 random bytes

Finally, the built payload is Base64 encoded (because of how HTTP is built, it should be without line breaks). After encoding, this blob is signed by the user's private key (corresponding to the public key info in the blob obviously)

__CurveID :__  
Here are the possible values for the curveID field, and to what curve they correspond

 CurveID | Curve name
-------- | -----------
 0x01    | secp112r1
 0x02    | secp112r2
 0x03    | secp128r1
 0x04    | secp128r2
 0x05    | secp160r1
 0x06    | secp160r2
 0x07    | secp160k1
 0x08    | secp192r1
 0x09    | secp192k1
 0x0A    | secp224r1
 0x0B    | secp224k1
 0x0C    | secp256r1
 0x0D    | secp256k1
 0x0E    | secp384r1
 0x0F    | secp521r1
 0x80    | sect113r1
 0x81    | sect113r2
 0x82    | sect131r1
 0x83    | sect131r2
 0x84    | sect163r1
 0x85    | sect163r2
 0x86    | sect163k1
 0x87    | sect193r1
 0x88    | sect193r2
 0x89    | sect233r1
 0x8A    | sect233k1
 0x8B    | sect239r1
 0x8C    | sect283r1
 0x8D    | sect283k1
 0x8E    | sect409r1
 0x8F    | sect409k1
 0x90    | sect571r1
 0x91    | sect571k1
 
__ActionType :__

Here are the possible values for the ActionType field, depending on the type of the actual request.

Value | Meaning
------|--------
0x00  | Normal (authenticated) HTTP request
0x01  | Registration
0x02  | Archive download/backup
0x03  | Transfer/Delete
0x04  | Restore/TimeMachine

### HPKA-Error protocol

Here is the different error numbers for the HPKA-Error header, in case some error occured

Value | Meaning
------|---------
0x01  | Malformed request
0x02  | Invalid signature
0x03  | Invalid key
0x04  | Non registered user
0x05  | Username not available (on registration)
0x06  | Forbidden action
0x07  | Unsupported action type
0x08  | Unknown action type
 
### HPKA User registration

When a user wants to register on the website using his public key, he appends the HPKA-Req (with ActionType == 0x01) and HPKA-Signature fields on a GET request on the website's home page. If the username is available, the server registers it and responds to the user with a "normal" reponse (status code = 200). Otherwise it will return status code 406, with a HPKA-Error: 5

### HPKA User data download/backup

The user can download the data hosted on the service (FB/Twitter data archive-like system)

The is done through the following steps:

* The client sends an HPKA authentified GET request on the website's home page. Here are the appended headers
	* HPKA-Req and HPKA-Signature
* The server's response will contain "HPKA-BuildArchive: 1" in case it has been accepted
* The user will continue it's usage of the website (or not) while the his data archive is built
* On next requests, if the archive is ready, the server will append "HPKA-ArchiveReady: 1" starting from the moment when the archive is ready. It will also append "HPKA-ArchiveLink" header which value is the link to download the archive. The download shold be prohibited if the standard HPKA auth fields are not appended. Once the archive has been downloaded by the user, the archive should be deleted and the link not valid anymore.

### HPKA User account transfer/delete protocol

The HPKA transfer process adds one additional step server side: the deletion of the user content after the archive download.

The process here ressembles to the one described above. The bootstrap is the only difference is that the first request is appended by "HPKA-Transfer: 1"

### HPKA User account restore/upload protocol

This process complements the HPKA transfer/deletion protocol. You can use it to transfer your account to an other server, or restore your account to some point in time (which can be useful, depending on what kind of service you deal with). The process goes as follows:

* The user sends a POST request on the server's home page, containing an "archive" field (where it is a simple HTTP upload). The request should contain