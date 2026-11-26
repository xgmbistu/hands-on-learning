# Test environment deployment

## Prerequisites and authorization

Environment identifier/owner, access, required services/data, versions, deployment scope, and authorization boundary.

## Build, synchronize, and deploy

Actual project commands and artifact paths. Preserve a defect baseline before updates; state effective-version verification, readiness checks, and expected results.

## Failure and rollback

Failure diagnostics, rollback version/commands/triggers, and data cleanup. Record environment retention/release ownership and actual results.

## Validation evidence

Command, code version, run/request identifiers, output, and conclusion. If no independent test deployment exists, explain the actual validation/release flow with implementation evidence.


During initialization, populate this execution guidance from actual project files. Record sources, working directory, commands, prerequisites, result criteria and cleanup/rollback. In `.wally/config.json`, set optional `readiness: pending` and concrete `gaps` when required information is missing; ask the user and wait for the required information. Mark the phase BLOCKED, not automatically skipped; a guide path alone must not activate it. Set ready only after static verification. This is not evidence of execution. Existing configs without readiness retain legacy command/guide semantics.

## Static verification (readiness: pending; not executed)

Source inspection found no real deployment setup: no Makefile, Dockerfile/Compose file, CI workflow, package manifest, or deploy script; README.md has no body and nosql.py (a local socket server on localhost:50505) is the only source file. No deploy command, target test environment, credential source, verification step, or rollback procedure is defined, so no command is guessed. Concrete gaps are recorded in deploy.gaps in .wally/config.json. Provide the real deploy command, target environment, credential acquisition method, and verification/rollback criteria before activating this phase. This is a static check only; nothing was built, deployed, or executed.
