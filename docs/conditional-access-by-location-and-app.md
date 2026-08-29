# Designing Conditional Access by country, network, and application: the mistakes that cost the most

Conditional Access is the closest thing Entra ID has to a firewall for identity, and like
a firewall it's only as good as the exceptions you carve into it. The failures are rarely
a policy that's missing — they're a policy that looks right, blocks what you tested, and
leaves open the exact path an attacker takes. This is a walk through designing
location- and application-aware policies, and the specific mistakes that turn a
confident-looking rule set into a false sense of security.

## Why the obvious designs fail

**Blocking "risky countries."** The instinct is to build a named location of countries
you don't operate in and block sign-ins from them. It feels decisive and it stops almost
nothing that matters. Any attacker can route through a VPN or a residential proxy in a
country on your allow list; country blocking raises the cost of a spray campaign slightly
and does nothing against a targeted one. Worse, it creates a false sense of coverage —
"we block those countries" becomes a reason not to require the controls that actually
help.

**IP allow-lists as the primary gate.** Trusting a set of corporate egress IPs and
loosening controls from them was defensible when everyone was in an office. With remote
work and mobile, a policy that keys on network location either constantly blocks
legitimate users or gets so many exceptions that it stops meaning anything. Network
location is a useful *signal*; it is a poor *gate*.

**Leaving policies in report-only forever.** Report-only mode is the right way to
validate a policy before enforcing it — and it's where policies go to die. A rule created
"to turn on next week" that's still evaluating-but-not-enforcing months later protects
nothing while showing up in the portal as though the tenant is covered.

**The exclusions you forget you made.** Every policy needs a break-glass account excluded
so a misconfiguration can't lock you out of your own tenant. That exclusion is correct and
necessary — and it's also a standing hole. If the excluded account isn't tightly
controlled and monitored, you've built a bypass into every policy at once. The same goes
for the "temporary" user exclusion added during an incident and never removed.

**"All cloud apps" versus the app that isn't covered.** A policy targeting all cloud apps
seems comprehensive until you learn that some first-party surfaces and newly onboarded
apps don't fall under it the way you assumed. Targeting matters, and a gap in targeting is
invisible unless you go looking for it.

## The approach

**Location is a signal in a layered decision, never the whole decision.** The durable
pattern isn't "block these countries" — it's "require a compliant device and
phishing-resistant authentication, everywhere, and treat an unfamiliar location as a
reason to require *more*, not as the only thing you check." If location is one input among
device compliance and strong auth, spoofing it buys the attacker nothing on its own.

**Design from access patterns, not from block-lists.** Start by describing how each group
actually works — admins, standard staff, service accounts, external partners — and write
policies that fit those patterns. A block-list grows by accretion and nobody can say what
it protects; a small set of policies mapped to real personas is auditable.

**Break-glass excluded, but watched.** Keep the exclusion; remove the risk by making the
excluded accounts cloud-only, with long random credentials, strong auth, and an alert on
every single sign-in. An exclusion you monitor is a safety valve; one you forget is a
backdoor.

**Report-only is a stage, not a state.** Every report-only policy should have an owner and
a date. If it's been evaluating for a month, either enforce it or delete it — a policy
that never enforces is worse than none, because it reads as coverage.

## Implementation

A Conditional Access policy is JSON you can read and diff. Named locations are defined
once and referenced by policies:

```http
GET https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations
GET https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies
```

A policy that requires strong authentication and a compliant device for administrators —
the layered pattern, not a country block — has this shape in its conditions and grant
controls:

```jsonc
{
  "displayName": "Admins require phishing-resistant MFA + compliant device",
  "state": "enabled",
  "conditions": {
    "users":        { "includeRoles": ["<privileged role template ids>"],
                      "excludeUsers": ["<break-glass account ids>"] },
    "applications": { "includeApplications": ["All"] },
    "locations":    { "includeLocations": ["All"] }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": ["compliantDevice"],
    "authenticationStrength": { "id": "<phishing-resistant strength id>" }
  }
}
```

Note what the policy does *not* do: it doesn't block by country. It requires strong
controls everywhere and excludes only the monitored break-glass accounts. Location, where
you use it, belongs as an added condition that *tightens* — for example, requiring a
compliant device is non-negotiable, and an unfamiliar location additionally triggers
sign-in-risk controls under Identity Protection.

Finding the gaps is its own task, and it's worth automating because the mistakes above are
invisible in the portal's per-policy view. The three to hunt for are report-only policies
that have gone stale, disabled policies that read as coverage, and user exclusions that
need justifying. I built that check into an open PowerShell module —
[EntraHygiene](https://github.com/earbona23/EntraHygiene), `Get-EhConditionalAccessGap` —
which flags exactly those three across every policy at once:

```powershell
Get-EhConditionalAccessGap | Format-Table Politica, Brecha, Detalle
```

The point of automating it is that "we have a policy for that" and "the policy is enabled,
enforced, and has no forgotten exclusions" are different claims, and only the second one
protects you.

## Limitations

- **Conditional Access is not a web application firewall.** It gates authentication, not
  what an authenticated session does. It reduces who can get in; it doesn't inspect their
  traffic afterward.
- **Location is spoofable and should be treated that way.** Use it to raise assurance
  requirements, never as the sole control. A design that depends on location accuracy has
  a soft center.
- **The strongest controls need the right licensing.** Sign-in risk and user risk
  conditions require Identity Protection (Entra ID P2); authentication strengths and most
  policy features require at least P1. Design within what you actually license, and don't
  assume a P2 feature is enforcing if you're on P1.
- **Policies interact.** Two policies with overlapping targets and different grant
  controls can combine in ways that aren't obvious from reading either alone. Test the
  combination, not just each policy.

## What I'd do differently starting over

I'd write the personas before the policies. The tenants that end up with an unmanageable
pile of location rules got there by adding one policy per incident, each a reaction to the
last surprise. The ones that stay auditable started from a short description of how each
group works and derived a small set of policies from it — and then treated every
report-only policy as a task with a deadline, not a permanent fixture. The controls
themselves are the easy part; the discipline that keeps the exception list from becoming
the attack surface is the whole game.
