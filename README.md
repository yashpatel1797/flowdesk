# FlowDesk: build and learning plan

FlowDesk accepts a support event, executes a versioned workflow, and helps an operator inspect and recover failed work. The goal is to understand and demonstrate backend correctness, then add a small full-stack interface.

The supplied FlowDesk specification defines the destination. The learning approach requested in chat controls the pace: implement one small concept, explain it, and pause before the next implementation. The specification's request to implement all of M1 immediately is deliberately split into lessons.

## How we will work

- One small implementation per turn; stop after its explanation. Continue when the learner replies to continue or asks for the next step.
- Prefer one concept and roughly one to three small implementation files per step. A larger change must be split where practical. This planning document is separate from implementation code.
- For each step, explain the problem, prerequisite fundamentals, every new statement or configuration field, the caller and execution flow, the architecture tradeoff, and a relevant failure case.
- Show the command or request that verifies the change, its observed result, and what remains unfinished.
- Include relevant interview questions with reasoned answers, plus one short prediction or practice exercise. Exercises are optional, not a condition for continuing.
- Introduce concepts just before using them. Keep the learning path connected to the support-request example.
- Use everyday language first, define technical words when they appear, and end each explanation with a short plain-language recap.
- Inspect relevant official documentation when selecting dependencies. Pin a compatible stable version set, commit the lockfile when Git is initialized, and record exact versions here as tools are added.
- Keep a learner-maintained time log of focused sessions. Do not infer study time from automated execution time.

## Roadmap

Each numbered item is a small lesson target, not an instruction to implement the entire list now. Split an item further when its code or prerequisite explanation becomes too large.

### Foundation

1. Create the private npm workspace and API package boundary. Learn package.json, package names, workspace discovery, and monorepo versus process boundaries.
2. Select and verify a supported Node.js LTS baseline and npm setup. Learn runtime versus package manager, versions, and reproducible environments. Obtain approval if a system installation is needed.
3. Add pinned TypeScript development tools and a lockfile. Learn dependencies versus development dependencies and install versus execution.
4. Add strict TypeScript configuration and a tiny executable entry point. Learn compilation, types, modules, and development versus production commands.
5. Create the Express application separately from the listening server. Learn HTTP, ports, request/response, imports, and function calls.
6. Add GET /health/live. Trace one request from the client through the route to the response; distinguish process liveness from dependency readiness.
7. Add configuration validation. Learn environment variables, startup failures, secrets, and why configuration belongs at the application boundary.
8. Add request IDs and structured request logs. Learn middleware order, correlation, and avoiding credential/payload leaks.
9. Add a consistent error response and Zod request validation. Learn unknown input, runtime validation versus TypeScript, and HTTP error semantics.

### M1: durable, authorized run submission

10. Add PostgreSQL through Docker Compose. Learn containers, ports, volumes, connections, and persistent storage. Verify required local tools first.
11. Add Prisma, the first tenant migration, and database readiness. Learn relational tables, primary keys, migrations, generated clients, and GET /health/ready semantics.
12. Add tenant-scoped API keys and seed two tenants. Learn random credentials, hashing, secret display, revocation, and safe seed reruns.
13. Authenticate requests and resolve tenant identity from the credential. Learn authentication versus authorization and why a request-supplied tenant ID is untrusted.
14. Add admin/viewer authorization with focused real-database checks. Learn least privilege and consistent responses for inaccessible resources.
15. Define the JSON workflow contract and allowlisted steps. Learn discriminated unions, schema versions, and runtime validation; start with sequential execution.
16. Add workflow drafts and POST /v1/workflows. Learn route, service, and database responsibilities through the actual call chain.
17. Add immutable workflow versions and POST /v1/workflows/:id/versions. Learn uniqueness, concurrent publishing, and why a run must retain its original definition.
18. Add the run schema, statuses, and tenant-consistent relationships. Learn foreign keys, composite constraints, indexes, and why copied tenant IDs alone do not enforce isolation.
19. Add input canonicalization and hashing. Define exactly which submitted fields participate in idempotency, including the selected workflow version. Learn JSON ordering and semantic equality.
20. Add POST /v1/runs with an idempotency key. Use the database uniqueness constraint and explicit transactions; explain duplicate requests and changed-payload conflicts.
21. Verify concurrent duplicates, conflicts, and tenant isolation against real PostgreSQL. Runs remain pending until M2; this checkpoint does not claim execution works.

### M2: execution and recovery

22. Add a run event record and sequence allocation. Learn durable history, atomic updates, and concurrent writers.
23. Add an outbox entry in the same transaction as run creation. Learn the database/queue dual-write problem and what atomicity actually covers.
24. Add Redis and BullMQ with a separate dispatcher process. Learn producers, consumers, at-least-once delivery, leases, and bounded dispatch batches.
25. Add a separate worker and safe run/step claiming. Explain locks, ownership, duplicate delivery, and what happens when a worker dies or loses its lease.
26. Persist step attempts and execute request normalization. Learn state transitions, attempt identity, saved output, and resume decisions.
27. Add condition evaluation and the three seeded workflow templates. Start with sequences; any supported branch must be bounded, validated, and cycle-free.
28. Add a controlled mock HTTP integration and an idempotent receiver. Learn network calls, side-effect identity, and why submission deduplication is a different guarantee.
29. Add timeouts and a deterministic failure simulator. Learn cancellation limits, HTTP 429/500, uncertain outcomes, and repeatable experiments.
30. Add bounded retries, backoff, and exhausted states. Coordinate queue retry policy with persisted attempt state and distinguish transient from permanent failures.
31. Add explicit admin retry with documented resume semantics. Learn which work may repeat, which work must be skipped, and why retry is a business operation.
32. Test crash recovery and redispatch. Kill the API after commit and the worker after the side effect; demonstrate receiver deduplication and document remaining failure windows and queue retention.

### M3: inspect, integrate, and measure

33. Add tenant-scoped run detail and a cursor-paginated run list with status filters. Learn stable ordering, cursor encoding, supporting indexes, and pagination tests.
34. Add reconnectable SSE with durable event IDs and Last-Event-ID. Learn streaming HTTP, replay, deduplication, retention, and reconnect races.
35. Publish the OpenAPI contract with executable seeded-local curl examples. Learn API contracts and how errors/authentication/idempotency appear to callers.
36. Add a small typed SDK with cancellation and timeout behavior. Learn runtime boundaries and when client retries need idempotency. Publish only after the API stabilizes.
37. Add the Next.js console shell and safe server-side handling of API credentials. Learn browser/server boundaries; never put administrative secrets in a public client bundle.
38. Add the run list and filters using TanStack Query. Learn server state, cache keys, loading/errors, and tenant-aware cache boundaries.
39. Add run detail, attempts, input/output/errors, timeline, and admin retry. Connect SSE to the UI without duplicate displayed events; add relevant React Testing Library coverage.
40. Verify the critical browser journey with Cypress. Submit duplicates, force a failure, inspect its history, restore the integration, and recover.
41. Benchmark 100 synthetic runs with 10 concurrent submitters first. Record fixture, revision, hardware, worker count, queue delay, throughput, and API p50/p95; separate acceptance from execution latency.

### M4: delivery and interview evidence

42. Add GitHub Actions for the checks built so far. Learn CI services, migrations in test environments, and handling secrets.
43. Deploy one documented environment and verify migrations, readiness, graceful shutdown, credentials, and a smoke journey. Choose hosting when the core works; publishing requires the appropriate account access.
44. Finish architecture decisions, limitations, deployment instructions, and the demo script. Audit every acceptance criterion from the specification.
45. Prepare an interview walkthrough and evidence-based resume bullets. Use actual measurements and tested recovery cases; explain remaining risks honestly.

## Destination architecture

```text
Console / SDK / curl
        |
        v
Express API --> PostgreSQL: runs + outbox in one transaction
                         |
                  outbox dispatcher
                         |
                         v
                    BullMQ / Redis
                         |
                         v
                       worker --> controlled integration
                         |
                         v
              PostgreSQL: attempts + events
                         |
                         v
                   SSE --> console
```

PostgreSQL owns business state. Redis transports work. The API, dispatcher, and worker will be separate processes in one repository. This does not require separate deployments or constitute a microservice platform.

The first reliable vertical path is: authenticated request -> validated workflow version -> durable run -> repeatable response for duplicate submissions. Async execution follows after this path is verified.

## Current checkpoint: lesson 2 complete

The npm workspace is configured. The project-local Node.js 24.22.1 runtime and its bundled npm 11.19.1 are downloaded and verified. There is no server, database, installed application dependency, or generated lockfile yet.

```text
flowdesk/
  README.md              learning plan and checkpoints
  package.json           private workspace root
  .nvmrc                 exact Node.js version for compatible version managers
  .gitignore             excludes local downloaded tools from future Git tracking
  apps/
    api/
      package.json       private API package, using ES modules
```

`apps/*` will discover runnable application packages. `packages/*` reserves a convention for reusable packages such as the future SDK; that directory is created only when needed.

From this directory, inspect the configuration without installing anything:

```powershell
npm pkg get name private workspaces
npm pkg get name private type --workspace @flowdesk/api
```

Expected: the root is private and declares both workspace patterns; npm resolves `@flowdesk/api` and reports `type` as `module`.

## Lesson 2: a consistent Node.js environment

JavaScript is our programming language. Node.js is the program that executes our backend JavaScript outside a browser. npm downloads project tools/libraries and runs commands declared in package.json. Node.js and npm are different tools with different version numbers.

The system terminal initially reported Node.js v20.12.0 and npm 10.5.0. FlowDesk selects Node.js 24.22.1 LTS instead. LTS means Long-Term Support: a release line maintained for an extended period. This is a supported baseline, not a claim to always use the newest release.

Official references checked on 2026-10-29:

- [Node.js 24.22.1 LTS release and download checksums](https://nodejs.org/en/blog/release/v24.22.1)
- [Node.js release support status](https://nodejs.org/en/about/previous-releases)
- [npm engines configuration](https://docs.npmjs.com/cli/v11/configuring-npm/package-json/#engines)

The root package.json now declares:

```json
"engines": {
  "node": ">=24.22.1 <25"
}
```

- `engines` describes the runtime versions this project targets.
- `node` identifies the runtime being described.
- `>=24.22.1` allows that release or a newer one.
- `<25` keeps the allowed releases within major version 24.
- Together they allow later 24.x updates, but reject 20.x and 25.x as supported project configurations.

This declaration does not install or switch Node.js. npm normally warns about incompatible engines; this field alone does not force every command to stop. The .nvmrc file records the exact baseline, 24.22.1, for version managers that support that file. It also does not switch Node by itself; no compatible manager was found in this environment.

For this Windows x64 machine, use an official portable Node.js archive under `.tools/node-v24.22.1-win-x64/`. The download must match its official SHA-256 checksum before extraction. Think of this check as comparing the downloaded file's fingerprint with the publisher's fingerprint. The .gitignore entry `.tools/` keeps these downloaded tools out of future source commits; each developer supplies their own runtime.

From the FlowDesk root, select the verified portable runtime for the current PowerShell terminal:

```powershell
$flowdeskNodeDirectory = (Resolve-Path '.\.tools\node-v24.22.1-win-x64').Path
$env:Path = "$flowdeskNodeDirectory;$env:Path"
node --version
npm.cmd --version
```

The first line gets the full directory path. The second puts it first in the current terminal's search list for programs, so `node` and `npm` use this local copy. This setting lasts for that terminal session; repeat it in a new terminal. The system-wide installation is unchanged. `npm.cmd` explicitly uses npm's Windows command launcher and avoids PowerShell script execution-policy issues.

Verification completed: the archive matched the official SHA-256 checksum, Node reported `v24.22.1`, npm reported `11.19.1`, npm read the expected engines range, and npm resolved `@flowdesk/api` using this runtime. The project configuration files were also checked for the expected values.

Execution flow: PowerShell finds node.exe using PATH -> Node executes the JavaScript you ask it to run. When you invoke npm, its command launcher uses Node to execute npm's JavaScript. Our API code will enter this flow in a later lesson.

Why this matters: a teammate or deployment machine should use a known runtime baseline. Otherwise a feature can work on one machine and fail on another before our application logic even runs.

Interview practice:

- **How are Node.js and npm different?** Node executes JavaScript. npm manages packages and runs project commands, using Node itself.
- **Does engines install the required runtime?** No. It records compatibility; installation or version selection is a separate action.
- **Why record an exact version and a range?** The exact version gives a reproducible starting point. The range states which updates we intend to support; updates still need verification.

Prediction exercise: does Node 25.0.0 satisfy `>=24.22.1 <25`? No: it fails the upper bound.

Plain-language recap: we chose the program version that will run FlowDesk and wrote the choice down so other machines can follow it. We have not started writing the API yet.

## Session log

| Session | Lesson | Focused study time | Questions / follow-up |
| --- | --- | --- | --- |
| 1 | Workspace boundary and roadmap | Learner to record | Next: Node.js baseline |
| 2 | Runtime baseline and local tool setup | Learner to record | Next: TypeScript development tools |
