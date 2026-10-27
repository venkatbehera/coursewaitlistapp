# AI-Assisted Engineering Record — CourseWaitlist

## Purpose

This record documents how AI was used as an engineering assistant while implementing the CourseWaitlist assessment. AI suggestions were reviewed against the stated requirements, and verification evidence was collected through automated tests and manual CLI execution.

## Activity 1 — Task Decomposition and Policy Selection

**Prompt used (summary):**

Review the CourseWaitlist assessment and propose a bounded implementation plan, explicit prerequisite and waitlist assumptions, core invariants, and focused tests. Avoid building out-of-scope registration features.

**AI response summary:**

Use a small C# service with an in-memory store. Protect state transitions, define how ineligible waitlisted learners are handled, and test duplicate requests, FIFO promotion, invalid drops, prerequisite failures, and concurrency.

**Decision:**

Accepted the bounded-service approach and documented the prerequisite and waitlist policies in `DESIGN.md`.

**Policy selected:**

Learners must satisfy the prerequisite to enroll or join the waitlist. If a waitlisted learner subsequently becomes ineligible, they are skipped during promotion but remain in the queue. Promotion continues to the earliest eligible learner.

**Verification approach:**

Compared the policy with the implementation and the relevant automated test scenarios.

## Activity 2 — Implementation and Test Review

**Prompt used (summary):**

Generate or review a minimal thread-safe enrollment service with atomic drop-and-promotion behavior, duplicate prevention, deterministic FIFO ordering, and xUnit tests, including concurrent enrollment attempts.

**AI response summary:**

A private lock can serialize in-process state transitions. A monotonically increasing sequence number can preserve deterministic FIFO order. Concurrency tests should verify that capacity is never exceeded and duplicate enrollment records are not created.

**Decision:**

Used these design ideas in `EnrollmentService.cs` and `EnrollmentServiceTests.cs`. Explicit result statuses were used for expected business outcomes, with validation for invalid learner IDs and non-positive capacity.

**Independent review:**

Reviewed the service behavior through automated tests and manual CLI scenarios covering enrollment, prerequisite rejection, waitlisting, duplicate requests, dropping, and promotion.

## Verification Evidence

### 1. Automated Tests

**Environment:** Visual Studio Test Explorer, .NET 8.0 project.

**Observed result:**

- Total tests: 9
- Passed: 9
- Failed: 0
- Skipped: 0
- Reported duration: 126 ms
- Reported warnings: 0
- Reported errors: 0

**Evidence file:** `evidence/automated-test-results.png`

The automated tests cover available capacity, full capacity, duplicate requests, FIFO promotion, prerequisite failures, invalid and repeated drops, invalid capacity, concurrent enrollment attempts, and retaining a waitlisted learner who later becomes ineligible.

### 2. Manual CLI Verification

**Environment:** Windows, .NET 8.0 console application.

**Scenarios exercised:**

- Enrolling eligible learners while capacity is available.
- Rejecting learners who do not meet the prerequisite.
- Adding eligible learners to the waitlist when the course is full.
- Preventing duplicate enrollment and waitlist entries.
- Dropping an enrolled learner and promoting the earliest eligible waitlisted learner.
- Checking enrollment capacity and waitlist state using the `status` command.

**Evidence files:**

- `evidence/cli-execution.png` — manual enrollment and drop workflow.
- `evidence/fifo-promotion.png` — waitlist ordering and FIFO promotion demonstration.

### 3. Verification Limitations

The screenshots establish the observed CLI behavior and the successful nine-test run. They do not independently prove every assertion inside each test. The implementation and test code should remain available for review.

The service uses an in-memory store and an in-process lock. It does not provide persistence across application restarts or coordination across multiple application instances.

## Files Informed

- `src/CourseWaitlist/EnrollmentService.cs`
- `src/CourseWaitlist/Program.cs`
- `tests/CourseWaitlist.Tests/EnrollmentServiceTests.cs`
- `DESIGN.md`

## Final Review Checklist

- [x] Run the automated test suite in Visual Studio.
- [x] Capture the automated test results.
- [x] Run the CLI and exercise the core enrollment and drop workflow.
- [x] Capture manual CLI verification evidence.
- [x] Document the prerequisite and waitlist policies.
- [x] Confirm the final project builds and tests pass using `dotnet test CourseWaitlist.sln --configuration Release`.
- [x] Confirm the evidence files are included in the final submission ZIP.
- [x] Review the final submission against the official assessment rubric.

**Final status:** The Visual Studio test run reports 9 passed, 0 failed, and 0 skipped. Manual CLI evidence has been captured. The final Release-configuration test command and final submission packaging remain to be confirmed.
