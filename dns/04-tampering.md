# Stage 4: Tampering with DNS answers (lab only)

**Lab:** Kali Linux VM, my own NSD (authoritative) and Unbound (resolver) on the loopback address. Nothing here touches any network or system I don't own.
**Date:** 2026-11-06

## Goal

See, in my own lab, the main ways a client can end up with a wrong DNS answer, organised by where in the chain the tampering happens.

## Where tampering can happen

| Where | Example | Done in the lab? |
|---|---|---|
| Client settings | A line in `/etc/hosts`, or the resolver set in the NetworkManager profile | Yes (Stage 2) |
| The resolver | Config or data on the resolver changed | Yes (section 2) |
| On the path | Forged replies racing the real one | Explained from a measurement (section 3) |
| The owner's side | The zone's own data changed | Yes (section 4) |
| The protocol itself | Tunneling, amplification | Explained only |

## 1. Client side

In Stage 2, a fake `/etc/hosts` entry for `google.com` made `getent hosts` return `192.0.2.1` (a documentation address) without asking DNS.

## 2. Resolver side

I added one line to Unbound's config: `local-data: "config.lab.test. 60 IN A 192.0.2.99"`.

```
$ dig @127.0.0.1 +noall +answer config.lab.test
config.lab.test.   60   IN   A   192.0.2.99
$ dig @127.0.0.1 -p 5300 +noall +answer config.lab.test
config.lab.test.   60   IN   A   192.0.2.20
```

The owner's data never changed. Only what this resolver tells its clients did, and the tampered answer looked completely normal (`NOERROR`, normal TTL). Removing the line restored the original answer.

## 3. What a resolver checks before trusting a reply

As I understand it, a reply is accepted if it matches the question, the transaction ID and the source port the query was sent from. I measured the port: five lookups of non-existent names forced Unbound to ask NSD five times, and it used five different source ports (38875, 12236, 11289, 34657, 50381). Random ports multiply the 65,536 possible transaction IDs, which is what makes blind forging impractical. (tcpdump does not decode port 5300 as DNS by default, so I could not see the IDs themselves.)

## 4. Owner side and cache lag

I changed the zone (`config` from 192.0.2.20 to 192.0.2.99, serial 1 to 2) and restarted NSD:

```
(cache primed)   config.lab.test.   60   IN   A   192.0.2.20
NSD              config.lab.test.   60   IN   A   192.0.2.99
Unbound          config.lab.test.   60   IN   A   192.0.2.20
```

Unbound kept serving the old answer from its cache even though the owner's data had changed, so the TTL delays a change by up to its length. The TTL printed 60 rather than a lower number, probably because less than a second passed. I then restored the zone (serial 3).

## Explained, not reproduced

Cache poisoning, on-path spoofing, DNS tunneling and amplification are described above but I did not build tools for them.

## What I learned

- The same wrong answer can be introduced at the client, at the resolver, or at the owner, and each is a different compromise.
- A tampered answer looks like any other answer; nothing in the reply marks it.
- Cached answers outlive changes at the source, for good and for bad.

## Open questions

- I did not test that Unbound adopts the new data once its cached copy expires; I restored the zone before waiting.
