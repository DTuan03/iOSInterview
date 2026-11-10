# Tuần 2 — Struct, Class, Enum · Value vs Reference · Copy-on-Write (16/11 – 22/11/2026)

> **Giai đoạn 1 · Swift Core** · Thời lượng: ~14 giờ
> Đây là tuần **bị hỏi nhiều nhất** trong cả Giai đoạn 1. Gần như 100% buổi phỏng vấn iOS có câu "struct khác class thế nào?" — và người phỏng vấn sẽ hỏi xoáy tiếp: copy-on-write, stack/heap, dispatch, init. Trả lời bề mặt là bị đánh giá "junior".

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích **value semantics vs reference semantics** bằng ví dụ code tự viết, kể cả trường hợp **struct chứa class**
- [ ] Liệt kê ≥ 6 điểm khác nhau giữa `struct` và `class` và nói được **khi nào chọn cái nào**
- [ ] Dùng thành thạo `enum` với raw value, associated value, `indirect`, `CaseIterable`
- [ ] Phân biệt stored / computed / lazy property, property observer, `static` vs `class`
- [ ] Giải thích designated vs convenience init, **two-phase initialization**, failable, `required`
- [ ] Giải thích **copy-on-write** hoạt động thế nào và **tự cài CoW** cho một struct bằng `isKnownUniquelyReferenced`
- [ ] Nói được struct có **luôn** nằm trên stack không (không!), và 3 kiểu method dispatch
- [ ] Làm xong 6 bài DSA tuần 2 (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 16/11** | 2h | Tạo `practice/week02/`. §1 **Value vs Reference** (đọc chậm, gõ lại toàn bộ ví dụ). Bài tập 1 · DSA: Group Anagrams | `day1.swift` |
| **T3 17/11** | 2h | §2 **Enum**: raw value, associated value, `indirect`. Bài tập 2, 3 · DSA: Top K Frequent Elements | `day2.swift` |
| **T4 18/11** | 2h | §3 **Property**: stored, computed, lazy, observer, static. Bài tập 4 · DSA: Product of Array Except Self | `day3.swift` |
| **T5 19/11** | 2h | §4 **Initializer** (phần khó nhất tuần — đọc 2 lần). Bài tập 5 · DSA: Longest Consecutive Sequence | `day4.swift` |
| **T6 20/11** | 1,5h | §5 `mutating`, kế thừa, `override`, `final` + §7 **Method dispatch**. Bài tập 6 · DSA: Valid Sudoku | `day5.swift` |
| **T7 21/11** | 3,5h | §6 **Stack/heap + Copy-on-write**. Bài tập 7 + Bài tập lớn (Playlist CoW). §8 Chọn struct hay class. Side project: Milestone 2 (model layer `Repository`/`Owner`, mock data) | `day6.swift`, `playlist.swift`, commit side project |
| **CN 22/11** | 2h | Tự trả lời 18 câu phỏng vấn ra giấy, rồi mới mở đáp án. Ghi âm 2 câu. DSA: làm lại Group Anagrams không nhìn code cũ. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. Value type vs Reference type — gốc rễ của mọi thứ tuần này

Trong Swift chỉ có 2 "họ" kiểu:

| Value type (giá trị) | Reference type (tham chiếu) |
|---|---|
| `struct`, `enum`, tuple | `class`, closure, actor |
| `Int`, `Double`, `Bool`, `String`, `Array`, `Dictionary`, `Set`, `Optional`… đều là **struct/enum** | `UIView`, `UIViewController`, `URLSession`, `NSObject`… |
| Gán / truyền vào hàm = **tạo bản sao độc lập** | Gán / truyền vào hàm = **copy cái tham chiếu**, cả hai cùng trỏ tới **một object** |

```swift
struct PointS { var x: Int; var y: Int }
class PointC {
    var x: Int; var y: Int
    init(x: Int, y: Int) { self.x = x; self.y = y }
}

var a = PointS(x: 1, y: 1)
var b = a              // b là BẢN SAO của a
b.x = 99
print(a.x, b.x)        // 1 99   ← a không bị ảnh hưởng

let c = PointC(x: 1, y: 1)
let d = c              // d và c cùng trỏ tới MỘT object
d.x = 99
print(c.x, d.x)        // 99 99  ← sửa qua d thì c cũng thấy
print(c === d)         // true   ← === : cùng một instance
```

Hình dung bộ nhớ:
```
Value:      a ─ [x:1, y:1]        b ─ [x:99, y:1]       (2 vùng nhớ riêng)

Reference:  c ──┐
                ├──► heap: PointC { x: 99, y: 1, refCount: 2 }
            d ──┘
```

**Value semantics** nghĩa là: "*sửa bản này không bao giờ làm thay đổi bản kia*". Đây là thứ quan trọng, không phải chuyện "struct hay class". Lợi ích:
- **Dễ suy luận**: một hàm nhận struct không thể lén sửa dữ liệu của bạn.
- **An toàn đa luồng hơn**: mỗi thread có bản sao riêng → không có data race trên bản sao đó.
- **Không có retain cycle**, không cần `weak`/`unowned` (tuần 5).

#### Bẫy kinh điển: struct chứa class
```swift
class Engine { var hp = 100 }
struct Car { var name: String; let engine: Engine }

let car1 = Car(name: "A", engine: Engine())
var car2 = car1           // copy struct → copy name, NHƯNG chỉ copy tham chiếu engine
car2.name = "B"
car2.engine.hp = 300      // let engine chỉ khoá tham chiếu, không khoá object
print(car1.name, car1.engine.hp)   // A 300   ← car1 bị ảnh hưởng!
```
→ Một struct **chỉ có value semantics khi mọi thứ bên trong nó cũng có value semantics**. Struct chứa class mutable = "value type giả". (Muốn sửa: dùng copy-on-write ở §6, hoặc đổi `Engine` thành struct.)

#### `let` với struct vs class (nhắc lại từ tuần 1, giờ hiểu tại sao)
- `let` struct: giá trị **chính là** dữ liệu → khoá toàn bộ, không sửa được thuộc tính nào, không gọi được `mutating` method.
- `let` class: giá trị của biến **chỉ là địa chỉ** → khoá địa chỉ, object vẫn sửa được.
```swift
struct P { var x = 0 }
let p = P()
// p.x = 10   // ❌ error: cannot assign to property: 'p' is a 'let' constant
```

#### Bảng so sánh struct vs class (thuộc lòng)
| | `struct` | `class` |
|---|---|---|
| Semantics | Value (copy khi gán) | Reference (chia sẻ) |
| Kế thừa | ❌ (chỉ conform protocol) | ✅ single inheritance |
| Memberwise init tự sinh | ✅ | ❌ (phải tự viết `init`, trừ khi mọi property có default) |
| Sửa thuộc tính trong method | Phải đánh dấu `mutating` | Không cần |
| `deinit` | ❌ | ✅ |
| Identity (`===`) | ❌ | ✅ |
| Quản lý bộ nhớ | Không có ARC cho bản thân struct (nhưng property là class thì vẫn có) | ARC đếm tham chiếu (tuần 5) |
| Cấp phát | Thường inline (stack / bên trong object chứa nó) | Heap |
| Method dispatch mặc định | Static | Dynamic (vtable) nếu không `final` |
| Type casting `as?`, `is` lúc runtime để kiểm tra lớp con | — | ✅ |
| Interop Objective-C | Hạn chế | ✅ (kế thừa `NSObject`) |

### §2. Enum

Enum trong Swift là **value type hạng nhất**: có method, computed property, conform protocol, có extension, có `mutating`.

#### 2.1. Enum cơ bản & `CaseIterable`
```swift
enum Direction: CaseIterable { case north, south, east, west }
print(Direction.allCases.count)   // 4
```
Enum không có associated value thì compiler tự cho `Equatable` và `Hashable`.

#### 2.2. Raw value — mỗi case gắn **một hằng số cố định cùng kiểu**
```swift
enum HTTPMethod: String {
    case get = "GET", post = "POST"
    case delete                       // raw value ngầm định = "delete" (tên case)
}
print(HTTPMethod.get.rawValue, HTTPMethod.delete.rawValue)   // GET delete

let m = HTTPMethod(rawValue: "POST")  // init?(rawValue:) — FAILABLE, trả HTTPMethod?
print(m == .post)                      // true
print(HTTPMethod(rawValue: "PUT") == nil)   // true

enum Planet: Int { case mercury = 1, venus, earth }
print(Planet.earth.rawValue)          // 3 — Int tự tăng từ case trước
```
- Raw value phải là literal: `String`, `Character`, số nguyên, số thực.
- Raw value **xác định lúc compile**, giống nhau cho mọi instance của case đó.
- Dùng nhiều để map với API/JSON (`Codable` tuần 4 sẽ tự dùng raw value).

#### 2.3. Associated value — mỗi **instance** mang dữ liệu riêng
```swift
enum LoadState {
    case idle
    case loading(progress: Double)
    case loaded([String])
    case failed(Error)
}

func render(_ s: LoadState) -> String {
    switch s {
    case .idle:                                     return "Chưa tải"
    case .loading(let p):                           return "Đang tải \(Int(p * 100))%"
    case .loaded(let items) where items.isEmpty:    return "Trống"
    case .loaded(let items):                        return "\(items.count) mục"
    case .failed(let e):                            return "Lỗi: \(e)"
    }
}
print(render(.loading(progress: 0.42)))   // Đang tải 42%
print(render(.loaded([])))                // Trống
print(render(.loaded(["a", "b"])))        // 2 mục
```
Đây là cách mô hình hoá **trạng thái màn hình** rất hay dùng trong MVVM: không thể vừa `loading` vừa `failed` cùng lúc — điều mà 3 biến `isLoading`, `error`, `items` riêng lẻ **không** đảm bảo được. ("Make impossible states impossible.")

| | Raw value | Associated value |
|---|---|---|
| Khi nào biết giá trị | Compile time | Runtime, mỗi instance khác nhau |
| Kiểu | Cùng 1 kiểu cho mọi case | Mỗi case một bộ kiểu riêng |
| Có `init?(rawValue:)` | ✅ | ❌ |
| Dùng chung được không | ❌ — `enum E: String { case a(Int) }` → *error: enum with raw type cannot have cases with arguments* | |

> Kích thước của enum: enum không có associated value chỉ tốn 1 byte (`MemoryLayout<Direction>.size == 1`). Enum có associated value có kích thước ≈ case lớn nhất + tag. `Optional<Int>` tốn 9 byte (8 + 1 tag), nhưng `Optional<String>` vẫn 16 byte như `String` — Swift giấu tag vào bit thừa ("spare bits") của payload.

#### 2.4. `indirect` — enum đệ quy
Value type phải biết **kích thước cố định** lúc compile. Một enum chứa chính nó thì kích thước là vô hạn → lỗi. `indirect` bảo compiler cất associated value vào **một box trên heap**, enum chỉ giữ con trỏ.
```swift
// enum Tree { case leaf(Int); case node(Tree, Tree) }
// ❌ error: recursive enum 'Tree' is not marked 'indirect'

indirect enum Expr {
    case number(Int)
    case add(Expr, Expr)
    case multiply(Expr, Expr)
}
func eval(_ e: Expr) -> Int {
    switch e {
    case .number(let n):          return n
    case .add(let l, let r):      return eval(l) + eval(r)
    case .multiply(let l, let r): return eval(l) * eval(r)
    }
}
print(eval(.multiply(.add(.number(2), .number(3)), .number(4))))   // 20
```
Có thể đặt `indirect` chỉ trên từng case: `indirect case add(Expr, Expr)`.
**Struct thì không có `indirect`**: `struct Node { var v: Int; var next: Node? }` → *error: value type 'Node' cannot have a stored property that recursively contains it*. Muốn linked list bằng struct phải bọc qua class hoặc Array.

#### 2.5. Method & `mutating` trong enum
```swift
enum Toggle {
    case on, off
    mutating func flip() { self = (self == .on) ? .off : .on }   // gán lại cả self
    var isOn: Bool { self == .on }
}
var t = Toggle.off
t.flip()
print(t, t.isOn)   // on true
```
Enum không có stored property (chỉ có computed và `static`).

### §3. Property

#### 3.1. Stored vs computed
```swift
struct Rect {
    var width: Double                 // stored: chiếm bộ nhớ
    var height: Double
    var area: Double { width * height }   // computed read-only: tính mỗi lần gọi, không chiếm bộ nhớ
    var square: Double {                  // computed get/set
        get { width }
        set { width = newValue; height = newValue }   // newValue là tên mặc định
    }
}
var r = Rect(width: 2, height: 3)
print(r.area)                        // 6.0
r.square = 5
print(r.width, r.height, r.area)     // 5.0 5.0 25.0
```
Computed property phải là `var` (kể cả read-only). Quy ước: computed property nên **rẻ** (O(1) hoặc gần thế) và không có side effect; nếu tốn kém thì viết thành method để người gọi biết.

#### 3.2. `lazy` — chỉ tính khi được truy cập lần đầu
```swift
final class ImageLoader {
    lazy var cache: [String: Int] = {
        print("Tạo cache")
        return [:]
    }()
    init() { print("init") }
}
let loader = ImageLoader()   // in "init" — cache CHƯA được tạo
loader.cache["a"] = 1        // in "Tạo cache" — lần truy cập đầu
loader.cache["b"] = 2        // không in gì nữa
print(loader.cache.count)    // 2
```
- Phải là `var` (`lazy let` → *error: 'lazy' cannot be used on a let*), vì giá trị được gán **sau** init.
- Initializer của lazy **được dùng `self`** (stored thường thì không) → hay dùng để tạo subview phụ thuộc thuộc tính khác.
- **Không thread-safe**: 2 thread truy cập lần đầu cùng lúc có thể khởi tạo 2 lần. (Ngược lại, biến global và `static let` thì Swift đảm bảo khởi tạo lazy **đúng 1 lần, thread-safe** — dùng `dispatch_once` bên dưới.)
- Trong struct, đọc lazy property là **mutating getter** → không dùng được trên `let`:
  ```swift
  struct Lazy { lazy var v = 1 }
  let lz = Lazy()
  // print(lz.v)   // ❌ error: cannot use mutating getter on immutable value: 'lz' is a 'let' constant
  ```

#### 3.3. Property observer: `willSet` / `didSet`
```swift
class StepCounter {
    var steps: Int = 0 {
        willSet { print("Sắp đổi \(steps) → \(newValue)") }
        didSet {
            if steps < 0 { steps = oldValue }   // gán lại trong didSet KHÔNG gọi lại observer
            print("Đã đổi \(oldValue) → \(steps)")
        }
    }
    init() { steps = 10 }   // gán trong init của chính class này: KHÔNG gọi observer
}
let counter = StepCounter()
counter.steps = 20
// Sắp đổi 10 → 20
// Đã đổi 10 → 20
counter.steps = -5
// Sắp đổi 20 → -5
// Đã đổi 20 → 20
```
Ghi nhớ:
- Observer **không chạy** khi gán trong `init` của chính kiểu đó (nhưng **có chạy** khi lớp con gán property của lớp cha sau `super.init`).
- Observer chạy cả khi giá trị mới bằng giá trị cũ.
- Truyền property có observer vào tham số `inout` → khi hàm kết thúc, `willSet`/`didSet` được gọi (do copy-in copy-out).
- Với struct: sửa **bất kỳ** thuộc tính con nào của một property kiểu struct cũng kích hoạt `didSet` của property đó (vì cả giá trị struct được coi là "thay mới"). Đây là cơ chế SwiftUI/`@Published` dựa vào để biết "có gì đó đã đổi".

#### 3.4. `static` và `class`
```swift
struct AppConfig {
    static let baseURL = "https://api.github.com"   // thuộc về KIỂU, không thuộc instance
}
print(AppConfig.baseURL)

class Animal {
    class func kind() -> String { "Animal" }   // class: lớp con override được
    static func fixed() -> String { "fixed" }  // static = "class final": không override được
}
class Dog: Animal {
    override class func kind() -> String { "Dog" }
    // override static func fixed() ...   // ❌ error: cannot override static method
}
print(Dog.kind())   // Dog
```
- `static let` được khởi tạo **lazy và thread-safe** → cách viết singleton chuẩn: `static let shared = Manager()`.
- `static var` (biến toàn cục mutable): ở **Swift 6 language mode** sẽ báo *error: static property '…' is not concurrency-safe because it is nonisolated global shared mutable state*. Tuần 7–8 học cách xử lý; tạm thời tránh `static var`.
- `class var` chỉ được là computed property.

### §4. Initializer

**Quy tắc vàng:** khi init xong, **mọi stored property phải có giá trị** (Optional `var` tự có `nil`).

#### 4.1. Memberwise init của struct
```swift
struct User {
    let id: Int
    var name: String
    var isAdmin: Bool = false
}
let u1 = User(id: 1, name: "An")                  // isAdmin dùng default
let u2 = User(id: 2, name: "Bình", isAdmin: true)
print(u1.isAdmin, u2.isAdmin)   // false true
```
- Nếu bạn viết **init tuỳ chỉnh trong thân struct**, memberwise init **biến mất**. Cách giữ cả hai: viết init tuỳ chỉnh trong **extension**:
  ```swift
  extension User {
      init(guestID: Int) { self.init(id: guestID, name: "Guest") }   // memberwise vẫn còn
  }
  print(User(guestID: 9).name)   // Guest
  ```
- `let` có giá trị mặc định (`let a = 1`) **không** xuất hiện trong memberwise init.
- Memberwise init có access level **thấp nhất** trong các stored property: có một property `private` → memberwise thành `private`. Với `public struct` trong framework/package, memberwise init chỉ là `internal` → phải tự viết `public init`.
- **Class không có memberwise init.** `class C { var x = 1 }` chỉ có `C()`; `C(x: 2)` → *error: argument passed to call that takes no arguments*. Class có property không default mà không viết init → *error: class 'X' has no initializers*.

#### 4.2. Failable init `init?`
```swift
struct Email {
    let value: String
    init?(_ raw: String) {
        guard raw.contains("@") else { return nil }   // thất bại → trả nil
        value = raw.lowercased()
    }
}
print(Email("A@B.com")?.value ?? "nil")   // a@b.com
print(Email("abc")?.value ?? "nil")       // nil
```
Ví dụ trong thư viện chuẩn: `Int("abc")`, `URL(string:)`, `UIImage(named:)`, `init?(rawValue:)`.

#### 4.3. Class: designated vs convenience
| Designated | Convenience |
|---|---|
| Init "chính", khởi tạo **đầy đủ** mọi property của class đó | Init "phụ", từ khoá `convenience` |
| Phải gọi **designated init của lớp cha** (`super.init`) — *delegate up* | Phải gọi **init khác của cùng class** (`self.init`) — *delegate across* |
| Mỗi class có ít nhất 1 | Tuỳ chọn |

```swift
class Vehicle {
    var wheels: Int
    var name: String
    init(wheels: Int, name: String) {     // designated
        self.wheels = wheels
        self.name = name
    }
    convenience init(name: String) {      // convenience → phải gọi self.init(...)
        self.init(wheels: 4, name: name)
    }
    func describe() -> String { "\(name): \(wheels) bánh" }
}
print(Vehicle(name: "Car").describe())    // Car: 4 bánh
```
Convenience mà không gọi `self.init` (ví dụ `convenience init() { self.wheels = 4 }`) → *error: 'self' used before 'self.init' call or assignment to 'self'*.

> Mẹo nhớ của Apple: **designated đi lên (up), convenience đi ngang (across)**.

#### 4.4. Two-phase initialization — câu hỏi xoáy hay gặp
Init của class có kế thừa diễn ra 2 pha:
- **Phase 1** (đi từ lớp con lên lớp cha): mỗi lớp khởi tạo **property của chính nó**, rồi gọi `super.init`. Khi lên tới lớp gốc và mọi property đã có giá trị → object "hợp lệ".
- **Phase 2** (đi từ lớp cha xuống lớp con): mỗi lớp được phép **dùng `self`**: gọi method, sửa property kế thừa, v.v.

```swift
class Bike: Vehicle {
    var hasBell: Bool
    init(hasBell: Bool) {
        // Phase 1: property của lớp con TRƯỚC
        self.hasBell = hasBell
        super.init(wheels: 2, name: "Bike")    // rồi mới đi lên lớp cha
        // Phase 2: giờ mới được dùng self
        self.name = hasBell ? "Bike 🔔" : "Bike"
        print(describe())
    }
    override func describe() -> String {
        super.describe() + (hasBell ? " (có chuông)" : "")
    }
}
let bike = Bike(hasBell: true)    // in: Bike 🔔: 2 bánh (có chuông)
```
Compiler thực thi 4 "safety check". Thử vi phạm từng cái:
```swift
// ❌ gọi super.init trước khi init property của lớp con:
//    init(hasBell: Bool) { super.init(wheels: 2, name: "B"); self.hasBell = hasBell }
//    → error: property 'self.hasBell' not initialized at super.init call
// ❌ sửa property kế thừa trước super.init:
//    init(hasBell: Bool) { self.hasBell = hasBell; self.wheels = 3; super.init(wheels: 2, name: "B") }
//    → error: 'self' used in property access 'wheels' before 'super.init' call
// ❌ gọi method trước super.init:
//    init(hasBell: Bool) { print(describe()); self.hasBell = hasBell; super.init(wheels: 2, name: "B") }
//    → error: 'self' used in method call 'describe' before 'super.init' call
```
**Tại sao Swift bắt buộc thế?** Để không bao giờ đọc một property chưa khởi tạo. Ví dụ ở Objective-C/Java, constructor lớp cha gọi một method bị lớp con override, method đó đọc property của lớp con **chưa được gán** → bug. Swift chặn từ lúc compile: lớp con phải gán xong property của mình *trước khi* lớp cha có cơ hội gọi method ảo.

#### 4.5. Kế thừa init tự động
Mặc định lớp con **không** kế thừa init của lớp cha, trừ 2 luật:
1. Lớp con **không khai báo designated init nào** (và property mới đều có default) → kế thừa **mọi designated init** của cha.
2. Lớp con có (kế thừa hoặc override) **tất cả** designated init của cha → kế thừa luôn **mọi convenience init**.
```swift
class A { var x: Int; init(x: Int) { self.x = x }; convenience init() { self.init(x: 0) } }
class B: A { var y = 1 }      // không viết init nào → luật 1 + luật 2
print(B().x, B(x: 5).x)       // 0 5
```

#### 4.6. `required init`
Bắt **mọi lớp con** phải có init này (để code generic gọi `type.init(...)` an toàn):
```swift
class Base {
    let id: String
    required init(id: String) { self.id = id }
}
class Sub: Base {
    let extra: Int
    init(extra: Int) { self.extra = extra; super.init(id: "x") }
    required init(id: String) { self.extra = 0; super.init(id: id) }   // bắt buộc phải có
}
func make<T: Base>(_ type: T.Type) -> T { type.init(id: "auto") }
print(make(Sub.self).id, make(Sub.self).extra)   // auto 0
```
Gặp hằng ngày ở UIKit: khi bạn viết `init(frame:)` riêng cho `UIView` subclass, Xcode ép thêm `required init?(coder:)` — vì `UIView` khai báo nó là `required` (storyboard/xib dùng init này).

### §5. `mutating`, kế thừa, `override`, `final`

#### 5.1. `mutating`
Method của struct/enum mặc định **không được sửa `self`**. Vì sao? `self` trong method của value type là một **bản `let`**. `mutating` biến `self` thành tham số **`inout`** ngầm → sửa xong được ghi ngược lại.
```swift
struct Counter {
    var value = 0
    mutating func increment() { value += 1 }
    mutating func reset() { self = Counter() }   // gán lại cả self cũng được
}
var cnt = Counter(); cnt.increment(); cnt.increment()
print(cnt.value)              // 2
cnt.reset(); print(cnt.value) // 0

// Thiếu mutating:  func inc() { value += 1 }
//   → error: left side of mutating operator isn't mutable: 'self' is immutable
// Gọi mutating trên let:  let s = Counter(); s.increment()
//   → error: cannot use mutating member on immutable value: 's' is a 'let' constant
```
Class không có `mutating` vì `self` là tham chiếu — sửa object không đổi tham chiếu.

#### 5.2. Kế thừa & `override`
- Chỉ class mới kế thừa được, và chỉ **đơn kế thừa** (1 lớp cha).
- Override method/property/subscript phải có từ khoá `override` (thiếu → *error: overriding declaration requires an 'override' keyword*). Nhờ vậy không bao giờ "vô tình" override.
- Gọi bản của lớp cha qua `super.method()`.
- Override được property (computed, hoặc thêm observer cho stored property của cha), nhưng không đổi kiểu.

#### 5.3. `final`
- `final class X` → không ai kế thừa được (*error: inheritance from a final class 'X'*).
- `final func` / `final var` → lớp con không override được (*error: instance method overrides a 'final' instance method*).
- Lợi ích: (1) **thể hiện ý đồ** thiết kế, (2) **hiệu năng**: compiler biết chắc không có override → gọi trực tiếp (static dispatch), có thể inline.
- Khuyến nghị: class trong app nên mặc định `final`, chỉ bỏ khi thật sự cần kế thừa.

### §6. Stack vs Heap & Copy-on-Write

#### 6.1. Stack và heap
| Stack | Heap |
|---|---|
| Mỗi thread một stack riêng | Dùng chung cho toàn process |
| Cấp phát = dịch con trỏ stack → **cực nhanh** | Phải tìm ô nhớ trống, cần đồng bộ giữa các thread → **chậm hơn** |
| Tự giải phóng khi hàm return | Giải phóng khi refCount về 0 (ARC) |
| Kích thước phải biết lúc compile | Kích thước tuỳ ý, sống lâu tuỳ ý |

- **Class instance** luôn nằm trên heap. Biến kiểu class chỉ là **con trỏ 8 byte**; object trên heap có thêm header ~16 byte (metadata pointer + reference count). Mỗi lần copy tham chiếu → retain/release **atomic** → tốn thêm.
- **Struct** được lưu **inline**: biến local → thường trên stack; là property của class → nằm trong object đó trên heap.

**Struct có luôn ở stack không? KHÔNG.** Struct lên heap khi:
1. Là property của một class instance.
2. Bị **escaping closure capture** (biến phải sống lâu hơn hàm → được "box" lên heap).
3. Được lưu trong existential (`any Protocol`) mà lớn hơn 3 word (24 byte) — tuần 4.
4. Là một case `indirect` của enum.

Và quan trọng hơn: struct **chứa** dữ liệu trên heap rất phổ biến — `Array`, `String`, `Dictionary` đều là struct nhỏ (8–16 byte) trỏ tới một buffer trên heap:
```swift
print(MemoryLayout<Int>.size, MemoryLayout<String>.size, MemoryLayout<[Int]>.size)  // 8 16 8
struct P3 { var x: Double; var y: Double; var z: Double }
print(MemoryLayout<P3>.size)   // 24
class C3 { var x = 0.0 }
print(MemoryLayout<C3>.size)   // 8 — chỉ là một con trỏ tới heap
```
→ Mảng 1 triệu phần tử thì biến `[Int]` vẫn chỉ 8 byte. Vậy "copy array" chẳng lẽ copy 1 triệu phần tử? → Copy-on-write.

#### 6.2. Copy-on-write (CoW)
**Ý tưởng:** khi gán `b = a`, **chưa copy dữ liệu**, cả hai dùng chung buffer. Chỉ khi một bên **ghi** (mutate) **và** buffer đang bị chia sẻ, mới copy thật.
```swift
func address(_ a: [Int]) -> String {
    a.withUnsafeBufferPointer { String(describing: $0.baseAddress!) }
}
var arr1 = [1, 2, 3]
var arr2 = arr1
print(address(arr1) == address(arr2))   // true  — chưa copy, dùng chung buffer
arr2.append(4)
print(address(arr1) == address(arr2))   // false — vừa ghi khi đang chia sẻ → copy
```
Cơ chế bên trong (đơn giản hoá):
```
struct Array { var buffer: BufferClass }   // buffer là CLASS trên heap, có refCount

mutating func append(x):
    if buffer không phải "uniquely referenced" (refCount > 1):
        buffer = buffer.copy()             // O(n) — chỉ xảy ra lần ghi đầu tiên
    buffer.append(x)
```
- Có sẵn CoW: `Array`, `Dictionary`, `Set`, `String`, `Data`.
- **Struct bạn tự viết KHÔNG tự có CoW.** Copy struct tự viết = copy từng stored property (property là Array thì Array đó lại CoW). Với struct nhỏ, copy luôn còn rẻ hơn — không cần CoW.

#### 6.3. Tự cài CoW cho struct bọc class
Khi nào cần? Khi struct của bạn bọc một **class** (để tiết kiệm bộ nhớ, hoặc bọc một API reference) mà vẫn muốn value semantics. Nhớ lại bug ở §1:
```swift
final class Box { var value = 0 }
struct Wrapper { var box = Box() }
var w1 = Wrapper()
var w2 = w1
w2.box.value = 42
print(w1.box.value)   // 42  ← struct nhưng hành xử như reference!
```
Cách sửa — pattern CoW chuẩn:
```swift
final class Storage {
    var values: [String: Int]
    init(values: [String: Int]) { self.values = values }
}

struct Inventory {
    private var storage = Storage(values: [:])   // private: không ai ngoài struct chạm vào class

    var count: Int { storage.values.count }                          // đọc: không copy
    func quantity(of item: String) -> Int { storage.values[item, default: 0] }

    mutating func add(_ item: String, _ n: Int = 1) {                // ghi: kiểm tra trước
        if !isKnownUniquelyReferenced(&storage) {
            print("  → copy storage")
            storage = Storage(values: storage.values)
        }
        storage.values[item, default: 0] += n
    }
}

var inv1 = Inventory()
inv1.add("táo")              // không copy: storage chỉ có 1 owner
var inv2 = inv1              // chưa copy gì cả, chỉ +1 refCount
inv2.add("cam")              // in "  → copy storage"
print(inv1.count, inv2.count)        // 1 2
inv2.add("cam")              // không copy nữa: inv2 đã sở hữu riêng
print(inv2.quantity(of: "cam"))      // 2
```
Ghi nhớ về `isKnownUniquelyReferenced(&obj)`:
- Trả `true` nếu object chỉ có **đúng 1 strong reference**. Nhận `inout` (dấu `&`) để chính lời gọi không tạo thêm reference tạm.
- Chỉ dùng trong **`mutating`** method; mọi method ghi **đều phải** đi qua bước kiểm tra này.
- Class bên trong phải `private`, không lộ ra ngoài — nếu ai đó giữ được tham chiếu tới nó thì CoW vô nghĩa.
- Không tự thread-safe: vẫn là value type, mỗi thread phải có bản riêng.

> **Hiệu năng thực tế** (người phỏng vấn thích hỏi): `array.append` trong vòng lặp trên một array **đang bị chia sẻ** chỉ copy lần đầu. Nhưng lỗi hay gặp là giữ một bản copy "vô tình" (ví dụ `let backup = items` ngay trước vòng lặp sửa `items`) → lần ghi đầu copy toàn bộ O(n).

### §7. Method dispatch — giới thiệu

"Dispatch" = cách chương trình chọn **đoạn code nào** sẽ chạy khi gọi một method.

| Loại | Cách hoạt động | Tốc độ | Khi nào |
|---|---|---|---|
| **Static (direct)** | Địa chỉ hàm biết lúc compile, gọi thẳng; còn **inline** được | Nhanh nhất | Method của struct/enum; `final`, `private`, `static` của class; method trong extension (không `@objc`) |
| **Dynamic — vtable** | Mỗi class có một bảng con trỏ hàm (virtual table). Lúc gọi: lấy class thật của object → tra bảng → nhảy | Nhanh, nhưng không inline được | Method thường của class, có thể bị override |
| **Dynamic — witness table** | Tương tự vtable nhưng cho protocol | | Gọi qua protocol (tuần 3, 4) |
| **Message** | `objc_msgSend` của Objective-C runtime, tra theo selector, có cache | Chậm nhất, linh hoạt nhất (swizzling, KVO) | `@objc dynamic`, class kế thừa `NSObject` gọi từ ObjC |

```swift
import Foundation

class Animal {
    func speak() -> String { "..." }          // vtable
    final func id() -> String { "animal" }    // static: không thể bị override
}
class Cat: Animal {
    override func speak() -> String { "Meo" }
}
let pet: Animal = Cat()      // kiểu tĩnh Animal, kiểu thật Cat
print(pet.speak())           // Meo — tra vtable lúc runtime

struct Calculator { func add(_ a: Int, _ b: Int) -> Int { a + b } }   // static

class Legacy: NSObject {
    @objc dynamic func hello() -> String { "hi" }   // message dispatch
}
print(Legacy().hello())      // hi
```
Compiler còn tự "de-virtualize": với **Whole Module Optimization** (mặc định ở Release), method `internal` không bị override trong module được suy ra là `final` → static dispatch. Đây là lý do `final` và `private` giúp hiệu năng.

### §8. Chọn struct hay class?

Theo bài *Choosing Between Structures and Classes* của Apple:
1. **Mặc định dùng struct.**
2. Dùng **class** khi cần:
   - **Identity**: nhiều nơi cần trỏ tới *cùng một* thực thể và thấy thay đổi của nhau (ví dụ `URLSession`, cache dùng chung, một `Player` đang phát nhạc, ViewModel trong UIKit được nhiều chỗ giữ).
   - **Interop Objective-C** / kế thừa từ framework (`UIView`, `UIViewController`, `NSObject`).
   - Quản lý **tài nguyên có vòng đời** cần `deinit` (file handle, socket, observer).
3. Muốn "chia sẻ hành vi" → dùng **protocol** + struct, thay vì kế thừa class (tuần 3: POP).

| Ví dụ | Chọn | Lý do |
|---|---|---|
| Model từ API: `Repository`, `User` | struct | Dữ liệu thuần, so sánh bằng giá trị, an toàn đa luồng |
| `NetworkClient`, `ImageCache` | class (thường `final`) | Một instance dùng chung, có state nội bộ |
| SwiftUI `View` | struct | Mô tả UI rẻ, tạo lại liên tục |
| UIKit `UIViewController` subclass | class | Framework yêu cầu kế thừa |
| Trạng thái màn hình `idle/loading/loaded/failed` | enum | Các trạng thái loại trừ nhau |

---

## ⚠️ Bẫy hay gặp

1. **"Struct luôn nằm trên stack, class nằm trên heap"** — sai một nửa. Struct là property của class, bị escaping closure capture, hay trong existential lớn đều có thể lên heap. Câu đúng: *value type được lưu inline, nơi lưu phụ thuộc ngữ cảnh*.
2. **"Struct là value type nên copy tốn kém"** — sai. Copy struct nhỏ rất rẻ; Array/String có CoW.
3. **"Mọi struct đều có copy-on-write"** — sai. Chỉ các kiểu được cài đặt CoW (Array, String, Dictionary, Set, Data…). Struct của bạn không tự có.
4. **Struct chứa class** không còn value semantics thật.
5. **Property observer không chạy trong `init`** của chính kiểu đó.
6. **Viết init trong thân struct làm mất memberwise init** → viết trong extension.
7. **`lazy` không thread-safe**, trong khi `static let` thì có.
8. **Enum không thể vừa có raw value vừa có associated value.**
9. **Gọi method/đọc `self` trước `super.init`** → lỗi compile (two-phase init).
10. **Quên `required init?(coder:)`** khi subclass `UIView`/`UIViewController` với init riêng.
11. **`static func` không override được**, `class func` thì được.
12. **Gán struct vào `let` rồi gọi `mutating`** → lỗi; `let` struct có `lazy var` cũng không đọc được.

---

## 🛠 Bài tập

> Tạo file trong `practice/week02/` và chạy bằng `swift tên_file.swift`. **Không xem lời giải trước khi tự làm ít nhất 20 phút.**

**Bài 1 — Dự đoán output (T2)**
Không chạy, viết ra giấy 4 số được in. Sau đó chạy để kiểm tra và giải thích từng số.
```swift
struct Score { var value: Int }
class Player {
    var name: String; var score: Score
    init(name: String, score: Score) { self.name = name; self.score = score }
}
var s1 = Score(value: 10)
let p1 = Player(name: "An", score: s1)
s1.value = 20
let p2 = p1
p2.score.value += 5
var s2 = p1.score
s2.value = 0
print(s1.value, p1.score.value, p2.score.value, s2.value)
```
<details><summary>Lời giải</summary>

`20 15 15 0`
- `s1` được **copy** vào `p1.score` lúc init → sửa `s1` thành 20 không ảnh hưởng `p1.score` (vẫn 10).
- `p2 = p1` copy **tham chiếu** → `p2.score.value += 5` sửa chính object đó → `p1.score.value == p2.score.value == 15`.
- `s2 = p1.score` lấy ra **bản sao** của struct → sửa `s2` không ảnh hưởng object.
</details>

**Bài 2 — Enum associated value (T3)**
Tạo `enum PaymentMethod` với `cash`, `card(last4: String, isCredit: Bool)`, `wallet(provider: String)`. Viết:
- `func fee(for amount: Double) -> Double`: cash 0; credit card 2%; debit card 1%; ví "MoMo" 1.000đ; ví khác 2.000đ.
- `var label: String`: "Tiền mặt", "Credit ••4242", "Ví MoMo"…
<details><summary>Lời giải</summary>

```swift
enum PaymentMethod {
    case cash
    case card(last4: String, isCredit: Bool)
    case wallet(provider: String)

    func fee(for amount: Double) -> Double {
        switch self {
        case .cash:                                          return 0
        case .card(_, isCredit: true):                       return amount * 0.02
        case .card:                                          return amount * 0.01
        case .wallet(let provider) where provider == "MoMo": return 1_000
        case .wallet:                                        return 2_000
        }
    }
    var label: String {
        switch self {
        case .cash:                     return "Tiền mặt"
        case let .card(last4, isCredit): return "\(isCredit ? "Credit" : "Debit") ••\(last4)"
        case .wallet(let p):            return "Ví \(p)"
        }
    }
}
let methods: [PaymentMethod] = [.cash, .card(last4: "4242", isCredit: true),
                                .card(last4: "1111", isCredit: false),
                                .wallet(provider: "MoMo"), .wallet(provider: "ZaloPay")]
for m in methods { print(m.label, m.fee(for: 100_000)) }
// Tiền mặt 0.0
// Credit ••4242 2000.0
// Debit ••1111 1000.0
// Ví MoMo 1000.0
// Ví ZaloPay 2000.0
```
Chú ý: `.card(_, isCredit: true)` match giá trị cụ thể trong pattern; `case .card:` (không bind gì) bắt mọi card còn lại. Thứ tự case quan trọng — case cụ thể phải đứng trước.
</details>

**Bài 3 — `indirect` enum (T3)**
Cài `indirect enum LinkedList { case empty; case node(Int, LinkedList) }` với `count`, `sum`, `prepending(_:)`, `description` dạng `"1 -> 2 -> 3 -> nil"`.
<details><summary>Lời giải</summary>

```swift
indirect enum LinkedList {
    case empty
    case node(Int, LinkedList)

    var count: Int {
        switch self { case .empty: return 0; case .node(_, let next): return 1 + next.count }
    }
    var sum: Int {
        switch self { case .empty: return 0; case let .node(v, next): return v + next.sum }
    }
    func prepending(_ v: Int) -> LinkedList { .node(v, self) }
    var description: String {
        switch self {
        case .empty: return "nil"
        case let .node(v, next): return "\(v) -> \(next.description)"
        }
    }
}
let list = LinkedList.empty.prepending(3).prepending(2).prepending(1)
print(list.description, list.count, list.sum)   // 1 -> 2 -> 3 -> nil 3 6
```
Vì là value type bất biến, `prepending` trả list mới nhưng **dùng chung** phần đuôi (các box trên heap) — đây là "persistent data structure".
</details>

**Bài 4 — Property (T4)**
`struct Thermostat` có stored `celsius` luôn bị kẹp trong [5, 35] (dùng `didSet`) và computed `fahrenheit` có cả get/set. Kiểm tra: đặt `fahrenheit = 212` thì `celsius` bằng bao nhiêu?
<details><summary>Lời giải</summary>

```swift
struct Thermostat {
    var celsius: Double {
        didSet {
            if celsius < 5 { celsius = 5 }
            if celsius > 35 { celsius = 35 }
        }
    }
    var fahrenheit: Double {
        get { celsius * 9 / 5 + 32 }
        set { celsius = (newValue - 32) * 5 / 9 }
    }
}
var th = Thermostat(celsius: 20)
th.fahrenheit = 212
print(th.celsius, th.fahrenheit)   // 35.0 95.0
th.celsius = -10
print(th.celsius)                  // 5.0
```
`fahrenheit = 212` → setter gán `celsius = 100` → `didSet` kẹp về 35. Lưu ý: nếu viết `Thermostat(celsius: 100)` thì **không** bị kẹp vì observer không chạy trong init! Muốn kẹp cả lúc init thì viết init riêng.
</details>

**Bài 5 — Initializer (T5)**
- `class BankAccount`: `let owner`, `private(set) var balance`. Failable designated `init?(owner:initialDeposit:)` (owner rỗng hoặc tiền âm → nil). Convenience `init?(owner:)` với số dư 0. `deposit`, `withdraw(_:) -> Bool`.
- `final class SavingsAccount: BankAccount` thêm `let rate` (rate ≤ 0 → nil), override `withdraw` không cho rút quá 10 triệu một lần, thêm `applyInterest()`.
<details><summary>Lời giải</summary>

```swift
class BankAccount {
    let owner: String
    private(set) var balance: Double

    init?(owner: String, initialDeposit: Double) {
        guard !owner.isEmpty, initialDeposit >= 0 else { return nil }
        self.owner = owner
        self.balance = initialDeposit
    }
    convenience init?(owner: String) { self.init(owner: owner, initialDeposit: 0) }

    func deposit(_ amount: Double) { balance += amount }
    func withdraw(_ amount: Double) -> Bool {
        guard amount <= balance else { return false }
        balance -= amount
        return true
    }
}

final class SavingsAccount: BankAccount {
    let rate: Double
    init?(owner: String, initialDeposit: Double, rate: Double) {
        guard rate > 0 else { return nil }
        self.rate = rate                                        // phase 1: property lớp con
        super.init(owner: owner, initialDeposit: initialDeposit) // rồi lên lớp cha (cũng failable)
    }
    override func withdraw(_ amount: Double) -> Bool {
        guard amount <= 10_000_000 else { return false }
        return super.withdraw(amount)
    }
    func applyInterest() { deposit(balance * rate) }
}

print(BankAccount(owner: "", initialDeposit: 10) == nil)                 // true
let sa = SavingsAccount(owner: "An", initialDeposit: 1_000, rate: 0.1)!
sa.applyInterest()
print(sa.balance, sa.withdraw(5_000), sa.withdraw(100))                  // 1100.0 false true
print(SavingsAccount(owner: "", initialDeposit: 1, rate: 0.1) == nil)    // true
```
- `failable` init gọi `super.init` failable → nếu cha trả nil thì cả init con trả nil.
- `SavingsAccount` có designated init riêng → **không** kế thừa `init?(owner:)` của cha (luật 1, 2 ở §4.5).
- `private(set)`: đọc được từ ngoài, chỉ ghi được bên trong.
</details>

**Bài 6 — Kế thừa, override, final (T6)**
Tạo `class Shape` có `area()` (mặc định 0) và `final func summary()` trả `"<TênLớp> area=<area>"`. Tạo `Circle(r:)` (làm tròn 2 chữ số) và `Square(side:)`. Duyệt `[Shape]` và in summary. Giải thích dispatch của `area()` và `summary()`.
<details><summary>Lời giải</summary>

```swift
class Shape {
    func area() -> Double { 0 }
    final func summary() -> String { "\(type(of: self)) area=\(area())" }
}
class Circle: Shape {
    let r: Double
    init(r: Double) { self.r = r }
    override func area() -> Double { (Double.pi * r * r * 100).rounded() / 100 }
}
class Square: Shape {
    let side: Double
    init(side: Double) { self.side = side }
    override func area() -> Double { side * side }
}
let shapes: [Shape] = [Circle(r: 1), Square(side: 2)]
for s in shapes { print(s.summary()) }
// Circle area=3.14
// Square area=4.0
```
`summary()` là `final` → static dispatch. Bên trong nó gọi `area()` → vtable dispatch → chạy bản của `Circle`/`Square`. `type(of: self)` trả kiểu **runtime**, không phải kiểu tĩnh `Shape`.
</details>

**Bài 7 — Quan sát CoW (T7)**
a) Dùng hàm `address(_:)` ở §6.2, chứng minh: (1) `var b = a` chưa copy; (2) **đọc** `b[0]` vẫn chưa copy; (3) `b[0] = 9` thì copy; (4) sau đó `b.append(1)` không đổi `a`.
b) Sửa `struct Wrapper { var box = Box() }` ở §6.3 để có value semantics bằng CoW: `w2.value = 42` không ảnh hưởng `w1`. Gợi ý: `box` thành `private`, thêm computed `value` có `get` / `set` (setter của computed property trên struct là mutating ngầm định).
<details><summary>Gợi ý lời giải b)</summary>

```swift
final class Box { var value = 0; init(_ v: Int = 0) { value = v } }
struct Wrapper {
    private var box = Box()
    var value: Int {
        get { box.value }
        set {
            if !isKnownUniquelyReferenced(&box) { box = Box(box.value) }
            box.value = newValue
        }
    }
}
var w1 = Wrapper()
var w2 = w1
w2.value = 42
print(w1.value, w2.value)   // 0 42
```
</details>

**🏋️ Bài tập lớn (T7) — `Playlist` có copy-on-write**
Viết `playlist.swift`:
- `struct Song: Equatable { let title: String; let duration: Int }`
- `struct Playlist` với `var name: String` (stored bình thường) và danh sách bài hát lưu trong một `private final class Storage`.
- API: `songs` (đọc), `totalDuration`, `add(_:)`, `remove(at:)`, `sharesStorage(with:) -> Bool` (để test).
- Mọi thao tác ghi phải đi qua `makeUnique()` dùng `isKnownUniquelyReferenced`. In `"  → copy storage"` mỗi khi copy thật.
- Viết đoạn test chứng minh: copy playlist không copy storage; đổi `name` không copy storage; `add` lần đầu trên bản copy thì copy đúng 1 lần; lần sau không copy nữa.
- Câu hỏi tự trả lời: tại sao `name` không cần CoW?

<details><summary>Lời giải tham khảo</summary>

```swift
struct Song: Equatable { let title: String; let duration: Int }

struct Playlist {
    private final class Storage {
        var songs: [Song]
        init(songs: [Song]) { self.songs = songs }
    }
    private var storage: Storage
    var name: String

    init(name: String, songs: [Song] = []) {
        self.name = name
        self.storage = Storage(songs: songs)
    }

    var songs: [Song] { storage.songs }
    var totalDuration: Int { storage.songs.reduce(0) { $0 + $1.duration } }

    private mutating func makeUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            print("  → copy storage")
            storage = Storage(songs: storage.songs)
        }
    }
    mutating func add(_ song: Song) { makeUnique(); storage.songs.append(song) }
    mutating func remove(at i: Int) { makeUnique(); storage.songs.remove(at: i) }

    func sharesStorage(with other: Playlist) -> Bool { storage === other.storage }
}

var chill = Playlist(name: "Chill", songs: [Song(title: "A", duration: 200)])
var copy = chill
print(chill.sharesStorage(with: copy))      // true
copy.name = "Chill (copy)"                  // đổi name KHÔNG chạm storage
print(chill.sharesStorage(with: copy))      // true
copy.add(Song(title: "B", duration: 180))   // in "  → copy storage"
print(chill.sharesStorage(with: copy))      // false
print(chill.songs.count, copy.songs.count)  // 1 2
copy.add(Song(title: "C", duration: 100))   // không copy nữa
print(copy.totalDuration)                   // 480
chill.add(Song(title: "D", duration: 1))    // không copy: chill giờ cũng là owner duy nhất
print(chill.songs.map(\.title))             // ["A", "D"]
```
`name` là `String` — bản thân đã là value type có CoW, nên copy struct `Playlist` tự copy đúng ngữ nghĩa. Chỉ phần bọc **class** mới cần ta tự làm CoW. (Thực tế `[Song]` cũng đã có CoW sẵn — bài này để luyện cơ chế; trong app thật bạn chỉ cần `var songs: [Song]`.)
</details>

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Struct và class khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Khác biệt cốt lõi là **value vs reference semantics**: gán struct tạo bản sao độc lập, gán class chỉ copy tham chiếu tới cùng một object. Từ đó kéo theo: class có kế thừa, `deinit`, identity (`===`), được quản lý bằng ARC và cấp phát trên heap; struct có memberwise init, cần `mutating` để sửa `self`, thường lưu inline (stack) và method mặc định static dispatch. Struct an toàn hơn khi đa luồng vì không chia sẻ trạng thái. Apple khuyến nghị mặc định dùng struct, chỉ dùng class khi cần identity, kế thừa hoặc interop Objective-C.
</details>

**Q2 ⭐ Value semantics là gì? Một struct có chứa property kiểu class thì còn value semantics không?**
<details><summary>Đáp án gợi ý</summary>

Value semantics: sửa một bản không bao giờ ảnh hưởng bản khác. Struct chứa class mutable thì **không** còn value semantics thật: copy struct chỉ copy tham chiếu, hai bản cùng sửa một object. Muốn giữ value semantics phải đổi property đó thành value type, làm class bất biến (chỉ `let`), hoặc cài copy-on-write với `isKnownUniquelyReferenced`.
</details>

**Q3 ⭐ Copy-on-write là gì? Hoạt động thế nào? Struct nào cũng có CoW à?**
<details><summary>Đáp án gợi ý</summary>

CoW là tối ưu: khi copy, các bản dùng chung một buffer trên heap; chỉ khi một bản **ghi** và buffer đang bị chia sẻ (refCount > 1) thì mới copy thật. Bên trong, Array/Dictionary/Set/String là struct nhỏ chứa tham chiếu tới một class buffer; mỗi method `mutating` kiểm tra `isKnownUniquelyReferenced` trước khi ghi. Không phải struct nào cũng có CoW — chỉ các kiểu được cài đặt như vậy (collection chuẩn, String, Data). Struct tự viết thì copy toàn bộ stored property (vốn rẻ nếu struct nhỏ).
</details>

**Q4 Làm sao tự cài copy-on-write cho một kiểu của mình?**
<details><summary>Đáp án gợi ý</summary>

Bọc dữ liệu trong một `private final class Storage`; struct giữ `private var storage`. Các getter đọc thẳng từ storage. Mọi method ghi là `mutating` và trước khi ghi gọi `if !isKnownUniquelyReferenced(&storage) { storage = Storage(copy of data) }`. Class phải private để không ai giữ được tham chiếu bên ngoài. Lưu ý hàm này không đồng bộ giữa các thread — value type vẫn phải được dùng theo kiểu mỗi thread một bản.
</details>

**Q5 ⭐ Khi nào chọn struct, khi nào chọn class?**
<details><summary>Đáp án gợi ý</summary>

Mặc định struct: model dữ liệu, DTO, state, SwiftUI view — dễ suy luận, thread-safe hơn, không retain cycle. Chọn class khi cần **identity/shared mutable state** (cache, session, service dùng chung), cần **kế thừa** từ framework (UIViewController, UIView, NSObject), cần `deinit` để giải phóng tài nguyên, hoặc interop Objective-C. Muốn tái sử dụng hành vi thì ưu tiên protocol + extension hơn kế thừa class.
</details>

**Q6 ⭐ Struct có luôn nằm trên stack không?**
<details><summary>Đáp án gợi ý</summary>

Không. Value type được lưu **inline** ở nơi chứa nó: biến local thì thường trên stack, nhưng nếu là property của class thì nằm trong object trên heap; nếu bị escaping closure capture thì được box lên heap; trong existential `any P` mà lớn hơn 3 word cũng bị box lên heap; case `indirect` cũng trên heap. Ngoài ra struct như Array/String tuy nhỏ nhưng trỏ tới buffer trên heap. Stack nhanh vì cấp phát chỉ là dịch con trỏ và không cần đồng bộ; heap cần tìm vùng trống, đồng bộ giữa thread và đếm tham chiếu atomic.
</details>

**Q7 ⭐ Raw value và associated value của enum khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Raw value là hằng số cố định gắn với mỗi case, cùng một kiểu (String/Int/…), biết lúc compile, có `init?(rawValue:)` — hợp để map với API. Associated value là dữ liệu đi kèm **từng instance**, mỗi case có thể khác kiểu, chỉ biết lúc runtime, lấy ra bằng pattern matching. Một enum không thể dùng cả hai cùng lúc. Associated value rất hợp để mô hình hoá state như `loading/loaded(data)/failed(error)`.
</details>

**Q8 `indirect` dùng để làm gì?**
<details><summary>Đáp án gợi ý</summary>

Cho phép enum đệ quy (case chứa chính kiểu enum đó, như cây biểu thức, linked list). Value type phải có kích thước cố định lúc compile, nên compiler cần `indirect` để lưu associated value vào một box trên heap, enum chỉ giữ con trỏ. Có thể đánh dấu cả enum hoặc từng case.
</details>

**Q9 Stored, computed và lazy property khác nhau thế nào? `lazy` có thread-safe không?**
<details><summary>Đáp án gợi ý</summary>

Stored chiếm bộ nhớ trong instance. Computed không lưu gì, tính lại mỗi lần truy cập qua getter/setter. `lazy` là stored nhưng chỉ khởi tạo khi truy cập lần đầu, phải là `var`, initializer được phép dùng `self` — hợp cho đối tượng tốn kém hoặc phụ thuộc property khác. `lazy` **không** thread-safe: hai thread truy cập lần đầu cùng lúc có thể khởi tạo hai lần. Ngược lại `static let` và biến global được khởi tạo lazy và đảm bảo đúng một lần, thread-safe.
</details>

**Q10 Property observer `willSet`/`didSet` khi nào không được gọi?**
<details><summary>Đáp án gợi ý</summary>

Không được gọi khi gán trong `init` của chính kiểu khai báo property (nhưng có gọi khi lớp con gán property của cha sau `super.init`), và không gọi lại khi gán chính property đó bên trong `didSet` của nó. Observer vẫn chạy khi giá trị mới bằng giá trị cũ, và chạy khi property được truyền qua `inout`. Với property kiểu struct, sửa bất kỳ thuộc tính con nào cũng kích hoạt `didSet`.
</details>

**Q11 ⭐ Designated init và convenience init khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Designated init khởi tạo đầy đủ mọi property của class và phải gọi designated init của lớp cha (delegate up). Convenience init là init phụ, phải gọi một init khác của cùng class (delegate across) và cuối cùng phải dẫn tới một designated init. Lớp con chỉ tự kế thừa init khi: không định nghĩa designated init nào (kế thừa hết designated), hoặc đã cung cấp tất cả designated init của cha (kế thừa thêm convenience).
</details>

**Q12 ⭐ Two-phase initialization là gì, tại sao cần?**
<details><summary>Đáp án gợi ý</summary>

Phase 1: từ lớp con lên lớp cha, mỗi lớp gán giá trị cho property của chính nó rồi gọi `super.init`; trong pha này không được dùng `self` (gọi method, đọc property). Phase 2: từ lớp cha xuống, object đã hợp lệ nên mỗi lớp được tuỳ chỉnh thêm, gọi method. Mục đích: đảm bảo không bao giờ truy cập property chưa khởi tạo — ví dụ lớp cha gọi một method bị lớp con override, method đó đọc property của lớp con chưa gán. Compiler kiểm tra lúc compile.
</details>

**Q13 Failable init và `required init` dùng khi nào? Vì sao subclass `UIView` hay phải viết `required init?(coder:)`?**
<details><summary>Đáp án gợi ý</summary>

Failable `init?` trả nil khi input không hợp lệ (như `Int("abc")`, `URL(string:)`, `init?(rawValue:)`). `required` bắt mọi lớp con phải cài init đó, để code dựa trên metatype (`type.init(...)`) gọi an toàn. `UIView` khai báo `init?(coder:)` là `required` vì storyboard/xib dùng nó để tạo view; khi lớp con tự viết designated init khác thì mất quyền kế thừa tự động, nên phải tự viết lại `required init?(coder:)` (thường là `fatalError` nếu không dùng storyboard).
</details>

**Q14 `mutating` là gì? Vì sao class không cần?**
<details><summary>Đáp án gợi ý</summary>

Trong method của struct/enum, `self` là bất biến. `mutating` biến `self` thành tham số `inout` ngầm, nên method được sửa property hoặc gán lại cả `self`; khi trả về, giá trị mới được ghi vào biến gốc. Vì thế không gọi được `mutating` trên `let`. Class không cần vì `self` là tham chiếu: sửa object không làm thay đổi tham chiếu.
</details>

**Q15 `static` và `class` (cho method/property kiểu) khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Cả hai khai báo thành viên của **kiểu**. `class` chỉ dùng trong class và cho phép lớp con override; `static` không override được (tương đương `class final`) và dùng được cho cả struct/enum/protocol. `class var` chỉ được là computed. `static let` hay dùng cho hằng và singleton vì được khởi tạo lazy và thread-safe.
</details>

**Q16 ⭐ Từ khoá `final` có tác dụng gì?**
<details><summary>Đáp án gợi ý</summary>

Ngăn kế thừa class hoặc override method/property. Hai lợi ích: thể hiện ý đồ thiết kế (class không được thiết kế để kế thừa) và hiệu năng — compiler biết chắc không có override nên dùng static dispatch thay vì tra vtable, còn có thể inline. `private` và Whole Module Optimization cũng giúp compiler tự suy ra `final`.
</details>

**Q17 ⭐ Swift có những loại method dispatch nào?**
<details><summary>Đáp án gợi ý</summary>

Ba loại: **static/direct** — địa chỉ biết lúc compile, nhanh nhất, inline được (struct/enum, `final`, `private`, `static`, extension); **table dispatch** — vtable cho method của class có thể override, witness table cho lời gọi qua protocol; **message dispatch** — `objc_msgSend` của Objective-C runtime cho `@objc dynamic`, chậm nhất nhưng hỗ trợ KVO, swizzling. Chọn static khi có thể vì nhanh và cho phép tối ưu.
</details>

**Q18 Vì sao struct đôi khi "mất" memberwise init? Có giữ lại được không?**
<details><summary>Đáp án gợi ý</summary>

Khi bạn định nghĩa bất kỳ init nào trong thân struct, compiler không sinh memberwise init nữa (để tránh bỏ qua logic khởi tạo bạn viết). Giữ lại bằng cách khai báo init tuỳ chỉnh trong `extension`. Ngoài ra memberwise init có access level thấp nhất trong các stored property (có property `private` thì nó thành `private`), và với `public struct` nó chỉ là `internal` → module khác muốn dùng thì phải tự viết `public init`.
</details>

**Q19 Vì sao UIKit dùng class cho view còn SwiftUI dùng struct?**
<details><summary>Đáp án gợi ý</summary>

UIKit view là object sống lâu, có identity, state nội bộ, nằm trong cây view, kế thừa sâu từ `UIResponder`/`NSObject` để interop Objective-C — nên phải là class. SwiftUI `View` chỉ là **mô tả** UI tại một thời điểm: rẻ, bất biến, được tạo lại liên tục mỗi khi state đổi; framework so sánh mô tả mới với cũ để cập nhật phần thật bên dưới. Struct cho phép tạo/huỷ cực rẻ (không cấp phát heap, không ARC) và không có trạng thái chia sẻ ngầm.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Làm xong Bài tập lớn `playlist.swift`, giải thích được từng dòng `makeUnique()`
- [ ] Trả lời được ≥ 15/19 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Struct khác class thế nào, khi nào dùng cái nào"
- [ ] Ghi âm 2 phút: "Copy-on-write hoạt động thế nào, struct có luôn ở stack không"
- [ ] Viết lại được 4 safety check của two-phase init bằng lời của mình
- [ ] Làm xong 6 bài DSA tuần 2
- [ ] Side project Milestone 2 xong (`struct Repository`, `struct Owner`, mock data, commit)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — Structures and Classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)
- [The Swift Programming Language — Enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/)
- [The Swift Programming Language — Properties](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/)
- [The Swift Programming Language — Initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/) (đọc kỹ phần *Class Inheritance and Initialization*)
- [The Swift Programming Language — Inheritance](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/inheritance/)
- Apple Developer — [Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)
- Apple Developer — [`isKnownUniquelyReferenced(_:)`](https://developer.apple.com/documentation/swift/isknownuniquelyreferenced(_:)-98zpp)
- WWDC 2016 — *Understanding Swift Performance* (bắt buộc xem: stack/heap, ARC, dispatch, existential)
- WWDC 2015 — *Building Better Apps with Value Types in Swift*
- Swift repo — [docs/OptimizationTips.rst](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst) (phần *Advice: Use copy-on-write semantics for large values* và *Reducing Dynamic Dispatch*)
