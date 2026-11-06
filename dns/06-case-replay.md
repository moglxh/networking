# Stage 6: Replaying the Sogou DNS hijack in a lab

**Lab:** Kali Linux VM, my own NSD (authoritative) and Unbound (resolver), all on the loopback interface. No malware: the "payload" is a one-line text file.
**Date:** 2026-11-06
**Source case:** "China-Aligned Hackers Hijack Sogou Input Method DNS Requests to Deploy DarkVigil Backdoor", The Hacker News (Nov 2026): https://thehackernews.com/2026/11/china-aligned-hackers-hijack-sogou.html

## The case in one paragraph

As reported, a component of the Sogou Input Method for Windows fetched a configuration over plain HTTP. By hijacking DNS requests for that host, attackers had the client download from a server they controlled, which delivered the DarkVigil backdoor. The report does not say how the DNS tampering was achieved, so this lab models the **effect** (the client's resolver returns the attackers' address) and not the method.

## What was replicated and what was not

| Real case | Lab stand-in |
|---|---|
| Client fetching a config over HTTP | `curl` fetching `config.lab.test` by name |
| The legitimate server | Harmless web server on `192.0.2.20` |
| The attackers' server | Harmless web server on `192.0.2.99` (serves a marker text) |
| The resolver the victim used | Unbound, with an answer changed by one config line |
| The owner of the name | NSD serving the `lab.test` zone |
| The backdoor | Not replicated. Out of scope. |

The documentation addresses (`192.0.2.x`) were added to the loopback interface for the test and removed afterwards, so nothing leaves the VM.

## Run and results

`lab/stage6.sh` builds the setup, runs the attack and the defences, and cleans up after itself.

| Step | What ran | Result |
|---|---|---|
| 1 | Name lookup (baseline) | `192.0.2.20` |
| 2 | Plain HTTP fetch (baseline) | `config=LEGIT` |
| 3 | HTTPS fetch, certificate checked (baseline) | `config=LEGIT` |
| Tamper | Resolver told to answer `192.0.2.99` for `config.lab.test` | Lookup now returns `192.0.2.99` |
| 4 | Plain HTTP fetch | **`config=TAMPERED`**: the client accepted the rogue server's file |
| 5 | HTTPS fetch, certificate checked | **Rejected** with `curl: (60)` certificate error; no file accepted |
| 6 | `dns-check.sh` (resolver vs owner) | `MISMATCH` |
| 7 | Capture of DNS replies, filtered for unexpected answers | Showed the `192.0.2.99` answer |

## Findings

- **Red:** with plain HTTP, redirecting DNS was enough to make the client take a file from the wrong server, and nothing on the client side noticed.
- **Blue, detection:** comparing the resolver with the owner and watching replies on the wire both caught the tampering. Both need a known-good reference.
- **Fix:** checking the server's identity over HTTPS stopped the same redirect, even with DNS fully compromised. (I do not know the details of the vendor's patch; this is the standard defence for this kind of flaw.)

## Limits of this lab

- The HTTPS client trusted one specific certificate (`--cacert`). A real client trusts a set of certificate authorities, so the defence there depends on that trust model.
- DNSSEC was not built.
- This models the resolver-side effect only; how the real attackers tampered with DNS is not known from the source.

## What I learned

- DNS tells a client where to go but cannot tell it whether the destination is genuine; that check has to happen at the application layer.
- A tampered answer is invisible to the client unless it verifies the server.
- Detection works best by comparing an answer against a trusted reference.

## Open questions

- How were the real DNS requests hijacked? The source does not say.
- How much of this would DNSSEC have prevented on its own?
