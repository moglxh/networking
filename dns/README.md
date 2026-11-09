# DNS

Hands-on notes on how DNS works, how it is attacked, and how tampering can be detected, done in an isolated lab (a Kali Linux VM) and written up after each stage. The capstone replays a real-world case: the Sogou Input Method DNS hijack.

| Stage | Topic | Notes |
|---|---|---|
| 1 | Observing DNS on the wire | [01-observing-dns.md](01-observing-dns.md) |
| 2 | How a Linux client decides where to look up a name | [02-resolution-on-linux.md](02-resolution-on-linux.md) |
| 3 | Running my own authoritative server and resolver | [03-own-resolver.md](03-own-resolver.md) |
| 4 | Tampering with DNS answers | [04-tampering.md](04-tampering.md) |
| 5 | Detecting tampered DNS answers | [05-detection.md](05-detection.md) |
| 6 | Replaying the Sogou DNS hijack in a lab | [06-case-replay.md](06-case-replay.md) |
| 7 | Closing up the case with final exfiltration results.
Scripts are in [`lab/`](lab/).

## Conventions

- Lab only. Nothing here is run against networks or systems I don't own. The case replay uses no malware.
- Outputs are sanitized: no hostnames, usernames, or public addresses.
- Each note covers: goal, what I observed, what I learned, open questions. Where something is unexplained, I say so.
