# 📘 SQL Server Knowledge Base

Kho ghi chú cá nhân về **SQL Server internals, performance tuning, indexing, execution plans, locking, TempDB, backup/restore**  
Dùng cho học tập, ôn tập và áp dụng thực tế trong môi trường production.

> 🎯 Mục tiêu: Hiểu **bản chất SQL Server vận hành thế nào**, không chỉ “viết query chạy được”.

---

## 🧠 Nội dung chính

### 🔹 Execution Plans & Query Optimizer
- [Execution Plan – Properties Reference](execution-plans/execution-plan-properties.md)
- [Execution Plan Reading Checklist](execution-plans/README.md)
- Cardinality Estimation *(coming soon)*

---

### 🔹 Indexing & Query Performance
- [Index Design Checklist](indexing/index-design-checklist.md)
- [Filtered Index](indexing/filtered-index.md)
- Composite Index & Ordering *(coming soon)*
- Key Lookup & Covering Index *(coming soon)*

---

### 🔹 Locking, Latching & Concurrency
- [Locks vs Latches](locking/locks-vs-latches.md)
- [Deadlocks – Causes & Patterns](locking/deadlocks.md)
- Isolation Levels & RCSI *(coming soon)*

---

### 🔹 TempDB Internals
- [How SQL Server Uses TempDB](tempdb/tempdb-usage.md)
- TempDB Contention & Optimization *(coming soon)*

---

### 🔹 Backup & Restore
- [Backup Types: Full, Diff, Log, Partial](backup-restore/backup-types.md)
- Restore Strategies *(coming soon)*

---

### 🔹 Storage, IO & Architecture
- RAID Levels & SAN vs Local Disk *(coming soon)*
- IO Patterns in SQL Server *(coming soon)*

---

## 📂 Repository Structure
```text
sql-server-knowledge-base/
│
├── README.md
│
├── execution-plans/
│ ├── README.md
│ └── execution-plan-properties.md
│
├── indexing/
│ ├── README.md
│ └── index-design-checklist.md
│
├── locking/
│ ├── README.md
│ └── locks-vs-latches.md
│
├── tempdb/
│ ├── README.md
│ └── tempdb-usage.md
│
└── backup-restore/
├── README.md
└── backup-types.md
```

---

## 🧩 Cách sử dụng repo này

- Mỗi **folder** là một chủ đề lớn
- Mỗi **file `.md`** là một chủ đề cụ thể
- `README.md` trong folder đóng vai trò **landing page**
- Dùng như:
  - Checklist khi debug production
  - Tài liệu ôn tập
  - Knowledge base cá nhân / team

---

## 🛠️ Công cụ & Nguồn tham khảo

- SQL Server Management Studio (SSMS)
- GitHub Markdown
- Blog: Paul White, Brent Ozar, Erik Darling
- Microsoft Learn (SQL Server Docs)

---

## 📌 Ghi chú

> Nội dung trong repo này mang tính **ghi chú kỹ thuật**, không phải tutorial cơ bản.  
> Ưu tiên **bản chất – nguyên nhân – trade-off**.

---

## 🚀 Kế hoạch mở rộng (Roadmap)

- [ ] Cardinality Estimation Deep Dive
- [ ] Parameter Sniffing Patterns
- [ ] TempDB Internals & Allocation Maps
- [ ] Wait Stats & Performance Troubleshooting
- [ ] Always On & HA/DR Basics

---

**Author:** Hai Le  
**Focus:** SQL Server Internals · Performance · Architecture


