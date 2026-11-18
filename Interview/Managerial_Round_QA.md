Prepared for: Software Developer / Backend Engineer interview \| Friday, 20 November 2026

Based on the uploaded Barclays resume and Atlas Trade Contributions documentation. Answers are first-person rehearsal drafts, not scripts that must be memorized.

# How to use this handbook

- Prioritize Sections 1–3 first: they are uniquely tied to your own experience and the manager is likely to probe them.

- Practice answers aloud: 45–90 seconds for behavioral questions, 2–4 minutes for projects, then handle follow-ups.

- For every claim, explain what you personally did versus what the team delivered. Never guess implementation details.

- Use STAR + Learning: Situation, Task, Action, Result, Learning; cite actual measured outcomes where available.

- For technology you have studied or certified in but not deployed professionally, distinguish learning from production experience.

**Evidence tags:** FACT = explicitly supported by your files; PRACTICE = a recommended response or hypothetical method; VERIFY = fill in from your actual recollection before interview.

# Contents

1\. Career narrative, SAP Fioneer and Barclays transition

2\. Five flagship projects with architecture, trade-offs and probing Q&A

3\. Managerial and behavioral questions (STAR)

4\. Technical–managerial scenarios and model answers

5\. Questions to ask the hiring manager

6\. Final rehearsal checklist and evidence / caution notes

# 1. Career narrative, SAP Fioneer and Barclays transition

### Q1. Tell me about yourself.

I’m a backend software developer with more than three years of experience at Barclays, currently working as a Senior Software Developer. I work mainly with Java, Spring Boot, REST APIs, SQL, messaging and distributed systems in financial trade-processing applications. My work has ranged from feature development and production support to performance and reliability improvements. For example, I introduced distributed caching into a validation flow, reducing cached lookups from about 956 milliseconds to 2 milliseconds. I also helped redesign booking-group execution for parallel processing, which improved large-request processing by about 57%, and worked on large-scale trade archival and asynchronous instrument processing. What I enjoy most is understanding a difficult business problem, choosing the right technical design, and taking responsibility for reliable delivery. I’m looking to broaden my engineering exposure while applying the financial-services context I’ve built, which is why this SAP Fioneer opportunity interests me.

### Q2. Why do you want to leave Barclays?

Barclays has given me a very strong foundation in financial systems, production ownership and engineering discipline, and I’m grateful for that. Over time, I’ve become very familiar with the existing platforms, their domain and workflows. Much of my day-to-day work now involves extending and maintaining established, sometimes legacy, systems. That work is valuable, but I feel the incremental learning is increasingly concentrated in business-domain specifics rather than new engineering approaches. I’ve therefore invested in upskilling and certifications outside my day-to-day responsibilities. I’d like my next role to give me more opportunities to use those skills in modern application design, architecture, cloud-native or scalable services, while still contributing my banking experience. My decision is about the next phase of technical growth, not dissatisfaction with Barclays or the team.

**Critical follow-up: “Are you saying you do not learn technically at Barclays?”** Not at all. I have worked on Coherence caching, concurrency, async job processing and Spring Boot migration—those have been substantial technical challenges. My point is that I want that type of engineering challenge to represent a larger share of my day-to-day work and to work in a different product architecture.

**VERIFY before using:** You mentioned multiple certifications and self-upskilling; your supplied resume does not list certificate names. Insert 2–3 real examples, dates and what you built or learned. Do not invent credentials.

### Q3. Why SAP Fioneer?

I’m interested in SAP Fioneer for two reasons: the domain fit and the engineering opportunity. SAP Fioneer builds software specifically for banks, insurers and other financial institutions, so my experience with complex trade workflows, controls, integrations and production reliability should transfer well. At the same time, its focus on evolving financial platforms and cloud-enabled software is aligned with the direction in which I want to grow technically. I also noticed the company emphasizes ownership, learning and partnership with customers. Those values resonate with the way I’ve worked—taking features through validation and rollout, investigating production issues and making practical performance improvements. I would like to contribute from day one with my Java and financial systems experience while learning the team’s product and architecture.

**Research-based note:** Company-specific statements above draw on SAP Fioneer official About/Careers pages. Do not claim that this particular team uses Kubernetes, a specific cloud or a particular architecture unless the interviewer or job description confirms it.

### Q4. Why should we hire you?

I bring a combination of hands-on backend engineering, financial-domain understanding and operational ownership. I’ve not only built APIs and business features; I’ve also improved latency, parallel processing and database performance with measured results, supported releases, and investigated incidents across multiple dependent services. I am comfortable reasoning about correctness, throughput and failure handling, which are critical in financial systems. I know there will be new technologies and domain concepts to learn here, but I have a track record of learning independently and translating that learning into production improvements.

### Q5. Where do you see yourself in three to five years?

I want to become a strong senior engineer or technical lead who can independently design and deliver reliable backend services, guide architectural decisions and mentor others. In the near term, I want to understand the product thoroughly and become a trusted contributor. Over time, I want to own larger components end to end and help the team improve performance, quality and development practices. I’m interested in growing technically first, rather than pursuing a management title simply for the title.

### Q6. What are your strengths and one area for improvement?

My strengths are problem decomposition, ownership and evidence-based debugging. For example, I’m comfortable tracing an incident across logs, SQL and external integrations instead of assuming the first service reporting an error caused it. An area I’m developing is making my technical communication more concise for audiences who don’t need every implementation detail. I’ve been improving that by starting with the business impact, then the decision and trade-off, and going deeper only when needed.

### Q7. Why switch when you were promoted to Senior Software Developer?

I view the promotion as recognition that I’ve delivered and taken ownership at Barclays, not as a reason I must stop exploring. I’m proud of that progress. I’m evaluating a move because the role could expose me to a broader set of technical challenges and financial software products. I’d choose a move only if the responsibilities and team are a strong fit.

### Q8. What if SAP Fioneer also has legacy code?

I expect a mature financial software company to have legacy components and modernization constraints. That does not discourage me. I have experience improving existing systems safely, which is valuable in this domain. I am not seeking an environment with zero legacy code; I’m seeking a role where I can contribute to a good balance of platform evolution, modern engineering practices and product delivery.

# 2. Five flagship projects: detailed interview deep dives

Use the same project story structure each time: business problem → architecture and your actions → design alternatives/trade-offs → measurable results → failure modes → learning.

## Project 1 — TRAC Nexus determination: Oracle Coherence distributed cache

**Business problem \[FACT\]** APAC bookings repeatedly invoked London-hosted TRAC for Nexus determination. A remote lookup took ~956 ms; many requests shared the same Trader BRID + Salesperson BRID inputs.

**Design and implementation \[FACT\]** Built NexusCache with NexusCacheKey = (Trader BRID, Salesperson BRID) and NexusCacheValue = resolved Nexus/TRAC code; used Coherence with a 24-hour expiry. In prepareDataForTracAndCallTrac(), a cache hit bypassed the remote call and populated Nexus on the trade; a miss called TRAC and cached the response. Added hit/miss logs and the nexus.cache.enabled feature toggle. Preserved existing exception/fallback and amendment/variation behavior.

**Outcome \[FACT\]** Cached lookup ~956 ms → ~2 ms (99.8% lower); test run of 20 bookings ~70 sec → ~19 sec (~73% lower). Distinguish cache-hit latency from end-to-end booking latency.

**Trade-offs \[FACT\]** Possible staleness for up to 24 hours; Coherence memory and operating cost; cold misses still incur the remote call. Local JVM cache would not share entries across nodes; per-trade keys had little reuse; database cache would add persistence latency.

### How would you explain it in two minutes?

TRAC is a remote dependency for Nexus determination, and our APAC trade-booking flow had almost a one-second network cost per eligible lookup. We noticed that the response inputs were repeated across many trades. I added a Coherence-backed distributed cache using Trader BRID and Salesperson BRID as the key. A hit returns the stored Nexus code without calling London; a miss uses the existing TRAC path and populates the cache. I kept the surrounding business rules and fallback unchanged and added a feature toggle so operations could disable the optimization without redeploying. Cached lookups went from about 956 milliseconds to 2 milliseconds, while a 20-booking test fell from about 70 to 19 seconds. The main trade-off was possible staleness over the 24-hour TTL, so cache validity and fallback behavior are the key design questions.

### Why Coherence rather than HashMap or local cache?

The application runs across multiple nodes, so a local map would give independent caches, duplicate remote calls and inconsistent warm-up across instances. Coherence makes cache entries accessible across the cluster. A HashMap also needs explicit concurrency, eviction and lifecycle management. The distributed cache does introduce network and operational cost, so it makes sense only because remote TRAC calls are much slower and keys are reused.

### What happens if the value becomes stale?

The implemented trade-off was a 24-hour expiry, so a changed mapping could remain stale until expiry. I would call that out transparently rather than claiming strong consistency. If the business required fresher results, I would discuss event-driven invalidation, shorter TTL, versioned keys or explicit cache clearing, balancing correctness and hit rate. Any change to regulatory-critical validity needs business approval.

### What if cache or TRAC is unavailable?

The documented change preserved existing fallback behavior for TRAC failures. In a design review I would distinguish cache failures from remote failures and ensure we never silently convert an unknown eligibility decision into an approved one. I would log/cache-miss metrics, use bounded timeouts, and follow the approved business policy for the trade workflow. I would check the exact deployed exception-path behavior before claiming specific fail-open or fail-closed semantics.

### How would you prove the improvement is real?

Measure cache hit/miss rate, p50/p95/p99 Nexus retrieval latency, end-to-end booking latency, remote TRAC call volume and error rate under representative inputs. Separate cold-cache and warm-cache tests and compare identical booking mixes across environments. The ~2 ms figure applies to cache hits, not every request.

## Project 2 — Parallel booking-group processing

**Business problem \[FACT\]** AMM/RAD/ALPHA requests with 1,000+ booking groups were processed sequentially, creating high latency and REST timeouts.

**Design and implementation \[FACT\]** Created FlowProductBookingParallelController; deep-cloned request context per group to isolate mutable state; used a runtime-configurable fixed-size pool (property booker.flow.product.booking.concurrent.thread.pool.size), with default 10 and guardrails 1–20. Propagated EpicUserSession and RequestSession into worker threads. Ran group-specific steps in parallel, then required shared sequential steps to execute once after aggregation; added 360-second timeout, named threads, validation, failure handling and shutdown.

**Outcome \[FACT\]** 1,710-group workload: 439.182 sec → 186.697 sec at 15 threads (57.49% faster). Small request: 7.2 sec → 3.388 sec at 8 threads. Optimal thread count was empirical and workload-dependent.

**Trade-offs \[FACT\]** Deep cloning increases heap use; concurrency adds orchestration and debugging complexity. Unlimited threads risk CPU saturation, context switching and dependent-system contention. Fire-and-forget was inappropriate because the upstream client required a synchronous result.

### Explain this project to a manager.

Our booking service was executing independent booking groups one after another, even though the server had capacity for concurrent work. For large requests that led to response delays and occasional timeout risk. I introduced a controller that creates an isolated context for each booking group and submits the work to a bounded, configurable thread pool. I made sure the worker threads carried the necessary user and request sessions and that shared final workflow steps ran only after all group results were collected. We tested different pool sizes rather than just increasing threads indefinitely: for 1,710 groups, the elapsed time dropped from 439 to 187 seconds at 15 threads, a 57.49% improvement. The most important part was preserving the original booking result and security context while improving throughput.

### How did you avoid race conditions?

Each group received a deep-cloned context to avoid concurrent mutation of the same request objects. Steps requiring global coordination were kept sequential after parallel tasks completed. We also had bounded pool sizing, error aggregation and session propagation. I would explain which mutable objects were copied, which were shared and how output consistency was validated rather than simply saying “thread safe.”

### How did you choose 15 threads? Why not 100?

We benchmarked different concurrency levels in representative workloads. The documentation shows that for large requests 15 threads gave the best practical balance; beyond roughly 15–20, contention and coordination overhead reduced benefits. An ideal pool size depends on CPU versus I/O wait, connection limits, memory and downstream service capacity, so I would tune it using actual workload metrics.

### What if one group fails or the whole request times out?

The implementation includes timeout and failure handling plus result aggregation. In an interview I would distinguish the observed implementation from the desired API contract: decide whether partial success is legal, capture failed group IDs, avoid committing a misleading success state, cancel outstanding work where safe and preserve enough information for retry. For financial bookings, retry must account for idempotency and partially committed trades.

### Why not CompletableFuture and return immediately?

The documented requirement was that upstream systems needed the synchronous result for the original booking request. We parallelized \*internally\* while keeping the external contract unchanged. If the contract could change, an asynchronous job API could improve client timeout behavior, but that would require a broader integration design.

## Project 3 — Listed trade archival: 18.5 million records

**Business problem \[FACT\]** Database growth (~2.3M trades monthly; 450GB+ storage), retrieval taking 40–60 seconds, CPU peaking at 85–90%, and backup/recovery window increasing to 12+ hours.

**Design and implementation \[FACT\]** Identified candidates older than 18 months by book/instrument, developed SQL scripts to move active rows to an archive schema, added on-demand unarchival, integrated repeatable cleanup into production release cycles and used before/after validation checks.

**Outcome \[FACT\]** 18.5M trades archived over six releases, freeing ~340 GB; average query time 45s → 8s (~82% reduction); reported peak CPU improved to 55–65%; backup/recovery window ~12h → ~5h. No reported post-archival production integrity incidents.

### Walk me through your trade archival solution.

Our active trade database had become expensive to query and maintain as historical listed trades accumulated. We first defined eligibility based on the business retention requirement, including the 18-month age threshold and book/instrument filters. I developed SQL scripts to transfer eligible rows into an archive schema, validated counts and data integrity before and after, and ensured that specific trades could be restored for audit or compliance needs. We executed the changes incrementally over six production releases rather than attempting one large high-risk movement. In total 18.5 million trades were archived, freeing about 340 GB and reducing average query retrieval from 45 to 8 seconds. For me, the core engineering challenge was correctness and recoverability, not simply making the SQL faster.

### How do you prevent data loss when moving millions of rows?

Use explicit archival criteria, a stable primary-key inventory, controlled batches, reconciliation of source/destination counts and key fields, and an auditable rollback/recovery plan. For each batch, the safe sequence and transaction scope must be designed carefully. In production I would verify foreign-key dependencies, downstream consumers and restore procedures. Our documented implementation included before/after validations and an unarchival capability; exact batching and transaction-isolation choices should be described only if I can confirm them.

### What if the archival fails after 10 million records?

Never assume the whole move is atomic. First identify committed batches and reconcile those records; then resume using a deterministic idempotent key and state/manifest, rather than blindly rerunning inserts. Avoid deleting active records until destination integrity is verified. If the failure compromises reconciliation, stop and use the approved rollback/unarchival procedure. This is a recommended design response; the uploaded notes do not specify our exact resume algorithm.

### Why did query latency improve?

Reducing active data can decrease scanned rows, index size, memory pressure and I/O, but the actual explanation requires execution plans before and after. I would not claim that data volume alone explains every query’s gain. I would compare index usage, table statistics, I/O and query plans to substantiate the improvement.

## Project 4 — Bulk instrument expiry: async tracked processing

**Business problem \[FACT\]** 50,000+ instruments required an expiry update; the earlier JMX process allowed at most 200 IDs per invocation and took ~1 hour for 1,000 instruments, with limited progress visibility.

**Design and implementation \[FACT\]** Added file-based CSV/TXT input through updateExpiryDateFromFile(); processed in batches of 10 using CompletableFuture and an ExecutorService of 20 threads. Generated UUID job IDs, kept JobProgress in ConcurrentHashMap, counted total/completed/failed with AtomicInteger, maintained per-instrument error details, applied business-date calendar conventions and persisted instruments individually to avoid shadow-booking conflicts. Exposed getJobProgress(), listActiveJobs(), clearJob(), plus filename sanitization.

**Outcome \[FACT\]** 1,000 instruments: ~60 min → ~2 min (~30x faster); supported 50,000+ instruments; tracked progress and isolated failures, with three months of monitoring reporting zero data corruption incidents.

### Explain the design end to end.

The original manual JMX operation was bounded at 200 instruments and blocked the operator, making a 50,000-plus instrument update both slow and hard to monitor. I changed it to accept a server-side file of IDs, create a unique job ID and execute the work asynchronously in small batches using a bounded executor. The service uses concurrent maps and atomic counters to expose progress and per-instrument failures while applying business-calendar rules to expiry dates. Importantly, it persists each instrument separately to avoid conflicts, so one failure does not stop the entire job. For 1,000 instruments we reduced processing time from around one hour to two minutes while adding operational visibility.

### Why ConcurrentHashMap and AtomicInteger?

Many worker threads update job state concurrently. ConcurrentHashMap supports thread-safe access to shared job and error maps; AtomicInteger prevents lost updates to counters without taking a global lock. These choices do not make a multi-field status update automatically atomic, so a coherent job snapshot still requires careful state design.

### What is the biggest limitation you would improve?

The job-state map is in JVM memory, so a process restart can lose status information. In a more durable enterprise design I would persist job metadata and per-item checkpoints in a database or queue, reconcile pending work after restart, support idempotent retries, and define retention/cleanup. That is a proposed improvement, not a claim about the implemented system.

### What about “unlimited” input size?

The previous per-call 200-ID cap was removed. That does not mean infinite memory or throughput. In a more general design I would stream large input files and enforce operational quotas, capacity limits and backpressure.

## Project 5 — CCLT/TRAC eligibility and booking controls

**Business problem \[FACT\]** Atlas workflows did not uniformly enforce counterparty eligibility and justification requirements, creating compliance and audit gaps across products and booking channels; T0 amendments were an identified remaining gap.

**Your documented work \[FACT\]** Performed end-to-end validation across Loan & Deposit quick/bulk, FX Forward bulk and Equity Swap bulk flows, covered positive/negative trader and TCU journeys, fixed missing client-name population in TRAC headers, investigated QA defects and analyzed the amendment flows requiring expanded protection. Do not describe T0 amendment as fully deployed if it is still being developed.

**Business decisions \[FACT\]** Soft blocks with an override justification were preferred to unconditional hard blocks for valid exceptions. Feature flag controlled rollout. Existing behavior continued during TRAC integration failures to avoid a trade-booking outage. Those decisions have compliance risk and must be understood as approved business policies, not generic best practices.

### Explain why this project mattered.

In a financial booking system, allowing a trade without the proper counterparty eligibility control can create audit and regulatory risk. We needed TRAC checks to behave consistently across several products and booking journeys. My role included end-to-end validation, investigating defects, fixing incomplete request data and taking ownership of identifying where T0 amendment flows needed the same protection as new bookings. I tested both permitted and non-permitted scenarios so the user received the correct warning or exception workflow. The design allowed justified exceptions rather than indiscriminately blocking genuine trades, and feature toggles supported controlled rollout. This work emphasized that correctness means the business workflow and audit trail are both accurate.

### Why soft-block rather than hard-block?

The documented business decision was to allow legitimate business exceptions with an explicit justification, rather than stopping every trade that produced a warning. That preserves necessary operations and traceability but also introduces risk. In the interview I would say the final policy must be defined by compliance and product owners, with authorization, auditing, monitoring and controls around overrides.

### Why continue on a TRAC outage? Is that safe?

That was the documented availability trade-off, but it is not a universal safe choice. I would explain the approved failure policy, how outage-driven bookings are logged and monitored, and what retrospective reconciliation or review applies. If the control is legally mandatory before booking, fail-open would be inappropriate. My engineering responsibility is to implement the agreed control and make the operational risk explicit.

### How do you ensure different booking workflows behave consistently?

Create a product-by-workflow test matrix covering quick entry, bulk, structured, variation and amendment where applicable; test valid/invalid counterparties, overrides, missing client data, downstream errors and user roles. Validate both API data and UI messages, and automate stable regression paths.

## Bonus project for follow-up — Delta One equity-swap STP

If asked for another end-to-end integration, explain GCAM → Delta One → Atlas, 40+ field mappings, proto schema changes, mandatory L1 acknowledgement validation, conversions, duplicate detection using swapId + lotId and correlation IDs. The documented business result was zero-touch trade capture, replacing manual Atlas booking. Explain concrete mapping and validation decisions, not a vague “I integrated systems.”

# 3. Managerial / behavioral questions — STAR + Learning

Responses below are rehearsal examples grounded in your contribution record when possible. Where the record does not include a personal conflict, deadline failure or career detail, the answer is a safe structure or explicitly marked VERIFY. Do not invent a specific incident.

### B1. Tell me about a challenging project you owned. \[STAR: parallel booking\]

S: Large booking requests were sequential and at risk of timing out. T: Improve performance without changing booking outcomes. A: I introduced a bounded, configurable parallel execution controller, isolated per-group contexts, propagated request sessions and coordinated final sequential steps. I benchmarked pool sizes and validated results. R: A 1,710-group workload dropped from 439 to 187 seconds, a 57.49% improvement. L: Parallelism succeeds when data isolation, downstream capacity and correctness are designed before simply adding threads.

### B2. Tell me about a time you demonstrated ownership. \[STAR: instrument expiry\]

S: A time-sensitive mass expiry update required processing more than 50,000 instruments. T: Replace manual, slow and opaque JMX operations with something safe and trackable. A: I implemented file-driven asynchronous execution, bounded workers, job IDs, progress tracking, atomic counters, per-instrument failures and business-date logic. R: A 1,000-ID run dropped from an hour to two minutes, and monitoring found no data-corruption issues. L: Ownership includes operational visibility and failure recovery, not just code delivery.

### B3. Tell me about an incident you investigated. \[STAR: publishing delay\]

S: A delayed trade publication was affecting downstream NYK batch processing. T: Determine whether Atlas was the source and help the involved teams restore confidence. A: I traced Atlas, Sophis and POG timestamps, confirmed the booking and publication stages were timely, provided SQL/log evidence to the downstream team, and identified limitations caused by short log retention. R: The investigation narrowed the issue beyond Atlas, supported coordinated follow-up and no recurrence was observed afterward. L: In distributed systems, cross-service timestamp evidence is more reliable than assumptions based on where a problem was first reported.

### B4. Tell me about a mistake or failure.

I try to discuss mistakes without hiding them, but I would not manufacture a story. A suitable real example should be a test case I initially missed, an estimate I underestimated or a design assumption that changed. I would explain: what I assumed; how I discovered the error; the immediate containment; how I communicated it; the correction and regression test; and what practice changed afterward. VERIFY: choose one real event you can defend under follow-up.

### B5. Have you ever missed a deadline?

I focus on surfacing delivery risk early rather than giving a false commitment. If a dependency or late defect threatens a timeline, I clarify the must-have scope, estimate remaining work, propose a safe phased delivery, and communicate the impact and options to the lead and business. If I had personally missed a deadline, I would own that fact and explain what I changed. VERIFY: insert a specific genuine example if one exists.

### B6. Describe a disagreement with a senior engineer or tech lead.

First I want to understand the constraints behind the proposed approach. I explain my concern using evidence—performance, maintenance cost, security or correctness—and suggest a small proof of concept where reasonable. I am comfortable following a decision after a fair discussion, except where it creates an unresolved material security, compliance or data-integrity risk, which I would escalate respectfully. VERIFY: add a real disagreement if asked for “tell me about a time.”

### B7. How do you handle competing urgent tasks?

I rank work by customer impact, regulatory or data-integrity risk, production severity and dependencies. I clarify deadlines and communicate what cannot be delivered simultaneously. I time-box investigations, ask for help when parallel streams make sense and keep stakeholders updated. For example, an active production approval block usually deserves faster attention than a nonurgent refactoring task.

### B8. How do you manage production pressure?

I separate containment from diagnosis. I establish impact and timeline, collect logs and metrics, check recent changes and downstream dependencies, and use a rollback or safe mitigation where appropriate. I communicate verified findings at regular intervals and document the root cause and prevention afterward. My cross-system trade-publishing and Reuters-fixing investigations helped me learn to avoid premature root-cause claims.

### B9. Tell me about a time you worked with another team. \[STAR: QA and controls\]

S: Counterparty eligibility needed consistent behavior across products and multiple booking workflows. T: Support reliable rollout and resolve defects. A: I coordinated validation with QA, analyzed reported failures, fixed missing TRAC client data and verified both positive and negative paths against business requirements. R: The integration became more consistent and auditable across supported flows. L: Integration work succeeds when developers, QA and business owners agree on the observable behavior, not just the code implementation.

### B10. How did you mentor a junior engineer?

I have helped new engineers understand system architecture through knowledge-transfer sessions. My approach is to explain the end-to-end flow, show where logs and data can be traced, provide a manageable initial task, review their reasoning rather than just the result and encourage them to document what they learned. VERIFY: add one actual mentee/task outcome if the interviewer asks for a concrete STAR example.

### B11. How do you handle unclear requirements?

I begin with the business intent and the failure cases. I identify users, data contracts, controls, exceptions and what “done” means, then convert that into concrete examples and acceptance tests. The CCLT work is a good example: different booking types and T0 amendments needed explicit decisions about warnings, override justification and integration failures before development could be considered complete.

### B12. How do you balance speed and code quality?

I protect correctness, security and financial controls first. For time-sensitive delivery, I isolate minimum scope, reuse existing patterns, automate high-risk tests and plan follow-up refactoring explicitly instead of silently accumulating debt. A feature toggle or phased rollout can reduce blast radius, but it is not a substitute for testing.

### B13. What if a teammate is underperforming?

I would first understand whether the issue is unclear expectations, workload, missing context or skill gaps. I would offer focused help, make deliverables concrete and follow up on progress. If customer impact continues, I would raise the concern factually with the lead rather than blaming the person. I would not imply that I have formal people-management experience when I do not.

### B14. How do you accept critical feedback?

I try to separate feedback about a solution from feedback about me. I ask for examples, understand the impact, agree on a specific improvement and check that the change is working. For a code review, I would discuss design concerns constructively and update code or document the agreed trade-off rather than turning the conversation into a personal debate.

### B15. Tell me about improving a process. \[STAR: folder automation\]

S: Operations colleagues had to execute multiple JMX actions and follow-on reports manually for folder moves. T: Reduce repetitive effort and execution mistakes. A: I consolidated the calls in a shell workflow, used token-based authentication, added operator confirmation and moved reporting to scheduled Autosys jobs. R: The workflow reduced manual touchpoints and dependency on operators for every step. L: Small workflow automation can deliver meaningful reliability without a platform redesign.

### B16. What motivates you?

I enjoy problems where technical decisions produce measurable improvements—faster booking, safer processing, better data quality or fewer operational incidents. I also enjoy learning a system deeply enough to simplify it for others. I want a role where I can keep building that expertise while expanding the technologies and architecture patterns I work with.

### B17. What would you do in your first 90 days?

In the first month I would understand the product architecture, development practices, release process, business-critical flows and team expectations. Next I would own small but meaningful changes, get comfortable with testing and production observability, and identify recurring friction points. By 90 days I would aim to deliver a feature independently, understand failure paths and propose one evidence-based improvement without forcing architectural changes too early.

### B18. Why should we trust you with a business-critical change?

I do not treat success as “the code compiled.” In financial workflows I check valid and invalid states, downstream dependencies, rollback and audit implications. In my performance work I kept booking behavior stable while changing execution; in archival I added validation and restore capability; in controls I tested authorization and exception paths. I communicate assumptions and residual risk early.

# 4. Technical–managerial questions: high-probability scenarios

For each scenario: clarify constraints → diagnose/design → trade-offs → failure handling → communication and verification. These are recommended approaches, not descriptions of how your company necessarily implemented the solution.

### T1. An API went from 200 ms to 5 s. How do you troubleshoot?

First define whether the problem affects all traffic or a segment and whether it coincides with a deployment or traffic spike. Check p50/p95/p99, errors, CPU, GC, thread and DB connection pools; trace the critical path and compare slow SQL, remote dependencies and cache hit rates. Mitigate safely via rollback, capacity or load shedding if necessary. Validate recovery, explain customer impact and create a root-cause action with monitoring. I would not increase timeouts before identifying the bottleneck.

### T2. Parallel processing caused duplicate bookings. What next?

Contain the issue by pausing/limiting affected execution; establish whether duplication comes from concurrent submission, retries or broker redelivery. Correlate request/job IDs, trace transactional boundaries and check uniqueness constraints. Add or enforce a business idempotency key and atomic state transition, then safely reconcile duplicates under operations controls. Do not trade off financial correctness for throughput.

### T3. Critical defect discovered hours before release. What do you do?

Assess severity, affected financial flows, likelihood and available mitigation; reproduce and show evidence. Recommend delaying the risky component, disabling it with a tested feature flag or releasing only unaffected scope. Inform the product owner, release owner and appropriate risk approvers. Document the decision and rollback plan. A manager values reliable judgment more than a rushed release.

### T4. Design processing for 50,000 instrument updates.

Define deadlines, per-item consistency, retries and audit needs. Submit a file/job request and return HTTP 202 with job ID; validate input, persist job metadata, queue bounded batches, perform idempotent updates with business-date rules, record item-level status, and expose job progress and failure reports. Apply backpressure, rate limits, authentication and monitoring. Unlike the documented in-memory JMX job, a reusable enterprise service should persist state across restarts.

### T5. Cache has 99% hit rate, but users report wrong results.

A high hit rate proves reuse, not correctness. Check whether the cache key includes every input that determines the answer, including tenant, regulatory context, version or effective date. Inspect TTL and invalidation, compare sampled cache values against source-of-truth calls, disable the feature if necessary and correct affected data under an approved remediation process. In regulatory decisions, correctness trumps latency.

### T6. Your cache cluster is down. How do you handle it?

Use a defined timeout and failure policy, avoid a cache outage cascading into thread starvation, and consider bounded fallback to the source service with circuit breaking. Monitor miss storm and dependency capacity. For mandatory validations, never claim that failing open is universally safe; follow the approved compliance policy and escalate where that policy is unclear.

### T7. How would you size a thread pool?

Classify tasks as CPU-bound, blocking I/O or mixed. Start with available cores and observed waiting-to-compute ratio, then benchmark throughput, p95 latency, heap, context switching, connection-pool pressure and dependency saturation. Bound the queue and rejection behavior. Our booking workload improved most at 15 threads in the tested environment, but that number should not be blindly generalized.

### T8. How would you prevent deadlocks?

Use consistent lock ordering, minimize lock scope, avoid holding locks across network calls, prefer immutable state and concurrency utilities, and use timed lock acquisition where appropriate. Diagnose using thread dumps and blocked-thread metrics. In databases, use short transactions, deterministic update ordering and appropriate retry strategies for deadlock victims.

### T9. Explain @Transactional and common pitfalls.

Spring transaction advice normally applies through a proxy. A call from one method to another within the same object can bypass that proxy, so a self-invoked annotated method may not create the intended transaction. By default runtime exceptions and errors trigger rollback; checked exceptions generally need explicit rollbackFor. Keep transactions appropriately short and know how propagation and isolation influence behavior. In a booking service, external calls plus long DB transactions require special care.

### T10. What happens if Kafka delivers the same message twice?

At-least-once delivery allows duplicates. Use a stable event/business ID, atomic deduplication with a unique constraint or state table, and an idempotent handler. Commit offsets only after the processing contract is satisfied; retries and poison-message handling need bounded policies. “Exactly once” marketing does not automatically make external databases or downstream side effects exactly once.

### T11. A database query is slow. What is your approach?

Capture the actual query and execution plan; examine scan versus seek, index selectivity, predicate sargability, joins, row estimates, statistics, blocking and I/O. Check if the issue is data skew or parameter sniffing. Add/rework indexes or queries based on measured plans, and consider partitioning or archival when lifecycle data is the real issue. Benchmark write overhead and rollback before applying an index to production.

### T12. How do you migrate 18.5M rows without disruption?

Define eligibility and retention approvals, dependency mapping, archiving batches, throttling, transaction boundaries, reconciliation and restore paths. Monitor lock duration, log growth, replication lag, CPU and query latency. Use progressive releases and pause/rollback criteria. Publish clear operational checkpoints so the process is both safe and auditable.

### T13. A bulk job fails after a JVM restart.

If status lived only in ConcurrentHashMap, job metadata is lost. A robust redesign persists job and per-item checkpoint state in a database or durable queue, uses leases/heartbeats, reclaims unfinished work and makes operations idempotent. It should clearly distinguish completed items, retryable failures and permanent failures.

### T14. How do you ensure microservices remain consistent?

First clarify whether immediate consistency is truly required. Use local transactions where possible, durable events/outbox for cross-service notifications, idempotent consumers and compensating actions for multi-step business processes. Track correlation IDs and reconcile failed transitions. Avoid assuming a distributed database transaction is always the right answer.

### T15. Two services disagree on a trade status. How do you investigate?

Identify the canonical source of truth and expected state machine. Correlate trade ID, message ID and timestamps across services; inspect queue lag, retry/DLQ state and out-of-order events. Determine whether the difference is temporary eventual consistency or a missed transition. Correct the state through a controlled replay/reconciliation path and prevent recurrence through monitoring or stronger contracts.

### T16. Production CPU is 95% and requests are timing out.

Establish thread-pool and traffic changes, GC behavior, hot threads and expensive SQL/external retry loops. Capture thread dumps and metrics before restarting unless user impact requires immediate mitigation. Reduce traffic or roll back a suspected regression, then profile the hot path. Communicate scope, containment and the evidence supporting root cause.

### T17. A Java service has an OutOfMemoryError.

Check heap usage over time, GC logs, heap dumps, container memory and recent feature changes. Distinguish a true retention leak from excessive allocation or undersized heap. Candidate causes include unbounded caches, in-memory job maps, queues, large cloned request objects and thread-local retention. Mitigate load/restart if necessary, then fix lifetime, bounds and monitoring rather than only increasing heap.

### T18. How would you implement rate limiting for a critical API?

Choose the protected dimension (client, account, endpoint) and business SLA. Use token bucket or leaky bucket with centralized coordination if multi-node, return meaningful 429 responses and retry hints, and distinguish abuse protection from backend concurrency caps. Monitor rejection rates and ensure privileged internal workflows have an explicitly approved policy.

### T19. How do you design retries for a financial transaction?

Retry only known transient errors with bounded exponential backoff and jitter. Ensure the operation is idempotent before retrying; ambiguous timeouts may mean the first operation succeeded. Use idempotency keys and a reconciliation/query path before reissuing financial side effects. Dead-letter unrecoverable failures with adequate audit context.

### T20. What is your deployment and rollback plan?

Use CI quality gates, automated unit/integration/regression tests, configuration review, smoke tests, observability and a backward-compatible DB migration strategy. Prefer canary/phased rollout or tested feature toggles for risky changes. Define who can roll back, which conditions trigger it and how schema or message-contract changes behave during mixed versions.

### T21. How do you handle sensitive logs in financial systems?

Log correlation IDs, transaction identifiers in an approved format, event transitions and error categories, while masking credentials, personal information and client-sensitive payloads. Enforce role-based access, retention policy and auditability. For debugging, collect the minimum authorized data needed; avoid copying production customer data into unrestricted systems or external tools.

### T22. How would you review a junior developer’s code?

Start with correctness and business requirements, then consider error cases, concurrency, security, testability and maintainability. Ask clarifying questions and explain trade-offs, not only style preferences. Suggest a narrow improvement path, verify tests and encourage ownership of the fix. Escalate release-blocking issues clearly but respectfully.

### T23. Spring Boot 2.6 to 2.7 migration: what actually changed?

We upgraded centralized dependencies, changed H2 1.4 to 2.x (including identity syntax and case/keyword compatibility), updated MSSQL drivers with connection encryption configuration, adapted schema and tests, and coordinated the change across six-plus microservices. I would explain the risk of transitive dependencies and backward compatibility. Important nuance: trustServerCertificate=true encrypts transport but bypasses server certificate validation; it should not be presented as the ideal final security posture. Production should prefer properly trusted certificates.

### T24. How do you integrate SonarQube with CI/CD?

Build and test the Java application in a merge-request pipeline, generate coverage and static-analysis artifacts, authenticate the scanner using a managed secret, upload results and apply a quality gate. Review new-code issues and exceptions with traceability. The contribution notes contain setup guidance but do not prove that I personally owned this whole integration. I would only say “I implemented it” if I can substantiate that.

### T25. An integration has partial success: some downstream operations passed, others failed.

Use an explicit business state machine and distinguish reversible from irreversible steps. Record durable progress, retry idempotent operations and compensate where approved. Expose partial failure rather than a misleading green status, provide reconciliation tools and notify stakeholders of impacted transactions. The right mechanism depends on each product’s settlement and control requirements.

# 5. Questions to ask the hiring manager

Choose three or four based on what has already been discussed. Ask genuine questions and listen to the answers; do not use this as a scripted speech.

- What are the most important engineering challenges this team is addressing in the next six to twelve months?

- What proportion of the role involves developing new capabilities versus maintaining or modernizing existing platforms?

- What backend stack, deployment architecture and cloud infrastructure does this specific team currently use?

- How much ownership does an engineer at my experience level have over design, testing, production monitoring and releases?

- What does strong performance look like in the first 90 days and first six months?

- How are architectural trade-offs and code quality decisions made within the team?

- How do engineering teams work with banking-domain specialists, product owners and customer stakeholders?

- Are there opportunities to contribute to scalability, reliability or modernization work?

- What mentoring, code-review and learning practices help engineers grow technically here?

- What would be the next steps and decision timeline after this final round?

**Recommended closing:** Thank you for the conversation. The combination of financial-domain challenges and engineering ownership sounds aligned with what I want to do next. Is there any part of my experience you would like me to clarify before we finish?

# 6. Final rehearsal checklist and integrity notes

- Memorize a 60–90 second introduction, not an entire paragraph word for word.

- Prepare exact explanations for five projects: diagram in your head, key classes/components, trade-offs, measured impact, tests, and failure behavior.

- For each behavioral topic, identify a true event with your personal actions and a learning. Use placeholders rather than fictional conflicts, missed deadlines or mistakes.

- Be fluent with Java concurrency, Spring transactions, SQL execution plans, idempotency, caching and production troubleshooting.

- Prepare two examples of cross-team collaboration and one example of receiving feedback.

- Know your actual certifications: names, dates and one relevant skill gained per certification.

- Do not reveal confidential production trade identifiers, internal hostnames, URLs, secrets, proprietary schemas or unauthorized customer data. Your interview explanations should stay at a sensible architectural level.

- Use neutral language about Barclays. “My current responsibilities are increasingly focused on established platforms” is stronger than “the company only has old technology.”

- Ask whether the SAP Fioneer role actually offers the modern engineering responsibilities you seek. Do not assume that all projects are greenfield.

- Confirm video, microphone, laptop, meeting invitation and a quiet space before the interview.

## About the information in this document

Your supporting documents contain verified project descriptions and measurable outcomes, but they are personal notes, not independent production audits. Several sample answers extrapolate sound engineering practices for interview scenarios. Such extrapolations are recommendations, not claims about implementation. In particular, do not present the hypothetical durable job store, idempotent restart logic, specific archival batching strategy, conflict stories or unlisted certificates as already completed work.

## Company references (official public pages)

SAP Fioneer — About us: https://www.sapfioneer.com/about-us/

SAP Fioneer — Careers: https://www.sapfioneer.com/careers/

SAP Fioneer — Careers article: https://www.sapfioneer.com/careers/blog/why-sap-fioneer-is-not-a-typical-software-company-and-why-that-matters-for-your-career/

Personal reference files: “Sagar_Mali_Resume_3YOE.pdf” and “Pasted text.txt” (Atlas Trade Contributions, Technical Changes and Business Impact).
