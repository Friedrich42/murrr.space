---
title: "How to integrate SmartID, MobileID and ID card to your python project"
date: 2022-05-08T20:55:23+03:00
lastmod: 2022-05-08T20:55:26+03:00
slug: "estonian-id-systems-integration"
draft: false
author: "Murr Kyuri"
categories:
- Estonian Identity cards
tags:
- id.ee
- SmartID
- MobileID
- authentication
- web-eid
- python
description: "A comprehensive guide of integration of Eesti (Estonian) identity systems to your python application with code examples"
---

## Abstract

Recently I had a chance to work with Estonian identification systems. It was an interesting experience that I'm willing to share.

Estonian id systems provide examples in java, .net, and php, but neither of these languages collide with my tech stack, so I decided to use python.

We will not talk about digital signing in this article, but instead focus on authentication with Web eID.

You can think of SmartID or MobileID systems as firebase, but much more complex one.


## SmartID

SmartID is a system that doesn't depend on your ID card or phone number.
It has an application that client installs on his/her device and verifies the code, entering PIN1.

From a developer perspective the process consists of three stages:
* [Initial request from backend to SmartID servers to create a session](#session-creation)
* User confirmation of code in the SmartID app.
* [Polling from backend to get result](#session-polling)

Terminology:
* ID code is a national identification number, which is also called social security number.
* SmartID servers or relying party are servers that your backend application will send requests to.

### Session creation

To make this request first thing you need is to ask his country and ID code.

The second thing you need is a cryptographically safe random value.
You need to compute SHA hash of the value and then encode the result to base64.
You can use SHA 256, 384 or 512. I'll use 256 in these examples.

The final pseudocode function looks like `base64encode(sha256(random()))`.

You can refer to [documentation](https://github.com/SK-EID/smart-id-documentation#23121-sending-authentication-request) for more information.
```python
import os
import hashlib

random_bytes = os.urandom(64)
random_sha256 = hashlib.sha256(random_bytes).digest()
random_data = base64.b64encode(random_sha256).decode()
```

Second thing you need is relying party information. I'll use DEMO and empty UUID value for demo purposes.
You can easily work with demo environment, so it is easier to test everything before release, but you will need to acquire real values for production use.
You can refer to [documentation](https://github.com/SK-EID/smart-id-documentation#22-relying-party-rest-interface) for more information.

SmartID uses allowedInteractionsOrder parameter to list interactions that should be available to client.
Not all app versions can support all interactions though. The Smart-ID server is aware of which app installations support which interactions. When processing an RP request the first interaction supported by the app is taken from allowedInteractionsOrder list and sent to client.
For more details refer to [documentation](https://github.com/SK-EID/smart-id-documentation#237-allowed-interactions-order)

Finally after collecting all the data, you have the body of your request to the relying party.

```python
request_verification_code_data = {
    'certificateLevel': 'QUALIFIED',
    'hashType': 'SHA256',
    'allowedInteractionsOrder': [
        {
            'type': 'verificationCodeChoice',
            'displayText60': 'Choose verification code'
        },
        {
            'type': 'displayTextAndPIN',
            'displayText60': 'Display text and pin'
        }
    ],
    'relyingPartyUUID': 'DEMO',
    'relyingPartyName': '00000000-0000-0000-0000-000000000000',
    'hash': random_data,
}
```

Send a POST request to https://sid.demo.sk.ee/smart-id-rp/v2, for demo purposes. You will need a production URL for live system.


The final request should look like:
```python
import requests

res = requests.post(
    url=f'https://sid.demo.sk.ee/smart-id-rp/v2/authentication/etsi/PNO{country}-{id_code}',
    json=request_verification_code_data,
)
```

You should receive 200 status code and sessionID key in response [if everything was done correctly](https://github.com/SK-EID/smart-id-documentation#233-http-status-code-usage.

example response should look like:
```json
{
  "sessionID": "de305d54-75b4-431b-adb2-eb6b9e546014"
}
```

You need to store that sessionID in some kind of database along with user ID code.

Also you need to return a verification code to user, which is computed from random value that you generated in the beginning.

```python
def get_verification_code(hash_value) -> str:
    """Compute verification code from a hash

    Verification Code is computed with: `integer(SHA256(hash)[-2:-1]) mod 10000`

    1. Take SHA256 result of hash_value
    2. Extract 2 rightmost bytes from it
    3. Interpret them as a big-endian unsigned short
    4. Take the last 4 digits in decimal

    Note: SHA256 is always used, i.e. the algorithm used when generating the hash does not matter

    based on https://github.com/SK-EID/smart-id-documentation#612-computing-the-verification-code
    """

    digest = hashlib.sha256(hash_value).digest()
    raw_value = struct.unpack(">H", digest[-2:])[0] % 10000

    return f"{raw_value:04}"

# SEND TO USER
vc = get_verification_code(random_sha256)
```

### Session polling

After you sent the verification code to the user, he needs to confirm it in the SmartID application.

Relying party server will return a certificate, which you need to parse and verify.

```python
from asn1crypto.x509 import Certificate as Asn1CryptoCertificate
from oscrypto._openssl.asymmetric import Certificate as OsCryptoCertificate, load_certificate


class CertificateHolderInfo:
    def __init__(self, given_name, surname, id_code, country, asn1_certificate):
        self.given_name: str = given_name
        self.surname: str = surname
        self.id_code: str = id_code
        self.country: str = country
        self.asn1_certificate: "Asn1CryptoCertificate" = asn1_certificate

    def __str__(self):
        return f'{self.given_name=}\n{self.surname=}\n{self.id_code=}\n{self.country=}\n{self.asn1_certificate=}\n'

    @classmethod
    def from_certificate(cls, cert: "Union[bytes, Asn1CryptoCertificate, OsCryptoCertificate]"):
        """
        Gets personal info from an oscrypto/asn1crypto Certificate object

        For a closer look at where the attributes come from:
        asn1crypto.x509.NameType
        """
        if isinstance(cert, bytes):
            cert = load_certificate(cert)
        cert: "Asn1CryptoCertificate" = getattr(cert, 'asn1', cert)
        subject = cert.subject.native

        # ID codes usually given as PNO{EE,LT,LV}-XXXXXX.
        # LV ID codes contain a dash so we need to be careful about it.
        id_code = subject['serial_number']
        if id_code.startswith('PNO'):
            prefix, id_code = id_code.split('-', 1)  # pylint: disable=unused-variable

        return cls(
            country=subject['country_name'],
            id_code=id_code,
            given_name=subject['given_name'],
            surname=subject['surname'],
            asn1_certificate=cert,
        )
```

Overall polling process looks like:
```python
for _ in range(10):
    res = requests.get(url=f"https://sid.demo.sk.ee/smart-id-rp/v2/authentication/session/{session_id}", params={'timeoutMs': 10000})
    if res.status_code != 200:
        time.sleep(2)
        continue

    if res.json()['state'] == 'COMPLETE' and res.json()['result'] != 'OK':
        raise RuntimeError(res.json()['result'])
    else:
        break

cert_value = base64.b64decode(res.json()['cert'])

user_data = CertificateHolderInfo.from_certificate(cert_value)
```


## Reference
* [Web eID](https://www.id.ee/en/article/web-eid/)
* [Web eID architecture documentation](https://github.com/web-eid/web-eid-system-architecture-doc)
* [SmartID documentation](https://github.com/SK-EID/smart-id-documentation)
