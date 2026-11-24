# LLM_USER_TESTING.md — device test rounds

How a device test round runs: the user operates the test devices, the
agents watch the logs, and the round's findings feed the gated protocol
of `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §1 as the next gap report. Read
it before a round starts and when a round's findings are analysed. The
procedure is run by the skill `device-round` (plugin `pipeline`); this
file is its reference. Everything that differs per project — the
devices and their identifiers, the build and install commands, the
capture and extract commands, the round directory, the traces worth
watching, platform steps — is in the project's
`LLM/LLM_PROJECT_USER_TESTING.md`.

A round is one session of the user exercising a flow on the test
devices while the logs of every device are captured in full. Nothing is
diagnosed during the round: the user's questions are collected and
answered in the analysis that follows.

---

## 1. Roles — one session, three models

| Role | Model | Where | Job |
|---|---|---|---|
| Main session | Fable | the conversation | Talks to the user. Relays each test step to the monitor, collects the user's questions, then runs the analysis (Phase 1) and the spec (Phase 2) — the user steers there. |
| Monitor | Sonnet | background subagent | Arms the captures, reads the logs after each step, keeps the timeline file. Never runs jj, never touches code. |
| Implementer | Opus | subagent in its own jj workspace | Implements the approved spec (Phase 3): tests, verify, build, install on every device, report. |

Why split: the monitor's work is matching log lines to steps — cheap
and plentiful; the analysis needs the deep model, because causes can
span the languages and layers of the app, and several devices; the
implementation works from a written spec. The user does not want
answers during the round — a question is a pointer for the analysis,
not a prompt for an immediate guess.

The main session is the only one that talks to the user; subagents
cannot. The model is switched with `/model` and stays for the session;
the subagents get theirs through the Agent tool's `model`.

Per-task jj rules apply unchanged: the implementer works in a
per-task workspace (§2.3 of the ways of working), the monitor and the
main session never run jj while it does.

---

## 2. Before the round

1. **Build state.** The devices carry the build of the last Phase 3
   (the build and install command is in the project's
   `LLM/LLM_PROJECT_USER_TESTING.md`). Note whether the install
   uninstalls first — that wipes the app's data on the device, and the
   user then redoes any setup at the start of the round. Don't
   reinstall for a round when the devices already carry the build.
2. **Devices.** List them with the command the project's file names;
   record each device's identifier in the timeline header. All
   unlocked and connected the way the capture needs, else no log.
3. **Round directory.** `round_<N>/` under the cache directory the
   project's file names — a cache outside the repo survives sessions
   and workspaces; the agent's scratchpad does NOT survive the session,
   so the captures never go there. `<N>` continues the numbering of the
   plan doc's gap reports.
4. **Arm the captures** (the monitor does this, in the background,
   before the user touches a device), one per device, with the
   project's capture commands. A capture can stall silently after
   minutes, and a restart loop inside the capture script is not enough
   on its own: the monitor checks that ALL capture files grow itself
   (`wc -c` on each, every few minutes and before reading a step's
   lines) and re-arms a capture that stopped; the re-arm goes into the
   timeline.
5. **Clock offset.** The monitor notes each device's clock against the
   host's at arming (the first log line): device clocks can differ, by
   hours. Every timeline entry carries every device's log time.
6. **A reboot or reconnect kills the capture.** After one: wait until
   the device is back, re-arm APPENDING to the same file with a marker
   line, and check at every later read that the file still grows and
   the capture process lives; a dead capture is re-armed at once and
   noted in the timeline.

Every helper script runs through `bash` (`bash <script> …`), never
directly: the monitor's shell is zsh, and scripts written for bash
error under it.

---

## 3. During the round

The user reports what they do and see, in their own words. Each message
is one of two kinds:

- **A step.** The main session forwards it to the monitor with the
  host time. The monitor finds the matching lines on every device (the
  project's extract command), appends a timeline entry and answers in a
  sentence or two — what the logs show, not why. The main session
  passes that on briefly.
- **A question** ("why did it show *…*?"). It goes unanswered into the
  timeline's `## Questions` section with the monitor's log excerpt from
  that moment. The user decides in the analysis which ones get dug into
  first.

What the monitor watches for depends on the round; the spec of the
last Phase 3 lists the traces its fixes added (the plan doc's
`Status`). Always worth a line: the traces the project's file lists,
and any error, crash, or user-visible notice.

The round ends when the user says so ("let's evaluate the logs"). The
monitor stops the captures, writes the closing entry and reports the
timeline file's path.

### Timeline file

`round_<N>/timeline.md`, kept by the monitor:

```
# Round <N> — <date>
Build: <stack tip change id>. Devices: A = <id>, B = <id>.
Clocks: host = A + 0:00:12; B = A + 0:00:03.

## Steps
| host | A | B | user said | logs show |
|---|---|---|---|---|
| 18:20:05 | 18:19:53 | 18:19:56 | <what the user said> | <matching lines, one sentence> |
| … |

## Questions
1. 18:31 — "<question>" — A 18:30:50 `<log line>`; B 18:30:53 `<log line>`.

## Observed, not asked
- <anything odd the monitor saw> — harmless?
```

---

## 4. After the round

The phases and their gates are those of
`LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §1.

1. **Analysis — Phase 1** (main session). Input: the timeline, the
   captures, the questions in the user's order. Output: the Gap Report
   in the plan doc, `## Current state (Gap Report, <date> — device
   round <N>)`, one numbered gap per cause (G1, G2, …) with the log
   lines that prove it, what worked, and open `!decide` lines. A gap in
   a rule is checked against the repository's tests and `plan/` docs
   first: if they ask for what was seen, it is a product finding
   (`!decide`), not a bug.
2. **Spec — Phase 2** (main session).
3. **Implementation — Phase 3** (implementer subagent, own workspace).
   Ends with the project's build and install on every device and the
   Phase 3 report. A fix that needs a design change comes back to the
   main session (Phase 2 again).
4. The next round starts at §2 with `<N> + 1`.

Durable findings go where §3 of the ways of working says: a decision
into `plan/design_decisions/`, a device gotcha into the project's
`README.md` or its `LLM/LLM_PROJECT_USER_TESTING.md`, a process lesson
that holds for every project here.
