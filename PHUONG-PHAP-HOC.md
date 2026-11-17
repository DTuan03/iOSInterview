# Phương pháp học — học thế nào cho "chuẩn"

> Đọc file này **trước khi** mở Tuần 1. Tài liệu học gì đã có trong `weeks/`; file này trả lời câu hỏi **học như thế nào** để nhớ lâu và trả lời được khi phỏng vấn.

---

## 1. Vì sao học mãi không vào

Đọc tài liệu, hoặc đọc AI giải thích, thì thấy hiểu hết. Nhưng gập lại là không nói ra được. Đó là **ảo giác "mình hiểu rồi"**:

| Khi đọc | Khi phỏng vấn |
|---|---|
| Bạn **nhận ra** kiến thức vì nó đang ở trước mặt | Bạn phải **tự gọi ra** kiến thức, không có gì trước mặt |
| Dễ, cảm giác tiến bộ nhanh | Khó, và chỉ làm được nếu đã **tập gọi ra** nhiều lần |

Nguyên tắc số 1:

> **Học = tự lôi kiến thức ra khỏi đầu, không phải nạp thêm vào đầu.**
> Ít nhất **một nửa** thời gian học phải là lúc **gập tài liệu lại** rồi tự nhớ, tự viết, tự nói.

Nhớ lại thấy khó, phải vắt óc = **đang học đúng**. Cảm giác trơn tru dễ chịu khi đọc lại = thường là đang tự lừa mình.

---

## 2. Khung một buổi học 90 phút (T2–T6)

Mỗi tối làm **đúng** khung này. Đừng tự nghĩ cách mới mỗi ngày, vì não sẽ mất sức vào việc quyết định thay vì học.

| Phút | Bước | Làm gì | Quy tắc |
|---|---|---|---|
| **0–5** | 🧠 **Nhớ lại hôm qua** | Lấy giấy trắng, viết ra mọi thứ nhớ được về bài hôm qua | **Không mở file.** Viết xong mới mở ra so, phần nào quên thì gạch chân, mai nhớ lại tiếp |
| **5–30** | 📖 **Đọc 1 mục (§)** | Đọc **một** mục trong file tuần | Đọc xong *mỗi đoạn nhỏ* thì cuộn đi chỗ khác, tự viết 2–3 dòng bằng lời mình. Không viết được thì đọc lại đoạn đó |
| **30–60** | ⌨️ **Gõ code và phá code** | Gõ lại ví dụ, **không copy-paste** | **Đoán output trước khi chạy.** Sau đó cố tình sửa sai (bỏ `weak`, đổi `struct` → `class`, xoá `guard`...) và đoán chuyện gì xảy ra. Đoán sai = vừa học được thứ thật |
| **60–80** | 🏋️ **Bài tập / DSA** | Làm bài tập của ngày hoặc bài DSA | Bấm giờ. **Bí 20 phút** mới được xem gợi ý; không xem lời giải ngay |
| **80–90** | 📝 **Đóng buổi** | Viết vào sổ: 3 điều học được, 1 điều còn mơ hồ. Tạo 3–5 flashcard | Điều mơ hồ → đầu buổi mai giải quyết trước |

> ⚠️ **Phần quan trọng nhất là 5 phút đầu và 10 phút cuối.** Hôm nào thiếu giờ thì cắt phần đọc, **không cắt hai phần này**.

### Hôm đó quá mệt?
Làm **bản 30 phút**: 5' nhớ lại → 15' Anki → 10' gõ lại 1 ví dụ code. Giữ được nhịp quan trọng hơn học nhiều.

---

## 3. Ôn theo lịch giãn cách — chống quên

Học xong một thứ mà không ôn thì 1 tuần sau quên khoảng 70%. Cách rẻ nhất để chống quên là ôn lại vào:

```
Ngày 1 → ngày 3 → ngày 7 → ngày 21 → ngày 60
```

Không cần tự nhớ lịch này. Dùng **Anki**, nó tự xếp lịch.

### Cài đặt Anki
1. Tải Anki tại [apps.ankiweb.net](https://apps.ankiweb.net) (miễn phí trên Mac, Windows, Android; bản web AnkiWeb dùng trên iPhone).
2. Tạo tài khoản AnkiWeb để đồng bộ máy tính ↔ điện thoại.
3. Tạo deck theo giai đoạn: `iOS::1-Swift`, `iOS::2-Memory-Concurrency`, `iOS::3-UI`, `iOS::4-Architecture`, `iOS::DSA`.
4. Mỗi ngày mở **10–15 phút** (trên xe bus, trước khi ngủ...). **Không bỏ ngày nào**, vì thẻ dồn lại rất nhanh.

### Cách viết thẻ tốt

| ❌ Thẻ tệ | ✅ Thẻ tốt |
|---|---|
| Mặt trước: "ARC" · Mặt sau: một đoạn văn dài | Mặt trước: "Khi nào dùng `unowned` thay `weak`?" · Mặt sau: 1–2 câu |
| "Liệt kê 10 operator của Combine" | "`debounce` khác `throttle` ở điểm nào?" |
| Chép nguyên định nghĩa từ tài liệu | Viết bằng lời của mình |

Các kiểu thẻ nên làm:
- **"Tại sao…?"**: Tại sao String không có `s[0]`?
- **"Nếu… thì sao?"**: Nếu bỏ `[weak self]` trong closure này thì sao?
- **"Output là gì?"**: dán đoạn code 5–8 dòng, mặt sau là output kèm giải thích ngắn.
- **"So sánh A vs B"**: `@StateObject` vs `@ObservedObject`.

Nguồn thẻ có sẵn: mục 🎤 **Câu hỏi phỏng vấn** của mỗi tuần, và [Tuần 13](weeks/week13-tet-review-1.md), [Tuần 14](weeks/week14-tet-review-2-cv.md) đã có khoảng 150 flashcard.

> Quy tắc: **chỉ tạo thẻ cho thứ bạn đã hiểu.** Anki giúp *nhớ*, không giúp *hiểu*.

---

## 4. Ghi chép đúng cách

- **Không chép lại tài liệu.** Tài liệu đã có trong repo.
- Mỗi chủ đề ghi đúng **4 dòng, bằng lời của bạn**:

```markdown
## Optional
1. Là gì: một enum có 2 case .none / .some(value), cho phép "không có giá trị" một cách an toàn.
2. Vì sao có: để compiler bắt mình xử lý nil, tránh crash null pointer như Obj-C.
3. Ví dụ: guard let user else { return }
4. Bẫy: array.first trả Optional nhưng array[0] thì crash nếu rỗng.
```

- Viết tay, Obsidian hay Notion đều được, miễn là **tự gõ**. Gợi ý: tạo `practice/notes.md`, mỗi tuần thêm một mục.

---

## 5. Cách dùng AI từ giờ

AI không xấu, chỉ cần đổi vai cho nó: **gia sư hỏi bạn**, không phải **người làm bài hộ bạn**.

| ✅ Được | ❌ Không |
|---|---|
| Hỏi gợi ý **sau khi** tự thử 20 phút | Nhờ viết code bài tập |
| Nhờ đóng vai người phỏng vấn | Bật autocomplete AI khi học |
| Nhờ chấm câu trả lời bạn **đã tự viết** | Đọc giải thích của AI thay vì tự nhớ |

> Tắt AI completion trong Xcode: **Settings → Text Editing → Editing → bỏ chọn "Predictive code completion"** (tên mục có thể khác nhẹ giữa các bản Xcode).

### Câu lệnh mẫu (copy dùng luôn)

**Khi bí bài tập** (chỉ sau khi đã tự thử 20 phút):
```
Tôi đang học Swift, đừng đưa lời giải. Đây là code của tôi và chỗ tôi bí: [...].
Chỉ cho tôi 1 gợi ý nhỏ nhất để tôi tự nghĩ tiếp.
```

**Khi tự kiểm tra cuối buổi:**
```
Đóng vai người phỏng vấn iOS. Hỏi tôi lần lượt 5 câu về [chủ đề hôm nay],
từng câu một, chờ tôi trả lời rồi mới hỏi tiếp. Sau mỗi câu, chấm 1–10,
chỉ ra chỗ sai/thiếu và hỏi xoáy thêm 1 câu.
```

**Khi đọc không hiểu:**
```
Giải thích [khái niệm] như cho người mới, kèm 1 ví dụ đời thường.
Sau đó hỏi ngược tôi 2 câu để kiểm tra xem tôi hiểu thật chưa.
```

**Khi review code bài tập của mình:**
```
Review đoạn code Swift này như senior iOS dev review PR của junior.
Chỉ ra lỗi, chỗ chưa idiomatic, edge case bị bỏ sót. Không viết lại toàn bộ code,
chỉ nêu vấn đề và gợi ý hướng sửa.
```

---

## 6. Làm sao biết mình đã "học xong" một chủ đề

Thấy hiểu khi đọc thì chưa đủ. Một chủ đề chỉ tính là xong khi qua **cả 3 bài thử**:

| # | Bài thử | Cách làm |
|---|---|---|
| 1 | 📄 **Giấy trắng** | Giải thích được trong 2 phút mà không nhìn gì. Ghi âm rồi nghe lại: chỗ nào ấp úng = chưa hiểu |
| 2 | ⌨️ **Code từ số 0** | Viết được ví dụ cốt lõi trong file trống, không autocomplete AI, không tra cứu |
| 3 | ❓ **"Tại sao" và "nếu… thì sao"** | Trả lời được câu xoáy, ví dụ "Tại sao String không có `s[0]`?", "Nếu bỏ `[weak self]` thì sao?" |

Đây cũng chính là cách người phỏng vấn kiểm tra bạn.

---

## 7. Nhịp một tuần

| Ngày | Làm gì |
|---|---|
| **T2–T6** | Mỗi tối một buổi 90 phút theo [khung ở mục 2](#2-khung-một-buổi-học-90-phút-t2t6) + 10–15 phút Anki |
| **T7** | Bài tập lớn + side project. Đây là lúc ghép các mảnh kiến thức lại thành thứ chạy được |
| **CN** | **Chỉ tự kiểm tra, không học mới:** tự trả lời câu 🎤 trên giấy rồi mới mở đáp án → ghi âm 2–3 câu → tick checklist → chỗ nào chưa đạt thì ghi vào "cần học lại" cho đầu tuần sau |

### Bảng theo dõi tuần (chép vào `practice/notes.md`)

```markdown
## Tuần N
| Ngày | Buổi 90' | Anki | DSA | Ghi chú mơ hồ |
|---|---|---|---|---|
| T2 | ✅ | ✅ | ✅ | ... |
| T3 | | | | |
...
Câu 🎤 tự trả lời đúng: __ / __
Cần học lại: ...
```

---

## 8. Khi bị chậm tiến độ

- **Đừng cố đuổi theo lịch bằng cách đọc lướt.** Học chắc 1 tuần còn hơn lướt 3 tuần.
- Lùi cả lộ trình 1 tuần là bình thường. Chỗ bù: **Tuần 13–14 (Tết) gộp 2 tuần ôn thành 1**.
- Lùi quá 2 tuần → cắt bớt **DSA và side project trước**, giữ phần lý thuyết và câu hỏi phỏng vấn.
- Tuần đầu sẽ chậm và hơi nản, vì não đang tập lại thói quen tự nghĩ thay vì hỏi AI. Sang tuần 3–4 sẽ nhanh hơn rõ.

---

## 9. Việc cần làm trước buổi học đầu tiên

- [ ] Cài Anki, tạo tài khoản AnkiWeb, tạo deck `iOS::1-Swift`
- [ ] Tạo thư mục `practice/week01/` và file `practice/notes.md`
- [ ] Tắt AI completion trong Xcode
- [ ] Chuẩn bị giấy trắng hoặc sổ tay cho bước "Nhớ lại hôm qua"
- [ ] Mở [Tuần 1](weeks/week01-swift-basics.md), học §1 theo đúng khung 90 phút
