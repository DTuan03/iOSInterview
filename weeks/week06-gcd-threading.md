# Tuần 6 — Thread, GCD & OperationQueue (14/12 – 20/12/2026)

> **Giai đoạn 2 · Bộ nhớ & Concurrency** · Thời lượng: ~14 giờ
> Swift Concurrency (tuần 7) là tương lai, nhưng **codebase thật ở các công ty vẫn đầy GCD**, và người phỏng vấn dùng GCD để kiểm tra bạn có **hiểu đa luồng thật sự** không: "đoạn code này in ra gì", "chỗ này có deadlock không", "viết cho tôi một cache thread-safe". Tuần này nắm chắc GCD thì tuần 7 sẽ hiểu *vì sao* async/await ra đời.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích process vs thread, vì sao UI chỉ được cập nhật trên main thread
- [ ] Phân biệt serial/concurrent queue, sync/async — **4 tổ hợp** và thứ tự in ra của từng tổ hợp
- [ ] Nhìn code chỉ ra được **deadlock** và giải thích bằng sơ đồ thời gian
- [ ] Dùng đúng `DispatchGroup`, `DispatchSemaphore`, `DispatchWorkItem` (cancel/debounce), `barrier`, `asyncAfter`
- [ ] Dùng `OperationQueue`: dependency, `maxConcurrentOperationCount`, cancel, viết **custom async Operation**
- [ ] Phân biệt race condition và data race; bắt data race bằng **Thread Sanitizer**
- [ ] Viết cache thread-safe bằng 3 cách (serial queue, concurrent + barrier, lock) và so sánh
- [ ] Giải thích priority inversion và thread explosion
- [ ] Làm xong 6 bài DSA tuần 6 — chủ đề **Binary Search** (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 14/12** | 2h | Tạo `practice/week06/`. §1 Process/thread/main thread, §2 GCD: queue & QoS, §3 **sync vs async**. Bài tập 1 · DSA: Binary Search | `day1.swift` |
| **T3 15/12** | 2h | §4 **Deadlock** (vẽ sơ đồ). Bài tập 2 · DSA: Search Insert Position | `day2.swift` + sơ đồ trên giấy |
| **T4 16/12** | 2h | §5 DispatchGroup, Semaphore, WorkItem, barrier, asyncAfter. Bài tập 3 · DSA: First Bad Version | `day3.swift` |
| **T5 17/12** | 2h | §6 **Race condition, data race, Thread Sanitizer**, §7 Lock & thread-safe. Bài tập 4 · DSA: Find First and Last Position of Element in Sorted Array | `day4.swift` chạy với `-sanitize=thread` |
| **T6 18/12** | 1,5h | §8 **OperationQueue**, §9 priority inversion & thread explosion. Bài tập 5 · DSA: Search in Rotated Sorted Array | `day5.swift` |
| **T7 19/12** | 4h | Bài tập 6 + **Bài tập lớn** `ImageLoader`. Side project: Milestone 6 (load ảnh bằng GCD + cache thread-safe) · DSA: Koko Eating Bananas | `imageloader.swift`, side project |
| **CN 20/12** | 2h | Tự trả lời 18 câu phỏng vấn. Ghi âm 2 câu. Vẽ lại sơ đồ deadlock & barrier. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

> Code non-UI chạy bằng `swift file.swift` (**Swift 5 language mode** — mặc định của lệnh này). Nếu bật Swift 6 mode, nhiều snippet GCD sẽ có cảnh báo/lỗi `Sendable` — đó chính là lý do của tuần 7–8. Muốn bật Thread Sanitizer: `swiftc -sanitize=thread file.swift -o app && ./app`.

### §1. Process, thread, main thread

- **Process** = một app đang chạy: có không gian bộ nhớ riêng (heap, code, tài nguyên). Các process không đọc được bộ nhớ của nhau.
- **Thread** = một luồng thực thi bên trong process. Mọi thread trong process **dùng chung heap**, mỗi thread có **stack riêng** (main thread ~1 MB trên iOS, thread phụ ~512 KB).
- CPU có vài core; hệ điều hành **chia thời gian** (context switch) để nhiều thread trông như chạy cùng lúc. Context switch tốn chi phí (lưu/khôi phục thanh ghi, cache bị nguội).

```
Process (app RepoScope)
├── Heap (dùng chung) ── object, cache, ảnh...  ← nơi xảy ra data race
├── Main thread  [stack] ── RunLoop: nhận touch → chạy code của bạn → vẽ UI → lặp lại (~60–120 lần/giây)
├── Thread 2     [stack] ── tải mạng
└── Thread 3     [stack] ── decode ảnh
```

**Vì sao UI phải chạy trên main thread?**
1. **UIKit không thread-safe** — các class UIKit không có lock bên trong (thêm lock vào mọi view thì chậm khủng khiếp).
2. **Main RunLoop** điều phối toàn bộ vòng đời UI: nhận sự kiện chạm, chạy layout, commit Core Animation transaction để render. Sửa view từ thread khác có thể rơi vào giữa một lần layout/render → giao diện sai, không cập nhật, hoặc crash ngẫu nhiên.
3. Một nguồn sự thật, một thứ tự → dễ suy luận.

Hệ quả kép: **không cập nhật UI ngoài main**, và **không chặn main** (tải mạng, đọc file lớn, decode ảnh trên main → app đơ; chặn quá lâu → watchdog kill app khi khởi động).

Xcode có **Main Thread Checker** (Scheme → Diagnostics, bật mặc định khi debug): gọi API UIKit ngoài main sẽ hiện cảnh báo tím *"Main Thread Checker: UI API called on a background thread"*.

### §2. GCD — Grand Central Dispatch

Thay vì tự tạo `Thread` (tốn kém, khó quản lý), bạn **ném công việc (closure) vào hàng đợi (queue)**, GCD tự quyết định chạy trên thread nào trong **thread pool** của nó.

**Hai loại queue:**
```
SERIAL queue: mỗi lúc 1 task, đúng thứ tự FIFO
  ──[task1]──[task2]──[task3]──▶ thời gian

CONCURRENT queue: nhiều task chạy cùng lúc (bắt đầu theo FIFO, kết thúc thứ tự bất kỳ)
  ──[task1──────]
  ──[task2───]
  ──[task3─────────]          ▶ thời gian
```

**Các queue có sẵn và tự tạo:**
| Queue | Loại | Dùng cho |
|---|---|---|
| `DispatchQueue.main` | Serial, chạy trên main thread | Cập nhật UI |
| `DispatchQueue.global(qos:)` | Concurrent, dùng chung toàn hệ thống | Việc nền ngắn, không cần thứ tự |
| `DispatchQueue(label: "com.app.db")` | Serial (mặc định) | Bảo vệ một tài nguyên (đồng bộ hoá) |
| `DispatchQueue(label:..., attributes: .concurrent)` | Concurrent riêng | Reader–writer với barrier |

**QoS (Quality of Service)** — mức ưu tiên, ảnh hưởng tới CPU, I/O và cả tiết kiệm pin:
| QoS | Ý nghĩa | Ví dụ |
|---|---|---|
| `.userInteractive` | Phải xong ngay, user đang nhìn (frame tiếp theo) | Animation, xử lý sự kiện (main thread có QoS này) |
| `.userInitiated` | User vừa bấm và đang chờ kết quả | Mở tài liệu, load dữ liệu màn hình vừa mở |
| `.default` | Mặc định | — |
| `.utility` | Lâu, user có thể thấy progress bar | Download lớn, import dữ liệu |
| `.background` | User không thấy | Backup, prefetch, dọn cache |

```swift
let q = DispatchQueue(label: "work", qos: .userInitiated)
DispatchQueue.global(qos: .background).async { print("background") }
```

**Mẫu kinh điển** — làm nặng ở nền, cập nhật UI ở main:
```swift
// Minh hoạ, bên trong một UIViewController
DispatchQueue.global(qos: .userInitiated).async {
    let image = decodeBigImage()            // nặng → nền
    DispatchQueue.main.async {
        self.imageView.image = image        // UI → main
    }
}
```

### §3. sync vs async — đọc kỹ

- `queue.async { }` — **ném task vào queue rồi đi tiếp ngay**, không chờ.
- `queue.sync { }` — **ném task vào queue và CHẶN thread hiện tại** cho tới khi task chạy xong.

> sync/async nói về **thread gọi có chờ hay không**. serial/concurrent nói về **queue chạy task thế nào**. Đây là 2 trục độc lập → 4 tổ hợp.

```swift
import Foundation
let serial = DispatchQueue(label: "com.demo.serial")
print("1")
serial.async { print("2 (async)") }
print("3")
serial.sync { print("4 (sync)") }
print("5")
```
Output khi chạy thử:
```
1
3
2 (async)
4 (sync)
5
```
Cái gì **đảm bảo**, cái gì **không**?
- `1` luôn đầu tiên. `2` luôn trước `4` (serial queue: FIFO). `4` luôn trước `5` (sync chặn tới khi 4 in xong).
- `2` và `3` **không đảm bảo thứ tự** — block 2 chạy trên thread khác, có thể in trước `3`. Phỏng vấn mà trả lời chắc chắn "1 3 2 4 5" là mất điểm; phải nói "thường là vậy nhưng 2 và 3 có thể đảo".

**Serial vs concurrent:**
```swift
let s2 = DispatchQueue(label: "s2")
for i in 1...5 { s2.async { print("serial", i) } }      // luôn 1,2,3,4,5

let conc = DispatchQueue(label: "c", attributes: .concurrent)
let g = DispatchGroup()
for i in 1...5 {
    conc.async(group: g) {
        Thread.sleep(forTimeInterval: Double.random(in: 0...0.05))
        print("concurrent", i)                        // thứ tự ngẫu nhiên mỗi lần chạy
    }
}
g.wait()
```

**`sync` dùng để làm gì?** Chủ yếu để **đọc dữ liệu được bảo vệ bởi một queue** và trả về giá trị (`queue.sync { storage[key] }`). Gần như không bao giờ dùng `sync` để chạy việc nặng, và **không bao giờ `DispatchQueue.main.sync` từ main thread**.

### §4. Deadlock

**Deadlock** = hai (hoặc nhiều) bên chờ nhau mãi mãi. Dạng hay gặp nhất với GCD: **gọi `sync` vào chính serial queue mà mình đang chạy trên đó**.

```swift
let q = DispatchQueue(label: "serial")
q.sync {
    print("vào block ngoài")
    q.sync { print("không bao giờ in") }    // 💥
}
```
```
Serial queue q:  [ block ngoài ............ (đang chạy, chờ block trong xong) ]
                                             │
                 block trong xếp hàng SAU block ngoài → chỉ chạy khi block ngoài xong
                                             │
                 block ngoài chờ block trong ◀┘  →  chờ nhau mãi mãi
```
Trên Apple platform, libdispatch phát hiện được trường hợp này và **crash** với `EXC_BREAKPOINT` trong `__DISPATCH_WAIT_FOR_QUEUE__` (crash log ghi *"BUG IN CLIENT OF LIBDISPATCH: dispatch_sync called on queue already owned by current thread"*). Không phải lúc nào cũng phát hiện được — nhiều deadlock chỉ đơn giản là **app treo im lặng**.

Biến thể kinh điển nhất:
```swift
// Đang ở main thread (viewDidLoad, IBAction...)
DispatchQueue.main.sync { updateUI() }    // 💥 main chờ main
```

Bảng tra nhanh (được kiểm chứng ở Bài tập 2):
| Code | Kết quả |
|---|---|
| `serial.sync { serial.sync {} }` | 💥 Deadlock (crash) |
| `serial.async { serial.sync {} }` | 💥 Deadlock — vẫn là sync vào queue đang chạy |
| `serial.sync { serial.async {} }` | ✅ OK — async không chờ |
| `serialA.sync { serialB.sync {} }` | ✅ OK (miễn là B không sync ngược lại vào A) |
| `concurrent.sync { concurrent.sync {} }` | ✅ OK — concurrent queue chạy được block thứ 2 |
| `concurrent.async { concurrent.sync(flags: .barrier) {} }` | ❌ Treo — barrier chờ block đang chạy (chính nó) xong |
| `DispatchQueue.main.sync {}` từ main | 💥 Deadlock |
| `DispatchQueue.main.sync {}` từ background | ✅ OK (nhưng vẫn nên dùng async) |

Deadlock kiểu "lock order": thread 1 giữ lock A chờ lock B, thread 2 giữ B chờ A. Phòng tránh: luôn lấy các lock theo **cùng một thứ tự**, giữ lock **càng ngắn càng tốt**, không gọi code lạ (callback, delegate) khi đang giữ lock.

### §5. Công cụ phối hợp: Group, Semaphore, WorkItem, Barrier, asyncAfter

**5.1. `DispatchGroup` — chờ nhiều việc xong rồi làm tiếp**
```swift
func fetch(_ id: Int, completion: @escaping (String) -> Void) {
    DispatchQueue.global().asyncAfter(deadline: .now() + .milliseconds(100 * id)) {
        completion("user\(id)")
    }
}
let group = DispatchGroup()
var results: [String] = []
let lock = NSLock()                       // completion chạy trên nhiều thread → phải khoá
for id in 1...3 {
    group.enter()                         // +1
    fetch(id) { name in
        lock.lock(); results.append(name); lock.unlock()
        group.leave()                     // −1 (PHẢI gọi đúng 1 lần ở mọi nhánh)
    }
}
group.notify(queue: .main) {              // chạy khi bộ đếm về 0, KHÔNG chặn thread
                                          // (chạy trong script: đổi thành .global() vì script không có main RunLoop)
    print("Xong hết:", results.sorted())  // ["user1", "user2", "user3"]
}
```
- `enter()`/`leave()` phải cân bằng: thiếu `leave` → `notify` không bao giờ chạy; thừa `leave` → crash.
- `group.wait()` thì **chặn** thread hiện tại (không gọi trên main). `wait(timeout:)` trả `.timedOut` nếu quá hạn.
- Nếu task là closure đồng bộ: `queue.async(group: group) { ... }` thay cho enter/leave.

**5.2. `DispatchSemaphore` — giới hạn số việc chạy cùng lúc**
Semaphore là bộ đếm: `wait()` giảm 1 (nếu đang 0 thì chặn chờ), `signal()` tăng 1.
```swift
let semaphore = DispatchSemaphore(value: 2)     // tối đa 2 việc cùng lúc
let group = DispatchGroup()
for i in 1...6 {
    DispatchQueue.global().async(group: group) {
        semaphore.wait()
        defer { semaphore.signal() }
        Thread.sleep(forTimeInterval: 0.05)     // giả lập download — tối đa 2 cái chạy đồng thời
        print("xong", i)
    }
}
group.wait()
```
Cảnh báo: mỗi thread đang `wait()` là một thread **bị chặn** → GCD tạo thêm thread → nguy cơ thread explosion (§9). Đừng dùng semaphore để "biến async thành sync" trên main thread (đơ app, có thể deadlock nếu callback cũng về main).

**5.3. `DispatchWorkItem` — task có thể huỷ (debounce search)**
```swift
final class SearchDebouncer {
    private var pending: DispatchWorkItem?
    private let queue = DispatchQueue(label: "search")
    func textChanged(_ text: String) {
        pending?.cancel()                                   // huỷ lần gõ trước
        let item = DispatchWorkItem { print("Gọi API với:", text) }
        pending = item
        queue.asyncAfter(deadline: .now() + .milliseconds(300), execute: item)
    }
}
let debouncer = SearchDebouncer()
for text in ["s", "sw", "swi", "swift"] {
    debouncer.textChanged(text)
    Thread.sleep(forTimeInterval: 0.05)
}
// Sau 300ms chỉ in: Gọi API với: swift
```
**Quan trọng:** `cancel()` chỉ ngăn item **chưa bắt đầu**. Item đang chạy dở **vẫn chạy hết** — trừ khi code bên trong tự kiểm tra `item.isCancelled`. (Giống hệt cancellation hợp tác của Swift Concurrency tuần sau.)

**5.4. Barrier — reader–writer trên concurrent queue**
Đọc thì cho chạy song song (nhanh), ghi thì phải độc quyền:
```
concurrent queue:  [read][read][read]  ║ [WRITE (barrier)] ║  [read][read]
                    song song           chờ mọi read trước xong,  chạy tiếp
                                        chạy MỘT MÌNH
```
```swift
final class ThreadSafeDictionary<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "tsdict", attributes: .concurrent)
    subscript(key: Key) -> Value? {
        get { queue.sync { storage[key] } }                                   // đọc: sync, song song
        set { queue.async(flags: .barrier) { self.storage[key] = newValue } } // ghi: barrier
    }
    var count: Int { queue.sync { storage.count } }
}
let dict = ThreadSafeDictionary<Int, String>()
DispatchQueue.concurrentPerform(iterations: 1000) { i in
    dict[i] = "v\(i)"
    _ = dict[i / 2]
}
print("count:", dict.count)    // count: 1000
```
- Barrier **chỉ có tác dụng trên concurrent queue tự tạo**. Trên `global()` nó bị bỏ qua (hành xử như async thường) — lỗi rất hay gặp.
- Ghi bằng `async(flags: .barrier)` để người ghi không phải chờ; đọc bằng `sync` để trả giá trị.

**5.5. `asyncAfter`** — chạy sau một khoảng thời gian, **không chặn** thread (khác `sleep`):
```swift
DispatchQueue.main.asyncAfter(deadline: .now() + 2) { self.hideToast() }   // minh hoạ trong một VC
```
Không huỷ được trực tiếp → nếu cần huỷ, dùng `DispatchWorkItem` như 5.3.

**5.6. `concurrentPerform`** — chạy một vòng lặp song song trên các core, **chờ** tới khi xong hết. Dùng cho tính toán CPU thuần (xử lý pixel, mảng lớn). GCD tự chọn số thread hợp lý → không bị thread explosion.

### §6. Race condition, data race, Thread Sanitizer

- **Data race**: ≥ 2 thread truy cập **cùng một vùng nhớ** cùng lúc, ít nhất 1 thread **ghi**, không có đồng bộ. → undefined behavior: sai số, crash ngẫu nhiên (`EXC_BAD_ACCESS` khi Array/Dictionary đang resize).
- **Race condition**: kết quả **phụ thuộc thứ tự thời gian** giữa các thread — lỗi logic, **có thể xảy ra kể cả khi mọi truy cập đều đã khoá**.

```swift
final class UnsafeCounter {
    var value = 0
    func increment() { value += 1 }        // đọc → cộng → ghi: 3 bước, không nguyên tử
}
let unsafe = UnsafeCounter()
DispatchQueue.concurrentPerform(iterations: 100_000) { _ in unsafe.increment() }
print("Unsafe:", unsafe.value)             // mỗi lần chạy một số khác, ví dụ 65316 (< 100000)
```
Vì sao mất số?
```
Thread A: đọc value = 7 ─────────── cộng → 8 ── ghi 8
Thread B:       đọc value = 7 ── cộng → 8 ──────────── ghi 8      ← mất một lần tăng
```

**Race condition dù đã khoá từng bước — "check-then-act":**
```swift
final class Account {
    private(set) var balance = 100
    private let lock = NSLock()
    func withdrawUnsafe(_ amount: Int) -> Bool {
        lock.lock(); let ok = balance >= amount; lock.unlock()       // kiểm tra (có khoá)
        if ok { Thread.sleep(forTimeInterval: 0.001); lock.lock(); balance -= amount; lock.unlock() }  // hành động (có khoá)
        return ok                                                    // nhưng KHÔNG nguyên tử với nhau
    }
    func withdraw(_ amount: Int) -> Bool {
        lock.lock(); defer { lock.unlock() }                         // ✅ kiểm tra + hành động trong 1 lần khoá
        guard balance >= amount else { return false }
        balance -= amount
        return true
    }
}
let acc = Account()
DispatchQueue.concurrentPerform(iterations: 10) { _ in _ = acc.withdrawUnsafe(30) }
print("Check-then-act sai:", acc.balance)    // số âm, ví dụ -200
let acc2 = Account()
DispatchQueue.concurrentPerform(iterations: 10) { _ in _ = acc2.withdraw(30) }
print("Đúng:", acc2.balance)                 // Đúng: 10
```
→ Bài học: **đơn vị khoá phải là cả thao tác nghiệp vụ**, không phải từng dòng.

**Thread Sanitizer (TSan)** — công cụ runtime phát hiện data race:
- Xcode: Scheme → Run → Diagnostics → ✅ **Thread Sanitizer** (không bật cùng Address Sanitizer được). App chậm 2–20 lần khi bật.
- Terminal: `swiftc -sanitize=thread file.swift -o app && ./app`
```swift
var counter = 0
DispatchQueue.concurrentPerform(iterations: 1000) { _ in counter += 1 }
```
TSan in ra:
```
WARNING: ThreadSanitizer: Swift access race (pid=...)
  Modifying access of Swift variable at 0x... by thread T1:
  ...
  Previous write of size 8 at 0x... by main thread:
```
TSan chỉ bắt race **thực sự xảy ra trong lần chạy đó** → phải chạy đúng luồng code có vấn đề. Nó bắt data race, **không** bắt race condition logic (check-then-act ở trên đã khoá đầy đủ nên TSan im lặng).

### §7. Lock & cách làm code thread-safe

**Nguyên tắc:** mọi truy cập (đọc **và** ghi) vào state chia sẻ phải đi qua **cùng một** cơ chế đồng bộ.

```swift
import Foundation
import os

// Cách 1: NSLock
final class LockedCounter {
    private var value = 0
    private let lock = NSLock()
    func increment() { lock.lock(); defer { lock.unlock() }; value += 1 }
    var current: Int { lock.lock(); defer { lock.unlock() }; return value }
}

// Cách 2: serial queue
final class QueueCounter {
    private var value = 0
    private let queue = DispatchQueue(label: "counter")
    func increment() { queue.sync { value += 1 } }
    var current: Int { queue.sync { value } }
}

// Cách 3: OSAllocatedUnfairLock (iOS 16+) — nhanh nhất, API an toàn
let unfair = OSAllocatedUnfairLock(initialState: 0)
DispatchQueue.concurrentPerform(iterations: 100_000) { _ in unfair.withLock { $0 += 1 } }
print("OSAllocatedUnfairLock:", unfair.withLock { $0 })    // 100000
```
Cả 3 cách đều cho đúng `100000`.

| Cơ chế | Ưu | Nhược / lưu ý |
|---|---|---|
| Serial `DispatchQueue` | Dễ đọc, ghi được bằng `async` (không chặn người gọi) | Chậm hơn lock; `sync` lồng nhau → deadlock |
| Concurrent queue + barrier | Đọc song song | Phức tạp hơn; chỉ đáng khi đọc nhiều hơn ghi rất nhiều |
| `NSLock` | Đơn giản, quen thuộc | Phải nhớ `unlock` (dùng `defer`); lock 2 lần trên cùng thread → deadlock |
| `NSRecursiveLock` | Cùng thread lock lại được | Thường là dấu hiệu thiết kế chưa tốt |
| `os_unfair_lock` | Rất nhanh, có priority donation | Dùng trực tiếp trong Swift **không an toàn** (struct có thể bị di chuyển trong bộ nhớ) → dùng `OSAllocatedUnfairLock` |
| `OSAllocatedUnfairLock` (iOS 16+) | Nhanh, API `withLock` an toàn | iOS 16+ |
| `Mutex` (module `Synchronization`, iOS 18+) | Chuẩn Swift 6, là `Sendable` | iOS 18+ |
| `DispatchSemaphore(value: 1)` | Dùng được như lock | **Không có priority donation** → dễ priority inversion; tránh dùng làm mutex |

**Không có `atomic` trong Swift** như `@property (atomic)` của Objective-C. Và kể cả ObjC `atomic` cũng chỉ bảo vệ getter/setter riêng lẻ, **không** bảo vệ `count += 1` (đọc rồi ghi). Từ iOS 18 có `Atomic` trong module `Synchronization` cho số nguyên.

```swift
import Synchronization          // Swift 6, iOS 18+/macOS 15+
final class Counter: Sendable {
    private let value = Mutex(0)
    func increment() { value.withLock { $0 += 1 } }
    var current: Int { value.withLock { $0 } }
}
```

### §8. OperationQueue

`Operation` / `OperationQueue` là lớp hướng đối tượng **xây trên GCD**, thêm những thứ GCD không có sẵn:

| | GCD | OperationQueue |
|---|---|---|
| Đơn vị | Closure | Object `Operation` (subclass được, giữ state) |
| Phụ thuộc giữa task | Tự làm bằng group/notify | `addDependency` — có sẵn, kể cả giữa các queue khác nhau |
| Giới hạn đồng thời | Tự làm bằng semaphore | `maxConcurrentOperationCount` |
| Huỷ | `DispatchWorkItem.cancel` (hạn chế) | `cancel()`, `cancelAllOperations()`, trạng thái `isCancelled` |
| Theo dõi trạng thái | Không | KVO: `isReady`, `isExecuting`, `isFinished` |
| Chi phí | Nhẹ hơn | Nặng hơn một chút |

```swift
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 2        // = 1 → hoạt động như serial queue

let download = BlockOperation { print("1. download") }
let parse = BlockOperation { print("2. parse") }
let save = BlockOperation { print("3. save") }
parse.addDependency(download)
save.addDependency(parse)
queue.addOperations([save, parse, download], waitUntilFinished: true)   // thứ tự add không quan trọng
// 1. download
// 2. parse
// 3. save
```

**Cancel:** `op.cancel()` chỉ **đặt cờ** `isCancelled = true`. Operation chưa chạy sẽ được bỏ qua (chuyển thẳng sang finished, và operation phụ thuộc vào nó vẫn được chạy tiếp — dependency chỉ nghĩa là "chờ xong", không phải "chờ thành công"). Operation đang chạy phải **tự kiểm tra** `isCancelled` trong `main()`.

**Custom async Operation** — câu hỏi hay gặp: *"Làm sao để Operation chờ một network request async?"*
`BlockOperation` coi là xong ngay khi closure return — nhưng request async thì callback về sau. Phải tự quản trạng thái `isExecuting`/`isFinished` và **phát KVO** để queue biết:
```swift
class AsyncOperation: Operation, @unchecked Sendable {
    private let lock = NSLock()
    private var _executing = false
    private var _finished = false

    override var isAsynchronous: Bool { true }
    override var isExecuting: Bool { lock.withLock { _executing } }
    override var isFinished: Bool { lock.withLock { _finished } }

    override func start() {
        if isCancelled { finish(); return }          // bị huỷ trước khi chạy → kết thúc luôn
        willChangeValue(forKey: "isExecuting")
        lock.withLock { _executing = true }
        didChangeValue(forKey: "isExecuting")
        main()
    }

    override func main() { fatalError("Subclass phải override main()") }

    func finish() {                                  // subclass gọi khi việc async xong
        willChangeValue(forKey: "isExecuting")
        willChangeValue(forKey: "isFinished")
        lock.withLock { _executing = false; _finished = true }
        didChangeValue(forKey: "isExecuting")
        didChangeValue(forKey: "isFinished")
    }
}

final class FakeDownloadOperation: AsyncOperation, @unchecked Sendable {
    let id: Int
    init(id: Int) { self.id = id }
    override func main() {
        DispatchQueue.global().asyncAfter(deadline: .now() + .milliseconds(50)) {
            if self.isCancelled { print("download \(self.id) bị huỷ"); self.finish(); return }
            print("download \(self.id) xong")
            self.finish()
        }
    }
}
let q2 = OperationQueue()
let ops = (1...3).map { FakeDownloadOperation(id: $0) }
let done = BlockOperation { print("Tất cả download xong") }
ops.forEach { done.addDependency($0) }
q2.addOperations(ops + [done], waitUntilFinished: false)
q2.waitUntilAllOperationsAreFinished()
// download 1/2/3 xong (thứ tự bất kỳ), rồi "Tất cả download xong"
```
Quên phát KVO `isFinished` → operation không bao giờ được coi là xong → dependency treo, queue đầy.

**Khi nào chọn OperationQueue thay GCD?** Khi cần dependency phức tạp, giới hạn số task đồng thời, huỷ hàng loạt (ví dụ huỷ download ảnh của các cell đã cuộn khỏi màn hình), hoặc theo dõi tiến độ. Việc đơn giản "chạy nền rồi về main" → GCD.

### §9. Priority inversion & thread explosion

**Priority inversion:** task ưu tiên **cao** phải chờ một tài nguyên (lock) đang bị task ưu tiên **thấp** giữ; trong khi đó các task ưu tiên **trung bình** chiếm CPU, task thấp không chạy được để nhả lock → task cao bị kẹt lâu.
```
High   (UI):        ──chờ lock────────────────────────────▶ (kẹt)
Medium (utility):   ████████████████████  (chiếm CPU)
Low    (background): [giữ lock]......(không được chạy để unlock)
```
Cách hệ thống giảm thiểu: lock "biết chủ" như `os_unfair_lock`, serial queue + `sync` cho phép **priority donation/QoS override** — tạm nâng QoS của task đang giữ tài nguyên. `DispatchSemaphore` **không biết ai đang giữ** nên không nâng được → tránh dùng semaphore làm lock giữa các QoS khác nhau. Xcode/Instruments có thể cảnh báo "Thread Performance Checker: ... priority inversion".

**Thread explosion:** khi task trên concurrent queue **bị chặn** (`sync`, `semaphore.wait()`, `sleep`, I/O đồng bộ), GCD nghĩ "core đang rảnh" và **tạo thêm thread** để chạy task khác. Ném 1000 task đều chặn → hàng chục/trăm thread:
- Mỗi thread tốn bộ nhớ stack (~512 KB) + chi phí kernel.
- Context switch liên tục → chậm hơn cả chạy tuần tự.
- Có giới hạn số thread của pool (khoảng 64) → hết thread → **deadlock** nếu các task đang chờ một task chưa có thread để chạy.

Phòng tránh: không chặn trong task GCD; dùng ít queue với **mục đích rõ ràng** (một serial queue cho mỗi subsystem) thay vì tạo hàng trăm queue; giới hạn đồng thời bằng `maxConcurrentOperationCount`; dùng `concurrentPerform` cho vòng lặp CPU. **Swift Concurrency** (tuần 7) giải quyết tận gốc bằng cooperative thread pool: số thread ≤ số core, task **suspend** thay vì chặn thread.

---

## ⚠️ Bẫy hay gặp

1. **`DispatchQueue.main.sync` trên main thread** → deadlock. Hỏi phỏng vấn kiểu "đoạn code này trong `viewDidLoad` có vấn đề gì".
2. **"async là chạy trên thread khác"** — sai. `DispatchQueue.main.async` chạy trên main; async chỉ nghĩa là không chờ.
3. **Barrier trên `DispatchQueue.global()`** — không có tác dụng, phải dùng concurrent queue tự tạo.
4. **Chỉ khoá phần ghi, quên khoá phần đọc** → vẫn data race. Đọc Array/Dictionary đang bị ghi có thể crash.
5. **`DispatchWorkItem.cancel()` / `Operation.cancel()` không dừng code đang chạy** — chỉ đặt cờ.
6. **Gọi callback/delegate trong khi đang giữ lock** → callback gọi lại vào object → lock lần 2 → deadlock.
7. **`group.enter()` không cân bằng `leave()`** ở nhánh lỗi → `notify` không bao giờ chạy.
8. **`semaphore.wait()` trên main thread** chờ callback cũng gửi về main → deadlock.
9. **Thứ tự in của code concurrent** — đừng khẳng định một thứ tự cụ thể nếu nó không được đảm bảo.
10. **TSan im lặng ≠ code đúng** — TSan chỉ thấy race trong luồng code đã chạy, và không bắt lỗi logic check-then-act.
11. **`deinit` của object có thể chạy trên thread nền** (nơi release cuối cùng) — đừng giả định main thread.

---

## 🛠 Bài tập

> Tạo file trong `practice/week06/`. Dự đoán trước khi chạy. Chạy mỗi bài **vài lần** — code concurrent có thể cho kết quả khác nhau giữa các lần chạy.

**Bài 1 — sync/async (T2)**
Đoạn sau in ra gì? Thứ tự này có **đảm bảo** không? Giải thích từng cặp.
```swift
import Foundation
let q = DispatchQueue(label: "q")
print("A")
q.async { print("B") }
q.sync { print("C") }
print("D")
q.async { print("E") }
q.sync {}
print("F")
```
<details><summary>Lời giải</summary>

`A B C D E F` — **luôn** đúng thứ tự này:
- B trước C: serial queue FIFO, C được xếp sau B.
- C trước D: `sync` chặn thread hiện tại tới khi C chạy xong.
- D trước E: E chỉ được đưa vào queue sau khi in D.
- E trước F: `q.sync {}` xếp sau E và chặn tới khi chạy xong → E chắc chắn đã in.
So sánh với ví dụ §3: ở đó `print("3")` chạy song song với block async nên không đảm bảo; ở đây mọi điểm "giao nhau" đều có `sync` chốt lại.
</details>

**Bài 2 — Săn deadlock (T3)**
Với mỗi trường hợp, đoán: chạy bình thường / crash / treo. Sau đó chạy thử bằng chương trình nhận tham số (biên dịch `swiftc` rồi chạy `./app b`, `./app c`...). Vẽ sơ đồ thời gian cho các trường hợp không chạy được.
```swift
import Foundation
setvbuf(stdout, nil, _IONBF, 0)     // in ngay, không buffer — để thấy output trước khi crash
let which = CommandLine.arguments[1]
let serial = DispatchQueue(label: "serial")
let conc = DispatchQueue(label: "conc", attributes: .concurrent)
switch which {
case "a": serial.sync { serial.sync { print("a inner") } }; print("a ok")
case "b": serial.sync { serial.async { print("b inner") } }; Thread.sleep(forTimeInterval: 0.1); print("b ok")
case "c": conc.sync { conc.sync { print("c inner") } }; print("c ok")
case "d": DispatchQueue.global().async { DispatchQueue.main.sync { print("d inner") }; print("d ok"); exit(0) }; dispatchMain()
case "e": conc.async { conc.sync(flags: .barrier) { print("e inner") } }; Thread.sleep(forTimeInterval: 1); print("e: inner chưa in → kẹt")
case "f": let s2 = DispatchQueue(label: "s2"); serial.sync { s2.sync { print("f inner") } }; print("f ok")
default: DispatchQueue.main.sync { print("never") }
}
```
<details><summary>Lời giải</summary>

| | Kết quả | Vì sao |
|---|---|---|
| a | 💥 crash (EXC_BREAKPOINT, exit code 133) | sync vào chính serial queue đang chạy |
| b | ✅ `b inner`, `b ok` | async không chờ, block trong chạy sau khi block ngoài xong |
| c | ✅ `c inner`, `c ok` | concurrent queue chạy được block thứ 2 song song |
| d | ✅ `d inner`, `d ok` | sync về main **từ thread nền**; main đang rảnh trong `dispatchMain()` |
| e | ❌ treo: in `e: inner chưa in → kẹt` | barrier phải chờ mọi block trước nó xong — gồm chính block đang gọi nó |
| f | ✅ `f inner`, `f ok` | hai queue khác nhau, không chờ vòng |
| default | 💥 crash | `main.sync` từ main thread |

Lưu ý `d`: `dispatchMain()` giữ main thread phục vụ main queue (trong app iOS, main RunLoop làm việc này).
</details>

**Bài 3 — DispatchGroup + giới hạn đồng thời (T4)**
a) Viết `fetchProfile`, `fetchRepos`, `fetchFollowers` giả lập (dùng `asyncAfter` với độ trễ khác nhau, trả kết quả qua completion). Gọi cả 3 song song, khi **tất cả** xong thì in tổng hợp. Đo thời gian: phải xấp xỉ độ trễ **lớn nhất**, không phải tổng.
b) Tải 6 "file" (mỗi file `Thread.sleep` 0,05s) nhưng **tối đa 2 file cùng lúc** bằng `DispatchSemaphore`. Chứng minh bằng một biến đếm `running`/`maxRunning` (nhớ khoá biến đếm!).
<details><summary>Lời giải b</summary>

```swift
import Foundation
let semaphore = DispatchSemaphore(value: 2)
let group = DispatchGroup()
let lock = NSLock()
var running = 0, maxRunning = 0
for _ in 1...6 {
    DispatchQueue.global().async(group: group) {
        semaphore.wait()
        lock.lock(); running += 1; maxRunning = max(maxRunning, running); lock.unlock()
        Thread.sleep(forTimeInterval: 0.05)
        lock.lock(); running -= 1; lock.unlock()
        semaphore.signal()
    }
}
group.wait()
print("Tối đa chạy cùng lúc:", maxRunning)   // Tối đa chạy cùng lúc: 2
```
</details>

**Bài 4 — Race & TSan (T5)**
a) Viết `UnsafeCounter`, chạy `concurrentPerform` 100.000 lần, in kết quả 3 lần chạy khác nhau.
b) Biên dịch với `-sanitize=thread`, chụp lại cảnh báo của TSan, chỉ ra 2 dòng truy cập xung đột.
c) Sửa bằng 3 cách: serial queue, `NSLock`, `OSAllocatedUnfairLock`. Chạy lại TSan → sạch.
d) Viết `class SafeArray<T>` có `append`, `subscript` get, `count`, `map` dùng concurrent queue + barrier.

**Bài 5 — Debounce & pipeline (T6)**
a) Viết lại `SearchDebouncer` (§5.3) **không nhìn**. Thử gõ 5 chuỗi cách nhau 50ms, chỉ chuỗi cuối được "gọi API".
b) Dựng pipeline bằng `OperationQueue`: `download → [resize, makeThumbnail] → upload` (resize và thumbnail chạy song song, upload chờ cả hai). Huỷ `resize` trước khi chạy và quan sát: `upload` có chạy không? Vì sao?
<details><summary>Gợi ý b</summary>

`upload` **vẫn chạy** — operation bị huỷ chuyển sang `isFinished = true`, và dependency chỉ chờ "finished", không quan tâm thành công hay bị huỷ. Muốn upload bỏ qua khi có bước bị huỷ, trong upload phải kiểm tra `dependencies.contains { $0.isCancelled }`.
</details>

**Bài 6 — Custom AsyncOperation (T7)**
Viết lại `AsyncOperation` (§8) không nhìn. Tạo subclass `DelayOperation(ms:)` dùng `asyncAfter`. Thêm 5 operation vào queue với `maxConcurrentOperationCount = 2` và một `BlockOperation` "Xong" phụ thuộc vào cả 5. Kiểm tra: (1) "Xong" in cuối; (2) thử **xoá** các dòng `willChangeValue`/`didChangeValue` → quan sát chương trình treo; giải thích.

**🏋️ Bài tập lớn (T7) — `ImageLoader` thread-safe với GCD**
Viết `imageloader.swift` (console, giả lập tải bằng hàm `fetch` được inject):
- `load(_ url: URL, completion: @escaping (Result<Data, Error>) -> Void)`
- Cache trong bộ nhớ, **thread-safe**
- **Gộp request trùng**: 20 lời gọi cho 5 URL khác nhau đang tải cùng lúc → chỉ 5 lần `fetch` thật
- Tối đa **4** lần fetch đồng thời
- Completion được gọi trên một `callbackQueue` cấu hình được (mặc định `.main`)
- Kiểm tra bằng TSan: không cảnh báo nào
Sau đó áp dụng vào side project (Milestone 6): ô ảnh avatar của repo owner tải bằng loader này, kết hợp `ImageCache` (NSCache) từ tuần 5.
<details><summary>Lời giải</summary>

```swift
import Foundation

final class ImageLoader {
    typealias Completion = (Result<Data, Error>) -> Void

    private var cache: [URL: Data] = [:]
    private var waiting: [URL: [Completion]] = [:]       // gộp request trùng
    private let stateQueue = DispatchQueue(label: "ImageLoader.state")   // serial: bảo vệ cache + waiting
    private let workQueue: OperationQueue = {
        let q = OperationQueue()
        q.maxConcurrentOperationCount = 4
        q.qualityOfService = .userInitiated
        return q
    }()
    private let callbackQueue: DispatchQueue
    private let fetch: (URL) throws -> Data               // inject để test

    init(callbackQueue: DispatchQueue = .main, fetch: @escaping (URL) throws -> Data) {
        self.callbackQueue = callbackQueue
        self.fetch = fetch
    }

    func load(_ url: URL, completion: @escaping Completion) {
        stateQueue.async {
            if let data = self.cache[url] {
                self.callbackQueue.async { completion(.success(data)) }
                return
            }
            if self.waiting[url] != nil {                   // đang tải → chỉ xếp hàng chờ
                self.waiting[url]!.append(completion)
                return
            }
            self.waiting[url] = [completion]
            self.workQueue.addOperation {
                let result = Result { try self.fetch(url) }
                self.stateQueue.async {
                    if case .success(let data) = result { self.cache[url] = data }
                    let callbacks = self.waiting.removeValue(forKey: url) ?? []
                    self.callbackQueue.async { callbacks.forEach { $0(result) } }
                }
            }
        }
    }
}

// --- test
let fetchCount = NSLock()
var fetches = 0
let loader = ImageLoader(callbackQueue: DispatchQueue(label: "callbacks")) { url in
    fetchCount.lock(); fetches += 1; fetchCount.unlock()
    Thread.sleep(forTimeInterval: 0.1)
    return Data(url.absoluteString.utf8)
}
let group = DispatchGroup()
let urls = (0..<20).map { URL(string: "https://example.com/\($0 % 5).png")! }
for url in urls {
    group.enter()
    loader.load(url) { _ in group.leave() }
}
group.wait()
print("20 request, 5 URL khác nhau → số lần fetch thật:", fetches)   // 5
group.enter()
loader.load(urls[0]) { r in print("lần 2 lấy từ cache:", (try? r.get())?.count ?? -1); group.leave() }  // 25
group.wait()
print("fetch sau khi cache:", fetches)                                  // 5
```
Điểm mấu chốt để nói khi phỏng vấn:
- **Mọi** state (`cache`, `waiting`) chỉ được đọc/ghi trên `stateQueue` → không data race.
- `fetch` chạy **ngoài** `stateQueue` (trên `workQueue`) → không chặn queue state trong lúc tải.
- Completion gọi **ngoài** `stateQueue` → người gọi có gọi lại `load` trong completion cũng không deadlock.
- Trong test, callback dùng queue riêng vì script không chạy main RunLoop; trong app để mặc định `.main`.
- Ở đây cố ý dùng `self` mạnh vì loader thường là singleton; trong app nếu loader có vòng đời ngắn, cân nhắc `[weak self]`.
</details>

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Process và thread khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Process là một chương trình đang chạy với không gian bộ nhớ riêng, cách ly với process khác. Thread là luồng thực thi bên trong process; các thread chia sẻ heap và tài nguyên của process nhưng có stack riêng. Tạo/chuyển thread rẻ hơn process nhưng vì chia sẻ bộ nhớ nên phải đồng bộ khi truy cập dữ liệu chung.
</details>

**Q2 ⭐ Vì sao phải cập nhật UI trên main thread?**
<details><summary>Đáp án gợi ý</summary>

UIKit (và phần lớn AppKit/SwiftUI) không thread-safe; main RunLoop điều phối event, layout và commit Core Animation để render. Cập nhật UI từ thread khác có thể chen vào giữa chu kỳ layout/render → UI không cập nhật, hiển thị sai hoặc crash ngẫu nhiên. Ngược lại, không được chặn main bằng việc nặng vì sẽ làm app đơ. Main Thread Checker giúp phát hiện vi phạm khi debug.
</details>

**Q3 ⭐ Serial queue và concurrent queue khác nhau thế nào? sync vs async?**
<details><summary>Đáp án gợi ý</summary>

Serial queue chạy mỗi lúc một task theo FIFO; concurrent queue có thể chạy nhiều task cùng lúc (bắt đầu theo FIFO, kết thúc thứ tự bất kỳ). `sync` chặn thread gọi cho tới khi task xong; `async` đưa task vào queue rồi trả về ngay. Hai khái niệm độc lập: queue quyết định cách chạy task, sync/async quyết định người gọi có chờ không.
</details>

**Q4 ⭐ Deadlock là gì? Cho ví dụ với GCD.**
<details><summary>Đáp án gợi ý</summary>

Deadlock là khi các bên chờ nhau mãi mãi. Ví dụ: đang chạy trên serial queue Q mà gọi `Q.sync { }` — block mới phải chờ block hiện tại xong, block hiện tại lại chờ block mới. Phổ biến nhất: `DispatchQueue.main.sync` từ main thread. Ngoài ra là lock ordering (2 thread giữ 2 lock chờ chéo). Tránh: không sync vào queue hiện tại, dùng async khi có thể, lấy lock theo thứ tự cố định, không gọi callback khi đang giữ lock.
</details>

**Q5 `DispatchQueue.main.async` từ main thread có chạy ngay không?**
<details><summary>Đáp án gợi ý</summary>

Không. Block được xếp vào cuối main queue và chạy ở một vòng RunLoop sau, khi code hiện tại đã return. Đôi khi dùng chủ ý để "đẩy việc sang vòng sau" (ví dụ chờ layout xong), nhưng lạm dụng là dấu hiệu code có vấn đề về thứ tự.
</details>

**Q6 ⭐ QoS là gì? Kể các mức.**
<details><summary>Đáp án gợi ý</summary>

Quality of Service cho hệ thống biết mức quan trọng của công việc để ưu tiên CPU, I/O và tiết kiệm năng lượng: `userInteractive` (UI, animation), `userInitiated` (user đang chờ kết quả), `default`, `utility` (việc dài có progress), `background` (user không thấy: backup, prefetch). Chọn đúng QoS giúp UI mượt và tiết kiệm pin; hệ thống có thể nâng QoS (override) để tránh priority inversion.
</details>

**Q7 ⭐ DispatchGroup dùng khi nào? `notify` khác `wait` thế nào?**
<details><summary>Đáp án gợi ý</summary>

Khi cần chạy nhiều việc song song và làm gì đó sau khi **tất cả** xong (ví dụ gọi 3 API rồi render). Dùng `enter/leave` cho API async hoặc `async(group:)`. `notify(queue:)` đăng ký block chạy khi nhóm xong, không chặn; `wait()` chặn thread hiện tại cho tới khi xong (có timeout) — không dùng trên main. enter/leave phải cân bằng.
</details>

**Q8 DispatchSemaphore dùng để làm gì? Rủi ro?**
<details><summary>Đáp án gợi ý</summary>

Bộ đếm cho phép tối đa N bên truy cập tài nguyên cùng lúc (giới hạn số download đồng thời), hoặc báo hiệu giữa các thread. Rủi ro: `wait()` chặn thread → nếu trên main thì đơ/deadlock; nhiều thread bị chặn dẫn đến thread explosion; không có priority donation nên dễ priority inversion khi dùng làm lock; quên `signal` thì treo vĩnh viễn.
</details>

**Q9 ⭐ Barrier là gì? Viết reader–writer thread-safe.**
<details><summary>Đáp án gợi ý</summary>

Barrier block trên **concurrent queue tự tạo** chờ mọi block trước nó xong, chạy một mình, rồi mới cho block sau chạy. Dùng cho reader–writer: đọc bằng `queue.sync { }` (song song với nhau), ghi bằng `queue.async(flags: .barrier) { }` (độc quyền). Barrier trên global queue không có tác dụng.
</details>

**Q10 ⭐ Race condition và data race khác nhau thế nào? Phát hiện bằng gì?**
<details><summary>Đáp án gợi ý</summary>

Data race: nhiều thread truy cập cùng vùng nhớ cùng lúc, ít nhất một ghi, không đồng bộ → undefined behavior. Race condition: kết quả phụ thuộc thứ tự thời gian giữa các thread — lỗi logic, có thể xảy ra cả khi không có data race (ví dụ check-then-act với hai lần khoá riêng). Phát hiện data race bằng Thread Sanitizer; race condition cần review logic, giữ invariant trong một critical section, viết test stress.
</details>

**Q11 ⭐ Làm thế nào để một class thread-safe? So sánh các cách.**
<details><summary>Đáp án gợi ý</summary>

Gom mọi truy cập state chia sẻ qua một cơ chế đồng bộ: serial queue (dễ đọc, ghi async được), concurrent queue + barrier (đọc song song, đáng khi đọc nhiều), lock (`NSLock`, `OSAllocatedUnfairLock`, `Mutex` — nhanh nhất cho critical section ngắn), hoặc actor (Swift Concurrency). Ngoài ra: ưu tiên immutable / value type, giảm state chia sẻ. Đơn vị khoá phải bao trọn thao tác nghiệp vụ.
</details>

**Q12 GCD và OperationQueue khác nhau thế nào? Khi nào dùng cái nào?**
<details><summary>Đáp án gợi ý</summary>

OperationQueue xây trên GCD, thêm: dependency giữa operation (cả khác queue), `maxConcurrentOperationCount`, huỷ từng cái hoặc hàng loạt, trạng thái KVO, subclass để đóng gói logic. GCD nhẹ hơn, cú pháp gọn, hợp với việc đơn giản (chạy nền rồi về main, debounce, barrier). Dùng OperationQueue cho pipeline có phụ thuộc, tải ảnh cần giới hạn và huỷ khi cell cuộn đi.
</details>

**Q13 Làm sao viết một Operation bọc tác vụ async (network)?**
<details><summary>Đáp án gợi ý</summary>

Subclass `Operation`, override `isAsynchronous = true`, tự quản `isExecuting`/`isFinished` (thread-safe) và phát KVO `willChangeValue/didChangeValue` cho các key đó. `start()` kiểm tra `isCancelled`, đặt executing, bắt đầu request; khi callback về thì set finished và phát KVO. Không phát KVO thì queue không biết operation xong → dependency treo.
</details>

**Q14 Cancel một DispatchWorkItem / Operation đang chạy thì sao?**
<details><summary>Đáp án gợi ý</summary>

Chỉ đặt cờ cancelled. Nếu chưa chạy thì sẽ bị bỏ qua; nếu đang chạy thì code vẫn tiếp tục trừ khi tự kiểm tra `isCancelled` và dừng. Đây là cancellation hợp tác (cooperative) — Swift Concurrency cũng theo mô hình này.
</details>

**Q15 Priority inversion là gì? iOS xử lý thế nào?**
<details><summary>Đáp án gợi ý</summary>

Task ưu tiên cao chờ tài nguyên do task ưu tiên thấp giữ, trong khi task trung bình chiếm CPU khiến task thấp không chạy được để nhả. Hệ thống giảm thiểu bằng priority donation / QoS override: `os_unfair_lock`, `sync` vào serial queue biết ai đang giữ nên tạm nâng QoS. Semaphore không biết owner nên không nâng được → tránh dùng semaphore làm lock giữa các QoS, giữ critical section ngắn, dùng QoS nhất quán.
</details>

**Q16 Thread explosion là gì? Tránh thế nào?**
<details><summary>Đáp án gợi ý</summary>

Khi nhiều task GCD bị chặn (sync, semaphore, sleep, I/O đồng bộ), GCD tạo thêm thread để giữ CPU bận → hàng trăm thread, tốn bộ nhớ stack, context switch liên tục, có thể chạm giới hạn pool và deadlock. Tránh: không chặn trong task, dùng số queue hợp lý theo subsystem, giới hạn đồng thời (OperationQueue), `concurrentPerform` cho vòng lặp CPU; Swift Concurrency dùng cooperative pool (số thread ≈ số core, suspend thay vì block) nên tránh được tận gốc.
</details>

**Q17 Thread Sanitizer là gì? Hạn chế?**
<details><summary>Đáp án gợi ý</summary>

Công cụ runtime (Scheme → Diagnostics) theo dõi truy cập bộ nhớ và báo data race kèm stack trace của hai truy cập xung đột. Hạn chế: chỉ thấy race trong code thực sự chạy, làm app chậm nhiều lần, không chạy cùng Address Sanitizer, không bắt race condition logic, chỉ chạy trên Simulator/macOS.
</details>

**Q18 Objective-C có `atomic`, Swift thì sao? `atomic` có làm code thread-safe không?**
<details><summary>Đáp án gợi ý</summary>

Swift không có từ khoá atomic cho property. ObjC `atomic` chỉ đảm bảo một lần get hoặc set riêng lẻ không bị xé (torn read), **không** đảm bảo thao tác ghép như `count += 1` hay check-then-act. Trong Swift dùng lock/queue/actor; từ iOS 18 có `Atomic` và `Mutex` trong module `Synchronization`.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–6 không xem lời giải trước
- [ ] Làm xong Bài tập lớn `ImageLoader`, chạy TSan sạch
- [ ] Trả lời được ≥ 14/18 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "sync vs async, serial vs concurrent, và vì sao `main.sync` trên main bị deadlock"
- [ ] Ghi âm 2 phút: "Viết cache thread-safe thế nào, so sánh serial queue / barrier / lock"
- [ ] Vẽ lại từ trí nhớ: sơ đồ deadlock, sơ đồ barrier, sơ đồ priority inversion
- [ ] Làm xong 6 bài DSA tuần 6 (Binary Search)
- [ ] Side project Milestone 6 xong (load ảnh bằng GCD + cache thread-safe)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- Apple — [Dispatch](https://developer.apple.com/documentation/dispatch), [DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue), [OperationQueue](https://developer.apple.com/documentation/foundation/operationqueue), [OSAllocatedUnfairLock](https://developer.apple.com/documentation/os/osallocatedunfairlock)
- Apple — [Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/) (cũ nhưng giải thích queue/operation rất rõ)
- Apple — [Diagnosing memory, thread, and crash issues early](https://developer.apple.com/documentation/xcode/diagnosing-memory-thread-and-crash-issues-early) (Thread Sanitizer, Main Thread Checker)
- WWDC 2017 — *Modernizing Grand Central Dispatch Usage* (**xem bắt buộc**: thread explosion, số queue hợp lý, QoS)
- WWDC 2016 — *Concurrent Programming With GCD in Swift 3* (sync/async, barrier, deadlock, priority inversion)
- WWDC 2015 — *Advanced NSOperations* (dependency, custom operation)
- WWDC 2015 — *Building Responsive and Efficient Apps with GCD* (QoS)
- Swift Evolution — [SE-0433 Synchronous Mutual Exclusion Lock (Mutex)](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md)
- objc.io — *Issue #2: Concurrent Programming* (nền tảng thread, lock, các vấn đề kinh điển)
