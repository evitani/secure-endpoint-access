# Service Verification Pair Rotation

It is good practice for a service to rotate [Service Verification Pairs](Terms-Entities.md#service-verification-pair) whenever the SVSK is at increased risk of compromise, such as during an infrastructure migration. It can also be used to encourage client application updates by forcing the installation of the new SVPK for continued signature verification.

To support with seamless rotation of keys, SEA is designed to provide an overlap period. During this period, two Service Verification pairs are considered valid and the service must support clients who expect either to be used. This allows up to 90 days for client applications to be updated with a replacement SVPK without impacting the end user experience.

_NB: Services should not rotate keys more regularly than is necessary to ensure the security of their channels. Doing so slows down responses, increases client application management, and introduces risk by requiring regular communication of replacement keys._

## Establishing Trust
While in a rotation period, the service will receive trust establishment requests from client applications and process them as normal. However, while calculating the signature of the Verification Challenge, it will calculate signatures with both of the accepted SVSKs. The resulting strings will both be placed into the `vc_signature` array, with the newer pair's signature first.

On receipt of this response, the client application will sequentially attempt to verify each of the two signatures. If the first signature fails verification, the second will be attempted. However, if the first succeeds, the second will not be attempted. If both fail, then the normal verification failure process will apply.

## Making Trusted Requests
If a client application has detected that rotation is occurring – by counting the total elements in `vc_signature` during trust establishment – then the trusted requests it sends should include the additional attribute `expects_svpk`. This should be a hexadecimal string representing the SHA-1 hash of the SVPK the client application is using to verify requests. In the event the request has no body, the same value should be sent as a query parameter in the URL named `_svpk`. The client application should not send this value – either as part of a JSON body or as a query parameter – except when it detects that rotation is occurring.

The service, on receipt of a request containing this value, should sign its response with the SVSK that relates to the SVPK expected by the client. This process ensures the service does not have to sign each response twice, and additionally allows the service to gather data on the progress of its key rotation. If the client application requests signature with an invalid key – one that is not currently in rotation – then [trust should be destroyed](Destroying-Trust.md).

During a single encryption session, the `expects_svpk` (or `_svpk` parameter) value must never change from that sent in the first trusted request. The service should monitor for this change, and it must [destroy trust](Destroying-Trust.md) if it occurs. If a client application wishes to change the certificate it is using, it must end sessions relying on the old certificate and re-establish them with the replacement.

## Restrictions
Rotation periods – where two Service Verification pairs are considered valid – must never last longer than 90 days. Client applications that observe a rotation period lasting longer than 90 days may optionally choose not to make trusted requests to the relevant service until the period ends.

While rotating Service Verification pairs does introduce a way to limit applications accessing SEA-protected services, it should not be seen as a way to protect against attackers emulating legitimate client applications. SVPKs are considered public information by the protocol, so an attacker may have access to the updated key as soon as it is shared with legitimate applications.

Rotation should never be used to replace a key that is known to have been compromised. From the moment of compromise all responses sent by the service should be considered as insecure, so a replacement pair should be implemented immediately and without retaining the old pair. This stops client applications sending private data over a compromised channel, as trust will be destroyed during the initial handshake until the application is updated to use the new pair.

Channels secured by the SEA protocol should not be used for the distribution of their own updated SVPKs; no service should ever send a new SVPK to a client application over a channel verified by the pair it is replacing. Doing so would allow an attacker with access to a compromised SVSK to trigger a rotation and inject its own certificate into the chain. 