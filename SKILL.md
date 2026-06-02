---
name: test-skill
description: "Generic test automation framework for AOS components on KVM simulator. Handles simulator startup, DUT boot waiting, SSH-based config deployment, snapshot extraction, PlantUML diagrams, and archive packing. Component-specific test logic is in specialized skills (e.g. /test-service-manager)."
---

# Test Skill — Generic AOS Test Automation Framework

## Commands

| Command | Description | Hint |
|---------|--------|------|
| `help` | Display this commands table | See how to use the skill |
| `init` | Create new project (project_config.json + folder structure) | Use when starting a new test case |
| `init --from-spec <spec>` | Create project from openspec file | Auto-generate config/ from spec |
| `validate` | Check config/ folder is complete before running | Use before `start` or `run` |
| `start` | Start simulator + run test + diagrams + pack | Full workflow: sim → run → diagrams → pack → stop |
| `start --endless` | Same as `start` but does NOT auto exit when done | Sim runs continuously, waits for user to call `stop` |
| `start --dryrun` | Same as `start` but SKIP pack (trial run) | Quick run without archiving |
| `run` | Run test (simulator already running, chassis available) | Use when sim is running, only need to deploy config |
| `diagrams` | Generate topo.puml + test[n].puml from config/output | Create PlantUML diagrams |
| `pack` | Package output + config into tar.gz | Bundle test results |
| `stop` | Exit simulator properly (type `exit` in sim shell) | MUST use after testing is done |
| `status` | Check if simulator/DUT is running | Ping DUT via aluminium |
| `logs` | View output logs (sim.log, dut*_svcmgr.log) | Check test results |
| `clean` | Delete entire output/ folder (logs + snapshots) | Use before re-running test |

## Purpose

Generic framework to automate AOS component testing across multiple DUTs using KVM simulator. Handles simulator startup, DUT boot waiting, SSH-based config deployment, and config snapshot extraction.

**Component-specific test logic** (traffic tests, binary deploy, report templates) lives in specialized skills:

| Specialized Skill | Component |
|-------------------|-----------|
| `/test-service-manager` | Service Manager (svcCmm/svcNi) — L2GRE, EVPN, UNP, SNMP |
| *(future)* | Other AOS components |

---

## ⚠️ Test Method Validation — MANDATORY Before Every Test (Sim + HW)

**Rule: MUST validate test method and confirm with user BEFORE running test.**

Applies to ALL test types: simulator, real hardware, VC, standalone.

### Step 1: Trace Call Chain (perform in `/service-manager-code`)
- Trace from target function → caller → trigger condition
- Determine EXACTLY which event/condition triggers the function to test
- **Tracing details** → see `/service-manager-code` → "Call Chain Tracing for Test Design"

### Step 2: Confirm Test Plan with User (use vscode_askQuestions)
Before configuring anything on DUT/sim, ask user:
- Function to test: `X()`
- Trigger condition identified (from step 1): `Y`
- Proposed test method: `Z`
- Current DUT: standalone / VC / sim
- "Does this test method cover the correct code path?"

### Step 3: If DUT/Sim Does Not Meet Requirements → REPORT IMMEDIATELY
- Example: function requires VC-takeover but DUT is standalone → report to user immediately
- DO NOT improvise a substitute test method then claim coverage
- Only run regression test (boot, init, basic path) and clearly note the limitation

### Lesson Learned (CRAOS8X-52352)
- `HandleAggEvntForNw()` (network) only triggers on VC-takeover
- `HandleAggEvntForSap()` (access) triggers on linkagg toggle
- Agent incorrectly assumed linkagg toggle = can test `ForNw()` without tracing call chain

---

## Cross-References

| Skill | When to reference |
|-------|---------------------|
| `/perforce-workflow` | Environment setup, P4 password handling |
| `/aos-build-optimize` | Build binary before deploying |
| `/simulator-operations` | DUT su access, binary deploy, SNMP, EVPN config |
| **`/test-service-manager-execution`** | **Execution Model (Terminal A/B/C), Phase 1-6 process details — ALWAYS read when implementing framework** |

## Error Handling / Fallback

| Failure | Fallback |
|---------|----------|
| SSH: "Host key verification failed" | `ssh-keygen -R <DUT_IP>` on aluminium BEFORE running framework |
| Simulator xterm error ("Can't open display") | STOP, kill sim, restart with `-x none` — DO NOT workaround with `DISPLAY=` |
| DUT does not boot within timeout | `sim.py --clean <N>` → restart sim → check SCF file is valid |
| Phase 2 (deploy) fail: command rejected | STOP — DO NOT continue to Phase 3. Fix config, restart from Phase 1 |
| Phase 3 (verify) fail: snapshot mismatch | Read swlog debug3 → find error lines → report root cause, DO NOT self-fix |
| Phase 4 (test) fail | Check component-specific skill for troubleshooting |
| Build binary fail before deploy | See `/aos-build-optimize` → `## Common Build Errors & Fixes` |
| P4 password prompt during test | Ask user via `vscode_askQuestions` — DO NOT enter it yourself |
| Expect script gets cancelled | VS Code detects "password" in output → redirect: `expect script.exp > /tmp/out.txt 2>&1` |

**Rule:** When any Phase fails → **STOP and report** to user. DO NOT auto-retry or skip phase.

**Rule:** ANY expect script with `assword:` prompt → ALWAYS redirect output to file (`> /tmp/output.txt 2>&1`). VS Code autopilot auto-cancels when it sees password prompt on terminal.

---

## OpenSpec (Test Specifications)

**Location:** `/home/tvu47178/.frameworks/test_service_manager/openspec/`

OpenSpec separates **WHAT to test** (spec) from **HOW to execute** (framework).

| Layer | Responsibility | File |
|-------|-------------|------|
| **Spec** | Config commands, expected output, pass/fail criteria | `openspec/specs/<category>/<name>.spec.md` |
| **Executor** | Sim start, SSH deploy, capture, compare, report | Framework scripts |

### Spec Format (YAML frontmatter + Markdown)
```yaml
---
spec_version: "1.0"
name: "Feature Scenario"
platform: ["OS6870"]
base_label: "OS_7_X_X_9918_R01"
duts: 1
---
```
Sections: Objective, Prerequisites, Sync Command, Config Commands, Expected Output, Pass Criteria, Fail Indicators, swlog Verification, Notes.

### Workflow: `init --from-spec`
1. Read spec file → parse config commands, DUT count, platform
2. Create project folder + `config/project_config.json`
3. Generate `config/dut[n].cmds` from spec's Config Commands section
4. Ready to `start`

---

## Core Components

### Framework Scripts
- run.sh — Main orchestration script (Phase 1 + Phase 2 + Phase 3 + Phase 4)
- ssh_ready.sh — DUT boot wait + SSH ready check
- configure.sh — Config deployment (sequential, all DUTs)
- configure_single.sh — Config deployment (single DUT — for parallel execution)
- verify.sh — Verify operational state (sequential, all DUTs)
- test.sh — Run test scripts from project_config.json (optional)
- ssh_helper.py — SSH/pexpect utilities
- Phase 5 (Archive) is handled by the skill directly (not a shell script)

### Templates
- templates/topo.puml.template — PlantUML topology diagram template
- templates/test.puml.template — PlantUML per-test sequence diagram template

### Supporting Files
- project_config.template.json — Template for project initialization
- environment — Environment setup commands
- sim — Simulator guide + guidelines

## Framework Location
`/home/tvu47178/.frameworks/test_service_manager/` (isolated, reusable)

## Project Location
`/path/to/<project_folder>/` (project-specific — flexible name)

**Naming convention (flexible):**
- CR ticket: `/path/to/CRAOS8X-56190/`
- Feature name: `/path/to/l2gre_evpn/`
- project.name from config: `/path/to/ospf_unp_test/`

## Project Folder Structure
```
/path/to/<project_folder>/
├─ config/
│  ├─ project_config.json     (Project metadata + simulator command)
│  ├─ dut[n].cmds             (DUT-n configuration commands)
│  └─ verify_dut[n].cmds      (Optional: DUT-n verify commands)
├─ output/
│  ├─ dut[n]_svcmgr.log      (Generated: Config snapshot from DUT-n)
│  ├─ dut[n]_verify.log      (Generated: Verify output from DUT-n)
│  ├─ topo.puml              (Generated: Topology diagram)
│  ├─ test[n].puml           (Generated: Per-test sequence diagram)
│  ├─ sim.log                (Generated: Simulator execution log)
│  ├─ run_test_final.log     (Generated: Test execution log)
│  └─ .dut_ips               (Generated: Extracted DUT IP registry)
├─ chassis                    (Generated: DUT IP registry from `show chassis`)
├─ <name>_<timestamp>.tar.gz  (Generated: Archive bundle)
└─ README.md                  (Optional: Project-specific notes)
```

---

## Input

### Simulator Command (in config/project_config.json)
```json
{
  "simulator": {
    "command": "./sim.py -x all -s /home/mac5889/simulator/kailash_single.scf -o OS_7_X_X_9885_R01"
  },
  "test": {
    "scripts": ["run_test.sh"]
  }
}
```

### Test Scripts (test.scripts — optional)

**Script design principles:**
- Each script in the list is **1 independent test case**, handles its own timing/orchestration internally
- Scripts with timing constraints (recv before send, background process, etc.) → combine into **1 single script**
- Only split into multiple scripts if they have **no dependency** on each other
- Scripts self-report PASS/FAIL and exit 0 (pass) or exit non-zero (fail)

**Case structure detection:**
- **Multi-case project:** `cases/*/config/dut1.cmds` exists → each subdirectory in `cases/` is 1 test case
- **Single-case project:** no `cases/` → use `config/dut[n].cmds` directly

### Config Files (dut[n].cmds)
Line-delimited commands in `config/`:
```
vlan 20 admin-state enable
service access port 1/1/7
service sdp 1 l2gre far-end 2.2.2.2
```

### Chassis File (Auto-Updated at root level)
After `show chassis` from simulator, save to `./chassis`.

## Output

### Config Snapshot Files (in output/ folder)
```
output/dut1_svcmgr.log         (config snapshot from DUT1)
output/dut[n]_svcmgr.log       (config snapshot from DUT-n)
output/topo.puml               (topology component diagram)
output/test[n].puml            (per-test sequence diagram)
output/sim.log                 (simulator execution log)
output/run_test_final.log      (test execution log)
output/.dut_ips                (extracted DUT IP registry)
```

---

## Initialization Rule

1. Copy template: `project_config.template.json` → `config/project_config.json`
2. User **MUST** fill in: `project.name`, `project.description`, `simulator.command`, `duts.count`, `test.scripts` (optional)
3. Skill verifies config is complete before proceeding
4. If config missing/incomplete → Skill halts with error message

## Validate Command

Checks: config/ exists, project_config.json valid (name, description, command, duts.count), dut[n].cmds exist and non-empty, output/ exists.

**Output format:**
```
=== Validate config/ ===
[✅] config/project_config.json — OK (name: l2gre_test_v1, duts: 2)
[✅] config/dut1.cmds — OK (29 commands)
[✅] config/dut2.cmds — OK (29 commands)
[✅] output/ — OK (exists)
Result: ALL CHECKS PASSED — ready to run
```

## Diagrams Command

Generate PlantUML diagrams from config + test results, write to `output/`.

| File | Type | Content |
|------|------|----------|
| `output/topo.puml` | Component diagram | Overview topology: DUTs + ports + VLANs + tunnel logic |
| `output/test[n].puml` | Sequence diagram | Detailed flow for each test case |

**Templates:** `/home/tvu47178/.frameworks/test_service_manager/templates/`

### topo.puml — Parse from:
- `config/dut[n].cmds` → Loopback IP, VLAN IPs, port roles, services
- `config/project_config.json` → Project name, DUT count
- `.env` → TAP interface names + MAC addresses
- `chassis` → DUT management IPs

### test[n].puml — Parse from:
- Test scripts → test count + traffic direction
- `config/dut[n].cmds` → Ingress mechanism + service config
- `.env` → TAP names + MAC addresses
- `output/run_test_final.log` → Test results (PASS/FAIL)

## Pack Command

```bash
tar -czf <project_name>_<YYYYMMDD_HHMMSS>.tar.gz config/ output/ .env chassis
```

## Clean Command

```bash
rm -rf /path/to/<project_folder>/output/*
rm -f /path/to/<project_folder>/chassis
```

---

## Execution Model & Process Details

> **Full details** → see `/test-service-manager-execution`
>
> Covers: Terminal A/B/C model, Phase 1-6 details, SSH PTY management, send_to_terminal patterns.

## SSH Host Key Rule

Simulator DUTs get new host keys every boot. Before running `run.sh`:
```bash
ssh aluminium "ssh-keygen -f /home/tvu47178/.ssh/known_hosts -R <DUT1_IP> && ssh-keygen -f /home/tvu47178/.ssh/known_hosts -R <DUT2_IP>"
```
Run this proactively after getting chassis IPs, before `run.sh`.

## DUT Prompt Patterns (Per-Model)

| Platform | User-mode Prompt | Su-mode Prompt | Expect pattern |
|----------|-----------------|----------------|----------------|
| OS6870 (Kailash) | `Kailash_sim` | `KAILASH #->` | `"_sim"` / `"#->"` |
| OS6570 (Whitney) | `Whitney_sim` | `WHITNEY #->` | `"_sim"` / `"#->"` |
| OS6560 (Nandi) | `Nandi_sim` | `NANDI #->` | `"_sim"` / `"#->"` |
| OS6920 (Zion) | `Zion_sim` | `ZION #->` | `"_sim"` / `"#->"` |
| OS9900 (Medora) | `Medora_sim` | `MEDORA #->` | `"_sim"` / `"#->"` |

**Safe generic expect:** `"_sim"` for user-mode, `"#->"` for su-mode.

**Rules:**
1. When encountering a new model → observe prompt output FIRST → record the pattern
2. DO NOT use `">"` or `"#"` as expect pattern — too generic
3. `su` command → password → drops into `<MODELNAME> #->` shell

## Key Features

| Feature | Behavior |
|---------|----------|
| **Dynamic DUT Count** | Scale 1..N based on dut*.cmds files |
| **Auto IP Detection** | Extract VC EMP IPs from `show chassis` |
| **SSH Retry** | Configurable timeout + interval |
| **Config Deployment** | Line-by-line via pexpect |
| **Exit Handling** | Auto-respond to confirmation prompts |
| **Snapshot Extraction** | Per-DUT `show configuration snapshot` |
| **Verify (Optional)** | Run verify_dut*.cmds show commands |
| **Test (Optional)** | Run user-specified test scripts |
| **Progress Tracking** | Update logs at each step |

## Clean Config Rule

- **`start` (from scratch):** Simulator just booted → DUTs are clean → **NO need to clean config**
- **`run` (sim already running):** May have old config → **MUST clean config** before deploy
- **`start --endless` + re-run:** Need to clean before deploying next case

## Session Process Cache — `session_procs.tmp`

Track background processes + temp files in project folder:
```
# session_procs.tmp — <CR-ID>
# SIMULATOR: <platform> instance <N> | PID= | HOST=aluminium
# BUILD: make <platform> | PID= | HOST=mandalay
# TEMP FILES: /tmp/<cr>_*.png, /tmp/<cr>_logs.zip
# CLEANUP: kill sim, sim.py --clean, rm /tmp/<CR-ID>_*, rm session_procs.tmp
```

## Background Operations Policy (MANDATORY)

- **Simulator start**, **p4 sync**, **rsync**, **build** → ALWAYS `mode=async`
- Report terminal ID + how to check output
- Only block (sync mode) when the next step depends on the result
