# Stage 3: Running my own authoritative server and resolver

**Lab:** Kali Linux VM, everything on the loopback address (`127.0.0.1`), no traffic leaves the VM.
**Date:** 2026-11-05

## Goal

Understand the two server roles in DNS by building both: an authoritative server that owns a zone, and a recursive resolver that clients ask. This is the base for the case-study lab, where the interesting question is what happens when the answer a resolver hands out is wrong.

## The roles

| Role | Does | Software | Flag it sets |
|---|---|---|---|
| Authoritative server | Holds the records for a zone and answers only from them | NSD | `aa` |
| Recursive resolver | Finds answers for clients and caches them | Unbound | `ra` |

```
dig → Unbound (resolver, :53) → NSD (authoritative for lab.test, :5300)
```

`lab.test` uses the reserved `.test` ending, and the addresses are from the documentation ranges, so nothing here can collide with the real internet.

## Authoritative server (NSD)

Two files: a **config** (how the server runs) and a **zone file** (what it answers).

- Config: listen on `127.0.0.1` port `5300`, and "I own `lab.test`".
- Zone file: `SOA` and `NS` records, plus `ns1`, `www` (192.0.2.10) and `config` (192.0.2.20). `config.lab.test` stands in for the host the Sogou client fetched its configuration from.
- TTL is 60 seconds so caching is quick to observe.

### A failure worth recording

My first query returned:

```
status: SERVFAIL
EDE: 14 (Not Ready): (Zone is configured but not loaded)
```

The server log gave the reason:

```
zonefile lab.test.zone does not exist
```

The config claimed the zone, but the zone file had not been created. I recreated it, `nsd-checkzone` reported `zone lab.test is ok`, and the answer came back with `flags: qr aa rd`.

## Resolver (Unbound)

Unbound has a **stub zone** that sends queries for `lab.test` straight to NSD at `127.0.0.1@5300`. A stub zone is used instead of a forward zone because NSD is not a resolver and cannot recurse. I also had to turn off two safety defaults for the lab (querying localhost, and DNSSEC validation for this unsigned zone).

## Same data, different roles

| | Asked NSD directly (:5300) | Asked Unbound (:53) |
|---|---|---|
| Flags | `qr aa rd` | `qr rd ra` |
| Answer | `192.0.2.10` | `192.0.2.10` |
| Sections | ANSWER 1, AUTHORITY 1, ADDITIONAL 2 | ANSWER 1, AUTHORITY 0, ADDITIONAL 1 |
| Query time | 0 ms | 3 ms |

The data is identical. The difference is who is vouching for it: NSD claims ownership (`aa`), Unbound is only passing it on (`ra`).

## Caching

```
$ dig @127.0.0.1 +noall +answer config.lab.test; sleep 5; dig @127.0.0.1 +noall +answer config.lab.test
config.lab.test.   60   IN   A   192.0.2.20
config.lab.test.   55   IN   A   192.0.2.20
```

The TTL dropped from 60 to 55 across a 5-second sleep. The second answer was served from Unbound's cache; it did not ask NSD again.

## What I learned

- A zone is a slice of the namespace that one server is responsible for; config says how a server runs, the zone file says what it answers.
- `aa` means the server owns the data; `ra` means it is a resolver that will look things up for me.
- Always validate (`nsd-checkzone`, `nsd-checkconf`) before starting a DNS server, and read the server log for the reason behind a SERVFAIL.
- A resolver's cache decides what its clients get until the TTL expires.

## Open questions

- `unbound-checkconf` complained that `/var/lib/unbound/root.key` did not exist, yet Unbound ran and answered. I have not confirmed why.
- I added `local-zone: "test." transparent` as a precaution and did not test whether it was needed.
