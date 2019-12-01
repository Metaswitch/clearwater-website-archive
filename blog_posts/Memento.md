Memento - Project Clearwater's Network-based Call List Application Server
------------------------------------------------------------------------
The architecture of Project Clearwater's [Sprout](https://github.com/Metaswitch/sprout) component allows us to easily build high-performance application servers. We talked about [Gemini](https://github.com/Metaswitch/gemini) in a previous [blog post](Gemini.md) and today we're covering [Memento](https://github.com/Metaswitch/memento).

## What is Memento?

All modern phones have some sort of "call list" feature - a record of calls made and received on the device. However as VoIP becomes more widespread, users can have many _different_ VoIP-enabled devices (for example a VoIP client on their PC, tablet, and smartphone). A simple "per-device" call list is no longer sufficient to provide a unified experience. Memento solves this problem. It is located in the network core and is triggered on all calls that a user makes or receives, regardless of what device they used. It then provides that user with a single unified call list. We called the application "Memento" because after the call has ended you are left with a small memento of it - the entry in your call list.

## How does it work?

Memento is a standard IMS application server. It is invoked on all calls made and received by any user that is subscribed to the service. Like all other application servers this is controlled using [Initial Filter Criteria](http://en.wikipedia.org/wiki/IP_Multimedia_Subsystem#Initial_Filter_Criteria). Whenever a user makes or receives a call Memento writes information about the call to a [Cassandra](http://cassandra.apache.org/) database. Call lists are retrieved over HTTP. Memento uses a read-only [XCAP](https://tools.ietf.org/html/rfc4825) interface. This is very similar to the IMS [Ut interface](http://www.3gpp.org/dynareport/24623.htm), though the data format is different (Ut deals with service configuration, not call lists). The HTTP interface supports the following features:

*   [TLS](http://en.wikipedia.org/wiki/Transport_Layer_Security) for security. This can either use a self-signed certificate or one signed by a certificate authority.
*   [Digest authentication](http://en.wikipedia.org/wiki/Digest_access_authentication). A user must know their password to retrieve their call list and this is checked against their credentials stored in Homestead.
*   [gzip](http://www.gzip.org/) compression of the call list data (to save on bandwidth).

Interestingly there is no standards-based format for the call lists themselves. We have invented a simple format based on [XML](http://en.wikipedia.org/wiki/XML).

    <call-list>
      <calls>
        <call>
          <to>
            <URI>alice@example.com</URI>
            <name>Alice Adams</name> <!-- If present -->
          </to>
          <from>
            <URI>bob@example.com</URI>
            <name>Bob Barker</name>  <!-- If present -->
          </from>
          <answered>1</answered>
          <outgoing>1</outgoing>
          <start-time>2002-05-30T09:30:10</start-time> <!-- Standard XML DateTime type -->
          <answer-time>2002-05-30T09:30:20</answer-time> <!-- Present iff call was answered-->
          <end-time>2002-05-30T09:35:00</end-time> <!-- Present iff call was answered-->
        </call>
      </calls>
    </call-list>

Calls are automatically removed from the database, either:

*   After a certain period of time (the default is one week) or
*   Once a user's list gets too long (there is no limit by default).

Both of these settings are configurable on a per-server basis. Additionally you can specify the maximum disk space to use for the call list database. Memento provides [SNMP](http://en.wikipedia.org/wiki/Simple_Network_Management_Protocol) statistics to spot when you are reaching the limit.

## Getting Started

To get started with Memento follow these steps:

*   Install Memento by running `sudo apt-get install memento memento-as`. It can either be installed on your existing Sprout node(s), or on a completely new node.
*   Configure iFCs to invoke Memento as an originating and terminating application server as described [here](http://clearwater.readthedocs.org/en/latest/Configuring_an_Application_Server/index.html?highlight=application%20server#ifc-configuration).
*   Calls to and from that subscriber will be logged in their call list. The list can be retrieved using cURL, by running the following command:`curl https://<memento-hostname>/org.projectclearwater.call-list/users/<public-identity>/call-list.xml --insecure --digest --user <private-identity>:<password>`. The public identity in the URL must be [URL encoded](http://www.w3schools.com/tags/ref_urlencode.asp).

## Where Next?

For more information take a look at the [Memento](https://github.com/Metaswitch/memento) repository on Github.
