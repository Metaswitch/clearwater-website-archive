Secrets of the IMS Data Model - Changing a user’s data
-----------------------------------------------------
_This is the third post in a three-part series on the IMS data model – see [here](Secrets_1.md) and [here](Secrets_3.md) for the other entries._

# **Changing a user’s data**

In the first and second blog posts in this series, we looked at the subscriber data model in IMS, the ways you can associate identities together, and the ways CSCFs and application servers learn about those identities. However, we can’t necessarily expect a subscription’s data to remain static forever - new identities and aliases can be added, authentication credentials can change, or a user’s subscription could be removed entirely. In this blog post, we’ll look at what’s involved in making those changes, particularly when that subscription is currently active on the network. When a network operator makes these administrative changes to a user in the HSS, if an S-CSCF is currently providing service to that user, the HSS sends either a Push-Profile-Request or a Registration-Termination-Request message to the S-CSCF to inform it of the change. This can cause difficulties, particularly when the scope of the data being changed doesn’t match the scope of the data learnt by the S-CSCF. Changes which affect an entire IMS subscriptions are the main example of this – the S-CSCF may never learn that two identities share the same IMS subscription, and the only way the HSS could indicate that they’re linked is through sending **optional** AVPs.

## Push-Profile-Request

Push-Profile-Requests can update any of the data sent in a Server-Assignment-Answer or Multimedia-Auth-Answer. A PPR can:

*   add an implicitly registered identity to an implicit registration set
*   remove an implicitly registered identity from an implicit registration set - although it's not allowed to remove the first one in the list, known as the Default Public User Identity, and must send a Registration-Termination-Request to do that
*   change the aliasing of an existing identity
*   change a service profile for an existing identity
*   change the authentication credentials for a private identity

In the first two cases, where the PPR actually changes what identities are registered, the S-CSCF will broadcast the change out to the wider network through sending NOTIFY messages to any UEs, P-CSCFs or application servers that have subscribed to the reg-event package, informing them of the new registration state. As all the information the S-CSCF needs is provided, performing these updates is relatively straightforward - the full User-Data, including all the public IDs in the implicit registration set, is provided for the first four, and authentication credentials are linked to a particular private identity. However, a Push-Profile-Request can also change the list of charging collection functions (CCFs) for a subscription, and this is more complicated: CCFs are data common to the whole IMS Subscription, but the whole IMS Subscription is never identified to the S-CSCF - the PPR just includes a single private identity. For example, if only Bob's mobile and Alice's mobile were online – recall from the first blog post in this series that their only association is that they are in the same IMS Subscription - then both their CCFs must be updated. This is why the Server-Assignment-Answer contains an Associated-Identities element with all the private identities in the IMS Subscription - when Bob's mobile and Alice's mobile are each registered, the SAA will include the four private identities (alice.smith@imstelephonecorp.example, bob.smith@imstelephonecorp.example, cedric.smith@imstelephonecorp.example, smith.family.landline@imstelephonecorp.example), so a PPR specifying any one of these identities can be associated with both numbers. As noted in the last blog post, though, it's optional to send Associated-Identities. The specification for Push-Profile-Requests does require that "The HSS shall always send a Private Identity that is known to the S-CSCF based on an earlier SAR/SAA procedure", so this can still be made to work. Although the specs aren't explicit, the way we think this has to work is either:

*   the HSS would send two PPRs, one specifying alice.smith@imstelephonecorp.example and one specifying bob.smith@imstelephonecorp.example.
*   the HSS would use a specific private user identity (e.g. the.smith.family@imstelephonecorp.example) on all Server-Assignment-Answers and on all Push-Profile-Requests, unofficially using that identity to identify the subscription

## Registration-Termination-Request

Registration-Termination-Requests notify an S-CSCF that a subscriber’s registration should be removed. Because this changes the user’s registration state, the S-CSCF then sends NOTIFYs to any network elements which have subscribed to the reg-event package (as it does for some types of Push-Profile-Requests). RTRs also share some of the complexities as updating the CCFs does, because they affect assignment to a particular S-CSCF, which (like charging information) is shared across a whole subscription. The types of RTR are:

*   PERMANENT\_TERMINATION - this removes the registration of a public identity with one or more specified private identities (not the whole IMS Subscription at once).
*   NEW\_SERVER\_ASSIGNED - this tells the S-CSCF that a new S-CSCF has already been assigned to the IMS subscription (i.e. while this S-CSCF was unavailable), and it should remove any data it has on that subscriber. (NOTIFY messages
*   SERVER\_CHANGE - this tells the S-CSCF that the HSS would like to assign a new S-CSCF to the IMS subscription (but has not done so yet). This RTR is the only one which requires the S-CSCF to send de-registration messages (e.g. NOTIFYs to UEs) for the whole subscription, so it includes some extra checking to confirm that the S-CSCF has done so - the S-CSCF returns an Associated-Identities element in the response, covering all private identities that were affected, and the HSS sends further RTRs for any private identities that were missed.
*   REMOVE\_SCSCF - this tells the S-CSCF that it is no longer assigned to an IMS Subscription for unregistered service. (This might identify the subscription only through a private identity - if so, it would have to match the one on the Server-Assignment-Answer from when this S-CSCF became assigned for unregistered service.)

Registration-Termination-Requests:

*   must always specify at least one private identity (in the User-Name AVP)
*   can specify additional private identities (in the Additional-Identities AVP)

Like PPRs, then, RTRs identify subscriptions primarily through private identities, so they need either:

*   the Additional-Identities AVP to be used on Server-Assignment-Answers, to tell the S-CSCF about all private identities in a subscription
*   to consistently use a specific private identity in the User-Name AVP to unofficially identify the subscription
*   or for the S-CSCF and HSS to carefully track which private identities have been exchanged on SAR/SAA requests

That said, RTRs have more flexibility than PPRs (in that they _can_ send all the private identities and public identities in a subscription) - we'd probably expect HSSes to do this as best practice, to give S-CSCFs the best chance of doing the right thing.

## References/further reading

Interested readers may want to read the 3GPP specs on this themselves, whether to check the detail of a particular point, check my reasoning, or to find more information on related topics. In this section, I've tried to give a brief overview of what the relevant sections of each specs are, and where you should look for the definitive information on particular topics:

*   For the detailed behaviour of PPR/RTRs, see TS 29.228 sections 6.1.2, 6.2.2 and 6.2.3 respectively
*   TS 24.229, section 5.4.1.8 covers some additional actions the S-CSCF should perform when it sees a Push-Profile-Request which updates a service profile
*   TS 24.229, section 5.4.2.1.1 describes the NOTIFY messages the S-CSCF sends out when it receives one of these administrative changes

I hope this has been a useful overview of how the IMS data model works, and how it can actually bring real benefits to users in managing their identities. If you have any questions or comments - and especially if you have any corrections - then please comment below or get in touch on the mailing list!
