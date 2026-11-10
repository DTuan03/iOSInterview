# Tuần 3 — Closure · Higher-order function · Protocol · Extension · POP (23/11 – 29/11/2026)

> **Giai đoạn 1 · Swift Core** · Thời lượng: ~14 giờ
> Closure và protocol là **hai thứ bạn dùng mỗi ngày** trong iOS (completion handler, delegate, `map`/`filter`, mock để test) nhưng hay dùng theo phản xạ mà không hiểu. Phỏng vấn sẽ hỏi: closure capture biến thế nào, `@escaping` để làm gì, vì sao method trong protocol extension "không override được".

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Viết closure từ dạng đầy đủ đến dạng rút gọn nhất, dùng trailing closure (kể cả nhiều trailing closure)
- [ ] Giải thích closure **capture biến theo tham chiếu**, capture list `[x]` thay đổi điều đó thế nào
- [ ] Giải thích `@escaping` vs non-escaping, vì sao mặc định là non-escaping, và khi nào phải viết `self.`
- [ ] Biết `@autoclosure` dùng để làm gì (lazy evaluation)
- [ ] Dùng thành thạo `map`, `filter`, `reduce`, `reduce(into:)`, `compactMap`, `flatMap`, `sorted`, `lazy`; phân biệt `forEach` với `for-in`
- [ ] Định nghĩa protocol với đủ loại requirement, default implementation, protocol composition, class-only protocol
- [ ] Giải thích **Protocol-Oriented Programming** khác OOP thế nào
- [ ] Giải thích bẫy **static dispatch của method chỉ khai báo trong protocol extension**
- [ ] Cài delegate pattern đúng chuẩn (`AnyObject` + `weak`)
- [ ] Làm xong 6 bài DSA tuần 3 — Two Pointers (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 23/11** | 2h | §1 Cú pháp closure + §2 **Capture** (đọc chậm). Bài tập 1, 2 · DSA: Valid Palindrome | `day1.swift` |
| **T3 24/11** | 2h | §3 **`@escaping`** + §4 `@autoclosure`. Bài tập 3 · DSA: Two Sum II – Input Array Is Sorted | `day2.swift` |
| **T4 25/11** | 2h | §5 **Higher-order function**. Bài tập 4 · DSA: 3Sum | `day3.swift` |
| **T5 26/11** | 2h | §6 **Protocol** + §7 Extension. Bài tập 5 · DSA: Container With Most Water | `day4.swift` |
| **T6 27/11** | 1,5h | §8 POP vs OOP + §9 **Bẫy dispatch của protocol extension** + §10 Delegate. Bài tập 6, 7 · DSA: Move Zeroes | `day5.swift` |
| **T7 28/11** | 3,5h | Bài tập lớn (Form validator). Side project: Milestone 3 (`RepositoryServiceProtocol` + `MockRepositoryService`, extension). DSA (tuỳ sức): Trapping Rain Water | `validator.swift`, commit side project |
| **CN 29/11** | 2h | Tự trả lời 19 câu phỏng vấn ra giấy, rồi mới mở đáp án. Ghi âm 2 câu. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. Closure là gì & cú pháp

**Closure** = một khối code có thể truyền đi, lưu lại, gọi sau, và **"nhớ" các biến xung quanh nơi nó được tạo**. Hàm (`func`) thực chất là một closure có tên. Closure có kiểu như mọi giá trị khác: `(Int) -> Int`, `() -> Void`, `(String, Error?) -> Void`.

Từ dạng đầy đủ đến dạng gọn nhất:
```swift
let names = ["Chi", "An", "Bình"]

let s1 = names.sorted(by: { (a: String, b: String) -> Bool in return a < b })  // đầy đủ
let s2 = names.sorted(by: { a, b in a < b })    // suy luận kiểu + bỏ return (1 biểu thức)
let s3 = names.sorted { $0 < $1 }               // shorthand argument + trailing closure
let s4 = names.sorted(by: <)                    // truyền thẳng toán tử (cũng là một hàm)
print(s1 == s2, s2 == s3, s3 == s4, s4)         // true true true ["An", "Bình", "Chi"]

let square: (Int) -> Int = { x in x * x }
print(square(5))   // 25
```

**Trailing closure**: closure là tham số cuối thì viết ra ngoài dấu ngoặc. **Nhiều trailing closure** (Swift 5.3+): closure đầu bỏ label, các closure sau giữ label:
```swift
func fetch(id: Int, onSuccess: (String) -> Void, onFailure: (String) -> Void) {
    if id > 0 { onSuccess("User \(id)") } else { onFailure("ID không hợp lệ") }
}
fetch(id: 1) { user in
    print("OK:", user)            // in: OK: User 1
} onFailure: { err in
    print("Lỗi:", err)
}
```
SwiftUI dùng cú pháp này khắp nơi: `Button { action } label: { Text("OK") }`.

> Quy ước: dùng `$0` khi closure ngắn, 1 dòng. Closure dài hoặc lồng nhau thì **đặt tên tham số** cho dễ đọc.

### §2. Capture — closure "nhớ" biến như thế nào

```swift
func makeCounter() -> () -> Int {
    var count = 0               // biến local của makeCounter
    return {
        count += 1              // closure capture `count`
        return count
    }
}
let c1 = makeCounter()
let c2 = makeCounter()
print(c1(), c1(), c1(), c2())   // 1 2 3 1
```
`makeCounter` đã return, vậy `count` sống ở đâu? Vì closure escape ra ngoài, compiler đưa `count` vào một **box trên heap**; closure giữ tham chiếu tới box đó. Mỗi lần gọi `makeCounter()` tạo box mới → `c1`, `c2` độc lập.

**Quy tắc 1: closure capture *biến* (theo tham chiếu), không phải giá trị tại thời điểm tạo.**
```swift
var x = 10
let byRef = { print("byRef:", x) }
let byValue = { [x] in print("byValue:", x) }   // capture list: chụp GIÁ TRỊ lúc tạo closure
x = 20
byRef()     // byRef: 20
byValue()   // byValue: 10
```

**Quy tắc 2: closure là reference type.** Gán closure cho biến khác thì cả hai cùng trỏ tới một closure (cùng box):
```swift
let counterCopy = c1
print(counterCopy())   // 4 — tiếp tục từ c1, không phải từ 1
```

**Quy tắc 3: capture list với class** chụp *tham chiếu* tại thời điểm tạo (vẫn là strong, trừ khi viết `weak`/`unowned`):
```swift
final class Counter { var n = 0 }
var counter = Counter()
let g = { [counter] in counter.n += 1; print("g:", counter.n) }   // chụp object hiện tại
counter = Counter()        // biến counter trỏ sang object mới
g(); g()                   // g: 1, g: 2 — vẫn sửa object CŨ
print(counter.n)           // 0
```

**`[weak self]`** — vì closure giữ **strong reference** tới mọi thứ nó capture, nếu `self` giữ closure và closure giữ `self` → **retain cycle** (rò rỉ bộ nhớ). Tuần 5 học kỹ; tuần này chỉ cần nhớ mẫu:
```swift
service.load { [weak self] result in
    guard let self else { return }    // Swift 5.8+: sau đó dùng title thay vì self.title cũng được
    self.title = result
}
```

**Vòng `for` không có bẫy như JavaScript:** mỗi vòng lặp là một hằng `i` mới:
```swift
var handlers: [() -> Void] = []
for i in 1...3 { handlers.append { print(i, terminator: " ") } }
handlers.forEach { $0() }   // 1 2 3
```

### §3. `@escaping` vs non-escaping

- **Non-escaping** (mặc định từ Swift 3): closure **chắc chắn được gọi xong trước khi hàm return**, không được lưu lại. Ví dụ: `map`, `filter`, `sorted`.
- **Escaping** (`@escaping`): closure có thể **sống lâu hơn hàm** — được lưu vào property, chạy bất đồng bộ (`DispatchQueue.main.async`, completion handler của network).

```swift
final class Scheduler {
    private var pending: [() -> Void] = []
    func later(_ handler: @escaping () -> Void) { pending.append(handler) }  // lưu lại → escaping
    func now(_ handler: () -> Void) { handler() }                           // gọi xong trong hàm
    func runAll() { pending.forEach { $0() }; pending.removeAll() }
}

final class ViewModel {
    let scheduler = Scheduler()
    var title = "A"
    func load() {
        scheduler.now { title = "B" }                       // non-escaping: self ngầm định OK
        scheduler.later { [weak self] in self?.title = "C" }
        scheduler.later { self.title = "D" }                // escaping: phải viết self. tường minh
    }
}
let vm = ViewModel()
vm.load()
print(vm.title)          // B
vm.scheduler.runAll()
print(vm.title)          // D (closure "C" chạy trước, "D" chạy sau)
```

Compiler kiểm tra chặt:
```swift
// Lưu closure non-escaping:
//   func later(_ handler: () -> Void) { pending.append(handler) }
//   ❌ error: converting non-escaping parameter 'handler' to generic parameter 'Element' may allow it to escape
// Bỏ self. trong escaping closure của class:
//   scheduler.later { title = "D" }
//   ❌ error: reference to property 'title' in closure requires explicit use of 'self' to make capture semantics explicit
// Escaping closure capture self trong mutating func của struct:
//   ❌ error: escaping closure captures mutating 'self' parameter
```

**Tại sao mặc định là non-escaping?**
1. **Hiệu năng**: compiler biết closure không sống lâu → context có thể cấp phát trên **stack**, không cần retain/release biến capture.
2. **An toàn**: non-escaping không thể gây retain cycle → không bắt bạn viết `self.`.
3. **Rõ ràng**: thấy `@escaping` trong signature là biết "cẩn thận memory".

**Bẫy:** closure **Optional** `(() -> Void)?` **luôn là escaping** ngầm định (vì nó được bọc trong enum Optional), không cần và không được viết `@escaping`.

**Vì sao chỉ class bắt viết `self.`?** Viết `self.` là để nhắc bạn "đang capture strong một reference → có thể retain cycle". Struct (value type) không gây retain cycle, nên từ Swift 5.3 được dùng `self` ngầm định trong escaping closure.

### §4. `@autoclosure`

`@autoclosure` tự bọc **một biểu thức** thành closure `() -> T`. Người gọi viết như truyền giá trị bình thường, nhưng biểu thức **chỉ được tính khi hàm thật sự gọi nó** (lazy evaluation).
```swift
func logDebug(_ message: @autoclosure () -> String, enabled: Bool) {
    if enabled { print(message()) }
}
func expensive() -> String { print("(tính toán tốn kém)"); return "chi tiết" }

logDebug(expensive(), enabled: false)   // không in gì — expensive() KHÔNG chạy
logDebug(expensive(), enabled: true)    // (tính toán tốn kém) / chi tiết

func myOr(_ lhs: Bool, _ rhs: @autoclosure () -> Bool) -> Bool { lhs ? true : rhs() }
print(myOr(true, expensive().isEmpty))  // true — expensive() không chạy (short-circuit)
```
Trong thư viện chuẩn: `assert(_:_:)`, `precondition`, toán tử `&&`, `||`, `??` đều dùng `@autoclosure`. Ví dụ `name ?? computeDefault()` chỉ gọi `computeDefault()` khi `name == nil`.
Đừng lạm dụng: người đọc code không thấy dấu hiệu nào cho biết biểu thức bị trì hoãn.

### §5. Higher-order function

Higher-order function = hàm **nhận** hàm khác làm tham số hoặc **trả về** hàm.

```swift
struct Repo { let name: String; let stars: Int; let language: String? }
let repos = [
    Repo(name: "Alamofire",   stars: 41_000, language: "Swift"),
    Repo(name: "SDWebImage",  stars: 25_000, language: "Objective-C"),
    Repo(name: "awesome-ios", stars: 50_000, language: nil),
    Repo(name: "Kingfisher",  stars: 23_000, language: "Swift"),
]

let names = repos.map { $0.name }
let names2 = repos.map(\.name)                  // key path dùng như hàm (Swift 5.2+)
print(names == names2)                          // true

print(repos.filter { $0.language == "Swift" }.map(\.name))   // ["Alamofire", "Kingfisher"]
print(repos.reduce(0) { $0 + $1.stars })                    // 139000
print(repos.compactMap(\.language))             // ["Swift", "Objective-C", "Swift"] — bỏ nil

let countByLang = repos.reduce(into: [String: Int]()) { result, repo in
    result[repo.language ?? "Unknown", default: 0] += 1
}
// ["Objective-C": 1, "Swift": 2, "Unknown": 1]

let top2 = repos.sorted { $0.stars > $1.stars }.prefix(2).map(\.name)
print(top2)                                     // ["awesome-ios", "Alamofire"]
```

**`map` vs `compactMap` vs `flatMap`:**
```swift
let words = ["1", "hai", "3"]
print(words.map { Int($0) })          // [Optional(1), nil, Optional(3)] — [Int?]
print(words.compactMap { Int($0) })   // [1, 3]                           — bỏ nil, [Int]
let nested = [[1, 2], [3], [], [4, 5]]
print(nested.flatMap { $0 })          // [1, 2, 3, 4, 5]                  — làm phẳng 1 cấp
```
(`flatMap` dùng để bỏ nil là cách cũ, đã deprecated → dùng `compactMap`.)

**`reduce` vs `reduce(into:)`:** `reduce(_:_:)` tạo giá trị mới mỗi bước — với Array/Dictionary làm accumulator thì mỗi bước **copy** → O(n²). `reduce(into:)` truyền accumulator dạng `inout` → sửa tại chỗ, O(n). Khi gom vào collection, luôn dùng `reduce(into:)`.

**Các hàm khác hay dùng:**
```swift
print(repos.first { $0.stars > 30_000 }?.name ?? "-")   // Alamofire  — dừng ở phần tử đầu tiên khớp
print(repos.contains { $0.language == nil })            // true
print(repos.allSatisfy { $0.stars > 20_000 })           // true
print(repos.min { $0.stars < $1.stars }!.name)          // Kingfisher
```
`sorted()` trả mảng mới, `sort()` sắp xếp tại chỗ (`mutating`). Cả hai O(n log n) và **stable** (tài liệu stdlib hiện tại đảm bảo: các phần tử "bằng nhau" giữ nguyên thứ tự ban đầu — các bản Swift rất cũ thì không đảm bảo điều này).

**`forEach` vs `for-in`:**
```swift
for n in [1, 2, 3, 4] {
    if n == 3 { break }          // dừng hẳn vòng lặp
    print("for", n)              // for 1, for 2
}
[1, 2, 3, 4].forEach { n in
    if n == 3 { return }         // chỉ thoát closure của phần tử này, KHÔNG dừng
    print("forEach", n)          // forEach 1, forEach 2, forEach 4
}
```
`forEach` không có `break`/`continue`, `return` chỉ như `continue`. Dùng `for-in` khi cần thoát sớm; `forEach` hợp cuối một chuỗi `map/filter`.

**`lazy`** — trì hoãn tính toán, không tạo mảng trung gian:
```swift
let result = (1...1_000_000).lazy
    .map { $0 * $0 }
    .filter { $0 % 7 == 0 }
    .prefix(3)
print(Array(result))     // [49, 196, 441] — chỉ tính đến khi đủ 3 phần tử
```
Không có `lazy`, `map` sẽ tạo mảng 1 triệu phần tử rồi `filter` tạo mảng khác. **Nhưng** lazy **không cache**: mỗi lần truy cập lại tính lại closure:
```swift
var mapCalls = 0
let lazyMapped = [1, 2, 3].lazy.map { x -> Int in mapCalls += 1; return x * 10 }
_ = lazyMapped[0]; _ = lazyMapped[0]
print(mapCalls)          // 2
```

### §6. Protocol

Protocol = **bản hợp đồng**: liệt kê những gì một kiểu phải có, không quy định cách làm. Struct, class, enum, actor đều conform được.

#### 6.1. Các loại requirement
```swift
protocol Describable {
    var title: String { get }               // ít nhất đọc được (let, var, computed đều OK)
    var subtitle: String { get set }        // phải đọc + ghi được (let hay computed chỉ get → không OK)
    func describe() -> String
    mutating func reset()                   // cho phép struct/enum sửa self
    init(title: String)
    static var kind: String { get }
}

struct Article: Describable {
    let title: String
    var subtitle = ""
    func describe() -> String { "\(title) — \(subtitle)" }
    mutating func reset() { subtitle = "" }
    init(title: String) { self.title = title }
    static let kind = "article"
}

final class Video: Describable {
    var title: String
    var subtitle = ""
    func describe() -> String { "▶︎ \(title)" }
    func reset() { subtitle = "" }                 // class: không cần mutating
    init(title: String) { self.title = title }     // final class → không cần `required`
    static var kind: String { "video" }
}

let items: [any Describable] = [Article(title: "Swift"), Video(title: "WWDC")]
for i in items { print(i.describe(), type(of: i).kind) }
// Swift —  article
// ▶︎ WWDC video
```
- Nếu `Video` không `final`, init phải là `required` (*error: initializer requirement 'init(title:)' can only be satisfied by a 'required' initializer in non-final class 'Video'*) — vì lớp con cũng phải thoả protocol.
- `any Describable` là **existential type** — "một hộp chứa bất kỳ kiểu nào conform". Tuần 4 sẽ so sánh với `some`.

#### 6.2. Default implementation qua extension
```swift
protocol Greeter {
    var name: String { get }
    func greet() -> String
}
extension Greeter {
    func greet() -> String { "Xin chào, \(name)" }      // mặc định cho requirement
    func shout() -> String { greet().uppercased() }      // method bổ sung (không phải requirement)
}
struct Vietnamese: Greeter { let name: String }           // dùng mặc định
struct English: Greeter {
    let name: String
    func greet() -> String { "Hello, \(name)" }           // tự cài, thay mặc định
}
print(Vietnamese(name: "An").greet())   // Xin chào, An
print(English(name: "Bob").shout())     // HELLO, BOB
```
Đây là cách Swift làm **"optional requirement"**: có default trong extension thì kiểu conform không bắt buộc cài. (`@objc optional` của Objective-C chỉ dùng được cho protocol `@objc`, gặp ở `UITableViewDelegate`.)

**Constrained extension** — default chỉ áp dụng khi thoả điều kiện:
```swift
extension Array where Element: Numeric {
    func sum() -> Element { reduce(0, +) }
}
print([1, 2, 3].sum(), [1.5, 2.5].sum())   // 6 4.0
// ["a"].sum()  // ❌ error: referencing instance method 'sum()' on 'Array' requires that 'String' conform to 'Numeric'
```

#### 6.3. Kế thừa & kết hợp (composition)
```swift
protocol Identifiable2 { var id: Int { get } }   // đặt tên 2 để tránh trùng Identifiable của stdlib
protocol Named { var name: String { get } }
protocol Entity: Identifiable2, Named {}           // protocol kế thừa NHIỀU protocol
struct Member: Entity { let id: Int; let name: String }

func show(_ x: some Identifiable2 & Named) { print(x.id, x.name) }   // composition bằng &
show(Member(id: 1, name: "An"))     // 1 An
typealias Profile = Identifiable2 & Named
```
Thư viện chuẩn: `typealias Codable = Decodable & Encodable`.

#### 6.4. Class-only protocol
```swift
protocol Listener: AnyObject { func onEvent() }
// struct S: Listener {}   // ❌ error: non-class type 'S' cannot conform to class protocol 'Listener'
```
Cần khi muốn giữ tham chiếu **`weak`** tới kiểu conform (`weak` chỉ áp dụng cho class) — chính là delegate (§10).

#### 6.5. Equatable, Hashable, Comparable, CustomStringConvertible
```swift
struct Version: Hashable, Comparable, CustomStringConvertible {
    let major: Int, minor: Int, patch: Int
    // Equatable, Hashable: compiler TỰ SINH vì mọi property đều Hashable
    static func < (lhs: Version, rhs: Version) -> Bool {      // Comparable: chỉ cần viết <
        (lhs.major, lhs.minor, lhs.patch) < (rhs.major, rhs.minor, rhs.patch)   // so sánh tuple
    }
    var description: String { "\(major).\(minor).\(patch)" }   // dùng khi print / "\(v)"
}
let vs = [Version(major: 1, minor: 10, patch: 0),
          Version(major: 1, minor: 2, patch: 3),
          Version(major: 0, minor: 9, patch: 9)]
print(vs.sorted())          // [0.9.9, 1.2.3, 1.10.0]  — có Comparable thì dùng được sorted(), max()
print(vs.max()!)            // 1.10.0
print(Set(vs + vs).count)   // 3
```
| Protocol | Cần cài | Tự sinh (synthesize) khi |
|---|---|---|
| `Equatable` | `==` | struct: mọi stored property Equatable; enum: mọi associated value Equatable (enum không có associated value thì luôn có) |
| `Hashable` | `hash(into:)` (+ `==`) | như trên nhưng Hashable |
| `Comparable` | `<` (các toán tử còn lại có sẵn) | chỉ enum **không có raw value**, associated value nếu có cũng Comparable (Swift 5.3+) |
| `CustomStringConvertible` | `var description` | không |

```swift
enum Size: Comparable { case small, medium, large }
print(Size.small < Size.large)   // true — so theo thứ tự khai báo case
```

**Tự viết `==`/`hash` theo identity** — hai User cùng `id` coi là một:
```swift
struct User: Hashable {
    let id: Int
    var name: String
    static func == (l: User, r: User) -> Bool { l.id == r.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }   // PHẢI dùng cùng tập field với ==
}
print(Set([User(id: 1, name: "An"), User(id: 1, name: "An (đổi tên)")]).count)   // 1
```
Struct có property là closure (không Equatable) thì không tự sinh được: *error: type 'S' does not conform to protocol 'Equatable'*.

### §7. Extension

Extension **thêm** chức năng cho kiểu có sẵn (kể cả kiểu không có source như `Int`, `String`, `UIView`):
```swift
import Foundation

extension Int {
    var isEven: Bool { self % 2 == 0 }                                    // computed property
    func times(_ action: () -> Void) { for _ in 0..<self { action() } }   // method
}
print(4.isEven)   // true
3.times { print("hi", terminator: " ") }   // hi hi hi

extension String {
    var trimmed: String { trimmingCharacters(in: .whitespacesAndNewlines) }   // cần import Foundation
}

extension Collection {
    subscript(safe index: Index) -> Element? { indices.contains(index) ? self[index] : nil }   // subscript
}
let arr = [10, 20]
print(arr[safe: 1] as Any, arr[safe: 5] as Any)   // Optional(20) nil

struct Point { var x = 0.0, y = 0.0 }
extension Point: CustomStringConvertible {                 // thêm conformance
    var description: String { "(\(x), \(y))" }
}
extension Point {
    init(both v: Double) { self.init(x: v, y: v) }         // init (memberwise vẫn còn — tuần 2)
    enum Quadrant { case first, other }                    // nested type
    var quadrant: Quadrant { x > 0 && y > 0 ? .first : .other }
}
print(Point(both: 2), Point(x: 1, y: -1).quadrant)        // (2.0, 2.0) other
```
**Extension KHÔNG được:**
- Thêm **stored property** (*error: extensions must not contain stored properties*) — vì sẽ thay đổi kích thước/layout bộ nhớ của kiểu.
- Override method có sẵn (với class thuần Swift).
- Thêm designated init cho class (chỉ convenience).

**Cách dùng hay trong dự án**: tách mỗi conformance ra một extension (`extension ViewController: UITableViewDataSource { ... }`) để file dễ đọc; dùng `// MARK: -`.

### §8. Protocol-Oriented Programming (POP) vs OOP

WWDC 2015 *"Protocol-Oriented Programming in Swift"* — "Swift is a protocol-oriented language".

**Vấn đề của kế thừa class (OOP):**
- Chỉ **đơn kế thừa**: `Duck` vừa bay vừa bơi; `Penguin` bơi không bay; `Drone` bay nhưng không phải con vật → cây kế thừa gượng ép, lớp cha phình to ("God class").
- Lớp con **thừa hưởng cả những gì không cần**, phải override để "tắt".
- Chỉ dùng được cho class → mất value semantics, dính ARC, shared mutable state.

**POP:** chia hành vi thành các protocol nhỏ + default implementation, kiểu nào cần gì thì "ghép" cái đó:
```swift
protocol Flyable { var maxAltitude: Int { get } }
extension Flyable { func fly() -> String { "Bay ở độ cao \(maxAltitude)m" } }
protocol Swimmable {}
extension Swimmable { func swim() -> String { "Bơi" } }

struct Duck: Flyable, Swimmable { let maxAltitude = 100 }
struct Penguin: Swimmable {}
struct Drone: Flyable { let maxAltitude = 500 }

print(Duck().fly(), Duck().swim(), Penguin().swim(), Drone().fly())
// Bay ở độ cao 100m Bơi Bơi Bay ở độ cao 500m
```

**Lợi ích lớn nhất trong thực tế — testability:** code phụ thuộc vào protocol thay vì kiểu cụ thể → dễ thay bằng mock khi test.
```swift
protocol Clock { func now() -> Int }
struct SystemClock: Clock { func now() -> Int { 1_700_000_000 } }   // bản thật (giả lập)
struct FixedClock: Clock { let value: Int; func now() -> Int { value } }   // bản cho test

struct Greeter {
    let clock: any Clock                         // phụ thuộc vào abstraction
    func greeting() -> String { clock.now() % 86_400 < 43_200 ? "Chào buổi sáng" : "Chào buổi chiều" }
}
print(Greeter(clock: FixedClock(value: 3_600)).greeting())    // Chào buổi sáng
print(Greeter(clock: FixedClock(value: 50_000)).greeting())   // Chào buổi chiều
```
Side project tuần này làm đúng việc này: `RepositoryServiceProtocol` + `MockRepositoryService`.

| | OOP (class inheritance) | POP |
|---|---|---|
| Tái sử dụng | Kế thừa từ 1 lớp cha | Conform nhiều protocol, default impl |
| Áp dụng cho | Chỉ class | struct, enum, class, actor |
| Ngữ nghĩa | Reference | Tuỳ kiểu (thường value) |
| Kết nối | Chặt (lớp con biết chi tiết lớp cha) | Lỏng (chỉ biết hợp đồng) |
| Khi nào vẫn dùng OOP | Framework yêu cầu (`UIViewController`), cần chia sẻ stored state + override | |

> Câu trả lời "trưởng thành": POP **không thay thế hoàn toàn** OOP. iOS vẫn dựa trên UIKit (class). Dùng POP cho model, service, abstraction để test; dùng class khi cần identity hoặc framework bắt buộc.

### §9. Bẫy kinh điển: dispatch của method trong protocol extension

```swift
protocol Greeting {
    func hello() -> String              // REQUIREMENT
}
extension Greeting {
    func hello() -> String { "Hello từ extension" }
    func bye() -> String { "Bye từ extension" }   // CHỈ có trong extension, KHÔNG phải requirement
}
struct Robot: Greeting {
    func hello() -> String { "Hello từ Robot" }
    func bye() -> String { "Bye từ Robot" }
}

let r1 = Robot()
print(r1.hello(), "|", r1.bye())     // Hello từ Robot | Bye từ Robot
let r2: any Greeting = Robot()
print(r2.hello(), "|", r2.bye())     // Hello từ Robot | Bye từ extension   ← !!!
func greet<T: Greeting>(_ g: T) -> String { g.bye() }
print(greet(Robot()))                // Bye từ extension
```
**Giải thích:**
- Mỗi kiểu conform có một **protocol witness table (PWT)** — bảng ghi "requirement X của protocol được cài bởi hàm nào của kiểu này". `hello()` là requirement → có trong bảng → gọi qua kiểu protocol vẫn tra bảng → ra bản của `Robot` (**dynamic dispatch**).
- `bye()` **không** phải requirement → không có trong bảng. Khi biến có kiểu tĩnh là `any Greeting` (hoặc generic `T: Greeting`), compiler chỉ biết bản trong extension → gọi thẳng (**static dispatch**). `Robot.bye()` chỉ là một hàm trùng tên, **không phải override**.
- Khi kiểu tĩnh là `Robot` thì compiler thấy `Robot.bye()` → gọi nó.

**Cách sửa:** khai báo `bye()` trong thân protocol (biến thành requirement).

**Biến thể với class** — lớp cha conform nhờ default, lớp con "cài lại":
```swift
protocol Sound { func make() -> String }
extension Sound { func make() -> String { "..." } }
class Animal: Sound {}                                  // dùng default, witness table trỏ vào bản extension
class Dog: Animal { func make() -> String { "Gâu" } }   // không có gì để override → chỉ là method mới
let d: Sound = Dog()
print(d.make())        // ...
print(Dog().make())    // Gâu
```
Witness table được lập cho `Animal` (dùng bản extension); `Dog` kế thừa conformance đó. Sửa: `Animal` tự cài `func make()` trong thân class, rồi `Dog` dùng `override`.

### §10. Delegate pattern

Delegate = một object **uỷ quyền** cho object khác xử lý sự kiện / cung cấp dữ liệu, qua một protocol. Dùng khắp UIKit: `UITableViewDelegate`, `UITextFieldDelegate`, `URLSessionDelegate`.

```swift
protocol DownloaderDelegate: AnyObject {                 // 1. AnyObject để dùng được weak
    func downloader(_ d: Downloader, didUpdate progress: Double)
    func downloaderDidFinish(_ d: Downloader)
}
extension DownloaderDelegate {
    func downloader(_ d: Downloader, didUpdate progress: Double) {}   // "optional" kiểu Swift
}

final class Downloader {
    weak var delegate: DownloaderDelegate?              // 2. WEAK để tránh retain cycle
    func start() {
        for p in [0.5, 1.0] { delegate?.downloader(self, didUpdate: p) }   // 3. truyền self làm tham số đầu
        delegate?.downloaderDidFinish(self)
    }
    deinit { print("Downloader deinit") }
}

final class Screen: DownloaderDelegate {
    let downloader = Downloader()                        // Screen giữ STRONG downloader
    init() { downloader.delegate = self }                // downloader giữ WEAK screen
    func downloader(_ d: Downloader, didUpdate progress: Double) { print("Tiến độ \(Int(progress * 100))%") }
    func downloaderDidFinish(_ d: Downloader) { print("Xong") }
    deinit { print("Screen deinit") }
}
do {
    let screen = Screen()
    screen.downloader.start()
}
print("hết scope")
// Tiến độ 50% / Tiến độ 100% / Xong / Screen deinit / Downloader deinit / hết scope
```
Nếu `delegate` là `strong`: Screen → Downloader → Screen → **retain cycle**, không dòng `deinit` nào được in.

**Quy ước đặt tên** (theo Cocoa): method nhận object gửi làm tham số đầu (`tableView(_:didSelectRowAt:)`) để một delegate xử lý được nhiều nguồn; dùng `did`/`will`/`should`.

**Delegate vs closure callback:**
| Delegate | Closure |
|---|---|
| Nhiều callback liên quan gom vào 1 protocol | 1–2 callback đơn giản |
| Quan hệ 1-1, lâu dài | Một lần (completion) hoặc ngắn |
| `weak` → khó retain cycle | Dễ quên `[weak self]` |
| Dài dòng hơn | Gọn, đặt logic ngay tại chỗ gọi |
Nhiều người nghe cùng lúc (1-n) → dùng `NotificationCenter` hoặc Combine (tuần 8).

---

## ⚠️ Bẫy hay gặp

1. **Closure capture *biến*, không capture *giá trị*** (trừ khi dùng capture list `[x]`).
2. **Closure là reference type** — gán sang biến khác vẫn dùng chung state.
3. **Closure Optional luôn escaping** — `var onTap: (() -> Void)?` cần `[weak self]`.
4. **`forEach` + `return`** không dừng vòng lặp.
5. **`reduce` để gom vào Array/Dictionary là O(n²)** → dùng `reduce(into:)`.
6. **`flatMap` để bỏ nil đã deprecated** → `compactMap`.
7. **Method chỉ khai báo trong protocol extension dùng static dispatch** → gọi qua `any P` hoặc generic sẽ ra bản extension.
8. **Delegate phải `weak`**, protocol phải `AnyObject`.
9. **Extension không thêm được stored property.**
10. **Tự viết `==` mà quên viết `hash(into:)` tương ứng** → vi phạm luật "bằng nhau thì cùng hash", Set/Dictionary chạy sai.
11. **Object tạm biến mất**: `Launcher().countdown.run()` với delegate `weak` — `Launcher` bị giải phóng ngay sau biểu thức, delegate thành `nil`, không có gì được gọi. Phải giữ object trong một biến.

---

## 🛠 Bài tập

> Tạo file trong `practice/week03/` và chạy bằng `swift tên_file.swift`. **Không xem lời giải trước khi tự làm ít nhất 20 phút.**

**Bài 1 — Closure trả về closure (T2)**
a) `makeMultiplier(_ factor: Int) -> (Int) -> Int`.
b) `makeAccumulator(start: Int) -> (Int) -> Int`: mỗi lần gọi cộng thêm và trả tổng hiện tại. `acc(5), acc(5), acc(-20)` với start 10 → `15 20 0`.
c) Viết lại `names.sorted(by:)` theo 4 dạng từ đầy đủ đến gọn nhất (§1) — gõ tay, không copy.
<details><summary>Lời giải</summary>

```swift
func makeMultiplier(_ factor: Int) -> (Int) -> Int { { $0 * factor } }
let triple = makeMultiplier(3)
print(triple(5))   // 15

func makeAccumulator(start: Int) -> (Int) -> Int {
    var total = start
    return { total += $0; return total }
}
let acc = makeAccumulator(start: 10)
print(acc(5), acc(5), acc(-20))   // 15 20 0
```
`total` được box lên heap vì closure escape khỏi hàm.
</details>

**Bài 2 — Dự đoán output capture (T2)**
Không chạy, viết ra output, rồi chạy kiểm tra:
```swift
var a = 1
var b = 1
let f = { [a] in print("f:", a, b) }
a = 2; b = 2
f()

final class Counter { var n = 0 }
var counter = Counter()
let g = { [counter] in counter.n += 1; print("g:", counter.n) }
counter = Counter()
g(); g()
print(counter.n)
```
<details><summary>Lời giải</summary>

```
f: 1 2
g: 1
g: 2
0
```
`a` được chụp giá trị (1) lúc tạo closure; `b` capture theo tham chiếu → thấy giá trị mới. `[counter]` chụp **tham chiếu tới object cũ**; gán `counter` sang object mới không ảnh hưởng closure, nên object mới vẫn `n == 0`.
</details>

**Bài 3 — `@escaping`: EventBus (T3)**
Viết `final class EventBus` với `subscribe(_ event: String, handler: ...)` và `publish(_ event: String, payload: String)`. Một event có nhiều handler. Thử bỏ `@escaping` và đọc lỗi.
<details><summary>Lời giải</summary>

```swift
final class EventBus {
    private var handlers: [String: [(String) -> Void]] = [:]
    func subscribe(_ event: String, handler: @escaping (String) -> Void) {
        handlers[event, default: []].append(handler)    // lưu lại → bắt buộc @escaping
    }
    func publish(_ event: String, payload: String) {
        handlers[event]?.forEach { $0(payload) }
    }
}
let bus = EventBus()
bus.subscribe("login") { print("Analytics: \($0)") }
bus.subscribe("login") { user in print("Chào \(user)") }
bus.publish("login", payload: "An")     // Analytics: An / Chào An
bus.publish("logout", payload: "An")    // không có ai nghe → không in
```
Câu hỏi thêm: nếu một ViewController subscribe và trong closure dùng `self` → ai giữ ai? (Bus giữ closure, closure giữ VC → nếu bus sống mãi thì VC không bao giờ được giải phóng → cần `[weak self]` và cơ chế unsubscribe.)
</details>

**Bài 4 — Higher-order function (T4)**
a) Cho `let raw = ["food:120", "transport:50", "food:80", "oops", "fun:200", "transport:x", "food:30"]`. Parse thành `[Txn]` (bỏ dòng lỗi) bằng `compactMap`; tính tổng theo category bằng `reduce(into:)`; in theo tổng giảm dần; lấy danh sách category của các giao dịch ≥ 100.
b) Tự cài `myMap`, `myFilter`, `myReduce` trong `extension Array` (dùng generic `<T>` đơn giản — tuần 4 học kỹ).
<details><summary>Lời giải</summary>

```swift
struct Txn { let category: String; let amount: Int }
let raw = ["food:120", "transport:50", "food:80", "oops", "fun:200", "transport:x", "food:30"]
let txns: [Txn] = raw.compactMap { line in
    let parts = line.split(separator: ":")
    guard parts.count == 2, let amount = Int(parts[1]) else { return nil }
    return Txn(category: String(parts[0]), amount: amount)
}
print(txns.count)   // 5
let byCategory = txns.reduce(into: [String: Int]()) { $0[$1.category, default: 0] += $1.amount }
for (k, v) in byCategory.sorted(by: { $0.value > $1.value }) { print(k, v) }
// food 230 / fun 200 / transport 50
print(txns.filter { $0.amount >= 100 }.map(\.category))   // ["food", "fun"]

extension Array {
    func myMap<T>(_ transform: (Element) -> T) -> [T] {
        var result: [T] = []
        result.reserveCapacity(count)
        for x in self { result.append(transform(x)) }
        return result
    }
    func myFilter(_ isIncluded: (Element) -> Bool) -> [Element] {
        var result: [Element] = []
        for x in self where isIncluded(x) { result.append(x) }
        return result
    }
    func myReduce<R>(_ initial: R, _ combine: (R, Element) -> R) -> R {
        var acc = initial
        for x in self { acc = combine(acc, x) }
        return acc
    }
}
print([1, 2, 3].myMap { $0 * 2 }, [1, 2, 3, 4].myFilter { $0.isMultiple(of: 2) }, [1, 2, 3].myReduce(0, +))
// [2, 4, 6] [2, 4] 6
```
Để ý: `transform` không cần `@escaping` vì chỉ được gọi trong hàm.
</details>

**Bài 5 — Protocol + default implementation (T5)**
`protocol Shape: CustomStringConvertible` có `name`, `area`. Default `description` dạng `"Circle (3.14)"` (làm tròn 2 chữ số) trong extension. Tạo `Circle`, `Rectangle`; sắp `[any Shape]` theo diện tích tăng dần.
<details><summary>Lời giải</summary>

```swift
protocol Shape: CustomStringConvertible {
    var name: String { get }
    var area: Double { get }
}
extension Shape {
    var description: String { "\(name) (\((area * 100).rounded() / 100))" }
}
struct Circle: Shape {
    let r: Double
    var name: String { "Circle" }
    var area: Double { .pi * r * r }
}
struct Rectangle: Shape {
    let w, h: Double
    var name: String { "Rectangle" }
    var area: Double { w * h }
}
let shapes: [any Shape] = [Circle(r: 1), Rectangle(w: 2, h: 3), Circle(r: 0.5)]
print(shapes.sorted { $0.area < $1.area }.map(\.description))
// ["Circle (0.79)", "Circle (3.14)", "Rectangle (6.0)"]
```
`[any Shape]` không dùng được `sorted()` không tham số vì `Shape` không `Comparable` — phải truyền closure.
</details>

**Bài 6 — Bẫy dispatch (T6)**
Không chạy, dự đoán output, rồi chạy kiểm tra và giải thích bằng khái niệm witness table:
```swift
protocol Greeting { func hello() -> String }
extension Greeting {
    func hello() -> String { "Hello từ extension" }
    func bye() -> String { "Bye từ extension" }
}
struct Robot: Greeting { func bye() -> String { "Bye từ Robot" } }
let r: any Greeting = Robot()
print(r.hello(), "|", r.bye())
```
Sửa protocol sao cho dòng in ra `Hello từ extension | Bye từ Robot`.
<details><summary>Lời giải</summary>

Ban đầu: `Hello từ extension | Bye từ extension`. `Robot` không cài `hello` → witness dùng bản extension; `bye` không phải requirement → static dispatch theo kiểu tĩnh `any Greeting` → bản extension.
Sửa: thêm `func bye() -> String` vào thân protocol:
```swift
protocol Greeting { func hello() -> String; func bye() -> String }
// (extension giữ nguyên)
let r: any Greeting = Robot()
print(r.hello(), "|", r.bye())   // Hello từ extension | Bye từ Robot
```
</details>

**Bài 7 — Delegate & closure (T6)**
Viết `final class Countdown(from:)` có `weak var delegate: CountdownDelegate?` (`didTick remaining:`, `countdownDidFinish`) **và** `var onTick: ((Int) -> Void)?`. `Launcher` dùng cả hai. Sau đó thử viết `Launcher().countdown.run()` và giải thích vì sao không in gì.
<details><summary>Lời giải</summary>

```swift
protocol CountdownDelegate: AnyObject {
    func countdown(_ c: Countdown, didTick remaining: Int)
    func countdownDidFinish(_ c: Countdown)
}
final class Countdown {
    weak var delegate: CountdownDelegate?
    var onTick: ((Int) -> Void)?               // Optional closure → ngầm escaping
    let from: Int
    init(from: Int) { self.from = from }
    func run() {
        for i in stride(from: from, to: 0, by: -1) {
            delegate?.countdown(self, didTick: i)
            onTick?(i)
        }
        delegate?.countdownDidFinish(self)
    }
}
final class Launcher: CountdownDelegate {
    let countdown = Countdown(from: 3)
    init() {
        countdown.delegate = self
        countdown.onTick = { [weak self] i in self?.log("closure \(i)") }   // weak: tránh cycle
    }
    func log(_ s: String) { print(s) }
    func countdown(_ c: Countdown, didTick remaining: Int) { print("delegate \(remaining)") }
    func countdownDidFinish(_ c: Countdown) { print("🚀") }
}
let launcher = Launcher()
launcher.countdown.run()
// delegate 3 / closure 3 / delegate 2 / closure 2 / delegate 1 / closure 1 / 🚀
```
`Launcher().countdown.run()`: object `Launcher` tạm không ai giữ strong → bị giải phóng ngay sau khi lấy `.countdown`; `delegate` (weak) và `self` trong closure (weak) đều thành `nil` → không in gì, kể cả 🚀.
</details>

**🏋️ Bài tập lớn (T7) — Form validator bằng protocol + closure**
Viết `validator.swift` (nhớ `import Foundation`):
- `protocol ValidationRule { var message: String { get }; func isValid(_ value: String) -> Bool }`
- Extension cho `ValidationRule` thêm `func validate(_:) -> String?` (nil nếu hợp lệ, ngược lại trả `message`).
- Các rule: `NotEmpty`, `MinLength(min:)`, `BlockRule(message:predicate:)` nhận closure.
- Thêm static factory `.containsDigit` (gợi ý: `extension ValidationRule where Self == BlockRule`).
- `struct Field { name, value, rules: [any ValidationRule] }` có `errors: [String]` (dùng `compactMap`).
- `struct FormValidator` có `isValid` (dùng `allSatisfy`) và `report() -> [String: [String]]` (dùng `reduce(into:)`).
- Test với username `"an"` (lỗi độ dài), password `"secret"` (thiếu số), email `"an@x.com"` (hợp lệ).

<details><summary>Lời giải tham khảo</summary>

```swift
import Foundation

protocol ValidationRule {
    var message: String { get }
    func isValid(_ value: String) -> Bool
}
extension ValidationRule {
    func validate(_ value: String) -> String? { isValid(value) ? nil : message }
}

struct NotEmpty: ValidationRule {
    var message = "Không được để trống"
    func isValid(_ value: String) -> Bool { !value.trimmingCharacters(in: .whitespaces).isEmpty }
}
struct MinLength: ValidationRule {
    let min: Int
    var message: String { "Tối thiểu \(min) ký tự" }
    func isValid(_ value: String) -> Bool { value.count >= min }
}
struct BlockRule: ValidationRule {
    let message: String
    let predicate: (String) -> Bool
    func isValid(_ value: String) -> Bool { predicate(value) }
}
extension ValidationRule where Self == BlockRule {
    static var containsDigit: BlockRule {
        BlockRule(message: "Phải có ít nhất 1 chữ số") { $0.contains(where: \.isNumber) }
    }
}

struct Field {
    let name: String
    let value: String
    let rules: [any ValidationRule]
    var errors: [String] { rules.compactMap { $0.validate(value) } }
}

struct FormValidator {
    let fields: [Field]
    var isValid: Bool { fields.allSatisfy { $0.errors.isEmpty } }
    func report() -> [String: [String]] {
        fields.reduce(into: [:]) { result, field in
            let errs = field.errors
            if !errs.isEmpty { result[field.name] = errs }
        }
    }
}

let form = FormValidator(fields: [
    Field(name: "username", value: "an", rules: [NotEmpty(), MinLength(min: 3)]),
    Field(name: "password", value: "secret", rules: [MinLength(min: 6), BlockRule.containsDigit]),
    Field(name: "email", value: "an@x.com",
          rules: [NotEmpty(), BlockRule(message: "Email không hợp lệ") { $0.contains("@") }]),
])
print(form.isValid)                               // false
for (k, v) in form.report().sorted(by: { $0.key < $1.key }) { print(k, v) }
// password ["Phải có ít nhất 1 chữ số"]
// username ["Tối thiểu 3 ký tự"]
```
Tự hỏi: `validate` nằm trong extension, không phải requirement → nếu một rule tự cài `validate` riêng thì gọi qua `any ValidationRule` sẽ ra bản nào? (§9)
</details>

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Closure là gì? Closure là value type hay reference type?**
<details><summary>Đáp án gợi ý</summary>

Closure là khối code có thể truyền đi, lưu trữ và gọi sau, có khả năng capture các biến/hằng trong ngữ cảnh nơi nó được tạo. Hàm toàn cục và method thực chất là các dạng closure đặc biệt. Closure là **reference type**: gán closure cho biến khác thì hai biến cùng trỏ tới một closure và chia sẻ các biến đã capture.
</details>

**Q2 ⭐ Closure capture biến như thế nào? Capture list dùng để làm gì?**
<details><summary>Đáp án gợi ý</summary>

Mặc định closure capture **biến** theo tham chiếu: thay đổi bên ngoài sau khi tạo closure thì closure thấy, và closure sửa thì bên ngoài thấy; nếu closure escape, biến được đưa lên heap (box) để sống tiếp. Capture list `[x]` chụp **giá trị** của `x` tại thời điểm tạo closure. Với class, capture list chụp tham chiếu (vẫn strong) tại thời điểm đó; `[weak self]`/`[unowned self]` đổi kiểu tham chiếu để tránh retain cycle.
</details>

**Q3 ⭐ `@escaping` là gì? Vì sao closure mặc định là non-escaping?**
<details><summary>Đáp án gợi ý</summary>

Escaping closure có thể được gọi sau khi hàm nhận nó đã return — được lưu vào property hoặc chạy bất đồng bộ (completion handler). Non-escaping được đảm bảo chạy xong trong hàm. Mặc định non-escaping vì: compiler tối ưu được (context trên stack, không cần retain), an toàn hơn (không gây retain cycle nên không bắt viết `self.`), và làm API rõ ràng — thấy `@escaping` là biết phải chú ý memory. Lưu ý: closure Optional luôn ngầm là escaping.
</details>

**Q4 Vì sao trong escaping closure của class phải viết `self.` tường minh?**
<details><summary>Đáp án gợi ý</summary>

Để buộc lập trình viên nhận thức rằng closure đang capture strong `self` — nguy cơ retain cycle nếu `self` cũng giữ closure. Có thể thay bằng `[weak self]` hoặc `[self]` trong capture list. Với struct thì không bắt (Swift 5.3+) vì value type không gây retain cycle; và sau `guard let self` trong closure `[weak self]` thì được dùng self ngầm định (Swift 5.8+).
</details>

**Q5 `@autoclosure` là gì? Cho ví dụ trong thư viện chuẩn.**
<details><summary>Đáp án gợi ý</summary>

Tự động bọc một biểu thức truyền vào thành closure không tham số, giúp trì hoãn việc tính biểu thức đến khi thật sự cần (lazy evaluation) mà cú pháp gọi vẫn như truyền giá trị. Ví dụ: `assert(condition, message)` — message chỉ tính khi assert fail; `??` — vế phải chỉ tính khi vế trái nil; `&&`, `||` để short-circuit.
</details>

**Q6 ⭐ `map`, `compactMap`, `flatMap` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`map` biến đổi từng phần tử, giữ nguyên số lượng (closure trả Optional thì được mảng Optional). `compactMap` biến đổi và loại bỏ kết quả `nil`. `flatMap` dùng khi closure trả về một sequence — nối các sequence lại, làm phẳng một cấp (`[[Int]] → [Int]`). Trên Optional, `flatMap` tránh `Optional` lồng nhau.
</details>

**Q7 `reduce` và `reduce(into:)` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`reduce(_:_:)` nhận closure `(Result, Element) -> Result`, mỗi bước trả về giá trị mới. `reduce(into:_:)` nhận closure `(inout Result, Element) -> Void`, sửa accumulator tại chỗ. Khi accumulator là Array/Dictionary, `reduce(into:)` tránh copy mỗi bước (O(n) thay vì O(n²)) nên nên dùng để gom nhóm, đếm.
</details>

**Q8 `forEach` khác `for-in` thế nào?**
<details><summary>Đáp án gợi ý</summary>

`forEach` nhận closure nên không dùng được `break`/`continue`; `return` chỉ thoát lần gọi closure hiện tại (giống `continue`). Không thoát sớm được. `for-in` hỗ trợ đầy đủ control flow, `where`, và dễ đọc khi thân vòng lặp dài. `forEach` hợp ở cuối một chuỗi functional ngắn.
</details>

**Q9 `lazy` trên collection dùng khi nào? Có nhược điểm gì?**
<details><summary>Đáp án gợi ý</summary>

`.lazy` biến chuỗi `map/filter/...` thành tính toán trì hoãn: không tạo mảng trung gian, chỉ tính những phần tử thực sự được dùng — hợp với dữ liệu lớn khi chỉ cần vài phần tử đầu (`first`, `prefix`). Nhược điểm: không cache, mỗi lần truy cập lại chạy closure; closure có side effect có thể chạy nhiều lần hoặc không chạy; debug khó hơn.
</details>

**Q10 ⭐ Protocol là gì? Default implementation hoạt động thế nào?**
<details><summary>Đáp án gợi ý</summary>

Protocol định nghĩa một tập requirement (property, method, init, subscript, associated type) mà kiểu conform phải đáp ứng — một hợp đồng không chứa trạng thái. Struct, enum, class, actor đều conform được và có thể conform nhiều protocol. Default implementation viết trong `extension` của protocol; kiểu conform không tự cài thì dùng bản mặc định — đây là cách Swift làm optional requirement và tái sử dụng hành vi. Có thể giới hạn default bằng `where` (constrained extension).
</details>

**Q11 ⭐ Protocol-Oriented Programming là gì? So với OOP?**
<details><summary>Đáp án gợi ý</summary>

POP thiết kế quanh các protocol nhỏ, mỗi cái mô tả một khả năng, kèm default implementation trong extension; kiểu cụ thể (thường là struct) ghép nhiều protocol lại. So với kế thừa class: không bị giới hạn đơn kế thừa, không thừa hưởng thứ không cần, dùng được cho value type (tránh shared mutable state), và liên kết lỏng — dễ thay bằng mock khi test. OOP vẫn cần khi framework yêu cầu (UIKit) hoặc cần chia sẻ stored state và override. Thực tế kết hợp cả hai.
</details>

**Q12 ⭐ Một method khai báo chỉ trong protocol extension, kiểu conform cũng cài method cùng tên. Gọi qua biến kiểu protocol thì chạy bản nào? Tại sao?**
<details><summary>Đáp án gợi ý</summary>

Chạy bản trong **extension**. Chỉ các requirement khai báo trong thân protocol mới có mặt trong protocol witness table và được dynamic dispatch. Method chỉ có trong extension được static dispatch theo kiểu tĩnh của biến: kiểu tĩnh là protocol (hoặc generic `T: P`) thì gọi bản extension; kiểu tĩnh là kiểu cụ thể thì gọi bản của kiểu đó. Muốn được "override" thì phải khai báo method đó làm requirement.
</details>

**Q13 Class-only protocol là gì? Khi nào cần?**
<details><summary>Đáp án gợi ý</summary>

Protocol kế thừa `AnyObject` (`protocol P: AnyObject`) chỉ cho class (và actor) conform. Cần khi muốn giữ tham chiếu `weak`/`unowned` tới kiểu conform — điển hình là delegate — vì `weak` chỉ áp dụng cho reference type. Cũng cần khi logic dựa trên identity (`===`).
</details>

**Q14 ⭐ Vì sao delegate phải là `weak`? Delegate khác closure callback thế nào?**
<details><summary>Đáp án gợi ý</summary>

Thường object chủ (ví dụ ViewController) giữ strong object con (table view, downloader), object con lại trỏ ngược về chủ qua delegate. Nếu delegate cũng strong → retain cycle, cả hai không bao giờ được giải phóng. Delegate hợp cho nhóm nhiều callback liên quan, quan hệ 1-1 lâu dài; closure gọn hơn cho 1–2 callback hoặc completion một lần, nhưng dễ quên `[weak self]`. Nhiều người nghe → NotificationCenter/Combine.
</details>

**Q15 Equatable, Hashable, Comparable: compiler tự sinh khi nào? Quy tắc khi tự viết?**
<details><summary>Đáp án gợi ý</summary>

Struct tự sinh `Equatable`/`Hashable` khi mọi stored property đều conform; enum khi mọi associated value đều conform (enum không associated value thì luôn có). `Comparable` chỉ tự sinh cho enum không có raw value. Khi tự viết `==` và `hash(into:)` phải dùng **cùng tập thuộc tính**: hai giá trị bằng nhau bắt buộc cùng hash; nếu không Set/Dictionary sẽ hoạt động sai. `Comparable` chỉ cần cài `<`.
</details>

**Q16 Extension làm được gì và không làm được gì?**
<details><summary>Đáp án gợi ý</summary>

Được: thêm computed property, method, `mutating` method, convenience init (class) hoặc init (struct — vẫn giữ memberwise), subscript, nested type, conformance protocol, constrained extension với `where`. Không được: thêm stored property (thay đổi layout bộ nhớ), override method có sẵn của class Swift, thêm designated init cho class. Có thể "giả lập" stored property bằng associated object của Objective-C runtime nhưng không khuyến khích.
</details>

**Q17 Protocol có thể có stored property không? `{ get }` và `{ get set }` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Protocol chỉ yêu cầu *có* property với tên và kiểu, không quy định stored hay computed. `{ get }` nghĩa là ít nhất đọc được — thoả bởi `let`, `var` hay computed read-only. `{ get set }` phải đọc và ghi được — không thoả bằng `let` hay computed chỉ có get. Extension của protocol không thêm được stored property.
</details>

**Q18 Higher-order function là gì? Tự cài `map` thế nào?**
<details><summary>Đáp án gợi ý</summary>

Là hàm nhận hàm khác làm tham số hoặc trả về hàm. `map` tự cài: tạo mảng kết quả rỗng, `reserveCapacity(count)`, duyệt từng phần tử, append `transform(x)`, trả kết quả. Closure `transform` không cần `@escaping` vì chỉ gọi trong hàm. `map` thật trong stdlib còn khai báo `rethrows` để closure ném lỗi được (tuần 4).
</details>

**Q19 Khi nào dùng `any Protocol` làm kiểu của mảng? Có hạn chế gì?**
<details><summary>Đáp án gợi ý</summary>

Khi cần chứa nhiều kiểu cụ thể khác nhau cùng conform một protocol (`[any Shape]` chứa cả Circle và Rectangle). Hạn chế: mỗi phần tử được bọc trong existential container (có thể cấp phát heap nếu giá trị lớn), gọi method qua witness table (dynamic dispatch), mất thông tin kiểu cụ thể nên không dùng được các API yêu cầu `Self` như `==` hay `sorted()` không tham số. Tuần 4 so sánh chi tiết với `some` và generic.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Làm xong Bài tập lớn `validator.swift`
- [ ] Trả lời được ≥ 15/19 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Closure capture biến thế nào, escaping là gì"
- [ ] Ghi âm 2 phút: "Bẫy method trong protocol extension — giải thích bằng witness table"
- [ ] Tự viết lại delegate pattern từ trí nhớ (protocol `AnyObject` + `weak var`)
- [ ] Làm xong 6 bài DSA tuần 3 (Two Pointers)
- [ ] Side project Milestone 3 xong (`RepositoryServiceProtocol`, `MockRepositoryService`, extension)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)
- [The Swift Programming Language — Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [The Swift Programming Language — Extensions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/)
- [The Swift Programming Language — Automatic Reference Counting: Strong Reference Cycles for Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/#Strong-Reference-Cycles-for-Closures) (đọc lướt, tuần 5 học kỹ)
- WWDC 2015 — *Protocol-Oriented Programming in Swift* (bài nói kinh điển, "Crusty")
- WWDC 2016 — *Protocol and Value Oriented Programming in UIKit Apps*
- WWDC 2016 — *Understanding Swift Performance* (phần protocol witness table, existential container)
- Swift by Sundell — *Capturing objects in Swift closures*, *Using autoclosure when designing Swift APIs*
- Apple Developer — [Equatable](https://developer.apple.com/documentation/swift/equatable), [Hashable](https://developer.apple.com/documentation/swift/hashable)
