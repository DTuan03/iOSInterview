# Tuần 5 — Bộ nhớ & ARC (07/12 – 13/12/2026)

> **Giai đoạn 2 · Bộ nhớ & Concurrency** · Thời lượng: ~14 giờ
> Đây là chủ đề **gần như chắc chắn bị hỏi** ở mọi buổi phỏng vấn iOS. Câu "retain cycle là gì, sửa thế nào" là câu khởi động; câu hỏi thật là **"weak và unowned khác nhau thế nào ở runtime"**, **"closure lồng nhau có leak không"**, **"bạn đã tìm leak bằng công cụ gì, từng bước ra sao"**. Tuần này phải tự tay tạo leak, nhìn thấy nó trong Xcode, rồi sửa.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích stack vs heap, vì sao class nằm trên heap và tốn hơn struct
- [ ] Giải thích ARC hoạt động thế nào (compiler chèn `retain`/`release`, deinit chạy khi strong count = 0) và khác GC ở điểm nào
- [ ] Vẽ được sơ đồ tham chiếu của một retain cycle (class↔class, closure↔self) và chỉ ra chỗ cắt
- [ ] Phân biệt `strong` / `weak` / `unowned`: hành vi khi object chết, khi nào dùng cái nào
- [ ] Viết đúng capture list `[weak self]`, `guard let self`, và biết **khi nào KHÔNG cần** `[weak self]`
- [ ] Nhận diện leak từ Timer, NotificationCenter block observer, delegate, closure lồng nhau
- [ ] Tìm một leak thật bằng **Memory Graph Debugger** và **Instruments (Leaks + Allocations)** theo từng bước
- [ ] Giải thích `NSCache` khác `Dictionary` ở điểm nào, khi nào còn cần `autoreleasepool`
- [ ] Làm xong 6 bài DSA tuần 5 — chủ đề **Stack** (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 07/12** | 2h | Tạo `practice/week05/`. §1 Stack vs heap, §2 **ARC** cơ bản + `deinit`. Bài tập 1 · DSA: Valid Parentheses | `day1.swift` in đúng thứ tự `init`/`deinit` |
| **T3 08/12** | 2h | §3 **Retain cycle** giữa 2 class, §4 **weak vs unowned**. Bài tập 2 · DSA: Min Stack | `day2.swift` — thấy leak rồi sửa |
| **T4 09/12** | 2h | §5 **Closure & capture list** (đọc kỹ nhất tuần). Bài tập 3 · DSA: Evaluate Reverse Polish Notation | `day3.swift` |
| **T5 10/12** | 2h | §6 Leak "đời thật": delegate, Timer, NotificationCenter, closure lồng nhau. Bài tập 4 · DSA: Daily Temperatures | `day4.swift` |
| **T6 11/12** | 1,5h | §7 Bên trong ARC (refcount, side table), §8 autoreleasepool, value type & bộ nhớ, **NSCache**. Bài tập 5 · DSA: Implement Queue using Stacks | `day5.swift` |
| **T7 12/12** | 4h | §9 **Memory Graph + Instruments** làm theo từng bước trên project thật. Bài tập 6 + Bài tập lớn. Side project: Milestone 5 (check leak bằng Memory Graph, thêm `ImageCache` dùng `NSCache`) · DSA: Decode String | Project `LeakLab`, side project có `ImageCache` |
| **CN 13/12** | 2h | Tự trả lời 18 câu phỏng vấn (viết trước, mở đáp án sau). Ghi âm 2 câu. Vẽ lại 3 sơ đồ retain cycle trên giấy. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

> Tất cả code non-UI trong file này chạy được bằng `swift file.swift` (mặc định Swift 5 language mode). Code có `import UIKit` thì gõ vào project iOS / Playground iOS.

### §1. Stack vs Heap — ôn lại từ gốc

Khi app chạy, bộ nhớ của mỗi thread có một vùng **stack**, cả process dùng chung một vùng **heap**.

```
            ┌────────────────────────────┐
 Thread 1 → │ STACK                      │  frame của hàm đang chạy:
            │  let n = 5        (8 byte) │  biến local, tham số, địa chỉ trả về
            │  let p: Point     (16 byte)│  → cấp phát = dịch con trỏ stack (cực rẻ)
            │  let user ────────────┐    │  → tự giải phóng khi hàm return
            └───────────────────────│────┘
                                    ▼
            ┌────────────────────────────┐
 Mọi thread │ HEAP                       │  object class:
 dùng chung │  [metadata | refcount |    │  → cấp phát = tìm ô trống (malloc), có lock
            │   name | age ...]          │  → ai giải phóng? → ARC đếm tham chiếu
            └────────────────────────────┘
```

| | Stack | Heap |
|---|---|---|
| Cấp phát | Dịch con trỏ stack, O(1), gần như miễn phí | `malloc`: tìm vùng trống, đồng bộ giữa các thread |
| Giải phóng | Tự động khi hàm return | Phải có người quyết định → **ARC** |
| Phạm vi | Riêng từng thread → không có data race | Dùng chung → cần đồng bộ |
| Chứa gì | Value type local (struct, enum, Int...), **con trỏ** tới object | Instance của class, closure có capture, buffer của Array/String/Dictionary |

**Điểm hay bị hỏi:**
- "Struct nằm trên stack" là **đúng phần lớn, không tuyệt đối**. Struct nằm trên heap khi: là thuộc tính của một class, bị capture bởi escaping closure và có thể bị thay đổi, bị bọc trong existential (`any Protocol`) lớn hơn 3 word, hoặc là `indirect enum`.
- `Array`, `String`, `Dictionary` là struct, nhưng **buffer chứa phần tử nằm trên heap** (struct chỉ giữ con trỏ tới buffer — đó là nền của copy-on-write, tuần 2).
- Object class trên heap có **header 16 byte** (64-bit): 8 byte con trỏ metadata (kiểu của object) + 8 byte **refcount**. Biến `let user = User()` trên stack chỉ là con trỏ 8 byte trỏ vào heap.

```swift
final class Ref {}
print(MemoryLayout<Int>.size, MemoryLayout<Ref>.size, MemoryLayout<String>.size, MemoryLayout<[Int]>.size)
// 8 8 16 8   ← biến kiểu class chỉ là con trỏ 8 byte; Array cũng chỉ là 1 con trỏ tới buffer
```

### §2. ARC — Automatic Reference Counting

**Ý tưởng:** mỗi object class có một bộ đếm **strong reference count** (RC). Có thêm một tham chiếu mạnh → RC + 1 (**retain**). Mất một tham chiếu mạnh → RC − 1 (**release**). RC về 0 → `deinit` chạy **ngay lập tức** và bộ nhớ được thu hồi.

"Automatic" nghĩa là **compiler tự chèn lệnh retain/release lúc compile**, không phải một tiến trình chạy nền như Garbage Collector của Java/Kotlin.

```swift
class Person {
    let name: String
    init(name: String) { self.name = name; print("init \(name)") }
    deinit { print("deinit \(name)") }
}

var a: Person? = Person(name: "An")   // RC = 1
var b = a                             // RC = 2
a = nil                               // RC = 1 → chưa deinit
print("a = nil xong")
b = nil                               // RC = 0 → deinit chạy NGAY
print("b = nil xong")

do {
    let p = Person(name: "Bình")
    print("trong scope: \(p.name)")
}   // hết scope → p bị release → deinit
print("ra khỏi scope")
```
Output:
```
init An
a = nil xong
deinit An
b = nil xong
init Bình
trong scope: Bình
deinit Bình
ra khỏi scope
```

Muốn tận mắt thấy compiler chèn retain? Chạy `swiftc -emit-ir -O file.swift | grep swift_retain` — bạn sẽ thấy các lời gọi `@swift_retain` / `@swift_release` trong code máy sinh ra.

**ARC vs Garbage Collection — câu hỏi kinh điển:**
| | ARC (Swift/ObjC) | Tracing GC (Java, Kotlin, C#) |
|---|---|---|
| Khi nào thu hồi | **Ngay** khi RC = 0, xác định được (deterministic) | Khi GC chạy, không biết trước |
| Chi phí | Mỗi lần retain/release là một **phép toán atomic** (rải đều) | Gom lại, có thể "stop-the-world" gây giật |
| Retain cycle | **Không tự xử lý được** → lập trình viên phải dùng weak/unowned | GC phát hiện được object không còn tới được |
| `deinit` | Chạy đúng lúc, dùng được để dọn tài nguyên | `finalize` không đảm bảo thời điểm |

→ Vì sao Apple chọn ARC: bộ nhớ dự đoán được, không giật UI, ít tốn RAM hơn GC (GC cần dư bộ nhớ để chạy hiệu quả) — quan trọng trên điện thoại.

**Ghi chú nâng cao (hay gây bất ngờ):** Swift **không hứa** object sống tới hết `}` của scope. Optimizer được phép release ngay sau **lần dùng cuối**. Nếu cần object sống tới một điểm nhất định (ví dụ giữ một observer), dùng `withExtendedLifetime(obj) { ... }`.

### §3. Retain cycle — khi ARC bó tay

Hai object giữ nhau **mạnh** → RC của cả hai không bao giờ về 0 dù bên ngoài không còn ai dùng → **memory leak**.

```swift
class Owner {
    let name: String
    var pet: Pet?
    init(name: String) { self.name = name }
    deinit { print("deinit Owner \(name)") }
}
class Pet {
    let name: String
    var owner: Owner?          // ❌ strong → cycle
    init(name: String) { self.name = name }
    deinit { print("deinit Pet \(name)") }
}
var owner: Owner? = Owner(name: "Chi")
var pet: Pet? = Pet(name: "Mèo")
owner?.pet = pet
pet?.owner = owner
owner = nil
pet = nil
print("Không thấy dòng deinit nào → leak")
```

Sơ đồ tham chiếu:
```
TRƯỚC khi gán nil                     SAU khi owner = nil; pet = nil
                                      
 var owner ──strong──▶ ┌───────┐      ┌───────┐ ──pet (strong)──▶ ┌─────┐
                       │ Owner │      │ Owner │                   │ Pet │
 var pet ──strong──┐   │ RC=2  │      │ RC=1  │ ◀──owner (strong)─│RC=1 │
                   ▼   └───────┘      └───────┘                   └─────┘
                 ┌─────┐                 ▲ không còn biến nào bên ngoài trỏ tới
                 │ Pet │                 │ nhưng RC ≠ 0 → không bao giờ deinit
                 │RC=2 │                 └── "đảo cô lập" = LEAK
                 └─────┘
```

**Cách sửa:** một trong hai chiều phải **không giữ mạnh**. Chọn chiều "con trỏ ngược về cha / người sở hữu":

```swift
class Pet2 {
    let name: String
    weak var owner: Owner2?     // ✅ weak: không tăng RC
    init(name: String) { self.name = name }
    deinit { print("deinit Pet2 \(name)") }
}
class Owner2 {
    let name: String
    var pet: Pet2?
    init(name: String) { self.name = name }
    deinit { print("deinit Owner2 \(name)") }
}
var o2: Owner2? = Owner2(name: "Dũng")
var p2: Pet2? = Pet2(name: "Chó")
o2?.pet = p2
p2?.owner = o2
o2 = nil                                  // in: deinit Owner2 Dũng
print("p2?.owner =", p2?.owner as Any)    // in: p2?.owner = nil  ← weak tự thành nil
p2 = nil                                  // in: deinit Pet2 Chó
```

**Quy tắc chọn chiều nào weak:** xác định **quan hệ sở hữu** (ownership). Cha sở hữu con → cha giữ strong; con trỏ về cha → weak. Ví dụ: `UIViewController` giữ `view` (strong), `view` giữ `delegate`/`dataSource` trỏ ngược về VC (weak).

### §4. strong vs weak vs unowned

```swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) { self.name = name }
    deinit { print("deinit Customer") }
}
class CreditCard {
    let number: String
    unowned let customer: Customer      // thẻ KHÔNG BAO GIỜ tồn tại lâu hơn khách hàng
    init(number: String, customer: Customer) { self.number = number; self.customer = customer }
    deinit { print("deinit CreditCard") }
}
var c: Customer? = Customer(name: "An")
c!.card = CreditCard(number: "1234", customer: c!)
c = nil
// deinit Customer
// deinit CreditCard
```

Nếu đoán sai vòng đời — object bị trỏ `unowned` chết trước — app **crash ngay** (không phải undefined behavior như C, mà là trap có chủ đích):
```swift
class Owner { deinit { print("deinit Owner") } }
class Tag {
    unowned let owner: Owner
    init(owner: Owner) { self.owner = owner }
}
var o: Owner? = Owner()
let tag = Tag(owner: o!)
o = nil                 // deinit Owner
print(tag.owner)        // 💥 Fatal error: Attempted to read an unowned reference but object 0x... was already destroyed
```

| | `strong` (mặc định) | `weak` | `unowned` |
|---|---|---|---|
| Tăng strong RC? | Có | Không | Không |
| Kiểu biến | Bất kỳ | **Luôn Optional** (`weak var x: T?`) | Thường non-optional (`unowned let x: T`), từ Swift 5 cho phép `unowned var x: T?` |
| Khi object đã deinit | — | Đọc ra `nil` | Đọc → **crash** |
| `var` / `let` | Cả hai | Trước đây chỉ `var`; từ Swift 6.2 cho phép `weak let` | Cả hai |
| Chi phí | retain/release | Tạo **side table** lần đầu, đọc phải kiểm tra | Rẻ hơn weak một chút |
| Dùng khi | Quan hệ sở hữu | Bên kia **có thể** chết trước (delegate, parent, closure async) | Bên kia **chắc chắn** sống lâu hơn hoặc bằng (con trỏ về owner có vòng đời ràng buộc) |

**Lời khuyên thực tế:** khi phân vân → dùng `weak`. Crash vì `unowned` thường xảy ra sau khi code được sửa đổi (ai đó đổi vòng đời mà không biết có `unowned`). Chỉ dùng `unowned` khi vòng đời được bảo đảm **bởi cấu trúc** (ví dụ object con được tạo trong `init` của cha và chỉ cha giữ nó).

`weak` chỉ áp dụng cho **class** (reference type). Protocol muốn dùng cho delegate `weak` phải giới hạn class:
```swift
protocol Delegate { func done() }
// class Worker { weak var delegate: Delegate? }
// ❌ error: 'weak' must not be applied to non-class-bound 'any Delegate'; consider adding a protocol conformance that has a class bound

protocol SafeDelegate: AnyObject { func done() }    // ✅ ràng buộc class
class Worker { weak var delegate: SafeDelegate? }
```

### §5. Closure & capture list — phần quan trọng nhất tuần

Closure là **reference type**. Khi closure dùng một biến bên ngoài, nó **capture** biến đó. Nếu biến đó là `self` (class), closure giữ **strong** reference tới `self`. Nếu `self` lại lưu closure trong một property → cycle.

```swift
class Downloader {
    var onFinish: (() -> Void)?
    var count = 0
    func start() {
        onFinish = {
            self.count += 1          // closure giữ self mạnh → cycle
        }
    }
    deinit { print("deinit Downloader") }
}
var d: Downloader? = Downloader()
d?.start()
d = nil
print("Downloader leak (không có deinit)")
```
```
  self (Downloader) ──onFinish (strong)──▶ closure
          ▲                                   │
          └────────── capture self (strong) ──┘
```

**Sửa bằng capture list** `[weak self]`:
```swift
class Downloader2 {
    var onFinish: (() -> Void)?
    var count = 0
    func start() {
        onFinish = { [weak self] in
            guard let self else { return }   // Swift 5.7+: viết tắt của guard let self = self
            self.count += 1                  // trong phần còn lại, self là strong tạm thời
        }
    }
    deinit { print("deinit Downloader2") }
}
var d2: Downloader2? = Downloader2()
d2?.start()
d2?.onFinish?()
d2 = nil        // in: deinit Downloader2
```

**`guard let self` có tạo lại cycle không?** Không. Nó tạo một tham chiếu strong **local, chỉ sống trong một lần closure chạy**. Hết lần chạy → release. Lợi ích: trong suốt thân closure, `self` không thể biến mất giữa chừng (không bị "nửa chạy nửa nil" như khi viết `self?.a(); self?.b()`).

**Capture list còn làm một việc khác: chụp giá trị tại thời điểm tạo closure.**
```swift
var x = 1
let printX = { print("x =", x) }                 // capture biến x (tham chiếu tới ô nhớ)
let printCaptured = { [x] in print("captured x =", x) }   // copy giá trị x lúc này
x = 2
printX()          // x = 2
printCaptured()   // captured x = 1
```

**Khi nào KHÔNG cần `[weak self]`** — câu hỏi phân loại ứng viên:

| Tình huống | Có cycle? | Cần weak? |
|---|---|---|
| Closure **non-escaping**: `map`, `filter`, `forEach`, `sorted(by:)` | Không — closure chết khi hàm trả về | Không |
| `UIView.animate(withDuration:) { self.view.alpha = 0 }` | Không — UIKit giữ closure tạm thời rồi thả | Không |
| `DispatchQueue.main.async { self.reload() }` | Không phải cycle, chỉ **kéo dài** đời sống self tới khi block chạy | Thường không; dùng weak nếu không muốn giữ VC sống thêm |
| `self.handler = { self.x }` (self lưu closure) | **Có** | **Có** |
| `lazy var greeting: () -> String = { "Hi \(self.name)" }` | **Có** (lazy var lưu closure trong self) | **Có** |
| Network callback lâu (vài giây), self là VC | Không cycle, nhưng giữ VC sống sau khi user đã thoát màn hình | Nên `weak` để không làm việc thừa |
| `Timer` repeat, `NotificationCenter` block, `cell.onTap` lưu trong cell do VC giữ | **Có** (gián tiếp) | **Có** |

> Nguyên tắc ngắn gọn: **hỏi "ai đang giữ closure này?"**. Nếu câu trả lời dẫn ngược về `self` (trực tiếp hoặc qua chuỗi) → cycle → weak. Nếu closure chỉ được giữ tạm bởi hệ thống rồi thả → không cycle.

**`[unowned self]`** cũng cắt được cycle nhưng crash nếu closure chạy sau khi self chết. Chỉ dùng khi closure chắc chắn không sống lâu hơn self (ví dụ closure được lưu trong chính self và chỉ self gọi đồng bộ). Với code async → **luôn `weak`**.

### §6. Leak "đời thật" hay gặp trong app iOS

**6.1. Delegate** — luôn `weak`, protocol phải `AnyObject`:
```swift
protocol DownloadDelegate: AnyObject { func didFinish(_ data: String) }
final class Downloader {
    weak var delegate: DownloadDelegate?
    func run() { delegate?.didFinish("xong") }
    deinit { print("deinit Downloader") }
}
final class ListScreen: DownloadDelegate {
    let downloader = Downloader()          // màn hình SỞ HỮU downloader → strong
    init() { downloader.delegate = self }   // downloader trỏ ngược → weak
    func didFinish(_ data: String) { print("nhận:", data) }
    deinit { print("deinit ListScreen") }
}
var screen: ListScreen? = ListScreen()
screen?.downloader.run()   // nhận: xong
screen = nil               // deinit ListScreen, rồi deinit Downloader
```

**6.2. Timer** — `RunLoop` giữ `Timer`, `Timer` giữ block (hoặc `target`). Nếu block giữ `self` mạnh, `self` không bao giờ chết; và `deinit` (nơi bạn định `invalidate()`) **không bao giờ được gọi**.
```
RunLoop ──▶ Timer ──▶ closure ──strong──▶ self (VC) ──▶ timer
  (RunLoop sống suốt đời app → chuỗi này không bao giờ đứt)
```
```swift
import Foundation
final class Clock {
    var timer: Timer?
    var ticks = 0
    func start() {
        timer = Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) { [weak self] _ in
            guard let self else { return }
            self.ticks += 1
            print("tick", self.ticks)
        }
    }
    func stop() { timer?.invalidate(); timer = nil }
    deinit { timer?.invalidate(); print("deinit Clock") }   // weak self → deinit được gọi → invalidate
}
var clock: Clock? = Clock()
clock?.start()
RunLoop.main.run(until: Date().addingTimeInterval(0.35))   // tick 1, tick 2, tick 3
clock = nil                                                // deinit Clock
```
Bản cũ `Timer.scheduledTimer(timeInterval:target:selector:...)` giữ **`target` mạnh** — `[weak self]` không áp dụng được. Phải `invalidate()` ở chỗ khác (`viewWillDisappear`) hoặc chuyển sang API block.

**6.3. NotificationCenter block observer** — `addObserver(forName:object:queue:using:)` trả về một token; NotificationCenter **giữ block mạnh** cho tới khi bạn `removeObserver(token)`.
```swift
extension Notification.Name { static let didLogin = Notification.Name("didLogin") }

final class Profile {
    var token: NSObjectProtocol?
    init() {
        token = NotificationCenter.default.addObserver(forName: .didLogin, object: nil, queue: .main) { [weak self] _ in
            self?.reload()
        }
    }
    func reload() { print("reload") }
    deinit {
        if let token { NotificationCenter.default.removeObserver(token) }
        print("deinit Profile")
    }
}
```
Nếu quên `[weak self]`: object không bao giờ deinit **và vẫn tiếp tục nhận notification** (chạy `reload()` trên một màn hình đã đóng) — bug rất khó hiểu. Lưu ý: API kiểu `addObserver(self, selector:...)` (selector-based) thì từ iOS 9 **không** giữ observer mạnh và tự huỷ đăng ký — chỉ API **block** mới cần cẩn thận.

**6.4. Closure lồng nhau — bẫy tinh vi:**
```swift
final class Screen {
    var handler: (() -> Void)?
    func setup() {
        handler = {                       // outer: KHÔNG có capture list
            DispatchQueue.main.async { [weak self] in
                print(self as Any)
            }
        }
    }
    func setupFixed() {
        handler = { [weak self] in        // ✅ weak ở closure NGOÀI CÙNG được lưu trữ
            DispatchQueue.main.async {
                print(self as Any)        // dùng lại self (đã là weak, Optional)
            }
        }
    }
    deinit { print("deinit Screen") }
}
var s: Screen? = Screen()
s?.setup()
s = nil            // ❌ không in deinit → LEAK
var s2: Screen? = Screen()
s2?.setupFixed()
s2 = nil           // deinit Screen
```
Vì sao leak? Để closure trong có thể capture `self` (dù weak), closure **ngoài** phải capture `self` trước — và nó capture **strong**. Closure ngoài được lưu trong `self.handler` → cycle. **Quy tắc: đặt `[weak self]` ở closure ngoài cùng mà có thể bị lưu trữ.**

**6.5. Cell callback trong UITableView** — VC giữ tableView → giữ cell → giữ closure → nếu closure giữ VC mạnh thì cycle:
```swift
import UIKit

final class AvatarCell: UITableViewCell {
    var onTapFollow: (() -> Void)?
}
final class FeedViewController: UITableViewController {
    var users: [String] = []
    func follow(_ name: String) {}
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Avatar", for: indexPath) as! AvatarCell
        let name = users[indexPath.row]          // capture giá trị, không capture indexPath (cell bị reuse)
        cell.onTapFollow = { [weak self] in
            self?.follow(name)
        }
        return cell
    }
}
```

### §7. Bên trong ARC: refcount, side table, vòng đời object

Interviewer ở mức 2 năm hay hỏi: **"weak tự thành nil bằng cách nào?"**

Mỗi object Swift có một trường refcount 64-bit ngay trong header (inline), chứa:
- **strong RC** — số tham chiếu strong
- **unowned RC** — số tham chiếu unowned (+1 đại diện cho toàn bộ strong refs)
- các cờ (đang deinit, ...)

Khi lần đầu có một `weak` trỏ tới object, runtime tạo một **side table** (một khối nhỏ riêng trên heap), chuyển các bộ đếm sang đó và trường inline trở thành con trỏ tới side table. **Biến weak trỏ vào side table, không trỏ thẳng vào object.**

```
  weak var w ───────────▶ ┌──────────────┐
                          │ SIDE TABLE   │──── object ptr ──▶ ┌───────────────┐
  strong var s ─────────────────────────────────────────────▶ │ OBJECT        │
                          │ strong RC    │                    │ metadata      │
                          │ unowned RC   │ ◀── inline field ──│ refcount* ────┘
                          │ weak RC      │                    │ properties... │
                          └──────────────┘                    └───────────────┘
```

Vòng đời object:
```
 LIVE ──strong RC = 0──▶ DEINITED ──unowned RC = 0──▶ FREED ──weak RC = 0──▶ DEAD
  │                        │  deinit đã chạy             │ bộ nhớ object        │ side table
  │                        │  đọc weak  → nil            │ đã trả lại           │ cũng được xoá
  │                        │  đọc unowned → CRASH (trap) │                      │
```
- Đọc `weak`: runtime vào side table, thấy object đã qua deinit → trả `nil`. Vì đi qua side table nên **không bao giờ đọc trúng vùng nhớ rác**.
- Đọc `unowned`: kiểm tra cờ deinit ngay trên object (object chưa bị free vì unowned RC > 0 giữ lại phần bộ nhớ) → nếu đã deinit thì trap có chủ đích. Còn `unowned(unsafe)` thì không kiểm tra gì — giống con trỏ C, đọc rác → **undefined behavior**. Gần như không bao giờ nên dùng.
- Retain/release là **phép toán atomic** (vì nhiều thread có thể cùng retain một object) → tốn hơn phép cộng thường. Đây là một lý do struct (không refcount) nhanh hơn class trong vòng lặp nóng.

**Đọc retain count?** `CFGetRetainCount` có tồn tại nhưng **không tin được** (optimizer thêm/bớt retain tạm thời). Trong phỏng vấn nói: "Không nên dựa vào retain count, kiểm tra bằng `deinit`, weak reference trong test, hoặc Memory Graph."

API hữu ích: `isKnownUniquelyReferenced(&obj)` — trả `true` nếu chỉ có đúng 1 strong reference. Đây là cách các collection cài copy-on-write.

### §8. autoreleasepool, value type & bộ nhớ, NSCache

**`autoreleasepool` — còn cần không?**
Đây là di sản Objective-C (MRC/ARC của ObjC): một số API Foundation/UIKit trả object "autoreleased" — sẽ được release khi pool hiện tại được "xả" (cuối mỗi vòng lặp RunLoop trên main thread). Trong **vòng lặp dài tạo nhiều object ObjC tạm** (`UIImage(contentsOfFile:)`, `NSString`, `Data` bridging, xử lý hàng nghìn ảnh), bộ nhớ đỉnh tăng vọt vì pool chưa kịp xả. Bọc thân vòng lặp trong `autoreleasepool` để xả sau mỗi vòng:
```swift
for i in 0..<3 {
    autoreleasepool {
        let s = NSString(format: "item %d", i)
        _ = s.length
    }   // object autoreleased trong vòng này được giải phóng ngay tại đây
}
```
Code Swift thuần (struct, class Swift) **không** dùng autorelease → không cần. GCD queue và thread của Swift Concurrency tự có pool. Kết luận cho phỏng vấn: *"Hiếm khi cần; chỉ dùng trong vòng lặp lớn gọi API Objective-C tạo nhiều object tạm, khi Instruments cho thấy memory peak cao."*

**Value type & bộ nhớ:**
- Struct không có refcount riêng, nhưng **struct chứa class** thì copy struct = retain class bên trong:
  ```swift
  final class Ref { deinit { print("deinit Ref") } }
  struct Holder { var ref: Ref }
  var h1: Holder? = Holder(ref: Ref())
  var h2 = h1
  h1 = nil
  print("h1 nil, Ref vẫn sống vì h2 còn giữ")
  h2 = nil        // deinit Ref
  ```
- Struct có nhiều property kiểu class/String/Array → mỗi lần copy phải retain nhiều lần → copy đắt hơn bạn tưởng.
- Struct **không tạo được retain cycle với chính nó** (không có tham chiếu), nhưng closure lưu trong struct vẫn có thể capture class và tạo cycle ở phía class.

**NSCache vs Dictionary:**

| | `Dictionary` | `NSCache` |
|---|---|---|
| Thread-safe | ❌ (cần tự khoá) | ✅ |
| Tự giải phóng khi thiếu RAM | ❌ — phải tự nghe memory warning | ✅ tự evict khi hệ thống báo áp lực bộ nhớ |
| Giới hạn | Không | `countLimit`, `totalCostLimit` (giới hạn "mềm", không đảm bảo chính xác) |
| Copy key | Key là value type | **Không copy key**; key và value phải là class (`AnyObject`) |
| Thứ tự evict | — | Không đảm bảo (không phải LRU chính xác) |
| Liệt kê toàn bộ | Có | Không có API liệt kê |

```swift
final class ImageBox { let bytes: Int; init(_ b: Int) { bytes = b } }
let cache = NSCache<NSString, ImageBox>()
cache.countLimit = 2
cache.setObject(ImageBox(10), forKey: "a")
cache.setObject(ImageBox(20), forKey: "b")
cache.setObject(ImageBox(30), forKey: "c")   // vượt countLimit → một entry bị evict
// Lần chạy thử in ra ["b", "c"], nhưng tài liệu KHÔNG đảm bảo entry nào bị đẩy ra
```

Key là struct (ví dụ `PhotoKey(id:size:)`) → bọc trong class con của `NSObject`, override `hash` và `isEqual`:
```swift
// (tiếp tục đoạn trên — dùng lại ImageBox)
struct PhotoKey: Hashable { let id: Int; let size: Int }
final class WrappedKey: NSObject {
    let key: PhotoKey
    init(_ key: PhotoKey) { self.key = key }
    override var hash: Int { key.hashValue }
    override func isEqual(_ object: Any?) -> Bool {
        guard let other = object as? WrappedKey else { return false }
        return other.key == key
    }
}
let c2 = NSCache<WrappedKey, ImageBox>()
c2.setObject(ImageBox(1), forKey: WrappedKey(PhotoKey(id: 1, size: 100)))
print(c2.object(forKey: WrappedKey(PhotoKey(id: 1, size: 100)))?.bytes as Any)   // Optional(1)
```

**Memory warning & Jetsam:** iOS không có swap cho app thường. Khi RAM căng, hệ thống gửi **memory warning** (`UIApplication.didReceiveMemoryWarningNotification`, `didReceiveMemoryWarning()` trong VC, `applicationDidReceiveMemoryWarning(_:)`). Nếu app vẫn vượt ngưỡng, **Jetsam** kill app không báo trước (không có crash log thông thường, chỉ có "Jetsam event" report). Việc của bạn: giải phóng cache, ảnh, dữ liệu tạo lại được. Thử trên Simulator: **Debug → Simulate Memory Warning**.

```swift
import UIKit

// Cache bằng Dictionary phải tự dọn khi có memory warning
final class DictionaryImageCache {
    private var storage: [URL: UIImage] = [:]
    private var token: NSObjectProtocol?

    init() {
        token = NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            self?.storage.removeAll()
        }
    }
    deinit { token.map(NotificationCenter.default.removeObserver) }
}
```

### §9. Tìm leak bằng công cụ — làm theo từng bước

> Đây là phần **"kể bằng kinh nghiệm"** trong phỏng vấn. Đừng chỉ đọc — làm trên project `LeakLab` (Bài tập lớn) hoặc side project.

**Bước 0 — Chuẩn bị:** Scheme → Edit Scheme → Run → **Diagnostics** → bật **Malloc Stack Logging: Live Allocations Only**. Nhờ vậy Memory Graph cho biết object được tạo **ở dòng code nào**.

**A. Memory Graph Debugger (Xcode)**
1. Chạy app (Debug). Vào màn hình nghi ngờ rồi **thoát ra** (push rồi pop, present rồi dismiss) — lặp lại 3–5 lần.
2. Bấm nút **Debug Memory Graph** (biểu tượng 3 hình tròn nối nhau trên thanh debug, cạnh nút View Hierarchy). App tạm dừng.
3. Panel trái liệt kê mọi kiểu đang sống kèm **số instance**. Gõ tên module/class của bạn vào ô filter dưới cùng. Dấu hiệu leak: `DetailViewController (5)` trong khi màn hình đã đóng hết.
4. Bộ lọc **"Show only leaked blocks"** (biểu tượng ⚠️ tím ở dưới panel) — Xcode chỉ đánh dấu được những cycle **không còn đường nào từ root tới** (leak thật). Object bị giữ bởi singleton (vẫn tới được) **không** bị đánh dấu → phải tự đếm instance.
5. Chọn một instance → khung giữa vẽ **đồ thị tham chiếu**: đường **đậm** là strong reference, đường **mờ/xám** là weak/unowned hoặc không xác định. Lần theo đường đậm tạo thành vòng → đó là cycle. Thường thấy một node tên `closure context` hoặc `__NSObserver`/`Timer` nằm trong vòng.
6. Mở **Memory Inspector** (panel phải) → xem **Backtrace** nơi object được cấp phát (nhờ Bước 0).
7. Sửa code (thêm `[weak self]`, `weak var delegate`, `invalidate()`), chạy lại, lặp lại bước 1–3: số instance phải về 0 (hoặc 1 nếu màn hình đang hiển thị).

Mẹo: **File → Export Memory Graph** ra file `.memgraph`, rồi dùng terminal `leaks App.memgraph` hoặc `heap App.memgraph` để phân tích/gửi cho đồng nghiệp.

**B. Instruments — Leaks**
1. **Product → Profile** (⌘I) → build Release-like → chọn template **Leaks** (gồm track Allocations + Leaks).
2. Bấm Record, thao tác như bước A.1.
3. Track Leaks định kỳ chụp snapshot; khi có leak sẽ xuất hiện **dấu ✕ đỏ**. Có thể bấm nút chụp snapshot thủ công.
4. Chọn đoạn có leak → bảng dưới liệt kê object bị leak. Đổi view sang **Cycles & Roots** để thấy sơ đồ vòng tham chiếu; chọn một dòng → panel phải hiện **stack trace** cấp phát → double-click nhảy về code.

**C. Instruments — Allocations (tìm "abandoned memory" — thứ Leaks không bắt được)**
Leak có 2 loại: **leaked** (không còn ai tới được — Leaks bắt được) và **abandoned** (vẫn tới được nhưng không bao giờ dùng lại — ví dụ VC bị nhét vào một array tĩnh, cache không giới hạn). Loại thứ hai chỉ thấy được qua tăng trưởng bộ nhớ:
1. Template **Allocations** → Record.
2. Đưa app về trạng thái gốc (màn hình chính) → bấm **Mark Generation** (Generation A).
3. Vào màn hình X rồi thoát ra → **Mark Generation** (B). Lặp lại vài lần (C, D...).
4. Mỗi generation hiển thị **Growth** — bộ nhớ được cấp phát trong khoảng đó **mà vẫn còn sống**. Nếu thao tác "vào rồi ra" lẽ ra trả về đúng trạng thái cũ mà mỗi generation vẫn tăng vài trăm KB → abandoned memory. Mở rộng generation để xem class nào còn sống.
5. Lọc theo tên class, chọn **Created & Persistent**, cột **# Persistent** cho biết còn bao nhiêu instance sống.

**D. Mẹo không cần công cụ**
- In trong `deinit` của mọi VC/ViewModel khi debug (`deinit { print("deinit \(Self.self)") }`).
- Symbolic breakpoint hoặc log trong `deinit` để thấy màn hình đóng mà không deinit.
- Unit test kiểm tra leak bằng weak reference (xem Bài tập 5).

---

## ⚠️ Bẫy hay gặp

1. **"Dùng `[weak self]` ở mọi closure cho chắc"** — sai tư duy. Interviewer muốn nghe bạn biết closure non-escaping và `UIView.animate` không gây cycle. Lạm dụng weak làm code rối và đôi khi gây bug (công việc bị bỏ dở âm thầm vì `self` đã nil).
2. **`[weak self]` đặt ở closure trong** khi closure ngoài được lưu trữ → vẫn leak (§6.4).
3. **`unowned` trong callback async** (network, timer, animation completion) → crash khi user thoát màn hình trước khi callback chạy.
4. **`weak var delegate: SomeProtocol?`** mà protocol không `: AnyObject` → lỗi compile.
5. **Timer target/selector giữ target mạnh** — `invalidate()` trong `deinit` không bao giờ chạy vì `deinit` không bao giờ được gọi.
6. **Gọi `self?.a(); self?.b()`** — self có thể thành nil giữa hai dòng (trên thread khác). Dùng `guard let self` để có đường chạy nhất quán.
7. **Memory Graph không báo leak ≠ không leak** — object bị giữ bởi singleton/static/cache không giới hạn là "abandoned memory", phải dùng Allocations + Mark Generation hoặc đếm instance.
8. **Retain count không đáng tin**, đừng `print(CFGetRetainCount(obj))` để "chứng minh" trong phỏng vấn.
9. **`lazy var` closure dùng `self`** → cycle (closure được lưu trong self).
10. **Struct không bao giờ leak?** — struct chứa closure capture class vẫn tham gia vào cycle phía class.

---

## 🛠 Bài tập

> Tạo file trong `practice/week05/`. **Dự đoán output trên giấy trước khi chạy.** Mỗi class đều thêm `deinit { print(...) }` để "nhìn thấy" ARC.

**Bài 1 — Dự đoán thứ tự (T2)**
Viết ra giấy output của đoạn sau, rồi chạy để kiểm tra:
```swift
class Node {
    let id: Int
    var next: Node?
    init(_ id: Int) { self.id = id; print("init \(id)") }
    deinit { print("deinit \(id)") }
}
var n1: Node? = Node(1)
var n2: Node? = Node(2)
n1?.next = n2
n2 = nil
print("A")
n1 = nil
print("B")
```
<details><summary>Lời giải</summary>

```
init 1
init 2
A
deinit 1
deinit 2
B
```
`n2 = nil` chưa deinit Node 2 vì `n1.next` vẫn giữ (RC = 1). Khi `n1 = nil`: Node 1 deinit → các property của nó được release → Node 2 về RC 0 → deinit. Thứ tự: **cha deinit trước, rồi mới release con**.
</details>

**Bài 2 — Sửa cycle 2 class (T3)**
Mô hình `Teacher` có `var students: [Student]`, `Student` có `var teacher: Teacher?`. Tạo 1 teacher + 2 student, nối hai chiều, gán tất cả về nil.
a) Chứng minh leak (không có dòng deinit). b) Sửa và giải thích tại sao chọn `weak` chứ không phải `unowned` (gợi ý: student có thể tồn tại khi chưa có giáo viên). c) Vẽ sơ đồ trước/sau khi sửa.

**Bài 3 — Closure nào leak? (T4)**
Cho class dưới, với từng hàm `f1`…`f5`, đoán **LEAK** hay **OK**, rồi chạy kiểm tra:
```swift
import Foundation
final class A {
    var name = "A"
    var handler: (() -> Void)?
    lazy var greeting: () -> String = { "Hi \(self.name)" }
    func f1() { [1, 2].forEach { _ in _ = self.name } }
    func f2() { DispatchQueue.global().async { _ = self.name } }
    func f3() { handler = { _ = self.name } }
    func f4() { let local = { _ = self.name }; local() }
    func f5() { _ = greeting() }
    deinit { print("deinit A") }
}
func check(_ label: String, _ body: (A) -> Void) {
    weak var w: A?
    do { let a = A(); w = a; body(a) }
    Thread.sleep(forTimeInterval: 0.1)
    print(label, w == nil ? "OK" : "LEAK")
}
check("f1", { $0.f1() })
check("f2", { $0.f2() })
check("f3", { $0.f3() })
check("f4", { $0.f4() })
check("f5", { $0.f5() })
```
<details><summary>Lời giải</summary>

```
deinit A
f1 OK        ← forEach: closure non-escaping
deinit A
f2 OK        ← escaping nhưng GCD thả block sau khi chạy: chỉ kéo dài đời sống, không cycle
f3 LEAK      ← self.handler giữ closure, closure giữ self
deinit A
f4 OK        ← closure local, chết khi hàm f4 return
f5 LEAK      ← lazy var lưu closure trong self (sau lần truy cập đầu tiên)
```
Sửa f3: `handler = { [weak self] in _ = self?.name }`. Sửa f5: `lazy var greeting: () -> String = { [unowned self] in "Hi \(self.name)" }` — ở đây `unowned` chấp nhận được vì closure nằm trong chính self, không thể sống lâu hơn self (trừ khi bạn lấy closure ra lưu chỗ khác — khi đó dùng `weak`).
</details>

**Bài 4 — Màn hình đồng hồ (T5)**
Viết class `CountdownScreen` có: một `Timer` repeat mỗi 0,1 giây, một observer `NotificationCenter` (block API) nghe `.didLogin`, một property `onFinish: (() -> Void)?` gọi `self.close()`. Ban đầu viết **sai cả 3 chỗ** → chứng minh không deinit. Sau đó sửa từng chỗ, mỗi lần chạy lại, ghi chú chỗ nào sửa xong thì deinit xuất hiện. (Dùng `RunLoop.main.run(until:)` để cho timer chạy trong script.)

**Bài 5 — Hàm kiểm tra leak (T6)**
Viết `func isDeallocated(_ build: () -> AnyObject) -> Bool` — tạo object bằng `build`, thả mọi tham chiếu strong, trả `true` nếu object đã được giải phóng. Thử với một class leak và một class sạch. (Đây chính là kỹ thuật viết unit test phát hiện leak.)
<details><summary>Lời giải</summary>

```swift
import Foundation

func isDeallocated(_ build: () -> AnyObject) -> Bool {
    weak var weakRef: AnyObject?
    autoreleasepool {
        let object = build()
        weakRef = object
    }                           // object hết scope ở đây
    return weakRef == nil
}

final class Leaky {
    var callback: (() -> Void)?
    init() { callback = { print(self) } }
}
final class Clean {
    var callback: (() -> Void)?
    init() { callback = { [weak self] in print(self as Any) } }
}
print("Leaky:", isDeallocated { Leaky() })   // Leaky: false
print("Clean:", isDeallocated { Clean() })   // Clean: true
```
Trong XCTest, pattern phổ biến: `addTeardownBlock { [weak sut] in XCTAssertNil(sut, "Leak!") }`.
</details>

**Bài 6 — ImageCache bằng NSCache (T7, làm trong side project)**
Viết `ImageCache` (singleton) dùng `NSCache<NSURL, UIImage>`, có `countLimit = 200`, `totalCostLimit ≈ 50 MB`, cost tính theo số byte của ảnh. API: `image(for:)`, `insert(_:for:)`, `removeAll()`. Gọi `removeAll()` trong `didReceiveMemoryWarning`? (Câu hỏi phụ: có cần không, khi NSCache đã tự evict?)
<details><summary>Lời giải</summary>

```swift
import UIKit

final class ImageCache {
    static let shared = ImageCache()
    private let cache = NSCache<NSURL, UIImage>()

    private init() {
        cache.countLimit = 200
        cache.totalCostLimit = 50 * 1024 * 1024   // ~50 MB
    }

    func image(for url: URL) -> UIImage? {
        cache.object(forKey: url as NSURL)
    }

    func insert(_ image: UIImage, for url: URL) {
        let cost = image.cgImage.map { $0.bytesPerRow * $0.height } ?? 1   // byte thực sự đã decode
        cache.setObject(image, forKey: url as NSURL, cost: cost)
    }

    func removeAll() { cache.removeAllObjects() }
}
```
Câu hỏi phụ: không bắt buộc — NSCache tự evict khi hệ thống báo áp lực bộ nhớ. Gọi `removeAll()` khi nhận memory warning là chủ động giải phóng nhanh hơn, vô hại vì cache tạo lại được. Lưu ý cost nên dùng kích thước **sau khi decode** (`bytesPerRow * height`), không phải dung lượng file PNG/JPEG.
</details>

**🏋️ Bài tập lớn (T7) — `LeakLab`: tạo leak, săn leak, sửa leak**
Tạo project iOS mới `LeakLab` (UIKit, storyboard hoặc code đều được):
- Màn hình gốc có 4 nút, mỗi nút push một `DetailViewController` chứa **một loại leak**:
  1. Closure lưu trong property dùng `self` (ví dụ `viewModel.onUpdate = { self.render() }`, VC giữ viewModel)
  2. `Timer.scheduledTimer(withTimeInterval:repeats:)` không weak, cập nhật label mỗi giây
  3. `NotificationCenter.addObserver(forName:...)` block không remove
  4. Delegate khai báo strong (`var delegate: DetailDelegate?`) + closure lồng nhau sai chỗ weak
- Mỗi VC có `deinit { print("deinit \(Self.self)") }`.
- **Nhiệm vụ:** push/pop mỗi màn 3 lần → (a) chụp Memory Graph, ghi lại số instance từng VC và ảnh chụp sơ đồ cycle; (b) chạy Instruments Leaks, chụp màn hình Cycles & Roots; (c) với leak số 3, dùng Allocations + Mark Generation cho thấy bộ nhớ tăng mỗi lần vào/ra; (d) sửa từng leak, chứng minh bằng log `deinit` và Memory Graph sạch.
- Viết `NOTES.md` trong project: mỗi leak 3 dòng — *nguyên nhân · cách phát hiện · cách sửa*. Đây là chất liệu để kể trong phỏng vấn ("Kể về một lần bạn debug memory leak").

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ ARC là gì? Hoạt động như thế nào?**
<details><summary>Đáp án gợi ý</summary>

ARC (Automatic Reference Counting) là cơ chế quản lý bộ nhớ cho reference type. Mỗi object có một bộ đếm strong reference; **compiler tự chèn** lệnh retain khi có tham chiếu strong mới và release khi tham chiếu mất đi. Khi strong count về 0, `deinit` chạy ngay và bộ nhớ được thu hồi. ARC chạy lúc compile-time + runtime (bộ đếm atomic), không phải một tiến trình nền. Nó chỉ áp dụng cho class, closure, actor — không áp dụng trực tiếp cho struct/enum.
</details>

**Q2 ⭐ ARC khác Garbage Collection thế nào? Ưu nhược điểm?**
<details><summary>Đáp án gợi ý</summary>

ARC giải phóng **ngay và xác định** khi RC = 0, chi phí rải đều trong từng lần retain/release, không có pause "stop-the-world", tốn ít RAM hơn — hợp với thiết bị di động. Nhược điểm: **không tự xử lý retain cycle**, lập trình viên phải dùng weak/unowned; retain/release atomic có chi phí. GC (tracing) phát hiện được cycle nhưng thời điểm thu hồi không đoán trước, có thể gây giật và cần dư bộ nhớ.
</details>

**Q3 ⭐ Retain cycle là gì? Cho ví dụ và cách sửa.**
<details><summary>Đáp án gợi ý</summary>

Hai (hoặc nhiều) object giữ nhau bằng strong reference tạo thành vòng → strong count không bao giờ về 0 dù bên ngoài không ai dùng → leak. Ví dụ phổ biến nhất: VC giữ ViewModel, ViewModel giữ closure `onUpdate`, closure capture `self` (VC) mạnh. Sửa: cắt một cạnh của vòng bằng `weak` hoặc `unowned` — thường là cạnh "trỏ ngược về owner" (delegate, parent, closure capture self).
</details>

**Q4 ⭐ `weak` và `unowned` khác nhau thế nào? Khi nào dùng cái nào?**
<details><summary>Đáp án gợi ý</summary>

Cả hai không tăng strong count. `weak` luôn là Optional và tự thành `nil` khi object bị giải phóng (đi qua side table) → an toàn. `unowned` không Optional (thường), giả định object còn sống; nếu truy cập sau khi object đã deinit thì **crash** (trap). Dùng `weak` khi bên kia có thể chết trước (delegate, closure async). Dùng `unowned` khi vòng đời được bảo đảm bằng cấu trúc — bên kia sống lâu hơn hoặc bằng (ví dụ `CreditCard.customer`). Khi phân vân → `weak`.
</details>

**Q5 ⭐ Khi nào cần `[weak self]` trong closure? Khi nào không cần?**
<details><summary>Đáp án gợi ý</summary>

Cần khi closure được **lưu trữ** bởi chính self hoặc bởi thứ self sở hữu (property, cell, timer, notification observer, `lazy var`) → cycle. Cũng nên dùng khi closure async chạy lâu mà không muốn giữ màn hình sống thêm. Không cần với closure non-escaping (`map`, `forEach`), `UIView.animate`, `DispatchQueue.main.async` ngắn — vì hệ thống chỉ giữ closure tạm thời rồi thả, không tạo vòng. Câu hỏi để quyết định: "Ai giữ closure này, chuỗi đó có dẫn về self không?"
</details>

**Q6 `guard let self` trong closure `[weak self]` có tạo lại retain cycle không?**
<details><summary>Đáp án gợi ý</summary>

Không. Nó tạo một strong reference **cục bộ** chỉ sống trong một lần thực thi closure; hết lần chạy thì release. Cycle chỉ xảy ra nếu closure **lưu trữ** strong reference (capture). Lợi ích của `guard let self` là đảm bảo self không biến mất giữa chừng, code chạy nhất quán.
</details>

**Q7 ⭐ Closure lồng nhau: đặt `[weak self]` ở đâu?**
<details><summary>Đáp án gợi ý</summary>

Ở closure **ngoài cùng** có thể bị lưu trữ. Nếu chỉ đặt `[weak self]` ở closure trong, closure ngoài phải capture self (strong) để truyền xuống → nếu closure ngoài được self giữ thì vẫn cycle. Đặt ở ngoài rồi bên trong dùng lại biến `self` (đã weak) là đủ.
</details>

**Q8 ⭐ Kể các nguồn leak hay gặp trong app iOS.**
<details><summary>Đáp án gợi ý</summary>

(1) Closure lưu trong property capture self; (2) delegate khai báo strong; (3) `Timer` repeat — RunLoop giữ timer, timer giữ target/closure; (4) `NotificationCenter` block observer không remove/không weak; (5) closure lồng nhau đặt weak sai chỗ; (6) callback của cell giữ VC; (7) `lazy var` closure dùng self; (8) singleton/static/array toàn cục giữ VC (abandoned memory); (9) Combine `sink` lưu cancellable trong self mà closure dùng self mạnh.
</details>

**Q9 ⭐ Bạn tìm memory leak như thế nào? (Kể từng bước)**
<details><summary>Đáp án gợi ý</summary>

Đầu tiên log `deinit` để phát hiện màn hình đóng mà không giải phóng. Sau đó: bật Malloc Stack Logging, vào/ra màn hình vài lần, mở **Memory Graph Debugger**, lọc theo class của mình, xem số instance và các leak được đánh dấu tím, chọn instance để xem đồ thị tham chiếu — đi theo các đường strong để thấy vòng (thường có closure context, timer hoặc observer), xem backtrace để biết dòng tạo object. Với leak không phải cycle (abandoned), dùng **Instruments Allocations** với Mark Generation để thấy bộ nhớ tăng sau mỗi lần lặp; Instruments **Leaks** để xem Cycles & Roots. Sửa xong thì lặp lại để xác nhận instance count về 0. Có thể thêm unit test dùng weak reference để chống tái phát.
</details>

**Q10 Delegate tại sao thường là `weak`? Protocol cần điều kiện gì?**
<details><summary>Đáp án gợi ý</summary>

Vì owner (VC) thường sở hữu object kia (strong), object đó trỏ ngược về owner qua delegate → nếu strong sẽ thành cycle. `weak` chỉ áp dụng cho class nên protocol phải ràng buộc `AnyObject` (`protocol XDelegate: AnyObject`). Ngoại lệ hiếm: object không được owner giữ (ví dụ `URLSession` giữ delegate strong một cách có chủ đích cho tới khi `invalidate`).
</details>

**Q11 Vì sao Timer hay gây leak? Sửa thế nào?**
<details><summary>Đáp án gợi ý</summary>

RunLoop giữ Timer đã schedule; Timer giữ target (API selector) hoặc closure (API block). Nếu target/closure giữ self mạnh, self không deinit, nên `invalidate()` đặt trong deinit không bao giờ chạy. Sửa: dùng API block với `[weak self]` và invalidate trong deinit; hoặc invalidate ở thời điểm vòng đời rõ ràng (`viewWillDisappear`); hoặc dùng proxy weak target cho API selector.
</details>

**Q12 weak reference tự thành `nil` như thế nào? Side table là gì?**
<details><summary>Đáp án gợi ý</summary>

Khi một object có weak reference đầu tiên, Swift runtime tạo **side table** — một khối riêng chứa strong/unowned/weak count và con trỏ tới object. Biến weak trỏ vào side table chứ không trỏ thẳng object. Khi object deinit, đọc qua side table thấy object đã chết → trả `nil`. Object được free khi strong và unowned count về 0; side table được giữ tới khi weak count về 0. Nhờ vậy weak không bao giờ trỏ vào vùng nhớ rác.
</details>

**Q13 Truy cập `unowned` sau khi object đã giải phóng thì sao? `unowned(unsafe)` khác gì?**
<details><summary>Đáp án gợi ý</summary>

`unowned` (safe): object đã deinit nhưng bộ nhớ chưa free (unowned count giữ lại), runtime kiểm tra và **crash có chủ đích** với "Attempted to read an unowned reference but object ... was already destroyed". `unowned(unsafe)` không kiểm tra, giống con trỏ thô — đọc vùng nhớ đã free, undefined behavior. Chỉ dùng khi cần tối ưu cực đoan và chắc chắn vòng đời.
</details>

**Q14 `autoreleasepool` dùng khi nào trong Swift?**
<details><summary>Đáp án gợi ý</summary>

Khi vòng lặp dài tạo nhiều object Objective-C autoreleased (ảnh, NSString, Data bridging...) — pool mặc định chỉ xả cuối vòng RunLoop nên memory peak tăng vọt. Bọc thân vòng lặp trong `autoreleasepool {}` để xả mỗi vòng. Code Swift thuần không cần.
</details>

**Q15 ⭐ NSCache khác Dictionary thế nào? Khi nào dùng NSCache?**
<details><summary>Đáp án gợi ý</summary>

NSCache thread-safe, tự evict khi hệ thống thiếu bộ nhớ, có `countLimit`/`totalCostLimit` (giới hạn mềm), không copy key, key/value phải là object. Dictionary không thread-safe, không tự dọn, phải tự nghe memory warning. Dùng NSCache cho dữ liệu **tạo lại được** (ảnh đã decode, kết quả tính toán) — không dùng cho dữ liệu bắt buộc phải còn.
</details>

**Q16 Struct có bị retain cycle không? Struct có refcount không?**
<details><summary>Đáp án gợi ý</summary>

Struct là value type, không có refcount riêng và không thể tạo vòng tham chiếu với chính nó. Nhưng struct chứa property kiểu class / closure / Array (buffer heap) thì copy struct sẽ retain các tham chiếu đó; closure lưu trong struct vẫn có thể capture class và góp phần tạo cycle ở phía class. Struct có thể nằm trên heap khi là property của class, bị capture bởi escaping closure, hoặc nằm trong existential lớn.
</details>

**Q17 Memory warning là gì? App nên làm gì? Jetsam là gì?**
<details><summary>Đáp án gợi ý</summary>

Khi RAM hệ thống căng, iOS gửi memory warning (`didReceiveMemoryWarning`, `UIApplication.didReceiveMemoryWarningNotification`). App nên giải phóng cache, ảnh, dữ liệu tạo lại được. Nếu vẫn vượt ngưỡng, **Jetsam** (cơ chế của kernel) kill app — không có crash log thông thường. Vì vậy giữ memory footprint thấp, đặc biệt khi decode ảnh lớn (downsample), và theo dõi bằng Xcode Memory Gauge / Instruments.
</details>

**Q18 `deinit` được gọi khi nào? Có gì cần lưu ý?**
<details><summary>Đáp án gợi ý</summary>

Ngay khi strong count về 0, trên thread thực hiện release cuối cùng (không nhất thiết main thread!). Chỉ class/actor có deinit. Trong deinit nên dọn tài nguyên: invalidate timer, remove observer, cancel task. Không nên thoát `self` ra ngoài (lưu self vào chỗ khác) trong deinit. Cha deinit trước, sau đó các property của nó mới được release. Swift không hứa object sống đến hết scope — có thể deinit ngay sau lần dùng cuối; cần kéo dài thì dùng `withExtendedLifetime`.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–6 không xem lời giải trước
- [ ] Làm xong `LeakLab`: 4 leak được phát hiện bằng công cụ và sửa, có `NOTES.md`
- [ ] Trả lời được ≥ 14/18 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "weak vs unowned, khi nào dùng cái nào, chuyện gì xảy ra ở runtime"
- [ ] Ghi âm 2 phút: "Kể lại cách bạn tìm một memory leak bằng Memory Graph và Instruments"
- [ ] Vẽ lại từ trí nhớ 3 sơ đồ: Owner↔Pet, closure↔self, RunLoop→Timer→closure→self
- [ ] Làm xong 6 bài DSA tuần 5 (Stack)
- [ ] Side project Milestone 5 xong (Memory Graph sạch, có `ImageCache` dùng `NSCache`)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/) — **đọc hết**, đặc biệt phần Strong Reference Cycles for Closures
- [The Swift Programming Language — Closures (Capturing Values, Escaping Closures)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)
- Apple Developer Documentation — [NSCache](https://developer.apple.com/documentation/foundation/nscache), [Gathering information about memory use](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use)
- WWDC 2021 — *ARC in Swift: Basics and beyond* (vòng đời object, `withExtendedLifetime`, vì sao không nên dựa vào thời điểm deinit)
- WWDC 2021 — *Detect and diagnose memory issues* (Memory Graph, Leaks, Allocations)
- WWDC 2018 — *iOS Memory Deep Dive* (memory footprint, Jetsam, downsample ảnh)
- WWDC 2016 — *Understanding Swift Performance* (stack vs heap, refcount, chi phí struct chứa reference)
- Mike Ash — *Friday Q&A 2017-09-22: Swift 4 Weak References* (side table giải thích chi tiết)
- Swift repo — [`RefCount.h`](https://github.com/swiftlang/swift/blob/main/stdlib/public/SwiftShims/swift/shims/RefCount.h) (comment đầu file mô tả vòng đời LIVE → DEINITED → FREED → DEAD)
