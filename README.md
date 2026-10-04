# Security writeups

> **Snapshot, not maintained.** These writeups stand as published, but this repository is
> not under active development: I am not adding new material on a schedule. Issues and
> corrections are welcome and I do read them — a reply may take a while. Last substantive
> change: September 2026.
>
> Maintained instead: [revtriage](https://github.com/earbona23/revtriage),
> [entra-tripwire](https://github.com/earbona23/entra-tripwire),
> [entraform](https://github.com/earbona23/entraform) and
> [vantage](https://github.com/earbona23/vantage).

Technical writeups on Microsoft security engineering — identity, Microsoft Graph, and the
operational side of running a tenant. Written for other practitioners, not beginners.

Each piece follows the same shape: the problem as anyone in the role lives it, why the
obvious approaches fall short, the approach with the reasoning made explicit, real code or
queries you can use, an honest limitations section, and what I'd do differently starting
over.

## Writeups

- **[Auditing Microsoft Graph permissions on app registrations](docs/auditing-graph-app-permissions.md)**
  — finding over-privileged application identities in an Entra ID tenant, why counting
  permissions is the wrong measure, and how to score capability instead. Pairs with
  [entra-privilege-auditor](https://github.com/earbona23/entra-privilege-auditor).

- **[Rotating a production TLS certificate chain without a maintenance window](docs/zero-downtime-cert-rotation.md)**
  — the order of operations and the validation that keep a live cert rotation from
  becoming an outage, and why validating from a browser gives you a false green.

- **[Designing Conditional Access by country, network, and application](docs/conditional-access-by-location-and-app.md)**
  — why country blocks and IP allow-lists disappoint, the layered pattern that doesn't, and
  the forgotten exclusions that quietly become the attack surface.

- **[Reducing a tenant's exposure: what to measure, what to attack first, how to make it stick](docs/reducing-tenant-exposure.md)**
  — why chasing a secure-score number misleads, the blast-radius order to fix things in,
  and why drift detection is what makes hardening last.

- **[Building Microsoft Graph automation with genuinely minimal permissions](docs/graph-automation-minimal-permissions.md)**
  — where over-privileged app identities actually come from, least-privilege per task, and
  killing the client secret with certificates and workload identity federation.

## A note on scope

These are methodology and technique. They contain no organization-specific detail — no
tenant, domain, host, or person, and no unremediated finding. Where a number would
identify a real environment, it isn't here. Everything is generic or lab-reproducible by
design, so the writeups are useful to any practitioner and safe to publish.

## License

The prose in this repository is released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); any code snippets are MIT. See
[LICENSE](LICENSE).
