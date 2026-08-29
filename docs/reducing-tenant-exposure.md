# Reducing a tenant's exposure: what to measure, what to attack first, how to make it stick

"Harden the tenant" is an unbounded task, and unbounded tasks either never start or turn
into a checklist someone runs once and never again. The useful version of the question is
narrower: given a live Microsoft 365 tenant, what do you measure, in what order do you fix
things, and how do you keep it from drifting back. This is a way to think about that
without chasing a dashboard number or trying to do everything at once.

## Why the obvious approaches fail

**Chasing a secure-score number.** Secure Score is a helpful inventory and a terrible
target. It weights recommendations by a generic model, not by your tenant's actual blast
radius, and optimizing the number pulls effort toward whatever is cheapest to toggle
rather than whatever most reduces risk. A tenant can climb the score while leaving a
permanent Global Administrator with password-only sign-in — the single worst thing in the
environment — untouched, because fixing it is "one point."

**Doing everything at once.** A hardening sprint that changes legacy authentication,
device compliance, admin roles, and app permissions in the same week is a sprint that
can't tell which change broke the helpdesk's integration, and can't roll back cleanly. On
a live tenant, order and reversibility matter as much as the changes themselves.

**One-time hardening.** The most common failure isn't doing the wrong things — it's doing
the right things once. Permissions get re-granted, new apps arrive, an exclusion added
during an incident stays, a new admin is created permanently "just for now." A tenant
that was hardened a year ago and never re-measured is an unknown tenant.

## The approach

**Measure the standing attack paths, identity first.** Exposure in a cloud tenant is
mostly about identity: who and what can authenticate, with how much privilege, behind how
weak a barrier. Before touching anything, enumerate the things an attacker would actually
use — privileged accounts and how they authenticate, application identities and their
permissions, whatever still allows legacy authentication, and devices that don't meet
policy. That inventory, not a score, is your baseline.

**Attack in order of blast radius.** Fix the things whose compromise gives the most, in
that order:

1. **Privileged identity.** Enforce phishing-resistant MFA on every admin, and convert
   permanent role assignments to eligible (PIM) so standing admin access shrinks to
   near-zero. A compromised admin is game over; this is always first.
2. **Legacy authentication.** Protocols that can't do modern auth bypass Conditional
   Access and MFA entirely. Blocking them removes an entire class of credential-stuffing
   path in one move.
3. **Application permissions.** The app identities holding tenant-wide permissions are the
   non-human equivalent of over-privileged admins, and they're the ones nobody watches.
   Rank them by capability and cut what isn't used.
4. **Device and session.** Require compliant or hybrid-joined devices for sensitive
   access, so a valid credential alone isn't enough.
5. **Data-layer controls.** Sharing defaults, external collaboration, sensitivity — real,
   but they matter less if the four above are open.

The order isn't arbitrary: each step assumes the ones above it are done, because there's
no point tightening data sharing while a password-only Global Admin exists.

**Sustain with drift detection, not re-audits.** The only version of hardening that holds
is one where you can see change. Snapshot the risky state — privileged assignments, app
permissions, policy posture — and compare each run against the last, so a new permanent
admin or a freshly granted app permission surfaces as a *diff*, not as something you might
notice on the next full review. Point-in-time hardening decays; change detection is what
makes it stick.

## Implementation

The first moves, concretely, and each one is measurable before and after.

**Privileged identity.** Find the admins without strong auth and the permanent
assignments, then fix and re-measure. An open PowerShell module,
[EntraHygiene](https://github.com/earbona23/EntraHygiene), gives you both:

```powershell
Get-EhPrivilegedWithoutMfa                        # admins behind a weak barrier
Get-EhRoleAssignment | Where-Object Tipo -eq 'Permanente'   # standing admin access
```

The goal is zero rows in the first and as few as possible in the second — permanent
assignments converted to eligible so activation is time-bound and logged.

**Legacy authentication.** Measure it from sign-in logs (filter for legacy client apps),
then block it with a Conditional Access policy and confirm the legacy sign-ins stop. This
is the single highest-leverage block in most tenants.

**Application permissions.** Rank app identities by capability, not count, and prune. I
built an open auditor for exactly this —
[entra-privilege-auditor](https://github.com/earbona23/entra-privilege-auditor) — which
scores each app by the risk of its granted permissions plus abandonment signals, and
supports a diff against a previous run so you can watch the number come down and catch new
grants:

```bash
python -m auditor.cli --live --formato json --salida today.json
python -m auditor.cli --diff last-month.json     # what changed since last time
```

That diff is the sustaining mechanism made concrete: the same tool that measures the
baseline is the one that tells you when it moved.

## Limitations

- **Measurement is relative, and that's fine.** You can't reduce exposure to zero, and a
  score that claims a number is selling certainty that doesn't exist. Report change — "from
  four password-only admins to zero," "from eleven high-risk app grants to three" — not an
  absolute grade.
- **You can't measure what you can't see.** Sign-in activity, risk signals, and some
  reports depend on licensing. Where a signal isn't available, don't infer it's clean;
  record that it's unmeasured.
- **Blast-radius order is a default, not a law.** A tenant with a specific data-exfiltration
  concern may reasonably pull the data layer forward. The point is to attack in a
  justified order, not to follow this list dogmatically.
- **Hardening changes are still changes.** Blocking legacy auth or requiring compliant
  devices will break something that quietly relied on the old behavior. Stage each change
  and keep the ability to roll back.

## What I'd do differently starting over

I'd instrument drift on day one, before fixing anything. The temptation is to spend the
first week making changes, because changes feel like progress. But the baseline snapshot
and the diff mechanism are what turn a one-time cleanup into a posture that lasts, and
they're cheapest to build before the environment is in motion. Fixing the four
blast-radius layers matters — but the thing that decides whether the tenant is still
hardened a year from now is whether you can see it drift while it's still one row in a
diff.
