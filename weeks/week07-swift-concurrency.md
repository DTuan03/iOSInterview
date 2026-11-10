# Tuần 7 — Swift Concurrency: async/await, Task, actor (21/12 – 27/12/2026)

> **Giai đoạn 2 · Bộ nhớ & Concurrency** · Thời lượng: ~14 giờ
> Tuần có **Giáng sinh** (24–25/12) nhưng vẫn là tuần học bình thường — nếu tối 24 có hẹn, dời phần T5 sang sáng T7.
> Phỏng vấn năm 2027 gần như chắc chắn hỏi Swift Concurrency. Câu hỏi phân loại ứng viên không phải "async/await là gì" mà là: **"await có tạo thread mới không?"**, **"actor reentrancy là gì?"**, **"Task {} và Task.detached khác gì?"**, **"cancel một Task thì code có dừng ngay không?"**. Tuần này phải hiểu cơ chế, không chỉ cú pháp.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích `async`/`await`, **suspension point** là gì, và vì sao await **không phải là thread**
- [ ] Giải thích **structured concurrency**: `async let`, `TaskGroup`/`ThrowingTaskGroup`, cây task, tự động huỷ task con
- [ ] Phân biệt `Task {}` vs `Task.detached {}` (kế thừa gì, không kế thừa gì)
- [ ] Giải thích **cancellation hợp tác**: `Task.checkCancellation()`, `Task.isCancelled`, `withTaskCancellationHandler`
- [ ] Giải thích actor: isolation, vì sao phải `await` từ bên ngoài, `nonisolated`, và **bẫy reentrancy** (kèm cách sửa)
- [ ] Dùng đúng `@MainActor`, `MainActor.run`, global actor; biết thay đổi của Swift 6.2 (`@concurrent`, default isolation)
- [ ] Bọc callback API thành async bằng `withCheckedThrowingContinuation` (resume **đúng 1 lần**)
- [ ] So sánh Swift Concurrency với GCD: cooperative thread pool, không thread explosion
- [ ] Gọi GitHub API bằng `URLSession` async, dùng `.task` trong SwiftUI đúng vòng đời
- [ ] Làm xong 6 bài DSA tuần 7 — chủ đề **Linked List** (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 21/12** | 2h | Tạo `practice/week07/`. §1 Vì sao cần async/await, §2 **async/await & suspension point**. Bài tập 1 · DSA: Reverse Linked List | `day1.swift` |
| **T3 22/12** | 2h | §3 **Structured concurrency**: `async let`, `TaskGroup`. Bài tập 2 · DSA: Merge Two Sorted Lists | `day2.swift` |
| **T4 23/12** | 2h | §4 **Task & cancellation**, `Task.detached`. Bài tập 3 · DSA: Linked List Cycle | `day3.swift` |
| **T5 24/12** 🎄 | 1,5h | §5 **Actor & reentrancy** (đọc kỹ nhất tuần). Bài tập 4 · DSA: Middle of the Linked List | `day4.swift` |
| **T6 25/12** 🎄 | 1,5h | §6 `@MainActor`, global actor, Swift 6.2; §7 **Continuation**; §8 so sánh với GCD. Bài tập 5 · DSA: Remove Nth Node From End of List | `day5.swift` |
| **T7 26/12** | 4h | §9 URLSession async + SwiftUI `.task`. Bài tập 6 + **Bài tập lớn** (actor `ImageLoader`). Side project: Milestone 7 (chuyển networking sang async/await với GitHub API thật, `ImageCache` dạng actor) · DSA: Reorder List | `imageloader-actor.swift`, side project |
| **CN 27/12** | 2h | Tự trả lời 20 câu phỏng vấn. Ghi âm 2 câu. So sánh code tuần 6 vs tuần 7 của cùng bài `ImageLoader`. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

> Code non-UI biên dịch ở **Swift 6 language mode**: `swiftc -swift-version 6 file.swift -o app && ./app`. Ở chế độ này, code top-level (ngoài hàm) được coi là chạy trên **MainActor** và được phép `await` trực tiếp. Lệnh `swift file.swift` (Swift 5 mode) cũng chạy được các ví dụ này.

### §1. Vì sao cần async/await?

Code GCD + completion handler tuần trước có những vấn đề cố hữu:
```swift
// Callback "kim tự tháp" — minh hoạ
fetchUser(id) { result in
    switch result {
    case .success(let user):
        fetchRepos(user) { result in
            switch result {
            case .success(let repos):
                DispatchQueue.main.async { self.show(repos) }
            case .failure(let error):
                DispatchQueue.main.async { self.show(error) }     // quên dòng này → UI treo spinner mãi
            }
        }
    case .failure(let error):
        DispatchQueue.main.async { self.show(error) }
    }
}
```
1. **Lồng nhau**, khó đọc; luồng lỗi không dùng được `throw`/`try`.
2. **Compiler không kiểm tra** completion có được gọi đúng 1 lần ở mọi nhánh hay không.
3. Dễ quên quay về main thread; dễ quên `[weak self]`.
4. Huỷ phải tự làm thủ công.
5. Data race chỉ phát hiện lúc chạy (TSan) — không phải lúc compile.
6. Thread bị chặn → thread explosion.

Swift Concurrency (Swift 5.5, iOS 13+) giải quyết: code async **viết như code tuần tự**, lỗi dùng `throws`, huỷ được lan truyền tự động trong cây task, **compiler kiểm tra data race** (actor, `Sendable` — tuần 8 học sâu).
```swift
let user = try await fetchUser(id)
let repos = try await fetchRepos(user)
show(repos)            // nếu hàm này @MainActor thì tự về main
```

### §2. async/await & suspension point

```swift
import Foundation

struct User: Sendable { let id: Int; let name: String }

func fetchUser(id: Int) async throws -> User {
    try await Task.sleep(for: .milliseconds(100))   // giả lập network
    return User(id: id, name: "user\(id)")
}
```
- `async` trong chữ ký: hàm này **có thể tạm dừng** (suspend) giữa chừng.
- `await`: đánh dấu một **suspension point** — chỗ hàm *có thể* tạm dừng để chờ. Bắt buộc viết ra để người đọc biết "ở đây có thể có chuyện khác xảy ra xen vào".
- `try await` khi hàm vừa `async` vừa `throws`.
- Chỉ gọi được hàm async từ **ngữ cảnh async** (hàm async khác hoặc một `Task`):
  ```swift
  // func tap() { let x = await load() }
  // ❌ error: 'async' call in a function that does not support concurrency
  // ✅ func tap() { Task { let x = await load() } }
  ```

**Chuyện gì xảy ra khi `await`? — câu hỏi số 1 của tuần**

```
Thread T1:  [ fetchUser chạy ... gặp await Task.sleep ]  → T1 được TRẢ LẠI cho pool, đi làm task khác
                                    │
                                    │  trạng thái hàm (biến local, vị trí đang chạy) được lưu
                                    │  vào "async frame" trên heap — KHÔNG có thread nào bị chặn
                                    ▼
                             ...100ms sau, kết quả sẵn sàng...
                                    │
Thread T3:                          └─▶ [ fetchUser chạy tiếp từ sau await ... return ]
                                         (có thể là thread khác T1!)
```
- **`await` không tạo thread**, cũng **không chặn thread**. Nó *trả thread lại* cho hệ thống trong lúc chờ. Đây là khác biệt cốt lõi với `semaphore.wait()` hay `group.wait()` của GCD.
- Sau `await`, code **có thể chạy tiếp trên thread khác** (trừ khi hàm thuộc một actor như `@MainActor` — khi đó nó quay về đúng actor đó).
- `await` là *có thể* suspend, không phải *chắc chắn* suspend — nếu kết quả có sẵn, hàm chạy tiếp luôn.

Bằng chứng "không phải thread": 10.000 task cùng chờ 1 giây, xong trong khoảng 1 giây, trên một máy chỉ có 8 core:
```swift
import Foundation

let clock = ContinuousClock()
let elapsed = await clock.measure {
    await withTaskGroup(of: Void.self) { group in
        for _ in 1...10_000 {
            group.addTask { try? await Task.sleep(for: .seconds(1)) }   // suspend, KHÔNG chặn thread
        }
    }
}
print("10.000 task, mỗi task chờ 1 giây → tổng:", elapsed < .seconds(2) ? "≈ 1 giây" : "\(elapsed)")
// 10.000 task, mỗi task chờ 1 giây → tổng: ≈ 1 giây
```
Làm điều tương tự bằng GCD với `Thread.sleep` trên global queue → hàng trăm thread bị chặn (thread explosion, tuần 6 §9) và chạy lâu hơn nhiều.

**Chú ý:** trong hàm async, các API chặn thread bị cấm hoặc cảnh báo — ví dụ:
```swift
// func f() async { print(Thread.isMainThread) }
// ❌ error: class property 'isMainThread' is unavailable from asynchronous contexts; Work intended for the main actor should be marked with @MainActor
```

### §3. Structured concurrency: `async let` và `TaskGroup`

`await` liên tiếp chạy **tuần tự**. Muốn **song song**:

```swift
func sequential() async throws -> [User] {
    let a = try await fetchUser(id: 1)      // chờ 100ms
    let b = try await fetchUser(id: 2)      // chờ thêm 100ms
    let c = try await fetchUser(id: 3)      // chờ thêm 100ms → tổng ~300ms
    return [a, b, c]
}

func parallel() async throws -> [User] {
    async let a = fetchUser(id: 1)          // bắt đầu NGAY, chạy song song
    async let b = fetchUser(id: 2)
    async let c = fetchUser(id: 3)
    return try await [a, b, c]              // chờ cả 3 → tổng ~100ms
}
```

**`async let`** — khi số lượng việc song song **biết trước lúc viết code** (2–5 việc khác nhau, kiểu khác nhau được).

**`TaskGroup` / `ThrowingTaskGroup`** — khi số lượng **động** (một mảng id):
```swift
func fetchAll(ids: [Int]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask { try await fetchUser(id: id) }
        }
        var users: [User] = []
        for try await user in group { users.append(user) }   // nhận theo thứ tự XONG, không theo thứ tự add
        return users.sorted { $0.id < $1.id }
    }
}

let clock = ContinuousClock()
var t = clock.now
let s = try await sequential()
print("sequential:", s.map(\.name), "— mất ≥ 300ms:", clock.now - t > .milliseconds(290))
t = clock.now
let p = try await parallel()
print("parallel:", p.map(\.name), "— mất < 200ms:", clock.now - t < .milliseconds(200))
print("group:", try await fetchAll(ids: Array(1...10)).count)
// sequential: ["user1", "user2", "user3"] — mất ≥ 300ms: true
// parallel: ["user1", "user2", "user3"] — mất < 200ms: true
// group: 10
```

**"Structured" nghĩa là gì?** Task con được gắn vào **cây task** của task cha, giống như biến local gắn vào scope:
```
Task cha (parallel())
├── child: fetchUser(1)
├── child: fetchUser(2)
└── child: fetchUser(3)
```
1. **Cha không thể kết thúc trước con** — ra khỏi scope `async let` / `withTaskGroup` thì mọi task con đã xong (hoặc bị huỷ và chờ xong). Không có task "mồ côi" chạy lén.
2. **Huỷ lan xuống**: cha bị huỷ → mọi con bị huỷ.
3. **Lỗi lan lên**: trong `ThrowingTaskGroup` / `async let`, một con throw → lỗi được ném ra ở chỗ `await`, các con còn lại **bị huỷ**.
4. Con **kế thừa** priority và task-local values của cha.

Lưu ý: `async let` không được `await` thì khi ra khỏi scope, task con bị **huỷ ngầm và vẫn được chờ** — không bị bỏ rơi.

### §4. Task: tạo, huỷ, detached

**Unstructured task — `Task { }`**: cầu nối từ code đồng bộ (button tap, `viewDidLoad`) sang thế giới async.
```swift
let task = Task {                // chạy ngay, không cần ai await
    try await processPages()
}
task.cancel()                    // huỷ
let value = try await task.value // chờ kết quả (nếu cần)
```
`Task { }` **kế thừa** từ nơi tạo ra nó: **actor hiện tại** (tạo trong `@MainActor` → closure chạy trên MainActor), **priority**, **task-local values**. Nhưng nó **không phải con** của task hiện tại: task hiện tại bị huỷ thì task mới **không** tự bị huỷ; phải giữ tham chiếu để tự huỷ.

**`Task.detached { }`**: **không kế thừa gì** (actor, priority, task-local). Dùng rất hiếm — khi chắc chắn muốn việc nặng **tách khỏi** actor hiện tại (ví dụ đang ở MainActor mà muốn xử lý ảnh lớn ở nền). Apple khuyên: ưu tiên `Task {}` hoặc structured concurrency; với Swift 6.2 có thể dùng hàm `@concurrent` thay cho `Task.detached` (§6).

| | `async let` / TaskGroup | `Task { }` | `Task.detached { }` |
|---|---|---|---|
| Structured (là con của task hiện tại) | ✅ | ❌ | ❌ |
| Tự huỷ khi cha huỷ | ✅ | ❌ | ❌ |
| Kế thừa actor | (con chạy ngoài actor) | ✅ | ❌ |
| Kế thừa priority, task-local | ✅ | ✅ | ❌ |
| Dùng khi | Song song trong hàm async | Từ code sync vào async; việc gắn với vòng đời object | Hiếm: tách hẳn khỏi ngữ cảnh |

**Priority:** `Task(priority: .userInitiated) { }` — `.high`, `.medium`, `.low`, `.userInitiated`, `.utility`, `.background`. Nếu một task priority cao `await` task priority thấp, runtime **nâng priority** task thấp (priority escalation) để tránh priority inversion.

**Cancellation là HỢP TÁC (cooperative)** — câu hỏi bẫy kinh điển:
> `task.cancel()` **không dừng code ngay**. Nó chỉ bật cờ "đã bị huỷ". Code bên trong phải **tự kiểm tra** và dừng.

```swift
import Foundation

func processPages() async throws -> Int {
    var done = 0
    for page in 1...10 {
        try Task.checkCancellation()          // ném CancellationError nếu đã bị huỷ
        try await Task.sleep(for: .milliseconds(50))   // Task.sleep cũng ném lỗi khi bị huỷ
        done = page
    }
    return done
}

let task = Task { try await processPages() }
try await Task.sleep(for: .milliseconds(120))
task.cancel()
do {
    let n = try await task.value
    print("xong", n)
} catch is CancellationError {
    print("Task bị huỷ → CancellationError")     // ← in dòng này
}

// Không kiểm tra → cancel không có tác dụng gì
let ignoring = Task {
    var sum = 0
    for i in 0..<3 { sum += i }
    return sum
}
ignoring.cancel()
print("Task không check cancel vẫn trả:", await ignoring.value)   // 3
```

3 cách phản ứng với cancel:
| API | Khi nào dùng |
|---|---|
| `try Task.checkCancellation()` | Muốn dừng hẳn bằng cách ném `CancellationError` |
| `if Task.isCancelled { ... }` | Muốn tự quyết: trả kết quả dở dang, dọn dẹp, trả giá trị mặc định |
| `withTaskCancellationHandler { } onCancel: { }` | Cần phản ứng **ngay lập tức** khi bị huỷ (ví dụ huỷ một request legacy); `onCancel` chạy đồng thời, trên thread bất kỳ |

```swift
// isCancelled: trả kết quả dở dang
func countPrimes(upTo n: Int) async -> (count: Int, finished: Bool) {
    var count = 0
    for i in 2...n {
        if i % 1000 == 0 {
            if Task.isCancelled { return (count, false) }
            await Task.yield()          // nhường thread cho task khác trong vòng lặp CPU dài
        }
        if (2..<i).allSatisfy({ i % $0 != 0 }) { count += 1 }
    }
    return (count, true)
}

// Huỷ cha → huỷ con (structured)
func child(_ name: String) async -> String {
    do { try await Task.sleep(for: .seconds(2)); return "\(name) xong" }
    catch { return "\(name) bị huỷ" }
}
let parent = Task {
    async let a = child("A")
    async let b = child("B")
    return await [a, b]
}
try await Task.sleep(for: .milliseconds(50))
parent.cancel()
print(await parent.value)       // ["A bị huỷ", "B bị huỷ"]
```

Các API của hệ thống (`URLSession.data`, `Task.sleep`) **tự kiểm tra cancel** và ném lỗi (`CancellationError` hoặc `URLError(.cancelled)`). Vòng lặp tính toán của bạn thì phải tự kiểm tra.

**Vòng đời Task trong UIKit — giữ và huỷ:**
```swift
final class RepoViewController: UIViewController {
    private var loadTask: Task<Void, Never>?
    // viewDidAppear: loadTask = Task { ... }
    // viewDidDisappear: loadTask?.cancel()
}
```
Code đầy đủ ở §9.

### §5. Actor — đọc kỹ nhất tuần

**Actor** là reference type (như class) nhưng **bảo vệ state của nó khỏi data race**: tại mỗi thời điểm, chỉ **một** task được chạy code bên trong actor. Có thể hình dung actor = class + một "serial queue" riêng, nhưng do **compiler** bắt buộc tuân thủ (không thể quên khoá).

```swift
actor BankAccount {
    let id: String                       // let bất biến: đọc từ ngoài không cần await
    private(set) var balance: Int

    init(id: String, balance: Int) { self.id = id; self.balance = balance }

    func deposit(_ amount: Int) { balance += amount }     // bên trong: truy cập trực tiếp, đồng bộ

    func withdraw(_ amount: Int) -> Bool {
        guard balance >= amount else { return false }
        balance -= amount
        return true
    }

    nonisolated var description: String { "Account \(id)" }   // không đụng state mutable → không cần isolation
}

let account = BankAccount(id: "A1", balance: 100)
await withTaskGroup(of: Void.self) { group in
    for _ in 1...1000 { group.addTask { await account.deposit(1) } }   // từ ngoài: phải await
}
print(await account.balance)            // 1100 — luôn đúng, không cần lock
print(account.id, account.description)  // A1 Account A1 — không cần await

// account.balance += 1
// ❌ error: actor-isolated property 'balance' can not be mutated from a nonisolated context
```
- **Actor isolation**: state `var` của actor chỉ được chạm **từ bên trong** actor một cách đồng bộ. Từ bên ngoài phải gọi qua `await` — vì có thể phải **chờ đến lượt** (actor đang bận phục vụ task khác).
- Không thể sửa property actor từ ngoài, kể cả với `await` — phải viết method.
- `nonisolated`: đánh dấu method/computed property **không cần isolation** (chỉ đọc `let` hoặc không đụng state) → gọi đồng bộ được từ ngoài. Hay dùng để conform protocol đồng bộ như `CustomStringConvertible`, `Hashable`.
- Actor **không hỗ trợ kế thừa** (ngoại lệ duy nhất: được kế thừa `NSObject` để dùng với Objective-C) → thiết kế bằng composition.

**⚠️ ACTOR REENTRANCY — bẫy số 1 của actor**

Actor chỉ đảm bảo **không có 2 đoạn code chạy cùng lúc** bên trong nó. Nhưng **tại mỗi `await` bên trong actor, actor "mở cửa"** cho task khác vào chạy. Khi hàm chạy tiếp sau `await`, **state có thể đã thay đổi**.

Ví dụ 1 — cache tải trùng:
```swift
actor ImageDownloader {
    private var cache: [String: String] = [:]
    var downloadCount = 0

    func image(for url: String) async -> String {
        if let cached = cache[url] { return cached }
        downloadCount += 1
        let image = await download(url)        // ⏸ suspension point: actor mở cửa cho call khác
        cache[url] = image
        return image
    }
    private func download(_ url: String) async -> String {
        try? await Task.sleep(for: .milliseconds(100))
        return "img(\(url))"
    }
}
let d = ImageDownloader()
async let x = d.image(for: "a")
async let y = d.image(for: "a")
_ = await (x, y)
print("Bản lỗi — số lần download:", await d.downloadCount)    // 2  ← tải trùng!
```
```
Task X: [check cache: trống][count=1][await download ⏸ ......................][ghi cache]
Task Y:                                [check cache: VẪN trống][count=2][await download ⏸ ....][ghi cache]
                                        ▲ Y vào được vì X đang suspend
```
**Sửa:** ghi lại "đang tải" **trước** `await`, task sau thấy thì chờ chung task đó:
```swift
actor FixedImageDownloader {
    private enum Entry { case inProgress(Task<String, Never>), ready(String) }
    private var cache: [String: Entry] = [:]
    var downloadCount = 0

    func image(for url: String) async -> String {
        if let entry = cache[url] {
            switch entry {
            case .ready(let image): return image
            case .inProgress(let task): return await task.value     // chờ chung, không tải lại
            }
        }
        downloadCount += 1
        let task = Task { await Self.download(url) }
        cache[url] = .inProgress(task)               // ghi TRƯỚC khi await
        let image = await task.value
        cache[url] = .ready(image)
        return image
    }
    private static func download(_ url: String) async -> String {
        try? await Task.sleep(for: .milliseconds(100))
        return "img(\(url))"
    }
}
let f = FixedImageDownloader()
async let x2 = f.image(for: "a")
async let y2 = f.image(for: "a")
_ = await (x2, y2)
print("Bản sửa — số lần download:", await f.downloadCount)    // 1
```

Ví dụ 2 — kiểm tra trước `await`, dùng sau `await`:
```swift
actor Wallet {
    var balance = 100
    func pay(_ amount: Int) async -> Bool {
        guard balance >= amount else { return false }
        try? await Task.sleep(for: .milliseconds(10))   // ví dụ: gọi server xác nhận
        balance -= amount                               // ❌ balance có thể đã đổi trong lúc chờ
        return true
    }
    func payFixed(_ amount: Int) async -> Bool {
        try? await Task.sleep(for: .milliseconds(10))
        guard balance >= amount else { return false }   // ✅ kiểm tra lại SAU await
        balance -= amount
        return true
    }
}
let w = Wallet()
async let p1 = w.pay(80)
async let p2 = w.pay(80)
print("pay:", await [p1, p2], "balance:", await w.balance)      // pay: [true, true] balance: -60
let w2 = Wallet()
async let q1 = w2.payFixed(80)
async let q2 = w2.payFixed(80)
let fixed = await [q1, q2]
print("payFixed thành công:", fixed.filter { $0 }.count, "balance:", await w2.balance)   // 1, balance: 20
```

**Quy tắc chống reentrancy:**
1. Thay đổi state **đồng bộ** (không có `await` ở giữa) — gom "kiểm tra + thay đổi" vào một đoạn liền.
2. Sau mỗi `await`, **coi mọi giả định về state là không còn đúng** → kiểm tra lại.
3. Với cache/dedupe: lưu `Task` đang chạy vào state **trước** khi `await`.

Vì sao Swift chọn actor **reentrant** thay vì khoá cả hàm? Nếu actor bị khoá suốt thời gian `await`, hai actor gọi lẫn nhau sẽ **deadlock**, và actor sẽ đứng im khi đang chờ network. Reentrancy tránh deadlock, đổi lại lập trình viên phải cẩn thận với state sau `await`.

### §6. `@MainActor`, global actor, `nonisolated`, Swift 6.2

**`@MainActor`** là một **global actor** đại diện cho main thread. Đánh dấu type/hàm/property với `@MainActor` → compiler bảo đảm nó chỉ chạy trên main thread; gọi từ nơi khác phải `await`.
```swift
@MainActor
final class ProfileViewModel {
    var name = ""
    var isLoading = false
    func load() async {
        isLoading = true
        defer { isLoading = false }
        let user = try? await fetch(id: 1)      // fetch chạy ngoài main; trong lúc chờ main RẢNH
        name = user ?? "?"                      // sau await: quay về MainActor → cập nhật UI an toàn
        print("load() chạy trên main thread:", onMainThread())   // true
    }
}
func onMainThread() -> Bool { Thread.isMainThread }   // hàm sync nên được dùng Thread.isMainThread
```
- `UIViewController`, `UIView`, SwiftUI `View` (từ iOS 18 SDK) đã được Apple đánh dấu `@MainActor` → `Task { }` tạo trong VC chạy trên main.
- Từ code không thuộc main: `await MainActor.run { label.text = "..." }` — tương đương `DispatchQueue.main.async` nhưng chờ được và có kiểm tra compile-time.
- **Đừng lẫn**: `@MainActor` không làm hàm "chạy chậm trên main" toàn bộ — mỗi `await` trong hàm đều **nhả main** trong lúc chờ.

**Global actor tự định nghĩa** — khi muốn nhiều type khác nhau dùng chung một "luồng tuần tự" (ví dụ mọi thứ liên quan database):
```swift
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
}
@DatabaseActor
final class Database {
    var rows: [String] = []
    func insert(_ row: String) { rows.append(row) }
}
// Từ hàm async khác:
// let db = Database()
// await db.insert("a")
// print(await db.rows)      // ["a"]
```

**`nonisolated`**: bỏ isolation cho một member của type có isolation (actor hoặc `@MainActor` class) — dùng khi member không chạm state mutable.

**Thay đổi quan trọng ở Swift 6.2 (Xcode 26) — "Approachable Concurrency":**

1. **Hàm `nonisolated async` chạy ở đâu?**
   - Trước đây (và mặc định khi chưa bật feature): luôn **nhảy ra global concurrent executor** (khỏi main).
   - Với upcoming feature `NonisolatedNonsendingByDefault` (bật sẵn trong project mới của Xcode 26): hàm `nonisolated async` **chạy trên actor của người gọi** (gọi từ MainActor → chạy trên main).
   - Muốn chắc chắn chạy ở nền: đánh dấu **`@concurrent`**.
2. **Default actor isolation**: build setting *Default Actor Isolation = MainActor* (flag `-default-isolation MainActor`) — mọi code không ghi rõ được coi là `@MainActor`. Project app mới trong Xcode 26 bật sẵn. Lý do: phần lớn code app là UI, chạy main là an toàn nhất; chỉ đánh dấu `@concurrent`/`nonisolated` ở chỗ cần chạy nền.

```swift
// swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault
import Foundation
func onMainThread() -> Bool { Thread.isMainThread }
nonisolated func inherit() async -> Bool { onMainThread() }
@concurrent func offMain() async -> Bool { onMainThread() }
@MainActor func demo() async {
    print("nonisolated (Swift 6.2 + NonisolatedNonsendingByDefault) chạy trên main:", await inherit())  // true
    print("@concurrent chạy trên main:", await offMain())                                                // false
}
await demo()
```
(Không bật feature, dòng đầu in `false`.) Trong phỏng vấn: nói được **"hàm async chạy ở đâu phụ thuộc isolation, và Swift 6.2 đổi mặc định cho hàm nonisolated async"** là điểm cộng lớn.

### §7. Continuation — bọc callback API thành async

Rất nhiều API (SDK cũ, thư viện bên thứ ba, delegate) vẫn dùng completion handler. Bọc lại:
```swift
import Foundation

func legacyFetch(id: Int, completion: @escaping @Sendable (Result<String, Error>) -> Void) {
    DispatchQueue.global().asyncAfter(deadline: .now() + .milliseconds(50)) {
        if id < 0 { completion(.failure(URLError(.badURL))) }
        else { completion(.success("item\(id)")) }
    }
}

func fetch(id: Int) async throws -> String {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch(id: id) { result in
            continuation.resume(with: result)      // resume ĐÚNG 1 lần
        }
    }
}

// API kiểu cũ (value?, error?) — phải xử lý ĐỦ mọi nhánh
func oldAPI(completion: @escaping @Sendable (String?, Error?) -> Void) {
    DispatchQueue.global().async { completion("ok", nil) }
}
func newAPI() async throws -> String {
    try await withCheckedThrowingContinuation { cont in
        oldAPI { value, error in
            if let error { cont.resume(throwing: error) }
            else if let value { cont.resume(returning: value) }
            else { cont.resume(throwing: URLError(.unknown)) }   // đừng quên nhánh này!
        }
    }
}
// print(try await fetch(id: 7))   → item7
// try await fetch(id: -1)         → ném URLError(.badURL)
// print(try await newAPI())       → ok
```

**Luật bất di bất dịch: resume ĐÚNG MỘT LẦN trên mọi đường đi.**
- Resume 2 lần → crash: `Fatal error: SWIFT TASK CONTINUATION MISUSE: twice() tried to resume its continuation more than once, returning 2!`
- Không bao giờ resume → task **treo vĩnh viễn** (rò rỉ), checked continuation in cảnh báo: `SWIFT TASK CONTINUATION MISUSE: never() leaked its continuation without resuming it. This may cause tasks waiting on it to remain suspended forever.`

| | `withCheckedContinuation` / `...ThrowingContinuation` | `withUnsafe...Continuation` |
|---|---|---|
| Kiểm tra resume 0 lần / 2 lần | ✅ (crash/cảnh báo rõ ràng) | ❌ (undefined behavior) |
| Chi phí | Nhỉnh hơn chút | Rẻ nhất |
| Dùng khi | **Mặc định** | Chỉ khi đã kiểm chứng kỹ và cần tối ưu |

Với API **gọi callback nhiều lần** (progress, location updates, delegate stream) → không dùng continuation mà dùng `AsyncStream` (tuần 8).

### §8. Cooperative thread pool & so sánh với GCD

Swift Concurrency chạy task trên một **cooperative thread pool**: số thread **không vượt quá số core CPU**. Task không bao giờ được chặn thread — khi chờ thì **suspend** và trả thread lại. Vì thế:
- **Không có thread explosion** (không có chuyện "thread bị chặn → tạo thêm thread").
- Context switch giữa các task rẻ (chỉ là gọi hàm tiếp tục continuation), không phải context switch thread của kernel.
- **Hợp đồng bắt buộc**: code trong task phải luôn "tiến lên được" — **không được chặn** bằng `semaphore.wait()`, `Thread.sleep`, lock giữ lâu, I/O đồng bộ dài. Chặn một thread trong pool chỉ có ~8 thread → làm cả app chậm, thậm chí deadlock.

| | GCD | Swift Concurrency |
|---|---|---|
| Đơn vị | Closure trên queue | Task / hàm async |
| Chờ | Chặn thread (`wait`, `sync`) hoặc callback | `await` — suspend, không chặn |
| Số thread | Có thể tăng vọt (thread explosion) | ≤ số core |
| Lỗi | Tự truyền qua `Result` | `throws` / `try` |
| Huỷ | Thủ công (`DispatchWorkItem`, cờ) | Lan truyền tự động trong cây task, cooperative |
| Data race | Phát hiện lúc chạy (TSan) | Phần lớn bắt **lúc compile** (actor, `Sendable`, Swift 6) |
| Bảo vệ state | Serial queue / lock tự quản | `actor` do compiler ép |
| Thứ tự | Serial queue đảm bảo FIFO | Actor **không** đảm bảo FIFO tuyệt đối; reentrant |
| Khi nào vẫn dùng GCD | Code cũ, API C, cần timing cụ thể, `concurrentPerform` cho CPU | Mọi code mới |

**Kết hợp hai thế giới:** gọi code GCD/callback từ async → continuation (§7). Gọi async từ code đồng bộ → `Task { }`. **Đừng** dùng semaphore để chờ một Task trong code đồng bộ (dễ deadlock).

### §9. URLSession async & SwiftUI `.task` (áp dụng vào RepoScope)

**URLSession async API (iOS 15+):** `data(from:)`, `data(for:)`, `download(from:)`, `upload(for:from:)`, `bytes(from:)` (AsyncSequence từng byte/dòng). Tự hỗ trợ cancellation: task bị huỷ → request bị huỷ, ném `URLError(.cancelled)`.

```swift
import SwiftUI
import UIKit

struct Repo: Decodable, Identifiable, Sendable {
    let id: Int
    let fullName: String
    let stargazersCount: Int
}
struct SearchResponse: Decodable, Sendable { let items: [Repo] }

enum APIError: Error { case badStatus(Int) }

struct GitHubClient: Sendable {
    let session: URLSession = .shared

    func searchRepos(query: String) async throws -> [Repo] {
        var components = URLComponents(string: "https://api.github.com/search/repositories")!
        components.queryItems = [URLQueryItem(name: "q", value: query)]
        var request = URLRequest(url: components.url!)
        request.setValue("application/vnd.github+json", forHTTPHeaderField: "Accept")

        let (data, response) = try await session.data(for: request)
        guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
            throw APIError.badStatus((response as? HTTPURLResponse)?.statusCode ?? -1)
        }
        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase      // full_name → fullName
        return try decoder.decode(SearchResponse.self, from: data).items
    }
}

@MainActor
@Observable
final class RepoListViewModel {
    var repos: [Repo] = []
    var errorMessage: String?
    private let client = GitHubClient()

    func search(_ query: String) async {
        do {
            repos = try await client.searchRepos(query: query)   // network ở nền, gán repos trên main
        } catch is CancellationError {
            // view biến mất → bỏ qua
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

struct RepoListView: View {
    @State private var viewModel = RepoListViewModel()
    @State private var query = "swift"

    var body: some View {
        List(viewModel.repos) { repo in
            Text(repo.fullName)
        }
        .task(id: query) {                  // chạy khi view xuất hiện, tự cancel khi biến mất / query đổi
            try? await Task.sleep(for: .milliseconds(300))   // debounce
            guard !Task.isCancelled else { return }
            await viewModel.search(query)
        }
        .searchable(text: $query)
    }
}
```

**`.task` trong SwiftUI** (giới thiệu — tuần 11–12 học sâu):
- Bắt đầu khi view **xuất hiện**, **tự động cancel** khi view biến mất → không cần lưu `Task`, không cần `[weak self]`.
- `.task(id:)`: khi `id` đổi, task cũ bị huỷ và task mới bắt đầu → debounce search chỉ với `Task.sleep` + kiểm tra cancel (so sánh với `DispatchWorkItem` tuần 6!).
- Closure chạy trên MainActor (vì `View` là `@MainActor`), các `await` bên trong nhả main trong lúc chờ.

**UIKit: tự quản vòng đời Task:**
```swift
final class RepoViewController: UIViewController {
    private var loadTask: Task<Void, Never>?
    private let client = GitHubClient()
    private let label = UILabel()

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        loadTask = Task { [weak self] in
            guard let client = self?.client else { return }
            let repos = (try? await client.searchRepos(query: "swift")) ?? []
            self?.label.text = "\(repos.count) repos"     // UIViewController là @MainActor → đang ở main
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadTask?.cancel()
    }
}
```
**Task có giữ `self` không?** Có — closure của `Task` capture `self` mạnh tới khi task **chạy xong**. Đây **không phải cycle vĩnh viễn** (trừ khi task chạy vô hạn, ví dụ `for await` một stream không kết thúc), nhưng giữ VC sống lâu hơn cần thiết. Với task dài hoặc vô hạn → `[weak self]` + huỷ trong `viewDidDisappear`/`deinit`. (Nối với tuần 5!)

**ImageCache dạng actor** (Milestone 7) — chính là pattern sửa reentrancy ở §5:
```swift
actor ImageCache {
    static let shared = ImageCache()
    private enum Entry { case inProgress(Task<UIImage, Error>), ready(UIImage) }
    private var entries: [URL: Entry] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let entry = entries[url] {
            switch entry {
            case .ready(let image): return image
            case .inProgress(let task): return try await task.value
            }
        }
        let task = Task<UIImage, Error> {
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else { throw URLError(.cannotDecodeContentData) }
            return image
        }
        entries[url] = .inProgress(task)
        do {
            let image = try await task.value
            entries[url] = .ready(image)
            return image
        } catch {
            entries[url] = nil          // lỗi → xoá để lần sau thử lại
            throw error
        }
    }
}
```
(Bản này chưa giới hạn bộ nhớ — có thể kết hợp `NSCache` của tuần 5 cho phần `.ready`.)

---

## ⚠️ Bẫy hay gặp

1. **"await tạo thread mới / chạy trên background thread"** — sai. await là suspension point, không tạo thread; hàm chạy ở đâu do **isolation** quyết định.
2. **"cancel() dừng task ngay"** — sai. Cancellation là hợp tác; code phải kiểm tra `isCancelled`/`checkCancellation()`.
3. **Actor reentrancy**: kiểm tra state trước `await` rồi dùng sau `await` → logic sai (tải trùng, trừ tiền âm).
4. **`Task.detached` để "chạy nền"** khi `Task {}` hoặc hàm `@concurrent` là đủ — mất priority, task-local, khó huỷ.
5. **`Task {}` trong `@MainActor` không có nghĩa là chạy nền** — nó kế thừa MainActor; code đồng bộ nặng bên trong vẫn chặn main.
6. **Chặn thread trong task** (`semaphore.wait()`, `Thread.sleep`, `DispatchQueue.sync` lâu) → làm nghẽn cooperative pool.
7. **Continuation resume 0 hoặc 2 lần** — đặc biệt ở nhánh `(nil, nil)` của API cũ.
8. **Quên huỷ Task lưu trong property** khi màn hình đóng → việc thừa, giữ `self` sống lâu.
9. **Nghĩ actor đảm bảo thứ tự FIFO như serial queue** — không đảm bảo hoàn toàn, và reentrancy làm xen kẽ công việc.
10. **`async let` mà quên `await`** → task con bị huỷ ngầm khi ra khỏi scope, kết quả mất.
11. **Gọi `Thread.isMainThread` / `Thread.current` trong hàm async** → lỗi compile ở Swift 6; dùng `MainActor.assertIsolated()` hoặc gắn `@MainActor` rõ ràng.

---

## 🛠 Bài tập

> Tạo file trong `practice/week07/`, biên dịch với `swiftc -swift-version 6 file.swift -o app && ./app`. Mục tiêu: **0 warning, 0 error** ở Swift 6 mode.

**Bài 1 — Từ callback sang async (T2)**
a) Viết lại `legacyFetch` (§7) và bọc thành `fetch(id:) async throws -> String` — không nhìn.
b) Cho 3 hàm callback giả lập `fetchUser(completion:)`, `fetchRepos(user:completion:)`, `fetchStars(repo:completion:)` lồng nhau. Viết lại chuỗi này bằng async/await, so sánh số dòng và cách xử lý lỗi.
c) Cố tình gọi `resume` 2 lần và quên `resume` — đọc thông báo của runtime.

**Bài 2 — TaskGroup có giới hạn đồng thời (T3)**
Viết `downloadAll(ids: [Int], limit: Int) async throws -> [Int: String]`: tải tất cả nhưng **không quá `limit` task chạy cùng lúc** (sliding window). Dùng một actor `Tracker` đếm `running`/`maxRunning` để chứng minh.
<details><summary>Lời giải</summary>

```swift
import Foundation

actor Tracker {
    var running = 0, maxRunning = 0
    func start() { running += 1; maxRunning = max(maxRunning, running) }
    func end() { running -= 1 }
}
let tracker = Tracker()

func download(_ id: Int) async throws -> String {
    await tracker.start()
    try await Task.sleep(for: .milliseconds(50))
    await tracker.end()
    return "file\(id)"
}

func downloadAll(ids: [Int], limit: Int) async throws -> [Int: String] {
    try await withThrowingTaskGroup(of: (Int, String).self) { group in
        var results: [Int: String] = [:]
        var iterator = ids.makeIterator()

        for _ in 0..<limit {                       // khởi động `limit` task đầu
            guard let id = iterator.next() else { break }
            group.addTask { (id, try await download(id)) }
        }
        while let (id, file) = try await group.next() {   // mỗi task xong → thêm 1 task mới
            results[id] = file
            if let next = iterator.next() {
                group.addTask { (next, try await download(next)) }
            }
        }
        return results
    }
}
let files = try await downloadAll(ids: Array(1...10), limit: 3)
print(files.count, "files, tối đa chạy cùng lúc:", await tracker.maxRunning)
// 10 files, tối đa chạy cùng lúc: 3
```
So với tuần 6: không cần semaphore (không chặn thread nào), không cần lock cho `results` (chỉ task cha ghi vào nó).
</details>

**Bài 3 — Timeout (T4)**
Viết `withTimeout(_ duration: Duration, operation:) async throws -> T`: nếu `operation` không xong trong `duration` thì ném `TimeoutError` **và huỷ** operation. Gợi ý: cho 2 task con "chạy đua" trong một group.
<details><summary>Lời giải</summary>

```swift
import Foundation

struct TimeoutError: Error {}
func withTimeout<T: Sendable>(
    _ duration: Duration,
    operation: @escaping @Sendable () async throws -> T
) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask { try await operation() }
        group.addTask {
            try await Task.sleep(for: duration)
            throw TimeoutError()
        }
        defer { group.cancelAll() }               // task thua bị huỷ
        return try await group.next()!            // ai xong (hoặc throw) trước thắng
    }
}
let fast = try await withTimeout(.milliseconds(200)) {
    try await Task.sleep(for: .milliseconds(50)); return "nhanh"
}
print(fast)                                       // nhanh
do {
    _ = try await withTimeout(.milliseconds(50)) {
        try await Task.sleep(for: .seconds(5)); return "chậm"
    }
} catch is TimeoutError {
    print("timeout!")                             // timeout!
}
```
Chương trình kết thúc ngay sau "timeout!" chứ không chờ 5 giây — vì `cancelAll()` làm `Task.sleep(for: .seconds(5))` ném lỗi ngay, và group chờ task con đó xong (structured) rồi mới thoát.
</details>

**Bài 4 — Actor & reentrancy (T5)**
a) Viết actor `Account` với `transfer(_ amount: Int, to other: Account) async -> Bool`. Gọi song song 2 lần `a.transfer(70, to: b)` khi `a` có 100. Kết quả phải là: 1 lần thành công, A = 30, B = 70.
b) Viết lại `ImageDownloader` bản lỗi (§5), chứng minh tải trùng, rồi sửa — không nhìn bản sửa.
<details><summary>Lời giải a</summary>

```swift
actor Account {
    let name: String
    private(set) var balance: Int
    init(name: String, balance: Int) { self.name = name; self.balance = balance }
    func deposit(_ amount: Int) { balance += amount }
    func transfer(_ amount: Int, to other: Account) async -> Bool {
        guard balance >= amount else { return false }
        balance -= amount                 // trừ TRƯỚC await → lần gọi thứ 2 thấy số dư mới
        await other.deposit(amount)       // suspension point
        return true
    }
}
let a = Account(name: "A", balance: 100)
let b = Account(name: "B", balance: 0)
async let t1 = a.transfer(70, to: b)
async let t2 = a.transfer(70, to: b)
let res = await [t1, t2]
print("thành công:", res.filter { $0 }.count, "A:", await a.balance, "B:", await b.balance)
// thành công: 1 A: 30 B: 70
```
Nếu đặt `balance -= amount` **sau** `await other.deposit(...)` → cả 2 lần đều qua được `guard` → A âm. Đây là reentrancy.
</details>

**Bài 5 — Dự đoán thứ tự (T6)**
```swift
@MainActor func order() async {
    print("1")
    Task { print("3") }
    print("2")
    await Task.yield()
    print("4")
}
await order()
```
In ra gì? Cái gì được đảm bảo?
<details><summary>Lời giải</summary>

Chạy thử in `1 2 3 4`. Đảm bảo: `1` trước `2`, và `3` chắc chắn **sau** `2` — vì `Task {}` kế thừa MainActor, mà MainActor đang bận chạy `order()` cho tới điểm suspend đầu tiên (`await Task.yield()`). `3` so với `4` thì **không được đảm bảo tuyệt đối** về mặt ngôn ngữ (thực tế `yield` nhường lượt nên `3` thường chạy trước `4`). Bài học: code trên cùng một actor chỉ xen kẽ nhau **tại các `await`**.
</details>

**Bài 6 — RepoScope gọi GitHub API thật (T7, side project)**
Trong side project: viết `GitHubClient.searchRepos(query:)` bằng `URLSession.data(for:)` (§9), ViewModel `@MainActor`, màn hình search có debounce bằng `.task(id:)` (SwiftUI) hoặc lưu/huỷ `Task` (UIKit). Thêm xử lý: lỗi HTTP 403 (rate limit của GitHub, 60 request/giờ khi chưa đăng nhập) hiển thị thông báo thân thiện; `CancellationError` thì bỏ qua. Biên dịch Swift 6 mode không warning.

**🏋️ Bài tập lớn (T7) — Port `ImageLoader` tuần 6 sang actor**
Viết `imageloader-actor.swift`: cùng yêu cầu với tuần 6 (cache, gộp request trùng, lỗi thì lần sau thử lại) nhưng dùng `actor` + `Task`, API `func load(_ url: URL) async throws -> Data`, `fetch` được inject. Test: 20 lời gọi song song cho 5 URL → đúng 5 lần fetch; URL lỗi gọi 2 lần → fetch 2 lần (không cache lỗi). Viết 5 dòng so sánh với bản GCD: số dòng, chỗ nào compiler giúp bạn, chỗ nào vẫn phải tự cẩn thận (reentrancy!).
<details><summary>Lời giải</summary>

```swift
import Foundation

actor ImageLoader {
    private enum Entry {
        case inProgress(Task<Data, Error>)
        case ready(Data)
    }
    private var entries: [URL: Entry] = [:]
    private let fetch: @Sendable (URL) async throws -> Data
    private(set) var fetchCount = 0

    init(fetch: @escaping @Sendable (URL) async throws -> Data) {
        self.fetch = fetch
    }

    func load(_ url: URL) async throws -> Data {
        if let entry = entries[url] {
            switch entry {
            case .ready(let data): return data
            case .inProgress(let task): return try await task.value
            }
        }
        fetchCount += 1
        let fetch = self.fetch
        let task = Task { try await fetch(url) }
        entries[url] = .inProgress(task)          // ghi trước await → chống reentrancy
        do {
            let data = try await task.value
            entries[url] = .ready(data)
            return data
        } catch {
            entries[url] = nil                    // không cache lỗi
            throw error
        }
    }

    func clear() { entries.removeAll() }
}

let loader = ImageLoader { url in
    try await Task.sleep(for: .milliseconds(100))
    if url.lastPathComponent == "bad.png" { throw URLError(.badServerResponse) }
    return Data(url.absoluteString.utf8)
}

let urls = (0..<20).map { URL(string: "https://example.com/\($0 % 5).png")! }
let sizes = try await withThrowingTaskGroup(of: Int.self) { group in
    for url in urls { group.addTask { try await loader.load(url).count } }
    var total = 0
    for try await size in group { total += size }
    return total
}
print("tổng bytes:", sizes, "· số lần fetch thật:", await loader.fetchCount)
// tổng bytes: 500 · số lần fetch thật: 5

let bad = URL(string: "https://example.com/bad.png")!
for _ in 1...2 {
    do { _ = try await loader.load(bad) } catch { print("lỗi, sẽ thử lại lần sau") }
}
print("fetchCount sau 2 lần lỗi:", await loader.fetchCount)
// lỗi, sẽ thử lại lần sau
// lỗi, sẽ thử lại lần sau
// fetchCount sau 2 lần lỗi: 7
```
So với GCD: không có `stateQueue`, không có mảng completion chờ — các caller trùng URL chỉ việc `await task.value`. Compiler cấm truy cập `entries` từ ngoài actor. Nhưng reentrancy vẫn là trách nhiệm của bạn: nếu ghi `entries[url]` **sau** `await` thì lại tải trùng.
</details>

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ async/await giải quyết vấn đề gì so với completion handler?**
<details><summary>Đáp án gợi ý</summary>

Code async viết tuần tự, dễ đọc, không lồng callback; lỗi dùng `throws`/`try` thay vì `Result` thủ công; compiler đảm bảo hàm luôn trả về/ném lỗi đúng một lần (không quên gọi completion); cancellation lan truyền tự động trong cây task; kết hợp actor/`Sendable` để bắt data race lúc compile; không chặn thread nên không thread explosion.
</details>

**Q2 ⭐ `await` có tạo thread mới không? Chuyện gì xảy ra ở một suspension point?**
<details><summary>Đáp án gợi ý</summary>

Không. `await` đánh dấu chỗ hàm có thể tạm dừng. Khi suspend, trạng thái hàm được lưu vào async frame trên heap và thread được trả lại cho cooperative pool để chạy việc khác — không thread nào bị chặn. Khi kết quả sẵn sàng, hàm được tiếp tục, có thể trên thread khác (trừ khi hàm thuộc một actor như MainActor thì quay về actor đó). Cũng vì vậy, sau mỗi await, state chung có thể đã thay đổi.
</details>

**Q3 ⭐ Structured concurrency là gì? `async let` khác `TaskGroup` thế nào?**
<details><summary>Đáp án gợi ý</summary>

Task con gắn vào cây task của cha và vòng đời bị giới hạn trong scope: cha không kết thúc trước con, huỷ cha thì huỷ con, lỗi của con lan lên cha (các con khác bị huỷ), con kế thừa priority/task-local. `async let` dùng khi số lượng việc song song cố định, biết lúc viết code (kiểu khác nhau được); `TaskGroup` dùng khi số lượng động, các task cùng kiểu kết quả, nhận kết quả theo thứ tự hoàn thành.
</details>

**Q4 ⭐ `Task {}` và `Task.detached {}` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`Task {}` là unstructured task kế thừa actor hiện tại (tạo trong `@MainActor` thì chạy trên MainActor), priority và task-local values. `Task.detached` không kế thừa gì cả. Cả hai đều không phải con của task hiện tại nên không tự huỷ theo — phải giữ handle để huỷ. Ưu tiên structured concurrency, rồi `Task {}`; `detached` chỉ khi thật sự muốn tách khỏi ngữ cảnh (Swift 6.2 có thể dùng hàm `@concurrent` thay thế).
</details>

**Q5 ⭐ Gọi `task.cancel()` thì code có dừng ngay không?**
<details><summary>Đáp án gợi ý</summary>

Không. Cancellation là hợp tác: `cancel()` chỉ đặt cờ, lan xuống các task con. Code phải tự kiểm tra bằng `try Task.checkCancellation()` (ném `CancellationError`) hoặc `Task.isCancelled` (tự xử lý), hoặc dùng `withTaskCancellationHandler` để phản ứng ngay. Các API hệ thống như `Task.sleep`, `URLSession.data` tự kiểm tra và ném lỗi. Vòng lặp tính toán không kiểm tra thì vẫn chạy đến hết.
</details>

**Q6 ⭐ Actor là gì? Khác class + serial queue thế nào?**
<details><summary>Đáp án gợi ý</summary>

Actor là reference type bảo vệ state mutable của nó: mỗi lúc chỉ một task chạy code bên trong actor, truy cập từ ngoài phải `await`. Khác class + serial queue: (1) **compiler ép buộc** isolation, không thể quên khoá; (2) không chặn thread — caller suspend thay vì `sync`; (3) actor **reentrant** — tại mỗi `await` bên trong, actor có thể xử lý lời gọi khác; (4) không đảm bảo FIFO tuyệt đối. Actor không hỗ trợ kế thừa.
</details>

**Q7 ⭐ Actor reentrancy là gì? Cho ví dụ và cách xử lý.**
<details><summary>Đáp án gợi ý</summary>

Khi một method của actor `await`, actor có thể chạy method khác trong lúc chờ; khi method đầu tiếp tục, state có thể đã thay đổi. Ví dụ cache: kiểm tra cache trống → await download → hai lời gọi cùng URL đều thấy trống → tải hai lần. Hoặc kiểm tra số dư → await → trừ tiền → số dư âm. Cách xử lý: thực hiện kiểm tra + thay đổi state đồng bộ (không có await ở giữa); sau await kiểm tra lại giả định; lưu `Task` đang chạy vào state trước khi await để các caller sau dùng chung. Swift chọn reentrant để tránh deadlock giữa các actor.
</details>

**Q8 ⭐ `@MainActor` là gì? Khác `DispatchQueue.main.async` thế nào?**
<details><summary>Đáp án gợi ý</summary>

`@MainActor` là global actor đại diện cho main thread; đánh dấu type/hàm/property để compiler đảm bảo chúng chỉ chạy trên main, gọi từ nơi khác phải `await`. Khác `DispatchQueue.main.async`: được kiểm tra lúc compile (quên về main là lỗi compile chứ không phải bug runtime), có thể chờ kết quả trả về, tích hợp cancellation. `UIViewController`, `UIView`, SwiftUI `View` đã là `@MainActor`. Từ ngữ cảnh khác có thể dùng `await MainActor.run { }`.
</details>

**Q9 Global actor là gì? Khi nào tự định nghĩa?**
<details><summary>Đáp án gợi ý</summary>

Global actor là một actor singleton có thể dùng làm attribute (`@MainActor`, hoặc `@globalActor actor DatabaseActor { static let shared = ... }`) để gom nhiều type/hàm khác nhau vào cùng một isolation domain. Tự định nghĩa khi một subsystem trải rộng nhiều type cần truy cập tuần tự, ví dụ mọi thứ chạm vào database/file cache.
</details>

**Q10 `nonisolated` dùng để làm gì?**
<details><summary>Đáp án gợi ý</summary>

Đánh dấu member của actor (hoặc type có global actor) không cần isolation — gọi đồng bộ được từ bên ngoài không cần `await`. Chỉ hợp lệ khi member không đụng state mutable (chỉ đọc `let`, tính toán thuần). Hay dùng để conform protocol đồng bộ như `Hashable`, `CustomStringConvertible`. Với hàm `nonisolated async`, Swift 6.2 (feature `NonisolatedNonsendingByDefault`) cho nó chạy trên actor của người gọi; muốn chạy nền thì dùng `@concurrent`.
</details>

**Q11 ⭐ Làm sao chuyển một API callback sang async/await?**
<details><summary>Đáp án gợi ý</summary>

Dùng `withCheckedContinuation` / `withCheckedThrowingContinuation`: gọi API cũ bên trong closure, trong callback gọi `continuation.resume(returning:)` / `resume(throwing:)` / `resume(with: result)`. Phải resume **đúng một lần** trên mọi nhánh: hai lần → crash, không lần nào → task treo vĩnh viễn (checked continuation sẽ cảnh báo). Callback gọi nhiều lần thì dùng `AsyncStream`. `withUnsafeContinuation` chỉ dùng khi đã chắc chắn và cần tối ưu.
</details>

**Q12 ⭐ So sánh Swift Concurrency với GCD.**
<details><summary>Đáp án gợi ý</summary>

GCD: closure trên queue, chờ bằng chặn thread hoặc callback, số thread có thể tăng vọt khi bị chặn, huỷ và xử lý lỗi thủ công, data race chỉ phát hiện lúc chạy. Swift Concurrency: hàm async/task, chờ bằng suspend không chặn thread, cooperative pool giới hạn số thread bằng số core (không thread explosion), cancellation và lỗi lan truyền theo cây task, actor + Sendable cho phép compiler bắt data race. Đổi lại phải tuân thủ "không chặn thread trong task" và cẩn thận reentrancy. GCD vẫn hữu ích cho code cũ, API C, `concurrentPerform`.
</details>

**Q13 Cooperative thread pool là gì? Vì sao không được dùng semaphore trong task?**
<details><summary>Đáp án gợi ý</summary>

Pool thread mà Swift Concurrency dùng để chạy task, số thread ≈ số core. Task "hợp tác" bằng cách suspend tại await để trả thread. Nếu chặn thread bằng `semaphore.wait()`, `Thread.sleep`, lock lâu… thread đó không làm việc khác được; vì pool nhỏ nên vài thread bị chặn là đủ làm nghẽn toàn bộ, thậm chí deadlock nếu thứ đang được chờ cần chính thread trong pool để chạy.
</details>

**Q14 Code sau một `await` trong hàm `@MainActor` chạy trên thread nào? Trong hàm không có isolation thì sao?**
<details><summary>Đáp án gợi ý</summary>

Hàm `@MainActor`: luôn quay về main thread sau await. Hàm không có isolation (nonisolated) async: mặc định truyền thống chạy trên global concurrent executor (thread bất kỳ trong pool, có thể đổi thread sau mỗi await); với Swift 6.2 khi bật `NonisolatedNonsendingByDefault`, nó chạy trên actor của caller. Không nên giả định thread cụ thể — hãy suy nghĩ theo isolation chứ không theo thread.
</details>

**Q15 ⭐ Task có gây retain cycle không? Có cần `[weak self]` trong `Task {}`?**
<details><summary>Đáp án gợi ý</summary>

Closure của Task capture `self` mạnh cho tới khi task kết thúc. Nếu task ngắn thì không phải cycle vĩnh viễn, chỉ kéo dài vòng đời self. Nếu self lưu task trong property **và** task chạy rất lâu/vô hạn (ví dụ `for await` trên stream không kết thúc), thì self không bao giờ được giải phóng → cần `[weak self]` và huỷ task khi màn hình đóng (`viewDidDisappear`/`deinit`). Trong SwiftUI dùng `.task` để hệ thống tự huỷ.
</details>

**Q16 `.task` modifier trong SwiftUI hoạt động thế nào? Khác `onAppear { Task { } }`?**
<details><summary>Đáp án gợi ý</summary>

`.task` khởi chạy một task gắn với vòng đời view: bắt đầu khi view xuất hiện, **tự huỷ** khi view biến mất. `.task(id:)` huỷ và chạy lại khi `id` thay đổi (hữu ích cho search/debounce). `onAppear { Task { } }` tạo unstructured task không ai huỷ → vẫn chạy sau khi view biến mất, có thể cập nhật state thừa.
</details>

**Q17 Priority của Task hoạt động thế nào? Priority inversion có xảy ra không?**
<details><summary>Đáp án gợi ý</summary>

Task có priority (`.high`, `.userInitiated`, `.medium`, `.low`, `.utility`, `.background`), mặc định kế thừa từ ngữ cảnh tạo. Khi task priority cao `await` một task priority thấp hoặc chờ một actor đang chạy việc priority thấp, runtime **escalate** priority của bên được chờ để giảm priority inversion. Không cần (và không nên) dùng priority để đảm bảo thứ tự logic.
</details>

**Q18 `withTaskCancellationHandler` dùng khi nào?**
<details><summary>Đáp án gợi ý</summary>

Khi cần phản ứng **ngay lúc** task bị huỷ thay vì chờ tới lần kiểm tra cờ tiếp theo — điển hình là khi bọc một API legacy có hàm cancel riêng (huỷ `URLSessionTask`, dừng một SDK). `onCancel` chạy đồng thời với operation, trên thread bất kỳ, nên state dùng trong đó phải thread-safe.
</details>

**Q19 Swift 6.2 "Approachable Concurrency" thay đổi gì?**
<details><summary>Đáp án gợi ý</summary>

(1) Hàm `nonisolated async` mặc định chạy trên actor của caller (`NonisolatedNonsendingByDefault`), muốn chạy nền phải ghi `@concurrent` — giảm lỗi Sendable khó hiểu khi gọi từ MainActor. (2) Có thể đặt **default actor isolation = MainActor** cho cả module (bật sẵn trong project app mới của Xcode 26) — code mặc định chạy main, chỉ phần cần song song mới đánh dấu. Ý tưởng: bắt đầu đơn luồng, thêm concurrency có chủ đích.
</details>

**Q20 Khi nào vẫn dùng GCD trong một codebase đã dùng async/await?**
<details><summary>Đáp án gợi ý</summary>

Khi làm việc với API C/ObjC yêu cầu dispatch queue (delegate queue của AVFoundation, CoreBluetooth), `DispatchQueue.concurrentPerform` cho vòng lặp CPU song song, timing chính xác với `DispatchSourceTimer`, hoặc code cũ chưa migrate. Ngoài ra code mới nên dùng Swift Concurrency; khi kết hợp thì bọc bằng continuation và không chặn thread của cooperative pool.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–6, biên dịch Swift 6 mode không warning
- [ ] Làm xong Bài tập lớn actor `ImageLoader`, viết 5 dòng so sánh với bản GCD tuần 6
- [ ] Trả lời được ≥ 16/20 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "await có tạo thread không — chuyện gì xảy ra ở suspension point"
- [ ] Ghi âm 2 phút: "Actor reentrancy là gì, ví dụ và cách sửa"
- [ ] Vẽ lại từ trí nhớ: cây task structured concurrency, sơ đồ reentrancy của cache
- [ ] Làm xong 6 bài DSA tuần 7 (Linked List)
- [ ] Side project Milestone 7 xong (networking async/await với GitHub API thật, `ImageCache` dạng actor)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) — **đọc hết**
- Apple — [Swift Concurrency](https://developer.apple.com/documentation/swift/concurrency), [URLSession](https://developer.apple.com/documentation/foundation/urlsession), [`task(id:priority:_:)`](https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:))
- WWDC 2021 — *Meet async/await in Swift* (**bắt buộc**)
- WWDC 2021 — *Explore structured concurrency in Swift* (async let, TaskGroup, cây task, cancellation)
- WWDC 2021 — *Protect mutable state with Swift actors* (actor, reentrancy, MainActor)
- WWDC 2021 — *Swift concurrency: Behind the scenes* (cooperative thread pool, async frame, vì sao không được chặn thread — rất hay cho phỏng vấn)
- WWDC 2021 — *Swift concurrency: Update a sample app* (migrate từ GCD sang async/await)
- WWDC 2022 — *Eliminate data races using Swift Concurrency*
- WWDC 2025 — *Embracing Swift concurrency* (Swift 6.2: `@concurrent`, default MainActor isolation)
- Swift Evolution — [SE-0296 async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md), [SE-0304 Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md), [SE-0306 Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md), [SE-0461 nonisolated(nonsending) & @concurrent](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
- Donny Wals — các bài về actor reentrancy, `Task` vs `Task.detached`
- Matt Massicotte — blog về Swift Concurrency & Swift 6 migration (giải thích isolation rất kỹ)
