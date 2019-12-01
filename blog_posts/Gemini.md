Gemini, Project Clearwater’s device twinning application server
-------------------------------------------------------------------
Our work to refactor and simplify Sprout has allowed us to build high-performance application servers easily. We have already released two such application servers, [Memento](https://github.com/Metaswitch/memento) and [Gemini](https://github.com/Metaswitch/gemini). This blog post provides an overview of Gemini, our device twinning application server. A blog post on Memento, our call list application server, will follow in the coming weeks.

**What does Gemini do?** Named after the constellation, Gemini is an application server responsible for twinning VoIP clients with a mobile phone hosted on a circuit-switched network.

*   When a call comes in to a terminating subscriber, Gemini forks the call to any VoIP clients twinned with the subscriber's circuit-switched device.
*   The subscriber can answer the call on any of the VoIP clients or on the circuit-switched device, and the other clients stop ringing.
*   Initially when forking the call, Gemini avoids forking to any VoIP clients that are actually running on the circuit-switched device - otherwise the user experience would be very confusing.
*   If Gemini finds that the circuit-switched device can't be reached over the circuit-switched network, Gemini retries the call to any VoIP client located on the circuit-switched device. This means that subscribers who have 4G or WiFi but no circuit-switched radio access can still receive calls.

**How does it work?** Gemini is invoked as a terminating application server for INVITEs. It forks the INVITE to the VoIP clients and the circuit-switched device. There are two mechanisms for triggering this forking.

*   Gemini adds a pre-configured prefix to the request URI of the INVITE for the circuit-switched device. This INVITE gets routed out via the [G-MSC](http://en.wikipedia.org/wiki/Network_switching_subsystem#Mobile_switching_center_.28MSC.29) (the interface to the PSTN) which strips off the prefix and forwards the request onto the correct device.
*   Gemini adds an ‘Accept-Contact’ header specifying ‘g.3gpp.ics’ to the INVITE for the circuit-switched device. This header triggers a different application server, an SCC-AS (more about this in a future blog post), which rewrites the request URI to a Mobile Station Routing Number that uniquely identifies the circuit-switched device in the mobile network. The request is again routed out via the GMSC which forwards the request on to the correct device.

A ‘Reject-Contact’ header is used on the INVITE for the VoIP client to prevent both the VoIP client and the mobile service from ringing on the same device. If Gemini finds out that the mobile device isn’t registered over the circuit-switched network, it will later try to contact a VoIP client hosted on the same device so that the device can still receive calls when it is out of range of mobile connectivity.
