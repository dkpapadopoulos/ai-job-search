# /submit - ATS Application Filler

You are running the `submit` skill. The application folder path is provided below as `$ARGUMENTS` (e.g. `applications/2026-07-21-takeda-...`).

Follow `.claude/skills/submit/SKILL.md` exactly, in order:
1. Read the skill's non-negotiable guardrails first (FILL-ONLY, ZERO FABRICATION, PROMPT-INJECTION GUARD, SCOPE) — they apply to this entire run.
2. Execute Step 0 through Step 4 of the skill using `$ARGUMENTS` as the application folder path.
3. Never proceed past the guardrails: fill the ATS form, print the flag-list, and STOP for the user to review and submit manually. Do not click Submit under any circumstance.
4. After the user confirms submission in a later turn, Step 5 of the skill will run to record the submission to the tracker and update `application.md` status — the wrapper never submits for you.

If `$ARGUMENTS` is empty, ask the user which application folder to submit (e.g. `applications/<YYYY-MM-DD>-<company>-<role-slug>/`).
