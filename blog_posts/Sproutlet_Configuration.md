This blog post discusses the recent changes to Sproutlet configuration. We recently (in release 94) introduced some backwards-incompatible changes to how the Sproutlets are configured; this post covers why we did this, and what changes users need to make.

 **Firstly, what are Sproutlets?** Sprout is Clearwater’s SIP router component. It has Sproutlets – these use Sprout’s infrastructure and hold the business logic of the S-CSCF/I-CSCF/BGCF, and various application servers. We currently have the following Sproutlets:

*   S-CSCF: Provides S-CSCF functionality
*   I-CSCF: Provides I-CSCF functionality
*   BGCF: Provides BGCF functionality
*   Gemini: An application server responsible for twinning VoIP clients with a mobile phone hosted on a native circuit-switched network. You can find out more [here](https://github.com/Metaswitch/gemini)
*   Memento: An application server responsible for providing network-based call lists. You can find out more [here](https://github.com/Metaswitch/memento)
*   CDiv: Provides call diversion functionality
*   MMTel: Acts as a basic MMTel AS
*   Mangelwurzel: Acts as a basic B2BUA

Multiple Sproutlets can run on the same node – for example a default Sprout node runs the S-CSCF Sproutlet, the BGCF Sproutlet, and the MMTel Sproutlet. The Sproutlets can also run on separate nodes – for example, you could have your S-CSCF and I-CSCF sproutlets running on separate clusters of Sprout nodes. You can find more information about Sproutlets in our previous blog post [here](TADHack_Sproutlets.md).

**Why was a configuration change needed?** At a high level, we made this change to improve Project Clearwater performance - by around 15% for calls! We also took this opportunity to improve the configuration model for Sproutlets, as the previous model overloaded various configuration options.

**And in more detail?** To explain this, we need to understand how Sproutlets route requests to each other. In a transaction, a request can be sent from one Sproutlet to another – for example the I-CSCF may want to route a request to the S-CSCF. In this case, the request may have the following top [Route header](https://en.wikipedia.org/wiki/Session_Initiation_Protocol) Route: <sip:scscf.sprout.test.cw-ngv.com:5054;transport=tcp> We could route this request by just sending the request to this URI. We’d make a DNS lookup for ‘scscf.sprout.test.cw-ngv.com’ and then route the request externally to this address (following normal SIP processing). If the I-CSCF and S-CSCF are collocated though, this is a bit of a waste. It’s better to notice that the request is destined for a Sproutlet on the same Sprout node, and just send the request directly to that Sproutlet without having to route the request externally. To successfully route a request to a Sproutlet then, the Sproutlet URI must be:

*   Externally routable by DNS to the correct Sprouts (as we support non-collocated Sproutlets).
*   **Detectable as destined for a particular Sproutlet by the Sprout node (in order to route the request internally)**

It’s this second bullet that’s led to the configuration change. Under the existing configuration model for the Sproutlets we found that we couldn’t reliably route messages between Sproutlets internally; instead we needed the extra network hops.

**So what’s the new configuration?** The new configuration model give three configuration parameters for each Sproutlet. These are:

*   <sproutlet>\_prefix: This is the name of the Sproutlet. It is used by the Sproutlet proxy to recognise requests destined for the Sproutlet. The default value is the name of the Sproutlet (e.g. the scscf_prefix has a default value of scscf).
*   <sproutlet>: This is the port that the Sproutlet listens on.
    *   If this is set to zero, then the Sproutlet is disabled.
    *   If it's set to a non-zero value then the Sproutlet is enabled, listening on that port.
    *   The default is 0 (with the exception of the S-CSCF/BGCF, where the default is 5054).
    *   Multiple Sproutlets can use the same listen port (but then their URIs must follow the recommended URI format described below).
*   <sproutlet>\_uri: This is the URI for the Sproutlet. It is used by the Sproutlet to recognise what requests are for it, and by other nodes to route requests to the Sproutlet. The default value is sip:<sproutlet\_prefix>.<sprout\_hostname>:<sproutlet\_port>;transport=tcp (e.g the scscf\_uri has a default value of sip:scscf.<sprout\_hostname>:5054;transport=tcp)

There is also one general new configuration option:

*   sproutlet\_port: This is a generic listening port that's used by all Sproutlets if they don't have their own port in the config. This is used to make the configuration of Sproutlets make more sense and be simpler (so if you have S-CSCF/Memento/MMTel/BGCF on a single Sproutlet, just configure sproutlet_port to 5054 rather than adding individual entries for each one). The default is 0\. If this is set, then all Sproutlets are enabled (unless their <sproutlet> option is set to 0).

**Ok, how do I use this configuration?** We expect that for most users the default configuration of the <sproutlet\_prefix> and <sproutlet\_uri> is the right choice. All you need to do then is set <sproutlet>=port in /etc/clearwater/shared_config for each Sproutlet that you want to enable (and you don’t need to set <sproutlet\_prefix> or <sproutlet\_uri>). For example, say you wanted collocated I-CSCF, S-CSCF, BGCF and Memento function on your Sprout node. You need to:

*   Set icscf=5052 in /etc/clearwater/shared_config
*   Set memento=5054 in /etc/clearwater/shared\_config (or sproutlet\_port=5054). Note you don’t need to set the S-CSCF or BGCF port as these Sproutlets are enabled by default.
*   Set the DNS records for the I-CSCF to be icscf.<sprout\_hostname>, the S-CSCF to be scscf.<sprout\_hostname>, and Memento to be memento.<sprout\_hostname>. The BGCF can't be routed to externally, so it doesn't need a DNS record.

**I’m upgrading my deployment – do I need to do anything?** If you have an existing Clearwater deployment (created before release 94), then you’ll probably need to change your S-CSCF configuration. The expected set of changes is:

*   Remove the scscf\_uri setting from /etc/clearwater/shared\_config, and run /usr/share/clearwater/clearwater-config-manager/upload\_shared\_config to propagate this change across your deployment.
*   Set up a DNS record for your S-CSCF URI – this should have the form scscf.<your sprout hostname>, and resolve to all your Sprout nodes that provide S-CSCF function

*   If you have I-CSCF function enabled in your deployment as well, then you’ll also need to change the default S-CSCF for the I-CSCF. Edit the /etc/clearwater/s-cscf.json file to reference sip:scscf.<your sprout hostname>:5054;transport=tcp, and run /usr/share/clearwater/clearwater-config-manager/upload\_scscf\_json to propagate this change across your deployment.
