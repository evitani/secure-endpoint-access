# Making Trusted Requests

Once trust is established, the client may make trusted requests to the service. Depending on which methods are supported by the service, the process differs. The client will only attempt to make requests using a method the service exposed during the initial handshake.

## Content Protection Requests
Content Protection means that any content inside a request and response body will be protected by SEA. However, content included in the request metadata, such as specific URLs and query parameters, will not be protected.

To make a trusted request using Content Protection, the client will make the request to the usual URL. If the request has a body, this body will be encoded as JSON, and nested in a parent JSON object:

```json
{
  "body": <original request body>,
  "tvv": "<Timed Verification Value>"
}
```

The [Timed Verification Value](Terms-Entities.md) for the current time is calculated by the client ([see details below](#calculating-the-timed-verification-value)), and the resulting JSON is encrypted with AES-256-CBC using the Traffic Encryption Key to create an encrypted body.

The request that is then sent by the client will include a JSON-encoded body as follows:
```json
{  
  "cid": "<Client ID>",
  "sid": "<Session ID>",
  "enc": "<encrypted body>"
}
```
An additional JSON attribute, `expects_svpk`, will be included in specific circumstances. See [Service Verification Pair Rotation](Service-Verification-Pair-Rotation.md) for information on its calculation and use.

## Encapsulation Requests
Encapsulation ensures that metadata related to a request is also protected by SEA. As such, requests require additional preprocessing. As with Content Protection requests, the body must be encoded as JSON, and nested into a parent JSON object:
```json
{
  "body": <original request body>,
  "destination": "<destination>",
  "args": {<url arguments>},
  "method": "<method>",
  "tvv": "<Timed Verification Value>"
}
```
The `destination` parameter should be a string representing the original destination of the request. The service will use this to route the request correctly after decrypting it, so the format of the destination will be defined by the service based on its internal logic. 

The `args` parameter should be an object containing the URL arguments for the original request, or an empty object if no arguments are present.

The `method` parameter should be a string containing either `GET`, `POST`, `PUT`, or `DELETE`, representing the method for the request.

This JSON object will be encrypted by the client with AES-256-CBC using the Traffic Encryption Key to create the encrypted request, and a request will be sent to:

`POST /sea-encapsulated`

The request will have a JSON-encoded body as follows:

```json
{
  "cid": "<Client ID>",
  "sid": "<Session ID>",
  "enc": "<encrypted request>"
}
```
An additional JSON parameter, `expects_svpk`, will be included in specific circumstances. See [Service Verification Pair Rotation](Service-Verification-Pair-Rotation.md) for information on this parameter's use.

## Handling the Request
The service, on receipt of a request, will first decrypt the `enc` content using the Traffic Encryption Key to recover the plain request. It will validate the included Timed Verification Value calculating it for itself and comparing the outputs. If the two do not match, the service may 'look back' one iteration by removing one from the integer used in the TVV calculation. If the Timed Verification Value still cannot be validated, [trust is destroyed](Destroying-Trust.md).

Should the validation succeed, the request will be processed as required by the service. The response to the client will be JSON-encoded, and nested into a parent JSON object:

```json
{
  "body": <original request body>,
  "tvv": "<Timed Verification Value>"
}
```
The Timed Verification Value should be newly-calculated, not re-used from the inbound request. This JSON body will then be encrypted with AES-256-CBC using the Traffic Encryption Key. A signature of the encrypted response will be created using the SVSK and represented as a base64-encoded string, and the results sent in a JSON-encoded body:

```json
{
  "enc": "<encrypted response>",
  "enc_signature": "<encrypted response signature>"
}
```

## Handling the Response
On receipt of a response, the client will first verify the signature matches the encrypted response string using the SVPK. If verification fails, the client will [destroy trust](Destroying-Trust.md). If it succeeds, the `enc` value will be decrypted using the Traffic Encryption Key. If this decryption does not result in a correctly-formatted JSON object, then trust will additionally be destroyed.

If the content is decrypted successfully, the Timed Verification Value will be verified by the client. If the two do not match, the client may 'look back' one iteration by removing one from the integer used in the TVV calculation. If the Timed Verification Value still cannot be validated, trust is destroyed.

If the TVV is validated, the body of the response is considered and can be processed by the client as required.

## Calculating the Timed Verification Value
The Timed Verification Value is generated by first calculating an integer that represents the current time in seconds since 1970-01-01T00:00:00, divided by 30. If the service is generating the Timed Verification Value, it should apply the Time Offset to the time prior to division. This integer is then appended to the Timed Verification Key, and SHA-1 hash of the resulting string is calculated. This hash, represented as a hexadecimal string, is the Timed Verification Value.

## Errors
If the service fails to decode a valid JSON object following decryption, or if it succeeds but cannot validate the Timed Verification Value, it will destroy trust and therefore return a `429 Untrusted` response.

In addition, the service will return a `501 Not Implemented` response should the client make a request using a method not supported, such as a Content Protection request to a service that only accepts Encapsulation requests.