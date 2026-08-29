# Building Microsoft Graph automation with genuinely minimal permissions

Most of the over-privileged application identities you'll find in a tenant audit didn't
get that way through malice or even carelessness — they got that way because broad
permissions are the path of least resistance. `.ReadWrite.All` "to be safe," one app
reused for five jobs, a client secret because it's faster than a certificate. Every one of
those is a reasonable-in-the-moment choice that becomes someone's finding two years later.
This is how to build Graph automation that doesn't become that finding.

## Why the obvious approach over-grants

**"Request the broad permission to be safe."** When you're not sure exactly which
permission a call needs, the fast move is to grant the wide one — `Directory.ReadWrite.All`
covers a lot. It also means that if the automation is ever compromised, the attacker
inherits write access to the whole directory to do a job that only ever read one group.
The convenience is real and so is the blast radius.

**Reaching for application permissions by default.** Application (app-only) permissions
are tempting because they work unattended and aren't bounded by a user's access — which is
exactly why they're more dangerous. An app-only `Mail.Read` reads every mailbox in the
tenant. If the task genuinely acts on behalf of one user, a delegated permission is both
sufficient and far smaller in blast radius. The right default is the *narrowest model that
does the job*, not the most powerful.

**One app for everything.** A single "automation" app registration that accumulates the
union of every script's permissions is a single credential whose compromise hands over
everything at once, and whose actual required permissions no one can reconstruct. It's
convenient to set up once and impossible to reason about later.

**Secrets because they're quick.** A client secret is two clicks; a certificate is a few
more. So secrets proliferate, land in config files and CI logs, and never get rotated —
the leak that a secret scanner finds in commit history started as "I'll switch to a cert
later."

## The approach

**Least privilege per task, and prefer read.** Start from the specific calls the
automation makes and grant only the permissions those calls require — and if it only
reads, grant only read. The discipline is to derive permissions from the code, not to pick
a comfortable superset up front.

**Choose the smallest identity model.** Delegated vs application isn't about convenience,
it's about blast radius. If the task acts as a specific user or within a user's access,
delegated is smaller. If it must run unattended across the tenant, app-only is necessary —
so make the *grant* as narrow as the model is broad.

**One app per automation.** A separate app registration per job means each one's
permissions are exactly what that job needs, its credential's compromise is contained to
that job, and its required access is legible from its grant. The small extra setup buys
containment and auditability.

**Kill the secret.** Prefer a certificate over a client secret, and prefer workload
identity federation over both — federation lets a CI pipeline or a cloud workload
authenticate with a short-lived token from its own platform, so there's no long-lived
credential to leak or rotate at all.

## Implementation

The `.default` scope is where minimal-permissions discipline quietly dies. Requesting
`https://graph.microsoft.com/.default` grants the app *every* permission it has been
consented for — so `.default` is only as minimal as the app's configured permissions. Keep
the app's granted permissions tight and `.default` stays tight; the scope string isn't the
control, the app's grant is.

Finding the actual minimal permission for a call means reading the "Permissions" section of
that API's Graph documentation, which lists them from least to most privileged, and taking
the first one that applies. If a call works with `Group.Read.All`, don't grant
`Group.ReadWrite.All`. This is tedious and it is the whole job.

Authenticate with a certificate instead of a secret:

```powershell
# App-only, certificate-based — no secret anywhere.
Connect-MgGraph -TenantId $tenant -ClientId $appId -CertificateThumbprint $thumb
```

Better still, in a pipeline, use workload identity federation so there is no stored
credential at all — the platform (for example a GitHub Actions OIDC token) exchanges its
own short-lived token for a Graph token. The automation holds nothing that can leak.

And verify the result from the outside: the same audit that finds over-privileged apps is
the check that your own automation passes.
[entra-privilege-auditor](https://github.com/earbona23/entra-privilege-auditor) scores an
app by the risk of its granted permissions plus abandonment signals — run it against your
own automation's app registration and it should score low, with no credential-age or
ownerless flags. If your own tooling trips the auditor, that's the signal to narrow the
grant before someone else's audit finds it.

## Limitations

- **Some APIs only offer coarse permissions.** Not every operation has a narrowly scoped
  permission; occasionally the least-privilege option is still broad. When that happens,
  document why, and compensate with a certificate, a single-purpose app, and monitoring —
  don't pretend the grant is smaller than it is.
- **Resource-specific and RBAC scoping is uneven.** Mechanisms that limit an app to
  specific mailboxes, sites, or groups (RBAC for Applications, resource-specific consent)
  exist for some workloads and not others. Use them where available; don't assume they
  cover your case.
- **Federation isn't available everywhere.** Workload identity federation is ideal when
  the platform supports it. Where it doesn't, a certificate with a defined rotation is the
  fallback — a secret is the last resort, never the default.
- **Minimal today drifts tomorrow.** A permission that was right when the automation was
  written may become unused after a refactor. Least privilege is a state you maintain, not
  one you reach once.

## What I'd do differently starting over

Certificate or federated auth from the first line, and one app per automation from the
first automation. Both are marginally more work up front and enormously cheaper than the
alternative: the secret sprawl and the shared over-privileged "automation app" are the two
things I'd most want to never have to unwind. The permission-narrowing is ongoing
discipline, but the credential model and the one-app-per-job boundary are decisions you
make once, at the start, and can't cheaply reverse — so make them correctly while it's
free.
