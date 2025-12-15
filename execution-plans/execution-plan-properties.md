# SQL Server Execution Plan – Properties Reference (DBA Level)

Tài liệu này dùng để **đọc, phân tích và hiểu chi tiết các Properties trong SQL Server Execution Plan**  
(phần hiển thị khi hover operator hoặc mở **Properties (F4)** trong SSMS).

> 🎯 Mục tiêu: Hiểu **vì sao Optimizer chọn plan này**, không chỉ “plan chạy nhanh hay chậm”.

---

## 1. Cost & Estimation (Cốt lõi của CBO)

| Property | Ý nghĩa | Khi nào quan trọng | Khi nào bỏ qua |
|--------|--------|------------------|---------------|
| **Estimated Operator Cost** | Cost ước tính của riêng operator | So sánh operator trong cùng plan | Không dùng so runtime |
| **Estimated Subtree Cost** | Tổng cost của operator + toàn bộ child | So sánh **toàn plan** | Không so giữa server |
| **Estimated I/O Cost** | I/O cost ước tính | Tìm IO-heavy operator | Cache nóng làm sai |
| **Estimated CPU Cost** | CPU cost ước tính | CPU bottleneck | Khi IO chiếm ưu thế |
| **Cost %** | % cost so với toàn plan | Tìm bottleneck nhanh | Không tuyệt đối |

📌 **Rule vàng**  
> Subtree Cost của root operator = tổng cost của execution plan

---

## 2. Rows & Cardinality (Nguyên nhân ~80% lỗi performance)

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Estimated Number of Rows** | Optimizer dự đoán | Luôn so với actual |
| **Actual Number of Rows** | Row thực tế | Phát hiện estimation sai |
| **Estimated Rows Per Execution** | Row mỗi lần chạy | Nested Loop |
| **Actual Rows Per Execution** | Thực tế | Rebind / Rewind |

🚨 **Red flag lớn nhất**
- Estimated = 1
- Actual = 100,000

➡ Gây:
- Plan sai
- Memory grant sai
- Join strategy sai

---

## 3. Memory & TempDB

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Memory Grant** | RAM cấp cho query | Sort / Hash |
| **Granted Memory** | RAM thực được cấp | Memory pressure |
| **Used Memory** | RAM thực dùng | Over / under grant |
| **Spill Level** | Spill TempDB (1/2/3) | Performance rất xấu |
| **Warnings** | Spill / Missing Index | Luôn kiểm tra |

⚠️ **Spill = performance killer**

---

## 4. Join & Loop Mechanics

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Logical Operation** | Logic join (Inner / Left) | Đúng nghiệp vụ |
| **Physical Operation** | Nested / Hash / Merge | Performance |
| **Rebinds** | Outer input đổi | Nested Loop |
| **Rewinds** | Reuse inner input | Tốt |
| **Estimated Rebinds** | Dự đoán | Sai → loop explosion |

🚨 **Nested Loop + Rebinds cao** = cảnh báo đỏ

---

## 5. Access Method (Scan / Seek)

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Index Name** | Index được dùng | Đúng index chưa |
| **Seek Predicate** | Predicate dùng để seek | Phải có |
| **Residual Predicate** | Filter sau khi đọc | Dấu hiệu index thiếu |
| **Ordered** | Output đã sorted | Tránh SORT |
| **Scan Direction** | Forward / Backward | ORDER BY |

📌 Residual Predicate nhiều → index chưa tối ưu

---

## 6. Parallelism

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Parallel** | Có song song không | CPU usage |
| **Estimated DOP** | DOP dự kiến | CPU pressure |
| **Actual DOP** | DOP thực tế | Throttling |
| **Wait Type** | CXPACKET / CXCONSUMER | Skew |

📌 Parallel ≠ luôn tốt

---

## 7. Plan Cache & Compilation

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Cached Plan Size** | Size plan trong cache | Cache pressure |
| **Compile Time** | Thời gian compile | Ad-hoc nhiều |
| **Compile CPU** | CPU cho compile | Parameter sniffing |
| **Optimization Level** | Full / Trivial | Plan quality |

📌 Plan lớn + nhiều ad-hoc → cache churn

---

## 8. Sort & Ordering

| Property | Ý nghĩa | Khi nào quan trọng |
|--------|--------|------------------|
| **Sort Warnings** | Spill | Luôn xem |
| **Top-N Sort** | Partial sort | TOP query |
| **Order By Columns** | Column sort | Index match |
| **Distinct Sort** | Dedup | Cost cao |

📌 SORT là **blocking operator**

---

## 9. Advanced / Internal (DBA Level)

| Property | Ý nghĩa | Ghi chú |
|--------|--------|--------|
| **NodeId** | ID operator | Debug XML |
| **Predicate** | Filter logic | Debug |
| **Defined Values** | Column output | Debug |
| **Estimated Execution Mode** | Row / Batch | Columnstore |

---

## 10. Execution Plan Reading Checklist (Thực tế)
- Estimated Rows vs Actual Rows (QUAN TRỌNG NHẤT)
- Operator có cost % cao
- Có SORT / HASH / KEY LOOKUP không
- Memory Grant & Spill
- Seek Predicate vs Residual Predicate
- Nested Loop + Rebind
- Parallelism & skew

---

## Key Takeaway

> **Execution plan properties không phải là performance metric,  
> mà là manh mối để hiểu quyết định của Optimizer.  
> Luôn đọc trong ngữ cảnh: rows → memory → join → sort.**

---

### Recommended Usage
- Dùng như **checklist khi debug slow query**
- Gắn vào **GitHub repo / Wiki nội bộ**
- Dùng khi **review index / query rewrite**

