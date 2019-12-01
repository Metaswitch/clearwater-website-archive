Application Notes
-----------------

### Proxy Transparency

The SIP processing elements in Clearwater, namely Bono and Sprout, are both transaction-stateful proxies in the sense defined in [RFC3261](https://www.ietf.org/rfc/rfc3261.txt). As proxies, they pass all SIP headers transparently. This means that Clearwater inherently supports the vast majority of SIP-related RFCs that define additional SIP behaviors since these behaviors are only relevant to User Agent Clients and User Agent Servers.

### Limitations on Built-in Calling Features

In general, it is necessary to implement Telephony Application Servers in the form of Back-to-Back User Agents (B2BUA). This is because some calling features require the manipulation of call legs that typically need to be anchored in the IMS infrastructure. Clearwater's built-in TAS is an extension of the transaction-stateful proxy function in Sprout. The functionality of this TAS is therefore limited to calling features that can be implemented by simple proxy-based manipulation of SIP messages. This includes the following kinds of features:

*   Call Forwarding in all its varieties
*   Call Barring in all its varieties
*   Caller ID presentation and restriction services

Calling features that involve manipulation of call legs, for example Call Transfer or Three-Way Calling, typically make use of the SIP [REFER](https://www.ietf.org/rfc/rfc3515.txt) request. Clearwater will pass such requests through transparently. In a perfect world with end-to-end SIP transparency, these calling features would require no explicit interaction with a TAS. Indeed, SIP endpoints that support sending and receiving REFER requests and that all are connected to a single Clearwater system should be able to perform Call Transfer and (provided a suitable conference bridge resource is available) invoke Three-Way Calling. However the reality is that, for a variety of reasons, network operators do not (yet) support the passing of REFER requests across network interconnects. It is therefore necessary for REFER requests to be intercepted by a B2BUA in the form of a TAS that can perform the necessary call leg manipulations on behalf of the endpoints. Clearwater's built-in TAS is not capable of fulfilling this role.

### Session Border Control

Bono implements the IMS Proxy-CSCF function. Session Border Controllers (SBCs) typically also implement the P-CSCF function. This does not mean that Bono is an SBC. SBCs perform a wide range of functions above and beyond the P-CSCF function, including protecting the IMS network from various kinds of malicious attack, manipulating the contents of SIP messages for reasons of interoperability and relaying media in support of NAT traversal. Bono does not perform any of these functions. While it is perfectly possible to use Clearwater without any kind of SBC, it may not be a good idea to do so. If the objective is to offer a voice and video calling service that is extremely inexpensive to set up and operate, deploying Clearwater without an access SBC may be appropriate. Note that Clearwater supports STUN / TURN / ICE and therefore minimizes the need for media relay to handle NAT traversal when deployed with clients that support these protocols. But such deployments will be vulnerable to denial of service attacks. Where security, SIP interop with B2BUA TAS functions and media relay or media anchoring are required, access SBCs should be deployed with Clearwater.
