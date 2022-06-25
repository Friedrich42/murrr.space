---
title: "How to integrate MobileID to your python project"
date: 2022-06-25T16:00:08+03:00
lastmod: 2022-06-25T16:00:13+03:00
slug: "mobile-id-system-integration"
draft: false
author: "Murr Kyuri"
categories:
- Estonian Identity Systems
tags:
- id.ee
- MobileID
- authentication
- web-eid
- python
description: "Guide to integration MobileID system to your python project"
---

## Abstract

This is the second article of series Estonian Identity Systems. Here we will talk about MobileID service integration.

Estonian id systems provide examples in java, .net, and php, but none of these languages collide with my tech stack, so I decided to use python.

We will not talk about digital signing in this article but instead, focus on authentication with Web eID.

MobileID system is very alike to SmartID from developer's perspective.
You might want to read the first article of the series 
["How to integrate SmartID to your python project"](/posts/smart-id-system-integration) if you haven't done it yet.


## MobileID

Mobile-ID allows people to use a mobile phone as a form of secure digital ID. Like the ID-card, it can be used to access secure e-services and digitally sign documents, but has the added advantage of not requiring a card reader.

The system is based on a special mobile SIM card, which the customer must request from a mobile phone operator. Private keys are stored on the mobile SIM card along with a small application supporting the authentication and signature functions.

From a developer's perspective the process consists of three stages:
* [Initial request from backend to MobileID servers to create a session](#session-creation)
* User confirmation of code.
* [Polling from backend to get result](#session-polling)

Terminology:
* ID code is a national identification number, which is also called social security number.
* MobileID servers(or relying party) are servers that your backend application will send requests to.

### Session creation

To make this request first thing you need is to ask user's mobile number and ID code.

Also, you need cryptographically safe random value.
You need to compute the SHA hash of the value and then encode the result to base64.
You can use SHA 256, 384, or 512. I'll use 256 in these examples.

The final pseudocode looks like `base64encode(sha256(random()))`.
You can refer to [documentation](https://github.com/SK-EID/MID#32-initiating-signing-and-authentication) for more information.

```python
import os
import hashlib

random_bytes = os.urandom(64)
random_sha256 = hashlib.sha256(random_bytes).digest()
random_data = base64.b64encode(random_sha256).decode()
```

The next thing you need is relying party information. I'll use DEMO and empty UUID value for demo purposes.
You can easily work with a demo environment, so it is easier to test everything before release, but you will need to acquire real values for production use.
You can refer to [SKID solutions MobileID page](https://www.skidsolutions.eu/en/services/mobile-id/) for more information.

MobileID For more details refer to [documentation](https://github.com/SK-EID/smart-id-documentation#237-allowed-interactions-order)

Finally, after collecting all the data, you have the body of your request to the relying party.

```python
{
    "nationalIdentityNumber": id_code,
    "phoneNumber": phone_number,
    "hashType": "sha256",
    "hash": random_data,
    "language": "ENG",
    # Casting to str to ensure translations are resolved
    "displayText": "a teeny-tiny message",  # NOTE: hard 20-char limit
    "displayTextFormat": "UCS-2",  # the other choice is GSM-7 which is 7-bit
    'relyingPartyUUID': '00000000-0000-0000-0000-000000000000',
    'relyingPartyName': 'DEMO',
}
```

You should send a POST request to https://tsp.demo.sk.ee/mid-api/authentication, for demo purposes. You will need a production URL for the live system.


The final request should look like this:
```python
import requests

res = requests.post(
    url=f'https://tsp.demo.sk.ee/mid-api/authentication',
    json=request_verification_code_data,
)
```

You should receive `200 OK` status code and a sessionID key in response [if everything was done correctly](https://github.com/SK-EID/MID#326-error-conditions).

example response should look like this:
```json
{
  "sessionID": "de305d54-75b4-431b-adb2-eb6b9e546014"
}
```

You need to store that sessionID in some kind of database along with the user ID code. I would suggest using something like Redis or Memcached because these values have TTL.

Also, you need to return a verification code to the user, which is computed from a random value that you generated in the beginning.

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

# SEND vc TO USER
# Usually sent to frontend where it is shown to user
vc = get_verification_code(random_sha256)
```

### Session polling

After you sent the verification code to the user, they need to confirm it.

Relying party server will return a certificate, which you need to parse and verify.

Here is a code example of how to parse a certificate:

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

Overall polling process looks like this:
```python
import pyasice


for i in range(10):
    res = r.get(url=f"https://tsp.demo.sk.ee/mid-api/authentication/session/{sessID}", params={'timeoutMs': 10000})
    if res.json()['state'] == 'COMPLETE' and res2.json()['result'] != 'OK':
        raise Exception(res2.json()['result'])
    else:
        break

data = res.json()

# Verify signature validity
cert_value = base64.b64decode(data["cert"])
signature_value = base64.b64decode(data["signature"]["value"])

try:
    pyasice.verify(cert_value, signature_value, hash_value, 'sha256', prehashed=True)
except pyasice.SignatureVerificationError as e:
    raise e

user_data = CertificateHolderInfo.from_certificate(cert_value)
```

## Conclusion

This was the second article about Estonian Identity Systems. In the next article we'll talk about ID card integration.
Please, leave comments in the discussion section if you have any constructive criticism.

## Reference
* [Web eID](https://www.id.ee/en/article/web-eid/)
* [Web eID architecture documentation](https://github.com/web-eid/web-eid-system-architecture-doc)
* [MobileID interfacing](https://www.id.ee/en/article/how-to-link-mobile-id-to-your-information-system/)
* [django-esteid project](https://github.com/thorgate/django-esteid)
* [MobileID documentation](https://github.com/SK-EID/MID)
