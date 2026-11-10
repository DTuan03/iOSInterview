# Tuần 13 — 🧧 Tết: Ôn tập Giai đoạn 1 + 2 bằng flashcard (01/02 – 07/02/2027)

> **Tuần nhẹ** · Thời lượng: ~3 giờ cả tuần (30–45 phút/ngày, nghỉ hẳn từ 29 Tết)
> Tết Đinh Mùi: **Thứ Bảy 06/02/2027**. 29 Tết là **Thứ Sáu 05/02**.
> Tuần này **không học kiến thức mới**. Mục tiêu là *kéo lại* những gì đã học ở tuần 1–8 bằng **active recall** (tự nhớ ra trước khi xem đáp án) — cách ôn hiệu quả hơn đọc lại nhiều lần. Ôn ít nhưng đều, rồi nghỉ Tết thật sự.

## 🎯 Mục tiêu tuần

- [ ] Đi hết **~95 flashcard** tuần 1–8, đánh dấu thẻ trả lời sai
- [ ] Đi lại các thẻ sai **lần 2** (cách lần 1 ít nhất 1 ngày)
- [ ] Làm **quiz đoán output** 10 câu, đúng ≥ 8/10
- [ ] Nói trôi chảy **Top 20 must-know** (mỗi câu ≤ 1 phút)
- [ ] DSA: làm lại 3 bài cũ (xem [dsa-plan](../tracks/dsa-plan.md))
- [ ] Nghỉ Tết trọn vẹn 05/02 – 07/02, không áy náy 🎉

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 01/02** | 45' | Flashcard **Tuần 1–2** (~24 thẻ). Quiz câu 1–3. DSA làm lại: Two Sum | Danh sách thẻ sai |
| **T3 02/02** | 45' | Flashcard **Tuần 3–4** (~24 thẻ). Quiz câu 4–5, 10. DSA làm lại: Valid Parentheses | Danh sách thẻ sai |
| **T4 03/02** | 45' | Flashcard **Tuần 5–6** (~24 thẻ). Quiz câu 6–7. DSA làm lại: Reverse Linked List | Danh sách thẻ sai |
| **T5 04/02** | 45' | Flashcard **Tuần 7–8** (~24 thẻ). Quiz câu 8–9. Đi lại **toàn bộ thẻ sai** từ T2–T4 | Danh sách thẻ sai lần 2 |
| **T6 05/02** (29 Tết) | — | 🧧 **Nghỉ.** Tuỳ chọn: 15' đọc lướt Top 20 trên điện thoại | — |
| **T7 06/02** (Mùng 1) | — | 🧧 **Nghỉ** | — |
| **CN 07/02** (Mùng 2) | — | 🧧 **Nghỉ.** Tuỳ chọn: 15' tick checklist | — |

> **Cách dùng flashcard:** đọc câu hỏi → **nói to** câu trả lời (hoặc viết 1–2 dòng) → mới mở `<details>`. Đúng ý chính = ✅. Sai hoặc ấp úng = ghi số thẻ vào notes. Không "nhìn qua là nhớ" — đó là ảo giác hiểu bài.
> Muốn ôn trên điện thoại khi về quê: copy các thẻ sai sang Anki / Notes.

---

## 🃏 Flashcards

### Tuần 1 — Swift nền tảng

**1.** Optional được cài đặt như thế nào?
<details><summary>Đáp án</summary>Là `enum Optional<Wrapped> { case none; case some(Wrapped) }`. `nil` là `.none`, `String?` là `Optional<String>`.</details>

**2.** Kể 5 cách unwrap Optional.
<details><summary>Đáp án</summary>`if let`, `guard let`, `??`, optional chaining `?.`, force unwrap `!` (thêm: `switch`/`if case let x?`, `map`/`flatMap`).</details>

**3.** Khi nào dùng `guard let` thay `if let`?
<details><summary>Đáp án</summary>Khi giá trị là điều kiện tiên quyết để hàm chạy tiếp: thoát sớm, tránh lồng nhau, biến unwrap dùng được ở phần còn lại của scope. Nhánh `else` bắt buộc thoát scope.</details>

**4.** `let` với class có làm object bất biến không?
<details><summary>Đáp án</summary>Không. `let` chỉ khoá tham chiếu; vẫn sửa được thuộc tính `var` của object. Với struct, `let` khoá toàn bộ giá trị.</details>

**5.** Vì sao không có `str[0]`?
<details><summary>Đáp án</summary>`Character` là extended grapheme cluster, độ dài byte thay đổi → tới ký tự thứ n phải duyệt từ đầu, O(n). Swift dùng `String.Index` để không tạo ảo giác O(1). `count` cũng O(n).</details>

**6.** `Substring` khác `String` thế nào?
<details><summary>Đáp án</summary>`Substring` dùng chung bộ nhớ với String gốc (tạo rẻ) nhưng giữ String gốc sống. Lưu lâu dài thì chuyển `String(sub)`.</details>

**7.** Độ phức tạp `contains` của Array, Set? `append` của Array?
<details><summary>Đáp án</summary>Array `contains` O(n); Set `contains` O(1) trung bình. `append` O(1) amortized (thỉnh thoảng cấp phát lại gấp đôi capacity).</details>

**8.** Quy tắc bắt buộc giữa `Hashable` và `Equatable`?
<details><summary>Đáp án</summary>Nếu `a == b` thì `hash(a) == hash(b)`. (Ngược lại không bắt buộc — va chạm hash là hợp lệ.)</details>

**9.** `switch` của Swift khác C ở đâu?
<details><summary>Đáp án</summary>Phải exhaustive, không tự fallthrough, match được range/tuple/enum associated value/`where`/type cast.</details>

**10.** `inout` hoạt động thế nào?
<details><summary>Đáp án</summary>Copy-in copy-out (có thể tối ưu thành truyền địa chỉ). Truyền `var` kèm `&`. Không truyền cùng biến vào 2 tham số inout (Law of Exclusivity).</details>

**11.** `Int.max + 1` thì sao?
<details><summary>Đáp án</summary>Crash runtime (overflow trap). Muốn wrap dùng `&+`.</details>

**12.** `==` vs `===`?
<details><summary>Đáp án</summary>`==` so sánh giá trị (Equatable). `===` so sánh có phải **cùng một instance** class không.</details>

### Tuần 2 — Struct, Class, Enum · Value vs Reference

**13.** Khác biệt cốt lõi giữa struct và class?
<details><summary>Đáp án</summary>Struct: value type (gán = copy), không kế thừa, memberwise init, không deinit, không ARC cho bản thân nó. Class: reference type (gán = chia sẻ), kế thừa, deinit, ARC, identity (`===`).</details>

**14.** Copy-on-write là gì? Kiểu nào có sẵn?
<details><summary>Đáp án</summary>Copy chỉ thực sự xảy ra khi một bản bị **sửa** mà buffer đang được chia sẻ (kiểm tra bằng `isKnownUniquelyReferenced`). `Array`, `Dictionary`, `Set`, `String` có sẵn; struct tự viết **không** tự có COW.</details>

**15.** Vì sao method sửa struct phải đánh dấu `mutating`?
<details><summary>Đáp án</summary>Vì `self` trong method của value type mặc định bất biến. `mutating` cho phép sửa `self` (về ngữ nghĩa là thay cả giá trị). Không gọi được `mutating` trên hằng `let`.</details>

**16.** Struct chứa một property kiểu class — copy struct thì sao?
<details><summary>Đáp án</summary>Chỉ copy **tham chiếu** → hai struct cùng trỏ một object; sửa object qua bản này thấy ở bản kia. Value semantics bị phá nếu không cẩn thận.</details>

**17.** `lazy var` là gì, có thread-safe không?
<details><summary>Đáp án</summary>Giá trị chỉ được tính lần đầu truy cập. Phải là `var`. **Không** thread-safe — hai thread truy cập lần đầu cùng lúc có thể khởi tạo 2 lần.</details>

**18.** Property observer `willSet`/`didSet` có chạy trong `init` không?
<details><summary>Đáp án</summary>Không chạy khi gán trong `init` của chính kiểu đó (chạy khi subclass gán property của superclass trong init của subclass).</details>

**19.** Designated vs convenience init?
<details><summary>Đáp án</summary>Designated khởi tạo đầy đủ mọi stored property của class và gọi `super.init`. Convenience phải gọi một init khác của **cùng class** (`self.init`), cuối cùng tới designated.</details>

**20.** Two-phase initialization là gì?
<details><summary>Đáp án</summary>Phase 1: mọi stored property được gán giá trị (từ subclass lên superclass). Phase 2: sau khi đủ, mới được dùng `self`, gọi method, tuỳ chỉnh thêm. Đảm bảo không đọc property chưa khởi tạo.</details>

**21.** Enum associated value vs raw value?
<details><summary>Đáp án</summary>Raw value: mỗi case một giá trị hằng cùng kiểu, xác định lúc compile (`case a = "A"`), có `init?(rawValue:)`. Associated value: mỗi instance mang dữ liệu riêng, kiểu tuỳ case (`case success(Data)`).</details>

**22.** `indirect enum` dùng khi nào?
<details><summary>Đáp án</summary>Khi enum đệ quy (case chứa chính kiểu đó, ví dụ cây biểu thức, linked list) — compiler lưu dữ liệu qua heap box để kích thước enum hữu hạn.</details>

**23.** `static` vs `class` func trong class?
<details><summary>Đáp án</summary>Cả hai là type method. `class` cho phép subclass override; `static` = `class final`, không override được.</details>

**24.** `final` có lợi gì?
<details><summary>Đáp án</summary>Cấm kế thừa/override → compiler dùng static dispatch thay vì dynamic (vtable), nhanh hơn, và thể hiện ý đồ thiết kế.</details>

### Tuần 3 — Closure · Higher-order function · Protocol · POP

**25.** Closure capture biến thế nào?
<details><summary>Đáp án</summary>Capture **tham chiếu** tới biến (không phải copy giá trị) — closure thấy giá trị mới nhất. Capture list `[x]` thì copy giá trị tại thời điểm tạo closure.</details>

**26.** `@escaping` nghĩa là gì?
<details><summary>Đáp án</summary>Closure có thể được gọi **sau khi** hàm trả về (lưu vào property, gọi trong async callback). Mặc định closure tham số là non-escaping. Escaping closure trong class phải ghi `self.` rõ ràng để nhắc nguy cơ retain cycle.</details>

**27.** `@autoclosure` để làm gì? Ví dụ trong stdlib?
<details><summary>Đáp án</summary>Tự bọc biểu thức thành closure → trì hoãn đánh giá. Ví dụ `??` (vế phải chỉ tính khi cần), `assert(_:)`, `@StateObject(wrappedValue:)`.</details>

**28.** `map`, `compactMap`, `flatMap` trên Array khác nhau?
<details><summary>Đáp án</summary>`map`: biến đổi 1–1. `compactMap`: biến đổi rồi bỏ `nil`. `flatMap`: biến đổi mỗi phần tử thành sequence rồi làm phẳng (`[[a]] → [a]`).</details>

**29.** `reduce(into:)` khác `reduce(_:_:)`?
<details><summary>Đáp án</summary>`into` truyền accumulator `inout` → sửa tại chỗ, tránh copy mỗi bước (hiệu quả khi accumulator là Array/Dictionary).</details>

**30.** Protocol-oriented programming là gì? Vì sao Swift khuyến khích?
<details><summary>Đáp án</summary>Thiết kế dựa trên protocol + extension (default implementation) + composition thay vì kế thừa class. Dùng được với value type, một kiểu conform nhiều protocol, tránh "kế thừa sâu" và superclass phình to, dễ mock khi test.</details>

**31.** Method khai trong protocol extension nhưng **không** khai trong protocol — gọi qua biến kiểu protocol thì chạy bản nào?
<details><summary>Đáp án</summary>Bản của **extension** (static dispatch), kể cả khi kiểu cụ thể có method cùng tên. Method **có** khai trong protocol (requirement) thì dynamic dispatch qua witness table → bản của kiểu cụ thể.</details>

**32.** Extension làm được gì và không làm được gì?
<details><summary>Đáp án</summary>Được: thêm computed property, method, init (convenience cho class), subscript, nested type, conform protocol. Không: thêm stored property, override method có sẵn (trừ `@objc dynamic`), thêm designated init cho class.</details>

**33.** Delegate pattern trong Swift viết thế nào để không retain cycle?
<details><summary>Đáp án</summary>Protocol ràng buộc `AnyObject` (`protocol XDelegate: AnyObject`) để khai được `weak var delegate: XDelegate?`.</details>

**34.** Trailing closure và multiple trailing closures?
<details><summary>Đáp án</summary>Closure cuối có thể viết ngoài ngoặc: `arr.map { $0 * 2 }`. Từ Swift 5.3 nhiều closure: closure đầu không label, các closure sau có label (`Button { } label: { }`).</details>

### Tuần 4 — Generics · some/any · Error · Property wrapper · Access control · Codable

**35.** Vì sao dùng generics thay `Any`?
<details><summary>Đáp án</summary>Giữ type safety lúc compile, không cần cast, compiler có thể specialize → nhanh. `Any` mất thông tin kiểu, phải cast runtime.</details>

**36.** `associatedtype` là gì?
<details><summary>Đáp án</summary>Placeholder kiểu bên trong protocol, kiểu conform quyết định cụ thể (ví dụ `Collection.Element`, `View.Body`). Protocol có associatedtype trước Swift 5.7 không dùng được làm kiểu biến trực tiếp.</details>

**37.** `some Protocol` vs `any Protocol`?
<details><summary>Đáp án</summary>`some`: opaque type — một kiểu cụ thể cố định, compiler biết, static dispatch, nhanh. `any`: existential box — chứa được nhiều kiểu khác nhau lúc runtime, dynamic dispatch, có chi phí box/cấp phát; mất thông tin associatedtype.</details>

**38.** `throws` vs `rethrows`?
<details><summary>Đáp án</summary>`rethrows`: hàm chỉ ném lỗi nếu closure tham số ném (ví dụ `map`). Gọi với closure không ném thì không cần `try`.</details>

**39.** `try`, `try?`, `try!` khác nhau?
<details><summary>Đáp án</summary>`try`: lan lỗi ra ngoài (trong `do/catch` hoặc hàm `throws`). `try?`: lỗi → `nil`. `try!`: lỗi → crash.</details>

**40.** Typed throws (Swift 6) là gì?
<details><summary>Đáp án</summary>`func f() throws(NetworkError)` — khai báo chính xác kiểu lỗi; `catch` nhận `error` đúng kiểu `NetworkError`, không phải `any Error`.</details>

**41.** `Result<Success, Failure>` dùng khi nào?
<details><summary>Đáp án</summary>Biểu diễn kết quả thành công/thất bại thành giá trị — hữu ích cho callback (completion handler) và lưu trữ kết quả. Với async/await, `throws` thường tự nhiên hơn.</details>

**42.** Property wrapper là gì? `wrappedValue` và `projectedValue`?
<details><summary>Đáp án</summary>Kiểu (`@propertyWrapper`) đóng gói logic get/set dùng lại (clamp, UserDefaults, trim). `wrappedValue` là giá trị khi truy cập `x`; `projectedValue` truy cập qua `$x` (ví dụ `Binding` của `@State`).</details>

**43.** 5 mức access control từ mở nhất đến kín nhất?
<details><summary>Đáp án</summary>`open` > `public` > `internal` (mặc định) > `fileprivate` > `private`. `open` cho phép subclass/override ngoài module; `public` thì không.</details>

**44.** `CodingKeys` dùng để làm gì?
<details><summary>Đáp án</summary>Đổi tên key JSON ↔ property (`case avatarURL = "avatar_url"`) hoặc loại property khỏi encode/decode. Đổi snake_case toàn bộ có thể dùng `decoder.keyDecodingStrategy = .convertFromSnakeCase`.</details>

**45.** Decode JSON có field thiếu/null thế nào cho an toàn?
<details><summary>Đáp án</summary>Khai property Optional (thiếu/null → nil), hoặc viết `init(from:)` tự custom dùng `decodeIfPresent` + giá trị mặc định.</details>

**46.** Generic `where` clause ví dụ?
<details><summary>Đáp án</summary>`func allEqual<C: Collection>(_ c: C) -> Bool where C.Element: Equatable` — ràng buộc thêm trên associated type.</details>

### Tuần 5 — Bộ nhớ & ARC

**47.** ARC là gì? Khác garbage collection?
<details><summary>Đáp án</summary>Automatic Reference Counting: compiler chèn retain/release lúc compile; object bị giải phóng **ngay** khi strong count về 0. GC chạy runtime, quét định kỳ, thời điểm giải phóng không xác định, xử lý được cycle. ARC **không** tự phá cycle.</details>

**48.** `strong`, `weak`, `unowned` khác nhau?
<details><summary>Đáp án</summary>`strong` tăng refcount. `weak` không tăng, luôn Optional, tự thành `nil` khi object chết. `unowned` không tăng, không Optional, truy cập khi object đã chết → crash.</details>

**49.** Khi nào dùng `unowned` thay `weak`?
<details><summary>Đáp án</summary>Khi chắc chắn object được tham chiếu sống **ít nhất lâu bằng** object tham chiếu (ví dụ `Customer` ↔ `CreditCard.customer`). Không chắc thì dùng `weak`.</details>

**50.** Retain cycle hay gặp nhất trong iOS?
<details><summary>Đáp án</summary>(1) Closure lưu trong property của `self` capture `self` mạnh. (2) Delegate khai `strong`. (3) Hai object tham chiếu mạnh lẫn nhau (parent ↔ child). (4) Timer / NotificationCenter block / Combine `sink` giữ `self`.</details>

**51.** `[weak self]` hay `[unowned self]` trong closure network?
<details><summary>Đáp án</summary>`[weak self]` + `guard let self else { return }` — request có thể trả về sau khi màn hình đã đóng; `unowned` sẽ crash.</details>

**52.** Closure trong `UIView.animate` hay `DispatchQueue.main.async` có cần `[weak self]` không?
<details><summary>Đáp án</summary>Thường không — closure non-escaping hoặc escaping nhưng **không được self lưu giữ**, chạy xong là giải phóng → không có cycle (chỉ kéo dài sự sống tạm thời).</details>

**53.** Tìm retain cycle bằng công cụ nào?
<details><summary>Đáp án</summary>Xcode **Memory Graph Debugger** (thấy object không được giải phóng, mũi tên tham chiếu, dấu ⚠️ leak); Instruments **Leaks** và **Allocations** (mark generation, đếm instance còn sống); kiểm tra `deinit` có được gọi không.</details>

**54.** Value type có bị ARC quản lý không?
<details><summary>Đáp án</summary>Bản thân struct/enum thì không, nhưng nếu nó chứa reference (class, closure, buffer của Array/String) thì copy struct sẽ retain các reference đó.</details>

**55.** Stack vs heap allocation?
<details><summary>Đáp án</summary>Stack: nhanh (dời con trỏ), tự giải phóng theo scope, dành cho value type kích thước cố định. Heap: cấp phát động, cần đồng bộ thread, refcount — instance class, closure có capture, buffer collection, existential lớn.</details>

**56.** `autoreleasepool` còn dùng khi nào trong Swift?
<details><summary>Đáp án</summary>Vòng lặp tạo nhiều object autoreleased từ API Objective-C (xử lý ảnh `UIImage`, `NSString`...) → bọc thân vòng lặp trong `autoreleasepool {}` để giảm peak memory.</details>

### Tuần 6 — Thread · GCD · OperationQueue

**57.** Serial vs concurrent queue?
<details><summary>Đáp án</summary>Serial: mỗi lúc chạy 1 task, theo thứ tự FIFO. Concurrent: nhiều task chạy đồng thời (bắt đầu theo FIFO, kết thúc không theo thứ tự). Main queue là serial.</details>

**58.** `sync` vs `async`?
<details><summary>Đáp án</summary>`async`: gửi task rồi trả về ngay. `sync`: gửi task và **chặn** thread gọi đến khi task xong. Liên quan đến thread gọi, không phải queue.</details>

**59.** Khi nào bị deadlock với GCD?
<details><summary>Đáp án</summary>Gọi `sync` lên **chính serial queue đang chạy** (ví dụ `DispatchQueue.main.sync` từ main thread) — task mới chờ task hiện tại xong, task hiện tại chờ task mới → chờ nhau mãi.</details>

**60.** Race condition là gì? Cách phòng?
<details><summary>Đáp án</summary>Nhiều thread truy cập cùng dữ liệu, ít nhất một ghi, không đồng bộ → kết quả phụ thuộc thứ tự chạy. Phòng: serial queue, concurrent queue + barrier, `NSLock`/`os_unfair_lock`, actor. Phát hiện: Thread Sanitizer.</details>

**61.** Reader–writer bằng GCD viết thế nào?
<details><summary>Đáp án</summary>Concurrent queue: đọc bằng `queue.sync { }` (đọc song song), ghi bằng `queue.async(flags: .barrier) { }` (ghi độc quyền).</details>

**62.** QoS là gì? Kể các mức.
<details><summary>Đáp án</summary>Quality of Service — mức ưu tiên: `userInteractive` > `userInitiated` > `default` > `utility` > `background`.</details>

**63.** `DispatchGroup` dùng làm gì?
<details><summary>Đáp án</summary>Chờ nhiều task bất đồng bộ cùng xong: `enter()`/`leave()` (hoặc `async(group:)`), rồi `notify(queue:)` (không chặn) hoặc `wait()` (chặn).</details>

**64.** OperationQueue hơn GCD ở điểm nào?
<details><summary>Đáp án</summary>Dependency giữa operation, huỷ (`cancel()`), giới hạn `maxConcurrentOperationCount`, KVO trạng thái (isExecuting/isFinished), Operation có thể subclass và tái sử dụng.</details>

**65.** Vì sao UI phải cập nhật trên main thread?
<details><summary>Đáp án</summary>UIKit không thread-safe; main thread chạy main run loop xử lý sự kiện và vẽ. Cập nhật từ thread khác → hành vi không xác định, crash, UI không cập nhật. Main Thread Checker báo lỗi.</details>

**66.** `DispatchSemaphore` dùng khi nào? Nguy cơ?
<details><summary>Đáp án</summary>Giới hạn số truy cập đồng thời (đếm), hoặc biến async thành chờ đồng bộ. Nguy cơ: `wait()` trên main thread → treo UI/deadlock; priority inversion; không dùng trong Swift Concurrency.</details>

**67.** Thread explosion là gì?
<details><summary>Đáp án</summary>Gửi quá nhiều block chặn (blocking) lên concurrent queue → GCD tạo thêm rất nhiều thread → tốn bộ nhớ, context switch. Swift Concurrency tránh bằng cooperative thread pool (số thread ≈ số core).</details>

### Tuần 7 — async/await · Task · TaskGroup · actor · @MainActor

**68.** `await` nghĩa là gì?
<details><summary>Đáp án</summary>Đánh dấu **suspension point**: hàm có thể tạm dừng, **nhả thread** cho việc khác; khi tiếp tục có thể ở thread khác (trừ khi bị cô lập vào actor như MainActor). Không chặn thread.</details>

**69.** `async let` vs `TaskGroup`?
<details><summary>Đáp án</summary>`async let`: số lượng task cố định biết lúc compile, kiểu kết quả có thể khác nhau. `TaskGroup`: số lượng động (lặp), cùng kiểu kết quả, lấy kết quả theo thứ tự hoàn thành. Cả hai là structured concurrency.</details>

**70.** Structured concurrency nghĩa là gì?
<details><summary>Đáp án</summary>Task con có phạm vi sống nằm trong task cha: cha không kết thúc trước khi con xong; huỷ cha → huỷ con; lỗi lan lên. Ngược lại `Task {}` và `Task.detached {}` là unstructured.</details>

**71.** `Task { }` vs `Task.detached { }`?
<details><summary>Đáp án</summary>`Task {}` kế thừa actor context (vd. MainActor), priority, task-local values. `Task.detached` không kế thừa gì. Cả hai phải tự huỷ thủ công.</details>

**72.** Cancellation trong Swift Concurrency hoạt động thế nào?
<details><summary>Đáp án</summary>Hợp tác (cooperative): `cancel()` chỉ đặt cờ. Code phải kiểm tra `Task.isCancelled` / `try Task.checkCancellation()`; API như `Task.sleep`, `URLSession` tự ném `CancellationError`.</details>

**73.** Actor đảm bảo điều gì?
<details><summary>Đáp án</summary>Truy cập trạng thái mutable bên trong được tuần tự hoá (mỗi lúc một task chạy trên actor) → không data race. Từ bên ngoài phải `await`.</details>

**74.** Actor reentrancy là gì?
<details><summary>Đáp án</summary>Tại mỗi `await` bên trong method actor, actor có thể chạy task khác → trạng thái có thể đã đổi khi quay lại. Đừng giả định state giữ nguyên qua `await`; đọc lại / kiểm tra lại sau await.</details>

**75.** `@MainActor` dùng thế nào?
<details><summary>Đáp án</summary>Global actor gắn với main thread. Đánh dấu class/func/property → luôn chạy trên main. Dùng cho ViewModel/UI code. Gọi từ ngoài phải `await` (hoặc `MainActor.run {}`).</details>

**76.** Chuyển API callback sang async thế nào?
<details><summary>Đáp án</summary>`withCheckedContinuation` / `withCheckedThrowingContinuation`; gọi `continuation.resume(...)` **đúng một lần** (gọi 2 lần → crash, không gọi → treo vĩnh viễn).</details>

**77.** Hàm `async` không thuộc actor (nonisolated) chạy ở đâu?
<details><summary>Đáp án</summary>Theo SE-0338: trên global concurrent executor (rời actor của caller). Swift 6.2 có chế độ mới (`NonisolatedNonsendingByDefault`, bật trong "Approachable Concurrency") cho nó chạy trên actor của caller; muốn chắc chắn chạy nền thì đánh dấu `@concurrent`.</details>

### Tuần 8 — Sendable · Swift 6 · AsyncSequence · Combine

**78.** `Sendable` là gì?
<details><summary>Đáp án</summary>Protocol đánh dấu kiểu an toàn để truyền qua ranh giới concurrency (giữa task/actor) mà không gây data race: value type có thành phần Sendable, actor, class `final` chỉ có `let` Sendable, hoặc class tự đảm bảo (`@unchecked Sendable`).</details>

**79.** Swift 6 strict concurrency thay đổi gì?
<details><summary>Đáp án</summary>Kiểm tra data race lúc **compile**: lỗi (không chỉ cảnh báo) khi truyền giá trị non-Sendable qua actor, truy cập state cô lập sai actor, global var mutable không cô lập.</details>

**80.** `@Sendable` closure?
<details><summary>Đáp án</summary>Closure có thể chạy đồng thời; chỉ được capture giá trị Sendable và không capture `var` mutable. Ví dụ closure của `Task {}`, `addTask`.</details>

**81.** `@unchecked Sendable` dùng khi nào?
<details><summary>Đáp án</summary>Khi class tự đảm bảo thread-safety bằng lock/queue mà compiler không chứng minh được. Trách nhiệm thuộc về bạn — dùng hạn chế.</details>

**82.** `nonisolated` trong actor nghĩa là gì?
<details><summary>Đáp án</summary>Method/property không cô lập vào actor → gọi không cần `await`, nhưng không được truy cập state mutable của actor (chỉ `let` Sendable).</details>

**83.** `AsyncSequence` là gì? Ví dụ?
<details><summary>Đáp án</summary>Sequence mà phần tử đến bất đồng bộ, duyệt bằng `for try await x in seq`. Ví dụ `URL.lines`, `NotificationCenter.notifications(named:)`, `AsyncStream`.</details>

**84.** `AsyncStream` dùng để làm gì?
<details><summary>Đáp án</summary>Bọc nguồn sự kiện kiểu callback/delegate (location updates, websocket) thành AsyncSequence: `continuation.yield(x)`, `finish()`, `onTermination` để dọn dẹp.</details>

**85.** Ba thành phần chính của Combine?
<details><summary>Đáp án</summary>Publisher (phát giá trị + completion), Operator (`map`, `filter`, `debounce`, `combineLatest`...), Subscriber (`sink`, `assign`). Subscription trả `AnyCancellable` — phải giữ lại, bị huỷ thì subscription dừng.</details>

**86.** `PassthroughSubject` vs `CurrentValueSubject`?
<details><summary>Đáp án</summary>Passthrough: không lưu giá trị, subscriber chỉ nhận giá trị phát sau khi đăng ký. CurrentValue: lưu giá trị hiện tại (`.value`), subscriber mới nhận ngay giá trị đó.</details>

**87.** Retain cycle với Combine xảy ra thế nào?
<details><summary>Đáp án</summary>`self` giữ `cancellables`, `sink { self.x = $0 }` capture self mạnh → cycle. Dùng `[weak self]`. `assign(to: \.x, on: self)` cũng giữ self mạnh → dùng `assign(to: &$x)` (iOS 14) cho `@Published`.</details>

**88.** `receive(on:)` vs `subscribe(on:)`?
<details><summary>Đáp án</summary>`receive(on:)`: các operator/subscriber **phía sau** nhận giá trị trên scheduler đó (vd. `.main` để cập nhật UI). `subscribe(on:)`: công việc subscribe/request **phía trên** chạy trên scheduler đó.</details>

**89.** `debounce` vs `throttle`?
<details><summary>Đáp án</summary>`debounce`: chỉ phát khi nguồn **ngừng** phát một khoảng thời gian (search box). `throttle`: phát tối đa một giá trị mỗi khoảng thời gian (đầu hoặc cuối khoảng) dù nguồn phát liên tục (scroll, nút bấm).</details>

**90.** Combine hay async/await cho code mới?
<details><summary>Đáp án</summary>async/await cho tác vụ một lần (network request) và luồng tuần tự; AsyncSequence cho stream đơn giản. Combine vẫn mạnh cho kết hợp nhiều stream theo thời gian (`combineLatest`, `debounce`) và codebase sẵn có. Apple đang đầu tư vào Swift Concurrency + Observation.</details>

**91.** Data race vs race condition?
<details><summary>Đáp án</summary>Data race: hai truy cập cùng vùng nhớ đồng thời, ít nhất một ghi, không đồng bộ → undefined behavior (Swift 6 chặn lúc compile). Race condition: lỗi logic do thứ tự sự kiện (vd. actor reentrancy, check-then-act) — vẫn có thể xảy ra **dù không có data race**.</details>

**92.** Vì sao `DispatchSemaphore.wait()` trong async function là ý tưởng tồi?
<details><summary>Đáp án</summary>Chặn một thread của cooperative pool (số thread có hạn) → có thể làm cạn pool, deadlock. Swift Concurrency yêu cầu task luôn "tiến triển" (forward progress); hãy dùng `await` thay vì chặn.</details>

---

## ⭐ Top 20 must-know (nói trôi được thì yên tâm ăn Tết)

> Mỗi câu nói to ≤ 1 phút. Đây là các câu **gần như chắc chắn** bị hỏi ở vòng kỹ thuật iOS 2 năm.

1. Optional là gì và các cách unwrap — thẻ 1–3
2. Struct vs class, khi nào chọn cái nào — thẻ 13
3. Copy-on-write — thẻ 14
4. Closure capture & capture list — thẻ 25
5. `@escaping` — thẻ 26
6. Protocol extension dispatch (static vs dynamic) — thẻ 31
7. POP vs kế thừa class — thẻ 30
8. `some` vs `any` — thẻ 37
9. ARC & strong/weak/unowned — thẻ 47–48
10. Các nguồn retain cycle & cách tìm — thẻ 50, 53
11. `[weak self]` vs `[unowned self]` — thẻ 51
12. Serial vs concurrent, sync vs async — thẻ 57–58
13. Deadlock với `main.sync` — thẻ 59
14. Race condition & cách làm thread-safe — thẻ 60–61
15. GCD vs OperationQueue — thẻ 64
16. async/await & suspension point — thẻ 68
17. Structured concurrency, `async let` vs TaskGroup — thẻ 69–70
18. Actor & reentrancy — thẻ 73–74
19. `@MainActor` & Sendable / Swift 6 — thẻ 75, 78–79
20. Combine cơ bản & retain cycle với `sink` — thẻ 85, 87

---

## 🧩 Quiz: đoán output

> Đọc code, **viết output ra giấy**, rồi mới mở đáp án. Mọi đoạn đều đã chạy bằng `swift file.swift` (Swift 6.3). Muốn tự kiểm: copy vào file và chạy.

**Câu 1 — Value vs reference (Tuần 2)**
```swift
struct PointS { var x: Int }
final class PointC { var x: Int; init(x: Int) { self.x = x } }

var s1 = PointS(x: 1)
var s2 = s1
s2.x = 2

let c1 = PointC(x: 1)
let c2 = c1
c2.x = 2

print(s1.x, s2.x, c1.x, c2.x)
```
<details><summary>Đáp án</summary>

```
1 2 2 2
```
`s2` là bản copy → `s1` giữ 1. `c1` và `c2` cùng trỏ một object → cả hai thấy 2. (Chú ý: `c2` là `let` nhưng vẫn sửa được `x`.)
</details>

**Câu 2 — Closure capture (Tuần 3)**
```swift
var x = 1
let a = { print("a:", x) }
let b = { [x] in print("b:", x) }
x = 2
a()
b()
```
<details><summary>Đáp án</summary>

```
a: 2
b: 1
```
`a` capture **biến** `x` (tham chiếu) → thấy giá trị mới. `b` có capture list → copy giá trị `x` lúc tạo closure.
</details>

**Câu 3 — `defer` (Tuần 4)**
```swift
func run() -> String {
    defer { print("defer 1") }
    defer { print("defer 2") }
    print("body")
    return "return"
}
print(run())
```
<details><summary>Đáp án</summary>

```
body
defer 2
defer 1
return
```
`defer` chạy khi rời scope, theo thứ tự **ngược** với khai báo (LIFO). Giá trị trả về được tính trước khi defer chạy, nhưng `print(run())` chỉ in sau khi hàm kết thúc hoàn toàn.
</details>

**Câu 4 — Optional `map` / `flatMap` (Tuần 1)**
```swift
let text: String? = "5"
let a = text.map { Int($0) }
let b = text.flatMap { Int($0) }
let c = text.flatMap { Int($0) }.map { $0 * 2 } ?? 0
let d: String? = nil
print(a as Any)
print(b as Any)
print(c, d?.count ?? -1)
```
<details><summary>Đáp án</summary>

```
Optional(Optional(5))
Optional(5)
10 -1
```
`Int(String)` trả `Int?` → `map` lồng thành `Int??`; `flatMap` làm phẳng. `d?.count` là `nil` → `?? -1`.
</details>

**Câu 5 — Protocol extension dispatch (Tuần 3)**
```swift
protocol Greeter { func hi() }
extension Greeter {
    func hi() { print("P.hi") }
    func bye() { print("P.bye") }
}
struct S: Greeter {
    func hi() { print("S.hi") }
    func bye() { print("S.bye") }
}
let g: Greeter = S()
g.hi()
g.bye()
let s = S()
s.bye()
```
<details><summary>Đáp án</summary>

```
S.hi
P.bye
S.bye
```
`hi` là **requirement** của protocol → dynamic dispatch qua witness table → bản của `S`. `bye` chỉ có trong extension → với biến kiểu `Greeter`, static dispatch → bản extension. Với biến kiểu `S` thì gọi bản của `S`. Câu hỏi phỏng vấn kinh điển.
</details>

**Câu 6 — ARC, retain cycle, weak (Tuần 5)**
```swift
final class Node {
    let name: String
    var next: Node?
    init(_ name: String) { self.name = name }
    deinit { print("deinit \(name)") }
}
do {
    let a = Node("A")
    let b = Node("B")
    a.next = b
    b.next = a
}
print("--- scope 1 xong")
var c: Node? = Node("C")
weak var w = c
c = nil
print(w?.name ?? "nil")
```
<details><summary>Đáp án</summary>

```
--- scope 1 xong
deinit C
nil
```
A và B giữ nhau mạnh → retain cycle → **không** có "deinit A/B" (leak). C chỉ có một strong reference; `c = nil` → refcount 0 → deinit ngay; `weak` tự thành `nil`. Sửa cycle: `weak var next`.
</details>

**Câu 7 — GCD serial queue, sync/async (Tuần 6)**
```swift
import Foundation
let queue = DispatchQueue(label: "serial")
print("1")
queue.async { print("2") }
queue.sync { print("3") }
print("4")
```
<details><summary>Đáp án</summary>

```
1
2
3
4
```
Queue serial: block "2" được xếp trước; `sync` block "3" phải chờ "2" xong; thread gọi bị chặn đến khi "3" xong rồi mới in "4". Thứ tự xác định. (Nếu queue là concurrent thì "2" và "3" có thể đảo.)
</details>

**Câu 8 — `async let` (Tuần 7)**
```swift
func work(_ n: Int) async -> Int {
    try? await Task.sleep(for: .milliseconds(n * 50))
    print("done \(n)")
    return n
}
async let a = work(3)
async let b = work(1)
print("started")
let sum = await a + b
print("sum \(sum)")
```
<details><summary>Đáp án</summary>

```
started
done 1
done 3
sum 4
```
`async let` khởi chạy task con **ngay** và chạy song song; dòng "started" in trước vì chưa `await`. `work(1)` ngủ 50ms nên xong trước `work(3)` (150ms). Tổng thời gian ≈ 150ms, không phải 200ms.
</details>

**Câu 9 — Actor reentrancy (Tuần 7)**
```swift
actor Counter {
    var value = 0
    func incrementSlowly() async {
        let current = value
        try? await Task.sleep(for: .milliseconds(100))
        value = current + 1
    }
}
let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<3 {
        group.addTask { await counter.incrementSlowly() }
    }
}
print(await counter.value)
```
<details><summary>Đáp án</summary>

```
1
```
Không phải 3! Tại `await Task.sleep`, actor **nhả** để chạy lời gọi khác (reentrancy) → cả 3 lời gọi đều đọc `current = 0` trước khi ai kịp ghi, rồi cùng ghi `0 + 1`. Không có data race (actor vẫn tuần tự), nhưng có **race condition logic**. Sửa: không giữ state đọc trước qua `await` — viết `value += 1` sau `await`, hoặc làm phần đọc–ghi trong một đoạn đồng bộ không có `await`.
</details>

**Câu 10 — Error handling (Tuần 4)**
```swift
enum LoadError: Error { case notFound }
func load(_ id: Int) throws -> String {
    guard id > 0 else { throw LoadError.notFound }
    return "item \(id)"
}
let r1 = try? load(1)
let r2 = try? load(0)
let r3 = Result { try load(-1) }
print(r1 as Any, r2 as Any)
switch r3 {
case .success(let v): print("ok", v)
case .failure(let e): print("fail", e)
}
```
<details><summary>Đáp án</summary>

```
Optional("item 1") nil
fail notFound
```
`try?` biến kết quả thành Optional, lỗi → `nil`. `Result { try ... }` (init `catching:`) bắt lỗi thành `.failure`.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Đi hết flashcard tuần 1–8 ít nhất 1 lần
- [ ] Đi lại các thẻ sai lần 2; số thẻ sai lần 2 ≤ 10
- [ ] Quiz đoán output đúng ≥ 8/10
- [ ] Nói được Top 20 must-know, mỗi câu ≤ 1 phút
- [ ] Làm lại 3 bài DSA cũ không xem lời giải cũ
- [ ] Ghi lại các thẻ vẫn sai → mang sang tuần 14 ôn thêm
- [ ] Đã nghỉ Tết thật sự 🧧

## 📖 Tài liệu đọc thêm

- Không bắt buộc. Nếu thẻ nào sai, mở lại đúng § tương ứng trong file tuần đó (week01 → week08) thay vì tìm tài liệu mới.
- [Anki](https://apps.ankiweb.net/) — nếu muốn chuyển flashcard sang app ôn ngắt quãng (spaced repetition)
