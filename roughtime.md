# Roughtime

* [Link](https://roughtime.googlesource.com/roughtime)

## Overview

### Secure, Rough Time
* How do we prove that information is "fresh"?
* Two common solutions are strictly increasing nonces and synchronized time.
* Nonces are "interactive" so they are out for now.
	* TODO(kkl): The definition of "interactive" here is not 100% clear to me.
* Our current NTP-based infrastructure is unauthenticated so a man-in-the-middle attacker could manipulate network time using NTP.
* Several security controls depend on correct time (e.g. Certificate expiration, OCSP responses)
* To compensate, systems usually use heuristic protections (e.g. infrequent NTP updates, disallowing large NTP shifts in time)

### So, Secure NTP?
* No. Proposals to retro-fit authentication on NTP are dated.

### Roughtime
* "Roughtime is a protocol that aims to achieve rough time synchronisation in a
  secure way that doesn't depend on any particular time server, and in such a
  way that, if a time server does misbehave, clients end up with cryptographic
  proof of it."
* "Rough" here means the current design doesn't need super exact time synchronization.
* Fun Fact: "... about 25% of certificate errors shown by Chrome appear to be caused by bad local clocks..."
* Distinction between authenticated NTP schemes and Roughtime:
	* Secure NTP schemes use symmetric key negotiation
	* Roughtime uses strictly asymmetric signatures
* Why strictly asymmetric signatures?
	* Proof that server signed the response (and not the client)
* High-Level Design
	* A complete fresh client sends random nonce to a Roughtime server.
	* The response includes current time, the nonce, and a signature of both.
	* If the client completely trusts the server, thats it!
	* Otherwise, the client can mix the previous response into a hash and use that as a nonce sent to another server.
		* Mixing the signature in creates a strict ordering of messages!
* What if the servers gave vastly different responses?
	* If the second server gave a time *before* the first server, we have proof of misbehavior due to the hash mixing thing mentioned before.
	* If the second server gave a time *after* the first server, the client can contact the first server again and get a signature from the first server of an earlier timestamp.
* In the two server scenario differing responses just proves misbehavior. With more servers, it becomes easy to identify misbehavior *and* get accurate time responses.

### Signing many requests
* Using fast primitives (Ed25519) and batching requests in Merkle trees will likely allow Roughtime to be performant despite being using asymmetric primitives for signatures.
