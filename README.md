# 📘 SQL Server Knowledge Base

A personal knowledge base for **SQL Server internals, performance tuning, indexing, execution plans, locking, TempDB, and backup/restore**.

> 🎯 Goal: Understand **how SQL Server really works**, not just how to write queries that run.

---

## 🧠 Main Topics

### 🔹 Execution Plans & Query Optimizer
- [Execution Plan – Properties Reference](execution-plans/execution-plan-properties.md)
- [Execution Plans Overview](execution-plans/README.md)
- Cardinality Estimation *(coming soon)*

---

### 🔹 Indexing & Query Performance
- [Index Design Checklist](indexing/index-design-checklist.md)
- Filtered Index *(coming soon)*
- Composite Index & Ordering *(coming soon)*
- Key Lookup & Covering Index *(coming soon)*

---

### 🔹 Locking, Latching & Concurrency
- Locks vs Latches *(coming soon)*
- Deadlocks – Causes & Patterns *(coming soon)*
- Isolation Levels & RCSI *(coming soon)*

---

### 🔹 TempDB Internals
- TempDB Usage & Internals *(coming soon)*
- TempDB Contention & Optimization *(coming soon)*

---

### 🔹 Backup & Restore
- Backup Types: Full, Differential, Log, Partial *(coming soon)*
- Restore Strategies *(coming soon)*

---

## 📂 Repository Structure

```text
sql-server-knowledge-base/
├── README.md
│
├── execution-plans/
│   ├── README.md
│   └── execution-plan-properties.md
│
├── indexing/
│   ├── README.md
│   └── index-design-checklist.md
│
├── locking/
│   ├── README.md
│   └── locks-vs-latches.md
│
├── tempdb/
│   ├── README.md
│   └── tempdb-usage.md
│
└── backup-restore/
    ├── README.md
    └── backup-types.md

```

---

## 🧩 How to Use This Repository

- Each **folder** represents a major SQL Server topic
- Each **.md file** focuses on a specific subject
- Folder-level README.md files act as **landing page**
- Designed to be used as:
  - A production troubleshooting checklist
  - A personal reference
  - A shared team knowledge base

---

## 🛠️ Tools & References

- SQL Server Management Studio (SSMS)
- GitHub Markdown
- Blog: Paul White, Brent Ozar, Erik Darling
- Microsoft Learn (SQL Server Docs)

---

## 🚀 Roadmap

- [ ] Cardinality Estimation Deep Dive
- [ ] Parameter Sniffing Patterns
- [ ] TempDB Internals & Allocation Maps
- [ ] Wait Stats & Performance Troubleshooting
- [ ] Always On & HA/DR Basics

---

**Author:** Hai Le  
**Focus:** SQL Server Internals · Performance · Architecture


