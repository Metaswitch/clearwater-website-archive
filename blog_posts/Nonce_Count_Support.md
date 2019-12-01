Nonce Count Support
-------------------
As you may have seen if you follow the Project Clearwater blog, the Articuno release added support for nonce counts. This allows IMS user equipment (UEs) to pre-calculate authentication responses when re-registering with Clearwater, which permits more efficient re-registration flows. This blog post explains nonce counts in a bit more detail, what the benefits are, and what you should be aware of enabling this feature on your deployment.

# Authentication without Nonce Counts

Authentication is the process by which a user proves they are who they say they are. For the purposes of authentication in IMS, a user is identified by their IMS private identity (IMPI). When their UE registers with the network it includes the user’s IMPI and needs to prove that the user owns this IMPI. The two most common authentication schemes are SIP Digest and AKA both of which make use of a shared secret that is accessible to the network and the UE. For SIP Digest this is the user’s password; for AKA it is a numeric code (which in the case of VoLTE is securely burned into the user’s SIM card). In order for authentication to be successful, the UE needs to prove it has access to the shared secret. It can’t just transmit the secret to the core network in plain text as it would be easy for an eavesdropper to observe the registration traffic and learn the secret. Both authentication schemes are based on HTTP digest authentication (defined in [RFC 2617](https://tools.ietf.org/html/rfc2617)) which defines a challenge/response procedure:

*   When the UE tries to register, the network generates a challenge. This includes at least some random data called the **nonce**.
*   The UE then runs an algorithm to compute a response to the challenge. The algorithm takes the shared secret and the challenge as inputs (and possibly extra information such as pieces of the underlying SIP message – depending on the algorithm). The UE transmits the authentication response to the network.
*   The network runs the same algorithm and generates an expected response. If this matches the response from the UE, the network treats the request as authenticated.

Importantly the algorithm to generate the response is one-way so it is impossible for an eavesdropper to extract the shared secret from the response (at least with current technology). In addition the network must discard the nonce as soon as the UE successfully authenticates to protect against [replay attacks](https://en.wikipedia.org/wiki/Replay_attack) copying an earlier authentication response. This challenge/response scheme is mapped to the following four SIP messages:

*   Initial REGISTER. The UE may explicitly include the users IMPI in the username field of the Authorization header. If it doesn’t the IMPI is derived from the public identity (IMPU) being registered (by stripping of the URI scheme, so the IMPU sip:alice@example.com becomes the IMPI alice@example.com).
*   401 Unauthorized. The network rejects the REGISTER messages with a 401 status code. This message contains an authentication challenge in the WWW-Authenticate header.
*   Authenticated REGISTER. The device calculates its authentication response and resubmits a REGISTER. The authentication response is placed in the Authorization header.
*   REGISTER response. The network checks the authentication response on the REGISTER. If it is incorrect, it rejects the REGISTER with a 403 Forbidden. If the authentication response is correct, it does further REGISTER processing, which hopefully succeeds and results in a 200 OK.

# Authentication with Nonce Counts

Although this scheme works it is not very efficient. Firstly it needs two REGISTER/response flows instead of just one. Secondly, generating an authentication challenge may be expensive for the network - each challenge may involve a request to the HSS, which is often a performance bottleneck. Fortunately RFC 2617 allows UEs to keep on using the same challenge and respond to it multiple times. In order to keep this secure against replay attacks, both the UE and the network track a “nonce count” which is a count of the number of times the challenge has been used to generate a response. The first response uses a nonce count of 1, the second response a nonce-count of 2, etc. The UE’s current view of the nonce count is included in the SIP REGISTER alongside the authentication response. If the network receives a response with a nonce count that has already been used it discards the response. This means that if an eavesdropper tried a replay attack it would fail because the nonce count would be too low. They can’t just arbitrarily increase the nonce count either (for example by replaying an earlier response that had nonce count 2 but setting the nonce count to 3). This is because the nonce count is an input the algorithm that calculates the response. If the eavesdropper just increased the nonce count (without updating the response) the response would now be incorrect and the network would reject it. They can’t work out the correct response either as the algorithm to calculate it requires access to the shared secret. Clearwater now supports nonce counts allowing for more efficient re-registration flows, and less load on the HSS. However there are some restrictions you should be aware of if you’re using Clearwater without certain IMS components.

# Restrictions in non-IMS Topologies

Clearwater is a flexible platform and can be deployed without certain elements normally present in an IMS network. Specifically it can run without an HSS (in which case users are provisioned directly in Homestead) and without an I-CSCF. Nonce count support has some restrictions if you enable it without one or both of these network elements.

## No HSS

Occasionally someone other than the intended user may gain access to their shared secret. Perhaps a 3rd party steals their phone (along with their SIM card), or learns their SIP Digest password. This 3rd party can now register as the user, which allows them to answer the user’s calls or make calls using their account. When this happen as operator needs to be able to put a new shared secret in place and kick the 3rd party out of the network. This is not a significant problem if nonce counts are not in use. The 3rd party UE will eventually re-register, which is treated as a completely fresh register. This causes a fresh challenge to be generated based on the current shared secret, which the 3rd party does not know. However when nonce counts _are_ in use the same challenge can be used over and over again (which is the whole point of nonce counts). Fortunately IMS defines the concept of “administrative deregistration”. The operator can use their HSS to send a Registration Termination request to the S-CSCF. This deregisters some/all UEs that have registered along with any challenges stored by the S-CSCF. This causes the UEs to have to register from scratch, which the 3rd party cannot do as they don’t know the new secret. When deployed with an external HSS, Clearwater supports administrative termination. When it is deployed _without_ an HSS (so when subscribers are provisioned directly on Homestead) administrative deregistration is not possible so if nonce count support is enabled there is no way to force a UE to become de-registered should a shared secret be learned by a 3rd party.

## No I-CSCF

So far we’ve only talked about authentication, which is proving that originator of a request owns the specified IMPI. Another aspect of IMS security is Authorization, which is the question “is this IMPI allowed to register for this IMPU?” Alice may be able to prove she owns the IMPI alice@example.com, but she probably shouldn’t be able to register for [sip:bob@example.com](bob@example.com). Authorization occurs at two points in an IMS core:

*   When a REGISTER flows through the I-CSCF, it sends a User-Authorization request to the HSS. This checks that the IMPI is allowed to register for the IMPU. The I-CSCF rejects the REGISTER if not.
*   When the REGISTER reaches the S-CSCF it sends a Multimedia-Auth Request to the HSS. The main purpose of this is to download information needed to send an authentication challenge, but it also causes the HSS to do another authorization check.

The whole point of nonce counts is to eliminate the need for the second bullet during re-registration flows. However the first authorization check should always happen, so there is no way for Alice to register as Bob. However if the deployment does not use an I-CSCF Alice could initially register as herself, and then re-register as Bob (since when nonce count support is enabled, no authorization checks are performed on the re-register). Clearwater can act as a combined S-CSCF and I-CSCF, and in this case there is no security hole.

# Clearwater Configuration

Clearwater nonce count support is off by default as this is secure in all deployment topologies. However if you want to enable it, simply:

* Set nonce\_count\_supported=Y in /etc/clearwater/shared\_config

* Run /usr/share/clearwater/clearwater-config-manager/scripts/upload\_shared\_config

Enabling nonce count support will reduce the load placed on Clearwater and your HSS by re-registration traffic. It may also make it easier to interoperate with certain SIP phones or test scripts.

# Further Reading

*   Read about the IMS data model in [its](Secrets_1.md) [full](Secrets_2.md) [glory](Secrets_3.md).
*   [RFC 2617](https://tools.ietf.org/html/rfc2617) – the scheme on which IMS authentication is based.
*   For a full explanation of AKA, take a look at [RFC 3310](https://tools.ietf.org/html/rfc3310)
