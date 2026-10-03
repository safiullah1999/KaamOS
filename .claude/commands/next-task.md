Read `IMPLEMENTATION_PLAN.md` and find the first unchecked `- [ ]` task in the current phase
(the first phase in `CLAUDE.md` §9 that is not complete).

Read the relevant subsystem doc in `docs/` for that task. Then, **in plan mode**, propose:
- the files to create or change,
- the tests to write (including an RLS isolation test if the task touches tenant data),
- how we will verify it (which `make` target, what the eval gate should show).

**Wait for my approval before editing any files.** Work on this ONE task only — do not start the next.

When I approve and the work is done:
- run `make test` and `make lint` and paste the output,
- tick the task's checkbox in `IMPLEMENTATION_PLAN.md`,
- if this completes a phase, update the status table in `CLAUDE.md` §9,
- propose a Conventional Commit message.

If anything in the docs is ambiguous or contradictory, stop and ask — do not guess.
