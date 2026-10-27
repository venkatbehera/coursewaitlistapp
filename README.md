# CourseWaitlist

A focused C#/.NET implementation of course enrollment, dropping, prerequisite validation, and FIFO waitlist promotion.

## Features

- Enroll eligible learners when seats are available.
- Add eligible learners to the waitlist when the course is full.
- Prevent duplicate enrollment and waitlist entries.
- Drop enrolled learners and promote the earliest eligible waitlisted learner.
- Skip ineligible waitlisted learners during promotion while retaining their queue entries.
- Protect state transitions using a lock.
- Validate learner IDs and course capacity.
- Include automated tests for business rules and concurrent enrollment attempts.

## Technology Stack

- C#
- .NET 8
- xUnit
- In-memory storage

## Project Structure

```text
CourseWaitlist/
├── src/
│   └── CourseWaitlist/
├── tests/
│   └── CourseWaitlist.Tests/
├── evidence/
│   ├── cli-execution.png
│   ├── automated-test-results.png
│   └── fifo-promotion.png
├── CourseWaitlist.sln
├── DESIGN.md
├── AI-USE-RECORD.md
├── RUBRIC-ASSESSMENT.md
└── README.md
```

## Prerequisite and Waitlist Policy

Learners must satisfy the prerequisite to enroll or join the waitlist. If a learner becomes ineligible while waiting, they are skipped during promotion but remain on the waitlist.

Eligible learners are promoted in FIFO order. Enrollment, dropping, and promotion are protected by an in-process lock to preserve capacity and state consistency.

## Prerequisites

Install the .NET 8 SDK.

## Build and Run

From the repository root:

```bash
dotnet restore CourseWaitlist.sln
dotnet build CourseWaitlist.sln --configuration Release
dotnet run --project src/CourseWaitlist/CourseWaitlist.csproj
```

## Run Automated Tests

```bash
dotnet test CourseWaitlist.sln --configuration Release
```

Review the test output for the actual pass/fail summary.

## CLI Commands

```text
enroll <learnerId>
drop <learnerId>
status
quit
```

## Design and Verification Documentation

- `DESIGN.md` — service boundary, invariants, waitlist policy, and concurrency approach.
- `AI-USE-RECORD.md` — AI-assisted engineering activities and verification evidence.
- `RUBRIC-ASSESSMENT.md` — assessment against the supplied rubric.
- `evidence/` — screenshots of CLI behavior and automated test results.

## Limitations

This implementation supports one course offering, uses in-memory storage, and coordinates concurrency only within a single application process. State is lost when the application stops. Persistent storage and multi-instance coordination are outside the scope of this assessment.
