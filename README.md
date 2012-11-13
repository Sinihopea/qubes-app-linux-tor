TorVM for Qubes (qubes-tor)
==========================

TorVM is a ProxyVM that provides torified networking to all its clients.

By default, any AppVM using the TorVM as its NetVM will be fully torified, so
even applications that are not Tor aware will be unable to access the outside
network directly.

Moreover, applications running behind a TorVM are not able to access globally
identifying information (IP address and MAC address).

Due to the nature of the Tor network, only TCP and DNS traffic is allowed. All
non-DNS UDP traffic is silently dropped.

## Warning + Disclaimer

1. TorVM is not a magic anonymizing solution. Protecting your identity requires a
change in behavior. Read the "Protecting Anonymity" section below.

2. Traffic originating from the TorVM itself **IS NOT** routed through
Tor. This includes system updates to the TorVM. Only traffic from VMs using
TorVM as their NetVM is torified.

Installation
============


0. *(Optional)* If you want to use a separate vm template for your TorVM

        qvm-clone fedora-17-x64 fedora-17-x64-net

1. In dom0, create a proxy vm and disable unnecessary services and enable qubes-tor


        qvm-create -p torvm
        qvm-service torvm -d qubes-netwatcher
        qvm-service torvm -d qubes-firewall
        qvm-service torvm -e qubes-tor
          
        # if you  created a new template in the previous step
        qvm-prefs torvm -s template fedora-17-x64-net

2. From your template vm, install the torproject Fedora repo

        sudo yum install qubes-tor-repo-*.fc17.x86_64.rpm

3. Then, in the template, install the TorVM init scripts

        sudo yum install qubes-tor-init-*.fc17.x86_64.rpm

5. Configure an AppVM to use TorVM as its netvm

        qvm-prefs -s work netvm torvm

6. Start the TorVM and any AppVM you have configured
6. From the AppVM, verify torified connectivity

        curl http://check.torproject.org

Tor logs to syslog, so to view messages use `sudo grep Tor /var/log/messages`

Usage
=====

Applications should "just work" behind a TorVM, however there are some steps
you can take to protect anonymity and increase performance.

## Protecting Anonymity

The TorVM only purports to prevent the leaking of two identifiers:

1. WAN IP Address
2. NIC MAC Address

This is accomplished through transparent TCP and transparent DNS proxying by
the TorVM.

The TorVM cannot anonymize information stored or transmitted from your AppVMs
behind the TorVM. 

*Non-comphrensive* list of identifiers TorVM does not protect:

* Time zone
* User names and real name
* Name+version of any client (e.g. IRC leaks name+version through CTCP)
* Metadata in files (e.g., exif data in images, author name in PDFs)
* License keys of non-free software

### Further Reading

* [Information on protocol leaks](https://trac.torproject.org/projects/tor/wiki/doc/TorifyHOWTO#Protocolleaks)
* [Official Tor Usage Warning](https://www.torproject.org/download/download-easy.html.en#warning)
* [Tor Browser Design](https://www.torproject.org/projects/torbrowser/design/)


## Performance

In order to mitigate identity correlation TorVM makes heavy use of Tor's new
[stream isolation feature][stream-isolation]. Read "Threat Model" below for more information.

However, this isn't desirable in all situations, particularly web browsing.
These days loading a single web page requires fetching resources (images,
javascript, css) from a dozen or more remote sources. Moreover, the use of
IsolateDestAddr in a modern web browser may create very uncommon HTTP behaviour
patterns, that could ease fingerprinting.

Additionally, you might have some apps that you want to ensure always share a
Tor circuit or always get their own.

For these reasons TorVM ships with two open SOCKS5 ports that provide Tor
access with different stream isolation settings:

* Port 9050 - Isolates destination port and address, and by SOCKS Auth  
	          Same as default settings listed above, but each app using a unique SOCKS
              user/pass gets its own circuit.
* Port 9049 - Isolates by SOCKS Auth and client address only  
              Each AppVM gets its own circuit, and each app using a unique SOCKS
              user/pass gets its own circuit

SOCKS Port 9049 should be used by web browsers.

Threat Model
============

TorVM assumes the same Adversary Model as [TorBrowser][tor-threats], but does
not, by itself, have the same security and privacy requirements.

## Proxy Obedience

The primary security requirement of TorVM is *Proxy Obedience*.

Client AppVMs MUST NOT bypass the Tor network and access the local physical
network, internal Qubes network, or the external physical network.

Proxy Obedience is assured through the following:

1. All TCP traffic from client VMs is routed through Tor
2. All DNS traffic from client VMs is routed through Tor
3. All non-DNS UDP traffic from client VMs is dropped
4. Reliance on the [Qubes OS network model][qubes-net] to enforce isolation

## Mitigate Identity Correlation

TorVM SHOULD prevent identity correlation among network services.

Without stream isolation, all traffic from different activities or "identities"
in different applications (e.g., web browser, IRC, email) end up being routed
through the same tor circuit. An adversary could correlate this activity to a
single pseudonym.

By default TorVM uses the most paranoid stream isolation settings for
transparently torified traffic:

* Each AppVM will use a separate tor circuit (`IsolateClientAddr`)
* Each destination port will use a separate circuit (`IsolateDestPort`)
* Each destination address will use a separate circuit (`IsolateDestAdr`)

For performance reasons less strict alternatives are provided, but must be
explicitly configured.

Future Work
===========
* Integrate Vidalia
* Create Tor Browser packages w/out bundled tor
* Use local DNS cache to speedup queries (pdnsd)
* Support arbitrary [DNS queries][dns]
* Configure bridge/relay/exit node
* Normalize TorVM fingerprint
* Optionally route TorVM traffic through Tor
* Fix Tor's openssl complaint

[stream-isolation]: https://gitweb.torproject.org/torspec.git/blob/HEAD:/proposals/171-separate-streams.txt
[tor-threats]: https://www.torproject.org/projects/torbrowser/design/#adversary
[qubes-net]: http://wiki.qubes-os.org/trac/wiki/QubesNet
[dns]: https://tails.boum.org/todo/support_arbitrary_dns_queries/
