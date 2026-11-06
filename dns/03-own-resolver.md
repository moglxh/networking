# Stage 3: Running my own authoritative server and resolver

**Lab:** Kali Linux VM, everything on the loopback address (`127.0.0.1`), so no traffic leaves the VM. Software: NSD 4.15.4 (authoritative), Unbound (resolver), dig 9.20.
**Date:** 2026-11-06

## Goal

Understand the two server roles in DNS by building both: an authoritative server that owns a zone, and a recursive resolver that clients ask. This is the base for the case-study lab, where the interesting question is what happens when the answer a resolver hands out is wrong.

## The roles

| Role | Does | Software | Flag it sets |
|---|---|---|---|
| Authoritative server | Holds the records for a zone and answers only from them | NSD | `aa` |
| Recursive resolver | Finds answers for clients and caches them | Unbound | `ra` |

```
client -> Unbound (resolver, port 53) -> NSD (authoritative for lab.test, port 5300)
```

`lab.test` uses the reserved `.test` ending and the addresses come from the documentation ranges, so nothing here can collide with the real internet.

## Authoritative server (NSD)

Two files: a **config** (how the server runs) and a **zone file** (what it answers).

`/etc/nsd/nsd.conf.d/lab.conf`:

```
server:
    ip-address: 127.0.0.1@5300
    hide-version: yes

zone:
    name: "lab.test"
    zonefile: "lab.test.zone"
```

`/etc/nsd/lab.test.zone`:

```
$ORIGIN lab.test.
$TTL 60
@       IN SOA  ns1.lab.test. admin.lab.test. (
                1       ; serial
                3600    ; refresh
                900     ; retry
                604800  ; expire
                60 )    ; negative-cache TTL
@       IN NS   ns1.lab.test.
ns1     IN A    127.0.0.1
www     IN A    192.0.2.10
config  IN A    192.0.2.20
```

- `SOA` is the zone's header record, `NS` names the responsible server, `A` maps a name to an IPv4 address.
- The TTL is 60 seconds so caching is quick to observe.
- `config.lab.test` stands in for the host the Sogou client fetched its configuration from in the case study.
- I validate before starting any DNS server: `nsd-checkconf` printed nothing and `nsd-checkzone` printed `zone lab.test is ok`.

### A failure worth recording

On my first attempt NSD returned:

```
status: SERVFAIL
EDE: 14 (Not Ready): (Zone is configured but not loaded)
```

The server log gave the reason:

```
zonefile lab.test.zone does not exist
```

The config claimed the zone, but the zone file had not been created. I created it, `nsd-checkzone` passed, and the answer came back with `flags: qr aa rd`.

## Resolver (Unbound)

`/etc/unbound/unbound.conf.d/lab.conf`:

```
server:
    do-not-query-localhost: no
    domain-insecure: "lab.test"
    local-zone: "test." transparent

stub-zone:
    name: "lab.test"
    stub-addr: 127.0.0.1@5300
```

- A **stub zone** tells Unbound to ask NSD directly for `lab.test`. A forward zone would hand the question to another resolver, which NSD is not (it cannot recurse).
- `do-not-query-localhost: no` turns off a safety default, because I am deliberately querying a server on the loopback address.
- `domain-insecure` skips DNSSEC validation for this unsigned lab zone (DNSSEC comes later).
- Unbound listens on `127.0.0.1:53` and `[::1]:53` only, so it is not reachable from outside the VM.

## Same data, different roles

| | Asked NSD directly (port 5300) | Asked Unbound (port 53) |
|---|---|---|
| Flags | `qr aa rd` | `qr rd ra` |
| Answer for `www.lab.test` | `192.0.2.10` | `192.0.2.10` |
| ANSWER / AUTHORITY / ADDITIONAL | 1 / 1 / 2 | 1 / 0 / 1 |

The data is identical. The difference is who is vouching for it: NSD claims ownership (`aa`), Unbound is passing it on (`ra`) and trimmed the reply to what I asked for.

## Caching

```
$ dig @127.0.0.1 +noall +answer config.lab.test; sleep 5; dig @127.0.0.1 +noall +answer config.lab.test
config.lab.test.   60   IN   A   192.0.2.20
config.lab.test.   55   IN   A   192.0.2.20
```

The TTL dropped from 60 to 55 across a 5-second sleep. The second answer was served from Unbound's cache; it did not ask NSD again.

## Rebuilt from a clean state

After finishing, I removed my configs and zone file and rebuilt everything from scratch to check I understood each step. The resolver results reproduced: `NOERROR`, flags `qr rd ra`, and the TTL going from 60 to 55.

## What I learned

- A zone is a slice of the namespace that one server is responsible for. Config says how a server runs; the zone file says what it answers.
- `aa` means the server owns the data; `ra` means it is a resolver that will look things up for me.
- Always validate (`nsd-checkzone`, `nsd-checkconf`) before starting a DNS server, and read the server log for the reason behind a SERVFAIL.
- A resolver's cache decides what its clients get until the TTL expires.

## Open questions

- My first `unbound-checkconf` complained that `/var/lib/unbound/root.key` did not exist, before the service had ever started. Later runs passed. I suspect the first start creates the file, but I have not confirmed it.
- I added `local-zone: "test." transparent` as a precaution and did not test whether the lab works without it.
