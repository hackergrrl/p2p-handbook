# P2P Handbook

## Status: Living Document

This handbook is very much a work in progress. It's incomplete, and I'm filling
in bits and pieces as I go!

You should file issues and ask questions about aspects of p2p you're interested
in!

## THE HANDBOOK

This handbook is an introduction to the primitives of peer-to-peer systems for
the web, and a guide to the powerful abstractions you can build on top of those
primitives to form distributed systems.

This handbook's goal is to prepare you to use the excellent ecosystem of p2p
modules available on npm today, and to develop your own p2p modules and
applications on top.

## Table of Contents

  - [Status: Living Document](#status-living-document)
  - [THE HANDBOOK](#the-handbook)
     - [introduction](#introduction)
     - [why you should use p2p](#why-you-should-use-p2p)
     - [why node &amp; javascript](#why-node-javascript)
     - [what superpowers does p2p grant?](#what-superpowers-does-p2p-grant)
     - [CAP theorum](#cap-theorum)
     - [p2p &quot;layers&quot; (roles)](#p2p-layers-roles)
     - [identity](#identity)
     - [discovery](#discovery)
        - [bittorrent dht](#bittorrent-dht)
        - [tracker / signal server](#tracker-signal-server)
        - [multicast dns](#multicast-dns)
        - [static bootstrap list](#static-bootstrap-list)
        - [dns](#dns)
     - [peer routing](#peer-routing)
     - [swarm / topology](#swarm-topology)
     - [content routing](#content-routing)
     - [p2p data structures](#p2p-data-structures)
        - [other handy not-strictly-p2p modules](#other-handy-not-strictly-p2p-modules)
     - [protocol](#protocol)
     - [awesome p2p modules](#awesome-p2p-modules)
     - [glueing p2p modules together to make apps](#glueing-p2p-modules-together-to-make-apps)
  - [Inspiration](#inspiration)

---

### introduction

This is the handbook I wish that I had when I started delving down the
peer-to-peer rabbit hole.

what is p2p

### why you should use p2p

### why node & javascript

### what superpowers does p2p grant?

technology grants us superpowers that we would otherwise not have

- offline-first
- abstract away corporate infrastructure
- data can flow more freely

### CAP theorum

- <https://en.wikipedia.org/wiki/CAP_theorem>
- dominictarr's article

Consistency. Availability. Partition tolerance. Choose two.

It's possible to "cheat" and get all three: _eventual consistency_

### p2p "layers" (roles)

Whereas the OSI model breaks networks into 7 layers, and the TCP/IP model does that into 4, it's beneficial to think of P2P in terms of _roles_:

```
+=============================+
|            roles            |
+=============================+
| applications                |
+-----------------------------+
| protocols & data structures |
+-----------------------------+
| content routing             |
+-----------------------------+
| swarm topology              |
+-----------------------------+
| peer routing                |
+-----------------------------+
| discovery                   |
+-----------------------------+
| identity                    |
+-----------------------------+
```

### Identity

In a centralized system, this is easy: ask the user to provide an identifier.
Maybe an email address, maybe a username. The server checks whether that
identifier has already been taken. If it has, the user must choose another
identifier. If the identity server is down, nobody can sign in, and nobody can
create new identities. In fact, the whole system might become unusable if the
software can't verify you are who you say you are.

In a distributed system, you are permanently stuck with such restrictions:

1. there is no single authority to register a new identifier with
2. you cannot know of all other peers in the network, so any identity you choose
   may be in conflict with another peer now or in the future
3. there is no single authority to corroborate your claim that you are who you
   say you are

One easy solution might be to have each peer generate a random number between 0
and some big number. If the number is big enough, you're unlikely to see a
conflict now or in the foreseeable future (though [the odds grow
quadratically](https://en.wikipedia.org/wiki/Birthday_attack). This solution
satisfies #1 and #2, but not #3: anyone who saw your identifier could
impersonate you on the network, since there are no authorities to check who's
who. We need a way for users to prove they are who they claim to be, without an
authority to provide confirmation.

What if you could provide some sort of proof that you were the author of a
given message? A digital signature that only the holder of that large random
number could produce?

[Public key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography)
does just this: it's like the above, except *two* "random" numbers are
generated: one you share widely as your identity, and the other you keep secret.
You can use the secret key to produce a signature on any data you'd like, which
anybody else in possession of the message, the signature, and your public key
can verify: all without any remote authorities.

Public keys are long and unwieldy for humans, like
`7bc0bd9bd557547b81d53196674c869c6d698164c68b0033fd4b48849ce59110`. You could
use a human "friendly" encoding like
[proquint](https://github.com/deoxxa/proquint) to make this
`bazol-rikur-lijit-rudij-jilam-kikag-satik-sipim-nojuk-hojam-rapis-gubab-rajuz-hafoh-jogun-bihan`.
Pronouncible, but still not particularly useful to a human being.

There is a conjecture called [Zooko's
Triangle](https://en.wikipedia.org/wiki/Zooko%27s_triangle) that claims no
system can achieve all three properties of being human-meaningful,
decentralized, and secure.

### discovery

given a key, find interested peers

#### bittorrent dht

The bittorrent dht, mainline, is huge: there are millions of nodes worldwide.
You can store arbitrary data in them, but be warned that most nodes are
configured to aggressively flush keypairs they receive.

This approach is great when the swarm lacks the resources to maintain a powerful
reliable central server for discovery -- a DHT is an excellent means for free
short-term storage of peer location information. Because of its low retention
duration, it may be required for peers to republish their contact information at
a regular interval.

 - dht
 - discovery-channel
 - webtorrent

#### tracker / signal server

Run a centralized server(s) at some known IP that many other peers also connect
to. This server can act as a connection broker, exchanging ip:ports of peers to
help them connect to each other directly.

This is also a partially effective means of traversing around certain types of
NATs.

This approach need not be centralized: bittorrent employs a decentralized set of
trackers, which peers are free to decide between to use for discovery.

This is useful when your potential swarm size is small, and your potential peers
are located broadly across the open internet and require very accessible points
(maybe served over port 80) to find.

 - signalhub

#### multicast dns

Multicast DNS is a very well supported protocol (bonjour/zeroconf+apple/airplay,
etc) for finding other machines that are on the same network as you.

It consists of sending a "query" message to the entire network. Based on its
contents, machines who believe themselves applicable can respond with their
information to facilitate direct contact between the two.

This is useful when your peers are likely to be on the same network as you.

 - mdns
 - discovery-channel

#### static bootstrap list

When all else fails, you can still ship your application with a list of
hardcoded peer locations, to help peers join the larger network. This could be
as simple as

```json
[
  "QmVvUkSZqM2EG1SK9s49uN6pizNhXVFHpuJgh53my4A4pP": "tcp://12.62.93.214:9090",
  "QmawE3T8oyxMgz5KaYvsJ2BgQUZWvnKzebwLhqGPA8sqGb": "udp://24.243.12.9:1234",
  "QmQXV24ZjKPGiXfeQd4etZGgywzd4wfaGKGS48EGVdrS6C": "utp://99.4.78.181:97"
]
```

#### dns

Store peer IDs and IP addresses as low-TTL DNS entries (dns round robin). This
relies on DNS' peer-to-peer-like record dissemination protocol to propagate new
peers over the time domain.

This requires a central service that would function a lot like a tracker: it
needs to both a) maintain a subset of active peers in the swarm, and b) publish
a further subset of them to a DNS server.

This approach is so useful because your peer application doesn't even need to
understand p2p protocols: it can just connect to some known example.com and get
a new peer every time it connects.

 - Q: any modules for publishing round robin dns entries?

### peer routing

given a peer-id and peer-info, get a duplex stream

"with this piece: what if we could map a public key to a duplex stream, across the world?"

sub-topics
  - NAT hole-punching
  - relaying

### swarm / topology

- the topology of peers in the network
- <http://blog.daviddias.me/2014/12/20/webrtc-ring>

1.  structured
  - fully-connected-topology
  - kademlia
  - chord
2.  unstructured
  - signalhub
  - bittorrent swarm

### content routing

given a message, figure out which peers to route it to
^ need to get a more solid understanding on this

1.  examples
    1.  gossip
        1.  secure-gossip
        2.  hyperlog / ssb (anti-entropy)

### p2p data structures

data structures well suited to an unreliable network (or offline) with untrusted peers

1. stream
2. crdt
  - <https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type>
  - <https://github.com/pfraze/crdt_notes>
3. append-only log
  1. hyperlog
  2. secure-scuttlebutt <https://scuttlebot.io/more/protocols/secure-scuttlebutt.html>
4. content addressable store
5. merkle linking (blockchain, hyperlog, ssb, etc)
  a. merkle dag (ipfs)
7. dht

#### other handy not-strictly-p2p modules

1. leveldb
2. abstract-blob-store

### protocol

what messages the peers of the network agree to exchange

1. streams: the ultimate transport-agnostic abstraction
2. multistream: multiplexing protocols over a stream

### awesome p2p modules

TODO: sort these into their respective sections? or have tiny per-module bits?

- signalhub (peer discovery)
- hyperlog (identity, data structure, protocol)
- discovery-channel (peer discovery)
- discovery-swarm (peer discovery + content routing + swarm)
- bittorrent-dht (peer discovery + swarm)
- ssb-keys (identity)
- pubsub-swarm (identity, peer discovery, swarm, content routing)
- secure-gossip (content routing, protocol)

### glueing p2p modules together to make apps

1. p2p picture sharing service (walk through glueing together modules from the
   roles to do this)

## Further Reading

- http://the-paper-trail.org/blog/distributed-systems-theory-for-the-distributed-systems-engineer/

## Inspiration

- <https://github.com/ipfs/specs/tree/master/libp2p>
- <https://github.com/mafintosh/p2p-workshop>
- <https://github.com/substack/stream-handbook>
- <https://ipfs.io>
- <https://github.com/dominictarr/>
- <https://github.com/diasdavid/>
- <https://github.com/mafintosh/>
- <https://github.com/jbenet/>
- <https://github.com/substack/>


