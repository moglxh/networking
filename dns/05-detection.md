# Stage 5: Detecting tampered DNS answers (lab only)

**Lab:** Kali Linux VM, my own NSD (authoritative) and Unbound (resolver) on the loopback address.
**Date:** 2026-11-06

## Goal

A tampered answer carries no marker (it comes back as a normal `NOERROR` with a normal TTL), so detection has to compare it with something trusted or watch for answers that should not appear.

## Detection 1: compare the resolver with the owner

`~/dns-check.sh` asks the resolver and the authoritative server the same question and reports whether they agree:

```bash
#!/bin/bash
name="$1"
resolver=$(dig @127.0.0.1 +short "$name" | sort | tr '\n' ' ')
owner=$(dig @127.0.0.1 -p 5300 +short "$name" | sort | tr '\n' ' ')
if [ "$resolver" = "$owner" ]; then
  echo "OK $name: $resolver"
else
  echo "MISMATCH $name: resolver=[$resolver] owner=[$owner]"
fi
```

| State | Output |
|---|---|
| Before tampering | `OK config.lab.test: 192.0.2.20` |
| Resolver tampered (`local-data` set to 192.0.2.99) | `MISMATCH config.lab.test: resolver=[192.0.2.99 ] owner=[192.0.2.20 ]` |
| After removing the tampering | `OK config.lab.test: 192.0.2.20` |

This needs a trusted source to compare against, and it only catches tampering that makes the resolver differ from the owner.

## Failure signals

| Situation | Status |
|---|---|
| Owner (NSD) stopped, resolver cannot get an answer | `SERVFAIL` |
| Owner running, name does not exist | `NXDOMAIN` |

In logs, `SERVFAIL` points at an unreachable or broken server, and `NXDOMAIN` at a missing name.

## What I learned

- Nothing in a DNS reply marks it as tampered; detection depends on a trusted reference.
- A mismatch between resolver and owner is a simple, reliable signal in a lab where the owner is known.

## Open questions

- Watching for unexpected answers on the wire is tested in Stage 6.
- The fix on the client side (verifying the server's identity over HTTPS) is tested in Stage 6.
- DNSSEC (signed answers) is not built here.
