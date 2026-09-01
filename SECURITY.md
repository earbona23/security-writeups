# Security policy

## Reporting a vulnerability

Open a [private security advisory](https://github.com/earbona23/security-writeups/security/advisories/new)
on this repository. Please do not open a public issue for a vulnerability.

You will get an acknowledgement within 72 hours and an assessment within seven days. There
is no bounty programme — this is a single-maintainer project — but every report is credited
in the advisory unless you ask me not to.

## What counts as a vulnerability here

This repository contains technical writing and example code on Microsoft security
engineering. It is prose and snippets, not a running tool, so its risks are different.

| Class | Why it matters |
|---|---|
| **Organisation-specific detail** | Nothing here should identify a real tenant, user, host, address or configuration. If you find something that does, report it privately and it will be removed — do not open a public issue quoting it. |
| **A code example that is unsafe to copy** | These snippets get pasted into real environments. An example that requests excessive permission, mishandles a token, or performs an unintended write is a defect worth reporting. |
| **Guidance that is wrong in a way that weakens posture** | Advice that would leave a reader less secure than before they read it. Being wrong in prose still ends up in someone's tenant. |
| **A secret or key material in an example** | Even an expired or fictional one, if it is shaped like the real thing. |

## Out of scope

- Disagreement about approach or style. Open a normal issue; that is a conversation worth
  having in public.
- Requests for coverage of new topics — a feature request.
