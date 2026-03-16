# DevOps Cookbooks Index

A curated collection of copy-paste ready cookbooks for common DevOps tasks.
Each cookbook is:

- Self-contained
- Opinionated with sane defaults
- Written in a step-by-step format
- Safe to version-control and review via pull requests

---

## Cookbook Menu

Tip: Start with the Jenkins cookbook if you're setting up CI/CD locally on macOS.

### CI/CD

| Tool    | Scenario                                             | Cookbook Link                                                                 |
|--------:|------------------------------------------------------|-------------------------------------------------------------------------------|
| Jenkins | Run Jenkins using Docker                    | [Jenkins Cookbook](jenkins.MD)                                               |
| Jenkins | Jenkins + Docker + Flask (3 environments)   | [Jenkins + Docker + Flask Cookbook](https://github.com/bilouro/cookbook/blob/main/Jenkins_Docker_Flask.MD) |

### Databases

| Tool       | Scenario                                                    | Cookbook Link                                    |
|-----------:|-------------------------------------------------------------|--------------------------------------------------|
| PostgreSQL | Run PostgreSQL on macOS using Docker                       | [PostgreSQL Cookbook](postgresql.MD)             |
| Metabase   | Run Metabase with PostgreSQL using Docker (incl. H2 migration) | [Metabase Cookbook](metabase.MD)            |

---

## Repository Layout (Suggested)

Although this repo can grow organically, a clean structure makes it easier to scale:

```text
.
├─ README.md                 # This index file
├─ jenkins.MD                # Jenkins on macOS with Docker cookbook
├─ Jenkins_Docker_Flask.MD   # Jenkins + Docker + Flask (3 environments) cookbook
├─ metabase.MD               # Metabase + PostgreSQL on Docker cookbook
├─ <tool>.MD                 # Future cookbooks (e.g., gitlab-ci.MD, argocd.MD, etc.)
└─ assets/                   # Optional: diagrams, screenshots, shared snippets
```
