# Tuần 8 — Sendable · Swift 6 strict concurrency · AsyncSequence · Combine (28/12/2026 – 03/01/2027)

> **Giai đoạn 2 · Bộ nhớ & Concurrency** · Thời lượng: ~12 giờ (tuần có Tết Dương lịch — **Thứ Sáu 01/01 nghỉ hẳn**)
> Tuần 7 bạn đã biết `async/await`, `Task`, `actor`. Tuần này trả lời câu hỏi tiếp theo mà người phỏng vấn **chắc chắn** hỏi ở mức 2 năm: *"Compiler làm sao biết code của bạn không có data race?"* → `Sendable`, isolation, Swift 6 language mode. Nửa sau tuần là **Combine** — nhiều codebase thật vẫn dùng, nên phải đọc hiểu và nói được khi nào chọn Combine, khi nào chọn async/await.
> Cuối tuần là **mốc M2** — có bài tự kiểm tra riêng ở cuối file.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích `Sendable` là gì, kiểu nào **tự động** Sendable, kiểu nào không, và khi nào được dùng `@unchecked Sendable`
- [ ] Giải thích `@Sendable` closure và vì sao nó cấm capture `var` / object non-Sendable
- [ ] Đọc và sửa được 4 lỗi Swift 6 kinh điển: global mutable state, main actor isolation, `sending ... risks causing data races`, capture trong `@Sendable` closure
- [ ] Nói được ý tưởng **region-based isolation** và từ khoá `sending` bằng 3–4 câu
- [ ] Trình bày quy trình **migrate** một project lên Swift 6 theo từng bước
- [ ] Viết `AsyncSequence` tự chế, dùng `AsyncStream` để **bọc API delegate/callback** và biết `onTermination`
- [ ] Viết pipeline Combine: `@Published` → `debounce` → `removeDuplicates` → `map` → `assign`/`sink`, giải thích `AnyCancellable` và retain cycle trong Combine
- [ ] Phân biệt `receive(on:)` vs `subscribe(on:)`, `flatMap` vs `switchToLatest`
- [ ] Trả lời được "Combine hay async/await, chọn cái nào?" có lý lẽ
- [ ] Làm xong 6 bài DSA tuần 8 (Linked List + Recursion — xem [dsa-plan](../tracks/dsa-plan.md))
- [ ] **Qua mốc M2** (phần cuối file)

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 28/12** | 2h | §1 **Sendable** (đọc kỹ). §2 `@Sendable` closure. Bài tập 1 · DSA: Reverse Linked List (làm cả iterative và recursive) | `day1.swift`, bảng phân loại Sendable |
| **T3 29/12** | 2h | §3 **Swift 6 strict concurrency — 5 lỗi kinh điển**. §4 Region-based isolation & `sending`. Bài tập 2 · DSA: Merge Two Sorted Lists | `day2.swift` compile sạch ở `-swift-version 6` |
| **T4 30/12** | 2h | §5 Migrate từng bước + Swift 6.2 "approachable concurrency". §6 **AsyncSequence / AsyncStream**. Bài tập 3, 4 · DSA: Linked List Cycle | `day3.swift` |
| **T5 31/12** | 1,5h | (Tất niên — tuần nhẹ) §7 **Combine cơ bản**: Publisher/Subscriber, Subject, `@Published`, `AnyCancellable`. Bài tập 5 | `day4.swift` |
| **T6 01/01** | **Nghỉ** | 🎉 Tết Dương lịch. Không học. (Nếu muốn: nghe lại 1 voice memo tuần 7, tối đa 15') | — |
| **T7 02/01** | 4h | §8 Operator & Scheduler, §9 Combine vs async/await. Bài tập 6, 7 + **Bài tập lớn**. DSA: Middle of the Linked List, Remove Nth Node From End of List. **Side project: Milestone 8** (bật Swift 6 language mode, sửa hết Sendable warning, search có debounce) | `search_debounce.swift`, commit side project |
| **CN 03/01** | 2,5h | Tự trả lời 18 câu phỏng vấn. **Làm bài tự kiểm tra M2** (60'). DSA: Palindrome Linked List. Tick checklist | Checklist ≥ 80%, M2 đạt |

> Tuần này mất 1 ngày (01/01), nên T7 và CN dài hơn bình thường 30'. Nếu T7 bận, **dời Bài tập 7 sang CN, giữ Bài tập lớn + side project ở T7.**

---

## 📚 Kiến thức cốt lõi

> Quy ước: code có ghi `// Swift 6 mode` nghĩa là đã compile bằng `swiftc -swift-version 6` (hoặc `swift -swift-version 6 file.swift`). Đây là chế độ bạn sẽ bật cho side project.

### §1. Sendable — "giấy thông hành" qua ranh giới concurrency

**Nhắc lại vấn đề (tuần 6):** *data race* xảy ra khi ≥ 2 luồng truy cập **cùng một vùng nhớ**, ít nhất một luồng **ghi**, và không có đồng bộ hoá. Kết quả: crash ngẫu nhiên, dữ liệu sai, rất khó tái hiện.

Swift Concurrency chia chương trình thành các **isolation domain** (miền cô lập):
- mỗi `actor` là một domain,
- `@MainActor` (main thread) là một domain,
- code `nonisolated` (không thuộc actor nào) là "vùng tự do", có thể chạy trên bất kỳ thread nào của cooperative pool.

Trong một domain, code chạy **tuần tự** → không race. Race chỉ có thể xảy ra khi **một giá trị đi qua ranh giới** giữa 2 domain (truyền vào actor, trả ra từ actor, capture vào `Task`...). `Sendable` là **protocol đánh dấu** (marker protocol — không có method nào, không tốn gì lúc runtime) nói với compiler: *"giá trị kiểu này đi qua ranh giới an toàn"*.

```
   ┌─────────── MainActor ──────────┐          ┌────────── actor Store ─────────┐
   │  let repo = Repo(...)          │  await   │                                │
   │  await store.save(repo) ───────┼─────────►│  func save(_ r: Repo)          │
   │                                │ ranh giới│                                │
   └────────────────────────────────┘ ↑        └────────────────────────────────┘
                                      compiler kiểm tra: Repo có Sendable không?
```

**Kiểu nào là Sendable?**

| Kiểu | Sendable? | Ghi chú |
|---|---|---|
| `Int`, `String`, `Bool`, `Double`, `Data`, `URL`, `Date`... | ✅ | Thư viện chuẩn đã khai báo |
| `struct` / `enum` mà **mọi** thuộc tính / associated value đều Sendable | ✅ **tự động** (nếu không `public`) | Copy là có bản riêng → không chia sẻ |
| `public struct` / `public enum` | ⚠️ Phải **khai báo tường minh** `: Sendable` | Compiler không suy luận cho API public (trừ `@frozen`) |
| `Array`, `Dictionary`, `Optional`, `Set`... | ✅ nếu phần tử Sendable | Conditional conformance |
| `actor` | ✅ luôn luôn | State được actor bảo vệ |
| Class gắn global actor (`@MainActor final class VM`) | ✅ | Mọi truy cập đều phải lên MainActor |
| `final class` chỉ có `let` thuộc tính Sendable, **khai báo** `: Sendable` | ✅ | Bất biến → không ai ghi |
| Class thường (có `var`, hoặc không `final`) | ❌ | Reference type, có state chia sẻ |
| Class kế thừa class khác (trừ `NSObject`) | ❌ | Lớp con có thể thêm state |
| Closure thường `() -> Void` | ❌ | Phải là `@Sendable () -> Void` |

**Bẫy quan trọng:** class **không bao giờ** tự động Sendable, kể cả `final class` chỉ có `let`. Bạn phải viết `: Sendable` để compiler kiểm tra.

```swift
// Swift 6 mode — compile & chạy được
import Foundation

// 1. Value types: tự Sendable nếu mọi thành phần Sendable
struct Repo: Sendable {          // khai báo tường minh vẫn tốt (bắt buộc nếu public)
    let id: Int
    var name: String             // var trong struct vẫn OK: mỗi bên giữ một bản copy
    var stars: Int
}
enum LoadState: Sendable { case idle, loading, loaded([Repo]), failed(String) }

// 2. final class + chỉ có let Sendable
final class AppConfig: Sendable {
    let baseURL: URL
    let pageSize: Int
    init(baseURL: URL, pageSize: Int) { self.baseURL = baseURL; self.pageSize = pageSize }
}

// 3. Actor luôn Sendable
actor Counter {
    private var value = 0
    func increment() -> Int { value += 1; return value }
}

// 4. @unchecked Sendable: TỰ bảo vệ bằng lock, compiler tin bạn
final class ThreadSafeCache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Int] = [:]
    func value(for key: String) -> Int? {
        lock.lock(); defer { lock.unlock() }
        return storage[key]
    }
    func set(_ value: Int, for key: String) {
        lock.lock(); defer { lock.unlock() }
        storage[key] = value
    }
}

func runAll() async {
    let cache = ThreadSafeCache()
    let counter = Counter()
    await withTaskGroup(of: Void.self) { group in
        for i in 0..<100 {
            group.addTask {                 // closure của addTask là @Sendable → chỉ capture được thứ Sendable
                cache.set(i, for: "k\(i % 10)")
                _ = await counter.increment()
            }
        }
    }
    print(await counter.increment())        // 101
    print(cache.value(for: "k3") != nil)    // true
}

await runAll()
```

**Các lỗi compiler khi khai báo sai** (đã kiểm tra bằng Swift 6.3):
```swift
// class Account: Sendable { var balance = 0 }
// ❌ error: non-final class 'Account' cannot conform to the 'Sendable' protocol
// ❌ error: stored property 'balance' of 'Sendable'-conforming class 'Account' is mutable

// class Base {}
// final class Child: Base, Sendable {}
// ❌ error: 'Sendable' class 'Child' cannot inherit from another class other than 'NSObject'
```

**`@unchecked Sendable` — khi nào được dùng?**
Khi bạn **tự** đảm bảo thread-safety bằng cơ chế compiler không hiểu: `NSLock`, `os_unfair_lock`, serial `DispatchQueue`, hoặc class của SDK cũ mà bạn biết chắc là thread-safe. Nó **tắt** kiểm tra → nếu bạn sai, data race quay lại và compiler không cứu. Quy tắc phỏng vấn: *"`@unchecked` là lời hứa, phải có lock đi kèm và comment giải thích."*

Từ iOS 18 có `Mutex` trong module `Synchronization` — một `final class` chứa `let state = Mutex<[String: Int]>([:])` là `Sendable` **thật** (không cần `@unchecked`). Nếu app còn hỗ trợ iOS 17 thì vẫn dùng `NSLock` + `@unchecked`.

**Thứ tự ưu tiên khi một kiểu cần Sendable:**
1. Đổi sang `struct`/`enum` (value type) — thường là đủ.
2. Làm class bất biến: `final` + toàn `let`.
3. Gắn vào actor: `@MainActor` (cho thứ liên quan UI) hoặc đổi thành `actor`.
4. Lock nội bộ + `@unchecked Sendable` (hoặc `Mutex`).
5. Cuối cùng mới là `nonisolated(unsafe)` / `@preconcurrency` (lối thoát tạm thời khi migrate).

### §2. `@Sendable` closure

Closure **capture** biến từ bên ngoài. Nếu closure chạy song song ở domain khác, những thứ nó capture cũng đi qua ranh giới. `@Sendable` trên kiểu hàm nghĩa là: *closure này có thể được gọi đồng thời từ nhiều nơi*, nên compiler bắt nó:
- **không** được capture `var` rồi sửa (capture `let` thì được),
- **không** được capture giá trị non-Sendable.

```swift
// Swift 6 mode
func demo() async {
    var count = 0
    let work: @Sendable () -> Void = { count += 1 }
    // ❌ error: mutation of captured var 'count' in concurrently-executing code
}

class Cache { var items: [String] = [] }
func demo2() {
    let c = Cache()
    let work: @Sendable () -> Void = { c.items.append("a") }
    // ❌ error: capture of 'c' with non-Sendable type 'Cache' in a '@Sendable' closure
}
```

**Bạn gặp `@Sendable` ở đâu?** `TaskGroup.addTask`, `Task.detached` (đời cũ), `DispatchQueue.async` khi bật strict concurrency, `AsyncStream.Continuation.onTermination`, completion handler của API mới (`@escaping @Sendable (Result<...>) -> Void`).

> **Chi tiết Swift 6:** closure của `Task { }` và `Task.detached { }` giờ được khai báo là **`sending`** chứ không phải `@Sendable`. Nghĩa là: được capture thứ non-Sendable **miễn là** bạn không dùng nó nữa ở chỗ cũ (xem §4). Vì thế nhiều code cũ "tự nhiên" hết lỗi khi lên Swift 6.

**Hàm/method cũng có thể là `@Sendable`**: method của `struct` Sendable, hàm global... Đây là lý do bạn truyền được `Repo.init` hay một global function vào `addTask`.

### §3. Swift 6 strict concurrency — 5 lỗi kinh điển và cách sửa

**Bối cảnh:**
| | Swift 5 mode (mặc định cũ) | Swift 5 mode + `Strict Concurrency Checking = Complete` | **Swift 6 language mode** |
|---|---|---|---|
| Kiểm tra data race | Rất ít | Đầy đủ, nhưng chỉ là **warning** | Đầy đủ, là **error** |
| Dùng khi | Code cũ | **Giai đoạn migrate** | Đích đến |

Xcode: Build Settings → *Swift Language Version* = Swift 6 (hoặc *Strict Concurrency Checking* = Complete khi còn ở Swift 5). SwiftPM: `swiftLanguageModes: [.v6]` (tools 6.0+).

> Swift 6 language mode **không phải** "compiler Swift 6". Compiler 6.x vẫn build được code Swift 5 mode. Language mode là "luật chơi" bạn chọn cho từng target.

#### Lỗi 1 — Global / static mutable state

```swift
// Swift 6 mode (đã kiểm tra)
import Foundation

// var requestCount = 0
// ❌ error: var 'requestCount' is not concurrency-safe because it is nonisolated global shared mutable state

let maxRetry = 3                             // ✅ cách 1: đổi thành let (kiểu Sendable)
@MainActor var requestCount = 0              // ✅ cách 2: gắn vào global actor
nonisolated(unsafe) var legacyFlag = false   // ✅ cách 3 (lối thoát): tự chịu trách nhiệm, phải có lý do
actor RequestCounter {                       // ✅ cách 4: đưa state vào actor
    static let shared = RequestCounter()
    private var count = 0
    func increment() -> Int { count += 1; return count }
}

struct Formatter {
    // static var shared = DateFormatter()
    // ❌ error: static property 'shared' is not concurrency-safe because it is nonisolated global shared mutable state

    // static let iso = ISO8601DateFormatter()
    // ❌ error: static property 'iso' is not concurrency-safe because non-'Sendable' type 'ISO8601DateFormatter' may have shared mutable state

    static let locale = Locale(identifier: "vi_VN")    // ✅ static let + kiểu Sendable
    @MainActor static let display = DateFormatter()    // ✅ hoặc cô lập vào MainActor
}
```
→ Điểm hay bị hỏi: **`static let` chưa chắc đã an toàn** — `let` chỉ khoá *tham chiếu*; nếu kiểu là class non-Sendable (như `DateFormatter`) thì vẫn có thể bị sửa từ nhiều thread. Singleton `static let shared = MyManager()` sẽ lỗi nếu `MyManager` không Sendable.

#### Lỗi 2 — Main actor isolation

```swift
@MainActor
final class ProfileViewModel {
    var name = ""
    func render() { print("render", name) }
}

// func update(vm: ProfileViewModel) { vm.name = "An" }
// ❌ error: main actor-isolated property 'name' can not be mutated from a nonisolated context
// (gọi method thì: call to main actor-isolated instance method 'render()' in a synchronous nonisolated context)

@MainActor func updateA(vm: ProfileViewModel) { vm.name = "An"; vm.render() }   // ✅ cách 1: hàm gọi cũng @MainActor
func updateB(vm: ProfileViewModel) async {                                    // ✅ cách 2: async + nhảy lên main
    await MainActor.run { vm.name = "Bình" }
    await vm.render()
}
```
Quy tắc: từ code **đồng bộ** không thể "nhảy" sang actor khác. Hoặc bạn đã ở sẵn trên MainActor, hoặc bạn phải `async` và `await`.

#### Lỗi 3 — `sending 'x' risks causing data races`

```swift
final class Draft { var text = "" }          // non-Sendable
actor Uploader {
    func upload(_ draft: Draft) { print("upload:", draft.text) }
    func upload(text: String) { print("upload:", text) }
}

func save(uploader: Uploader) async {
    let draft = Draft()
    draft.text = "hello"
    await uploader.upload(draft)             // ✅ OK: sau dòng này draft không còn được dùng
    // draft.text = "edit"
    // ❌ nếu bỏ comment: error: sending 'draft' risks causing data races
    //    note: sending 'draft' to actor-isolated instance method 'upload' risks causing data races
    //          between actor-isolated and local nonisolated uses
}

func saveCopy(uploader: Uploader) async {
    let draft = Draft()
    draft.text = "copy"
    await uploader.upload(text: draft.text)  // ✅ chỉ gửi String (Sendable), draft vẫn là của ta
    draft.text = "vẫn sửa được"
}
```
Cách sửa, theo thứ tự ưu tiên: (a) đừng dùng lại giá trị sau khi gửi; (b) chỉ gửi phần Sendable (`String`, struct); (c) đổi kiểu sang `struct`/Sendable; (d) để actor tự tạo object.

#### Lỗi 4 — Capture non-Sendable trong `@Sendable` closure
Xem §2. Sửa: capture bản copy `let` (`[value] in`), đổi kiểu sang Sendable, hoặc dùng `Task { }` (closure `sending`) thay vì API yêu cầu `@Sendable`.

#### Lỗi 5 — Protocol conformance "vượt" sang main actor

Rất hay gặp với delegate: class UI là `@MainActor`, protocol thì không.
```swift
protocol DataDelegate: AnyObject { func didReceive(_ s: String) }

// @MainActor final class Screen: DataDelegate {
//     var text = ""
//     func didReceive(_ s: String) { text = s }
// }
// ❌ error: conformance of 'Screen' to protocol 'DataDelegate' crosses into main actor-isolated code and can cause data races
//    note: main actor-isolated instance method 'didReceive' cannot satisfy nonisolated requirement

// ✅ Cách 1 (Swift 6.2+): isolated conformance — conformance chỉ dùng được trên MainActor
@MainActor final class Screen2: @MainActor DataDelegate {
    var text = ""
    func didReceive(_ s: String) { text = s }
}

// ✅ Cách 2: nonisolated + khẳng định đang ở main (crash nếu sai — dùng khi SDK đảm bảo gọi trên main)
@MainActor final class Screen: DataDelegate {
    var text = ""
    nonisolated func didReceive(_ s: String) {
        MainActor.assumeIsolated { text = s }
    }
}
```
Cách 3: nếu protocol do bạn viết và chỉ dùng cho UI → gắn `@MainActor` cho cả protocol.

> **Mẹo đọc lỗi:** Swift 6 báo lỗi **theo đợt**. Lỗi type-check (1, 2, 4, 5) hiện trước; lỗi `sending` (3) do một pass phân tích luồng chạy **sau** — sửa xong đợt 1 thì đợt 2 mới hiện. Đừng hoảng khi "sửa 2 lỗi lại ra 1 lỗi mới".

### §4. Region-based isolation & `sending` (giới thiệu)

Trước Swift 6, luật đơn giản: *non-Sendable thì không được qua ranh giới, hết*. Quá chặt — ví dụ tạo một object mới, cấu hình, rồi giao hẳn cho actor là hoàn toàn an toàn, nhưng vẫn bị cấm.

**Region-based isolation (SE-0414)**: compiler theo dõi các **vùng (region)** giá trị non-Sendable có liên kết với nhau. Một region có thể:
- **disconnected** — chưa dính tới actor nào (vd. object vừa `init` trong hàm),
- thuộc về một actor / MainActor.

Một region disconnected **được phép chuyển** sang domain khác, với điều kiện **sau đó không ai ở domain cũ đụng vào nữa**. Đó chính là lý do `save(uploader:)` ở Lỗi 3 compile được, còn thêm `draft.text = "edit"` thì lỗi.

```
let draft = Draft()          region R = {draft}   (disconnected)
draft.text = "hello"         vẫn R
await uploader.upload(draft) R được "chuyển" sang actor Uploader
draft.text = "edit"          ❌ dùng R sau khi đã chuyển → lỗi
```

**`sending` (SE-0430)** là từ khoá để **hàm khai báo** điều đó trong signature: tham số `sending` = "tôi nhận quyền sở hữu, caller không được dùng tiếp"; kết quả `sending` = "giá trị trả ra là region mới, caller muốn gửi đi đâu thì gửi".
```swift
// Swift 6 mode
final class Payload { var bytes: [UInt8] = [] }

func makePayload() -> sending Payload {
    let p = Payload()
    p.bytes = [1, 2, 3]
    return p
}
actor Sink {
    var stored: [Payload] = []
    func take(_ p: sending Payload) { stored.append(p) }
}
// let p = makePayload(); await sink.take(p)   ✅ — non-Sendable nhưng được chuyển giao an toàn
```
Bạn không cần viết `sending` thường xuyên; cần **hiểu** để đọc lỗi và để biết vì sao `Task { }`, `CheckedContinuation.resume(returning:)` dùng nó.

### §5. Migrate một codebase lên Swift 6 — từng bước

Apple khuyến nghị đi **từ dưới lên**, **từng module/target một**:

1. **Bật cảnh báo trước:** giữ Swift 5 mode, đặt *Strict Concurrency Checking* = `Targeted` → sửa → `Complete` → sửa. Mọi thứ vẫn là warning nên app vẫn build/release được.
2. **Bắt đầu từ module lá** (model, utilities, networking) — ít phụ thuộc nhất. Model → làm `struct` + `Sendable`.
3. **Gắn `@MainActor`** cho mọi thứ thuộc UI: ViewController, ViewModel, Coordinator. (UIKit `UIViewController`, `UIView` đã là `@MainActor` sẵn.)
4. **Xử lý global/singleton:** `let`, `@MainActor`, hoặc đổi thành `actor`.
5. **Service có state dùng chung** (cache, token store) → `actor`, hoặc lock + `@unchecked Sendable` nếu cần API đồng bộ.
6. **Callback/delegate cũ** → bọc bằng `withCheckedContinuation` / `AsyncStream` (§6).
7. **Thư viện bên thứ ba chưa hỗ trợ** → `@preconcurrency import ThatModule` để hạ lỗi của *riêng module đó* xuống warning/ẩn.
8. Khi target không còn warning → chuyển *Swift Language Version* = 6. Lặp lại với module tiếp theo.

**Swift 6.2 (Xcode 26) — "Approachable Concurrency":** để bớt đau đầu, Swift 6.2 thêm:
- **Default actor isolation = MainActor** (`-default-isolation MainActor`, Xcode: *Default Actor Isolation*): trong module đó, mọi thứ không ghi gì **mặc định là `@MainActor`**. Project app mới tạo bằng Xcode 26 bật sẵn. Hợp với app target (phần lớn code là UI); không hợp với module networking/thư viện.
- **`nonisolated(nonsending)` mặc định** cho hàm `async` không cô lập: hàm chạy **trên actor của caller** thay vì luôn nhảy sang global pool như Swift 6.0/6.1.
- **`@concurrent`**: muốn hàm async chắc chắn chạy song song ngoài actor (vd. decode ảnh nặng) thì đánh dấu `@concurrent`.
```swift
@concurrent
func decode(_ data: [UInt8]) async -> Int { data.reduce(0) { $0 + Int($1) } }   // luôn chạy trên global executor
```
> Phỏng vấn năm 2027 rất có thể hỏi: *"Project bạn bật default MainActor isolation chưa? Thế thì hàm nặng chạy ở đâu?"* → trả lời: mặc định trên main; việc nặng tách ra hàm `@concurrent` hoặc `nonisolated` type riêng.

### §6. AsyncSequence & AsyncStream

**`AsyncSequence`** giống `Sequence` nhưng mỗi phần tử có thể **phải chờ**. Duyệt bằng `for await` (hoặc `for try await` nếu có thể throw). Khi `next()` trả `nil` → kết thúc. Có sẵn `map`, `filter`, `prefix`, `first(where:)`, `reduce`...

Bạn đã dùng mà không để ý: `URLSession.shared.bytes(from:)`, `NotificationCenter.default.notifications(named:)`, `FileHandle.bytes.lines`, `publisher.values` (Combine → async).

```swift
// Swift 6 mode — compile & chạy được
import Foundation

// 1. AsyncSequence tự viết: chỉ cần makeAsyncIterator() + next()
struct Countdown: AsyncSequence {
    typealias Element = Int
    let start: Int
    struct AsyncIterator: AsyncIteratorProtocol {
        var current: Int
        mutating func next() async -> Int? {
            guard current > 0 else { return nil }    // nil = kết thúc sequence
            defer { current -= 1 }
            return current
        }
    }
    func makeAsyncIterator() -> AsyncIterator { AsyncIterator(current: start) }
}

for await n in Countdown(start: 3) { print("countdown", n) }       // 3, 2, 1
let evens = Countdown(start: 6).filter { $0 % 2 == 0 }.map { $0 * 10 }
for await x in evens { print("mapped", x) }                         // 60, 40, 20
```

Viết iterator bằng tay hiếm khi cần. Trong thực tế dùng **`AsyncStream`** — một "ống" mà bạn **đẩy** giá trị vào bằng `continuation.yield(_:)`, kết thúc bằng `continuation.finish()`. Đây là công cụ chính để **bọc API delegate/callback phát nhiều lần** (location, Bluetooth, WebSocket, progress download...).

| Cần bọc | Dùng |
|---|---|
| Callback **một lần** (completion handler) | `withCheckedContinuation` / `withCheckedThrowingContinuation` (tuần 7) |
| Callback/delegate **nhiều lần** | `AsyncStream` / `AsyncThrowingStream` |

```swift
// Swift 6 mode — Bọc một API delegate cũ
protocol TemperatureSensorDelegate: AnyObject {
    func sensor(_ sensor: TemperatureSensor, didRead value: Double)
    func sensorDidStop(_ sensor: TemperatureSensor)
}

// Giả lập class của SDK cũ. Nó tự đảm bảo thread-safe nên ta đánh dấu @unchecked.
final class TemperatureSensor: @unchecked Sendable {
    weak var delegate: TemperatureSensorDelegate?
    func start() {
        DispatchQueue.global().async {
            for v in [20.5, 21.0, 21.7] {
                Thread.sleep(forTimeInterval: 0.05)
                self.delegate?.sensor(self, didRead: v)
            }
            self.delegate?.sensorDidStop(self)
        }
    }
    func stop() { /* giải phóng phần cứng */ }
}

final class SensorBridge: TemperatureSensorDelegate {
    private let sensor = TemperatureSensor()
    private var continuation: AsyncStream<Double>.Continuation?

    func readings() -> AsyncStream<Double> {
        let (stream, continuation) = AsyncStream.makeStream(of: Double.self,
                                                             bufferingPolicy: .bufferingNewest(10))
        self.continuation = continuation
        continuation.onTermination = { [sensor] _ in sensor.stop() }  // dọn dẹp khi bị huỷ/kết thúc
        sensor.delegate = self
        sensor.start()
        return stream
    }
    func sensor(_ sensor: TemperatureSensor, didRead value: Double) { continuation?.yield(value) }
    func sensorDidStop(_ sensor: TemperatureSensor) { continuation?.finish() }
}

let bridge = SensorBridge()
for await t in bridge.readings() { print("temp", t) }   // temp 20.5 / temp 21.0 / temp 21.7
print("stream finished")
```
> Thử bỏ `@unchecked Sendable` ở `TemperatureSensor` rồi compile Swift 6: bạn sẽ gặp `error: capture of 'sensor' with non-Sendable type 'TemperatureSensor' in a '@Sendable' closure` — vì `onTermination` là `@Sendable`. Đây là lỗi **rất thật** khi bọc SDK cũ.

**4 điều phải nhớ về AsyncStream:**
1. **`onTermination`** được gọi khi stream `finish()` **hoặc** consumer bị huỷ (task cancel, `break` khỏi vòng lặp và stream bị giải phóng). Đây là chỗ gọi `stop()`, `removeObserver`, `invalidate()`.
2. **Buffering policy:** `.unbounded` (mặc định — producer nhanh hơn consumer thì RAM tăng mãi), `.bufferingNewest(n)` (giữ n giá trị mới nhất — hợp với location/sensor), `.bufferingOldest(n)`.
3. **Chỉ một consumer.** `AsyncStream` không phải "broadcast" như Combine Subject; hai vòng `for await` trên cùng stream sẽ **chia nhau** phần tử (hành vi không xác định). Cần nhiều subscriber → Combine, hoặc tạo stream riêng cho mỗi subscriber.
4. **Gọi `finish()`** khi nguồn kết thúc, nếu không vòng `for await` treo mãi.

### §7. Combine cơ bản

Combine (iOS 13+) là framework **reactive** của Apple: mô hình dữ liệu là **luồng giá trị theo thời gian**.

```
 Publisher ──subscribe──► Subscriber
     │  ◄──── Subscription (request demand: .unlimited / .max(n))
     │ ─────── value, value, value ... ───────►
     │ ─────── completion (.finished | .failure(Error)) ──►   (sau completion không còn value)
```
- **`Publisher<Output, Failure>`**: phát ra 0..n giá trị kiểu `Output`, rồi có thể kết thúc bằng `.finished` hoặc `.failure(Failure)`. `Failure == Never` nghĩa là không bao giờ lỗi.
- **`Subscriber`**: nhận giá trị. Bạn gần như chỉ dùng 2 subscriber có sẵn: **`sink`** và **`assign`**.
- **Subscription** có **back-pressure**: subscriber nói "cho tôi n giá trị" (*demand*). `sink`/`assign` luôn xin `.unlimited`.
- **Không có subscriber thì không có gì chạy** (hầu hết publisher là "lazy"/cold).

**Publisher hay gặp:** `Just(x)`, `[1,2,3].publisher`, `Future` (một giá trị, async), `URLSession.dataTaskPublisher(for:)`, `NotificationCenter.default.publisher(for:)`, `Timer.publish(every:on:in:)`, `@Published`.

**Subject** — publisher mà bạn tự `send` giá trị vào (cầu nối code imperative → Combine):
| | `PassthroughSubject` | `CurrentValueSubject` |
|---|---|---|
| Giữ giá trị hiện tại? | Không | Có (`.value`) |
| Subscriber mới nhận gì? | Chỉ giá trị **sau khi** subscribe | Nhận **ngay** giá trị hiện tại, rồi các giá trị sau |
| Dùng cho | Sự kiện (tap, "đã lưu") | Trạng thái (đang loading, user hiện tại) |

**`@Published`** là property wrapper: `$property` là một publisher (giống `CurrentValueSubject`), **phát giá trị trong `willSet`** — tức là khi sink chạy, thuộc tính *chưa* được gán giá trị mới. Bẫy hay gặp: đọc `self.property` bên trong sink sẽ ra giá trị **cũ**; hãy dùng tham số của closure.

```swift
// Swift 6 mode — compile & chạy được (Combine có trên macOS nên chạy bằng `swift` được)
import Foundation
import Combine

var bag = Set<AnyCancellable>()

[1, 2, 3, 4, 5].publisher
    .filter { $0 % 2 == 1 }
    .map { $0 * 10 }
    .sink(receiveCompletion: { print("completion:", $0) },
          receiveValue: { print("value:", $0) })
    .store(in: &bag)
// value: 10 / value: 30 / value: 50 / completion: finished

let passthrough = PassthroughSubject<String, Never>()
let current = CurrentValueSubject<Int, Never>(0)
passthrough.sink { print("passthrough:", $0) }.store(in: &bag)
current.sink { print("current:", $0) }.store(in: &bag)     // in ngay "current: 0"
passthrough.send("hello")                                  // passthrough: hello
current.send(1)                                            // current: 1
print("current.value =", current.value)                    // current.value = 1
```

**`AnyCancellable` và bộ nhớ:**
- `sink`/`assign(to:on:)` trả về `AnyCancellable`. **Subscription sống chừng nào `AnyCancellable` còn sống**; khi nó bị `deinit` → tự `cancel()`.
- Quên giữ lại (`_ = publisher.sink {...}`) → subscription bị huỷ **ngay lập tức**, bạn không nhận được gì (lỗi rất hay gặp).
- Giữ trong `Set<AnyCancellable>` là thuộc tính của object → subscription sống cùng object.

```swift
// Swift 5 mode — chạy được
import Combine

final class Leaky {
    @Published var text = ""
    var upper = ""
    private var bag = Set<AnyCancellable>()
    init() {
        $text.map { $0.uppercased() }
            .assign(to: \.upper, on: self)      // assign(to:on:) giữ STRONG self
            .store(in: &bag)                    // self → bag → subscription → self  ⇒ retain cycle
    }
    deinit { print("Leaky deinit") }
}

final class Fixed {
    @Published var text = ""
    @Published var upper = ""
    private var bag = Set<AnyCancellable>()
    init() {
        $text.map { $0.uppercased() }
            .assign(to: &$upper)                // cách 1: assign(to: &$prop) — subscription gắn với @Published, không trả AnyCancellable
        $text.sink { [weak self] value in       // cách 2: sink + [weak self]
            self?.log(value)
        }.store(in: &bag)
    }
    func log(_ v: String) {}
    deinit { print("Fixed deinit") }
}

do { _ = Leaky() }     // không in gì → LEAK
do { _ = Fixed() }     // Fixed deinit

let subject = PassthroughSubject<Int, Never>()
var token: AnyCancellable? = subject.sink { print("got", $0) }
subject.send(1)        // got 1
token = nil            // AnyCancellable deinit → cancel
subject.send(2)        // không in gì
```
→ Đây là câu hỏi **rất hay gặp**: "Combine gây retain cycle ở đâu?" — `sink { self.... }` không `[weak self]`, và `assign(to:on: self)`.

### §8. Operator & Scheduler

| Nhóm | Operator | Ý nghĩa |
|---|---|---|
| Biến đổi | `map`, `compactMap`, `tryMap`, `scan` | Như trên collection; `scan` = reduce nhưng phát từng bước |
| Lọc | `filter`, `removeDuplicates`, `dropFirst`, `first`, `prefix` | `removeDuplicates` chỉ bỏ giá trị **trùng liền kề** |
| Thời gian | `debounce`, `throttle`, `delay`, `timeout` | `debounce`: chờ "im lặng" X ms rồi mới phát giá trị cuối; `throttle`: tối đa 1 giá trị mỗi X ms |
| Kết hợp | `combineLatest`, `merge`, `zip` | `combineLatest`: phát khi **bất kỳ** nguồn nào đổi (sau khi mỗi nguồn có ≥ 1 giá trị); `zip`: ghép **cặp** theo thứ tự |
| Lồng publisher | `flatMap`, `switchToLatest` | `flatMap` giữ **mọi** publisher con; `map + switchToLatest` **huỷ** cái cũ khi có cái mới |
| Lỗi | `catch`, `replaceError(with:)`, `retry(n)`, `mapError` | Sau khi lỗi, upstream đã **kết thúc** — muốn sống tiếp phải `catch` *bên trong* `flatMap` |
| Thread | `receive(on:)`, `subscribe(on:)` | Xem dưới |
| Khác | `eraseToAnyPublisher()`, `share()`, `handleEvents` | Ẩn kiểu cụ thể; chia sẻ 1 subscription cho nhiều subscriber |

**Ví dụ search kinh điển** (đã chạy, Swift 6 mode):
```swift
final class SearchViewModel {
    @Published var query = ""
    @Published private(set) var results: [String] = []
    private let all = ["swift", "swiftui", "combine", "concurrency", "uikit"]

    init(scheduler: DispatchQueue = .main) {
        $query
            .map { $0.trimmingCharacters(in: .whitespaces).lowercased() }
            .debounce(for: .milliseconds(300), scheduler: scheduler)   // gõ liên tục → chỉ lấy lần cuối
            .removeDuplicates()                                         // "swift" và "swift " → cùng là "swift"
            .map { [all] q in q.isEmpty ? [] : all.filter { $0.hasPrefix(q) } }
            .assign(to: &$results)
    }
}
// Gõ "s","sw","swi","swift" cách nhau 50ms → chỉ 1 lần: results: ["swift", "swiftui"]
// 600ms sau gõ "swift " → trim thành "swift" → removeDuplicates chặn → không in gì
// gõ "co" → results: ["combine", "concurrency"]
```
Chú ý thứ tự: `debounce` **trước** `removeDuplicates`, và `map` chuẩn hoá (trim/lowercase) **trước** cả hai.

**`flatMap` vs `switchToLatest`** — câu hỏi kinh điển cho search:
```swift
// Swift 6 mode — compile & chạy được
func search(_ q: String, delayMs: Int) -> AnyPublisher<String, Never> {
    Just("kết quả cho \(q)")
        .delay(for: .milliseconds(delayMs), scheduler: DispatchQueue.global())
        .eraseToAnyPublisher()
}
let queries = PassthroughSubject<(String, Int), Never>()

queries.flatMap { q, d in search(q, delayMs: d) }              // giữ MỌI request
    .receive(on: DispatchQueue.main)
    .sink { print("flatMap:", $0) }.store(in: &bag)

queries.map { q, d in search(q, delayMs: d) }.switchToLatest() // request mới → huỷ request cũ
    .receive(on: DispatchQueue.main)
    .sink { print("switchToLatest:", $0) }.store(in: &bag)

queries.send(("sw", 300))      // request chậm, gửi trước
queries.send(("swift", 50))    // request nhanh, gửi sau
// Kết quả thường thấy (thời gian phụ thuộc scheduler, thứ tự 2 dòng đầu có thể đổi):
// flatMap: kết quả cho swift
// switchToLatest: kết quả cho swift
// flatMap: kết quả cho sw              ← kết quả CŨ về sau, ghi đè kết quả mới → bug UI!
// switchToLatest không bao giờ in "sw" vì request đó đã bị huỷ.
```

**Scheduler** = "chạy ở đâu và khi nào": `DispatchQueue.main`, `RunLoop.main`, `OperationQueue`, `ImmediateScheduler`.
- **`receive(on:)`**: mọi thứ **phía dưới** (downstream) nó chạy trên scheduler đó. Dùng `receive(on: DispatchQueue.main)` ngay trước `sink`/`assign` cập nhật UI.
- **`subscribe(on:)`**: ảnh hưởng tới **phía trên** — nơi thực hiện việc subscribe/request (và với publisher đồng bộ, nơi phát giá trị). Ít dùng; `URLSession` publisher đã tự chạy nền.
- `DispatchQueue.main` vs `RunLoop.main`: `RunLoop.main` (mode default) **bị hoãn khi user đang scroll** (tuần 9: RunLoop mode). Cập nhật UI thường dùng `DispatchQueue.main`.

**Cầu nối hai thế giới:**
```swift
let numbers = [1, 2, 3].publisher
Task {
    var sum = 0
    for await n in numbers.values { sum += n }     // Combine → AsyncSequence
    print("sum via values:", sum)                  // sum via values: 6
}
// async → Combine: bọc trong Future { promise in Task { promise(.success(await work())) } }
```

### §9. Combine vs async/await — chọn cái nào?

| Tiêu chí | Combine | async/await + AsyncSequence |
|---|---|---|
| Bản chất | Luồng giá trị theo thời gian, **push**, nhiều subscriber | Code tuần tự; AsyncSequence là **pull**, 1 consumer |
| Một kết quả (gọi API) | Dài dòng (`Future`, `AnyPublisher`) | **Tự nhiên nhất** |
| Toán tử thời gian (debounce, throttle, combineLatest) | **Có sẵn** | Không có trong stdlib → dùng package `swift-async-algorithms` hoặc tự viết |
| Huỷ | `AnyCancellable` | `Task.cancel()`, structured concurrency tự huỷ con |
| Kiểm tra data race lúc compile | Gần như không (không thiết kế cho Sendable) | **Có** (Sendable, actor, Swift 6) |
| Đọc / debug | Stack trace khó đọc, kiểu generic dài | Như code thường |
| Tương lai | Apple **không phát triển thêm** từ ~2020; SwiftUI chuyển sang `@Observable` (iOS 17) | Hướng chính của Swift |

**Câu trả lời mẫu:** *"Code mới mình dùng async/await cho mọi tác vụ một lần (network, DB) và actor cho state. Combine mình dùng khi bài toán thật sự là luồng sự kiện cần kết hợp theo thời gian — debounce search, validate form bằng combineLatest — hoặc khi codebase cũ đã dùng Combine/`@Published` với `ObservableObject`. Hai thế giới nối với nhau bằng `.values` và `Future`."*

---

## ⚠️ Bẫy hay gặp

1. **"Class final chỉ có `let` là tự Sendable"** — sai. Class **không bao giờ** tự động Sendable; phải khai báo. Struct `public` cũng phải khai báo.
2. **`static let shared = Manager()` không tự an toàn** — nếu `Manager` non-Sendable, Swift 6 báo lỗi. `let` chỉ khoá tham chiếu.
3. **`@unchecked Sendable` không phải "nút tắt lỗi"** — nó là lời hứa bạn đã tự đồng bộ hoá. Không có lock mà dùng `@unchecked` = data race chờ nổ.
4. **`MainActor.assumeIsolated` crash** nếu thực tế không ở main thread — chỉ dùng khi API đảm bảo gọi trên main.
5. **`nonisolated(unsafe)` / `@preconcurrency`** là công cụ migrate, không phải giải pháp lâu dài. Phỏng vấn mà nói "mình thêm `nonisolated(unsafe)` cho hết lỗi" là mất điểm.
6. **AsyncStream quên `finish()`** → `for await` treo mãi, task không bao giờ xong. **Quên `onTermination`** → sensor/observer không được dừng.
7. **AsyncStream chỉ có 1 consumer**, không broadcast như Subject.
8. **Không giữ `AnyCancellable`** → subscription chết ngay, sink không bao giờ chạy.
9. **`@Published` phát trong `willSet`** → đọc `self.prop` trong sink ra giá trị cũ.
10. **`flatMap` cho search** → kết quả cũ về sau ghi đè kết quả mới. Dùng `map + switchToLatest`.
11. **Lỗi trong Combine kết thúc cả pipeline** — `catch` ở ngoài cùng thì sau lỗi đầu tiên ô search "chết" luôn. Đặt `catch`/`replaceError` **bên trong** `flatMap`/`map` của từng request.
12. **`removeDuplicates` chỉ bỏ trùng liền kề**, không phải "unique toàn bộ".
13. **Combine + Swift 6 có thể crash lúc runtime.** Closure `sink` viết trong ngữ cảnh `@MainActor` (ViewController, ViewModel `@MainActor`, top-level code) được compiler coi là main-actor-isolated; Swift 6 chèn kiểm tra isolation lúc chạy. Nếu publisher phát giá trị trên thread nền (`Future` + `Task`, `URLSession.dataTaskPublisher`, `delay(scheduler: .global())`) mà thiếu `.receive(on: DispatchQueue.main)` → app **crash** tại `dispatch_assert_queue` (đã thử với Swift 6.3). Ở Swift 5 mode cùng code đó chỉ là data race âm thầm.

---

## 🛠 Bài tập

> Tạo `practice/week08/`. Mọi file non-UI chạy bằng `swift -swift-version 6 file.swift`. **Mục tiêu tuần này: không có warning/error nào ở Swift 6 mode.**

**Bài 1 — Phân loại Sendable (T2)**
Không compile, đoán xem kiểu nào **được** dùng làm `T` trong `func check<T: Sendable>(_: T.Type) {}`. Sau đó viết file và compile để kiểm chứng.
```swift
import Foundation
struct Point { var x: Int; var y: Int }
struct Row { let cache: NSCache<NSString, NSString> }
final class Token { let value: String; init(value: String) { self.value = value } }
enum Event { case tap(Int), text(String) }
actor Store {}
@MainActor final class VM { var x = 0 }
public struct PublicPoint { var x: Int }
```
<details><summary>Lời giải</summary>

| Kiểu | Sendable? | Lý do |
|---|---|---|
| `Point` | ✅ | struct internal, mọi thuộc tính Sendable → tự suy luận |
| `Row` | ❌ | `NSCache` là class non-Sendable |
| `Token` | ❌ | class **không bao giờ** tự Sendable — phải viết `final class Token: Sendable` |
| `Event` | ✅ | enum với associated value Sendable |
| `Store` | ✅ | actor luôn Sendable |
| `VM` | ✅ | class gắn global actor là Sendable |
| `PublicPoint` | ❌ | type `public` không được suy luận, phải khai báo `: Sendable` |

Compiler (Swift 6.3) báo đúng 3 lỗi: `type 'Row' does not conform to the 'Sendable' protocol`, tương tự cho `Token` và `PublicPoint`.
</details>

**Bài 2 — Sửa lỗi Swift 6 (T3)**
File sau compile ở Swift 5 mode nhưng lỗi ở Swift 6. Sửa cho hết lỗi **mà không dùng** `nonisolated(unsafe)`, `@unchecked`, `@preconcurrency`. Ghi lại từng thông báo lỗi bạn gặp.
```swift
import Foundation

var cachedRepos: [String] = []

final class RepoListViewModel {
    var repos: [String] = []
}

@MainActor final class RepoListView {
    var title = ""
}

func refresh(view: RepoListView) async {
    let repos = await loadRepos()
    view.title = "\(repos.count) repos"
}

func loadRepos() async -> [String] { ["swift", "uikit"] }

actor Analytics {
    func track(_ vm: RepoListViewModel) {}
}

func report(analytics: Analytics) async {
    let vm = RepoListViewModel()
    await analytics.track(vm)
    vm.repos.append("x")
}
```
<details><summary>Lời giải</summary>

Đợt 1 (type-check):
- `var 'cachedRepos' is not concurrency-safe because it is nonisolated global shared mutable state`
- `main actor-isolated property 'title' can not be mutated from a nonisolated context`

Sửa xong đợt 1, **đợt 2** mới hiện:
- `sending 'vm' risks causing data races` — vì `vm` được gửi sang actor rồi vẫn bị sửa.

```swift
import Foundation

@MainActor var cachedRepos: [String] = []        // (1) hoặc đưa vào actor / bỏ hẳn biến global

struct RepoListViewModel: Sendable {              // (3) đổi sang struct → gửi đi là gửi bản copy
    var repos: [String] = []
}

@MainActor final class RepoListView {
    var title = ""
}

@MainActor                                         // (2) hàm cập nhật UI chạy trên MainActor
func refresh(view: RepoListView) async {
    let repos = await loadRepos()
    view.title = "\(repos.count) repos"
}

func loadRepos() async -> [String] { ["swift", "uikit"] }

actor Analytics {
    func track(_ vm: RepoListViewModel) {}
}

func report(analytics: Analytics) async {
    var vm = RepoListViewModel()
    await analytics.track(vm)
    vm.repos.append("x")                           // OK: actor nhận bản copy riêng
}
```
Cách khác cho (3): giữ class nhưng đổi thứ tự — sửa `vm` xong rồi mới gửi đi và không đụng lại nữa.
</details>

**Bài 3 — Bọc callback nhiều lần bằng AsyncThrowingStream (T4)**
Cho API cũ (không được sửa):
```swift
final class LegacyDownloader: @unchecked Sendable {
    func download(url: String,
                  progress: @escaping @Sendable (Double) -> Void,
                  completion: @escaping @Sendable (Result<Data, Error>) -> Void) {
        DispatchQueue.global().async {
            for p in [0.25, 0.5, 0.75, 1.0] {
                Thread.sleep(forTimeInterval: 0.02)
                progress(p)
            }
            if url.isEmpty { completion(.failure(URLError(.badURL))) }
            else { completion(.success(Data("ok".utf8))) }
        }
    }
}
```
Viết `func download(url: String) -> AsyncThrowingStream<DownloadEvent, Error>` với `enum DownloadEvent { case progress(Double), finished(Data) }`, rồi dùng `for try await` in tiến độ.
<details><summary>Lời giải</summary>

```swift
enum DownloadEvent: Sendable {
    case progress(Double)
    case finished(Data)
}

extension LegacyDownloader {
    func download(url: String) -> AsyncThrowingStream<DownloadEvent, Error> {
        AsyncThrowingStream { continuation in
            download(url: url,
                     progress: { continuation.yield(.progress($0)) },
                     completion: { result in
                         switch result {
                         case .success(let data):
                             continuation.yield(.finished(data))
                             continuation.finish()
                         case .failure(let error):
                             continuation.finish(throwing: error)
                         }
                     })
        }
    }
}

let d = LegacyDownloader()
do {
    for try await event in d.download(url: "https://x") {
        switch event {
        case .progress(let p): print("progress", p)
        case .finished(let data): print("done", data.count, "bytes")
        }
    }
    for try await _ in d.download(url: "") {}
} catch {
    print("error:", (error as? URLError)?.code == .badURL)
}
// progress 0.25 / progress 0.5 / progress 0.75 / progress 1.0 / done 2 bytes / error: true
```
`Continuation` là Sendable nên capture được vào closure `@Sendable`. Lỗi thì `finish(throwing:)` — vòng `for try await` sẽ ném lỗi ra.
</details>

**Bài 4 — Stream có dọn dẹp (T4)**
Viết `func ticks(every interval: Duration) -> AsyncStream<Int>` phát 1, 2, 3... mãi mãi. Consumer `break` khi nhận 3. Chứng minh rằng task sinh số **đã dừng** (in ra trong `onTermination`).
<details><summary>Lời giải</summary>

```swift
func ticks(every interval: Duration) -> AsyncStream<Int> {
    AsyncStream { continuation in
        let task = Task {
            var n = 0
            while !Task.isCancelled {
                try? await Task.sleep(for: interval)
                n += 1
                continuation.yield(n)
            }
        }
        continuation.onTermination = { reason in
            print("onTermination:", reason)
            task.cancel()
        }
    }
}

func consume() async {
    for await t in ticks(every: .milliseconds(20)) {
        print("tick", t)
        if t == 3 { break }      // thoát vòng lặp → stream bị giải phóng → onTermination(.cancelled)
    }
}
await consume()
// tick 1 / tick 2 / tick 3 / onTermination: cancelled
```
Chú ý: đặt vòng lặp **trong một hàm**. Ở top-level script, stream có thể sống tới cuối chương trình nên `onTermination` không chạy ngay — đúng là "stream bị giải phóng" mới kích hoạt cancel.
</details>

**Bài 5 — Validate form bằng Combine (T5)**
`SignUpViewModel` có `@Published email, password, confirm` và `@Published private(set) var isSubmitEnabled`. Bật nút khi email chứa `@`, password ≥ 8 ký tự và `confirm == password`. Không được tạo retain cycle.
<details><summary>Lời giải</summary>

```swift
import Combine

final class SignUpViewModel {
    @Published var email = ""
    @Published var password = ""
    @Published var confirm = ""
    @Published private(set) var isSubmitEnabled = false

    init() {
        Publishers.CombineLatest3($email, $password, $confirm)
            .map { email, pw, confirm in
                email.contains("@") && pw.count >= 8 && pw == confirm
            }
            .removeDuplicates()
            .assign(to: &$isSubmitEnabled)
    }
}

let vm = SignUpViewModel()
let c = vm.$isSubmitEnabled.sink { print("enabled:", $0) }
vm.email = "a@b.com"
vm.password = "12345678"
vm.confirm = "1234567"
vm.confirm = "12345678"
vm.password = "x"
c.cancel()
// enabled: false / enabled: true / enabled: false
```
`removeDuplicates` giúp UI không bị cập nhật thừa khi giá trị không đổi.
</details>

**Bài 6 — Tìm leak (T7)**
Trong project cũ, tìm (hoặc tự tạo) một ViewController có `viewModel.$items.sink { self.tableView.reloadData() }.store(in: &cancellables)`. Push rồi pop màn hình đó, mở **Memory Graph Debugger** (tuần 5) chứng minh VC không được giải phóng. Sửa bằng `[weak self]` và kiểm tra lại `deinit`. Viết 3 dòng giải thích chuỗi tham chiếu.
<details><summary>Gợi ý</summary>

Chuỗi: `VC → cancellables (Set) → AnyCancellable → Subscription → closure sink → self (VC)`. Cắt bằng `[weak self]`. Bài này cũng là một phần của **M2**.
</details>

**Bài 7 — Viết lại hai chiều (T7)**
a) Viết lại pipeline search ở §8 (debounce + removeDuplicates + switchToLatest) bằng async/await **không dùng Combine**. (Gợi ý: mỗi lần query đổi → huỷ `Task` cũ, `Task.sleep` 300ms rồi mới gọi API.)
b) Ngược lại, bọc `func fetchUser() async throws -> String` thành `AnyPublisher<String, Error>` bằng `Future`.
<details><summary>Lời giải (b)</summary>

```swift
func fetchUserPublisher() -> AnyPublisher<String, Error> {
    Deferred {                                 // Deferred: chỉ chạy khi có subscriber (Future thì chạy NGAY khi tạo)
        Future { promise in
            nonisolated(unsafe) let promise = promise   // Promise không Sendable — Combine ra đời trước Swift Concurrency
            Task {
                do { promise(.success(try await fetchUser())) }
                catch { promise(.failure(error)) }
            }
        }
    }
    .eraseToAnyPublisher()
}
```
Không có dòng `nonisolated(unsafe)`, Swift 6 báo: `passing closure as a 'sending' parameter risks causing data races ...` (`note: closure captures 'promise' which is accessible to code in the current task`). Đây là ví dụ điển hình cho "Combine không được thiết kế cho Sendable".

Khi subscribe ở top-level / trong code `@MainActor` (Swift 6 mode), nhớ `.receive(on: DispatchQueue.main)` trước `sink` — xem Bẫy 13.

Lời giải (a) chính là Bài tập lớn bên dưới.
</details>

**🏋️ Bài tập lớn (T7) — Search có debounce bằng AsyncStream (console)**
Viết `search_debounce.swift` (Swift 6 mode, không warning):
- `struct Repo: Sendable, Equatable`, `protocol RepoSearching: Sendable` với `func search(_:) async throws -> [Repo]`, một `FakeGitHubAPI` giả lập delay 100ms.
- `@MainActor final class SearchViewModel` nhận text qua `queryChanged(_:)`, đẩy vào `AsyncStream`.
- Pipeline: chuẩn hoá (trim + lowercase) → debounce 300ms → bỏ trùng → bỏ query rỗng → gọi API.
- `deinit` phải `finish()` stream và huỷ task.
- Demo: gõ "s","sw","swi","swift" cách nhau 50ms → **chỉ 1 request**. Gõ "swift " → **không** request. Gõ "al" → 1 request. In tổng số request = 2.
- Mở rộng (nếu còn thời gian): viết bản Combine tương đương và so sánh số dòng code + độ dễ đọc.

<details><summary>Lời giải tham khảo</summary>

```swift
import Foundation

struct Repo: Sendable, Equatable { let name: String; let stars: Int }

protocol RepoSearching: Sendable {
    func search(_ query: String) async throws -> [Repo]
}

struct FakeGitHubAPI: RepoSearching {
    let all = [Repo(name: "swift", stars: 67000), Repo(name: "swift-format", stars: 2500),
               Repo(name: "swiftlint", stars: 18000), Repo(name: "alamofire", stars: 41000)]
    func search(_ query: String) async throws -> [Repo] {
        try await Task.sleep(for: .milliseconds(100))          // giả lập network; ném CancellationError nếu bị huỷ
        return all.filter { $0.name.hasPrefix(query) }
    }
}

@MainActor
final class SearchViewModel {
    private(set) var results: [Repo] = []
    private(set) var requestCount = 0
    private let api: RepoSearching
    private let input: AsyncStream<String>.Continuation
    private var pipeline: Task<Void, Never>?

    init(api: RepoSearching) {
        self.api = api
        let (stream, continuation) = AsyncStream.makeStream(of: String.self)
        self.input = continuation
        pipeline = Task { [weak self] in
            var debounceTask: Task<Void, Never>?
            var last: String?
            for await raw in stream {
                let q = raw.trimmingCharacters(in: .whitespaces).lowercased()
                debounceTask?.cancel()                       // gõ tiếp → huỷ lần chờ trước
                debounceTask = Task { [weak self] in
                    do { try await Task.sleep(for: .milliseconds(300)) } catch { return }
                    guard q != last, !q.isEmpty else { return }   // removeDuplicates + bỏ query rỗng
                    last = q
                    await self?.performSearch(q)
                }
            }
            debounceTask?.cancel()
        }
    }

    func queryChanged(_ text: String) { input.yield(text) }

    private func performSearch(_ q: String) async {
        requestCount += 1
        do {
            results = try await api.search(q)
            print("[\(q)] →", results.map(\.name))
        } catch {
            print("[\(q)] cancelled/failed:", error)
        }
    }

    deinit { input.finish(); pipeline?.cancel() }
}

@MainActor
func demo() async throws {
    let vm = SearchViewModel(api: FakeGitHubAPI())
    for text in ["s", "sw", "swi", "swift"] {
        vm.queryChanged(text)
        try await Task.sleep(for: .milliseconds(50))
    }
    try await Task.sleep(for: .milliseconds(600))
    vm.queryChanged("swift ")                                // trim ra "swift" → trùng → bỏ qua
    try await Task.sleep(for: .milliseconds(600))
    vm.queryChanged("al")
    try await Task.sleep(for: .milliseconds(600))
    print("requests:", vm.requestCount)
}

try await demo()
// [swift] → ["swift", "swift-format", "swiftlint"]
// [al] → ["alamofire"]
// requests: 2
```
Điểm cần tự giải thích được:
- Vì sao `Task { }` trong `init` của class `@MainActor` cũng chạy trên MainActor (kế thừa isolation) → `last`, `debounceTask` không bị race.
- Vì sao `[weak self]`: `Task` giữ strong closure; không weak thì VM sống mãi vì pipeline không bao giờ kết thúc.
- Phiên bản này **chưa** huỷ request đang bay khi có query mới (khác `switchToLatest`). Thử thêm: giữ `searchTask` và `cancel()` nó trước khi gọi request mới.
</details>

**📱 Side project — Milestone 8 (T7):** bật Swift 6 language mode cho RepoScope, sửa hết lỗi Sendable theo quy trình §5 (ghi lại từng lỗi gặp và cách sửa vào `MIGRATION_NOTES.md`), làm ô search có debounce bằng AsyncStream **hoặc** Combine. Chi tiết: [side-project](../tracks/side-project.md).

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ `Sendable` là gì? Vì sao Swift cần nó?**
<details><summary>Đáp án gợi ý</summary>

`Sendable` là marker protocol báo cho compiler biết giá trị của kiểu đó **an toàn khi đi qua ranh giới giữa các isolation domain** (actor, MainActor, task chạy song song). Nhờ nó, Swift 6 phát hiện data race **lúc compile** thay vì để crash ngẫu nhiên lúc runtime. Không có chi phí runtime — chỉ là kiểm tra kiểu.
</details>

**Q2 ⭐ Kiểu nào tự động Sendable? Class có tự Sendable không?**
<details><summary>Đáp án gợi ý</summary>

Struct/enum **không public** có mọi thành phần Sendable thì tự suy luận; actor luôn Sendable; class gắn global actor như `@MainActor` là Sendable. Class thường **không bao giờ** tự Sendable — phải khai báo và đáp ứng: `final`, mọi stored property là `let` kiểu Sendable, không kế thừa class nào ngoài `NSObject`. Type `public` cũng phải khai báo tường minh.
</details>

**Q3 ⭐ `@unchecked Sendable` dùng khi nào? Rủi ro?**
<details><summary>Đáp án gợi ý</summary>

Khi class có state mutable nhưng **tự đồng bộ** bằng cơ chế compiler không hiểu — lock, serial queue — hoặc class SDK cũ biết chắc là thread-safe. Nó tắt kiểm tra của compiler, nên nếu cài đặt sai thì data race quay lại mà không có cảnh báo. Mình luôn kèm lock và comment lý do; với iOS 18+ ưu tiên `Mutex` để được Sendable thật; nếu được thì dùng actor.
</details>

**Q4 `@Sendable` closure khác closure thường thế nào?**
<details><summary>Đáp án gợi ý</summary>

`@Sendable` báo closure có thể chạy đồng thời ở domain khác, nên compiler cấm nó capture `var` để sửa (`mutation of captured var ... in concurrently-executing code`) và cấm capture giá trị non-Sendable. Bản thân closure `@Sendable` thì là Sendable, truyền qua ranh giới được. Gặp ở `TaskGroup.addTask`, `onTermination`, completion handler của API concurrency-aware.
</details>

**Q5 ⭐ Swift 6 language mode khác Swift 5 mode thế nào? Có phải cứ dùng Xcode mới là Swift 6?**
<details><summary>Đáp án gợi ý</summary>

Language mode là cấu hình **theo target**, độc lập với phiên bản compiler. Swift 6 mode bật **data-race safety đầy đủ** và biến các vi phạm thành **error**. Swift 5 mode có thể bật *Strict Concurrency Checking = Complete* để thấy cùng các vấn đề nhưng ở dạng warning — dùng trong giai đoạn migrate. Compiler 6.x build được cả hai mode, và các module ở mode khác nhau dùng chung được.
</details>

**Q6 ⭐ Gặp lỗi "var 'x' is not concurrency-safe because it is nonisolated global shared mutable state", bạn sửa thế nào?**
<details><summary>Đáp án gợi ý</summary>

Biến global/static `var` có thể bị truy cập từ mọi thread. Tuỳ trường hợp: đổi thành `let` nếu không cần sửa (và kiểu phải Sendable); gắn `@MainActor` nếu chỉ UI dùng; đưa vào `actor` nếu là state dùng chung; hoặc bọc trong class có lock. `nonisolated(unsafe)` chỉ là lối thoát tạm khi migrate, phải tự chứng minh an toàn. Lưu ý `static let shared` của class non-Sendable cũng bị lỗi tương tự.
</details>

**Q7 ⭐ Lỗi "sending 'x' risks causing data races" nghĩa là gì?**
<details><summary>Đáp án gợi ý</summary>

Bạn đang chuyển một giá trị **non-Sendable** sang domain khác (vd. truyền vào method của actor) **trong khi** domain hiện tại vẫn còn dùng nó sau đó — hai bên có thể truy cập cùng lúc. Nhờ region-based isolation, nếu sau khi gửi bạn không đụng lại thì compiler cho phép. Sửa: không dùng lại sau khi gửi, chỉ gửi phần dữ liệu Sendable (String, struct), đổi kiểu sang struct/Sendable, hoặc để actor tự tạo object.
</details>

**Q8 Region-based isolation và từ khoá `sending` là gì?**
<details><summary>Đáp án gợi ý</summary>

Region-based isolation (SE-0414) cho compiler theo dõi các "vùng" giá trị non-Sendable. Một vùng chưa gắn với actor nào ("disconnected") được phép chuyển sang actor khác, miễn là sau đó không ai ở phía cũ dùng nữa — giống chuyển quyền sở hữu. `sending` (SE-0430) đưa điều đó vào signature: tham số `sending` nghĩa là hàm nhận quyền sở hữu; kết quả `sending` nghĩa là trả về một vùng mới mà caller tuỳ ý gửi đi. Ví dụ closure của `Task { }` là `sending`.
</details>

**Q9 ⭐ Bạn migrate một project lớn lên Swift 6 như thế nào?**
<details><summary>Đáp án gợi ý</summary>

Từng module, từ module lá lên: bật Strict Concurrency = Targeted rồi Complete ở Swift 5 mode để thấy warning mà vẫn ship được. Model chuyển sang struct Sendable; UI (VC, ViewModel) gắn `@MainActor`; singleton/global thành `let`, `@MainActor` hoặc actor; service có state thành actor; callback cũ bọc bằng continuation/AsyncStream; thư viện bên thứ ba chưa hỗ trợ thì `@preconcurrency import`. Hết warning thì chuyển language mode của target đó sang 6. Tránh rải `@unchecked`/`nonisolated(unsafe)` để chữa cháy.
</details>

**Q10 Một `@MainActor` class implement delegate của một protocol không isolated thì gặp lỗi gì, sửa ra sao?**
<details><summary>Đáp án gợi ý</summary>

Lỗi "conformance ... crosses into main actor-isolated code" vì method main-actor không thoả requirement nonisolated. Cách sửa: Swift 6.2+ dùng isolated conformance `: @MainActor Protocol`; hoặc đánh dấu method `nonisolated` rồi dùng `MainActor.assumeIsolated` nếu chắc chắn được gọi trên main (sai sẽ crash), hoặc nhảy bằng `Task { @MainActor in ... }`; nếu protocol do mình sở hữu và chỉ dùng cho UI thì gắn `@MainActor` cho protocol.
</details>

**Q11 Swift 6.2 "approachable concurrency" thay đổi gì?**
<details><summary>Đáp án gợi ý</summary>

Có tuỳ chọn **default actor isolation = MainActor** cho cả module — hợp app target vì đa số code là UI. Hàm `async` nonisolated mặc định chạy trên actor của **caller** (`nonisolated(nonsending)`) thay vì luôn nhảy sang global pool, nên bớt lỗi sending. Muốn chạy song song thật thì đánh dấu `@concurrent`. Mục tiêu là code đơn luồng mặc định an toàn, chỉ "opt-in" song song khi cần.
</details>

**Q12 ⭐ AsyncStream dùng để làm gì? Bọc một delegate như CLLocationManager thế nào?**
<details><summary>Đáp án gợi ý</summary>

AsyncStream biến nguồn phát **nhiều giá trị** theo kiểu push (delegate, callback, notification) thành `AsyncSequence` để dùng `for await`. Tạo stream bằng `AsyncStream.makeStream` hoặc closure init, giữ `continuation`; trong delegate method gọi `continuation.yield(value)`; khi nguồn kết thúc gọi `finish()` (lỗi thì dùng AsyncThrowingStream + `finish(throwing:)`); đặt `onTermination` để `stopUpdatingLocation()` khi consumer huỷ. Chọn buffering policy phù hợp, vd. `.bufferingNewest(1)` cho location. Callback **một lần** thì dùng `withCheckedContinuation` thay vì stream.
</details>

**Q13 AsyncSequence khác Combine Publisher thế nào?**
<details><summary>Đáp án gợi ý</summary>

AsyncSequence là **pull**: consumer gọi `next()` khi sẵn sàng (back-pressure tự nhiên), một consumer cho mỗi iterator, tích hợp với structured concurrency, cancellation và Sendable checking. Publisher là **push**: producer đẩy giá trị tới nhiều subscriber, có bộ operator thời gian phong phú (debounce, combineLatest), huỷ bằng AnyCancellable, nhưng không được compiler kiểm tra data race.
</details>

**Q14 ⭐ Giải thích Publisher, Subscriber, Subscription trong Combine. `AnyCancellable` để làm gì?**
<details><summary>Đáp án gợi ý</summary>

Publisher phát 0..n giá trị rồi có thể kết thúc bằng finished hoặc failure. Subscriber nhận giá trị; khi subscribe, publisher trao một Subscription, subscriber qua đó xin số lượng giá trị (demand) và có thể cancel. `sink`/`assign` trả về `AnyCancellable` — một token, khi bị deinit thì tự cancel subscription. Vì vậy phải lưu nó (thường vào `Set<AnyCancellable>` của object), không thì subscription chết ngay.
</details>

**Q15 ⭐ Combine gây retain cycle ở đâu? Cách tránh?**
<details><summary>Đáp án gợi ý</summary>

Object giữ `cancellables` → AnyCancellable → subscription → closure. Nếu closure `sink` capture `self` mạnh, hoặc dùng `assign(to: \.prop, on: self)` (giữ self mạnh) thì thành vòng. Tránh: `[weak self]` trong sink; dùng `assign(to: &$prop)` (iOS 14+) — nó gắn vòng đời subscription vào `@Published` và không trả AnyCancellable. Kiểm tra bằng `deinit` log hoặc Memory Graph.
</details>

**Q16 `PassthroughSubject` vs `CurrentValueSubject` vs `@Published`?**
<details><summary>Đáp án gợi ý</summary>

Passthrough không lưu giá trị — subscriber chỉ nhận những gì phát sau khi subscribe; hợp cho sự kiện. CurrentValue lưu giá trị hiện tại, subscriber mới nhận ngay giá trị đó; hợp cho state. `@Published` là property wrapper hoạt động gần như CurrentValueSubject qua `$prop`, nhưng phát trong `willSet` — đọc property bên trong sink sẽ ra giá trị cũ.
</details>

**Q17 ⭐ `debounce` vs `throttle`? `flatMap` vs `switchToLatest` khi làm search?**
<details><summary>Đáp án gợi ý</summary>

`debounce` đợi nguồn **im lặng** một khoảng rồi phát giá trị cuối — hợp search khi gõ. `throttle` phát tối đa một giá trị mỗi khoảng thời gian, dù nguồn vẫn liên tục — hợp scroll/nút bấm liên tục. Với search, `flatMap` giữ mọi request nên response cũ có thể về sau và ghi đè kết quả mới; `map` + `switchToLatest` huỷ request cũ khi có query mới, nên luôn hiển thị kết quả của query mới nhất.
</details>

**Q18 `receive(on:)` vs `subscribe(on:)`?**
<details><summary>Đáp án gợi ý</summary>

`receive(on:)` đổi scheduler cho các operator và subscriber **phía sau** nó — dùng `receive(on: DispatchQueue.main)` trước khi cập nhật UI. `subscribe(on:)` ảnh hưởng **phía trước**: nơi subscription được tạo và request được gửi lên upstream — dùng khi upstream làm việc nặng đồng bộ lúc subscribe. Lưu ý `RunLoop.main` bị hoãn trong lúc scroll (tracking mode), `DispatchQueue.main` thì không.
</details>

**Q19 ⭐ Combine hay async/await — bạn chọn cái nào?**
<details><summary>Đáp án gợi ý</summary>

Async/await là mặc định cho code mới: tác vụ một lần (network, DB) viết tuần tự, dễ đọc, có structured cancellation và được Swift 6 kiểm tra data race. Combine phù hợp khi bài toán là luồng sự kiện theo thời gian cần operator như debounce, combineLatest, hoặc codebase đã dùng `ObservableObject`/`@Published`. Apple gần như không phát triển thêm Combine và SwiftUI đã chuyển sang `@Observable`, nên mình không đưa Combine vào chỗ mới nếu không cần. Hai thế giới nối bằng `.values` và `Future`.
</details>

---

## 🏁 Mốc M2 — Tự kiểm tra (CN 03/01, 60 phút)

> Mốc M2 trong [README](../README.md): *"Tự tìm và sửa một retain cycle bằng Memory Graph; viết code async/await + actor không lỗi compile Swift 6."* Làm **không nhìn tài liệu, không AI**, bấm giờ.

**Phần A — Memory (20')**
- [ ] Mở project (side project hoặc project demo), tạo cố ý 2 retain cycle: một bằng closure (`sink`/`Timer`/completion lưu vào thuộc tính) và một bằng delegate `strong`.
- [ ] Dùng **Debug Memory Graph** chỉ ra được cả 2 (biểu tượng `!` tím hoặc instance không biến mất sau khi pop), chụp màn hình.
- [ ] Sửa và chứng minh `deinit` chạy.
- [ ] Giải thích miệng trong 1 phút: `weak` vs `unowned`, khi nào `unowned` crash.

**Phần B — Concurrency Swift 6 (30')**
Viết từ đầu file `m2.swift`, compile bằng `swiftc -swift-version 6 -parse-as-library`, **0 error, 0 warning**:
- [ ] `actor ImageCache` với `func image(for url: URL) async throws -> Data`: nếu đã có trong cache trả ngay; nếu **đang tải** URL đó thì **chờ chung** task đang chạy (không tải 2 lần — dùng `[URL: Task<Data, Error>]`); nhớ xử lý reentrancy (tuần 7).
- [ ] Hàm `loadAll(urls:) async throws -> [Data]` tải song song bằng `withThrowingTaskGroup`, **giữ đúng thứ tự** đầu vào.
- [ ] Một `@MainActor final class GalleryViewModel` gọi `loadAll` và cập nhật `var count`.
- [ ] Một `struct Photo: Sendable` và ít nhất một chỗ bạn **cố tình** gây lỗi `sending ... risks causing data races` rồi sửa.

**Phần C — Nói (10')**
Ghi âm, mỗi câu ≤ 2 phút: (1) Sendable là gì và `@unchecked` khi nào; (2) actor reentrancy; (3) Combine vs async/await.

**Đạt M2 khi:** Phần A xong cả 4 ô, Phần B compile sạch và chạy ra đúng kết quả, Phần C không bị ấp úng quá 2 lần mỗi câu. Chưa đạt → dùng 2–3 ngày đầu tuần 9 để bù, **đừng** bỏ qua — Giai đoạn 3 (UI) giả định bạn đã chắc phần này.

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Bài tập lớn `search_debounce.swift` chạy đúng và compile sạch ở Swift 6 mode
- [ ] Gặp và tự sửa được cả 5 loại lỗi ở §3 (ghi lại thông báo lỗi nguyên văn vào notes)
- [ ] Trả lời được ≥ 15/19 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Sendable là gì, và mình migrate project lên Swift 6 thế nào"
- [ ] Ghi âm 2 phút: "Combine hay async/await"
- [ ] Làm xong 6 bài DSA tuần 8 (Linked List + Recursion)
- [ ] Side project Milestone 8: Swift 6 mode + search debounce, đã commit
- [ ] **Đạt mốc M2**
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) (phần *Sendable Types*)
- [Swift.org — Migrating to Swift 6](https://www.swift.org/migration/documentation/migrationguide/) — đọc *Data Race Safety*, *Common Compiler Errors*, *Incremental Adoption*
- [Apple — `Sendable`](https://developer.apple.com/documentation/swift/sendable) · [`AsyncStream`](https://developer.apple.com/documentation/swift/asyncstream) · [Combine](https://developer.apple.com/documentation/combine)
- Swift Evolution: [SE-0302 Sendable](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md), [SE-0414 Region-based isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md), [SE-0430 `sending`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md)
- WWDC24 — *Migrate your app to Swift 6*
- WWDC22 — *Eliminate data races using Swift Concurrency*
- WWDC21 — *Meet AsyncSequence*
- WWDC25 — *Embracing Swift concurrency* (approachable concurrency, default MainActor isolation)
- WWDC19 — *Introducing Combine* và *Combine in Practice*
- [swift-async-algorithms](https://github.com/apple/swift-async-algorithms) — `debounce`, `throttle`, `combineLatest` cho AsyncSequence
- Donny Wals — loạt bài về Swift 6 & Combine (donnywals.com) · Matt Massicotte — "Swift concurrency" (massicotte.org) · Antoine van der Lee — SwiftLee (Sendable, Swift 6 migration)
