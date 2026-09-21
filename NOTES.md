# Project Journal — Distributed Rate Limiter Ticket Purchasing System

A running log of decisions, blockers, and what I learned while building this. Meant to be skimmed
before interviews, not read start to finish — bullet points over prose.

---

## Sept 13, 2026 — Environment Setup

### Goal for the day
Get a Spring Boot skeleton running locally and pushed to GitHub.

### What happened

**Choosing the stack**
- Landed on Java (Spring Boot) + Redis + PostgreSQL + Docker + AWS after wanting a project that
  actually demonstrates distributed-systems concepts, not just CRUD
- Core idea: rate limiting + atomic inventory counters are only interesting because multiple app
  instances need to agree on state at the same time — that's what makes it "distributed"

**VS Code / JDK confusion**
- Initially ran commands in three different terminals (PowerShell, Git Bash, WSL) without realizing
  they're not interchangeable
- **Key learning:** PowerShell and Git Bash both read Windows' installed programs (same `PATH`). WSL
  is a fully separate Linux environment — different filesystem, different `PATH`, different
  `JAVA_HOME`. Installing Java on Windows does nothing for WSL.
- Decided to standardize on WSL as the primary dev terminal, since it's closer to how Linux
  containers/production will actually behave later (Docker, AWS)

**`JAVA_HOME` not set (WSL)**
- Error: `JAVA_HOME environment variable is not defined correctly`
- Root cause: WSL had no JDK installed at all yet, separate from Windows' JDK
- Fix: installed Amazon Corretto via apt, set `JAVA_HOME` in `~/.bashrc`

**Corretto repo signature error**
- Error: `NO_PUBKEY ... The repository is not signed`
- Root cause: a stray `~` character got appended to the keyring filename when copy-pasting a
  multi-line command (`corretto-keyring.gpg~` instead of `corretto-keyring.gpg`) — the repo list
  referenced a filename that didn't actually exist
- **Key learning:** copy-pasting piped/multi-line shell commands is a common source of invisible
  bugs. Worth typing manually or inspecting in a plain text editor first, especially for anything
  with `sudo`.
- Fix: removed the bad keyring file and repo registration entirely, redid the setup carefully

**Version mismatch (Java 26 vs LTS)**
- Had initially picked Java 26 in both Spring Initializr and the WSL install, without realizing 26
  isn't an LTS (long-term support) release
- **Key learning:** Spring Boot 3.x is built/tested against LTS versions (17, 21) — non-LTS versions
  risk obscure compatibility issues with build plugins
- Standardized on **Corretto 21** across WSL install, `JAVA_HOME`, and `pom.xml`'s `java.version`

**App boots, then fails on DataSource**
- Error: `Failed to configure a DataSource: 'url' attribute is not specified`
- Root cause: added Spring Data JPA + PostgreSQL Driver dependencies, which makes Spring Boot try to
  connect to a real database on startup — but no Postgres was running yet
- **Not a bug** — expected behavior once you add a DB dependency without a DB to point at. Next step
  is Docker Compose for local Redis + Postgres.

**Git push authentication**
- `git push` hung waiting for a password, and my actual GitHub password didn't work
- **Key learning:** GitHub disabled password auth over HTTPS a while back — need a Personal Access
  Token (PAT) instead, generated from GitHub settings, used in place of a password

**`.gitignore` / `target/` hygiene**
- Made sure `target/` (Maven's build output) was properly ignored before the first real commit, so
  compiled `.class` files never got pushed to GitHub

### State at end of day
- ✅ Java 21 (Corretto) working correctly in WSL, `JAVA_HOME` set
- ✅ `pom.xml` targets Java 21
- ✅ Spring Boot skeleton compiles and Tomcat starts
- ✅ Repo created, `.gitignore` verified, first real commit pushed successfully
- ⏳ Next: Docker Compose for local Redis + Postgres, `application.yml` wiring, then start on Phase 1
  entities (`Event`, `Ticket`)

### Reflection
Most of today wasn't "Spring Boot" at all — it was learning how Windows, WSL, and Git actually relate
to each other, which I didn't have a mental model for going in. Good reminder that environment setup
is its own skill, separate from the actual coding, and worth taking seriously rather than rushing
through.

---

## Template for future entries

```markdown
## [Date] — [What I worked on]

### Goal for the day


### What happened
- 

### Blockers hit
- Error:
- Root cause:
- Fix:
- Key learning:

### State at end of day
- ✅ 
- ⏳ Next:
```

## September 20, 2026 - Docker Setup

## Goal for the Day - 

## Learning How to Set Up a Docker Image
# docker-compose file important info
  - set volumes to postgres_data to ensure docker compose down does not wipe my database and keeps data persistent 
    across container restarts
  - healthchecks let other tools later on know when Postgres/Redis can accept connections

# application.yml file important info
  - ddl-auto: update <-- tells Hibernate to auto-create/alter tables based on @Entity classes, but only used for early development. Want to switch to Flyway or Liquibase migrations before AWS to ensure destructive schema changes by Hibernate not made
  - localhost <-- since app runs directly in WSL and Docker Compose publishes the ports to my host, localhost works here

  - **IMPORTANT DEBUGGING LESSON**: stale build cache masks config fixes: Maven's spring-boot:run does incremental builds — it skips recompiling if .class files in target/ look up-to-date by timestamp, even if the compiler config (pom.xml) changed since they were built. This can make a real fix appear not to work. When a build error doesn't match what your current config says it should be, run ./mvnw clean spring-boot:run — clean deletes target/ entirely, forcing a full fresh recompile
  - Hanging request usually means something upstream is blocked, not that the request path is broken.
  
  - Issue Today: I chased what looked like a networking issue — checked IPv6 resolution, checked Windows Firewall, checked VPN interference — before realizing the actual dependency was down and the app was blocking indefinitely waiting on it.