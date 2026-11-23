# 009 Multi-project: proposal

This document answers issue #7: the prototype configures one GitHub repository, so should it support several configured projects, or every project under the agent's GitHub account? It builds on [001 Domain model](001-domain.md), [002 Authorization](002-authorization.md), [003 Execution](003-execution.md) and [007 Architecture](007-architecture.md).

It is a design analysis, not a description of what this build does: [006 Deployment](006-deployment.md) documents the one-repository behaviour that ships today. Nothing here is implemented yet.

The short answer: support **an explicit list of repositories**, implemented as several Source instances of the `github` kind. "Every project of the account" is not a separate design; its only safe form is the same explicit list, optionally filled from a suggestion list. The cost is not the GitHub API. It is identity: today an Issue, a Thread, a read position, a reply and a pull request are all named by a bare number, and two repositories both have an issue #12.

## What "one project" means today

`Deployment` holds one `repository` and one GitHub account (`src/deployment.ts`). Everything downstream reads that scalar:

- `Commands` prefixes every API path with `repos/<repository>/` and refuses any other path (`src/infra/commands.ts`). This restriction is a guardrail against prompt-injected calls, not an OS boundary, but it is the only thing that keeps the agent's GitHub writes inside the project.
- The GitHub Source compares `notification.repository.full_name` with it, filters subject URLs through `githubAddress(url, repository)`, and polls only repository-wide streams of that repository (`src/sources/github.ts`).
- The execution environment exposes it as `DSH_SUPERAGENT_GITHUB_REPOSITORY`, and the documented prepare command clones from it (`docs/006-deployment.md`); the pull request head is `refs/pull/<number>/head` of that repository.
- Reply addresses carry `{ number, url }` and no repository; `sendGithub` re-derives the repository from the deployment (`src/sources/github-channel.ts`).
- Setup verifies this one repository, its push access, and writes one `gh` shim and one git credential helper for the account (`src/deployment.ts`, `src/infra/environment.ts`).
- The panel shows one repository row, and `SETUP_FIELDS` names setup-sensitive fields per Source kind (`src/app/views/settings.ts`, `src/deployment.ts`).

A Source *kind* is already described as reusable and a Source *instance* as one configuration of it ([001 Domain model](001-domain.md)). The implementation collapses the two: `SourceRegistry` keys stored instances by kind id, and `resolve` returns `{ id: kind.kind, kind: kind.kind, … }` (`src/sources/registry.ts`). One kind therefore means one instance, one card, one Policy part, one health entry and one polling loop.

## The two questions in issue #7

**Several configured projects.** The operator names the projects the agent works in. Today that works only as several deployments: one profile, state directory, panel and service process each (`docs/006-deployment.md`). It is isolated, it needs no code, and it costs the operator one process and one database per project.

**Every project of the account.** GitHub offers two different things under that name, and neither is "all projects" without a further rule:

- The account notifications feed, `GET /notifications`, reports threads the account participates in or watches: mentions, authored threads, comments, review requests, and everything in a watched repository (`reason: subscribed`). It does **not** report a new Issue or comment in a repository the account has push access to but does not watch and is not mentioned in. GitHub's reference lists these endpoints for classic personal access tokens only; the agent's `gh` OAuth token answered them in a spot check on 2026-11-23, but that must be checked against the agent credential before relying on this feed.
- Repository-wide observation, `GET /repos/{owner}/{repo}/…`, reports everything in a repository, but only in repositories someone enumerates. `GET /user/repos` can enumerate them, yet enumerating is not admitting: a new repository, a moved one, or one granted by mistake would enter the agent's scope silently.

The current Source already uses the repository-scoped notifications endpoint, because `Commands` rewrites `notifications` to `repos/<repository>/notifications`. Moving to the account feed would widen what arrives without changing what is authorized, and it would need a second, explicitly account-scoped call: relaxing `Commands` to pass unprefixed paths through would remove the repository confinement from every caller at once.

## What the API costs

The agent account gets 5,000 core requests per hour, shared with assessments, replies, pull request writes and every worker's `gh`. Conditional requests that answer `304 Not Modified` do not count against that limit.

A round of the current poller for one idle repository costs, at the 60-second default:

| Read | Requests | Conditional today |
| --- | --- | --- |
| notifications | 1 | no |
| new Issues and pull requests | 1 | yes, while the cursor does not move |
| conversation comments | 1 | yes, as above |
| inline review comments | 1 | yes, as above |
| default-branch CI | 4 | no |
| one open pull request with CI | 3 | no |
| two subscribed Threads | 2–8 | no |

Of those, the three repository-wide streams can answer `304` while nothing changes; the notification call and the CI reads always count. An idle project with full observation therefore counts about five requests a round and three more while one of its pull requests is open, so 300–480 requests an hour. Keeping the poller under a third of the 5,000-request budget allows three to five fully observed projects, and about 25 if only notifications are polled.

The largest cheap improvement is independent of this design: the notification call sends no `If-None-Match` or `If-Modified-Since`, although GitHub documents the endpoint as optimized for conditional polling. Measured against the live API on 2026-11-23, a request carrying the stored ETag answered `304 Not Modified` and left the rate-limit counter unchanged, whereas the unconditional call decremented it; both the repository-scoped and the account endpoint behaved this way. The code already reads the response headers for `X-Poll-Interval`, so storing the ETag and treating `304` as "nothing changed" is a small change at one call site.

## Where the cost is: identity

Issue and pull request numbers restart at 1 in every repository. Every name the prototype persists is derived from the number alone:

| Named by the number alone | Result with two repositories |
| --- | --- |
| `ThreadRef` and `threadKey` = `github:<number>` (`src/core/model.ts`, `src/sources/github-trigger.ts`) | both repositories' issue #12 are one Thread row, so a repo-B comment (whose Trigger id is repo-qualified) lands on repo-A's Thread, and its fan-out reaches repo-A's Tickets |
| Issue Trigger id `key('github-issue', String(number))`, pull request id the same | the second repository's issue is discarded as `duplicate` by `Intake.receive` |
| Default-branch CI Thread `ci:<branch>` and `key('github-ci', 'branch:main', …)` | two `main` branches share a Thread and a Trigger id |
| Pull request CI `key('github-ci', number, revision)` and `KV.github.ci(number)` | one repository's CI replaces the other's fingerprint; the Trigger ids collide |
| `KV.github.since`, `retries` (keyed by number), `cursor(number)`, `stream(id)`, `branchCi` (`src/kv.ts`) | read positions advance for whichever repository polled last |
| Channel address `{ number, url }` and `sendGithub`'s repository from the deployment | a reply to repository B's #12 is posted on repository A's #12 |
| `pullsOf`/`pullOf`, `claimPull` (compares `meta.number`), `observePull` (branch last segment) | a pull request in B can bind to A's ticket, and the one-open-pull-request-per-ticket check spans both |
| `find` maps any `#12` or any repository's URL to `github:12` | search resolves to the wrong Thread |
| `KV.sources.labels(source)` and the `source:<id>` settings block | one label map and one card for both |

Two more facts follow from that table. A repository must be part of the Trigger's `meta`, because the pull request routes, the Thread context reader and the execution environment all need it after intake. And a repository must be part of identity, not only of `meta`, because `Thread` and `Trigger` rows are keyed by it.

## Option A: several configured projects

**A1, several deployments.** Works today: one state directory, profile and panel per project. Full isolation and no code change; the operator pays one process, one hourly check and one panel per project. Keep it as the honest baseline, and as the answer for a project that needs its own accounts or its own OS user.

**A2, several Source instances of one kind.** One process, one panel, one policy; one `github` instance per repository. This is the design the domain model already describes, and it is what the rest of this document assumes. It requires:

1. **Instance identity.** A stored instance carries its kind; the first instance of a kind keeps the id `github`, so all persisted `source` strings, Thread keys, Policy parts, health names and KV keys stay valid. Further instances get ids such as `github-widgets`, and `Trigger.source` becomes the instance id.
2. **Repository-shaped identity.** Issue and Trigger ids include the repository (or the instance id); Thread refs do; `githubAddress` and `sendGithub` take the repository from the Trigger's channel; `find` resolves a URL through the instance whose repository it names.
3. **Per-instance deployment.** A resolved deployment per instance (repository, account, push access), a `Commands` bound to it, and execution variables whose values come from the ticket's own instance. The variable names stay as they are, so the documented prepare command keeps working for every repository.
4. **Per-instance polling and health.** One polling loop per instance, with its own cursors, failure counter, `X-Poll-Interval` and health entry; one repository's failure must not stop another's.
5. **Per-instance Policy.** The allowed list, bot list and admission rows live under the instance, not under the kind. Listing a person in one repository does not authorize them in another, which is what [002 Authorization](002-authorization.md) already implies: a Mandate records its Source, and Sources never share identities.
6. **Settings and panel.** A Source card per instance, with an add action that names the kind, and setup problems reported per instance.

**A3, one instance holding a list of repositories.** This saves the instance plumbing but not the identity work: every item in the list still needs its own Thread namespace, read position, health state and admission rows, or it collides exactly as above. It also makes one polling loop serve several repositories, so one slow or failing repository delays the rest. It is a worse shape for the same cost.

## Option B: every project of the account

B as stated should not be built. Its only safety property would be implicit scope: one credential, one allowed list keyed by Source, and workers that run `danger-full-access` as the operator's OS user (`docs/006-deployment.md`). The philosophy in [000 Philosophy](000-philosophy.md) asks to start narrow and widen deliberately; an implicit "any repository this account can push to" contradicts it. The concrete risks are the ones a scope bug produces: work in a repository the operator did not intend, a reply posted in the wrong repository, or a pull request opened against a fork.

The useful part of B is discovery, and it fits on top of A2:

- `GET /user/repos` suggests repositories the account can reach; the panel offers them.
- Only an explicit, enabled instance polls. A repository that is renamed, archived, deleted or no longer pushable stops being admitted and is named in its instance's health, rather than retried forever.
- Allow or deny decisions use the repository's numeric id, so a rename does not look like a new project.
- The account notifications feed, if used at all, is a trigger source only for admitted repositories; anything else it reports is recorded as filtered, not evaluated.

With that rule B becomes "A2 with a suggestion list", and it can be added after A2 without changing identity again.

## Recommendation

Do A2 in this order. Each step leaves the prototype working, and the first three are behaviour-preserving.

1. **Instance identity without behaviour change.** Add `kind` to stored instances and resolve `instance → kind` in the registry, settings view, history and Policy lookup. Migration writes the field; the `github` instance keeps every id, Thread key, health name and KV key it has today.
2. **Per-instance deployment and execution environment.** Resolve a deployment per instance, bind `Commands`, setup problems and execution-variable values to the ticket's instance. One repository configured still behaves exactly as now.
3. **Repository-shaped identity.** Put the repository into Trigger `meta`, Issue/PR/CI Trigger ids, Thread refs, channel addresses and the PR binding routes (`pullsOf`, `claimPull`, `observePull`, `pullEndNotices`), with a migration for existing `github:<number>` rows. Names stay byte-identical for the primary instance, so history is not re-ingested.
4. **Polling and health per instance.** One loop per configured instance, per-instance KV keys, failure counters and health entries.
5. **Settings, API and panel.** Add a Source instance of an installed kind, remove one, and show each instance's title, setup problem, counts and Policy part.
6. **Discovery, optionally.** Offer `GET /user/repos` as suggestions that an operator enables; never poll an unadmitted repository.

Between steps 2 and 3, make the notification call conditional. It is one call site, it is measured to cost nothing when nothing changed, and it applies whether or not multi-project is ever enabled.

Two limits stay explicit. Several GitHub accounts are out of scope: one `gh` shim and one credential helper serve the agent account, and per-account instances would need per-instance wrappers. And a project that needs its own account, OS user or schedule uses A1, a separate deployment, which this design does not replace.

## What each phase must keep

The rules [001 Domain model](001-domain.md) already states, restated where multi-project makes them easy to break:

- A Thread, a Trigger and a Channel belong to one Source instance; ids that drop the instance or repository are wrong even if they happen to be unique today.
- Admission and allowed lists are per instance. An accept in one repository grants nothing in another.
- Removing an instance stops its polling, but the store keeps its Threads, Tickets, queued Triggers and Mandates. Decide explicitly what removal does to them; the registry's `scope` answers `listened: true` for a source string with neither an installed kind nor a stored instance, so removal cannot rely on the gate to refuse that source's stored Triggers.
- A Ticket's execution environment describes the ticket's repository, not the deployment's.
- A failing repository is reported on its own instance and does not hold back the others.

## Verification

The test harness already stubs `gh` per repository (`tests/sources.spec.ts`), so the phases are checkable without live GitHub:

- Two repositories with the same issue number produce distinct Trigger ids and Thread keys, both ingested.
- A reply to one repository's Thread is posted through a path under that repository, and a path outside every configured repository is refused.
- A conditional notification request answers `304` and advances the cursor without counting a request.
- Per-repository cursors advance independently; a 404 in one instance leaves the other ready.
- The same login allowed in one instance is `not_a_member` in another; a Mandate accepted there is not revalidated by the other instance's list.
- A ticket from the second instance gets that repository's `DSH_SUPERAGENT_GITHUB_REPOSITORY` and binds only its own repository's pull request with the same number.
- A database migrated from schema 11 keeps every Thread, Trigger, ticket and read position of the primary instance unchanged.
