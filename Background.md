## 1. Learning Objectives
### By the end of this module, students will be able to:
- Explain why DNSSEC was created and what security problem it solves (and does not solve).
- Describe the chain of trust from the root zone down to an individual DNS record.
- Distinguish the roles of the KSK, ZSK, DNSKEY, RRSIG, and DS records.
- Trace a DNSSEC validation and classify its outcome as Secure, Insecure, Bogus, or Indeterminate.
- Use command-line tools (delv, dig) to query DNSSEC status for real domains.
- Read a packet capture (pcap) of a DNSSEC-validating resolution and identify each query/response's role in the chain of trust.
- Explain authenticated denial of existence using NSEC and NSEC3.
## 2. Key Terms
- RRSIG (Resource Record Signature) — A digital signature covering an RRset, generated with a zone's private key. Resolvers use the corresponding public key (in a DNSKEY record) to verify it.
- RRset (Resource Record Set) — A collection of DNS records that share the same name, class, and type (e.g., all the A records for www.ietf.org). DNSSEC signs whole RRsets, not individual records.
- DNSKEY — A record that publishes a zone's public key(s). A zone typically publishes both its ZSK and KSK public keys in its DNSKEY RRset.
- DS (Delegation Signer) — A record kept in the parent zone containing a cryptographic hash of a child zone's KSK. The resolver hashes the child's published KSK and compares it against the parent's DS record to confirm the child zone is authentic.
- NSEC / NSEC3 — Records used for authenticated denial of existence — proving a name does not exist without needing a signature for every possible non-existent name. NSEC lists the next existing name in the zone (leaking zone contents); NSEC3 hashes owner names first, which makes zone-walking (enumerating all names) much harder.
- SK (Zone Signing Key) — A key pair used to sign the individual RRsets within a zone. Because it signs frequently, it is typically shorter-lived and rotated more often than the KSK.
- KSK (Key Signing Key) — A key pair whose private key signs only the zone's DNSKEY RRset (in effect, vouching for the ZSK). Its public key is hashed to create the DS record published in the parent zone.
- Parent Zone — The zone one level up in the DNS hierarchy that delegates a subdomain to another operator. For example, .org is the parent of ietf.org.
- Child Zone — A delegated subdomain that generates its own KSK/ZSK pair and provides a hashed KSK (DS record) to its parent for publication.
- Root DNS Name Servers — The thirteen logical name server clusters at the top of the DNS hierarchy. The root zone's KSK is the trust anchor for the entire DNSSEC system.
- Trust Anchor — A public key (or its hash) that a validator trusts as a starting point without needing to verify it against a parent, because it was configured/installed out-of-band. The root zone KSK is the trust anchor built into virtually every validating resolver.
- Validation States — The result a validating resolver assigns to a response: Secure (a full, verifiable chain of trust exists), Insecure (the zone is deliberately unsigned and the parent has no DS for it), Bogus (a signature failed validation — treated as an attack and the response is withheld), or Indeterminate (no trust anchor is configured to evaluate the chain).
- Zone Signing Ceremony — The formal, witnessed, and recorded event where the root zone's KSK signs the root DNSKEY RRset, held quarterly by ICANN with multiple trusted community representatives present.
- Algorithm Rollover / Key Rollover — The scheduled process of retiring an old signing key and introducing a new one without breaking validation for resolvers that cached the old key.
## 3. Background
DNS was designed in the 1980s, long before the internet reached its current scale and long before cache poisoning, spoofing, and man-in-the-middle attacks were serious considerations. A resolver has no built-in way to know whether a response it receives actually came from the authoritative server it queried, or from an attacker who intercepted or guessed the query and raced a forged answer back first (a classic example is the Kaminsky cache-poisoning attack of 2008).
DNSSEC addresses this gap. It is important to be precise about what it does and does not do:
- It DOES authenticate the origin of DNS data and verify that the data was not modified in transit.
- It DOES let a resolver detect if a response was forged or tampered with, and refuse to use it.
- It does NOT encrypt DNS traffic; queries and responses are still visible to anyone observing the network (that problem is instead addressed by separate protocols such as DNS-over-TLS or DNS-over-HTTPS).
- It does NOT protect availability; DNSSEC does not stop denial-of-service attacks against name servers.
## 4. How the Chain of Trust Works
Within a single zone, two key pairs cooperate:
- The ZSK's private key signs the zone's RRsets (A, AAAA, MX, etc.), producing RRSIG records.
- The KSK's private key signs the DNSKEY RRset itself (which contains the public ZSK and public KSK), producing an RRSIG over the keys.
- The zone's operator hashes the public KSK and gives that hash to the parent zone, which publishes it as a DS record and signs it with the parent's own ZSK.
- This repeats one level up at a time, child to parent to grandparent, until the resolver reaches the root zone.
- The root zone's KSK is self-signed. It cannot be verified by any parent, so it is instead trusted directly: it is the trust anchor, distributed out-of-band to every validating resolver and vouched for publicly through the DNSSEC Root Signing Ceremony.
A resolver validating a record, therefore, does not just check one signature; it walks the entire chain, verifying at every step that each zone's DS record hash matches the actual KSK the child zone presents, and that every RRSIG is valid and within its signature validity window.
