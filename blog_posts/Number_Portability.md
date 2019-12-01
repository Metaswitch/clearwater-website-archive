# Number Portability

## Setting the Scene

Phone subscribers sometimes change their phone provider (in fact, you’ve probably done it yourself at some point). They might do so because their moving house to somewhere where their current provider has no coverage, or because they’ve found a better deal with another provider, or a multitude of other reasons. When this happens, it has historically often been possible for the subscriber to keep their phone number (especially if they’re staying in one area code, or transferring a mobile number in Europe), so that their contacts don’t have to update their contact lists and so that they don’t have to update their contact information with any companies/websites they might have dealings with. This process is known as “porting” the number to the new carrier and the feature as a whole is called “Number Portability”.

## In a Pre-IMS World

Pre-VoIP Telcos have a well-established method for handling this situation. At some point in call flow (see below for the options of where this happens), a provider determines that the requested number has been ported and discovers a `routing number` that it should use to get to the provider that now hosts the target subscriber. The provider then sends a request to the `routing number` that includes the originally requested number. The new home provider receives this request, extracts the original target number and continues to process the call as they would for a normal hosted subscriber (applying call services, routing to endpoints etc.) The `routing number` lookup can happen in a few places:

1.  ACQ (All Call Query) - The operator that originates the call checks a centralized database for a `routing number` associated with the requested number.
2.  QoR (Query on Release) - The operator that originates the call forwards the call to the original owner and, if that fails, checks a centralized database for a `routing number`.
3.  RoP (Return to Pivot) - The operator that originates the call forwards the call to the original owner. The original owner performs a lookup in a local database and finds a `routing number`, it then sends this `routing number` back to the originator operator, who forwards the call. This can happen multiple times if the number has been ported multiple times.
4.  OR (Onward Routing) - The operator that originates the call forwards the call to the original owner. The original owner performs a lookup in a local database and finds a `routing number`, and forwards the call to that `routing number` to reach the new terminating provider. This can happen multiple times if the number has been ported multiple times.

Of these four options, ACQ is the most commonly used model, since Home Location Register (HLR) lookups are very fast and efficient and it saves the original home network from having to do work to serve a subscriber that has been ported away.

## In an IMS World

IMS networks, many identities are SIP URIs and therefore (similar to email addresses) reference a specific service provider. As such these SIP URI identities can’t be ported between providers. On the other hand, particularly in VoLTE deployments, IMS allows subscribers to have TEL URIs as identities which (like phone numbers) can be ported. This occurs in three ways:

1.  The local deployment may want to route a call to a number that has been ported to a non-IMS deployment (Case A)
2.  The local deployment may want to route a call to a number that has been ported to another IMS deployment (Case B)
3.  An IMS subscriber has been ported from the local deployment to another deployment and the deployment wishes to route the call to that number’s new home (Case C)

The IMS specifications, in particular [TS 24.229](http://www.3gpp.org/dynareport/24229.htm), give approaches for how to handle each of these cases.

## Case A - Target number ported to a non-IMS deployment

If a call is made to a phone number that has been ported to a non-IMS deployment, the following process will occur:

1.  The target phone number may be checked for porting at one of the following points:
    *   Application Server - possibly an IM-SSF or a dedicated Number Portability AS
    *   Originating S-CSCF - Through an ENUM lookup
    *   BGCF - Through an ENUM lookup
    *   MGCF - Through an ENUM lookup
2.  Once the LNP lookup has been performed, the call will be routed to the learned `routing number` via the MGCF and thence arrive at the correct provider network.

## Case B - A target number may have been ported to another IMS deployment

This is the same case as the previous one, except that this time the deployment that’s originating the call and the deployment that the target number was ported to are both IMS deployments and have a SIP peering relationship. Here the deployments can skip pre-IMS number portability processing and directly route between each other. The process that is followed is similar to Case A except:

*   The target number will be translated to a routeable SIP URI (without the “user=phone” parameter).
*   The translation process is performed by an AS (method unspecified) or the originating S-CSCF (probably though ENUM). The translation will have already been applied (though one of these mechanisms) before BGCF/MGCF are invoked.

## Case C - Call to a locally Ported-Out Number

The last case that’s interesting is when a deployment handles call for a number which is nominally owned by the local deployment but which has been ported out. If the call is being made from another subscriber in the local deployment, then this case can be handled exactly as in Case A or Case B. If neither Case A, nor Case B are supported on this deployment, or the call is coming in to the deployment from some other deployment that is not aware that the number has been ported out then the I-CSCF will handle the LNP lookup by querying the requested number in ENUM (after first checking for the subscriber in the HSS as it would for a normally hosted subscriber).

*   If the ENUM lookup returns a routeable SIP URI, then the I-CSCF will route that appropriately (either to a local terminating S-CSCF if the number wasn’t ported after all, or to the BGCF if the SIP URI belongs to a different domain).
*   If the ENUM lookup returns LNP information, the I-CSCF will update the ReqURI and route the request to the BGCF to be routed on to the MGCF and thence to the `routing number`.

## Performing the LNP lookup

### LNP lookups at an AS

This approach allows a carrier to continue to use some of their pre-IMS infrastructure to handle number portability. Here LNP lookups are performed by an Application Server, in particular they may be performed on an IM-SSF (which is capable of querying the pre-IMS network to answer LNP queries). The actual implementation of the AS is unspecified by IMS but crucially it must conform to the ISC interface between the S-CSCF and all AS instances.

### LNP lookups using ENUM

Here the LNP lookup is performed by the originating S-CSCF (after applying call services), the BGCF or the MGCF in the originator’s network. The lookup is performed by querying an ENUM database for the target number. The ENUM service may reply with a result that rewrites the target number to a TEL URI with the appropriate parameters to indicate the number is ported. As an example, the number `+16505551234` is ported to another provider who’s routing number is `+16305550000`. An ENUM lookup on the number might return:

    $ dig NAPTR 4.3.2.1.5.5.5.0.5.6.1.e164.arpa
    4.3.2.1.5.5.5.0.5.6.1.e164.arpa. 500 IN  NAPTR   1 1 "u" "E2U+sip" "!(^.*$)!tel:\\1;npdi;rn=+16305550000!" .

The regular expression in the response (`!(^.*$)!tel:\\1;npdi;rn=+16305550000!`) will translate `+16505551234` to `tel:+16505551234;npdi;rn=+16305550000`, adding the routing number in the `rn` parameter.

### Using the learned LNP information

Now that LNP lookups have been performed (and we still have a TEL URI or a SIP URI with `user=phone`) the call will be routed to the BGCF and then to the MGCF to be forwarded to the target deployment. The BGCF or MGCF will use the `rn` parameter to pick a route for a given call.

## Providing LNP with Clearwater

Project Clearwater can apply LNP function through a few mechanisms:

*   By calling out to an AS during call processing, allowing the AS to perform the appropriate lookups.
*   By performing an ENUM lookup when handing a call to a phone number (either a `tel:` URI or a `sip:` URI with the `user=phone` parameter).
*   When acting as a BGCF, Bono’s routing configuration can inspect the `rn` parameter (if present) to pick routes for a given call.

For help configuring these features, see our documentation on configuring [ISC Application Servers](https://clearwater.readthedocs.org/en/latest/Configuring_an_Application_Server/index.html), [ENUM](https://clearwater.readthedocs.org/en/latest/ENUM/index.html) and [BGCF](https://clearwater.readthedocs.org/en/latest/IBCF/index.html#bgcf-configuration) in Clearwater.
