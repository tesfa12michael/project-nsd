# Project NSD — New Server Deployment

**An AI-supervised deployment pipeline that ships a secured, full-stack web server from a single form submission.**

An operator fills in a client name, a target server, a domain, a repository or two, and submits. What comes back is either a live, attack-tested link, or a precise escalation naming exactly where the build stopped and why. No senior engineer sits with it while it runs.

The compute cost of a full run is about six US dollars.

| | |
|---|---|
| **Human time per deployment** | ~1–3 person-days of senior work → one form submission, unattended |
| **Wall clock** | Hours to days → ~20–40 minutes |
| **WAF attack-block score (live)** | 10/13 → **12/13**, the threshold the suite reports as production-ready |
| **Cost per deployment** | ~$1,860 contractor rate (Ethiopia) → **~$6** compute, a ~300× local reduction |
| **Failure behavior** | Pipeline stops or continues on a broken base → bounded self-heal, cap 5, then a human |

Full reasoning, threat model, postmortem, and cited value analysis: **[docs/NSDCaseStudy.pdf](docs/NSDCaseStudy.pdf)**.

---

## Why this exists

Standing up a hardened full-stack server is not one task. It is firewall rules, a reverse proxy, a web application firewall, a database with the schema grants modern Postgres actually requires, a backend, a frontend build that will run the box out of memory if you let it, routing, TLS, and a verification pass that means something. Each stage has a failure mode that only surfaces at the end.

In Ethiopia, a contractor handling that engagement can charge up to 300,000 birr — roughly $1,860 at the May 2026 rate. That is not a local quirk. Freelance infrastructure-automation projects are quoted at $2,000–$5,000 globally, and senior DevOps work bills past $150 an hour. It is the worldwide price of an afternoon of scarce expertise.

The cost that lands after the invoice is the one worth naming. Human error accounts for an estimated 80% of IT outages, and a misconfigured or absent WAF is a direct path to an incident whose average cost reached $4.44M globally in 2025. The gap between *the deploy finished* and *the server is defensible* is where those numbers live.

---

## How it works

Three environments, connected by two narrow SSH hops. Nothing pretends they are the same machine.

```
  OPERATOR
     │  form: client, target, domain, repos, options
     ▼
┌─────────────────────────────────────────────────────────────┐
│ ORCHESTRATION · n8n (Docker)                                │
│   clean data → prep client files, SSH trust, snapshot       │
│   → role orchestrator (9 roles, in order) → Telegram        │
└─────────────────────────────────────────────────────────────┘
     │  SSH · container → host
     ▼
┌─────────────────────────────────────────────────────────────┐
│ COMMAND CENTER · Ansible on host                            │
│                                                             │
│   SELF-HEALING SUPERVISOR (per role)                        │
│     run role ──▶ read exit code ──▶ 0 ? ──▶ next role       │
│                       │                                     │
│                     fail                                    │
│                       ▼                                     │
│     Qwen agent ──▶ ONE fix, via an allowlisted tool         │
│     that can write only to this client's config files       │
│                       │                                     │
│                       ▼                                     │
│     attempt ≤ 5 ? ──▶ re-run role                           │
│                  └──▶ escalate to a human                   │
└─────────────────────────────────────────────────────────────┘
     │  SSH · host → target
     ▼
┌─────────────────────────────────────────────────────────────┐
│ TARGET SERVER · built role by role                          │
│   hardening → nginx → modsecurity → postgresql →            │
│   php-laravel → react → nginx-routing → ssl → verification  │
│                                                             │
│   gate: 13-check attack suite + independent HTTP GET        │
└─────────────────────────────────────────────────────────────┘
```

A run moves through four movements. **Intake** cleans the form, writes per-client files on the control plane, establishes SSH trust, and takes a pre-deployment snapshot. **Build** runs nine Ansible roles in dependency order, each wrapped in the supervisor above. **Security gate** fires a 13-check suite at the live domain; if the machine is not blocking properly, a bounded agent tunes the firewall and the suite runs again. **Verification and handoff** confirms services are active, has n8n independently request the public URL, and reports to Telegram with the link.

The order is the real dependency chain of the work. You cannot route traffic to an application that does not exist, cannot obtain a certificate before the domain resolves, and cannot honestly verify security before the thing you are securing is up. A failure early in the chain halts the rest rather than stacking work onto a broken foundation.

### The one behavior that defines the system

Every role-running command ends by printing its exit code, and the workflow reads it.

The original pipeline ended each command with `echo "EXIT_CODE:$?"` and then never parsed the result. Because the `echo` always succeeded, the SSH node reported success even when the playbook underneath it had failed, and a broken step flowed quietly into the next one. The core technique of the rebuild is simple to describe and consequential in effect: parse the code, branch on it, and treat *no visible error* as the beginning of verification rather than the end of it.

### Bounded agency

The AI proposes. Deterministic code disposes.

The self-heal agent reads a sanitized error and the failing client files, and makes exactly one change — only to files inside that client's own directory, enforced by an allowlist that refuses the shared Ansible roles, the SSH keys, the vault, and the inventory's target line. It cannot run arbitrary shell. It cannot touch another client. It does not re-run the role, and it does not decide how many attempts remain; the workflow owns the counter. When a fix needs to reach the live server, a deterministic node deploys it and restarts Nginx.

That pattern generalizes beyond this project: let the model analyze and propose inside a narrow, typed interface, and let ordinary code execute, count, and gate.

### Independent verification

The final proof is an HTTP request made by n8n itself against the public URL — not a `curl` run on the target checking its own health. A server asserting that it is up is not evidence. An external client reaching it is. That distinction is the whole reason the success message can be trusted.

---

## The stack, and what each part is doing

| Layer | Component | Role in the system |
|---|---|---|
| Orchestration | **n8n** (self-hosted, Docker) | Owns the form, control flow, retry counters, escalation gates, and integrations |
| Execution | **Ansible** | Does the actual building. Idempotency is what makes re-running a failed role safe |
| Defense | **ModSecurity v3 + OWASP CRS v4** | WAF engine and ruleset, tuned to paranoia level 2 at anomaly threshold 5 |
| Data | **PostgreSQL** | One isolated database, user, and grant set per client |
| Backend | **PHP 8.3 + Laravel** | Deployed with Composer, migrated on the box |
| Frontend | **Node 20 + React (Vite)** | Built on the server with a swap file and a raised heap ceiling |
| Edge | **Nginx + Certbot / Let's Encrypt** | Reverse proxy and static host; single-domain and split-domain templates; TLS and renewal |
| Intelligence | **Qwen 3.7** (Alibaba Cloud) | The self-heal agent. Selected as capable, inexpensive, and already integrated |
| Human loop | **Telegram** | Success reports and escalations |

Ansible's idempotency is not a nice-to-have here. It is the foundation the entire retry mechanism stands on: a role that half-completed and then failed can be re-run without corrupting the state it already reached.

The model is treated as a replaceable component. The guardrails around it — the allowlist, the deterministic counter, the human escalation — are the parts that are not.

---

## Security model

The design rule underneath everything: nothing that arrives from outside is trusted, and nothing sensitive is ever allowed to leave. Form values are attacker-controlled until proven otherwise. The AI's output is a suggestion, never a command. Secrets exist to be used by the machine and never to be seen — not in a log, not in a prompt, not in an error message, not in the execution history n8n keeps by default.

| Surface | Control |
|---|---|
| Form input → command injection | Values validated against an expected shape and written into structured per-client files. Nothing from the form is interpolated into a shell command. Playbooks and profiles are allowlisted, not free-text |
| Prompt injection → the agent | One tool, writing only inside the current client's config directory. No shell, no cross-client reach, no control over its own retry budget. Injection can waste an attempt; it cannot escalate |
| Secret leakage → logs and execution data | `no_log` on sensitive tasks, diff disabled for secret-bearing templates, sanitized error text to the model and the operator. Credentials referenced by name from n8n's store and Ansible Vault, never inlined |
| SSH trust across three environments | Two scoped keypairs rather than one shared key. Host-key verification stays on. No security downgrade for convenience |
| Result spoofing → false success | The suite attacks the live domain and n8n makes an independent request. Success is a measured fact, not a claim |

The most important architectural decision in the system is a security decision disguised as a plumbing detail: n8n and Ansible run in different environments, and the only thing connecting them is a narrow SSH hop. Installing Ansible inside the n8n container would have been easier, and would have collapsed two trust domains into one — handing the layer exposed to form input and model output direct execution rights on the host. The boundary is inconvenient on purpose.

The agent is treated as the least-trusted component in the system, not the most. It is the one piece that can be steered by adversarial text, so it gets the smallest blast radius available: one tool, one directory, no shell, a counter it cannot touch, and a human at the end of the rope.

Because secrets are referenced by name and never inlined, sanitizing this project for publication was a mechanical step rather than an audit. There were no values to scrub, because none were ever written down.

---

## What's in this repository

```
docs/
  NSDCaseStudy.pdf              Full case study — architecture, threat model,
                                postmortem, cited value analysis
workflows/
  projectnsd.json               Main pipeline (33 nodes): form intake, prep,
                                role orchestration, security gate, handoff
  nsdrunroleselfhealing.json    Per-role supervisor (13 nodes): run, read exit
                                code, agent fix, bounded retry, escalate
  nsdclientfileopstool.json     The agent's only hands (4 nodes): allowlisted
                                writes to one client's config directory
Sanitized/
  nsdansibleplaybooks.zip       Ansible control plane — 9 roles, 5 playbooks,
                                templates, WAF suite, example client
  *.json                        Copies of the three workflow exports above
```

Inside the archive:

- **9 roles** — `server-hardening`, `nginx-install`, `modsecurity`, `postgresql`, `php-laravel`, `react-frontend`, `nginx-routing`, `ssl-setup`, `verification`
- **Deployment profiles** — A (single server), B (web + database), C (web + database + custom), plus a full-stack master playbook and a single-role test playbook
- **`files/waf-test.sh`** — the 13-check gate: SQLi, XSS, command injection, path traversal, remote file inclusion, oversized payloads, and a false-positive check against normal traffic. Passes at ≥ 12/13
- **`clients/example-client/`** — the per-client model: inventory, vars, ModSecurity overrides, and environment examples
- **`PLACEHOLDERS.md`** — every token to replace before a run

---

## Running it

Validated on Ubuntu. Other platforms extend through an explicit compatibility matrix, not a universal promise.

**You will need:** a control-plane host with Ansible installed, a self-hosted n8n instance in Docker on that host, a target server, a domain pointed at it, an Alibaba Cloud credential for the model, and a Telegram bot and chat ID.

1. **Unpack the control plane.** Clone the repository and extract the Ansible archive onto the host.

   ```bash
   git clone https://github.com/tesfa12michael/project-nsd.git
   unzip project-nsd/Sanitized/nsdansibleplaybooks.zip
   ```

2. **Generate two keypairs.** Never one shared key.

   ```bash
   ssh-keygen -t ed25519 -f ansible_key -N ""
   ssh-keygen -t ed25519 -f n8n_local_key -N ""
   ```

   Trust `n8n_local_key.pub` in the host's `authorized_keys`, and provision `ansible_key.pub` to the target. The n8n-side private key must be readable by the `node` user inside the container — permissions `644`, not `600`. That is not a typo; see the postmortem below.

3. **Replace the placeholders.** Search the tree for `_HERE` and work through `PLACEHOLDERS.md`: server IP, app domain, API domain, Let's Encrypt email, database password. Values in `group_vars/all.yml` and `roles/*/defaults/main.yml` are fallbacks only — real deployments are driven by `clients/<client>/vars.yml`.

4. **Import the workflows** into n8n, then attach your own credentials: SSH, Alibaba Cloud, Telegram. Every credential is referenced by name, so nothing needs editing inside the workflow JSON.

5. **Submit the form.** The pipeline handles the rest, and reports to Telegram either way.

Two different clients are two different directories under `clients/`, and nothing else. The shared roles are never edited to accommodate a deployment. That single rule is what makes the system reusable rather than a one-off.

---

## What broke

The fixes are where the engineering lives. The case study covers four failures in full; these two matter most to anyone reading the code.

**The WAF that inspected everything and blocked nothing.** ModSecurity was installed, 800+ OWASP rules were loaded, and the attack suite sailed straight through — every payload returned `200 OK`. `SecRuleEngine` was `On`. The rules were loaded. The site still failed. The layer below the on/off switch is the CRS anomaly-scoring model, governed by a paranoia level that decides which rules even execute. The ruleset was sitting at level 1, catching blunt attacks and missing evasions. Raising blocking and detection paranoia to level 2 while keeping the anomaly threshold at the recommended 5 closed the evasions without flooding the site with false positives. The real challenge wasn't turning the WAF on. It was understanding that *on* and *blocking effectively* are two different dials.

**A passing test that was lying.** Two branch conditions read an upstream value through n8n's paired-item reference. After each role was rewired through a sub-workflow, that reference silently stopped resolving, because a sub-workflow's output does not carry the item lineage it depends on. The conditions did not error. They were rescued by a second, redundant condition that happened to be true, so the branches "passed" while ignoring the field they were meant to evaluate. This is the exact hazard of visual orchestration: a type checker would have caught it, and the canvas hid it behind a green checkmark.

Also worth knowing: the slowest step in a run is compiling ModSecurity v3 from source, at fifteen to twenty-five minutes on a fresh target. It dominates wall-clock time. A prebuilt module is the obvious next optimization; it was left as-is because it is idempotent and correct, and correctness came first.

---

## Trade-offs and limits

**n8n over a bespoke orchestrator.** A custom Python or Go service could have run this. n8n won because the problem is control flow with human gates, retries, and a dozen integrations, and n8n makes that visible and modifiable without a redeploy. The cost is real: a visual graph is harder to unit-test than a codebase, and expression bugs hide where a type checker would catch them. The second failure above is exactly such a bug.

**Ansible over shell scripts.** A shell script that fails halfway leaves the box in an unknown state. An Ansible role that fails halfway can be re-run to converge. Since the entire self-heal mechanism depends on safely re-running a failed role, shell was never actually an option. The cost is a steeper concept — roles, inventory, variable precedence — for the one property the system cannot live without.

**The agent writes config, but never executes.** Handing the model an open SSH tool would have been faster to build and far less safe. An agent with an unrestricted shell is a liability whose blast radius is the whole server. Given a choice between a capable agent and a governable one, this system chooses governable every time.

**WAF hardening seeded per client, not in the shared role.** The durable fix could have gone into the shared ModSecurity role. It went into per-client configuration instead, seeded at file-creation time, to preserve the rule that shared automation is never edited for a specific deployment. The same two lines now live in every client folder rather than in one place. For a reusable system, that redundancy is the correct price.

**Where the value stops.** This does initial deployment, and it does it well. It is not drift detection, ongoing patching, or disaster recovery. The ModSecurity compile is slow. The reusable model is validated on Ubuntu. The value is real and it is bounded, and saying so is the difference between a case study and a sales page.

---

## Reflections

Supervised, self-healing, security-gated deployment has generally lived inside platform teams at companies with platform-team budgets. There is no longer a good reason for that. Everything here is open-source, self-hostable, or priced in cents: n8n on a server you own, Ansible from Red Hat, ModSecurity and the OWASP CRS from the community, and an inexpensive model doing a small, bounded job. It fits on two modest virtual machines.

The real challenge with this workflow isn't teaching an AI to fix a broken deployment. Models are good at that, and getting better weekly. The real challenge is building the structure around the AI that makes its help trustworthy — the allowlist that bounds what it can touch, the counter it doesn't control, the human it has to answer to, the independent check that proves the result. The intelligence was the easy part. The governance was the engineering.

---

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, run it against your own infrastructure.

---

**Tesfa Michael** — AI Engineer building production systems across automation, ML, and applied AI. Built with a business operator's mind.

Every figure in this README is measured on the live system or cited to a named source in [the case study](docs/NSDCaseStudy.pdf).
