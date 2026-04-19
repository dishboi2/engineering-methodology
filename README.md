# Diagnostic Discipline for LLM-Adjacent Systems

A methodology document extracted from building and debugging a threat-intelligence 
platform with local LLM integration, multi-service Docker topology, and a 
human-in-the-loop forge pipeline.

This is not about any specific system. The patterns below emerged from real 
failures and real fixes, session by session. Each pattern has a failure mode 
that motivated it.

---

## 1. Identity Query Before Fix

Before reasoning about any object — a process, a file, a port holder, a database 
row, a container's actual state — run the identity query first. Do not 
pattern-match from partial data.

**The failure mode it prevents:** assuming an object is what it appears to be, 
building logic on the assumption, and tearing down something load-bearing when 
the assumption was wrong.

**The discipline:** 
- `tasklist /FI`, `Get-CimInstance`, `wmic`, `ps`, `lsof`, `netstat + PID lookup`, 
  `docker inspect`, `SELECT FROM information_schema`, `ls -la` — whichever tool 
  shows ground truth for the object class in question.
- Paste raw output, don't summarize.
- The cost of an identity query is seconds. The cost of skipping it can be hours.

**Case study:** A port-11434 listener can be a legitimate Ollama instance, 
an svchost portproxy, or a stale process. They look identical in casual 
observation. They demand different fixes. Identity query before any action.

---

## 2. Verify From The Thing Being Configured, Not The Thing Doing The Configuring

When you change configuration, verify the change took effect by reading the 
configured object's state — not by trusting the configurator's return code.

**The failure mode it prevents:** a configurator that silently fails, reports 
success, and leaves the system in an unconfigured state. You then debug as if 
the config applied, when it didn't.

**Examples:**
- Changed a Docker container's env var? Read `docker inspect` to confirm the 
  running container has it, don't trust `docker compose up -d`'s exit code.
- Applied a database migration? Query `information_schema` to confirm the 
  columns exist, don't trust the migration tool's log.
- Updated a healthcheck? Run the healthcheck command manually inside the 
  container to confirm it behaves as designed, don't trust the `HEALTHY` status.

---

## 3. Savepoint-Per-Row for Batch Database Operations

For any loop that writes multiple rows to a database from a single connection 
with autocommit disabled, wrap each row in a savepoint.

**Pattern:**

```
BEGIN
  for row in batch:
    SAVEPOINT row_scope
    try:
      do_work(row)
      RELEASE SAVEPOINT row_scope
    except:
      ROLLBACK TO SAVEPOINT row_scope
COMMIT
```

**The failure mode it prevents:** one row's failure poisons the transaction, 
every subsequent row gets "current transaction is aborted," the whole batch 
fails. Silently — the exception handler catches each failure and logs it, 
the loop continues, but every row after the first failure is already lost.

**The shape to target:** one transaction per batch. N savepoints. N releases 
on success. Zero rollbacks on a clean batch. Single commit at the end. Caller 
owns the commit, not the helpers.

---

## 4. Singleton Client for File-Descriptor-Expensive Connections

HTTP clients, database connections, gRPC channels — any client with expensive 
underlying resources should be a singleton, instantiated once per worker 
process and reused.

**The failure mode it prevents:** per-task or per-request client instantiation 
leaks sockets under load. Each task opens a new connection, the client library 
doesn't release the FD properly on task exit, and the worker exhausts its file 
descriptor ulimit. Symptom: `OSError(24, 'Too many open files')` followed by a 
retry storm that compounds the leak.

**The pattern:** module-level client, lazy-init on first use, explicit cleanup 
in worker shutdown hooks if the framework supports them.

---

## 5. Local-Capture Inside Session Scope for ORM Helpers

For SQLAlchemy tasks (or any ORM with session-scoped object lifecycles) where 
post-session logic needs attributes from a fetched entity, capture those 
attributes as plain Python locals *inside* the session's `with` block. Use 
the locals, not the entity, after session close.

**The failure mode it prevents:** `DetachedInstanceError` when code accesses 
entity attributes after the session that owned them has closed. The entity 
was fine while the session was open. The moment you exit the context, attribute 
access triggers a lazy-load that has no session to load against.

**Pattern:**

```
with get_session() as session:
    entity = session.get(Entity, id)
    do_work(entity)
    # capture what you need post-session
    entity_name = entity.name
    entity_owner_id = entity.owner_id

# session closed; use the locals, not the entity
downstream_call(entity_name, entity_owner_id)
```

**Stronger version for helpers:** keep the entire processing chain inside 
the session context. Only capture primitive summary values (counts, IDs, 
flags) for post-session use.

---

## 6. Readiness Probe With Graceful Degrade

When one service depends on another, don't trust the dependency to be ready 
when the dependent starts. Add a readiness probe with a short timeout, 
finite retries, and a graceful-degrade path when the ceiling is exceeded.

**Parameters that worked in practice:**
- 2-second timeout per probe attempt
- 5 max attempts
- 1-second backoff between attempts
- Total ceiling ~10 seconds before degrading
- Degrade path: log a clear warning, return None or a sentinel, let callers 
  skip dependency-specific logic

**The failure mode it prevents:** dependent service uses a default HTTP client 
with no timeout, blocks indefinitely on an unavailable dependency, and the 
whole process hangs until an operator intervenes. 11-minute hang from an 
unready vector database is a real observed failure of this class.

---

## 7. The "Lying Healthcheck" Antipattern

A healthcheck that exits 0 instantly without verifying the service is actually 
serving its purpose is worse than no healthcheck at all.

**Why worse:** a missing healthcheck fails fast and visibly. `depends_on` 
without `service_healthy` doesn't work, and you see the error. A lying 
healthcheck produces correct-looking behavior that silently breaks under load. 
`depends_on: service_healthy` against `CMD: true` looks correct in yaml and 
waits for nothing.

**The rule:** healthchecks must verify the service is actually serving its 
purpose, not that its process has started. TCP probe for port-bound services, 
HTTP probe for HTTP services, actual query probe for databases. Null probes 
are the configurator pretending to be the configured.

---

## 8. Rescue-Copy-With-Writer-Pause for Destructive Infrastructure Ops

Any infrastructure change that could destroy persistent state must earn the 
data's survival, not assume it.

**Sequence:**

1. Identity-query current state (what exists, where, how much)
2. **Pause or stop all writers to the affected subsystem**
3. Rescue copy to host-level storage
4. Pre-state snapshot for post-verification (counts, checksums, representative 
   samples)
5. Apply the change
6. Seed from rescue
7. Verify preservation against the pre-state snapshot
8. Resume writers
9. Keep rescue on disk for a cooldown period (24 hours minimum) before cleanup

**The writer-pause step is load-bearing.** Without it, writes between the 
rescue copy and the destructive op land in the old state and are destroyed 
when the op runs. On a self-healing subsystem this costs you a small amount 
of data that re-writes itself within minutes. On a non-self-healing subsystem, 
the same gap is unrecoverable data loss.

**Writer enumeration as prerequisite:** if you cannot enumerate every writer 
to the subsystem, stop. You do not yet understand the blast radius of the 
destructive op.

---

## 9. Scope Gates Between Sessions

Work ships in discrete sessions with explicit verification criteria. Each 
session fixes one thing, verifies it fixed, commits, and closes. Scope creep 
within a session ("one more small thing") is the path to sessions that can't 
close cleanly.

**Markers of a session closing cleanly:**
- The original verification criteria pass with raw-output evidence
- Any bugs surfaced during the session that aren't in scope are documented 
  in a scope-notes file with enough detail for a future session to pick up
- A commit with a message that names what was fixed and what it exposed 
  (if anything)
- The git log reads as a story, not a pile

**The cost of the discipline:** you ship fewer fixes per hour. You can't 
"just fix this other small thing while I'm here." 

**The benefit:** when a regression appears three weeks later, you can bisect 
through clean commits and find the change that caused it. You can't do that 
through a sprawling commit that fixed five things at once.

---

## 10. Identity Queries Compound; Assumption Chains Compound

A session that applies discipline 1-9 does not just fix the bug at hand. It 
reveals the *next* bug that was structurally invisible while the outer bug 
ran interference.

**Observed pattern across multiple sessions:**
- Session A fixes a transaction cascade, which reveals a latent SQL projection 
  bug that was hidden because an earlier exception masked it.
- Session B fixes the SQL bug, which reveals a cursor-scope bug whose wrong 
  output was being discarded by the prior error path.
- Session C fixes the cursor bug, which reveals an entirely empty downstream 
  pipeline whose emptiness was being masked by a baseline default value.
- Session D fixes the pipeline, which reveals an ORM session-scope bug that 
  only executed when the pipeline actually ran.

Each fix is structural. Each fix reveals the next layer. This is not a sign 
of things going wrong; it is the diagnostic order working correctly. Plan 
for it. Expect every session to potentially surface the next one.

**The stopping signal:** a session whose verification completes without 
surfacing a new layer. That session closes the chain for its subsystem.

---

## 11. Cross-Shell Variable Expansion Hygiene

When invoking commands through shell layers — e.g., `wsl bash -c "..."` from 
a Windows cmd or PowerShell context, or bash-through-ssh, or any nested shell 
invocation — variable expansion is unreliable. The outer shell may consume 
the variable before it reaches the inner shell, and the inner command runs 
with an empty value.

**Failure mode:** `docker cp container:/data/. $RESCUE` where `$RESCUE` is 
defined in the outer shell but not exported to the inner WSL context. The 
outer cmd.exe or PowerShell layer reads `$RESCUE` as empty, and `docker cp` 
runs as `docker cp container:/data/. /` — scattering the container's state 
across the inner shell's root filesystem.

**Mitigations:**
- Use literal paths inside the inner command
- Write the command to a `.sh` file and invoke the file by path
- Use a heredoc that the inner shell parses
- If variables are required, confirm export behavior with a deliberate echo 
  test before running the destructive command

---

## Why Document This

These patterns are not novel individually. Most of them are folklore in 
systems engineering — savepoints, singletons, readiness probes, careful 
healthchecks. What the discipline adds is the *habit of applying them 
systematically*, the *meta-pattern of expecting layered bugs*, and the 
*scope gate between sessions* that keeps diagnostic work legible.

The cost is slower fixes in the short term. The benefit is a system that 
remains debuggable as it grows, and a history that future-you (or a 
collaborator) can read.

---

*Extracted from production debugging sessions. Patterns earned through 
failure, not invented from principle.*
