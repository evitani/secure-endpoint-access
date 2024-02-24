# Destroying Trust

At any time, either party may destroy a trusted relationship. When either party chooses to, or becomes aware that the other party wishes to, destroy trust, it will delete all associated keys. No keys other than the Service Verification Pair may be maintained in any form by either party.

## Service
The service will destroy trust by deleting the Session ID and all associated keys. It will do this if a trusted session exceeds its [Accepted Dormancy Period](Terms-Entities.md#accepted-dormancy-period), or if it receives an instruction from the client to destroy trust (see below). Implementations may also choose to destroy trust as a result of logic checks, such as if a user signs out during a session, or a network change is detected.

Following trust being destroyed for any reason, all future requests to a destroyed session will receive at `429 Untrusted` response. This includes in response to the destruction request as described below. The service will not differentiate between sessions that have never existed and those that have been destroyed in the way it responds to untrusted requests.

## Client
The client will destroy trust by sending a request to the service:

`DELETE /sea-trust/<Session ID>`

On sending the destruction request, the client will not wait for a response from the service, but it will delete all keys associated with the session. The client will do this when a user signs out, or if it receives a `429 Untrusted` response from the service. _NB: if the client destroys trust because of a `429 Untrusted` response, it will not send a destruction request to the service._

The client may also choose to destroy trust at key points during its workflow, such as when it detects it is entering a dormant state. Leaving trusted relationships open unnecessarily allows an attacker who gains access to the associated keys to make trusted requests, so care should be taken to close them and avoid this.

When trust has been destroyed, either by the client or detected by the client following a `429 Untrusted` response, the client should return the application to a state from which a newly trusted relationship can be created, but no such relationship should automatically be attempted.

## Attack Vectors
It is possible for a sufficiently-advanced attacker who can intercept requests between the client and the service to deliberately delay or capture a request to destroy trust. The service should therefore not rely on the client in initiating the destruction of trust, and must always implement the checks required of it.

It should be noted that an attacker who is capable of behaving in this manner does not represent a particular threat beyond maintaining a session, given the client itself will still be the only actor able to  send requests over the session. The attacker would additionally have to access the associated keys to compromise the session's security.