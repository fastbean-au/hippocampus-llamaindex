# Security policy

## Reporting a vulnerability

**Please report privately, not in a public issue.**

This repository is one piece of the [Hippocampus](https://github.com/fastbean-au/hippocampus)
family, and every advisory across the family is handled in **one place** — the service repository's
Security tab:

**[Report a vulnerability](https://github.com/fastbean-au/hippocampus/security/advisories/new)**

That opens a private advisory visible only to the maintainer. Say in the report that the finding is
in `hippocampus-llamaindex`. If private reporting is unavailable to you, open a public issue asking
for a private channel — **without any detail of the finding** — and you will be sent one.

Helpful things to include, in rough order of usefulness:

- The version or commit of this repository, and of the Hippocampus service it was run against.
- The relevant configuration, **with secrets redacted**.
- What an attacker gains, and what access they need to start.
- The smallest reproduction you have.

This is maintained by one person, in their own time. There is no response-time commitment. You will
get an acknowledgement, an assessment of whether it is in scope, and — where a fix is warranted — a
release and a credit unless you would rather not be named. Please give a fix a reasonable window
before disclosing publicly.

## Supported versions

Pre-1.0, like the service. Fixes land on `main` and ship in the next release; there are no
backported patch branches, so **the latest release is the supported one**.

## Scope

**In scope** — the adapter (`llama_index/memory/hippocampus/`) — how it authenticates to a
Hippocampus instance, what it puts into a memory, and what it returns into an LLM's context — and
the published `llama-index-memory-hippocampus` distribution.

**Out of scope** — the Hippocampus service itself, which has its own
[policy](https://github.com/fastbean-au/hippocampus/blob/main/SECURITY.md) and is where a finding in
the store, the API, or the auth package belongs; and the public demo at `hippocampus-demo.com`,
which exists to be poked at and runs generated data on deliberately unrealistic decay clocks.

## Worth knowing before reporting

This adapter puts stored memories **into a model's context**. A store holding records from more than
one trust domain will therefore surface them together unless the tokens it is given are
group-scoped, and group scoping is a [soft
partition](https://github.com/fastbean-au/hippocampus/blob/main/docs/security.md#group-scoping-and-the-trust-boundary)
rather than isolation. That is documented behaviour of the service, not a fault in the adapter.
