# Security writeups

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

## A note on scope

These are methodology and technique. They contain no organization-specific detail — no
tenant, domain, host, or person, and no unremediated finding. Where a number would
identify a real environment, it isn't here. Everything is generic or lab-reproducible by
design, so the writeups are useful to any practitioner and safe to publish.

## License

The prose in this repository is released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); any code snippets are MIT. See
[LICENSE](LICENSE).
