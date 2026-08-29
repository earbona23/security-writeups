# Rotating a production TLS certificate chain without a maintenance window

Certificate rotation is one of those tasks that is trivial ninety-nine times and a
production outage the hundredth. The difference is rarely the certificate itself — it's
an intermediate that didn't get bundled, a service that cached the old chain, or a
client that pinned something. This is a methodology for doing it on a live system with
no downtime, and, as much as the steps, the order of operations and how you validate
before you cut.

## Why the naive approach bites

**"Just replace the file and reload."** This works right up until the new certificate
was issued from a different intermediate than the old one. Browsers often paper over a
missing intermediate using AIA fetching or a cached copy, so it looks fine from your
laptop — and then a server-to-server client with a stricter chain builder fails, and you
find out from the API consumer, not from your own check. Validating from a
well-provisioned desktop is the single most common way a "successful" rotation turns
into an incident an hour later.

**Rotating everything at once.** If you swap the leaf, the intermediate bundle, and the
private key across every node simultaneously, you've also removed your ability to roll
back cleanly and to bisect what broke. On a live system you want the change to be
reversible at every step, and staged so a failure is contained to one node.

**Trusting the expiry date as the deadline.** The real deadline is often earlier than the
notAfter date: an intermediate in the chain may expire before the leaf, and clients that
build the full chain will fail at the intermediate's expiry, not the leaf's. Rotating
the leaf while leaving a soon-to-expire intermediate in the bundle solves the problem
you were looking at and not the one that will actually page you.

## The approach

Four principles keep this safe.

**1. Build and verify the full chain offline first.** Before anything touches
production, assemble leaf + intermediate(s) in the right order and verify it as a
strict client would — without leaning on any cached or AIA-fetched intermediate. The
chain has to stand on its own.

**2. Validate against the strictest client you have, not the friendliest.** A desktop
browser is the friendliest possible verifier. The clients that break are the strict
ones: server-side HTTP libraries, mutual-TLS peers, mobile apps with their own trust
store. Test against those explicitly.

**3. Stage the rollout so every step is reversible.** One node, or one node behind the
load balancer, before the fleet. Keep the previous certificate and key in place until
the new one is confirmed serving correctly. The ability to revert in seconds is worth
more than the few minutes a staged rollout costs.

**4. Cut over at the layer that actually terminates TLS.** Know where termination
happens — origin, reverse proxy, CDN edge — because that's the only place the swap
matters, and each layer may cache the chain differently.

## Implementation

Verify the assembled chain offline, in order, without network help:

```bash
# Leaf must come first, then each intermediate up the chain.
cat leaf.crt intermediate.crt > fullchain.pem

# Verify against the root explicitly, refusing to fetch missing intermediates.
openssl verify -CAfile root.crt -untrusted intermediate.crt leaf.crt
```

Confirm the leaf and the private key actually match before you deploy — a mismatched key
is a classic and avoidable outage:

```bash
diff <(openssl x509 -noout -modulus -in leaf.crt   | openssl md5) \
     <(openssl rsa  -noout -modulus -in leaf.key   | openssl md5)
# identical output = the key matches the certificate
```

Once it's serving, validate the chain *as presented by the server*, which is what
clients actually see — not what's in your files:

```bash
# -showcerts prints the full chain the server sends; check the order and that no
# intermediate is missing.
echo | openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts

# Check the expiry of every cert in the presented chain, not just the leaf.
echo | openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts 2>/dev/null \
  | awk '/BEGIN CERT/{c++} {print > "cert-" c ".pem"}'
for f in cert-*.pem; do echo "$f: $(openssl x509 -noout -enddate -in "$f")"; done
```

The `-servername` flag matters: on a host serving multiple certificates via SNI, omitting
it validates the wrong one and gives you a false green. Test with the SNI the real clients
send.

If you serve OCSP stapling, the stapled response is tied to the certificate, and a stale
staple can cause hard failures on clients that treat a missing-or-invalid staple as fatal.
Confirm the new certificate staples a valid response after the swap:

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com -status 2>/dev/null \
  | grep -A17 "OCSP response"
# "Cert Status: good" with a fresh thisUpdate/nextUpdate window = the staple is current
```

Know your termination layer before any of this, because it decides where the swap even
takes effect. If a CDN or a reverse proxy terminates TLS, the origin certificate is
irrelevant to what clients see — the edge presents its own, on its own cache schedule, and
"I replaced the file on the origin" changes nothing at the edge. If a load balancer
terminates, the certificate lives in the balancer's store, not on the nodes behind it.
Rotating the wrong layer is a quiet way to believe you've done the work while every client
still sees the old chain.

Order of operations, end to end:

1. Assemble and verify the chain offline (`openssl verify`, key-match check).
2. Deploy to one node; leave the rest on the old certificate.
3. Validate that node with `s_client` **and** with your strictest real client.
4. Roll to the fleet.
5. Only then retire the old certificate and key.

## Limitations

- **Certificate pinning breaks this model.** If a mobile app or partner pins the leaf or
  intermediate, a rotation requires a coordinated client update; no server-side care
  helps. Know whether anything pins before you start.
- **CDNs and reverse proxies cache chains** on their own schedule; a correct origin does
  not guarantee a correct edge. Validate at the edge the clients reach.
- **This covers rotation, not the compromise case.** If the private key may have leaked,
  rotation is necessary but not sufficient — the old certificate must also be revoked,
  and revocation has its own propagation delays.
- **Automate it before you rely on it.** A carefully hand-done rotation is repeatable
  exactly once. The steps here are the specification for the automation, not a substitute
  for it.

## What I'd do differently starting over

I'd measure the *presented* chain from outside continuously, not just at rotation time. Most
chain incidents are avoidable with a check that fails days ahead — an external probe that
verifies the full chain and every intermediate's expiry against a strict builder, on a
schedule, alerting well before the earliest notAfter. The rotation procedure matters, but
the thing that actually prevents the 3 a.m. page is knowing the chain is wrong while it's
still cheap to fix.
