# Lộ trình 19 tuần — iOS Developer phỏng vấn ở mức 2 năm kinh nghiệm

> Bắt đầu: **Thứ Hai 09/11/2026** · Kết thúc: **Chủ Nhật 21/03/2027** (≈ 14/2 âm lịch)
> Tết Đinh Mùi: **Thứ Bảy 06/02/2027** → tuần 13 và 14 là tuần nhẹ, chỉ ôn lại.

Kế hoạch này viết cho người **đã từng làm iOS nhưng gần như quên hết**. Vì vậy mỗi tuần đều có:
phần giải thích kiến thức từ gốc, code ví dụ chạy được, bài tập, câu hỏi phỏng vấn kèm đáp án, và checklist tự kiểm tra.

---

## 1. Nguyên tắc học (đọc kỹ trước khi bắt đầu)

### 1.1. Quy tắc dùng AI trong 19 tuần này
| Được làm | Không được làm |
|---|---|
| Hỏi AI **"tại sao"** sau khi bạn đã tự thử ít nhất 20 phút | Nhờ AI viết code bài tập |
| Nhờ AI **đóng vai người phỏng vấn**, hỏi xoáy bạn | Bật Copilot / autocomplete AI khi làm bài tập |
| Nhờ AI chấm câu trả lời bạn **đã tự viết ra** | Đọc đáp án trước khi tự trả lời |
| Nhờ AI giải thích một đoạn lỗi compiler | Copy-paste code mà không gõ lại |

> Tạo một **Xcode profile riêng** hoặc tắt "Predictive Code Completion" trong Xcode → Settings → Text Editing khi học.

### 1.2. Phương pháp học mỗi chủ đề (vòng 4 bước)
1. **Đọc** phần "Kiến thức cốt lõi" trong file tuần (15–20 phút).
2. **Gõ lại** code ví dụ vào Playground hoặc file `.swift`, **sửa thử** để xem nó gãy ở đâu.
3. **Làm bài tập** — không nhìn lời giải.
4. **Nói lại** (Feynman): mở voice memo, giải thích chủ đề trong 2 phút như đang trả lời phỏng vấn. Nghe lại, chỗ nào ấp úng = chỗ chưa hiểu.

### 1.3. Cách chạy code nhanh không cần mở Xcode
Máy bạn đã có Swift 6.3. Tạo file rồi chạy:
```bash
swift week01/day1.swift
```
Với code UIKit/SwiftUI thì dùng Xcode Playground hoặc project trong `project/`.

---

## 2. Quỹ thời gian

Giả định bạn đi làm full-time:

| Ngày | Thời lượng | Nội dung |
|---|---|---|
| Thứ 2 – Thứ 6 | **1,5–2 giờ/tối** | 60–80' lý thuyết + bài tập, 30' DSA |
| Thứ 7 | **3–4 giờ** | Side project + bài tập lớn trong tuần |
| Chủ Nhật | **2 giờ** | Ôn tập, tự quiz, nói lại, checklist. Nghỉ ngơi phần còn lại |

Tổng ≈ **13–15 giờ/tuần**, ~260 giờ cho cả lộ trình. Nếu tuần nào bận, **cắt DSA và side project trước, giữ lý thuyết + câu hỏi phỏng vấn**.

---

## 3. Lịch tổng thể

| Tuần | Ngày | Chủ đề | File |
|---|---|---|---|
| **GIAI ĐOẠN 1 — SWIFT CORE** ||||
| 1 | 09/11 – 15/11 | Swift nền tảng: kiểu dữ liệu, Optional, collection, hàm, control flow | [week01](weeks/week01-swift-basics.md) |
| 2 | 16/11 – 22/11 | Struct, Class, Enum · value vs reference · copy-on-write · property · init | [week02](weeks/week02-types-value-reference.md) |
| 3 | 23/11 – 29/11 | Closure · higher-order function · Protocol · Extension · POP | [week03](weeks/week03-closures-protocols.md) |
| 4 | 30/11 – 06/12 | Generics · associatedtype · `some`/`any` · Error handling · Property wrapper · Access control · Codable | [week04](weeks/week04-generics-advanced-swift.md) |
| **GIAI ĐOẠN 2 — BỘ NHỚ & CONCURRENCY** ||||
| 5 | 07/12 – 13/12 | ARC · retain cycle · weak/unowned · Memory Graph · Instruments | [week05](weeks/week05-memory-arc.md) |
| 6 | 14/12 – 20/12 | Thread · GCD · OperationQueue · race condition · deadlock · thread-safety | [week06](weeks/week06-gcd-threading.md) |
| 7 | 21/12 – 27/12 | async/await · Task · TaskGroup · actor · @MainActor | [week07](weeks/week07-swift-concurrency.md) |
| 8 | 28/12 – 03/01 | Sendable · Swift 6 strict concurrency · AsyncSequence · Combine cơ bản | [week08](weeks/week08-concurrency-advanced-combine.md) |
| **GIAI ĐOẠN 3 — UI** ||||
| 9 | 04/01 – 10/01 | App lifecycle · VC lifecycle · RunLoop · responder chain · render pipeline | [week09](weeks/week09-uikit-lifecycle.md) |
| 10 | 11/01 – 17/01 | Auto Layout · UITableView/UICollectionView · reuse · diffable · scroll performance | [week10](weeks/week10-uikit-layout-lists.md) |
| 11 | 18/01 – 24/01 | SwiftUI: View là gì · state management · @Observable · data flow | [week11](weeks/week11-swiftui-fundamentals.md) |
| 12 | 25/01 – 31/01 | SwiftUI nâng cao: identity, re-render, navigation, interop UIKit | [week12](weeks/week12-swiftui-advanced.md) |
| **TẾT — TUẦN NHẸ** ||||
| 13 | 01/02 – 07/02 | 🧧 Ôn tập Giai đoạn 1 + 2 bằng flashcard (≈ 30–45'/ngày) | [week13](weeks/week13-tet-review-1.md) |
| 14 | 08/02 – 14/02 | 🧧 Ôn tập Giai đoạn 3 + viết CV bản nháp | [week14](weeks/week14-tet-review-2-cv.md) |
| **GIAI ĐOẠN 4 — KIẾN TRÚC & HỆ THỐNG** ||||
| 15 | 15/02 – 21/02 | MVC/MVVM/Clean/VIPER/TCA · SOLID · DI · Coordinator | [week15](weeks/week15-architecture.md) |
| 16 | 22/02 – 28/02 | Networking · Persistence · Unit test & mock | [week16](weeks/week16-networking-persistence-testing.md) |
| 17 | 01/03 – 07/03 | Push · background · Keychain/security · performance · build/release/CI · Git | [week17](weeks/week17-system-performance-release.md) |
| **GIAI ĐOẠN 5 — VỀ ĐÍCH** ||||
| 18 | 08/03 – 14/03 | Mobile system design · kể chuyện dự án (STAR) · mock interview | [week18](weeks/week18-system-design-mock.md) |
| 19 | 15/03 – 21/03 | Mock cuối · phỏng vấn thật · deal lương · checklist ngày phỏng vấn | [week19](weeks/week19-final-interview.md) |

**Chạy song song suốt 19 tuần:**
- 🧮 [DSA — lộ trình thuật toán theo tuần](tracks/dsa-plan.md)
- 📱 [Side project — app tự xây từ đầu, mỗi tuần một milestone](tracks/side-project.md)

---

## 4. Mốc kiểm tra (milestone)

| Mốc | Ngày | Bạn phải làm được |
|---|---|---|
| ✅ M1 | 06/12 | Giải thích value vs reference, closure capture, protocol vs class inheritance, generics **không cần nhìn tài liệu** |
| ✅ M2 | 03/01 | Tự tìm và sửa một retain cycle bằng Memory Graph; viết code async/await + actor không lỗi compile Swift 6 |
| ✅ M3 | 31/01 | Dựng một màn hình list + detail bằng cả UIKit và SwiftUI không tra cứu |
| ✅ M4 | 14/02 | CV bản nháp xong; ôn xong flashcard Giai đoạn 1–3 |
| ✅ M5 | 07/03 | Side project chạy được: network + cache + test + kiến trúc rõ ràng, đẩy lên GitHub |
| ✅ M6 | 21/03 | Đã làm ≥ 4 buổi mock interview, ≥ 2 buổi phỏng vấn thật |

> **Gợi ý nghỉ việc:** chỉ nộp đơn nghỉ **sau khi đã có offer bằng văn bản**. Bắt đầu rải CV từ tuần 17–18.

---

## 5. Cấu trúc thư mục

```
Interview/
├── README.md               ← file này
├── weeks/                  ← 19 file, mỗi tuần một file
├── tracks/
│   ├── dsa-plan.md         ← thuật toán theo tuần
│   └── side-project.md     ← app thực hành theo tuần
└── practice/               ← bạn tự tạo: code bài tập của mình (week01/, week02/, ...)
```

## 6. Cách dùng mỗi file tuần

Mỗi file tuần có cùng cấu trúc:
1. **Mục tiêu tuần** — cuối tuần bạn phải nói/làm được gì
2. **Lịch từng ngày** — Thứ 2 → Chủ Nhật, làm gì, bao lâu
3. **Kiến thức cốt lõi** — giải thích từ gốc + code ví dụ
4. **Bẫy hay gặp** — những thứ người phỏng vấn thích hỏi xoáy
5. **Bài tập** — có đề, có gợi ý, lời giải giấu trong `<details>`
6. **Câu hỏi phỏng vấn** — đáp án mẫu giấu trong `<details>`, **tự trả lời trước khi mở**
7. **Checklist cuối tuần** — tick hết mới sang tuần sau
8. **Tài liệu đọc thêm**

> Nếu một tuần chưa tick đủ ≥ 80% checklist, đừng ngại **lùi 2–3 ngày**. Tuần 13–14 (Tết) là vùng đệm để bù.
