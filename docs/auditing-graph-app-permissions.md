# Auditing Microsoft Graph permissions on app registrations without losing your mind

Every Microsoft 365 tenant of any age has the same quiet problem: application
identities holding Graph permissions nobody remembers granting. A reporting
integration with tenant-wide `Mail.Read`. An automation app that can assign directory
roles. A vendor connector with `Directory.ReadWrite.All` that was set up once and never
revisited. None of it shows up in a daily alert, and none of it is anyone's job to
review — which is exactly why it accumulates.

This is a walk through how to audit that surface methodically, why the obvious ways of
looking at it mislead, and how to turn the result into something you can act on.

## Why the obvious approaches fall short

**The Entra portal, app by app.** You can open each app registration and read its API
permissions. In a tenant with a few dozen apps this is already tedious; past that it's
not happening on any schedule. Worse, the portal shows you permissions one app at a
time, which is the wrong axis. The question that matters isn't "what does this app
have" — it's "which apps can read all mail," and the portal can't answer that.

**Counting permissions.** A natural instinct is to rank apps by how many permissions
they hold. It's a bad proxy. An app with fifteen low-impact delegated scopes is less
dangerous than one with a single `RoleManagement.ReadWrite.Directory`. Volume is not
risk; capability is.

**Treating delegated and application permissions the same.** They are not the same, and
conflating them is the most common mistake. A *delegated* permission is exercised in the
context of a signed-in user and is bounded by what that user could already do. An
*application* permission acts with no user present and typically spans the whole tenant.
`Mail.Read` as a delegated scope reads the signed-in user's mail; as an application
permission it reads everyone's. Any audit that scores them equally is scoring the wrong
thing.

**Auditing app registrations but not service principals.** An app registration is the
definition; the service principal is the instance of it that actually exists in your
tenant and holds the granted permissions — and for multi-tenant and gallery apps, the
service principal can exist with consent granted even when there's no app registration in
your directory at all. An audit that enumerates only `/applications` misses every
third-party and gallery app a user or admin consented to. Both `/applications` and
`/servicePrincipals` have to be in scope.

## The approach

Three ideas make this tractable.

**1. Pivot on capability, not on the app.** Build the inventory so you can ask "which
identities can do X," where X is a risk category — read all mailboxes, write to the
directory, assign roles. That reframing is what turns a list into a prioritization.

**2. Put the risk judgment in data, not in code.** What counts as "critical" depends on
the organization. The risk level of each permission belongs in an editable catalog with
the *reason* attached, so the classification is transparent and adjustable rather than
buried in a script. A permission that isn't in the catalog should score as *unknown* —
weighted above "low," never ignored. What nobody has reviewed is uncertain, not safe.

**3. Fold in abandonment signals.** Over-privilege is worse when it's also unwatched.
A credential that's expired or never rotated, an app with no owner, an app with no
sign-in in a year — none of these add privilege, but each raises the odds that the
privilege is exploitable without anyone noticing. They belong in the score as
multipliers on top of the permission weight.

## Implementation

Enumerating the identities is a read-only Graph call:

```http
GET https://graph.microsoft.com/v1.0/applications?$select=id,appId,displayName,requiredResourceAccess,passwordCredentials,keyCredentials
```

`requiredResourceAccess` gives you the permissions, but as GUIDs against a resource
(Microsoft Graph's own service principal). To make them human-readable, resolve the
GUIDs once from Graph's service principal, which lists every app role and delegated
scope with its friendly name:

```http
GET /servicePrincipals?$filter=appId eq '00000003-0000-0000-c000-000000000000'&$select=appRoles,oauth2PermissionScopes
```

Build a `{ guid -> "Mail.Read" }` map from `appRoles` (application permissions) and
`oauth2PermissionScopes` (delegated). Now, for each app, split its `resourceAccess`
entries by `type`: `Role` is an application permission, `Scope` is delegated. Keeping
that distinction is the whole point.

A handful of application permissions deserve to dominate any risk ranking, because a
single one of them is enough to own the tenant. It's worth knowing them by sight:

| Permission | Why it's at the top |
|---|---|
| `Application.ReadWrite.All` | The app can grant *itself* any other permission — the key that opens every other door |
| `RoleManagement.ReadWrite.Directory` | Can assign privileged roles, up to Global Administrator |
| `Directory.ReadWrite.All` | Write access to users, groups, and roles across the directory |
| `Mail.ReadWrite` (application) | Read and modify every mailbox in the tenant |
| `AppRoleAssignment.ReadWrite.All` | Can assign app-role grants — a self-escalation path |

The common thread is that each is a path to *persistence and escalation*, not just data
access. An app with `Application.ReadWrite.All` that gets compromised doesn't need any
other permission — it grants itself whatever it's missing. These should sit at the top of
the report regardless of how many other permissions an app carries.

For the score, weight by risk level and lean application permissions heavier than
delegated:

```
base = Σ weight(application permission) + 0.5 · Σ weight(delegated permission)
```

Then apply abandonment multipliers — no owner, expired or expiring credential, stale
sign-in, over-long secret lifetime — as compounding factors. Owners come from
`GET /applications/{id}/owners`; credential dates are in `passwordCredentials` and
`keyCredentials`; sign-in activity, where your licensing allows it, from the service
principal sign-in activity report.

One deliberate choice at the tenant level: **sum the per-app scores, don't average
them.** Twenty medium-risk apps are a bigger problem than one, and an average would hide
exactly the sprawl you're trying to surface.

I built this out as an open tool —
[entra-privilege-auditor](https://github.com/earbona23/entra-privilege-auditor) — with
the risk catalog as an editable YAML file and console/JSON/HTML output. It runs on a
synthetic tenant with no credentials so you can see the shape of the report before
pointing it at anything real.

## Limitations

- **Permission risk is a judgment, not a fact.** A weight table is a starting point that
  every organization should adjust. Publishing the reasoning next to each level is what
  keeps it honest.
- **Sign-in activity depends on licensing.** Where the report isn't available, don't
  infer abandonment — a missing signal is not evidence of disuse. Stay silent rather than
  guess.
- **This finds over-privilege, not misuse.** An app *can* read all mail; whether it
  *does* is a different question that needs sign-in and audit-log analysis on top.
- **Consent history is out of scope here.** Knowing *when* and *by whom* a permission was
  granted is valuable and lives in the directory audit logs; this audit answers the
  standing-state question, not the how-did-we-get-here one.

## What I'd do differently starting over

I'd design for the *diff* from the first line, not as an afterthought. The single number
a tenant scores today is far less useful than what moved since last time: a new app with
critical permissions, a scope added to an existing one, a score that jumped. Recurring
audits live or die on signal-to-noise, and the only durable way to keep the noise down
is to report change, not state. Everything else — the catalog, the weights, the
exporters — is easier to get right than that.
