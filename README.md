# Lab: Opening Custom Ports — When There Is No `firewall-cmd` Service Name

**Series:** linux-ops-mastery — RHCSA Firewall
**Subjects covered:** `firewall-cmd --add-port`, `--remove-port`, port syntax `PORT/PROTO`, `--permanent`, `firewall-cmd --reload`, verifying with `--list-ports`, rich vs simple rules preview
**Career arcs covered:** RHCSA (non-standard daemon ports), RHCE (parameterized playbooks), SRE (vendor apps on random high ports), DevOps (Kubernetes NodePort ranges), AI/MLOps (Jupyter, Ray, custom training APIs)
**Prerequisite:** Lab **firewalld-add-services** (persistence/reload rhythm)
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 baseline ports · 2–3 runtime add/remove custom · 4 permanent + reload · 5 edge: bad syntax · 6 capstone + cleanup

---

## Objective

Not every workload maps to `/usr/lib/firewalld/services/*.xml`. Your company might run gRPC on **50051/tcp**, an internal app on **8444/tcp**, or a metrics scraper on **19100/tcp**. `firewall-cmd --add-port` is the precise tool: you specify **`number/protocol`** and optionally zone scope, mirror to **`--permanent`**, **`--reload`**, and verify with **`--list-ports`**.

This lab intentionally avoids inventing new service XML — you practice the **atomic port open**, the rollback, and the “syntax police” errors that appear on exams when students type `8444-tcp` instead of **`8444/tcp`**.

> **Lab safety note:** Open ports are real attack surface. Task 6 **removes** permanent port rules and reloads — complete Cleanup on shared systems.

---

## Concept: Ports Are Scalar Matches, Services Are Bundles

```
Service "http"
   └─► expands to 80/tcp (+ optional extras)

Custom port 8444/tcp
   └─► exactly one allow match — no friendly alias unless you build XML
```

Visually inside a zone:

```
┌──────────── public zone runtime rules ────────────┐
│ services: ssh ...  ──► multiple derived matches    │
│ ports: 8444/tcp    ──► explicit scalar match      │
└────────────────────────────────────────────────────┘
```

> **Why this matters:** Exam scenarios love odd ports because they test whether you know the **`/tcp` vs `/udp`** delimiter — not whether you memorized Apache trivia.

---

## 📜 Why Custom Port Openings Exist — The Story

Historically, host firewalls listed raw **matches** — protocol, source, destination, port — long before friendly names. That precision never left us: even with `firewalld` services, production systems always sprouted **one-off ports** for monitoring endpoints, license servers, database replicas, and ad-hoc dashboards.

`firewalld` keeps services and ports as **parallel axes** on the same zone object: you can allow `https` **and** `8444/tcp` together without merging them into one XML definition. Red Hat systems have used `firewalld` as the default firewall service since **RHEL 7**, evolving the backend while keeping the **`PORT/PROTO`** string format stable for scripting.

> **The point of the story:** Ports are the escape hatch when the catalog does not know your application’s name yet.

---

## 👪 The Custom Port Family — Who Lives There

### By syntax token

| Token | Meaning |
|---|---|
| `8444/tcp` | TCP port 8444 |
| `19100/udp` | UDP port 19100 |
| `1000-2000/tcp` | Inclusive TCP range (powerful — use carefully) |

### By persistence

| Command shape | Effect |
|---|---|
| `--add-port=8444/tcp` | Runtime only |
| `--permanent --add-port=8444/tcp` | Survives reboot after reload |
| `--reload` | Applies permanent → runtime |

### By verification

| Question | Command |
|---|---|
| What ports are open? | `firewall-cmd --list-ports` |
| Saved for boot? | `firewall-cmd --permanent --list-ports` |

> **The point of the family tree:** Slash direction matters — **`8444/tcp`**, not `8444-tcp`.

---

## 🔬 The Anatomy of `--add-port=8444/tcp` — In One Diagram

```
$ firewall-cmd --add-port=8444/tcp
success

$ firewall-cmd --list-ports
8444/tcp
```

Parser mental model:

```
 8444   /   tcp
   │       └─ L4 protocol keyword (tcp|udp|sctp|dccp)
   └─ decimal port number (IANA assigned or local admin choice)
```

> **Reading rule:** Always echo the string back with `--list-ports` — humans transpose digits under stress.

---

## 📚 Custom Port Reference Table

| Task | Command | Notes |
|---|---|---|
| Runtime open | `firewall-cmd --zone=public --add-port=50051/tcp` | gRPC-style example |
| Permanent open | `firewall-cmd --permanent --zone=public --add-port=50051/tcp` | Writes zone config |
| Reload | `firewall-cmd --reload` | Apply saved state |
| Remove | `--remove-port=50051/tcp` | Symmetric |
| List | `--list-ports --zone=public` | Runtime view |
| Rich rules (future topic) | `firewall-cmd --list-rich-rules` | When you need source IP scoping |

> **Rule one of ports:** If `--reload` “removes” your test, you forgot `--permanent`.

---

## 🧪 Extended Verification Playbook (Optional Depth)

| Situation | Command | What “good” looks like |
|---|---|---|
| Listener proof | `ss -lntp \| grep ':9443'` | Shows process bound — firewall alone never starts daemons |
| UDP listener proof | `ss -lunp \| grep ':19100'` | Confirms UDP expectations |
| Protocol typo drill | `firewall-cmd --add-port=9443/udp` then `list-ports` | Different intent than lab’s TCP choice |
| Range awareness | `firewall-cmd --permanent --add-port=2000-2010/tcp` (DO NOT reload in shared labs) | Understand power — then remove if you tried it |
| Compare to rich rules | `firewall-cmd --list-rich-rules` | When you need source IP scoping beyond simple opens |
| SELinux cross-check | `getenforce` | `Enforcing` may still block bind despite open port |
| nft angle | `nft list ruleset \| head` | Sometimes clearest compiled truth |
| Rollback rehearsal | Save `list-ports` before/after in notes | Same discipline as enterprise change windows |

Treat this block as **post-lab enrichment** — deepen after you can already complete Tasks 1–6 cold.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Port syntax questions are fast points if your fingers know the slash. |
| **RHCE candidate** | Jinja templates often build `item.port ~ '/' ~ item.proto` strings — same grammar. |
| **SRE / Platform** | Vendor docs give port lists — map them literally with `--add-port`. |
| **DevOps** | NodePort ranges might be opened as ranges — practice caution. |
| **AI / MLOps** | Random high ports for sidecars are normal — never expect a service XML. |

---

## 🔧 The 6 Tasks

> Uses `ZONE=public` — adjust if your classroom uses another.

---

### Task 1 — Baseline: list runtime vs permanent ports (often empty)

**Purpose:** Confirm starting emptiness so later diffs are obvious.

```bash
sudo -i
ZONE=public
echo "RUNTIME PORTS:"; firewall-cmd --list-ports --zone=$ZONE
echo "PERMANENT PORTS:"; firewall-cmd --permanent --list-ports --zone=$ZONE
```

**Human-Readable Breakdown:** Stock VMs frequently show **blank** port lines — services carry HTTP/S, not explicit `80/tcp` here.

**Reading it left to right:** Two reads, two namespaces — same lesson as services lab.

**The story:** Empty baseline is good — any new token is your change alone.

**Expected output:**

```text
RUNTIME PORTS:

PERMANENT PORTS:

```

**Switches**

| Token | Meaning |
|---|---|
| `--list-ports` | Shows `PORT/PROTO` tokens only |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Many unexpected ports | Host drift — note and clean with owner approval |

---

### Task 2 — Core A: add `9443/tcp` at runtime and verify

**Purpose:** Immediate open for lab API imaginary workload.

```bash
firewall-cmd --zone=$ZONE --add-port=9443/tcp
firewall-cmd --zone=$ZONE --list-ports
```

**Human-Readable Breakdown:** `9443/tcp` is plausible for alternate-HTTPS admin UIs — memorable digits.

**Reading it left to right:** Add → list; token should round-trip identically.

**The story:** Runtime-only is great for “does the daemon actually bind?” smoke tests.

**Expected output:**

```text
success
9443/tcp
```

**Switches**

| Token | Meaning |
|---|---|
| `--add-port=PORT/PROTO` | Single opening |
| `--list-ports` | Verification |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `INVALID_PORT` | Check slash direction and protocol keyword |

---

### Task 3 — Core B: remove runtime port to rehearse rollback

**Purpose:** Symmetric `--remove-port` before touching permanent files.

```bash
firewall-cmd --zone=$ZONE --remove-port=9443/tcp
firewall-cmd --zone=$ZONE --list-ports
```

**Human-Readable Breakdown:** Removal should return empty ports list (unless other ports existed pre-lab).

**Reading it left to right:** Same string token used for add and remove — consistency matters.

**The story:** Rollback strings must match **exactly**, including protocol.

**Expected output:**

```text
success

```

**Switches**

| Token | Meaning |
|---|---|
| `--remove-port` | Deletes matching opening |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| NOT_ENABLED | Port wasn’t present — `list-ports` first |

---

### Task 4 — Permanent + reload: open `50051/tcp` and `19100/udp`

**Purpose:** RHCSA persistence combo on two protocols.

```bash
firewall-cmd --permanent --zone=$ZONE --add-port=50051/tcp
firewall-cmd --permanent --zone=$ZONE --add-port=19100/udp
firewall-cmd --reload
echo RUNTIME; firewall-cmd --list-ports --zone=$ZONE
echo PERMANENT; firewall-cmd --permanent --list-ports --zone=$ZONE
```

**Human-Readable Breakdown:** Order stays: **write permanent** → **reload** → **dual list**.

**Reading it left to right:** Two different protos prove you understand **`/tcp` vs `/udp`**.

**The story:** Reload is the handshake between **disk truth** and **kernel truth**.

**Expected output:**

```text
success
success
success
RUNTIME
50051/tcp 19100/udp
PERMANENT
50051/tcp 19100/udp
```

**Switches**

| Token | Meaning |
|---|---|
| `--permanent` | Durable configuration |
| `--reload` | Apply |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Only one proto shows | Re-run the missing `--add-port` |

---

### Task 5 — Edge case: trigger syntax errors + optional safe range peek

**Purpose:** Learn failure strings and respect range power.

```bash
firewall-cmd --add-port=8444-tcp 2>&1 | head -n 3
firewall-cmd --add-port=8444/tcp --zone=$ZONE
firewall-cmd --list-ports --zone=$ZONE
```

**Human-Readable Breakdown:** First add uses **wrong delimiter** (`-` not `/`) — expect error. Second fixes grammar.

**Reading it left to right:** stderr merged for easy capture; success path adds valid port alongside prior permanents.

**The story:** Under exam adrenaline, the slash is what separates first attempt from second attempt.

**Expected output:**

```text
Error: INVALID_PORT: 8444-tcp
success
50051/tcp 19100/udp 8444/tcp
```

**Switches**

| Token | Meaning |
|---|---|
| `2>&1` | Merge stderr for classroom display |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Error text varies | Intent still INVALID_PORT |

---

### Task 6 — Capstone: document open ports, then strip all lab additions

**Purpose:** Prove complete rollback of **both** permanent test ports and runtime extra.

```bash
{
  date
  firewall-cmd --list-ports --zone=$ZONE
} | tee /tmp/ports-lab.txt
cat /tmp/ports-lab.txt
```

**Human-Readable Breakdown:** Snapshot should list the three lab ports from Tasks 4–5.

**The story:** Capstone = “I can open odd ports, persist them, and remove them without ghost state.”

**Expected output:**

```text
50051/tcp 19100/udp 8444/tcp
```

**Switches**

| Token | Meaning |
|---|---|
| `tee` | Evidence file |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Extra ports from old labs | Remove deliberately or reset VM image |

**Cleanup**

```bash
firewall-cmd --permanent --zone=$ZONE --remove-port=50051/tcp
firewall-cmd --permanent --zone=$ZONE --remove-port=19100/udp
firewall-cmd --permanent --zone=$ZONE --remove-port=8444/tcp
firewall-cmd --reload
firewall-cmd --list-ports --zone=$ZONE
firewall-cmd --permanent --list-ports --zone=$ZONE
rm -f /tmp/ports-lab.txt
```

---

## 🔍 Custom Port Decision Guide

```
Need to expose a listener?
  │
  ├── "Is there a stock service name?"
  │       └── YES → Lab 58 --add-service
  │
  ├── "Single TCP/UDP port?"
  │       └── --add-port=PORT/proto (runtime) → permanent mirror → reload
  │
  ├── "Entire range?"
  │       └── Range syntax allowed — tighten source IPs with rich rules later
  │
  └── "Still blocked?"
          └── Check SELinux (`getenforce`, booleans) and process bind address
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Compare runtime vs permanent port lists (likely empty)
- [ ] 02 Add `9443/tcp` runtime; verify with `--list-ports`
- [ ] 03 Remove `9443/tcp` runtime; verify empty
- [ ] 04 Add permanent `50051/tcp` + `19100/udp`; reload; verify parity
- [ ] 05 Provoke INVALID_PORT then add valid `8444/tcp`
- [ ] 06 Snapshot with `tee`, then Cleanup permanent removals + reload + delete `/tmp` file

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `8444-tcp` typo | INVALID_PORT | Use slash |
| Forgot reload | “It vanished” confusion | `firewall-cmd --reload` |
| Opened UDP for TCP app | Connection reset mystery | Match daemon protocol |
| Huge ranges casually | Wide blast radius | Narrow with rich rules |
| Removed port from runtime only | Reappears after reload | Remove from `--permanent` too |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Drill muscle memory: **`digits/slash/letters`** — say it out loud while typing.

**RHCE candidate**
- Show how you’d loop `firewall-cmd` tasks idempotently with `changed_when` based on `--list-ports`.

**SRE / Platform interview**
- Discuss compensating controls when wide NodePort ranges must open.

**DevOps**
- Keep port registry YAML aligned with actual `list-ports` output in CI drift tests.

**AI / MLOps**
- Sidecar health checks on high ports — document proto (`grpc` often tcp).

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [firewalld-add-services](https://github.com/kelvintechnical/firewalld-add-services) | Named opens + reload rhythm |
| [active-firewall-zones](https://github.com/kelvintechnical/active-firewall-zones) | Which zone actually owns the port list |
| [inspecting-iptables](https://github.com/kelvintechnical/inspecting-iptables) | Seeing counters after traffic hits |

---

## 🎓 After the Lab — 60-Second Oral Exam

Answer out loud without scrolling:

- What delimiter separates port number from protocol in `--add-port`?
- Why is `9443-tcp` invalid while `9443/tcp` is valid?
- What two-protocol pair did you persist in Task 4?
- Why does opening a firewall port not guarantee the daemon listens?
- Which command removes a permanent port before `reload`?
- How would you open **UDP** `9999` with one flag pair?
- Why might SELinux still block clients even when `list-ports` looks correct?
- What is the danger of wide port **ranges** on edge-facing NICs?
- Name the log unit you would tail after a failed `--add-port`.

If any answer wobbles, redo Tasks 2–4 slowly — syntax is the graded tripwire.

```text
PORT/PROTO quick card:  443/tcp   53/udp   1000-1010/tcp
Never write:           443-tcp   53-udp   (wrong)
```

> **Proctor hint:** If your fingers type a hyphen where a slash belongs, **stop** — fix the token before you `reload`.

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
