# angry-tester: hostile QA + security tester for any codebase

`angry-tester` is a [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skill. It assumes every line of your code is broken and tries to prove that the app can be:

| Impact | Meaning | Examples |
|---|---|---|
| `HACK` | security breach or data leak | SQL injection, IDOR, leaked secrets, vulnerable dependencies |
| `DOWN` | crash, outage or resource exhaustion | huge payloads, ReDoS, missing timeouts, no rate limiting, race conditions |
| `CRACK` | auth bypass or business-logic abuse | negative prices, coupon reuse, trusting client-sent role or price |
| `BUG` | functional defect | wrong result on a positive, negative or boundary case |

It writes **real tests** in your project's own test framework, runs them, and turns the results into two reports:

| Format | File |
|---|---|
| Markdown | `angry-tester-report/ANGRY_TESTER_REPORT.md` |
| Excel | `angry-tester-report/angry_tester_report.xlsx` |

It works with any stack. It detects the language, framework and test tool itself (Java/Spring, Node/TypeScript, Python, Go, .NET, Ruby, PHP and others).

## Quick start

1. Install the skill (needs Claude Code, Git and Python 3.9+; see [Install](#install)):
   ```bash
   git clone https://github.com/vipulraut-coditas/angry-tester-agent.git
   cp -r angry-tester-agent/skill ~/.claude/skills/angry-tester
   pip install openpyxl
   ```
2. Open any repo in Claude Code and type:
   ```
   /angry-tester
   ```
3. Answer the scope questions: complete application, one feature, or recent changes, and which focus areas.
4. Open the reports in `<repo>/angry-tester-report/`.

A session looks like this:

```
> /angry-tester
? What should I test?
  > Complete application
    Specific feature/module
    Recent changes only
? Which focus areas? [x] Security (Hack) [x] Availability (Down) [x] Logic abuse (Crack) [x] Functional
Scope: Feature: Login (src/auth) | Focus: Security, Availability, Logic, Functional | Repo: C:\work\my-app
... recon, attack plan, writing tests, running tests ...
Verdict: The login flow can be bypassed with a single SQL injection payload and has no brute-force protection.
Findings: Critical 1, High 2, Medium 3, Low 1   Tests: PASS 18, FAIL 6, NOT RUN 1
Markdown: C:\work\my-app\angry-tester-report\ANGRY_TESTER_REPORT_login.md
Excel:    C:\work\my-app\angry-tester-report\angry_tester_report_login.xlsx
```

## Install

The skill is a folder of files. Claude Code loads skills from `~/.claude/skills/` (all repos) or `<repo>/.claude/skills/` (one repo).

**For every repo (user level):**

```bash
rm -rf ~/.claude/skills/angry-tester
cp -r angry-tester-agent/skill ~/.claude/skills/angry-tester
```

On Windows PowerShell:

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\angry-tester" -ErrorAction SilentlyContinue
Copy-Item -Recurse angry-tester-agent\skill "$env:USERPROFILE\.claude\skills\angry-tester"
```

**For one repo (shared with your team through Git):**

```bash
mkdir -p .claude/skills
cp -r path/to/angry-tester-agent/skill .claude/skills/angry-tester
git add .claude/skills/angry-tester
```

**Python packages for the report builder:**

```bash
pip install openpyxl            # required, builds the Excel file
pip install "jsonschema>=4"     # optional, full schema validation of the findings
```

**Updating:** after you change this repo (`git pull` or your own edits), run the copy command again. The installed copy does not update by itself.

Check it is installed: start Claude Code and type `/` and look for `angry-tester`, or ask *"what skills do you have?"*.

## Usage

**Slash command:**

```
/angry-tester
```

**Plain language.** Claude Code picks up the skill from phrases like:

- *"run angry tester on this repo"*
- *"attack my code"*, *"try to break this"*
- *"security test this repo"*
- *"test the login feature for vulnerabilities"*
- *"angry test the changes in feature/payments against main"*

If your message already says the scope or focus (for example *"test the payments feature for security only"*), it skips those questions.

**Scopes:**

| Scope | What it tests |
|---|---|
| Complete application | every entry point: routes, consumers, cron jobs, CLI, uploads, webhooks, websockets |
| Specific feature/module | the feature's full code path: route, controller, service, DB queries, validators, auth middleware, config. Shared issues that affect it are tagged `Outside scope - affects feature` |
| Recent changes only | the code changed in a branch, PR or commit range (`git diff`) |

**Focus areas:** Security (Hack), Availability (Down), Logic abuse (Crack), Functional (positive, negative, boundary). All are selected by default.

## Output

Everything is written inside the repo you test:

| Path | What |
|---|---|
| `angry-tester/tests/` (or `angry_*` files in your test folder) | the new attack tests; review, keep or delete them |
| `angry-tester/findings.json` | raw findings, attack surface, trade-offs, gaps |
| `angry-tester/test_results.json` | every test with input, expected, actual and PASS / FAIL / NOT RUN |
| `angry-tester-report/ANGRY_TESTER_REPORT.md` | the readable report |
| `angry-tester-report/angry_tester_report.xlsx` | the same data as a sortable, colour-coded workbook |

Feature scope adds the feature name to the report file names (`ANGRY_TESTER_REPORT_login.md`); diff scope adds `_diff_<range>`.

**Markdown report sections:** Scope, Verdict, Scorecard (severity x impact, test totals), Top 5 ways it gets hacked / goes down / gets cracked, All findings, Finding details (scenario, PoC input, evidence, business impact, fix), Test scenarios, Trade-offs, Attack surface, Gaps.

**Excel sheets:**

| Sheet | Columns |
|---|---|
| Summary | project, date, scope, focus, stack, verdict, counts by severity, impact and test result |
| Findings | ID, Title, Impact, Severity, Confidence, Scope, Category, File, Line, Scenario, PoC Input, Evidence, Business Impact, Fix, Status |
| Test Cases | Test ID, Category, Endpoint/Function, Scenario, Input, Expected, Actual, Result, Linked Finding |
| Attack Surface | Entry Point, Method, Auth, Roles, Inputs, Touches, Risk Notes |
| Trade-offs | Area, Option A, Option B, Risk, Recommendation |

Severity colours: Critical dark red, High red, Medium orange, Low yellow. Result colours: PASS green, FAIL red, NOT RUN grey.

Tip: add `angry-tester/` and `angry-tester-report/` to `.gitignore` if you don't want them committed.

You can also rebuild the reports by hand from the JSON files:

```bash
python ~/.claude/skills/angry-tester/scripts/build_report.py \
  --findings angry-tester/findings.json \
  --tests angry-tester/test_results.json \
  --out angry-tester-report
```

## Safety rules

The skill is told to:

- test only code you own, and confirm the repo path first
- run everything locally or in a sandbox; never send attack traffic to production, staging or third-party hosts unless you name an environment you own
- never touch real or shared databases (in-memory DBs, mocks or local containers only)
- never modify your application source; fixes are suggested in the report only
- use minimal, harmless proof-of-concept inputs, and bound load tests (payload size, request count, timeouts)
- mask any secrets it finds (`AKIA****XYZ`)

## Troubleshooting

| Problem | Fix |
|---|---|
| `/angry-tester` is not listed | The folder must be `~/.claude/skills/angry-tester/SKILL.md` (not `.../angry-tester/skill/SKILL.md`). Re-run the copy command and restart Claude Code. |
| `openpyxl is required` | `pip install openpyxl`, then ask Claude to rebuild the report. |
| `Schema validation failed` | Claude wrote JSON that doesn't match the schema. Ask it to fix `findings.json` / `test_results.json` and rerun the builder. |
| Many tests are `NOT RUN` | The app needs a service the tests couldn't mock (DB, queue, external API). Check the Gaps section for the reason, start the dependency locally or tell Claude how to mock it, then rerun. |
| `python` not found on Windows | Use `py` instead of `python`. |

## How it works

```
Phase 0 Scope ─► Phase 1 Recon ─► Phase 2 Attack plan ─► Phase 3 Write & run tests ─► Phase 4 Judge ─► Phase 5 Reports
(ask: app /      (stack, entry     (HACK, DOWN, CRACK,     (project's own framework,    (impact, severity,  (build_report.py:
 feature / diff,  points, config,   functional, trade-offs) failing test = confirmed)    confidence, fix)    .md + .xlsx)
 focus)           secrets)
```

1. **Scope** asks what to test and prints one confirmation line.
2. **Recon** detects the stack, lists every in-scope entry point and finds config and secrets. This becomes the attack surface table.
3. **Attack plan** walks an OWASP Top 10 + API Top 10 checklist plus availability, logic-abuse and functional cases. Every candidate must point to a file and line.
4. **Tests** assert the *safe* behaviour, so a test fails when the attack works. The failure is the evidence.
5. **Judge** gives every finding an impact, severity (with a CVSS-style reason), confidence (`Confirmed`, `Likely`, `Theoretical`) and a concrete fix, and writes `findings.json` and `test_results.json`.
6. **Reports** are built by `scripts/build_report.py`, which validates both files against `schemas/findings.schema.json` and writes the Markdown and Excel reports.

## Project layout

```
skill/                        copy this folder to ~/.claude/skills/angry-tester
  SKILL.md                    the agent's instructions (phases, checklist, rules)
  scripts/build_report.py     JSON -> Markdown + Excel report builder
  schemas/findings.schema.json  schema for findings.json and test_results.json
README.md
LICENSE
```

## License

MIT
