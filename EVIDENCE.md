# FestPass Jenkins challenge: evidence page

**Student (Git author alias):** Parth Puri
**Class code:** 001 · **Challenge code:** 00262
**Jenkins job:** `festpass-ci` (Pipeline script from SCM)

## Job configuration

| Setting | Value |
|---|---|
| Repository URL | `git://127.0.0.1:9418/festpass-java` (local read-only Git daemon) |
| Credentials | none |
| Branch specifier | `*/main` |
| Script path | `Jenkinsfile` |
| Agent | `student-laptop` (separate inbound WebSocket agent, 1 executor; built-in node has 0 executors) |
| Trigger | `triggers { pollSCM('* * * * *') }` committed in the `Jenkinsfile` (commit `404bcd3`); no UI trigger, no "Build periodically" |

## Build and commit trace

| Build | Cause | Commit | Result | Tests | Product stages | JAR |
|---|---|---|---|---|---|---|
| #1 | Started by user (bootstrap "Build Now") | `404bcd3` Configure my Jenkins assignment | FAILURE | 28 run, 3 failed | Build JAR, Smoke test JAR, Save artifact **skipped** | not published |
| #3 | **Started by an SCM change** | `1bb4cd3` Personalize my help introduction | FAILURE | 28 run, 3 failed | Build JAR, Smoke test JAR, Save artifact **skipped** | not published |
| #4 | **Started by an SCM change** | `6bfe104` Fix cancel: release the student's seat when a booking is cancelled | **SUCCESS** | **28 run, 0 failed** | all stages ran | `festpass.jar` published |

Build #2 was an accidental second click on Build Now and rebuilt `404bcd3` (same red result). It is not part of the trace.

In each automatic build, the commit Jenkins checked out matches `git log -1 --oneline` in the project.
See `Checking out Revision ...` in [`evidence/build-3-red-automatic/console.txt`](evidence/build-3-red-automatic/console.txt) and [`evidence/build-4-green-automatic/console.txt`](evidence/build-4-green-automatic/console.txt).

## Diagnosis: one failing assertion

From the red build's Test Result:

```
FAIL cancellationsSurviveReload: Cancellation was lost
java.lang.AssertionError: Cancellation was lost
```

The test books student `S1` for event `java`, saves, reloads, cancels `S1`, saves again, and expects the reloaded roster to be **empty**.
The roster still contained `S1`. Two other tests fail for the same reason:

- `cancellationFreesSeat`: the event has 1 seat. After `S1` books and cancels, `S2` should get the seat, but the app said *SOLD OUT*.
- `cliWorkflowPersistsAcrossCommands`: `list` did not reflect a persisted cancellation.

**The broken business rule:** cancelling an existing booking must remove the student from the event and free the seat.
`BookingService.cancel()` checked that the booking existed but never removed it.
The command reported success, but the seat stayed taken: nobody else could book it, and the saved file still showed the old booking.

## Repair (minimal fix, tests unchanged)

One line added to `src/main/java/BookingService.java` (commit `6bfe104`):

```diff
         if (!event.students.contains(studentId)) {
             throw new IllegalArgumentException("No booking for student: " + studentId);
         }
+        event.students.remove(studentId);
     }
```

No tests, build helpers, evidence code, or pipeline stages were changed. The only `Jenkinsfile` change is the `pollSCM` trigger.

## Green build and artifacts

- Build #4: **SUCCESS, 28 tests, 0 failures**; smoke test: `PASS: seven real-JAR commands; persistence, sold-out rejection, cancellation, and seat reuse verified.`
- Red builds #1 and #3 skipped Build JAR / Smoke test JAR / Save artifact and published **no JAR** (`JarSha256 = NOT_PUBLISHED`), but did archive `submission-evidence.csv`.
- Final JAR: [`evidence/build-4-green-automatic/festpass.jar`](evidence/build-4-green-automatic/festpass.jar), SHA-256 `c39d9527a332871821f5843c0faf606ee2af54cd4f07a532299ed67ac13e930a` (matches `JarSha256` in the green evidence CSV).

Output of `java -jar festpass.jar help` in the download folder:

```
FestPass - campus event registration, maintained by Parth Puri
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
  add-event ID "TITLE" CAPACITY
  list
  book EVENT_ID STUDENT_ID
  cancel EVENT_ID STUDENT_ID
  roster EVENT_ID
  summary
  help
Default data file: data/festpass.tsv
IDs are case-sensitive. Use synthetic student IDs, not personal data.
Exit codes: 0 success, 2 invalid request, 3 storage failure.
```

## Evidence files

| File | Build |
|---|---|
| [`evidence/build-3-red-automatic/submission-evidence.csv`](evidence/build-3-red-automatic/submission-evidence.csv) | red, automatic (#3) |
| [`evidence/build-4-green-automatic/submission-evidence.csv`](evidence/build-4-green-automatic/submission-evidence.csv) | green, automatic (#4) |
| [`evidence/build-1-red-bootstrap/submission-evidence.csv`](evidence/build-1-red-bootstrap/submission-evidence.csv) | red, manual bootstrap (#1) |
| `evidence/*/console.txt` | Jenkins console output for #3 and #4 |
| `evidence/build-4-green-automatic/build-info.txt`, `smoke-test.txt` | archived with the green build |

This `EVIDENCE.md` and the `evidence/` folder were added in a separate commit after build #4, so Jenkins did not build them. Commits `c88a14d` to `6bfe104` are exactly the history Jenkins built.
