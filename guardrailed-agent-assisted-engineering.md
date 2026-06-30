# Guardrailed Agent-Assisted Engineering: A Personal Use Case and Experience
## Preface

**Author notes: I am not a mobile developer**, I came to this project with software engineering experience but no prior iOS platform knowledge. I state this at the outset because it is the single most important design constraint in everything that follows. This writeup is not meant to be suggestive or representative of the **specifics** of how I think a governance model should be adopted; it does walk through an reference process towards the end that shows how adopting or evolving a system *could* be done. It is also meant to highlight my own experiences (successes, failures, etc.) and to suggest that intensive focus should be lent towards quality management within an overall system, under which agents, models, harnesses, skills, plugins, etc. all operate.

---

## 1. Executive Summary

In early 2026 I built a personal-use iOS parental-control application using a guardrailed agent-assisted workflow. The 'why' is effectively that I have two young daughters who are going to grow up and need to be equipped to navigate social media and the vast ecosystem of smart device applications in a healthy way. I have a huge part and role as a parent to ensure that I'm doing my part to productively educate them; but I also wanted a mechanism to set boundaries on what can/can't be accessed to leave ample space to allow for education and other methods. As noted below, I found some shortcomings with what is in the market when compared to my own desires for what I would want in a product, so I started working on this project.

My general thought process has been that if I invest in the overall governance system and a sound context base that I should be able to create something that is safe enough for *personal* use. My security and privacy background and experience building services on AWS was definitely helpful from an infrastructure perspective. 

The system is not autonomous. Every architecture decision, security invariant, and test mandate was authored or explicitly approved by me. AI coding agents contributed implementation code, diagnostic hypotheses, and document drafts. Human checkpoints controlled all specification, deployment, merge, and security decisions.

The project is not commercially deployed. Its risk posture — bounded household users, no regulated data at scale, no uptime SLA, Apple's own system-level framework as an independent backstop — made it an appropriate scope for a structured experiment. Nothing in this document implies that the methods or posture described here are sufficient for commercial, public, regulated, safety-critical, or high-availability software. That distinction is addressed explicitly below.

What this case study documents: a structured approach to using AI coding agents productively on a technically ambitious personal project where someone lacks prior domain expertise in certain areas, while keeping human judgment in control of every consequential decision, and documenting what went wrong along the way, using 'what went wrong' as a feedback loop for continual improvement.

---

## 2. Why This Was an Appropriate Bounded Experiment
The application is personal-use only, not distributed on the App Store, with a user population of one parent and a small number of devices in a single household. This scope imposes four conditions that made the experiment honest rather than reckless:

- **No regulated data at scale.** The system handles credentials and policy configuration for a household, not a commercial user population subject to COPPA, GDPR enforcement, or HIPAA. However, given my background in security and privacy the setup is still hardened to adopt the controls you would see in compliant workloads subject to the aforementioned regulations.
- **An independent system-level backstop.** Apple's Screen Time/FamilyControls framework enforces restrictions at the OS level, independent of the application's code quality. A bug in the app's policy logic produces a user-facing inconvenience, not a safety event. I did not pursue developing custom extensions, etc. because I fully concede my own working knowledge limitations.
- **Single developer, full accountability.** I made every consequential decision and bear all consequences. There is no risk of diffused accountability masking agent-generated defects across an organization.
- **Reversible failures.** Every failure mode  is recoverable: re-pair the device, re-apply restrictions, redeploy the backend.

These conditions do not generalize to commercial software. They are the reason the experiment could be conducted honestly, with documented gaps rather than concealed ones.

---

## 3. The Product Problem
Public data spanning more than thirty commercial parental-control products revealed two gaps that motivated building rather than buying:
- **iOS enforcement resistance.** Existing products that do not require Mobile Device Management (MDM) enrollment cannot enforce restrictions that resist switching browsers or uninstalling monitoring apps. Apple's FamilyControls framework, available to approved developers under a distribution entitlement, provides OS-level enforcement that commercial MDM-free products cannot match.
- **Emerging content categories.** AI chatbot interfaces and newer social media surfaces were not adequately addressed by the surveyed products at the time of the March 2026 market analysis.

The decision to build rather than buy was deliberate. I accepted the full maintenance burden and Apple entitlement gatekeeping risk in exchange for enforcement that operates at the OS level rather than at the app layer.

There are two separate entitlements that I focused on:
- Apple's FamilyControls entitlement (approved) which is the major one that was developed against.
- URL Filtering is a separate entitlement that I'm exploring to do more proxy-esque type control to address use cases where FamilyControls may not be able to assert desired controls.

---

## 4. My Starting Knowledge Gap

I came to this project with some software engineering experience and cloud infrastructure experience but no prior iOS platform knowledge. This is not a footnote, it is the primary design constraint. Every structural decision described in this document is partly a response to that gap and I used feedback loops via newly identified gaps to 'feed' the system with improvements.

Specifically, I leveraged agents/models to establish a knowledge baseline using reputable sources in the following areas that I did not have independently from experience:

- Apple FamilyControls and ManagedSettings API surface: how `ManagedSettingsStore` and `DeviceActivityMonitor` interact with the OS.
- SwiftUI `@StateObject` lifecycle semantics — the source of a documented persistence bug that I uncovered.
- Xcode `.pbxproj` file structure and the four-entry requirement for adding new build targets.
- The `convertFromSnakeCase` + explicit `CodingKeys` interaction order — the source of a recurring silent failure class that I came across.
- ECDSA ES256 JWT design pattern for device-scoped authentication tokens. I have design experience here but not hands on implementation experience.

In each case, the workflow required verification before encoding the knowledge as a project rule. Agent-supplied knowledge was confirmed against Apple Developer Documentation, Swift documentation, or — most reliably — a failing test that made the behavior observable before it was committed.

---

## 5. The Guardrailed Agent-Assisted Engineering System (the "system")

### Multiple Agents, One Governance Harness

The project used multiple agentic platforms (for lack of a better term).
- **Frontier Providers** — Anthropic (Claude Code), OpenAI (Codex)
- **Open Source Providers** - Ollama (predominantly cloud models, GLM (latest) and Kimi (latest) models)
- **Harness** - tailored harness that takes into account my shortcomings from an experience perspective to layer in mitigating guardrails, built around Matt Pocock's [skills repository](https://github.com/mattpocock/skills) and the [Pi Coding Agent](https://pi.dev/)
All agents (opus, sonnet, gpt, glm, kimi) are subject to the same set of skills, context and progressive disclosure approaches, etc. to maintain consistency. I also use this setup in other projects and dogfood the system against itself. (for better or worse)

The system is designed around a provider-neutral core — governance profiles, domain glossary, workflow skills, and coding standards — so the same policy documents apply regardless of which agent executes a task. Something that is on the short list is creating a single set of policy definitions paired with a mechanism to deploy for multiple provider context. (Codex, Claude Code, Pi, etc.) I have not worked through this just yet.

The governance model described here is personal-scale: one developer, the provider set up mentioned, bounded personal projects. Below I address what would need to change to support scaling.

### Governance Architecture

I broke out the system development work into phases:
**1 -  Advisory:** Policy exists in documents. Agents are instructed that they should not access credentials, personal files, or unrelated data. No runtime enforcement prevents a violation. This is a directive approach from a controls perspective.

**2 - Partially Preventive:** Advisory (above) is mapped to managed permission profiles for one provider (Codex, in my case) on macOS. Hard-denied paths include credential directories, package authentication files. The managed profile is installed manually through MacOS System Settings, I don't use an MDM today. 

### Autonomy Boundaries

Three tiers of agent action are explicitly defined:

- **Autonomous (no human approval required):** Reading project-local files, running project checks, creating temporary files in a designated location, staging issue bodies for review, running non-mutating git commands, updating the domain glossary during modeling sessions.
- **Requires human approval:** Publishing issues to GitHub, applying the `ready-for-agent` implementation gate, starting implementation, adding dependencies, touching external services, pushing code, handling security-sensitive disclosures.
- **Prohibited entirely (harness baseline):** Reading credential paths, reading active `.env` files, reading personal directories outside the code tree, copying secrets into documentation or tests, using the provider's bypass-approvals flag outside a separately trusted sandbox.

In addition to system prohibitions, I leverage project-level prohibitions that prevent the coding agent from performing any AWS deployment operation. Deploy operations require human SSO authentication and human judgment about the state of the staging environment.

### Human Decision Points
Ten explicit human checkpoints are used in my standard workflow. They include: approving terminology during domain modeling; approving the Product Requirements Document before publication; approving issue-body specifications and breakdown; applying the `ready-for-agent` label; confirming the test plan before TDD coding begins; and squash-merging the PR after CI passes. In some cases a skill or workflow leverages other skills via sub-agent dispatch, but there is always opportunity for human interjection. All merging is done manually.

---
## 6. Architecture and Major Decision History
Decision-making is context embedded into each project. A list of some example decisions made using the overall system are below:
### Build vs. Buy 
Analysis of 30+ commercial products revealed coverage gaps in iOS-native enforcement and emerging content categories. I decided to build rather than buy, accepted maintenance burden in exchange for OS-level enforcement.
### Single Binary, Role-Based UI
One iOS binary serves both the device and managed devices; the onboarding flow selects the role. This simplifies Apple Developer account management (one bundle identifier) but requires correct role-conditional view rendering everywhere. I wanted to keep this simple because I am one person and I also spent time researching and working through the Apple Developer setup so that if it broke at any point, I understood it well.
### Custom ECDSA Device JWT Instead of Cognito
The original design transmitted the guardian's Cognito credentials to managed devices — a credential isolation violation. The replacement: managed devices authenticate via a custom ES256 P-256 JWT obtained via a single-use QR pairing token with a 10-minute TTL and a 256-bit random value stored as a SHA-256 hash in the database  Credentials never appear on managed devices. There are compelling security reasons for how I changed this, but also, budgetary reasons.

### Single-Table DynamoDB
One DynamoDB table with composite patterns. It's acceptable at personal-use scale but at scale would need to change given the implication is an entire table scan for certain things, so this gets expensive quickly when scale is large.

### Atomic Multi-Item Writes 
All writes touching multiple DynamoDB items use `transact_write_items` with a `ConditionExpression` guard to mitigate a split-brain state that would result from a Lambda crash between two writes. A 409 is returned in a failed state to mitigate silent failures.
### Certificate Pinning to AWS Root CAs
The iOS networking layer validates TLS connections against five pinned Amazon Trust Services root CA SHA-256 hashes. Non-matching server certificates cancel the connection. Trade-off: Amazon root CA rotation — rare and announced with significant lead time — would require an app update to restore connectivity. Root CA rotation is a known operational risk accepted as manageable for a personal-use app. I also have direct experience with the pains of cert pinning, PKI, intermediate and root certs and while a risk, knowing what I know today (via pain), I feel comfortable with this part of the architecture.

### Security Scanner Installed from Immutable Artifact, Not PR Source
The CI security scanner is installed from an immutable release artifact in a private package repository, not from the PR's changed files. This mitigates a supply-chain substitution attack where a malicious PR replaces the scanner with a version that always passes. Failure is configured to be closed: if the scanner produces no output, CI fails.

Additionally, throughout all of this "SaaSpocalyse" content (nonsense) I pressure tested what building your own security linter looks like. There are reasons there are SAST companies and projects that have significant contribution. I ended up with a combination of semgrep (can tailor to project), native language/tech security linters (e.g. bandit), tailored git-leaks, etc. that I pruned to be tailored to my project context.

### Dual-Sided iOS-Backend Contract Tests
After the fifth occurrence of the same silent field-name decoding failure at the iOS/backend boundary, dual-sided contract tests were mandated. This is admittedly an oversight I had during planning that I would eventually figure out, but not without pain. One Python test serializes the Pydantic model and asserts exact JSON field names. One Swift test decodes the same JSON with the production decoder configuration. The backend side is CI-enforced. The iOS side runs locally only at the evidence date. Part of the reason for running iOS testing locally is for budgetary reasons and some hit or miss experiences using cloud MacOS runners (which also, budget considerations). Debugging the tests locally is simply easier for me and more budget-friendly.

### TDD as a Non-Negotiable Project Rule
Writing tests aren't fun. Red-Green-Refactor is written into the project configuration as non-negotiable with no exceptions. The configuration instructs agents/models to stop and delete implementation code if it was written before a failing test. This compensates for a developer without deep domain expertise by ensuring that agent-generated code has a behavioral specification in the form of a failing test before implementation begins. Without this constraint, a knowledge gap in the developer would have compounded a knowledge gap in the agent. I needed to make up for my shortcomings.

---

## 7. Quality-Management Layers
Quality controls divide into two categories: those enforced by CI on every pull request to `main`, and those that are locally executed but not CI-gated. The distinction matters and is stated explicitly for every control below. Re: local tests I noted above why I chose to do local testing for certain things vs. rely on cloud runners using MacOS.
### CI-Enforced on Every Pull Request

| Control                                | What It Enforces                                                                                                   |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| pytest with ≥80% line-coverage minimum | Backend logic paths reach ≥80% line coverage; CI exits non-zero below threshold                                    |
| ruff lint and format                   | Python style, import order, naming, and antipattern detection including security-sensitive rule sets               |
| bandit static analysis                 | Common Python security antipatterns: hardcoded secrets, shell injection, dangerous builtins                        |
| pip-audit                              | Known CVEs in direct and transitive Python dependencies                                                            |
| Custom security linting (fail-closed)  | Multi-tier scan; installed from immutable artifact store; absence of output is treated as failure.                 |
| TypeScript type-check (`tsc --noEmit`) | TypeScript type correctness in the web dashboard                                                                   |
| ESLint                                 | TypeScript/TSX style and rule conformance                                                                          |
| Next.js production build               | Dashboard production build succeeds; catches SSR and static-export errors that type-checking misses                |
| GitHub Actions SHA pinning             | All CI workflow action references are pinned to full commit SHAs, preventing tag-mutable supply-chain substitution |
| OIDC keyless AWS credentials           | No static AWS access keys in the repository or GitHub Secrets                                                      |
| Staging health check after deploy      | API Gateway and Lambda return HTTP 200 on `/health` after every deploy to `main`; failure is blocking              |

### Locally Executed, Not CI-Gated
The below is not CI gated but I do leverage pre-commit hooks for some:

| Control                                         | What It Covers                                                                                                                                               |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| iOS XCTest suite (~287 functions, 34 files)     | Networking, services, state, utilities, policy logic on a simulator on MacOS                                                                                 |
| Dashboard unit tests (~23 files, vitest)        | Auth hooks, API client, page renders, form flows, middleware                                                                                                 |
| Playwright E2E (login, session, dashboard home) | Dashboard flows against a local development server, Firefox and mobile Firefox. This is for a local dashboard that I use for testing/conceptualizing things. |
| CDK infrastructure tests (61 assertions)        | Stack synthesis, route registration, Lambda bundling, Cognito configuration                                                                                  |
| mypy strict-mode type checking                  | Backend Python type safety                                                                                                                                   |
| Pre-commit ruff and ESLint hook                 | Staged file quality before commit                                                                                                                            |

### What These Controls Do Not Prove

Tests do not prove absence of defects or security issues! 
- I use a number of (budget-friendly) security linters and some one off freemium model providers to do checks.
- This has not been pentested formally by a reputable third party.
Security is of course never guaranteed.

The specific limitations relevant to this project are:
- No test verifies that a specific application or category is actually blocked on a physical device after a policy is applied. All restriction tests use empty token sets or mock `ManagedSettingsStore` instances because real `ApplicationToken` objects require a live Screen Time authorisation context that cannot be created in a simulator environment. 
	- To compensate for this I do manual testing on the app deployed to my phone. I'd love to get to this level of testing, eventually, but I'm on the cheapo setup.
- The CI coverage gate ensures that at least 80% of backend source lines are executed by the test suite. It does not prove that executed lines assert correct behavior, that branch coverage is adequate, or that mocked AWS services accurately reflect live service behavior. 
	- To compensate for this I also do some manual testing and leveraging CDK helps preserve a consistent configuration baseline (vs. custom scripts). Same as the above regarding how I'd love to move to a matured setup here, eventually.

---

## 8. Threat Model and Autonomy Boundaries
### Credential Isolation Architecture

The device authentication architecture was redesigned to eliminate a documented credential isolation violation:
- **Parent accounts** authenticate via Cognito with optional TOTP MFA and enforced password complexity (uppercase, lowercase, digits, symbols, minimum 8 characters).
	- Eventually I will support social identity model to ideally rely on upstream authentication complexity and capabilities. 
- **Managed devices** authenticate via a custom ES256 P-256 JWT scoped to `(device_id, profile_id, owner_id, role=managed_device)` with a one-hour expiry. The JWT is obtained by scanning a QR code encoding a single-use pairing token with a 10-minute TTL.
- The device pairing architecture was redesigned specifically to remove guardian credentials from the managed-device path. The original design was replaced with a QR-based single-use pairing token flow; the replacement commit is documented in version history. Full automated verification that no code path incidentally passes credentials is not separately evidenced beyond the architecture and that commit.
	- In the future this is a testing gap that I would love to close. For now, since this is an architecture limited to personal use the risk is lower since I am the only 'tenant', but in a multi-tenant context this is a hard requirement.
- Device credentials are stored in the iOS Keychain with `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`, preventing iCloud Keychain synchronisation.
- Certificate pinning is implemented in the iOS networking layer: the validator checks connections against five pinned Amazon Trust Services root CA SHA-256 hashes and cancels non-matching connections. The validator is tested in isolation. Whether the pinning delegate is correctly wired to the production `URLSession` instance is not separately verified by an automated integration test — this is a documented coverage gap and part of future plans.

### Eleven Codified Security Invariants
Eleven named security invariants are encoded in the project configuration, each with a CWE citation, the affected code pattern, correct and incorrect code examples, and a reference to the issue where the vulnerability was first encountered. 
- These govern agent behavior: agents are required to follow these invariants when generating code and to flag violations during review.
- This pattern is the output of using the aforementioned security tooling and then creating a feedback loop where upon detection that security pattern is then documented in way that it is loaded as progressive context where needed *in addition to* lint inspection.

Two examples that trace directly to documented issues:
- **CWE-841 / 362 (DynamoDB multi-item writes):** All writes touching multiple DynamoDB items must use `transact_write_items` with a `ConditionExpression` guard. Violations produce split-brain state under Lambda retry as mentioned above. 
- **CWE-284 (Cognito admin API):** The JWT `sub` claim must not be used as the Cognito `Username` parameter for admin operations. A silent `UserNotFoundException` was caused by this mistake in a prior implementation. This was also particularly painful because it highlighted failure mode / error handling / logging gaps, so the debugging was rough.

### Agent Autonomy Limits on Deployment and Security-Sensitive Actions

The project configuration explicitly prohibits the coding agent from performing AWS deployment operations
- This is a documented prohibition, not a soft recommendation. Deploy operations require human SSO authentication and human judgment about staging environment state. I manually run 'cdk' commands in terminal.
- For certain 'gh' actions I will allow the agent to be instructed to create issues, however, I need to approve the formatting, details, etc.  I currently only leverage the agent/model for formatting of PR descriptions as well.

The hard-deny list prevents an agent/model from reading credential directories, package authentication files, Docker and Kubernetes configuration, and active workspace environment files. The list covers documented concrete paths:
- Sensitive data stored at unlisted paths and external tool surfaces — MCP connectors, browser automation, application connectors — require separate review. As mentioned above I currently only use playwright today.
- This also includes `.env` files. Whenever the contents of these files are needed I have to manually do things like stand up something on `localhost:*`, etc.

## 9. A Failure or Correction That Improved the System

### Five Occurrences of the Same Bug Before Structural Prevention

The single most instructive failure pattern I experienced was a repeated class of bug rather than a single incident.

**The failure:** a field-name mismatch between the Python backend (which serialises to `snake_case`) and the iOS Swift decoder (which uses `convertFromSnakeCase` plus explicit `CodingKeys` mappings) produced a silent `DecodingError.keyNotFound`. The iOS feature would stop working with no user-visible error message and no CI signal. So I had translation problems and silent failure, note the above reflection on lack of failure mode observability / error handling / logging.

This failure class occurred **five times** before a structural solution was adopted. I suppose I am a glutton for punishment:
- **Why the same bug survived five iterations:** iOS unit tests used hardcoded mock JSON that matched the expected shape. When the backend model changed, the mock did not update. The iOS test passed; the feature broke silently on a device. This was a hard lesson in bridging that gap in development and testing.
- **Agent diagnosis:** An agent/model eventually identified the `convertFromSnakeCase` + explicit `CodingKeys` interaction order as the mechanism using one of my systems skills. The order in which Swift's `JSONDecoder` applies key transformations means that explicitly-mapped `CodingKeys` values must account for the transformation that has already occurred. This was confirmed by writing a failing test and observing the specific `keyNotFound` error with the affected key name — not just "decoding failed".
- **Human decision:** After the fifth occurrence, I mandated dual-sided contract tests as a non-negotiable structural pattern. The agent/model handed me the root cause to make a decision around. The developer made the structural mandate. This distinction matters: the agent identified the cause; the developer decided that the fix must be systematic rather than per-incident.
- **The structural fix:** One Python test serialises the Pydantic response model and asserts the exact JSON field names in the output. One Swift test decodes the same JSON using the production `JSONDecoder` configuration. These tests catch field-name divergence at the schema level before it reaches a device.
- **What remains open:** The iOS side of this test runs locally only; it is not CI-enforced at the evidence date. There are reasons for this, as I mentioned above. Coverage extends to device-pairing response models; other iOS-consumed API endpoints may not yet have dual-sided coverage.

---

## 10. Project Statistics (through end of June-2026)
Some data that might be interesting, or not, that I pulled out of the iOS project:
- **Commit history:** 50 commits, single author, all using what I believe is a conventional commits format.
- **Test Suite**
	- **Backend:** 327 test functions across 33 files, organized by API router, repository layer, service, authentication, configuration, and data model domains. Enforced at ≥80% line coverage on every pull request.
	- **iOS:** 287 XCTest functions across 34 files, covering networking (API client, authentication manager, certificate pinning, endpoint construction), services (authentication, device authentication, policy, activity reporting), utilities (Keychain, logging security, token management), and state management.
	- **CI pipeline runs:** Four independent CI jobs: backend tests, backend lint and security scanning, custom security scanner, dashboard build, run on every pull request. 26 historical security scanner run records with run IDs, timestamps, and findings are posted to the PR as a comment for review and subsequently fixes where needed or tailoring the scanner with false positive feedback.
	- **Structural correctness evidence:** The CDK FastAPI-to-CDK route cross-reference test was drafted with agent assistance and caught 13 real missing API route registrations in a single commit. A different perspective of this might be that my directive system to the agents/models weren't good enough!
- **Incremental Progressive Context**: I currently have eight progressive disclosure documents record real failures with investigation logs, root causes, fixes, and prevention rules that updated the project configuration reflective of my feedback loop taking place over time.


---

## 11. What This Demonstrates

**A developer without domain expertise can build a working application using agent/model assistance, if compensating controls are proportionate to the knowledge gap.** -  The iOS-backend contract test pattern, the security invariant codification, and the progressive disclosure document structure all trace to domain knowledge the developer did not have independently. All were addressed through verified agent contributions followed by human-mandated structural prevention.

**There is a fine balance to how you restrict agent/model contributions to be bounded, verifiable, and non-autonomous.**
- Every agent-generated implementation was preceded by a failing test or had to produce a passing test to count as complete. Agents were explicitly prohibited from deploying, merging, or managing Apple Developer account operations. Architectural proposals required human approval before implementation began.
- This was tailored to *my* working style, though, and it will differ from others who have different appetites for risk, different working knowledges, etc.

**Incident-driven documentation, when encoded as durable project rules, prevents recurrence of specific failure classes.** 
- Eight progressive disclosure documents each produced at least one configuration update. The field-name mismatch failure class occurred five times and has not been evidenced as recurring after the dual-sided contract test mandate.
- If we go up a layer to concept this is an example that ties back into building out context and, in turn, thinking about how you orchestrate progressive disclosure in the most effective way possible.

**A governance system can define consistent autonomy boundaries across multiple agent providers through shared policy documents and provider-specific enforcement adapters.**
- "Can" is maybe not being genuine to the fact that it is a lot of work. However, the system I am test driving and dog-fooding can distribute configurations that are provider-neutral, with nuances for certain configurations. Of note, I've only done this for the platforms/tools referenced above and still have some future work to nail down to extend beyond just Skills + Context Management.

---

## 12. What This Does Not Demonstrate

**This does not prove that tests demonstrate absence of defects or security issues.** 
- The 80% coverage gate ensures 80% of source lines are executed; it does not prove that all executed lines behave correctly. The security scanner passing means no configured rules fired at the current threshold; it does not mean no vulnerability exists. 
- Security is never guaranteed and there is nothing in this system that orchestrates penetration testing by an accredited, reputable, third party. I would not feel comfortable making something like my iOS project commercially available without some expertise in mobile and AWS/backend pentesting performing a review.


**This does not prove the approach scales to commercial team development nor organization adoption.**
- My workflow depends on a single developer who authored all specifications, reviewed all agent output, and retained all deployment and merge authority. Distributing those responsibilities across a team introduces coordination, attribution, and accountability questions this project did not face.
- What I've built does not imply that a company could adopt it "off the shelf"

**This does not prove that the two-axis review provides independent verification.** The review sub-agents are the same class of model that produced the implementation. They provide a second pass with separated concerns; they do not provide the independence that an external reviewer with a different knowledge base would.

---
## Conclusion

Agents are most safely used as accelerants within a well-designed set of human-gated controls. What made me successful was acknowledging shortcomings, embracing strengths and then designing a system with gates that result in quality outputs.

###  Major Takeaways
- **The system matters more than the model:** (ducks)
	- It's easy to lose sight of the overall system with news cycles that are keyed in on Opus X now outpaces GPT Y and open models are now the equivalent of Y and models are being trained and put out to market at a quicker and quicker pace. The greatest model in the world can produce great output as well as it can produce slop under, depending on the overall governance system in place.
	- 'Quality Management' is a term I constantly think of. At one point in my career I was working closely with [ISO 9001](https://www.iso.org/standard/62085.html) to support two major clients. On some days I was agonizing over how intensive it could get; today I think it's one of the most important standards to consider for your 'system' and I've been revisiting it. [ISO 42001](https://www.iso.org/standard/42001) is a more recent, AI-focused, framework but don't forget about ISO 9001!
- **The system must be malleable and tailored:**
	- I'm a huge fan of people like Matt Pocock and Mario Zechner, however, to take what they've contributed publicly without critical thought as to *how* it results in your desired output(s) will inherently introduce operational risk. Whatever you grab off the shelf should be considered a component of components of the overall system so someone who acknowledges it as a building block and then ensures the other blocks are built well will potentially have more success than someone who does not.
	- My system reflects my profile of working expertise, accounts for shortcomings in that expertise given what I set out to achieve, my own personal 'culture', how I like to see productivity reflected, the developer tooling and platforms I use, end user computing, etc. and the output is my own system that works for me. 
	- Also, you likely have a system today whether it is formally defined or not. I started from scratch so my system defaulted to what is described here, many people, organizations, etc. will be faced with how they evolve their current system to incorporate new AI capabilities safely.
- **Failures are valuable inputs, not just incidents:** Part of how you tailor your system is by leveraging your failures, oversights, errors, etc. to sharpen your setup and the great news is that you can accelerate creating your incident template, RCA and actioning triage of the post mortem improvements quite efficiently while still gating it.
- **Where you gate with a human decision is more of a design challenge than a workflow preference:** Another way of presenting the system is in terms of adversarial risk management. When you gate with a human decision it should be where you identify the most risk procedurally based on the current state of your agent governance system.
- As the governance system matures, enforcement gaps start to close, feedback loops add invariants, test coverage expands, and the risk calculation at each gate changes. Gates that exist today because a control is absent may become automateable tomorrow and so on. 

# Designing A System

**Disclaimer** - I'm not going fully into what this looks like, in practice, because it would be an even larger wall of text than this is already. This is also for reference purposes and is not meant to suggest how this gets done, it's simply how my brain thinks about it.
________
The way I tend to think about a System is as a set of governance components, which can be arranged in a multitude of ways, but for the purposes of this exercise I think about them in terms of:
- **Policy**: - what are my consistent requirements?
	- *Enforcement Level*s: Global for company-wide policies, project for team-specific policies where team policies should not conflict with organization policies.
	- *Derived From*: 
		- Finance, Legal, Security, Privacy, Engineering operating requirements.
		- Commercial Arrangements.
		- Market Expectations (e.g. SOC 2, ISO 27001, GDPR, product suite and functionality, etc.)
		- Company objectives.
		- Risk posture.
- **Capabilities** - what can my overall system do and with what?
	- *Enforcement Levels*: Global for capabilities deemed as needing to be in place for alignment to company-wide policies, project in alignment to team-specific policies where team policies should not conflict with organization policies.
	- *Derived From* - Policy.
- **Culture** - how is the system designed to align to company culture?
- **People/Skillsets** - review of skill gaps, domain expertise, team size and risk identification relative to Policy.
- **Tech Stack** - inventory and documentation of product technologies and corporate technologies.
- **Process** - inventory of existing procedures to identify how the current or a potential future state fits in with an updated system.

There is a system in place whether intentional or not so the above is not meant to indicate introduction of a new system, but rather a way to frame the current state such that you can dig into each area and the aggregate to understand how it changes or evolves to adopt new AI-driven capabilities.

## Procedurally...
...this *can* look like the following. Again, this is how my brain works but I tend to try to step back a layer or two to think about how this shares concepts with other concepts and at it's core it's just IT Change Management. It may be way more complex than other IT Change Management projects, but that's because it is way more complex not because it's anything different, conceptually.

## 1 - Document Your System
If you don't have this in place already, document it. Fortunately, you can now dictate to models if that is something you are comfortable with to go through and start laying out the baseline context and source of truth. Document this across all components.

## 2 - Define Risk Cases Against System Documentation
Define your risks against your system documentation and then stack rank them using your organizations risk scoring system. You may need to develop your own to get this done and fortunately there are many good public-facing resources available.

## 3 - Document Your Gaps
Based on those risks identify your operational gaps, this is going to highlight where you will be focusing paired with the stack ranking of the highest-rated risks.

## 4 - Reconcile Gaps, Risks and Policy Requirements
Reconcile what you identified in 2 & 3 against what you are *required* to do. In theory this should have also informed your risk rating exercise in 2. This can be done by gap, by risk, or whatever way makes the most sense and is the most usable by the team involved in this process.

## 5 - Assess Technology Coverage
Given what is reconciled in 4 above apply your technology stack against your list to gauge what is or isn't possible and then layer that into your stack rank from 4 to line up tech stack capabilities for each line item.

## 6 - Design and Planning
Planning is had, but the absence of it presents major operational risk when you land on the other side with outputs lacking forethought, spend some time here thinking through adoption of changes, the inputs you need that should inform those changes and then designing it all to try to play out what it looks like in practice paired with some adversarial pressure testing of does it **actually** work in practice.

## 7 - Prioritize and Execute
Again, nothing new here, this is IT Change Management. Using your operational context of priorities, resources, etc. start to work through the prioritized list of initiatives that come out of steps 1 - 6 to go change your system.

## 8 - Operate and Evolve
This ties directly back to my pattern of using failures as a mechanism to inform how you alter design over time to improve. Additionally, it seems as though there is going to forever be a, "X releases new capability Y that lets you do Z", and with some rapidity, so use those as opportunities to revisit your system.

______________
# Closing
Hopefully this is all interesting to ponder and consider, particularly if it pressure tests how you may think about some of these things, I'm always trying to learn more and pressure test how *I* think about governance, system design, technical implementations, and all of the infinite considerations that I can't accurately reflect in a list. (so much that I use a skill just for this)

Happy AI-ing and I hope you all get some inspiration to be a responsible steward!
