# Learning Plans: CI/CD, SQL, Linux

## Scope

Three 4-week zero-to-proficient learning plans, matching the format and depth of the existing Python, Docker, and K8s plans.

## Target Audience

Learner who has completed Python + Docker + K8s fundamentals. Plans leverage this foundation where applicable (e.g., CI/CD plan uses Docker+K8s for deployment; SQL plan uses Python for DB access; Linux plan ties into container internals).

## Design Decisions

| Topic | Decision | Rationale |
|-------|----------|-----------|
| CI/CD tool | GitHub Actions | Cloud-native, free tier, natural fit with existing Docker/K8s knowledge |
| Database | PostgreSQL | Feature-complete, SQL standards compliant, strong K8s/cloud-native ecosystem |
| Linux depth | User-space focus | Complements container knowledge; networking week ties directly into Docker/K8s internals |

## Plan Structures

### CI/CD (GitHub Actions) — 4 weeks
- Week 1: Concepts, triggers, first pipeline
- Week 2: Matrix build, caching, artifacts, secrets
- Week 3: Docker image build/push, K8s deployment (ties into existing knowledge)
- Week 4: Environments, reusable workflows, full CI/CD pipeline project

### SQL/Database (PostgreSQL) — 4 weeks
- Week 1: Relational model, PostgreSQL setup, CRUD
- Week 2: JOINs, aggregation, CTE, window functions
- Week 3: Indexes, normalization, transactions, EXPLAIN
- Week 4: psycopg2/SQLAlchemy with Python, migrations, capstone project

### Linux In-Depth — 4 weeks
- Week 1: Filesystem, processes, shell scripting
- Week 2: systemd, users/PAM, storage
- Week 3: TCP/IP, tcpdump/iptables, container networking internals
- Week 4: Performance profiling, cgroup v2, strace/lsof/perf, troubleshooting

## Format (matches existing plans)
- ASCII roadmap diagram
- Day-by-day breakdown with topic tables + exercises
- Weekly capstone exercises
- Milestone checklist
- Recommended resources
