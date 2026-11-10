# Tuần 4 — Generics · `some`/`any` · Error handling · Property wrapper · Access control · Codable (30/11 – 06/12/2026)

> **Giai đoạn 1 · Swift Core** · Thời lượng: ~15 giờ · **Kết thúc tuần: Milestone M1**
> Tuần cuối của Swift Core và là tuần "nặng não" nhất. Generics và `some`/`any` là chỗ ứng viên 2 năm hay **dùng được nhưng không giải thích được**. Codable thì gần như chắc chắn có trong bài live-coding (parse JSON từ API). Cuối tuần có bài tự kiểm tra M1 — làm nghiêm túc.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Viết generic function, generic type, có constraint và `where` clause
- [ ] Viết protocol có `associatedtype` và giải thích vì sao nó khác protocol thường
- [ ] Giải thích **`some` vs `any`**: khác nhau ở đâu, chi phí của existential (boxing, dynamic dispatch), khi nào dùng cái nào
- [ ] Tự cài **type erasure** (`AnyXxx`) bằng closure như `AnyPublisher`
- [ ] Dùng thành thạo `throws`, `do/catch` với pattern, `try?`/`try!`, `rethrows`, `Result`, **typed throws**, `defer`
- [ ] Tự viết một **property wrapper** có `projectedValue`; hiểu result builder ở mức đọc được
- [ ] Thuộc 6 mức **access control** và khác biệt `open` vs `public`
- [ ] Decode JSON thực tế: `CodingKeys`, snake_case, nested, custom `init(from:)`, ngày tháng, đọc hiểu `DecodingError`
- [ ] Làm xong 6 bài DSA tuần 4 — Sliding Window (xem [dsa-plan](../tracks/dsa-plan.md))
- [ ] **Qua bài tự kiểm tra Milestone M1** (cuối file)

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 30/11** | 2h | §1 **Generics** + §2 `associatedtype`. Bài tập 1, 2 · DSA: Best Time to Buy and Sell Stock | `day1.swift` |
| **T3 01/12** | 2h | §3 **`some` vs `any`** (đọc 2 lần) + §4 Type erasure. Bài tập 3, 4 · DSA: Longest Substring Without Repeating Characters | `day2.swift` |
| **T4 02/12** | 2h | §5 **Error handling** + `defer`. Bài tập 5 · DSA: Longest Repeating Character Replacement | `day3.swift` |
| **T5 03/12** | 2h | §6 Property wrapper + §7 Result builder + §8 Access control. Bài tập 6 · DSA: Permutation in String | `day4.swift` |
| **T6 04/12** | 2h | §9 **Codable** (rất quan trọng cho live-coding). Bài tập 7 · DSA: Minimum Size Subarray Sum | `day5.swift` |
| **T7 05/12** | 4h | Bài tập lớn (mini `APIClient` generic). Side project: Milestone 4 (decode JSON từ file trong bundle, error enum, `APIClient` generic skeleton). DSA (tuỳ sức): Minimum Window Substring | `apiclient.swift`, commit side project |
| **CN 06/12** | 2,5h | 🏁 **Bài tự kiểm tra M1** (60') → tự trả lời 20 câu phỏng vấn → tick checklist. Nếu M1 chưa đạt: dùng T2–T3 tuần sau để bù trước khi vào tuần 5 | Kết quả M1 ghi vào notes |

---

## 📚 Kiến thức cốt lõi

### §1. Generics

**Vấn đề:** viết hàm đổi chỗ 2 `Int`, rồi 2 `String`, rồi 2 `Double`… → lặp code. Dùng `Any` thì mất type safety. **Generics** cho phép viết một lần, dùng cho mọi kiểu, mà compiler vẫn kiểm tra kiểu.

```swift
func swapTwo<T>(_ a: inout T, _ b: inout T) {   // T là "type parameter" — placeholder cho một kiểu
    let tmp = a; a = b; b = tmp
}
var x = 1, y = 2
swapTwo(&x, &y); print(x, y)        // 2 1
var s1 = "a", s2 = "b"
swapTwo(&s1, &s2); print(s1, s2)    // b a
// swapTwo(&x, &s1)  // ❌ error: cannot convert value of type 'String' to expected argument type 'Int'
//                   //    — cả hai tham số phải CÙNG kiểu T
```

#### 1.1. Constraint — giới hạn kiểu được dùng
Không có constraint, compiler không biết `T` làm được gì:
```swift
// func same<T>(_ a: T, _ b: T) -> Bool { a == b }
// ❌ error: binary operator '==' cannot be applied to two 'T' operands

func firstIndex<T: Equatable>(of target: T, in items: [T]) -> Int? {   // T phải Equatable
    for (i, item) in items.enumerated() where item == target { return i }
    return nil
}
print(firstIndex(of: "c", in: ["a", "b", "c"]) ?? -1)   // 2

func largest<T: Comparable>(_ items: [T]) -> T? {
    guard var best = items.first else { return nil }
    for x in items.dropFirst() where x > best { best = x }
    return best
}
print(largest([3, 9, 2]) ?? 0, largest(["b", "z", "a"]) ?? "")   // 9 z
```

#### 1.2. Generic type
```swift
struct Stack<Element> {
    private var items: [Element] = []
    var isEmpty: Bool { items.isEmpty }
    var top: Element? { items.last }
    mutating func push(_ x: Element) { items.append(x) }
    @discardableResult mutating func pop() -> Element? { items.popLast() }
}
var st = Stack<Int>()
st.push(1); st.push(2)
print(st.pop() ?? -1, st.top ?? -1)   // 2 1
var words = Stack<String>()           // Stack<String> và Stack<Int> là HAI KIỂU KHÁC NHAU
words.push("hi")
```
`Array<Element>`, `Dictionary<Key, Value>`, `Optional<Wrapped>`, `Result<Success, Failure>` đều là generic type.

#### 1.3. `where` clause & conditional extension
```swift
extension Stack where Element: Numeric {          // chỉ Stack chứa số mới có sum()
    func sum() -> Element { items.reduce(0, +) }
}
print(st.sum())   // 1

extension Stack: CustomStringConvertible where Element: CustomStringConvertible {   // conditional conformance
    var description: String { items.map(\.description).joined(separator: " | ") }
}
print(words)      // hi

func allEqual<C: Collection>(_ c: C) -> Bool where C.Element: Equatable {
    guard let first = c.first else { return true }
    return c.allSatisfy { $0 == first }
}
print(allEqual([1, 1, 1]), allEqual("aab"))   // true false

func merge<A: Sequence, B: Sequence>(_ a: A, _ b: B) -> [A.Element]
where A.Element == B.Element, A.Element: Comparable {      // ràng buộc giữa 2 type parameter
    (Array(a) + Array(b)).sorted()
}
print(merge([3, 1], Set([2])))   // [1, 2, 3]
```
Conditional conformance là lý do `[Int]` là `Equatable` còn `[() -> Void]` thì không: `extension Array: Equatable where Element: Equatable`.

#### 1.4. Generics chạy thế nào? (hỏi xoáy)
- **Specialization**: khi compiler thấy kiểu cụ thể (ví dụ `largest([3, 9, 2])` với `Int`) và thấy được thân hàm (cùng module, hoặc `@inlinable`), nó **sinh riêng một bản** `largest<Int>` → nhanh như viết tay, static dispatch.
- Khi không specialize được, Swift dùng một bản chung, truyền ẩn **type metadata** và **witness table** của `T` vào hàm → gọi method qua bảng (chậm hơn chút nhưng vẫn không box giá trị).
- Khác Java: Swift **không** xoá kiểu (no type erasure lúc runtime) — `T` vẫn được biết ở runtime.

### §2. `associatedtype` — generic cho protocol

Protocol không dùng được cú pháp `<T>` như struct. Thay vào đó, protocol khai báo **associated type**: "một kiểu mà kiểu conform sẽ quyết định".
```swift
protocol Container {
    associatedtype Item                 // placeholder
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntBox: Container {
    private var storage: [Int] = []
    var count: Int { storage.count }
    mutating func append(_ item: Int) { storage.append(item) }   // compiler SUY RA Item == Int
    subscript(i: Int) -> Int { storage[i] }
}

extension Stack: Container {                                     // Item == Element
    var count: Int { items.count }
    mutating func append(_ item: Element) { push(item) }
    subscript(i: Int) -> Element { items[i] }
}

func allItemsMatch<C1: Container, C2: Container>(_ a: C1, _ b: C2) -> Bool
where C1.Item == C2.Item, C1.Item: Equatable {
    guard a.count == b.count else { return false }
    for i in 0..<a.count where a[i] != b[i] { return false }
    return true
}
var ib = IntBox(); ib.append(1)
var sb = Stack<Int>(); sb.push(1)
print(allItemsMatch(ib, sb))   // true
```
- Có thể ràng buộc: `associatedtype Key: Hashable`.
- Ví dụ thật: `Sequence.Element`, `Collection.Index`, `Identifiable.ID`, SwiftUI `View.Body`.
- Protocol có `associatedtype` (hoặc yêu cầu `Self`, như `Equatable`) từng **không dùng được làm kiểu biến** (lỗi nổi tiếng *"can only be used as a generic constraint"*). Từ Swift 5.7 dùng được dưới dạng `any Container` — nhưng bị hạn chế (xem §3).
- **Primary associated type** (Swift 5.7): `protocol Container<Item>` → viết được `some Container<Int>`, `any Container<String>`. `Sequence<Element>`, `Collection<Element>` đều đã có.

### §3. `some` vs `any` — phần quan trọng nhất tuần

```swift
protocol Shape { func area() -> Double }
struct Square: Shape { var side: Double; func area() -> Double { side * side } }
struct Circle: Shape { var r: Double; func area() -> Double { 3.14 * r * r } }
```

#### 3.1. `some` — opaque type: "một kiểu cụ thể, cố định, nhưng tôi không nói tên"
```swift
func makeSquare() -> some Shape { Square(side: 2) }   // trả về: một kiểu Shape cụ thể (thật ra là Square)
print(makeSquare().area())   // 4.0
```
- Kiểu thật được **compiler biết**, chỉ **ẩn với người gọi**. Mọi đường return phải cùng một kiểu:
  ```swift
  // func make(_ f: Bool) -> some Shape { if f { return Square(side: 1) } else { return Circle(r: 1) } }
  // ❌ error: function declares an opaque return type 'some Shape', but the return statements
  //           in its body do not have matching underlying types
  ```
- Ở **vị trí tham số**, `some` là cú pháp gọn của generic (Swift 5.7):
  ```swift
  func describe(_ shape: some Shape) -> String { "area = \(shape.area())" }
  // y hệt: func describe<S: Shape>(_ shape: S) -> String
  ```
- Ví dụ quen thuộc: `var body: some View` — kiểu thật là một kiểu lồng nhau khổng lồ như `VStack<TupleView<(Text, Button<Text>)>>`, `some` giúp không phải viết ra mà compiler vẫn tối ưu được.

#### 3.2. `any` — existential type: "một cái hộp chứa *bất kỳ* kiểu nào conform"
```swift
var shapes: [any Shape] = [Square(side: 1), Circle(r: 1)]   // trộn nhiều kiểu được
shapes.append(Square(side: 3))
print(shapes.map { $0.area() })   // [1.0, 3.14, 9.0]

func randomShape(_ flag: Bool) -> any Shape { flag ? Square(side: 1) : Circle(r: 1) }   // trả kiểu khác nhau được
// let mixed: [some Shape] = [Square(side: 1), Circle(r: 1)]   // ❌ some không trộn kiểu được
```

#### 3.3. Chi phí của `any` (interviewer rất thích hỏi)
Giá trị `any Shape` được cất trong **existential container**:
```
┌──────────────────────────────┐
│ value buffer: 3 word (24 B)  │ ← giá trị ≤ 24 byte cất inline; lớn hơn → cấp phát HEAP, buffer giữ con trỏ
│ type metadata pointer (8 B)  │ ← kiểu thật là gì
│ witness table pointer (8 B)  │ ← bảng hàm của protocol (mỗi protocol thêm 1 con trỏ)
└──────────────────────────────┘
```
```swift
print(MemoryLayout<any Shape>.size)                            // 40
print(MemoryLayout<Square>.size)                               // 8
print(MemoryLayout<any AnyObject>.size, MemoryLayout<Any>.size)  // 8 32
```
Hệ quả:
1. **Boxing**: struct > 24 byte bị đưa lên heap (cấp phát + ARC).
2. **Dynamic dispatch**: mọi lời gọi đi qua witness table, compiler **không inline / specialize** được.
3. **Mất thông tin kiểu**: không dùng được các API cần `Self` hoặc associated type một cách tự nhiên:
   ```swift
   let a: any Equatable = 1
   let b: any Equatable = 1
   // print(a == b)   // ❌ error: binary operator '==' cannot be applied to two 'any Equatable' operands
   ```
   (`a` có thể là Int còn `b` là String — compiler không thể cho phép `==`.)

**Implicit opening** (Swift 5.7): truyền `any Shape` vào hàm generic/`some` thì compiler tự "mở hộp" lấy kiểu thật:
```swift
let anyShape: any Shape = Circle(r: 2)
print(describe(anyShape))   // area = 12.56
```

#### 3.4. Khi nào dùng cái nào?
| | Generic `<T: P>` / `some P` | `any P` |
|---|---|---|
| Kiểu thật | Một kiểu cố định, compiler biết | Có thể khác nhau mỗi giá trị, chỉ biết lúc runtime |
| Trộn nhiều kiểu trong mảng | ❌ | ✅ |
| Hiệu năng | Tốt (specialize, static dispatch, không box) | Có chi phí (box, witness table) |
| Dùng API `Self`/associated type | ✅ | Hạn chế |
| Ví dụ | `var body: some View`, `func render(_ v: some View)` | `[any Shape]`, `let service: any RepositoryServiceProtocol` (dependency injection) |

**Quy tắc thực dụng: mặc định dùng `some` (hoặc generic); chỉ dùng `any` khi thật sự cần chứa nhiều kiểu khác nhau, hoặc cần lưu một dependency mà kiểu thật thay đổi (thật/mock).** Từ Swift 5.6 nên viết rõ `any`; bỏ `any` với protocol có associated type thì compiler cảnh báo *"use of protocol 'Container' as a type must be written 'any Container'"*.

### §4. Type erasure

**Bài toán:** muốn lưu "một loader bất kỳ trả về `Int`" trong một property / mảng. Trước Swift 5.7 không làm được với protocol có associated type; kể cả bây giờ, nhiều codebase (và Combine) vẫn dùng **type erasure**: bọc kiểu cụ thể trong một wrapper generic, giấu kiểu thật đi, chỉ giữ lại kiểu "đầu ra".
```swift
protocol DataLoader {
    associatedtype Output
    func load() -> Output
}
struct NumberLoader: DataLoader { func load() -> Int { 42 } }
struct RandomNumberLoader: DataLoader { func load() -> Int { 7 } }
struct TextLoader: DataLoader { func load() -> String { "hello" } }

struct AnyLoader<Output>: DataLoader {                 // kiểu "đã xoá" — chỉ còn Output
    private let _load: () -> Output                     // giữ hành vi dưới dạng closure
    init<L: DataLoader>(_ loader: L) where L.Output == Output {
        _load = loader.load                             // capture method của loader thật
    }
    func load() -> Output { _load() }
}
extension DataLoader {
    func eraseToAnyLoader() -> AnyLoader<Output> { AnyLoader(self) }   // giống eraseToAnyPublisher()
}

let loaders: [AnyLoader<Int>] = [AnyLoader(NumberLoader()), RandomNumberLoader().eraseToAnyLoader()]
print(loaders.map { $0.load() })   // [42, 7]
// let bad: AnyLoader<Int> = AnyLoader(TextLoader())
// ❌ error: initializer 'init(_:)' requires the types 'Int' and 'String' be equivalent
```
Đây chính là cách `AnyPublisher<Output, Failure>`, `AnySequence<Element>`, `AnyView`, `AnyHashable` hoạt động: bên ngoài chỉ thấy `AnyPublisher<[Repo], Error>`, không thấy chuỗi `Map<FlatMap<URLSession.DataTaskPublisher, ...>>` bên trong.

**Cách hiện đại (Swift 5.7+)** với primary associated type:
```swift
protocol Loader2<Output> {
    associatedtype Output
    func load() -> Output
}
struct N2: Loader2 { func load() -> Int { 1 } }
let modern: [any Loader2<Int>] = [N2()]
print(modern.map { $0.load() })   // [1]
func makeLoader() -> some Loader2<Int> { N2() }
```
Phỏng vấn: nên biết **cả hai** — tự viết type erasure (hiểu cơ chế, đọc được code cũ/Combine) và biết `any P<T>` giờ thay được phần lớn trường hợp.

### §5. Error handling

#### 5.1. Định nghĩa và ném lỗi
```swift
import Foundation

enum NetworkError: Error {        // bất kỳ kiểu nào conform Error đều ném được; enum là phổ biến nhất
    case invalidURL
    case badStatus(code: Int)
    case decoding(String)
    case offline
}

func fetch(path: String, status: Int) throws -> String {
    guard path.hasPrefix("/") else { throw NetworkError.invalidURL }
    guard (200..<300).contains(status) else { throw NetworkError.badStatus(code: status) }
    return "data for \(path)"
}
```
Swift error **không phải exception** kiểu C++/Java: không unwind stack tốn kém, thực chất là **giá trị trả về đặc biệt** (một thanh ghi chứa error). `throws` hiện rõ trên signature và mọi lời gọi phải có `try` → biết chính xác dòng nào có thể lỗi.
```swift
// func g() throws { let x = fetch(path: "/a", status: 200) }
// ❌ error: call can throw but is not marked with 'try'
// func g() { let x = try fetch(path: "/a", status: 200) }
// ❌ error: errors thrown from here are not handled
```

#### 5.2. `do/catch` với pattern
```swift
do {
    let data = try fetch(path: "/users", status: 200)
    print(data)                                   // data for /users
    _ = try fetch(path: "/users", status: 404)
    print("không chạy tới đây")
} catch NetworkError.badStatus(let code) where code >= 500 {
    print("Lỗi server", code)
} catch NetworkError.badStatus(let code) {
    print("Lỗi client", code)                     // Lỗi client 404
} catch let error as NetworkError {
    print("Lỗi mạng khác:", error)
} catch {
    print("Lỗi không xác định:", error)           // `error` có sẵn trong catch không pattern
}
```

#### 5.3. `try?` và `try!`
```swift
let maybe = try? fetch(path: "users", status: 200)   // lỗi → nil, kiểu String?
print(maybe as Any)                                  // nil
let sure = try! fetch(path: "/ok", status: 200)      // lỗi → CRASH. Chỉ dùng khi chắc chắn
```
`try?` tiện nhưng **nuốt mất thông tin lỗi** — đừng dùng cho chỗ cần báo lỗi cho người dùng hay log.

#### 5.4. `rethrows`
Hàm nhận closure có thể throw; **chỉ throw khi closure throw**. Người gọi truyền closure không throw thì không cần `try`. `map`, `filter`, `first(where:)` đều là `rethrows`.
```swift
func retry<T>(_ times: Int, _ body: () throws -> T) rethrows -> T {
    for _ in 1..<times { if let v = try? body() { return v } }
    return try body()
}
let n = retry(3) { 5 }                 // closure không throw → không cần try
var attempts = 0
let r = try? retry(3) { () throws -> String in
    attempts += 1
    if attempts < 3 { throw NetworkError.offline }
    return "OK sau \(attempts) lần"
}
print(r ?? "fail")                     // OK sau 3 lần
```

#### 5.5. `Result<Success, Failure>`
`Result` là một enum `.success(Success)` / `.failure(Failure)` — lưu kết quả **như một giá trị** để truyền đi, cất lại. Trước async/await dùng chủ yếu cho completion handler; bây giờ vẫn hay gặp.
```swift
func load(_ ok: Bool) -> Result<Int, NetworkError> { ok ? .success(10) : .failure(.offline) }

switch load(false) {
case .success(let v): print("được", v)
case .failure(let e): print("lỗi", e)            // lỗi offline
}
print(load(true).map { $0 * 2 })                 // success(20)  — map/mapError/flatMap
do { print(try load(true).get()) } catch { }     // 10 — get() chuyển Result → throws
let res = Result { try fetch(path: "x", status: 200) }   // throws → Result<String, any Error>
if case .failure(let e) = res { print("bắt được:", e) }  // bắt được: invalidURL
```

#### 5.6. Typed throws (Swift 6, SE-0413)
```swift
enum ValidationError: Error { case empty, tooShort(min: Int) }

func validate(_ s: String) throws(ValidationError) -> String {   // chỉ ném ValidationError
    if s.isEmpty { throw .empty }                                 // viết gọn .empty được
    if s.count < 3 { throw .tooShort(min: 3) }
    return s
}
do {
    _ = try validate("ab")
} catch {
    // error có kiểu ValidationError (không phải any Error) → switch exhaustive, không cần default
    switch error {
    case .empty: print("rỗng")
    case .tooShort(let min): print("ngắn hơn \(min)")   // ngắn hơn 3
    }
}
// Ném sai kiểu: func f() throws(ValidationError) { throw NetworkError.offline }
// ❌ error: thrown expression type 'NetworkError' cannot be converted to error type 'ValidationError'
```
- `throws` = `throws(any Error)`; hàm không throw ≈ `throws(Never)`.
- Ưu: compiler biết chính xác loại lỗi, không cần `as?`, không box `any Error`.
- Nhược: **khoá cứng API** — thêm loại lỗi mới là breaking change. Khuyến nghị của Swift team: **mặc định vẫn dùng `throws` thường**; dùng typed throws trong module nội bộ, khi miền lỗi thực sự đóng, hoặc code generic cần truyền nguyên loại lỗi.

#### 5.7. `LocalizedError` — thông báo cho người dùng
```swift
extension NetworkError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .offline: return "Không có kết nối mạng"
        case .badStatus(let c): return "Máy chủ trả mã \(c)"
        default: return "Đã có lỗi xảy ra"
        }
    }
}
let e: Error = NetworkError.offline
print(e.localizedDescription)    // Không có kết nối mạng
```

#### 5.8. `defer`
Code trong `defer` chạy **khi thoát scope hiện tại**, bằng bất kỳ đường nào (return, throw, hết block). Nhiều `defer` chạy theo **thứ tự ngược** (LIFO).
```swift
func process() {
    print("mở file")
    defer { print("đóng file") }
    defer { print("defer 2") }
    print("xử lý")
}
process()
// mở file / xử lý / defer 2 / đóng file

func parse(_ s: String) throws -> Int {
    defer { print("defer chạy cả khi throw") }
    guard let v = Int(s) else { throw NetworkError.decoding(s) }
    return v
}
_ = try? parse("x")   // defer chạy cả khi throw
```
Dùng cho: unlock lock, đóng file, `isLoading = false`, gọi completion "đảm bảo luôn gọi". Không được `return`/`throw` ra khỏi `defer`.

### §6. Property wrapper

Property wrapper = **tái sử dụng logic get/set** của property. Bạn đã dùng nhiều: `@State`, `@Published`, `@AppStorage`, `@MainActor`... (cái cuối không phải wrapper nhưng cùng cú pháp attribute).

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    let range: ClosedRange<Value>
    init(wrappedValue: Value, _ range: ClosedRange<Value>) {     // wrappedValue: giá trị sau dấu =
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
    var wrappedValue: Value {                                     // BẮT BUỘC có
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

struct Player {
    @Clamped(0...100) var health: Int = 150
    @Clamped(0.0...1.0) var volume: Double = 0.5
}
var p = Player()
print(p.health)      // 100 — giá trị khởi tạo cũng bị kẹp
p.health -= 130
print(p.health)      // 0
p.volume = 2
print(p.volume)      // 1.0
```
**Compiler biến đổi thành gì?**
```swift
// @Clamped(0...100) var health: Int = 150   trở thành:
// private var _health = Clamped(wrappedValue: 150, 0...100)
// var health: Int {
//     get { _health.wrappedValue }
//     set { _health.wrappedValue = newValue }
// }
// var $health: ... { _health.projectedValue }     // nếu wrapper có projectedValue
```

**`projectedValue`** — giá trị phụ, truy cập bằng `$tên` (như `$text` trong SwiftUI trả `Binding`):
```swift
@propertyWrapper
struct Tracked<Value> {
    private var value: Value
    private(set) var history: [Value] = []
    init(wrappedValue: Value) { value = wrappedValue }
    var wrappedValue: Value {
        get { value }
        set { history.append(value); value = newValue }
    }
    var projectedValue: [Value] { history }
}
struct Doc { @Tracked var title = "v1" }
var d = Doc()
d.title = "v2"; d.title = "v3"
print(d.title, d.$title)    // v3 ["v1", "v2"]
```

**Ví dụ thực tế: lưu UserDefaults**
```swift
@propertyWrapper
struct UserDefault<Value> {
    let key: String
    let defaultValue: Value
    var container: UserDefaults = .standard
    var wrappedValue: Value {
        get { container.object(forKey: key) as? Value ?? defaultValue }
        nonmutating set { container.set(newValue, forKey: key) }   // không sửa struct → dùng được trên let
    }
}
struct AppSettings {
    @UserDefault(key: "onboardingDone", defaultValue: false) var onboardingDone: Bool
}
```
Hạn chế cần biết: wrapper không khai báo được trong protocol requirement; wrapper trên property làm mất `lazy`; logic ẩn → lạm dụng làm code khó đọc.

### §7. Result builder (giới thiệu)

Result builder biến **một chuỗi câu lệnh** (kể cả `if`, `for`) thành **một giá trị** bằng cách gọi các hàm `buildXxx`. `@ViewBuilder` của SwiftUI chính là một result builder — đó là lý do trong `body` bạn liệt kê các view mà không cần `return [a, b, c]`.
```swift
@resultBuilder
enum LinesBuilder {
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
    static func buildExpression(_ line: String) -> [String] { [line] }        // mỗi dòng String
    static func buildOptional(_ part: [String]?) -> [String] { part ?? [] }    // if không có else
    static func buildEither(first part: [String]) -> [String] { part }         // if/else nhánh 1
    static func buildEither(second part: [String]) -> [String] { part }        // if/else nhánh 2
    static func buildArray(_ parts: [[String]]) -> [String] { parts.flatMap { $0 } }   // for-in
}
func makeReport(@LinesBuilder _ content: () -> [String]) -> String {
    content().joined(separator: "\n")
}
let isAdmin = true
let tags = ["swift", "ios"]
print(makeReport {
    "# Báo cáo"
    if isAdmin { "Quyền: admin" } else { "Quyền: user" }
    for t in tags { "- \(t)" }
    if tags.isEmpty { "(không có tag)" }
})
// # Báo cáo
// Quyền: admin
// - swift
// - ios
```
Mức 2 năm: chỉ cần giải thích được result builder là gì, liên hệ `@ViewBuilder`, và biết vì sao trong `body` dùng `if` được nhưng (trước đây) không dùng `guard`/`switch` phức tạp tuỳ ý.

### §8. Access control

| Mức | Thấy được ở đâu | Kế thừa / override ngoài module |
|---|---|---|
| `open` | Mọi module | ✅ (chỉ cho class và member của class) |
| `public` | Mọi module | ❌ |
| `package` (Swift 5.9) | Các module trong cùng **Swift package** | ❌ |
| `internal` *(mặc định)* | Trong cùng module (target) | — |
| `fileprivate` | Trong cùng file | — |
| `private` | Trong scope khai báo + **extension của nó trong cùng file** (Swift 4+) | — |

```swift
// File Account.swift
public struct Account {
    public let id: Int
    private var pin: String
    fileprivate var note = ""
    public private(set) var balance: Double = 0       // đọc public, ghi chỉ bên trong
    public init(id: Int, pin: String) { self.id = id; self.pin = pin }   // public struct phải tự viết public init
    public mutating func deposit(_ x: Double) { balance += x }
}
extension Account {
    func checkPin(_ p: String) -> Bool { pin == p }   // extension cùng file → thấy private
}

// File main.swift (khác file)
var acc = Account(id: 1, pin: "1234")
acc.deposit(100)
print(acc.balance, acc.checkPin("1234"))   // 100.0 true
// acc.balance = 5      // ❌ error: cannot assign to property: 'balance' setter is inaccessible
// print(acc.pin)       // ❌ error: 'pin' is inaccessible due to 'private' protection level
// print(acc.note)      // ❌ error: 'note' is inaccessible due to 'fileprivate' protection level
```
- Quy tắc: một thực thể không được "public hơn" thứ nó phụ thuộc (hàm `public` không thể nhận tham số kiểu `internal`).
- `open` vs `public`: thư viện dùng `public` để giữ quyền thay đổi nội bộ; chỉ `open` những class **được thiết kế để kế thừa**. (UIKit: `open class UIViewController`.)
- Lợi ích hiệu năng: `private`/`fileprivate` giúp compiler suy ra `final` → static dispatch (tuần 2).
- Trong app: dùng `private` cho mọi thứ không cần lộ (IBOutlet, helper), `private(set)` cho state của ViewModel.

### §9. Codable chuyên sâu

`Codable = Decodable & Encodable`. Compiler tự sinh code encode/decode nếu mọi property đều Codable.

#### 9.1. Cơ bản & `CodingKeys`
```swift
import Foundation

struct Owner: Codable { let login: String; let id: Int }

struct Repository: Codable {
    let id: Int
    let name: String
    let fullName: String
    let stars: Int
    let description: String?           // Optional: thiếu key HOẶC null → nil
    let owner: Owner                   // nested object: chỉ cần kiểu con cũng Codable
    enum CodingKeys: String, CodingKey {
        case id, name, description, owner
        case fullName = "full_name"
        case stars = "stargazers_count"   // đổi tên tuỳ ý
    }
}
let json = """
{
  "id": 1, "name": "swift", "full_name": "apple/swift",
  "stargazers_count": 67000, "description": null,
  "owner": { "login": "apple", "id": 10639145 },
  "language": "C++"
}
""".data(using: .utf8)!
let repo = try! JSONDecoder().decode(Repository.self, from: json)
print(repo.fullName, repo.stars, repo.description ?? "-", repo.owner.login)
// apple/swift 67000 - apple
```
- Key thừa trong JSON (`language`) **bị bỏ qua**, không lỗi.
- Đã khai `CodingKeys` thì phải liệt kê **mọi** property cần decode (property không có trong CodingKeys phải có giá trị mặc định).

#### 9.2. `keyDecodingStrategy`
```swift
struct Repo2: Decodable { let fullName: String; let stargazersCount: Int }
let dec = JSONDecoder()
dec.keyDecodingStrategy = .convertFromSnakeCase   // full_name → fullName
let r2 = try! dec.decode(Repo2.self, from: json)
print(r2.fullName, r2.stargazersCount)            // apple/swift 67000
```
⚠️ **Bẫy:** đã dùng `.convertFromSnakeCase` mà còn viết `case fullName = "full_name"` trong CodingKeys → **decode lỗi** (keyNotFound), vì strategy đổi key JSON thành `fullName` *trước* khi so với CodingKeys. Chọn một trong hai cách.

#### 9.3. Response bọc ngoài
```swift
struct SearchResponse: Decodable {
    let totalCount: Int
    let items: [Repo2]
}
let json3 = #"{"total_count": 2, "incomplete_results": false, "items": [{"full_name": "a/b", "stargazers_count": 1}, {"full_name": "c/d", "stargazers_count": 2}]}"#.data(using: .utf8)!
let sr = try! dec.decode(SearchResponse.self, from: json3)
print(sr.totalCount, sr.items.map(\.fullName))   // 2 ["a/b", "c/d"]
```
(`#"..."#` là raw string — không cần escape dấu `"`.)

#### 9.4. Custom `init(from:)` + nested container — "làm phẳng" JSON
```swift
struct Weather: Decodable {
    let city: String
    let temperature: Double
    let condition: String

    enum CodingKeys: String, CodingKey { case name, main, weather }
    enum MainKeys: String, CodingKey { case temp }
    struct Condition: Decodable { let description: String }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        city = try c.decode(String.self, forKey: .name)
        let main = try c.nestedContainer(keyedBy: MainKeys.self, forKey: .main)   // đi vào "main"
        temperature = try main.decode(Double.self, forKey: .temp)
        let conditions = try c.decode([Condition].self, forKey: .weather)
        condition = conditions.first?.description ?? "unknown"
    }
}
let json4 = #"{"name": "Hà Nội", "main": {"temp": 28.5, "humidity": 80}, "weather": [{"description": "mưa nhẹ"}]}"#.data(using: .utf8)!
let w = try! JSONDecoder().decode(Weather.self, from: json4)
print(w.city, w.temperature, w.condition)   // Hà Nội 28.5 mưa nhẹ
```

#### 9.5. Giá trị mặc định & API "bẩn" (số lúc là number lúc là string)
```swift
struct Product: Decodable {
    let id: Int
    let name: String
    let price: Double
    let tags: [String]
    enum CodingKeys: String, CodingKey { case id, name, price, tags }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        id = try c.decode(Int.self, forKey: .id)
        name = try c.decode(String.self, forKey: .name)
        if let p = try? c.decode(Double.self, forKey: .price) {
            price = p
        } else {
            let s = try c.decode(String.self, forKey: .price)
            guard let p = Double(s) else {
                throw DecodingError.dataCorruptedError(forKey: .price, in: c,
                                                       debugDescription: "price không phải số: \(s)")
            }
            price = p
        }
        tags = try c.decodeIfPresent([String].self, forKey: .tags) ?? []   // thiếu → mảng rỗng
    }
}
let json5 = #"[{"id": 1, "name": "A", "price": 9.5}, {"id": 2, "name": "B", "price": "12.0", "tags": ["new"]}]"#.data(using: .utf8)!
let products = try! JSONDecoder().decode([Product].self, from: json5)
print(products.map { "\($0.name) \($0.price) \($0.tags)" })   // ["A 9.5 []", "B 12.0 [\"new\"]"]
```
`decode` vs `decodeIfPresent`: `decode(Optional…)`/`decodeIfPresent` trả nil khi key thiếu hoặc null; `decode` kiểu non-optional thì lỗi.

#### 9.6. Enum với giá trị lạ từ server
Server thêm giá trị mới (`"internal"`) → enum raw value decode **lỗi cả response**. Phòng thủ:
```swift
enum Visibility: String, Decodable {
    case `public`, `private`, unknown
    init(from decoder: Decoder) throws {
        let raw = try decoder.singleValueContainer().decode(String.self)
        self = Visibility(rawValue: raw) ?? .unknown
    }
}
let vis = try! JSONDecoder().decode([Visibility].self, from: #"["public", "internal"]"#.data(using: .utf8)!)
print(vis.map(\.rawValue))   // ["public", "unknown"]
```

#### 9.7. Ngày tháng
```swift
struct Event: Codable { let name: String; let createdAt: Date }

let isoDecoder = JSONDecoder()
isoDecoder.dateDecodingStrategy = .iso8601                 // "2026-11-30T08:00:00Z" — GitHub API dùng kiểu này
let ev = try! isoDecoder.decode(Event.self,
    from: #"{"name": "push", "createdAt": "2026-11-30T08:00:00Z"}"#.data(using: .utf8)!)
print(ev.createdAt.timeIntervalSince1970)                  // 1796025600.0

let epochDecoder = JSONDecoder()
epochDecoder.dateDecodingStrategy = .secondsSince1970      // 1796025600
let ev2 = try! epochDecoder.decode(Event.self,
    from: #"{"name": "push", "createdAt": 1796025600}"#.data(using: .utf8)!)
print(ev2.createdAt == ev.createdAt)                       // true

let fmt = DateFormatter()
fmt.locale = Locale(identifier: "en_US_POSIX")             // BẮT BUỘC cho format cố định
fmt.timeZone = TimeZone(identifier: "UTC")
fmt.dateFormat = "dd/MM/yyyy"
let customDecoder = JSONDecoder()
customDecoder.dateDecodingStrategy = .formatted(fmt)
let ev3 = try! customDecoder.decode(Event.self,
    from: #"{"name": "x", "createdAt": "30/11/2026"}"#.data(using: .utf8)!)
print(ev3.createdAt.timeIntervalSince1970)                 // 1795996800.0
```
Mặc định (`.deferredToDate`) là **số giây tính từ 01/01/2001** — hay gây bug khi server gửi epoch 1970. Có thêm `.millisecondsSince1970` và `.custom { decoder in ... }`.

#### 9.8. Encode
```swift
let enc = JSONEncoder()
enc.outputFormatting = [.prettyPrinted, .sortedKeys]
enc.keyEncodingStrategy = .convertToSnakeCase
enc.dateEncodingStrategy = .iso8601
struct Out: Encodable { let userName: String; let createdAt: Date }
print(String(data: try! enc.encode(Out(userName: "an", createdAt: ev.createdAt)), encoding: .utf8)!)
// {
//   "created_at" : "2026-11-30T08:00:00Z",
//   "user_name" : "an"
// }
```

#### 9.9. Đọc hiểu `DecodingError` — kỹ năng debug quan trọng
```swift
func tryDecode<T: Decodable>(_ type: T.Type, _ s: String) {
    do {
        _ = try JSONDecoder().decode(type, from: Data(s.utf8))
        print("OK")
    } catch let DecodingError.keyNotFound(key, ctx) {
        print("Thiếu key '\(key.stringValue)' tại \(ctx.codingPath.map(\.stringValue))")
    } catch let DecodingError.typeMismatch(type, ctx) {
        print("Sai kiểu, cần \(type) tại \(ctx.codingPath.map(\.stringValue))")
    } catch let DecodingError.valueNotFound(type, ctx) {
        print("Giá trị null, cần \(type) tại \(ctx.codingPath.map(\.stringValue))")
    } catch let DecodingError.dataCorrupted(ctx) {
        print("Dữ liệu hỏng:", ctx.debugDescription)
    } catch {
        print("Lỗi khác:", error)
    }
}
tryDecode(Owner.self, #"{"login": "a"}"#)               // Thiếu key 'id' tại []
tryDecode(Owner.self, #"{"login": "a", "id": "10"}"#)   // Sai kiểu, cần Int tại ["id"]
tryDecode(Owner.self, #"{"login": null, "id": 1}"#)     // Giá trị null, cần String tại ["login"]
tryDecode(Owner.self, #"{"login": "a", "id": 1"#)       // Dữ liệu hỏng: The given data was not valid JSON.
tryDecode([Owner].self, #"[{"login": "a", "id": 1}, {"login": "b"}]"#)   // Thiếu key 'id' tại ["Index 1"]
```
`codingPath` cho biết **chính xác chỗ hỏng** — luôn in ra khi debug, đừng chỉ in `error.localizedDescription` (rất chung chung).

---

## ⚠️ Bẫy hay gặp

1. **`some` không phải `any`**: `some` = một kiểu cố định (compiler biết); `any` = hộp chứa kiểu bất kỳ. Trả 2 kiểu khác nhau từ hàm `-> some P` là lỗi.
2. **`any P` có chi phí**: existential 40 byte, giá trị > 24 byte bị box lên heap, dynamic dispatch, không specialize.
3. **Không gọi được `==` giữa hai `any Equatable`.**
4. **Generic `T` không có constraint thì không làm được gì** ngoài gán/truyền.
5. **`try?` nuốt lỗi** — mất thông tin để log/hiển thị.
6. **`defer` chạy theo thứ tự ngược** và chạy cả khi throw.
7. **Typed throws khoá cứng API** — mặc định vẫn dùng `throws`.
8. **`convertFromSnakeCase` + CodingKeys snake_case** → decode lỗi.
9. **Enum raw value gặp giá trị lạ** từ server → hỏng cả response. Thêm case `unknown`.
10. **Date mặc định là reference date 2001**, không phải 1970.
11. **`DateFormatter` thiếu `en_US_POSIX`** → máy người dùng để lịch Phật giáo / 12h là parse sai.
12. **`public struct` không có public memberwise init** — module khác không tạo được instance.
13. **Đã viết `CodingKeys` thì phải liệt kê đủ** các property cần decode.

---

## 🛠 Bài tập

> Tạo file trong `practice/week04/` và chạy bằng `swift tên_file.swift`. **Không xem lời giải trước khi tự làm ít nhất 20 phút.**

**Bài 1 — Generic Queue (T2)**
Viết `struct Queue<Element>` dùng **hai mảng** (`inbox`, `outbox`) để `enqueue` O(1) và `dequeue` O(1) amortized. Có `isEmpty`, `count`. Thêm `contains(_:)` chỉ khi `Element: Equatable`.
<details><summary>Lời giải</summary>

```swift
struct Queue<Element> {
    private var inbox: [Element] = []
    private var outbox: [Element] = []
    var isEmpty: Bool { inbox.isEmpty && outbox.isEmpty }
    var count: Int { inbox.count + outbox.count }
    mutating func enqueue(_ x: Element) { inbox.append(x) }
    mutating func dequeue() -> Element? {
        if outbox.isEmpty {
            outbox = inbox.reversed()       // O(n) nhưng mỗi phần tử chỉ bị đảo 1 lần
            inbox.removeAll()
        }
        return outbox.popLast()             // O(1)
    }
}
extension Queue where Element: Equatable {
    func contains(_ x: Element) -> Bool { inbox.contains(x) || outbox.contains(x) }
}
var q = Queue<String>()
q.enqueue("a"); q.enqueue("b"); q.enqueue("c")
print(q.dequeue() ?? "-", q.count, q.contains("c"))   // a 2 true
q.enqueue("d")
print(q.dequeue() ?? "-", q.dequeue() ?? "-", q.dequeue() ?? "-", q.dequeue() ?? "nil")   // b c d nil
```
Vì sao không dùng `array.removeFirst()`? Nó là O(n) (phải dịch mọi phần tử).
</details>

**Bài 2 — Protocol có associatedtype (T2)**
`protocol Cache { associatedtype Key: Hashable; associatedtype Value; mutating func set(_:for:); func value(for:) -> Value? }`. Cài `struct MemoryCache<Key, Value>` có `capacity`, khi đầy thì xoá key cũ nhất (FIFO). Viết hàm generic `warmUp(_ cache: inout C, with pairs: [(C.Key, C.Value)])`.
<details><summary>Lời giải</summary>

```swift
protocol Cache {
    associatedtype Key: Hashable
    associatedtype Value
    mutating func set(_ value: Value, for key: Key)
    func value(for key: Key) -> Value?
}
struct MemoryCache<Key: Hashable, Value>: Cache {
    private var storage: [Key: Value] = [:]
    private var order: [Key] = []
    let capacity: Int
    init(capacity: Int) { self.capacity = capacity }
    mutating func set(_ value: Value, for key: Key) {
        if storage[key] == nil { order.append(key) }
        storage[key] = value
        if order.count > capacity {
            let oldest = order.removeFirst()
            storage[oldest] = nil
        }
    }
    func value(for key: Key) -> Value? { storage[key] }
}
func warmUp<C: Cache>(_ cache: inout C, with pairs: [(C.Key, C.Value)]) {
    for (k, v) in pairs { cache.set(v, for: k) }
}
var cache = MemoryCache<String, Int>(capacity: 2)
warmUp(&cache, with: [("a", 1), ("b", 2), ("c", 3)])
print(cache.value(for: "a") as Any, cache.value(for: "c") as Any)   // nil Optional(3)
```
Ở đây generic parameter `Key`, `Value` của struct **trùng tên** với associated type → compiler tự khớp.
</details>

**Bài 3 — `some` hay `any`? (T3)**
Với `Shape`, `Square`, `Circle` ở §3, mỗi dòng dưới đây compile hay không? Nếu không, lỗi gì và sửa thế nào? Tự trả lời trước, rồi gõ vào file để kiểm tra.
```swift
// a) func make(_ f: Bool) -> some Shape { if f { return Square(side: 1) } else { return Circle(r: 1) } }
// b) let mixed: [any Shape] = [Square(side: 1), Circle(r: 1)]
// c) let mixed2: [some Shape] = [Square(side: 1), Circle(r: 1)]
// d) func show(_ s: some Shape) {}; let x: any Shape = Circle(r: 1); show(x)
// e) let a: any Equatable = 1; let b: any Equatable = 1; print(a == b)
// f) func biggest(_ items: [some Shape]) -> Double { items.map { $0.area() }.max() ?? 0 }
```
<details><summary>Lời giải</summary>

- a) ❌ *function declares an opaque return type 'some Shape', but the return statements in its body do not have matching underlying types* → đổi thành `-> any Shape`.
- b) ✅
- c) ❌ (compiler báo *conflicting arguments to generic parameter*) → `[some Shape]` phải cùng một kiểu; dùng `[any Shape]`.
- d) ✅ nhờ implicit opening (Swift 5.7+).
- e) ❌ *binary operator '==' cannot be applied to two 'any Equatable' operands* → dùng generic `func same<T: Equatable>(_ a: T, _ b: T)`.
- f) ✅ — `[some Shape]` ở tham số là `[T] where T: Shape`: một mảng **đồng nhất**; gọi với `[Square]` được, với `[any Shape]` thì không.
</details>

**Bài 4 — Type erasure (T3)**
`protocol Validator { associatedtype Value; func validate(_ value: Value) -> Bool }`. Viết `MinValue`, `EvenNumber` (cho `Int`), `NonEmpty` (cho `String`). Viết `AnyValidator<Value>` có 2 init: từ một `Validator` bất kỳ cùng `Value`, và từ một closure. Tạo `[AnyValidator<Int>]` gồm "≥ 10", "chẵn", "< 100" và kiểm tra `[12, 13, 4, 200]`.
<details><summary>Lời giải</summary>

```swift
protocol Validator {
    associatedtype Value
    func validate(_ value: Value) -> Bool
}
struct MinValue: Validator { let min: Int; func validate(_ v: Int) -> Bool { v >= min } }
struct EvenNumber: Validator { func validate(_ v: Int) -> Bool { v % 2 == 0 } }
struct NonEmpty: Validator { func validate(_ v: String) -> Bool { !v.isEmpty } }

struct AnyValidator<Value>: Validator {
    private let _validate: (Value) -> Bool
    init<V: Validator>(_ base: V) where V.Value == Value { _validate = base.validate }
    init(_ closure: @escaping (Value) -> Bool) { _validate = closure }
    func validate(_ value: Value) -> Bool { _validate(value) }
}
let intRules: [AnyValidator<Int>] = [AnyValidator(MinValue(min: 10)),
                                     AnyValidator(EvenNumber()),
                                     AnyValidator { $0 < 100 }]
for n in [12, 13, 4, 200] { print(n, intRules.allSatisfy { $0.validate(n) }) }
// 12 true / 13 false / 4 false / 200 false
```
Câu hỏi thêm: viết lại bằng `protocol Validator<Value>` và `[any Validator<Int>]` — mất gì? (Không còn init từ closure trực tiếp; phải tạo struct riêng cho rule closure.)
</details>

**Bài 5 — Error handling (T4)**
`enum LoginError: Error, LocalizedError` với `emptyField(String)`, `wrongPassword(attemptsLeft: Int)`, `locked`. `final class LoginService` có `func login(user:password:) throws(LoginError) -> String` (mật khẩu đúng `"123456"`, sai 3 lần thì `locked`), ghi `log` "start"/"end" bằng `defer`. Thêm `loginResult(...) -> Result<String, LoginError>`. Thử chuỗi mật khẩu `["", "1", "2", "123456", "3"]`.
<details><summary>Lời giải</summary>

```swift
import Foundation

enum LoginError: Error, LocalizedError, Equatable {
    case emptyField(String)
    case wrongPassword(attemptsLeft: Int)
    case locked
    var errorDescription: String? {
        switch self {
        case .emptyField(let f): return "Vui lòng nhập \(f)"
        case .wrongPassword(let n): return "Sai mật khẩu, còn \(n) lần thử"
        case .locked: return "Tài khoản đã bị khoá"
        }
    }
}
final class LoginService {
    private var failures = 0
    private(set) var log: [String] = []

    func login(user: String, password: String) throws(LoginError) -> String {
        log.append("start \(user)")
        defer { log.append("end \(user)") }
        guard !user.isEmpty else { throw .emptyField("tên đăng nhập") }
        guard !password.isEmpty else { throw .emptyField("mật khẩu") }
        guard failures < 3 else { throw .locked }
        guard password == "123456" else {
            failures += 1
            if failures >= 3 { throw .locked }
            throw .wrongPassword(attemptsLeft: 3 - failures)
        }
        failures = 0
        return "token-\(user)"
    }
    func loginResult(user: String, password: String) -> Result<String, LoginError> {
        Result { () throws(LoginError) -> String in try login(user: user, password: password) }
    }
}
let svc = LoginService()
for pw in ["", "1", "2", "123456", "3"] {
    do {
        print("OK", try svc.login(user: "an", password: pw))
    } catch {
        print(error.localizedDescription)
    }
}
// Vui lòng nhập mật khẩu
// Sai mật khẩu, còn 2 lần thử
// Sai mật khẩu, còn 1 lần thử
// OK token-an
// Sai mật khẩu, còn 2 lần thử
print(svc.log.count)   // 10 — mỗi lần gọi đều có start + end, kể cả khi throw
if case .failure(let e) = svc.loginResult(user: "", password: "x") { print(e.localizedDescription) }
// Vui lòng nhập tên đăng nhập
```
</details>

**Bài 6 — Property wrapper + access control (T5)**
a) Viết `@NonNegative` cho kiểu `Numeric & Comparable`: giá trị âm bị đưa về 0; `projectedValue: Bool` cho biết lần gán gần nhất có bị sửa không. Dùng trong `struct CartItem { let name; @NonNegative var quantity: Int = 0; @NonNegative var discount: Double = 0 }`.
b) Trong `CartItem`, đặt access level hợp lý nếu đây là một `public` model của một Swift package: cái gì `public`, cái gì `private(set)`, cần thêm init gì?
<details><summary>Lời giải a)</summary>

```swift
@propertyWrapper
struct NonNegative<Value: Numeric & Comparable> {
    private var value: Value
    private(set) var wasCorrected = false
    init(wrappedValue: Value) {
        value = max(wrappedValue, 0)
        wasCorrected = wrappedValue < 0
    }
    var wrappedValue: Value {
        get { value }
        set { wasCorrected = newValue < 0; value = max(newValue, 0) }
    }
    var projectedValue: Bool { wasCorrected }
}
struct CartItem {
    let name: String
    @NonNegative var quantity: Int = 0
    @NonNegative var discount: Double = 0
}
var item = CartItem(name: "Áo")
item.quantity = -3
print(item.quantity, item.$quantity)    // 0 true
item.quantity = 2
print(item.quantity, item.$quantity)    // 2 false
let item2 = CartItem(name: "Quần", quantity: 5, discount: -1)   // memberwise init vẫn có
print(item2.discount, item2.$discount)  // 0.0 true
```
b) `public struct CartItem`, `public let name`, `@NonNegative public var quantity` (hoặc `public private(set)` nếu chỉ sửa qua method), và **phải viết `public init(...)`** vì memberwise init chỉ là `internal`; bản thân wrapper cũng phải `public` nếu dùng trên property `public`.
</details>

**Bài 7 — Codable thực tế (T6)**
Decode JSON sau thành `[GitHubRepo]` dùng `.convertFromSnakeCase` + `.iso8601`. Yêu cầu: `language` có thể null; `topics` có thể thiếu → `[]`; `visibility` là enum với case `unknown` cho giá trị lạ. In: tên, ngôn ngữ (hoặc "—"), visibility, số topic, năm tạo.
```json
[
  {"id": 1, "full_name": "apple/swift", "stargazers_count": 67000, "language": "C++",
   "visibility": "public", "created_at": "2014-06-03T16:00:00Z", "topics": ["compiler"]},
  {"id": 2, "full_name": "me/secret", "stargazers_count": 0, "language": null,
   "visibility": "internal", "created_at": "2026-11-30T08:00:00Z"}
]
```
<details><summary>Lời giải</summary>

```swift
import Foundation

struct GitHubRepo: Decodable {
    enum Visibility: String, Decodable {
        case `public`, `private`, unknown
        init(from decoder: Decoder) throws {
            let raw = try decoder.singleValueContainer().decode(String.self)
            self = Visibility(rawValue: raw) ?? .unknown
        }
    }
    let id: Int
    let fullName: String
    let stargazersCount: Int
    let language: String?
    let visibility: Visibility
    let createdAt: Date
    let topics: [String]

    enum CodingKeys: String, CodingKey {        // camelCase vì đã dùng convertFromSnakeCase
        case id, fullName, stargazersCount, language, visibility, createdAt, topics
    }
    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        id = try c.decode(Int.self, forKey: .id)
        fullName = try c.decode(String.self, forKey: .fullName)
        stargazersCount = try c.decode(Int.self, forKey: .stargazersCount)
        language = try c.decodeIfPresent(String.self, forKey: .language)
        visibility = try c.decode(Visibility.self, forKey: .visibility)
        createdAt = try c.decode(Date.self, forKey: .createdAt)
        topics = try c.decodeIfPresent([String].self, forKey: .topics) ?? []
    }
}
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
decoder.dateDecodingStrategy = .iso8601
let repos = try! decoder.decode([GitHubRepo].self, from: Data(json.utf8))   // json: chuỗi ở đề bài
let utc = TimeZone(identifier: "UTC")!
for r in repos {
    let year = Calendar(identifier: .gregorian).dateComponents(in: utc, from: r.createdAt).year!
    print(r.fullName, r.language ?? "—", r.visibility.rawValue, r.topics.count, year)
}
// apple/swift C++ public 1 2014
// me/secret — unknown 0 2026
```
Vì sao phải viết `init(from:)` mà không để compiler tự sinh? Vì tự sinh thì `topics: [String]` thiếu key sẽ lỗi `keyNotFound` (compiler chỉ dùng `decodeIfPresent` cho property **Optional**).
</details>

**🏋️ Bài tập lớn (T7) — Mini `APIClient` generic (offline)**
Viết `apiclient.swift` — đây là bản nháp cho side project Milestone 4:
- `enum APIError: Error { case notFound(path: String), server(status: Int), decoding(String) }`
- `struct Endpoint<Response: Decodable> { let path: String }` + static factory **có ràng buộc kiểu**: `.repos(of:)` trả `Endpoint<[RepoDTO]>`, `.user(_:)` trả `Endpoint<UserDTO>`.
- `protocol Transport { func data(for path: String) throws(APIError) -> Data }` + `MockTransport` trả JSON theo path (status ≠ 2xx → `.server`).
- `final class APIClient` nhận `any Transport`; `func send<Response>(_ endpoint: Endpoint<Response>) throws(APIError) -> Response` (lỗi decode → `.decoding`); `func result(...) -> Result<Response, APIError>`.
- Kiểm tra: gọi `client.send(.repos(of: "an"))` **không cần ghi kiểu** mà vẫn ra `[RepoDTO]`; xử lý 3 lỗi: path không tồn tại, status 503, JSON thiếu field.

<details><summary>Lời giải tham khảo</summary>

```swift
import Foundation

enum APIError: Error, Equatable {
    case notFound(path: String)
    case server(status: Int)
    case decoding(String)
}

struct RepoDTO: Decodable, Equatable { let id: Int; let name: String; let stargazersCount: Int }
struct UserDTO: Decodable { let login: String; let publicRepos: Int }

struct Endpoint<Response: Decodable> {
    let path: String
}
extension Endpoint where Response == [RepoDTO] {
    static func repos(of user: String) -> Endpoint { Endpoint(path: "/users/\(user)/repos") }
}
extension Endpoint where Response == UserDTO {
    static func user(_ login: String) -> Endpoint { Endpoint(path: "/users/\(login)") }
}

protocol Transport {
    func data(for path: String) throws(APIError) -> Data
}
struct MockTransport: Transport {
    let routes: [String: (status: Int, body: String)]
    func data(for path: String) throws(APIError) -> Data {
        guard let route = routes[path] else { throw .notFound(path: path) }
        guard (200..<300).contains(route.status) else { throw .server(status: route.status) }
        return Data(route.body.utf8)
    }
}

final class APIClient {
    private let transport: any Transport
    private let decoder: JSONDecoder = {
        let d = JSONDecoder()
        d.keyDecodingStrategy = .convertFromSnakeCase
        return d
    }()
    init(transport: any Transport) { self.transport = transport }

    func send<Response>(_ endpoint: Endpoint<Response>) throws(APIError) -> Response {
        let data = try transport.data(for: endpoint.path)
        do {
            return try decoder.decode(Response.self, from: data)
        } catch {
            throw .decoding(String(describing: error))   // đổi DecodingError → APIError
        }
    }
    func result<Response>(_ endpoint: Endpoint<Response>) -> Result<Response, APIError> {
        do { return .success(try send(endpoint)) } catch { return .failure(error) }
    }
}

let transport = MockTransport(routes: [
    "/users/an/repos": (200, #"[{"id": 1, "name": "RepoScope", "stargazers_count": 5}, {"id": 2, "name": "dotfiles", "stargazers_count": 1}]"#),
    "/users/an": (200, #"{"login": "an", "public_repos": 2}"#),
    "/users/broken": (200, #"{"login": "broken"}"#),
    "/users/down": (503, ""),
])
let client = APIClient(transport: transport)

do {
    let repos = try client.send(.repos(of: "an"))     // suy ra [RepoDTO]
    print(repos.map(\.name), repos.reduce(0) { $0 + $1.stargazersCount })   // ["RepoScope", "dotfiles"] 6
    let user = try client.send(.user("an"))            // suy ra UserDTO
    print(user.login, user.publicRepos)                // an 2
} catch {
    print("Không mong đợi:", error)
}

for login in ["ghost", "down", "broken"] {
    switch client.result(.user(login)) {
    case .success(let u): print("OK", u.login)
    case .failure(.notFound(let path)): print("404", path)
    case .failure(.server(let status)): print("Server", status)
    case .failure(.decoding): print("Decode lỗi")
    }
}
// 404 /users/ghost
// Server 503
// Decode lỗi
```
Điểm đáng nói khi phỏng vấn: `Endpoint` mang **kiểu response trong type parameter** → không thể decode nhầm kiểu; `Transport` là protocol → thay `MockTransport` bằng `URLSessionTransport` (tuần 16) mà `APIClient` không đổi; `switch` trên `Result<_, APIError>` exhaustive, không cần `default`.
</details>

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Generics là gì? Lợi ích so với dùng `Any`?**
<details><summary>Đáp án gợi ý</summary>

Generics cho phép viết hàm/kiểu làm việc với nhiều kiểu thông qua type parameter, kèm constraint để nói kiểu đó phải có khả năng gì. So với `Any`: giữ type safety (compiler kiểm tra, không cần ép kiểu `as!`), và hiệu năng tốt vì compiler có thể specialize — sinh bản riêng cho từng kiểu cụ thể, dùng static dispatch, không phải box giá trị. Toàn bộ collection của stdlib (`Array<Element>`, `Dictionary<Key, Value>`, `Optional`) đều là generic.
</details>

**Q2 `associatedtype` là gì? Khác generic parameter của struct thế nào?**
<details><summary>Đáp án gợi ý</summary>

`associatedtype` là placeholder kiểu bên trong protocol; kiểu conform sẽ quyết định nó là gì (thường compiler tự suy ra từ chữ ký method). Khác generic parameter: generic parameter do **người dùng** chọn khi dùng kiểu (`Stack<Int>`), còn associated type do **kiểu conform** quyết định — mỗi kiểu conform chỉ có một lựa chọn (`Array<Int>` có `Element == Int`). Ràng buộc được bằng `:` và `where`. Swift 5.7 có primary associated type (`protocol Container<Item>`) để viết `some Container<Int>`.
</details>

**Q3 ⭐ `some` và `any` khác nhau thế nào? Khi nào dùng cái nào?**
<details><summary>Đáp án gợi ý</summary>

`some P` là opaque type: đại diện cho **một kiểu cụ thể cố định** mà compiler biết nhưng ẩn với người dùng — giữ được thông tin kiểu, specialize được, static dispatch, không box. Ở vị trí tham số nó là cú pháp gọn của generic. `any P` là existential: một hộp có thể chứa **bất kỳ** kiểu nào conform, kiểu thật chỉ biết lúc runtime — linh hoạt (mảng trộn nhiều kiểu, trả kiểu khác nhau) nhưng tốn chi phí: existential container, có thể cấp phát heap, dynamic dispatch qua witness table, và hạn chế với API dùng `Self`/associated type. Mặc định dùng `some`/generic; dùng `any` khi thật sự cần tính không đồng nhất hoặc lưu dependency có thể thay thế.
</details>

**Q4 ⭐ Existential container là gì? Vì sao `any P` chậm hơn?**
<details><summary>Đáp án gợi ý</summary>

Giá trị `any P` được lưu trong một cấu trúc cố định: 3 word value buffer (giá trị ≤ 24 byte cất inline, lớn hơn thì cấp phát trên heap và buffer giữ con trỏ), một con trỏ type metadata và một con trỏ protocol witness table cho mỗi protocol — tổng 40 byte cho một protocol. Chậm hơn vì: có thể cấp phát heap + ARC, mọi lời gọi qua witness table (không inline), compiler không specialize được, và copy giá trị phải qua value witness table.
</details>

**Q5 ⭐ `var body: some View` — vì sao dùng `some` mà không dùng `any View`?**
<details><summary>Đáp án gợi ý</summary>

Kiểu thật của body là một kiểu generic lồng nhau rất dài do `@ViewBuilder` tạo ra (`VStack<TupleView<...>>`). `some View` cho phép không phải viết ra kiểu đó nhưng compiler vẫn biết chính xác — SwiftUI dựa vào kiểu tĩnh này để so sánh cấu trúc view (structural identity) và diff hiệu quả, đồng thời tránh chi phí existential. `any View` (hay `AnyView`) xoá thông tin kiểu → SwiftUI mất khả năng tối ưu diff.
</details>

**Q6 Type erasure là gì? Tự cài thế nào? Ví dụ trong framework?**
<details><summary>Đáp án gợi ý</summary>

Type erasure là bọc một giá trị có kiểu cụ thể (thường conform protocol có associated type) vào một wrapper generic chỉ lộ ra những kiểu cần thiết, giấu kiểu thật. Cài: struct `AnyX<Output>` có init generic `init<T: X>(_ base: T) where T.Output == Output`, lưu các method của base dưới dạng closure, rồi forward lời gọi. Ví dụ: `AnyPublisher`, `AnySequence`, `AnyHashable`, `AnyView`. Từ Swift 5.7, `any X<Output>` thay được nhiều trường hợp, nhưng type erasure vẫn hữu ích khi cần thêm hành vi (init từ closure) hoặc cần một kiểu cụ thể.
</details>

**Q7 ⭐ Swift xử lý lỗi thế nào? `try`, `try?`, `try!` khác nhau ra sao?**
<details><summary>Đáp án gợi ý</summary>

Hàm đánh dấu `throws` có thể ném một giá trị conform `Error` (thường là enum). Người gọi phải dùng `try` trong `do/catch` hoặc trong hàm cũng `throws`. `catch` hỗ trợ pattern matching (case cụ thể, `where`, `as`). `try?` biến lỗi thành `nil` — tiện nhưng mất thông tin lỗi. `try!` crash nếu có lỗi — chỉ dùng khi chắc chắn không lỗi. Error của Swift không phải exception tốn kém: nó là giá trị trả về qua một thanh ghi riêng, không unwind stack kiểu C++.
</details>

**Q8 `rethrows` là gì?**
<details><summary>Đáp án gợi ý</summary>

Đánh dấu hàm nhận closure throwing và chỉ ném lỗi khi chính closure đó ném. Nếu người gọi truyền closure không throw thì gọi hàm không cần `try`. `map`, `filter`, `forEach`, `sorted(by:)` đều `rethrows`. Hàm `rethrows` không được tự ném lỗi riêng ngoài lỗi từ closure.
</details>

**Q9 `Result` dùng khi nào khi đã có `throws` và async/await?**
<details><summary>Đáp án gợi ý</summary>

`Result` biến kết quả thành một **giá trị**: lưu được vào property, truyền qua nhiều tầng, đưa vào collection, dùng trong completion handler (code cũ, API callback), hoặc khi cần xử lý thành công/thất bại sau. Có `map`, `mapError`, `flatMap`, `get()` (chuyển lại thành throws) và `init(catching:)`. Code mới tuyến tính thì `async throws` gọn hơn.
</details>

**Q10 Typed throws (Swift 6) là gì? Có nên dùng mọi nơi không?**
<details><summary>Đáp án gợi ý</summary>

`func f() throws(MyError)` khai báo chính xác kiểu lỗi; trong `catch`, `error` có kiểu `MyError` nên switch exhaustive, không cần cast, không box existential. Không nên dùng mọi nơi: nó khoá cứng API — thêm loại lỗi mới hoặc gọi API ném lỗi khác là phải đổi signature (breaking change). Khuyến nghị: mặc định `throws` thường; typed throws cho module nội bộ, miền lỗi thật sự đóng, hoặc code generic cần chuyển nguyên kiểu lỗi (như `rethrows` tổng quát).
</details>

**Q11 ⭐ `defer` chạy khi nào? Nhiều `defer` thì thứ tự thế nào?**
<details><summary>Đáp án gợi ý</summary>

Chạy ngay trước khi thoát scope chứa nó, bất kể thoát bằng `return`, `throw`, `break` hay hết block. Nhiều `defer` chạy theo thứ tự ngược với thứ tự khai báo (LIFO). Dùng để dọn dẹp chắc chắn: unlock, đóng file, tắt loading, gọi completion. Trong `defer` không được `return` hay `throw` ra ngoài.
</details>

**Q12 ⭐ Property wrapper là gì? Compiler biến đổi nó thế nào?**
<details><summary>Đáp án gợi ý</summary>

Là kiểu đánh dấu `@propertyWrapper` có `wrappedValue`, dùng để tái sử dụng logic get/set cho property. Khi viết `@Clamped(0...100) var health = 150`, compiler sinh một stored property ẩn `_health = Clamped(wrappedValue: 150, 0...100)` và một computed `health` forward get/set tới `_health.wrappedValue`. Nếu có `projectedValue` thì `$health` trả về nó (như `$text` trả `Binding` trong SwiftUI). Ví dụ: `@State`, `@Published`, `@AppStorage`.
</details>

**Q13 Result builder là gì? Liên quan gì tới SwiftUI?**
<details><summary>Đáp án gợi ý</summary>

Là kiểu `@resultBuilder` định nghĩa các hàm static `buildBlock`, `buildOptional`, `buildEither`, `buildArray`, `buildExpression`… Compiler biến các câu lệnh trong closure được đánh dấu thành lời gọi các hàm đó để tạo ra một giá trị duy nhất. `@ViewBuilder` của SwiftUI là result builder: liệt kê nhiều view trong `body` thành `TupleView`, `if/else` thành `_ConditionalContent`. Dùng để tạo DSL khai báo.
</details>

**Q14 ⭐ Các mức access control trong Swift? `open` khác `public` thế nào?**
<details><summary>Đáp án gợi ý</summary>

Từ rộng tới hẹp: `open`, `public`, `package`, `internal` (mặc định), `fileprivate`, `private`. `public` cho module khác dùng nhưng không cho subclass/override; `open` (chỉ cho class và member) cho phép cả subclass và override ngoài module. `package` giới hạn trong các module cùng một Swift package. `private` giới hạn trong scope khai báo và extension của nó trong cùng file; `fileprivate` trong cả file. Có thể tách mức đọc/ghi: `private(set)`. Mức truy cập hẹp còn giúp compiler suy ra `final` để tối ưu dispatch.
</details>

**Q15 ⭐ Làm sao decode JSON có key snake_case, field có thể null hoặc thiếu?**
<details><summary>Đáp án gợi ý</summary>

Key: dùng `decoder.keyDecodingStrategy = .convertFromSnakeCase`, hoặc khai `CodingKeys` map từng key (không dùng cả hai cùng lúc cho cùng key). Field null/thiếu: khai báo property là Optional — synthesized decoder dùng `decodeIfPresent`. Muốn giá trị mặc định thay vì nil (ví dụ mảng rỗng) thì viết `init(from:)` với `decodeIfPresent(...) ?? default`. Key thừa trong JSON bị bỏ qua.
</details>

**Q16 Khi nào phải viết `init(from:)` thủ công?**
<details><summary>Đáp án gợi ý</summary>

Khi cấu trúc JSON khác cấu trúc model: làm phẳng object lồng nhau (`nestedContainer`), giá trị mặc định cho field thiếu, một field có nhiều kiểu (number hoặc string), validate dữ liệu và ném `DecodingError.dataCorruptedError`, enum có fallback `unknown`, hoặc biến đổi giá trị (string → URL, timestamp → Date theo cách riêng).
</details>

**Q17 Có những `DecodingError` nào? Debug lỗi decode thế nào?**
<details><summary>Đáp án gợi ý</summary>

`keyNotFound` (thiếu key bắt buộc), `typeMismatch` (sai kiểu, ví dụ "10" thay vì 10), `valueNotFound` (null cho field non-optional), `dataCorrupted` (JSON hỏng, hoặc giá trị không hợp lệ như date sai format). Mỗi lỗi có `context.codingPath` cho biết chính xác vị trí (ví dụ `["items", "Index 3", "owner", "id"]`) và `debugDescription`. Khi debug in đầy đủ error thay vì `localizedDescription`; trong app thật nên log codingPath để phát hiện server đổi API.
</details>

**Q18 Decode ngày tháng thế nào? Bẫy hay gặp?**
<details><summary>Đáp án gợi ý</summary>

Đặt `dateDecodingStrategy`: `.iso8601`, `.secondsSince1970`, `.millisecondsSince1970`, `.formatted(DateFormatter)` hoặc `.custom`. Bẫy: mặc định `.deferredToDate` hiểu số là giây kể từ 2001 chứ không phải 1970; `DateFormatter` phải đặt `locale = en_US_POSIX` và `timeZone` rõ ràng, nếu không máy dùng lịch khác/12h sẽ parse sai; `.iso8601` mặc định không nhận phần giây thập phân — cần `.custom` với `ISO8601DateFormatter` có `.withFractionalSeconds`; `DateFormatter` tạo tốn kém nên tái sử dụng.
</details>

**Q19 Server thêm giá trị enum mới thì app cũ crash/hỏng. Phòng tránh thế nào?**
<details><summary>Đáp án gợi ý</summary>

Enum raw value tự sinh `Decodable` sẽ ném `dataCorrupted` với giá trị lạ, làm hỏng decode cả response. Phòng tránh: thêm case `unknown` (hoặc `unknown(String)`) và tự viết `init(from:)` dùng `singleValueContainer`, fallback về `unknown` khi `init(rawValue:)` trả nil. Ở UI thì xử lý `unknown` một cách an toàn (ẩn đi, hiển thị mặc định).
</details>

**Q20 Vì sao generic function có thể nhanh hơn hàm nhận `any Protocol`?**
<details><summary>Đáp án gợi ý</summary>

Với generic, compiler biết kiểu cụ thể tại nơi gọi và có thể specialize: sinh bản hàm riêng cho kiểu đó, gọi method trực tiếp, inline, giữ giá trị trên stack không box. Với `any Protocol`, giá trị nằm trong existential container (có thể heap), mỗi lời gọi tra witness table và compiler không thể inline. Khi không specialize được (khác module, không `@inlinable`), generic vẫn truyền metadata + witness table ngầm nhưng không cần box giá trị.
</details>

---

## 🏁 Milestone M1 — Bài tự kiểm tra (CN 06/12)

> M1: *"Giải thích value vs reference, closure capture, protocol vs class inheritance, generics **không cần nhìn tài liệu**."*
> Làm trong **60 phút**, tắt tài liệu, tắt AI. Ghi âm phần A, viết tay/gõ phần B.

**Phần A — Nói (mỗi câu tối đa 2 phút, ghi âm, nghe lại và tự chấm)**
1. Struct khác class thế nào? Struct chứa class thì sao? Copy-on-write hoạt động ra sao? *(tuần 2)*
2. Closure capture biến thế nào? Capture list, `@escaping`, `[weak self]` để làm gì? *(tuần 3)*
3. Protocol + extension vs kế thừa class: khi nào dùng cái nào? Bẫy method chỉ có trong protocol extension? *(tuần 3)*
4. Generics, `some` và `any` khác nhau thế nào, chi phí ra sao? *(tuần 4)*
5. Optional là gì bên dưới, `map`/`flatMap` trên Optional? *(tuần 1)*

**Phần B — Code (không chạy thử cho đến khi viết xong cả 3 bài)**
1. *(10')* Dự đoán output và giải thích:
   ```swift
   struct S { var n = 0 }
   final class C { var n = 0 }
   var s = S(); let c = C()
   var closures: [() -> Void] = []
   closures.append { [s] in print("s:", s.n) }
   closures.append { print("c:", c.n) }
   s.n = 5; c.n = 5
   closures.forEach { $0() }
   ```
2. *(20')* Viết `struct Stack<Element>` có `push`, `pop`, `peek`; thêm `extension Stack where Element: Equatable` có `contains`; cho `Stack` conform `CustomStringConvertible` khi `Element: CustomStringConvertible`.
3. *(20')* Cho JSON `{"total_count": 1, "items": [{"id": 7, "full_name": "a/b", "owner": {"login": "a"}, "pushed_at": "2026-12-01T10:00:00Z"}]}` — viết model + decode, in `fullName`, `owner.login`, năm của `pushedAt`. Xử lý lỗi bằng `do/catch` in ra `codingPath`.

<details><summary>Đáp án Phần B.1</summary>

```
s: 0
c: 5
```
`[s]` chụp **bản sao** của struct lúc tạo closure (n = 0). Closure thứ hai capture **hằng** `c` — một tham chiếu tới object, nên thấy `n` đã đổi thành 5.
</details>

**Tiêu chí đạt M1**
- [ ] Phần A: ≥ 4/5 câu nói trôi chảy, không ấp úng quá 5 giây, không sai kiến thức
- [ ] Phần B.1 đúng và giải thích được
- [ ] Phần B.2 và B.3 compile được sau tối đa 2 lần sửa lỗi, chạy đúng
- [ ] Trả lời được ≥ 15/20 câu phỏng vấn tuần 4 trước khi mở đáp án

**Chưa đạt?** Ghi rõ câu nào yếu → T2–T3 tuần sau (07–08/12) đọc lại đúng § đó và làm lại bài tập liên quan **trước khi** bắt đầu tuần 5. Tuần 13 (Tết) sẽ ôn lại toàn bộ bằng flashcard.

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Làm xong Bài tập lớn `apiclient.swift`
- [ ] Trả lời được ≥ 15/20 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "`some` vs `any`, existential container"
- [ ] Ghi âm 2 phút: "Decode một JSON phức tạp: các công cụ Codable tôi dùng"
- [ ] Tự viết lại type erasure `AnyLoader` từ trí nhớ
- [ ] Làm xong 6 bài DSA tuần 4 (Sliding Window)
- [ ] Side project Milestone 4 xong (decode JSON từ file trong bundle, error enum, `APIClient` generic skeleton)
- [ ] 🏁 **Đạt Milestone M1** (hoặc đã lên kế hoạch bù cụ thể)
- [ ] Viết 5 dòng tự nhận xét về cả Giai đoạn 1: phần nào vẫn còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)
- [The Swift Programming Language — Opaque and Boxed Protocol Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/)
- [The Swift Programming Language — Error Handling](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [The Swift Programming Language — Access Control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)
- [The Swift Programming Language — Properties: Property Wrappers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers)
- [The Swift Programming Language — Advanced Operators / Result Builders](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/advancedoperators/#Result-Builders)
- Apple Developer — [Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/encoding-and-decoding-custom-types)
- WWDC 2022 — *Embrace Swift generics* và *Design protocol interfaces in Swift* (bắt buộc xem cho `some`/`any`)
- WWDC 2016 — *Understanding Swift Performance* (phần existential container — xem lại)
- WWDC 2019 — *Modern Swift API Design* (phần property wrapper)
- Swift Evolution — [SE-0413 Typed throws](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md), [SE-0335 Introduce existential `any`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)
- Swift by Sundell — *Type erasure using closures*, *Customizing Codable types in Swift*
