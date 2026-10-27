# CourseWaitlist — Design Document

## 1. Overview

CourseWaitlist is a .NET 8 application that manages enrollment for a single course offering. It supports learner enrollment, waitlisting when capacity is full, dropping an enrollment, and promoting the next eligible learner from the waitlist.

The application uses an in-memory data store and a console interface for demonstration. The core enrollment logic is isolated in `EnrollmentService`, allowing it to be tested independently of the console application.

## 2. Scope and Boundaries

### In Scope

- Enroll a learner when course capacity is available.
- Add eligible learners to a waitlist when the course is full.
- Prevent duplicate enrollment and waitlist entries.
- Validate learner IDs and prerequisite eligibility.
- Drop an existing enrollment.
- Promote the next eligible waitlisted learner after a successful drop.
- Preserve FIFO ordering among waitlisted learners.
- Protect shared state against concurrent operations.
- Return explicit statuses for successful and unsuccessful operations.

### Out of Scope

- Multiple course offerings.
- Persistent database storage.
- User authentication and authorization.
- Web APIs or a graphical user interface.
- Distributed coordination across application instances.
- Notifications, cancellation policies, and administrative reporting.

## 3. Architecture

The application follows a simple service-oriented structure.

```text
Console Application (Program.cs)
             |
             v
    EnrollmentService
             |
             v
       In-Memory State
       - Enrolled learners
       - Waitlisted learners
       - FIFO sequence numbers

Automated Tests
      |
      v
EnrollmentService
```

### Components

**Program.cs**
- Provides a simple command-line interface.
- Accepts `enroll`, `drop`, `status`, and `quit` commands.
- Displays operation results and the current course snapshot.
- Supplies a demonstration prerequisite eligibility predicate.

**EnrollmentService.cs**
- Contains the core enrollment and waitlist business logic.
- Validates inputs and prevents duplicate requests.
- Manages enrollment capacity and waitlist ordering.
- Promotes eligible learners after an enrollment is dropped.
- Synchronizes access to shared state.

**EnrollmentServiceTests.cs**
- Tests business rules, edge cases, FIFO promotion, prerequisite handling, and concurrent operations.

## 4. Data Model

The service maintains the following state:

| Field | Purpose |
|---|---|
| `_capacity` | Maximum number of enrolled learners; fixed at construction. |
| `_enrolled` | A `HashSet<string>` containing currently enrolled learner IDs. |
| `_waitlisted` | A dictionary mapping each waitlisted learner ID to a monotonically increasing sequence number. |
| `_nextSequence` | Generates sequence numbers to preserve waitlist order. |
| `_meetsPrerequisite` | A predicate used to determine whether a learner satisfies the course prerequisites. |
| `_gate` | A private lock used to synchronize state access. |

The constructor rejects a non-positive capacity and a null prerequisite predicate.

Learner IDs are trimmed before processing and compared using ordinal, case-sensitive comparison within the service. The console demonstration's prerequisite lookup uses a case-insensitive set of eligible IDs.

## 5. Core Operations

### 5.1 Enroll

When `Enroll(learnerId)` is called:

1. Reject a null, empty, or whitespace-only ID with `InvalidLearnerId`.
2. Trim the learner ID.
3. Acquire the service lock.
4. If the learner is already enrolled, return `AlreadyEnrolled`.
5. If the learner is already waitlisted, return `AlreadyWaitlisted`.
6. If the learner does not satisfy the prerequisite, return `PrerequisiteNotMet`.
7. If capacity is available, add the learner to `_enrolled` and return `Enrolled`.
8. Otherwise, assign the next sequence number, add the learner to `_waitlisted`, and return `Waitlisted`.

The lock ensures that concurrent enrollment requests cannot exceed course capacity or create duplicate entries.

### 5.2 Drop

When `Drop(learnerId)` is called:

1. Reject a null, empty, or whitespace-only ID with `InvalidLearnerId`.
2. Trim the learner ID.
3. Acquire the service lock.
4. If the learner is not enrolled, return `NotEnrolled` without changing state.
5. Remove the learner from `_enrolled`.
6. Attempt to promote the next eligible learner from the waitlist.
7. Return `Dropped`, including the promoted learner ID when promotion succeeds.

Dropping an enrollment and promoting a replacement happen under the same lock, preventing another operation from observing or creating an inconsistent intermediate state.

### 5.3 FIFO Promotion

The service examines waitlisted learners in ascending sequence-number order.

- The first eligible learner is removed from the waitlist and added to the enrolled set.
- Learners who are currently ineligible are skipped but remain on the waitlist.
- The search continues until an eligible learner is found or the queue is exhausted.
- A learner who later becomes eligible can be considered during a subsequent promotion.
- A defensive check avoids enrolling a learner who is already enrolled.

This policy preserves the relative order of waitlisted learners while preventing a currently ineligible learner from being promoted.

### 5.4 Get Snapshot

`GetSnapshot()` returns a consistent view of the current state:

- Configured capacity.
- Enrolled learner IDs.
- Waitlisted learner IDs in FIFO order.

The method uses the service lock and returns collections representing a snapshot rather than exposing the internal mutable collections.

## 6. Business Rules and Assumptions

1. The service manages one course offering.
2. Course capacity is fixed when the service is constructed.
3. A learner must satisfy the prerequisite to enroll or join the waitlist.
4. Duplicate enrollment and duplicate waitlist requests are rejected.
5. Learner IDs are trimmed and compared case-sensitively by the service.
6. Only a currently enrolled learner can be dropped.
7. A failed drop does not modify enrollment or waitlist state.
8. A successful drop triggers an immediate attempt to promote the next eligible waitlisted learner.
9. A waitlisted learner who is currently ineligible remains in the queue and is skipped during promotion.
10. The prerequisite predicate is evaluated while the service lock is held. The predicate is expected to be a short, non-blocking operation.
11. All state is held in process memory and is lost when the application exits.

## 7. Concurrency and Consistency

All operations that read or modify shared enrollment state use the same private lock.

This protects the following invariants:

- The number of enrolled learners never exceeds capacity.
- A learner cannot be both enrolled and waitlisted.
- A learner cannot appear more than once in either collection.
- Waitlist order remains deterministic.
- A successful drop and its associated promotion are performed as one synchronized state transition.
- Snapshot reads do not observe partially completed state changes.

The automated test suite includes a concurrent enrollment scenario to validate capacity and duplicate-entry protections under parallel requests.

The lock provides **in-process thread safety only**. It does not coordinate multiple application instances or provide database-level transaction guarantees.

## 8. Result Handling

The service returns an `EnrollmentResult` containing a status, a human-readable message, and an optional promoted learner ID.

Statuses include:

- `Enrolled`
- `Waitlisted`
- `AlreadyEnrolled`
- `AlreadyWaitlisted`
- `PrerequisiteNotMet`
- `InvalidLearnerId`
- `Dropped`
- `NotEnrolled`

Explicit result statuses allow callers and tests to distinguish business outcomes without relying on message text.

`CourseSnapshot` represents the course's current capacity, enrolled learners, and waitlisted learners.

## 9. Complexity

Let *n* be the number of waitlisted learners.

- Enrollment membership checks and updates are expected to be O(1) on average.
- Dropping an enrolled learner is expected to be O(1) on average, excluding waitlist promotion.
- Waitlist promotion is O(n log n) in the current implementation because waitlisted entries are ordered by sequence number before examination.
- Snapshot creation includes sorting enrolled learners and ordering waitlisted learners, so it requires additional work proportional to the collection sizes.

The implementation prioritizes simplicity and predictable behavior for a small, in-memory course offering.

## 10. Testing Strategy

The xUnit test suite covers:

- Enrollment when capacity is available.
- Waitlisting when capacity is full.
- Duplicate enrollment and waitlist requests.
- FIFO promotion after a learner drops.
- Prerequisite failures.
- Invalid learner IDs and repeated or invalid drops.
- Concurrent enrollment requests and capacity enforcement.
- Rejection of invalid course capacity.
- Skipping a currently ineligible waitlisted learner while retaining that learner in the queue.

Automated tests validate the service directly without depending on console input or output.

## 11. Trade-offs and Future Improvements

### Current Trade-offs

- **In-memory storage:** Simple to run and test, but state does not survive application restarts.
- **Single lock:** Provides straightforward consistency, but serializes concurrent service operations.
- **Prerequisite evaluation under lock:** Keeps the state transition synchronized; a slow predicate can reduce throughput.
- **Ordered dictionary values:** Keep FIFO ordering simple, but promotion requires sorting the waitlist.

### Potential Improvements

If the application needed to support production workloads, possible improvements would include:

- Persisting enrollment and waitlist state in a database.
- Using database transactions, unique constraints, and appropriate concurrency controls.
- Supporting multiple course offerings.
- Introducing cancellation, auditing, and notification workflows.
- Adding API endpoints and authentication.
- Using a queue data structure or indexed priority structure if waitlist volume justified it.
- Adding structured logging and operational monitoring.

## 12. Summary

CourseWaitlist separates its business logic from its console interface, uses explicit operation results, preserves FIFO waitlist ordering, and protects shared state with synchronization. Its design is intentionally small and self-contained, making the enrollment rules straightforward to understand and verify with automated tests.
