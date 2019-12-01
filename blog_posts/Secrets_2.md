Secrets of the IMS Data Model - Passing associations around the network
--------------------------------------------------------------------------------
_This is the second post in a three-part series on the IMS data model – see [here](Secrets_1.md) and [here](Secrets_3.md) for the other entries._

# **Passing associations around the network**

In the first blog post in this series, we looked at the enormously rich, flexible and complex data model available in IMS, and the possibilities and concepts it offers for associating identities together. But we didn’t cover the technical nuts and bolts needed to make these associations work – that is, when the Smith family sign up for an account and their associations are provisioned into the HSS, how does the data then get communicated to the network elements providing call service?   In this blog post, I’ll try and answer that question, by looking at how the S-CSCF, the application servers, and the UEs learn about these associations.

### S-CSCF

The main way an S-CSCF learns these associations is through the Server-Assignment-Request/Server-Assignment-Answer flow, which happens when a subscriber registers (and also at other times, like a call to an unregistered subscriber). If a subscriber's associations subsequently change, though, the HSS can subsequently notify the S-CSCF through a Push-Profile-Request or a Registration-Termination-Request – these mechanisms are covered in the next blog post in this series.

#### Server-Assignment-Request

On the Server-Assignment-Request, the S-CSCF provides the Public-Identity element, containing the public identity that was explicitly registered. On the Server-Assignment-Response, the HSS provides the User-Data element, an XML block which includes:

*   all the public identities that are implicitly registered alongside that public identity
*   the service profiles for each of those public identities
*   optionally, some alias indications, indicating which public identities are aliases of each other. If this is not provided, the S-CSCF assumes that all identities sharing a service profile are aliases.

The HSS also provides the User-Name element (containing one private identity associated with the public identity) and can also optionally provide the Associated-Identities element (containing all private identities in the IMS Subscription). Going back to the five ways public identities can be associated, then:

*   an S-CSCF never learns all the other public identities in the same IMS subscription. (This doesn't matter for call service, though, because the subscription is a commercial/administrative grouping.) However, it can (optionally) learn all the private identities in the IMS Subscription, which is useful for handling PPRs and RTRs.
*   an S-CSCF always learns which identities are implicitly registered together
*   an S-CSCF always learns which identities share a service profile
*   an S-CSCF has no way to learn the full set of public identities which share a private identity with a given public identity. Going back to the example of the Smith family, when Alice registers her work mobile number, the S-CSCF can’t learn that Alice’s private identity can also register her landline. (In practice, this doesn't matter.)

### UEs

A UE only learns which identities are implicitly registered together - if a REGISTER has caused other identities to be implicitly registered, those other identities are returned in the P-Associated-URI header on the 200 OK response. UEs can also subscribe to the reg-event package, so that if the implicit registration set changes after the REGISTER flow, then the S-CSCF will send a NOTIFY message (in the reg-event package) to alert the UE to this change. (The P-CSCF sees REGISTER responses and can also subscribe to reg-event NOTIFYs, so can learn the same information from these - so it can let calls from an implicitly registered identity through, for example, rather than treating it as an attempt to steal service.)

### AS

An application server has two ways to learn about identity associations:

*   the third-party registration mechanism means that, for each service profile in an implicit registration set, a REGISTER message will be sent to the AS. This allows it to learn the same information that UEs can learn.
*   the Sh interface to the HSS allows an AS to query various data about a subscriber - including their associated identities - by sending a User-Data-Request with the IMSPublicIdentity Data-Reference.

On the Sh interface, the AS provides a public identity to identify a subscriber, and can request four groups of identities relating to the given public identity:

*   IMPLICIT\_IDENTITIES - all identities in the same implicit registration set as the given public identity
*   ALIAS\_IDENTITIES - all identities which are aliases of the given public identity
*   ALL\_IDENTITIES - which provides all public identities which share a private identity with the given public identity
*   REGISTERED\_IDENTITIES - which is the same as ALL\_IDENTITIES, except that it excludes any public identities which aren't registered

A User-Data-Request can also ask for all private identities that a public identity is associated with, by requesting the IMSPrivateUserIdentity Data-Reference instead of IMSPublicIdentity. App servers can also subscribe to updates by sending a Subscribe-Notifications-Request, which means they'll always have an accurate view if the identity sets change. Overall, this means that:

*   an AS never learns all the other identities in the same IMS subscription. As with the S-CSCF, this doesn't matter for call service, because the subscription is a commercial/administrative grouping.
*   an AS can learn which identities are implicitly registered together, either through third-party registration or through a Sh request.
*   an AS has no way to learn which identities share a service profile
*   an AS can learn which identities are aliases through a Sh request.
*   an AS can learn the full set of public identities which share a private identity with a given public identity, through an Sh request.

## References/further reading

Interested readers may want to read the 3GPP specs on this themselves, whether to check the detail of a particular point, check my reasoning, or to find more information on related topics. In this section, I've tried to give a brief overview of what the relevant sections of each specs are, and where you should look for the definitive information on particular topics:

*   For the structure of User-Data, see TS 29.228 appendix B.2, and also the XML schema attached to TS 29.228
*   For the detailed behaviour of SAR/PPR/RTRs, see TS 29.228 sections 6.1.2, 6.2.2 and 6.2.3 respectively
*   For the detailed behaviour of UDRs, see TS 29.328 section 6.1.1 - table 6.1.1.1 describes the four identity sets you can request
*   Communicating implicitly registered identities to UEs, see TS 24.229 section 5.4.1.2.2F (P-Associated-URI headers conveying the implicit registration set) and section 5.4.1.8 (Push-Profile-Request updates)
*   For third-party registration towards application servers (including the fact that only one user identity per service profile is sent), see TS 24.229 section 5.4.1.7

I hope this has been a useful overview of how the IMS data model works, and how it can actually bring real benefits to users in managing their identities. If you have any questions or comments - and especially if you have any corrections - then please comment below or get in touch on the mailing list!
