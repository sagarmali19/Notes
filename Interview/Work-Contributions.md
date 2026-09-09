# Atlas Trade Contributions, Technical Changes and Business Impact

*Consolidated technical documentation*

---

## 1. TRAC Nexus Determination Performance Optimisation

### Problem Resolved
- TRAC Nexus determination required a remote API call to London for APAC bookings, introducing ~956 ms latency per eligible trade.
- High-volume traders booking hundreds/thousands of trades experienced significant booking delays because every eligible trade triggered a TRAC call.
- Many trades share the same Nexus determination inputs (Trader BRID + Sales BRID), resulting in repeated identical TRAC requests.
- Network latency and remote service dependency became a major contributor to booking response times.
- Performance degradation scaled linearly with trading volume, impacting user experience during heavy booking periods.

### Technical Changes Implemented
- Introduced a distributed Coherence cache (NexusCache) with a 24-hour expiry.
- Added custom cache key (NexusCacheKey) using:
    - Trader BRID
    - Sales Person BRID
- Added custom cache value (NexusCacheValue) to store resolved Nexus/TRAC codes.
- Implemented cache lookup before invoking TRAC in `prepareDataForTracAndCallTrac()`.
- On cache hit:
    - Skip remote TRAC invocation.
    - Retrieve Nexus codes directly from Coherence.
    - Persist Nexus on the trade when required.
- On cache miss:
    - Invoke TRAC normally.
    - Store returned Nexus codes in cache for future reuse.
- Added logging for cache key creation, cache hit/miss tracking, and cache population.
- Integrated cache with existing Nexus processing flow without altering Nexus assessment business logic.
- Preserved existing fallback behaviour for TRAC failures, amendments, variations, and Nexus copy scenarios.
- Added feature toggle (`nexus.cache.enabled`) allowing cache enablement/disablement without deployment changes.

### Real-Time Impact
- Average Nexus lookup reduced from ~956 ms to ~2 ms on cache hits.
- Approximately 99.8% reduction in Nexus retrieval time for cached requests.
- Test execution for 20 bookings reduced from ~70 seconds to ~19 seconds.
- Approximately 73% reduction in end-to-end processing time during testing.
- Significant reduction in calls to the London-hosted TRAC service.
- Improved trader booking experience, particularly for bulk/high-frequency booking activity.
- Reduced dependency on network latency and cross-region communication.
- Improved system scalability as booking volumes increase.

### Trade-offs Made
- Nexus results may remain stale for up to 24 hours if underlying regulatory mappings change during the cache lifetime.
- Additional memory consumption introduced in Coherence cluster for cached Nexus entries.
- First request for a Trader/Salesperson combination still incurs full TRAC latency (cache warm-up cost).
- Cache effectiveness depends on repeated reuse of the same Trader/Salesperson combinations.
- Distributed cache introduces operational considerations such as cache monitoring, sizing, and expiry management.

### Alternative Approaches Considered and Why Rejected
- **Cache by Trade ID** — Low reuse potential because most trades are unique; would deliver far fewer cache hits.
- **Cache by Counterparty + Legal Entity + Trader + Salesperson** — More accurate representation of TRAC inputs, but significantly lower hit ratio due to highly granular keys and reduced business value compared to current approach.
- **Local JVM Cache** — Faster lookup than distributed cache, but ineffective in multi-node deployments; cache entries unavailable across application servers.
- **Increase APAC-to-London TRAC capacity** — Does not eliminate network round-trip latency; higher infrastructure and operational cost.
- **Persistent Database Cache** — Additional database load, slower than in-memory Coherence access, and more operational complexity than distributed caching.

---

## 2. CCLT/TRAC Eligibility Validation Across Booking Workflows

### Problem Being Resolved
- Atlas allowed booking trades with external counterparties without a consistent, automated CCLT/TRAC eligibility check across all booking workflows.
- Traders could book trades with ineligible counterparties without being forced to provide a business justification.
- Different products and booking channels (Quick Entry, Bulk, Structured, Variation) lacked consistent validation behaviour.
- No standard mechanism existed to surface TRAC warnings back to users and capture override reasons for audit and control purposes.
- T0 amendment flows currently do not enforce the same CCLT controls as new bookings, creating a control gap.

### Technical Changes Delivered
- Performed end-to-end validation testing of the new TRAC validation workflow across multiple Atlas booking flows.
- Validated both positive and negative counterparty eligibility scenarios for Trader and TCU user journeys.
- Tested Loan & Deposit Quick Entry workflow integration with TRAC validation.
- Tested Loan & Deposit Bulk Booking workflow integration with TRAC validation.
- Tested FX Forward Bulk Booking workflow integration with TRAC validation.
- Tested Equity Swap Bulk Booking workflow integration with TRAC validation.
- Identified and fixed missing client name population in TRAC request headers, improving request completeness and auditability.
- Investigated and validated defects discovered by QA, including Full Term Event related issues.
- Verified that TRAC warnings, eligibility responses, and workflow behaviour matched business requirements across supported products.
- Took ownership of the T0 Amendment CCLT enhancement by analysing impacted products, identifying affected workflows, and defining required development changes.
- Closes functional gaps identified during UAT before production rollout.

### Real-Time Business Impact
- Prevents non-compliant trades from being booked without appropriate justification.
- Improves regulatory and control compliance by enforcing TRAC-based eligibility checks.
- Provides traders with immediate visibility of CCLT warnings before trade completion.
- Creates an auditable record of override decisions and booking control reasons.
- Reduces operational risk by ensuring consistent eligibility validation across booking channels.
- Improves data quality by ensuring mandatory client information is sent to TRAC.
- Increases confidence that newly onboarded products behave consistently with enterprise control standards.

### Trade-offs & Design Decisions
- **Soft-block approach instead of hard-blocking trades** — Chosen because genuine business exceptions must still be executable; hard-blocking was rejected as it could prevent legitimate revenue-generating trades.
- **Feature toggle for TRAC validation** — Allows controlled rollout and emergency disablement; always-on enforcement was rejected due to operational risk during early adoption.
- **Continue workflow on integration failures** — Prevents trade booking outages when TRAC is unavailable; strict dependency on TRAC was rejected because it would create a single point of failure.
- **Use PTAX derived from existing Atlas product mappings** — Faster implementation with minimal platform dependency; waiting for a fully centralised PTAX resolution service was rejected due to delivery timelines and broader platform dependencies.
- **Prioritise UAT coverage across high-volume products first** — Maximises risk reduction for production rollout; attempting to onboard every product simultaneously was rejected because it would significantly increase delivery and testing effort.

### T0 Amendment Ownership
- Identified that amendment workflows currently do not receive equivalent CCLT protection as new bookings.
- Analysed impacted products and booking journeys requiring enhancement.
- Mapped amendment workflows where TRAC validation must be introduced.
- Defined required design and implementation changes for amendment processing.
- Driving development and validation activities to ensure consistent control enforcement between new bookings and T0 amendments.

---

## 3. Multi-threaded Parallel Booking-Group Processing

### Problem Being Resolved
- AMM/RAD/ALPHA booking requests containing large numbers of booking groups (1000+) were processed sequentially, causing very high response times and REST request timeouts.
- The existing flow under `flowProductBooking` could not efficiently utilise available CPU resources because each booking group was processed one after another.
- Large trade booking requests created a bottleneck for upstream systems, impacting throughput and user experience.
- Increased request volume directly increased booking latency due to the linear execution model.
- The solution needed to improve performance without changing booking behaviour, trade creation logic, or workflow outcomes.

### Technical Changes Implemented
- Introduced a dedicated `FlowProductBookingParallelController` to process booking groups concurrently.
- Split a single request into independent booking-group-level contexts using deep cloning, enabling safe parallel execution.
- Added a configurable fixed-size thread pool controlled through runtime property `booker.flow.product.booking.concurrent.thread.pool.size`.
- Implemented dynamic thread-pool resizing at runtime without application restart.
- Separated workflow execution into:
    - Parallel steps executed concurrently per booking group.
    - Sequential steps executed once after all parallel processing completes.
- Implemented session propagation (EpicUserSession, RequestSession) into worker threads to preserve existing security and request context behaviour.
- Added timeout protection (360 seconds) to prevent indefinitely running booking requests.
- Added validation, failure handling, result aggregation, logging, and graceful executor shutdown.
- Introduced thread naming (`concurrent-flow-*`) for easier production troubleshooting and thread-dump analysis.
- Established guardrails through minimum, maximum, and default pool sizes (1–20, default 10).

### Real Business / Production Impact
- Small requests improved from 7.2 sec to 3.388 sec with 8 threads (52.94% improvement).
- Large 1,710-group request improved from 439.182 sec to 186.697 sec with 15 threads (57.49% improvement).
- Alternate node testing confirmed scaling benefits, with 33.72% improvement at 15 threads.
- Stress testing demonstrated consistent gains ranging from 34% to 51% for most request sizes.
- OTC workflows showed improvements up to 77.77%, with production-like requests improving by 28%.
- ICE batch testing improved from 156 sec to 42 sec (73% improvement).
- Reduced likelihood of upstream REST request timeouts for large booking requests.
- Increased booking throughput without requiring infrastructure expansion or functional workflow changes.

### Trade-offs and Design Decisions
- Accepted: Higher memory consumption due to creation of per-booking-group cloned contexts.
- Accepted: Additional thread-management overhead in exchange for significantly lower execution time.
- Accepted: Slight complexity increase in workflow orchestration to achieve scalability.
- Rejected unlimited thread creation: Could cause CPU saturation, excessive context switching, JVM instability, and reduced performance.
- Rejected very high pool sizes (>15–20): Testing showed thread contention and coordination overhead outweighed parallelism benefits.
- Rejected processing entire request in one thread: Root cause of timeout and poor scalability.
- Rejected asynchronous fire-and-forget processing: Upstream systems require synchronous booking completion and response.
- Rejected changing booking logic itself: Solution focused on execution efficiency while maintaining identical business behaviour and booking outcomes.

### Key Conclusion
- Parallelising booking-group processing transformed the booking flow from a sequential bottleneck into a scalable execution model.
- Performance testing across multiple environments consistently showed that 15 threads provides the optimal balance between concurrency, resource utilisation, and execution overhead for large booking requests, delivering up to 57.49% faster processing while preserving existing booking behaviour.

---

## 4. Delta One STP - for Equity Swaps

### Problem Resolved
- Fragmented STP gap: Atlas leg required manual/semi-manual trade booking while Delta One was automated, causing control gaps and operational risk.
- System synchronization: Ensured GCAM → Delta One → Atlas flow maintains consistency between client-centric booking (D1) and trade booking (Atlas).
- Data loss risk: Missing or incorrect field mapping between upstream systems could result in incomplete trade records or booking failures.
- Operational overhead: Manual intervention slowed down equity swap trade lifecycle for index underlyings.

### Technical Changes Implemented
- Field mapping ownership: Analyzed upstream Delta1 request schema and mapped 40+ fields (mandatory + non-mandatory) to Atlas Trade booking request schema.
- Proto schema evolution: Contributed proto changes to `FlowProductBookingRequest` to accommodate new D1 STP requirements for equity swap creation.
- Validation logic: Implemented L1 ACK validation (technical receipt) for envelope/header/payload mandatory fields with specific error codes (1.0–1.9).
- Data transformation: Built conversion logic for field types (e.g., `collateralPercentage /100`, breakable rights mapping to ISDA conventions, date parsing `yyyyMMdd`).
- Correlation tracking: Implemented booking group creation using swapId + lotId pairs with D1-STP-SWAP-ID keyValue schema for duplicate detection.
- Exception handling: Added AtlasSystemException guards for null mandatory fields, unsupported enum values, bilateral rights mismatches, and duplicate trades.

### Real-Time Impact
- End-to-end STP: Equity swap trades now flow automatically from GCAM through Delta1 to Atlas with zero manual intervention.
- Reduced booking time: Eliminated manual Atlas booking step, cutting trade capture time from hours to seconds.
- Audit trail: L1/L2 ACK messages with correlationId and messageId enable full traceability across systems.
- Error detection: Early validation failures caught at L1 ACK prevent downstream booking issues and provide immediate feedback.
- System consistency: Both Delta One and Atlas maintain synchronized state using gcamTradeId and delta1ContractIdentifier references.

---

## 5. Drool Rule Validation in Atlas Trade

### What is Drool Rule Validation in Atlas Trade
- Business rules engine that validates trade operations before execution.
- Pre-validation purpose: Blocks invalid trade modifications at the web layer before sending to downstream systems (FISS).
- Executes declarative rules in `.drl` files against trade data during workflow steps.
- Prevents system inconsistencies and manual cleanup by catching violations early.
- Context: Atlas/Sophis → Atlas Web → FISS flow; validation occurs at Atlas Web boundary.

### Problem Resolved
- SSS deal condition trades cannot be processed by FISS system.
- Atlas/Sophis doesn't send SSS trades to FISS, but Atlas Web allowed amend/cancel operations.
- Created data inconsistency risk: Users could trigger amend/cancel that would fail downstream.
- Business rule: Amend/cancellations must originate upstream or use offsetting + rebooking approach.
- Solution: Block amend/cancellations for SSS deal condition trades at Atlas Web layer.

### Technical Changes Made
- Added new Drools rule in relevant `.drl` file under appropriate kiebase (likely MANUAL_AMEND_LISTED_TRADE or AMEND_LISTED_TRADE).
- Configured DroolsValidator in `trade-capture-validation-context.xml` with kiebase reference.
- Declared global variables in validation step (e.g., AtlasWorkflowManualListedTradeValidationStep).
- Created helper function in `dealAndTradeValidationMediator` to check SSS deal condition.
- Added unique drool rule number and error message in `tradeValidationErrors.properties`.
- Implemented integration tests to validate SSS condition blocking behavior.
- Rule fires when amend/cancel operation detected on trade with SSS deal condition.

---

## 6. Listed Trade Archival

### Problem Resolved
- Database growth: 2.3M+ trades accumulated monthly consuming 450GB+ storage.
- Query performance degradation: 40–60 second response times for trade retrieval queries.
- Server resource strain: CPU utilization spiking to 85–90% during peak trading hours.
- Backup/recovery window expanded from 4 hours to 12+ hours due to DB size.

### Technical Changes Made
- Analyzed trade data by book and instrument combination to identify archival candidates (trades older than 18 months).
- Developed SQL archival scripts to move trades from active tables to archive schema.
- Implemented unarchival mechanism for compliance/audit requirements (restore specific trades on-demand).
- Automated cleanup process integrated into each production release cycle.
- Added data validation checks pre/post archival to ensure zero data loss.

### Real-Time Impact
- Archived 18.5M trades across 6 releases, freeing 340GB database storage (75% reduction).
- Query performance improved: Average retrieval time reduced from 45s to 8s (82% improvement).
- CPU utilization decreased to 55–65% during peak hours (30% improvement).
- Backup/recovery window reduced to 5 hours (58% faster).
- Zero production incidents or data integrity issues post-archival.

---

## 7. Pending Indicative Prices Report for Basket Swaps

### Problem Resolved
- Indicative prices for basket swaps were not included in the existing pending indicative prices report (SQL ID 593).
- Clients received incomplete data - missing FX rates and indicative prices for basket instruments (both initial bookings and substitutions).
- Manual effort required to identify pending indicative prices for basket positions.

### Technical Changes Made
- Extended SQL query to retrieve indicative prices/FX rates from `payoff_overridden_observations` table for basket swaps.
- Mirrored existing column structure to maintain consistency with current report format.
- Implemented logic for two basket scenarios: initial basket booking and basket substitution events.
- Used UNION operator to merge basket swap data with existing non-basket results.
- Leveraged existing ATLAS Perl framework for report generation and email distribution.

---

## 8. Bulk Instrument Expiry Update

### Problem Resolved
- 50,000+ instruments with May 2026 expiry needed bulk update to May 2029 before expiration.
- Original JMX limitation: Max 200 instruments per call, 1 hour to process 1000 instruments.
- Manual overhead: Required multiple invocations, no progress tracking, no per-instrument error handling.
- Production risk: Large-scale update with zero tolerance for instrument corruption or downtime.
- Blocking operation: User had to wait synchronously for completion.

### Technical Changes Made
- File-based input: `updateExpiryDateFromFile()` reads unlimited instrument IDs from CSV/TXT on server.
- Asynchronous execution: CompletableFuture with ExecutorService (20 threads) for parallel processing.
- Batch processing: Instruments split into batches of 10, processed concurrently.
- Job tracking system: ConcurrentHashMap<String, JobProgress> stores job state with UUID-based job IDs.
- Thread-safe counters: AtomicInteger for total/completed/failed counts, ConcurrentHashMap for per-instrument errors.
- Calendar-aware expiry: Added `SwapCalendarUtil.rollDateByConvention()` for proper business day adjustments.
- Individual persistence: Each instrument persisted separately to avoid shadow booking conflicts.
- Progress monitoring: `getJobProgress()`, `listActiveJobs()`, `clearJob()` methods for observability.
- Path traversal protection: Filename sanitization to prevent security vulnerabilities.

### Real-Time Impact
- Performance: 30x improvement - 1000 instruments now process in 2 minutes vs 1 hour.
- Scalability: No instrument count limit (vs 200 cap), handled 50,000+ instruments successfully.
- Zero production incidents: 3 months daily monitoring, zero instrument corruption or data issues.
- User experience: Fire-and-forget async execution, check progress via job ID.
- Operational visibility: Real-time progress tracking (completed/failed/remaining), elapsed time, percentage.
- Error isolation: Failed instruments don't block others, detailed per-instrument error messages.
- Production validation: End-to-end testing in production environment without impacting live instruments.

---

## 9. Folder Move Automation

### Problem Resolved
- Manual overhead: RTB operators manually triggered multiple JMX calls for each folder move activity.
- Time inefficiency: Each folder move required repetitive manual steps across various JMX endpoints.
- Error-prone process: Manual triggering increased risk of missed steps or incorrect sequences.
- Report generation bottleneck: RTB had to wait and manually generate reports post-folder move.
- Operational dependency: RTB involvement required throughout the entire folder move lifecycle.

### Technical Changes Made
- Automated JMX invocation: Shell script consolidates multiple JMX calls into single execution flow.
- BAM token authentication: Implemented secure token-based authentication for automated JMX triggering.
- Autosys job integration: Offloaded report generation to scheduled Autosys jobs (eliminated manual step).
- Confirmation-based workflow: Script requires simple operator confirmation before execution (safety gate).
- Error handling abstraction: Centralized error detection—RTB only intervenes on failures.

---

## 10. Additional Enhancements and Defect Fixes

### ENTRY timelines on next reset payment date filtered incorrectly
- **Problem:** ENTRY timelines settled on the next reset payment date were filtered out as reset-start entries of the next period.
- **Impact:** Subsequent unwind timelines could fail as matching start positions were not found.
- **Change:** Updated timeline applicability logic to also consider effective date against reset date.
- **Outcome:** Valid ENTRY timelines are retained and unwind calculations work correctly.

### Multiple basket substitutions with same settlement date causing incorrect interest calculation
- **Problem:** Constituent-level interest calculations picked timelines belonging to later substitutions when multiple substitutions shared the same settlement date.
- **Impact:** Incorrect interest amounts were calculated.
- **Change:** Enhanced timeline filtering to match both settlement date and effective date.
- **Outcome:** Interest calculations now correctly identify the applicable substitution event.

### Accum/Deccum Redemption trades missing Nexus flag
- **Problem:** HKMA Nexus details present on Version 1 were missing on redemption versions.
- **Impact:** Trades were not reported correctly, resulting in stale positions.
- **Change:** Extended Nexus detail copy-forward logic to Redemption lifecycle events.
- **Outcome:** Nexus reporting remains consistent across trade lifecycle events.

### Migration of D1 reports to Job Processor
- Migrated D1 report processing from Atlas Extract Framework to Job Processor.
- Supported migration validation and execution flow changes.
- Improved alignment with standard processing architecture.

### PnL adjustment support for Interest Rate and Dividend Legs
- Added new trade capture and retrieval fields for Interest Rate Leg and Dividend Leg.
- Enabled support for applying PnL adjustment behaviour.
- Ensured downstream consumers receive the new attributes.

### Deal Group cancellation support
- **Problem:** Existing cancellation restrictions prevented users from cancelling deal groups.
- **Change:** Introduced new property `CanCancelDealGroup`.
- **Logic:** `CanCancel && AtlasBookingGroupIdentifier != null`.
- **Outcome:** Restored deal group cancellation capability while preserving existing controls.

### DEV3 environment expansion and RC deployment
- Configured DEV3 environment with newly added server ldnpsr000002789.
- Installed and validated new release candidate.
- Supported environment readiness and rollout activities.

### TCU allocation trade auto-approval issues
- Investigated auto-approval behaviour for allocation trade cancellations.
- Investigated auto-approval behaviour for allocation trade amendments.
- Analysed workflow and approval processing logic to identify root causes.

### Custom lot unwind payment calculation enhancement
- Implemented payment calculation changes for custom lot unwind scenarios.
- Validated calculation behaviour against business requirements.
- Improved accuracy of unwind payment processing.

### Listed trade subscriber analysis
- Identified subscribers consuming listed trade events through event logs.
- Analysed subscriber behaviour for booking, amendment and cancellation events.
- Verified subscriber registration for newly created contracts.
- Traced subscribers to processors and reviewed approval/trade status related logic.

### Approval flow simplification
- Removed internal booking related validation from approval workflow.
- Reduced unnecessary approval dependencies.
- Simplified overall approval processing.

### Synthetic stock approval enhancements
- Excluded internal counterparties from synthetic stock approval flow.
- Fixed issue where rejected approvals did not update trade status to PENDING_BOOKING.
- Improved approval workflow accuracy and trade state management.

### Hong Kong 4PM BFIX enablement
- Delivered Atlas Trade service changes required for Hong Kong 4PM BFIX onboarding.
- Supported production enablement activities.
- Validated service-side readiness for BFIX processing.

---

## 11. Production Support Contributions & Business Impact

### Trade Publishing Delay Investigation (EDW / POG Integration)
- Investigated delayed publication of trade 1783218823 which impacted NYK batch processing.
- Analysed Atlas, Sophis and POG message flow timestamps to identify where delay occurred.
- Verified Atlas processing and shadow booking completed successfully without delay.
- Confirmed trade was published from Atlas to POG immediately after booking.
- Correlated publish and acknowledgement timestamps, demonstrating delay occurred downstream after POG publication.
- Provided SQL queries and evidence for POG team investigation.
- Coordinated with RTB and downstream support teams for root cause analysis.
- Identified log retention limitations and established process for earlier incident reporting to enable future investigations.
- Helped close the issue by documenting findings and confirming no recurrence was observed afterwards.

### Reuters Market Data / Observation Fixing Delay
- Investigated delayed Reuters close price fixings for Australian underlyings.
- Performed log analysis across AMD scheduler and Atlas Observation services.
- Compared expected fixing publication time against actual observation receipt time.
- Validated Reuters market data was published by AMD at scheduled time.
- Identified Atlas had not received or recorded the expected publication during impacted periods.
- Collected and presented evidence linking AMD publication and Atlas observation behaviour.
- Coordinated with market data support teams to isolate potential Reuters registration and processing issues.
- Confirmed affected fixings subsequently resumed arriving on time.
- Reduced business uncertainty by providing technical evidence and narrowing investigation scope.

### Loan Approval Pending Workflow Issue
- Investigated production cases where loan approval tasks remained pending and could not be approved.
- Analysed approval workflow errors reported by business users.
- Validated impacted trade states and workflow execution path.
- Worked with support teams to identify approval bottlenecks.
- Assisted RTB and operations teams in restoring approval processing and reducing user impact.

### Listed Order ID Stamping Investigation
- Investigated execution records where status completed successfully but LISTED_ORDER_ID was not populated.
- Performed database analysis on execution lifecycle data.
- Reviewed execution status transitions and trade workflow processing.
- Provided evidence to identify discrepancy between completion status and order stamping.
- Assisted downstream teams in root cause analysis and resolution.

### Approval Flow Defect for Matured / Expired / Exercised Trades
- Investigated production issue where approval workflow was incorrectly skipped during amendments.
- Performed code-level analysis of approval eligibility logic.
- Identified gap in trade status validation for full novation scenarios.
- Determined that matured trades were excluded from amendment approval checks.
- Proposed and validated logic enhancement to include additional lifecycle statuses.
- Prevented unauthorised workflow bypass and improved control compliance.

### Overall Production Support Value Delivered
- Acted as first point of technical investigation for RTB and business-reported production incidents.
- Performed log analysis, SQL investigation, workflow tracing, and code-level debugging across Atlas, Sophis, Reuters, POG and EDW integrations.
- Collaborated effectively with RTB, BTB, market data, downstream platform and infrastructure teams.
- Reduced incident resolution time by quickly identifying probable failure domains and providing evidence-based findings.
- Improved operational stability by documenting root causes, proposing fixes, and preventing recurrence of production issues.
- Maintained smooth business operations by supporting critical trading, reporting, market data and approval workflows.

---

## 12. BATHS Automation and QA Support

- Developed and maintained BATHS automation integration tests across multiple Atlas Trade products including FX Spot, NDF, Swaption, Cross Currency Swaps, and Individual Trader workflows.
- Built and enhanced FX Spot test harnesses, enabling automated validation of end-to-end booking and processing scenarios.
- Created product-specific automated test coverage to improve regression testing and reduce manual QA effort.
- Designed and developed REST API test packs covering all supported Atlas Trade products and business flows.
- Partnered closely with the QA team to identify automation gaps and expand test coverage for new features and releases.
- Investigated, debugged, and resolved automation test failures, ensuring stability and reliability of CI/CD test executions.
- Provided continuous support for test maintenance, framework enhancements, and new scenario onboarding within the BATHS framework.
- Validated complex trade booking scenarios across multiple product types to ensure functional and integration-level correctness.
- Contributed to improving overall release quality and regression confidence through automation-first testing practices.
- Acted as a key liaison between Development and QA teams, accelerating issue resolution and test implementation.

---

## 13. Spring Boot 2.7.5 Migration

### Migration Steps
1. Upgrade rods-parent to Spring Boot 2.7.5 (centralized dependency management).
2. Upgrade H2 from 1.4 to 2.1.120 with schema changes (IDENTITY syntax).
3. Upgrade MSSQL driver from v9 to v10 with encryption defaults.
4. Update each application (memphis-processor, memphis-rt, rods-delimited-file-processor).
5. Modify `application.properties` for H2 compatibility modes and MSSQL encryption.
6. Refactor `schema.sql` files to use `GENERATED BY DEFAULT AS IDENTITY`.
7. Set Kafka schema registry URL as optional in producer/consumer configs.

### Problem Resolved
- Security vulnerabilities in H2 1.4.x and unencrypted MSSQL connections.
- No long-term support on Spring Boot 2.6.8 (outdated).
- Performance gaps: slow startup, high memory footprint, poor resource utilization.
- Breaking changes risk: Direct upgrade to 3.0 would require extensive refactoring.

### Technical Changes Made
- rods-parent: Spring Boot 2.7.5 dependency; Kafka schema registry URL optional.
- H2 syntax: `IDENTITY(1,1) NOT NULL` → `GENERATED BY DEFAULT AS IDENTITY`.
- MSSQL connection: Added `encrypt=true;trustServerCertificate=true` to datasource URLs.
- H2 properties: `MODE=MSSQLServer;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false;CASE_INSENSITIVE_IDENTIFIERS=TRUE`.
- Reserved keywords: Handled via NON_KEYWORDS or quoted identifiers.
- Primary key fix: Added secKey2 as PK in RODS_Memphis_DIM_Security.
- Test adjustments: Manual trimming of trailing spaces in char columns (workaround).

### Real-Time Impact
- Improved security: H2 vulnerabilities patched; MSSQL connections encrypted by default.
- Performance gains: Faster startup times, reduced memory usage.
- Breaking changes: Apps won't run without encrypt/trustServerCertificate configuration.
- Test failures: H2 2.x backward incompatibility broke existing schema definitions.
- Deployment risk: Requires coordination across 6+ microservices.

# Detailed Guide: Integrating SonarQube with GitLab Pipeline for Spring Projects at Barclays

Here's a comprehensive explanation of how to integrate SonarQube with your GitLab CI/CD pipeline for Spring/Java projects at Barclays:

## Understanding the Architecture

### Instance-Level Integration
Barclays' GitLab instance is configured at the instance level with SonarQube. This means:
- Automatic project synchronization between GitLab and SonarQube
- Native integration capabilities built-in (no manual configuration needed)
- Short-lived tokens automatically minted through CSM (Central Secrets Management)
- Projects are dynamically created on first scan using `$CI_PROJECT_NAME`
## Prerequisites
- **Java Version Requirements:** Java 17+ (Java 21 recommended for new projects); Spring Boot 3.2.x or higher (if using Spring Boot)
- **Build Tool Configuration:** Maven wrapper configured; Gradle wrapper configured (version 8.5+)
- **Minimum Dependency Versions:** Spring Boot: 3.2; Mockito: 5.0.0; Jackson: 2.17.0; JaCoCo: 0.8.11; Logback: 1.5.6; Log4j: 2.17.0; Lombok: 1.18.30
- **Access Requirements:** SonarQube system account (part of analysis group with "read issues" permission); Central Secrets Management (CSM) access; GitLab role configured for CSM secret access
## Step 1: Set Up Security Credentials
- Generate a SonarQube API token from a system account (not your personal account)
- The system account must be part of the analysis group
- The account needs permission to read issues (for native GitLab integration)
- Store the generated token in Central Secrets Management (CSM)
- Use the key-value secret pair name: `SONARQUBE_SCAN_TOKEN`
- Configure a GitLab role that maps to this secret
- Navigate to your secret in CSM, click "View secret access config", and note `csm_role`, `csm_auth_url` and `csm_secret_url`
## Step 2: Understanding the Workflow
- **Trigger:** Pipeline runs on commit/merge request
- **Authentication:** Component retrieves token from CSM automatically
- **Build:** Your Spring project is compiled
- **Scan:** SonarQube scanner analyzes code quality
- **Upload:** Results are sent to SonarQube server
- **Quality Gate:** SonarQube evaluates against quality standards
- **Feedback:** Results appear in GitLab merge request
## Scan Types Available
- **maven-plugin:** Uses SonarScanner Maven plugin (recommended for Maven projects)
- **gradle-plugin:** Uses SonarScanner Gradle plugin (for Gradle projects)
- **sonar-project-properties:** Uses standalone scanner with sonar-project.properties file
## Native GitLab Integration Features
- Pipeline Code Quality Report
- Merge Request Decorations
- Code Quality Widget
- Merge Request Comments
- Quality Gate Enforcement
## Best Practices for Accurate Results
- Use Merge Request Pipelines
- Configure your project to use Merge Request Pipelines
- This ensures SonarQube analyzes only the changes in the MR
- Provides differential analysis (new issues vs. existing issues)
- Test all configuration changes in a merge request before merging to main branch
- Failure to test can result in false positives or unexpected results
## Monitoring and Troubleshooting
- **In GitLab:** Check pipeline logs, code quality reports, merge request widgets and decorations
- **In SonarQube:** Navigate to the dashboard to view detailed metrics, code smells, bugs, vulnerabilities and historical trends
- **Quality Gate Failures:** Review issues, fix them or use override mode if justified, and document reasons
- **Authentication Errors:** Verify CSM role, token permissions and read issues permission
- **Scan Not Running:** Verify builder image, CSM URLs and pipeline logs
## What Makes This Barclays-Specific
- **Strategic Platform:** Uses Barclays' enterprise SonarQube instance
- **CSM Integration:** Leverages Barclays' Central Secrets Management
- **Builder Images:** Uses standardized Barclays builder images
- **Pre-configured:** Instance-level setup eliminates manual configuration
- **Compliance:** Automatically enforces enterprise quality standards
- **Security:** Short-lived tokens, no static credentials in pipelines
## Maven pom.xml Configuration

```xml
<properties>
    <!-- Existing properties -->
    <java.version>21</java.version>
    <spring-boot.version>3.2.x</spring-boot.version>
    <sonar.projectKey>${project.artifactId}</sonar.projectKey>
    <sonar.projectName>${project.name}</sonar.projectName>
    <sonar.java.coveragePlugin>jacoco</sonar.java.coveragePlugin>
    <sonar.coverage.jacoco.xmlReportPaths>${project.build.directory}/site/jacoco/jacoco.xml</sonar.coverage.jacoco.xmlReportPaths>
</properties>
```

## Maven-based .gitlab-ci.yml

```yaml
stages:
  - scan
include:
  - project: barclays/devtooling/images/builder-images
    file: builder-images.yml
  - component: $CI_SERVER_HOST/barclays/gitlab/components/sonar/scan@release/1.x
    inputs:
      scan_type: maven-plugin
      builder_image: $BUILDER_IMAGE_JAVA_21
      csm_role: <your_gitlab_role_assigned_to_sonar_secret>
      csm_auth_url: <your_csm_auth_url_for_sonar_secret>
      csm_secret_url: <your_csm_secret_url_for_sonar_secret>
```