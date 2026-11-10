# Tuần 1 — Swift nền tảng (09/11 – 15/11/2026)

> **Giai đoạn 1 · Swift Core** · Thời lượng: ~14 giờ
> Tuần này "khởi động lại máy". Kiến thức dễ nhưng **người phỏng vấn hay hỏi xoáy ở chính những chỗ dễ** (Optional, String, inout, switch). Đừng lướt.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích `let` vs `var`, type inference, type safety của Swift
- [ ] Giải thích **Optional thực chất là gì** (một `enum`) và 5 cách unwrap, khi nào dùng cách nào
- [ ] Giải thích tại sao `String` trong Swift **không truy cập được bằng `str[0]`**
- [ ] Biết độ phức tạp các thao tác cơ bản của `Array`, `Dictionary`, `Set`
- [ ] Viết `switch` với pattern matching (tuple, range, `where`, `case let`)
- [ ] Giải thích `inout` hoạt động thế nào (copy-in copy-out)
- [ ] Làm xong 6 bài DSA tuần 1 (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 09/11** | 2h | Setup (tạo `practice/week01/`, tắt AI completion). Đọc §1 Biến, hằng, kiểu dữ liệu. Làm Bài tập 1 | `day1.swift` chạy được |
| **T3 10/11** | 2h | §2 **Optional** (đọc kỹ, đây là phần quan trọng nhất tuần). Bài tập 2, 3 | `day2.swift` |
| **T4 11/11** | 2h | §3 **String & Character**. Bài tập 4 · DSA: Valid Anagram | `day3.swift` |
| **T5 12/11** | 2h | §4 **Collection**: Array, Dictionary, Set. Bài tập 5, 6 · DSA: Two Sum | `day4.swift` |
| **T6 13/11** | 1,5h | §5 Control flow + `switch` pattern matching. Bài tập 7 · DSA: Contains Duplicate | `day5.swift` |
| **T7 14/11** | 3,5h | §6 **Hàm**: label, default, `inout`, variadic, function type. Bài tập 8 + Bài tập lớn. Side project: Milestone 1 | `day6.swift`, repo side project |
| **CN 15/11** | 2h | Tự trả lời 15 câu phỏng vấn (viết ra giấy trước, rồi mới mở đáp án). Nói lại 3 câu bằng voice memo. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. Biến, hằng, kiểu dữ liệu

```swift
let maxRetry = 3          // hằng: gán 1 lần, không đổi được
var count = 0             // biến: đổi được
count += 1

let pi: Double = 3.14     // khai báo kiểu tường minh
let name = "Tuấn"         // type inference: Swift tự suy ra String
```

**Điểm cần nhớ:**
- Swift là ngôn ngữ **type-safe** và **statically typed**: kiểu được kiểm tra lúc compile, không tự ép kiểu ngầm.
  ```swift
  let a: Int = 5
  let b: Double = 2.5
  // let c = a + b        // ❌ lỗi compile: không cộng Int với Double
  let c = Double(a) + b   // ✅ phải ép tường minh
  ```
- **Luôn ưu tiên `let`**. Xcode sẽ cảnh báo nếu bạn khai `var` mà không đổi. Lý do: code dễ suy luận hơn, an toàn hơn trong đa luồng, compiler tối ưu tốt hơn.
- `let` với **class** (reference type) chỉ khoá *tham chiếu*, không khoá *nội dung object*. Với **struct** (value type) thì khoá toàn bộ. (Sẽ học sâu ở tuần 2, nhưng nhớ trước câu này.)
  ```swift
  class Box { var value = 0 }
  let box = Box()
  box.value = 10       // ✅ được: đổi nội dung object
  // box = Box()       // ❌ không được: đổi tham chiếu

  struct Point { var x = 0 }
  let p = Point()
  // p.x = 10          // ❌ không được
  ```
- Các kiểu cơ bản: `Int` (64-bit trên máy 64-bit), `Double` (64-bit, mặc định cho số thực), `Float` (32-bit), `Bool`, `String`, `Character`.
- **Integer overflow là crash**, không âm thầm tràn như C:
  ```swift
  var x = Int.max
  // x += 1          // 💥 runtime crash: arithmetic overflow
  x = x &+ 1         // dùng toán tử &+ nếu thực sự muốn wrap-around → Int.min
  ```
- `typealias` đặt tên khác cho kiểu: `typealias UserID = String`.
- **Tuple**: gom nhiều giá trị, hay dùng để trả nhiều giá trị từ hàm.
  ```swift
  let http = (code: 404, message: "Not Found")
  print(http.code, http.message)
  let (code, _) = http   // destructuring, bỏ qua phần tử thứ 2
  ```

### §2. Optional — phần quan trọng nhất tuần

**Optional nghĩa là "có thể có giá trị, có thể không có (`nil`)"**. Trong Swift, một biến kiểu `String` **không bao giờ** được là `nil`. Muốn có `nil` phải khai `String?`.

**Bản chất:** `Optional` là một `enum` generic trong thư viện chuẩn:
```swift
enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}
// String? chỉ là cách viết tắt của Optional<String>
// nil chỉ là cách viết tắt của Optional.none
```
→ Đây là câu trả lời ăn điểm khi bị hỏi "Optional là gì?".

**5 cách lấy giá trị ra (unwrap):**

```swift
let input: String? = "42"

// 1. Optional binding: if let
if let input {                      // Swift 5.7+: viết tắt của if let input = input
    print("Có giá trị: \(input)")
}

// 2. guard let — thoát sớm, biến dùng được ở phần còn lại của hàm
func parse(_ text: String?) -> Int {
    guard let text, let number = Int(text) else {
        return 0                    // bắt buộc phải thoát (return/throw/break/continue)
    }
    return number                   // text và number dùng được ở đây
}

// 3. Nil-coalescing ?? — giá trị mặc định
let name: String? = nil
let displayName = name ?? "Khách"

// 4. Optional chaining ?. — gọi tiếp, nếu nil ở đâu thì cả chuỗi nil
let length = name?.count            // kiểu Int?, không phải Int

// 5. Force unwrap ! — crash nếu nil. CHỈ dùng khi chắc chắn 100%
let url = URL(string: "https://apple.com")!   // chấp nhận được: chuỗi hằng, chắc chắn hợp lệ
```

**Khi nào dùng cái nào?**
| Tình huống | Dùng |
|---|---|
| Thiếu giá trị thì hàm không làm tiếp được | `guard let` (giảm lồng nhau, "early exit") |
| Chỉ cần xử lý khi có giá trị, còn lại bỏ qua | `if let` |
| Có giá trị mặc định hợp lý | `??` |
| Truy cập thuộc tính lồng sâu | `?.` |
| Lỗi lập trình nếu nil (IBOutlet, URL hằng) | `!` |

**`map` và `flatMap` trên Optional** (hay bị hỏi ở mức 2 năm):
```swift
let text: String? = "5"
let doubled = text.map { $0 + $0 }          // Optional("55")
let number = text.flatMap { Int($0) }       // Optional(5) — Int(...) trả Int?, flatMap làm phẳng
let wrong = text.map { Int($0) }            // Optional(Optional(5)) — Int?? lồng 2 lớp
```

**Implicitly Unwrapped Optional `String!`**: vẫn là Optional, nhưng tự unwrap khi dùng (crash nếu nil). Gặp nhiều ở `@IBOutlet weak var label: UILabel!` vì outlet được gán sau `init` nhưng trước khi bạn dùng.

### §3. String & Character

```swift
let s = "Xin chào 👋🏽"
print(s.count)                 // 10 — đếm theo ký tự người đọc thấy
print(s.utf8.count)            // 18 — số byte UTF-8 ("à" 2 byte, 👋🏽 8 byte)
// s[0]                        // ❌ lỗi compile!
let first = s[s.startIndex]                        // "X"
let third = s[s.index(s.startIndex, offsetBy: 2)]  // "n"
```

**Tại sao không có `s[0]`?** Vì `Character` trong Swift là một **extended grapheme cluster** — một "ký tự người đọc thấy" có thể gồm nhiều Unicode scalar (ví dụ `👋🏽` = bàn tay + màu da; `é` có thể là 1 hoặc 2 scalar). Mỗi ký tự chiếm số byte khác nhau, nên muốn đến ký tự thứ n phải **đi từ đầu** → O(n). Swift cố tình không cho cú pháp `s[0]` để bạn không tưởng nhầm nó là O(1).

→ Hệ quả: `s.count` cũng là **O(n)**. Trong vòng lặp nặng, nếu cần truy cập theo vị trí nhiều lần, chuyển sang `Array(s)` trước.

**Thao tác hay dùng:**
```swift
import Foundation                 // cần cho trimmingCharacters(in:)
let email = "  Tuan@Example.com "
let clean = email.trimmingCharacters(in: .whitespaces).lowercased()
clean.hasPrefix("tuan")           // true
clean.contains("@")               // true
let parts = clean.split(separator: "@")   // ["tuan", "example.com"] — kiểu [Substring]
let user = String(parts[0])       // Substring → String

let multi = """
    Dòng 1
    Dòng 2
    """
```

**`Substring`** dùng chung bộ nhớ với String gốc (không copy) → nhanh, nhưng **giữ cả String gốc trong bộ nhớ**. Không nên lưu `Substring` lâu dài; chuyển sang `String(...)`.

**String là value type** nhưng dùng **copy-on-write** — gán `let b = a` không copy ngay, chỉ copy khi một bên bị sửa. (Học sâu ở tuần 2.)

### §4. Collection: Array, Dictionary, Set

Cả 3 đều là **struct (value type)** với **copy-on-write**.

```swift
// Array — có thứ tự, cho phép trùng
var numbers = [3, 1, 2]
numbers.append(4)            // O(1) trung bình (amortized)
numbers.insert(0, at: 0)     // O(n) — phải dịch các phần tử
numbers.remove(at: 0)        // O(n)
numbers.contains(2)          // O(n)
let sorted = numbers.sorted() // O(n log n), trả mảng mới
numbers.first                // Int? — an toàn
// numbers[10]               // 💥 crash: index out of range

// Dictionary — key–value, không thứ tự, key phải Hashable
var ages: [String: Int] = ["An": 25, "Bình": 30]
ages["Chi"] = 28             // thêm / sửa: O(1) trung bình
let age = ages["Dũng"]       // Int? — key không tồn tại → nil, KHÔNG crash
let safeAge = ages["Dũng", default: 0]
ages["An"] = nil             // xoá key
for (name, age) in ages { print(name, age) }   // thứ tự không đảm bảo!

// Set — không thứ tự, không trùng, phần tử phải Hashable
var tags: Set = ["swift", "ios"]
tags.insert("swift")         // không thêm vì đã có
tags.contains("ios")         // O(1) trung bình — nhanh hơn Array nhiều
let a: Set = [1, 2, 3], b: Set = [2, 3, 4]
a.intersection(b)            // [2, 3]
a.union(b)                   // [1, 2, 3, 4]
a.subtracting(b)             // [1]
```

**Bảng độ phức tạp cần thuộc:**
| Thao tác | Array | Dictionary | Set |
|---|---|---|---|
| Truy cập theo index/key | O(1) | O(1)* | — |
| Tìm kiếm `contains` | O(n) | O(1)* (theo key) | O(1)* |
| Thêm cuối / insert | O(1)** / O(n) | O(1)* | O(1)* |
| Xoá | O(n) | O(1)* | O(1)* |

\* trung bình, xấu nhất O(n) khi hash va chạm nhiều · \** amortized: thỉnh thoảng phải cấp phát lại vùng nhớ gấp đôi

**`Hashable`** là gì? Kiểu có thể tính ra một giá trị băm (`hashValue`) và so sánh `==`. Quy tắc: **nếu `a == b` thì `hash(a) == hash(b)`** (ngược lại không cần). Struct mà mọi thuộc tính đều Hashable thì chỉ cần khai `: Hashable`, compiler tự sinh code.

**Hàm hay dùng (sẽ học sâu ở tuần 3):**
```swift
let prices = [100, 250, 80]
prices.map { $0 * 2 }            // [200, 500, 160]
prices.filter { $0 > 90 }        // [100, 250]
prices.reduce(0, +)              // 430
prices.min(), prices.max()       // Optional
Dictionary(grouping: ["an", "anh", "bình"], by: { $0.first! })  // ["a": ["an","anh"], "b": ["bình"]]
```

### §5. Control flow & switch

```swift
for i in 0..<3 { }            // 0,1,2   (half-open range)
for i in 0...3 { }            // 0,1,2,3 (closed range)
for i in stride(from: 10, to: 0, by: -2) { }   // 10,8,6,4,2
for (index, value) in ["a", "b"].enumerated() { }
while count < 10 { count += 1 }
repeat { count -= 1 } while count > 0          // chạy ít nhất 1 lần
```

**`switch` trong Swift mạnh hơn nhiều so với C/Java:**
- **Phải exhaustive** (bao hết mọi trường hợp), nếu không có `default` → lỗi compile.
- **Không tự rơi xuống case sau** (không cần `break`). Muốn rơi xuống thì dùng `fallthrough` (hiếm khi dùng).
- Match được range, tuple, enum có associated value, điều kiện `where`.

```swift
let point = (2, -2)
switch point {
case (0, 0):
    print("Gốc toạ độ")
case (_, 0):
    print("Trên trục x")
case (let x, let y) where x == y:
    print("Trên đường y = x")
case (let x, let y) where x == -y:
    print("Trên đường y = -x")        // ← in ra dòng này
default:
    print("Chỗ khác")
}

let statusCode = 404
switch statusCode {
case 200..<300: print("Thành công")
case 400, 404:  print("Lỗi client")     // nhiều giá trị trong 1 case
case 500...:    print("Lỗi server")
default:        print("Khác")
}
```

`if case` / `guard case` dùng pattern matching với 1 trường hợp:
```swift
enum Result { case success(Int), failure(String) }
let r = Result.success(10)
if case .success(let value) = r, value > 5 { print(value) }
```

### §6. Hàm

```swift
// Argument label (bên ngoài) và parameter name (bên trong)
func send(message text: String, to user: String) {
    print("Gửi '\(text)' tới \(user)")
}
send(message: "Hi", to: "An")

// _ bỏ label
func square(_ x: Int) -> Int { x * x }    // 1 biểu thức → không cần return
square(4)

// Default parameter
func greet(_ name: String, greeting: String = "Xin chào") -> String {
    "\(greeting), \(name)"
}
greet("An")                        // "Xin chào, An"

// Variadic
func sum(_ numbers: Int...) -> Int { numbers.reduce(0, +) }
sum(1, 2, 3)

// Trả nhiều giá trị bằng tuple, có thể Optional
func minMax(_ a: [Int]) -> (min: Int, max: Int)? {
    guard let first = a.first else { return nil }
    var lo = first, hi = first
    for x in a.dropFirst() { lo = min(lo, x); hi = max(hi, x) }
    return (lo, hi)
}
```

**`inout`** — cho phép hàm sửa biến truyền vào:
```swift
func double(_ x: inout Int) { x *= 2 }
var n = 5
double(&n)        // n == 10 — phải có dấu &
```
Cơ chế: **copy-in copy-out** — khi gọi, giá trị được copy vào hàm; khi hàm kết thúc, giá trị được copy ngược lại biến gốc. (Compiler có thể tối ưu thành truyền địa chỉ, nhưng về ngữ nghĩa là copy-in copy-out.)
Quy tắc: không truyền `let`, không truyền literal, không truyền cùng một biến vào 2 tham số `inout` (lỗi *exclusive access*).

**Hàm là "first-class citizen"** — có kiểu, gán được vào biến, truyền làm tham số:
```swift
let op: (Int, Int) -> Int = (+)
func apply(_ f: (Int) -> Int, to x: Int) -> Int { f(x) }
apply(square, to: 3)       // 9
```

---

## ⚠️ Bẫy hay gặp

1. **`let` với class vẫn sửa được thuộc tính** — đừng nói "let là bất biến" một cách chung chung.
2. **`array.first` trả Optional, `array[0]` thì crash nếu rỗng.** Phỏng vấn viết code trên giấy hay bị trừ điểm vì `array[0]` mà không check rỗng.
3. **`dict[key]` trả Optional** — nên `dict[key]?.append(x)` sẽ **không làm gì** nếu key chưa có. Dùng `dict[key, default: []].append(x)`.
4. **`s.count` của String là O(n)** — đừng gọi trong điều kiện vòng lặp chạy nhiều lần.
5. **Thứ tự duyệt Dictionary/Set không đảm bảo**, thậm chí khác nhau giữa các lần chạy.
6. **`guard` bắt buộc thoát scope** trong nhánh `else`.
7. **Optional so sánh được với nil, và `nil < 5` không compile** — `Optional<Int>` không `Comparable`.
8. **`==` vs `===`**: `==` so sánh giá trị (Equatable), `===` so sánh **cùng một instance** (chỉ dùng cho class).

---

## 🛠 Bài tập

> Tạo file trong `practice/week01/` và chạy bằng `swift tên_file.swift`. **Không xem lời giải trước khi tự làm ít nhất 20 phút.**

**Bài 1 — Đổi nhiệt độ (T2)**
Viết hàm `toFahrenheit(_ c: Double) -> Double` và `describe(_ c: Double) -> String` trả `"Lạnh"` (< 15), `"Mát"` (15–25), `"Nóng"` (> 25). In kết quả cho `[-5, 15, 20, 30.5]`.

**Bài 2 — Parse an toàn (T3)**
Cho `let raw = ["10", "abc", nil, "7", "", "-3"]` kiểu `[String?]`. Tính tổng các số hợp lệ. Viết **2 cách**: dùng vòng `for` + `if let`, và dùng `compactMap`.
<details><summary>Lời giải</summary>

```swift
let raw: [String?] = ["10", "abc", nil, "7", "", "-3"]
// Cách 1
var total = 0
for item in raw {
    if let item, let n = Int(item) { total += n }
}
// Cách 2
let total2 = raw.compactMap { $0 }.compactMap { Int($0) }.reduce(0, +)
// hoặc: raw.compactMap { $0.flatMap(Int.init) }.reduce(0, +)
print(total, total2)   // 14 14
```
</details>

**Bài 3 — Optional chaining (T3)**
```swift
struct Address { var city: String? }
struct User { var name: String; var address: Address? }
let users: [User] = [
    User(name: "An", address: Address(city: "Hà Nội")),
    User(name: "Bình", address: nil),
    User(name: "Chi", address: Address(city: nil)),
]
```
In ra `"<tên>: <thành phố hoặc 'Chưa rõ'>"` cho mỗi user, trong **1 dòng code** cho phần lấy thành phố.
<details><summary>Lời giải</summary>

```swift
for u in users { print("\(u.name): \(u.address?.city ?? "Chưa rõ")") }
```
</details>

**Bài 4 — String (T4)**
a) `isPalindrome(_ s: String) -> Bool`, bỏ qua hoa/thường và ký tự không phải chữ/số. `"A man, a plan, a canal: Panama"` → `true`.
b) `reverseWords("  xin   chào thế giới ")` → `"giới thế chào xin"`.
c) Đếm số lần xuất hiện mỗi ký tự trong `"hello world"`, in theo thứ tự chữ cái.
<details><summary>Lời giải</summary>

```swift
func isPalindrome(_ s: String) -> Bool {
    let chars = Array(s.lowercased().filter { $0.isLetter || $0.isNumber })
    var i = 0, j = chars.count - 1
    while i < j {
        if chars[i] != chars[j] { return false }
        i += 1; j -= 1
    }
    return true
}

func reverseWords(_ s: String) -> String {
    s.split(separator: " ").reversed().joined(separator: " ")
}

var freq: [Character: Int] = [:]
for ch in "hello world" where ch != " " { freq[ch, default: 0] += 1 }
for key in freq.keys.sorted() { print(key, freq[key]!) }
```
Chú ý `split` mặc định bỏ phần tử rỗng → xử lý được nhiều dấu cách liền nhau.
</details>

**Bài 5 — Dictionary (T5)**
Cho danh sách đơn hàng `[(customer: String, amount: Int)]`. Trả về Dictionary tổng tiền theo khách, và tên khách chi nhiều nhất.
<details><summary>Lời giải</summary>

```swift
let orders = [("An", 100), ("Bình", 50), ("An", 70), ("Chi", 200)]
var totals: [String: Int] = [:]
for (c, a) in orders { totals[c, default: 0] += a }
let top = totals.max { $0.value < $1.value }?.key
print(totals, top ?? "-")
```
</details>

**Bài 6 — Set (T5)**
Hai bạn có danh sách bài hát yêu thích. Tìm: bài cả hai cùng thích, bài chỉ A thích, tổng số bài khác nhau. Giải thích vì sao dùng Set mà không dùng Array.

**Bài 7 — Switch (T6)**
Viết `classify(age: Int?, isStudent: Bool) -> String` dùng **một** `switch` trên tuple `(age, isStudent)`:
- `nil` → "Không rõ" · `< 0` → "Không hợp lệ" · `0...5` → "Miễn phí" · `6...22` + student → "Vé SV" · `>= 60` → "Vé người cao tuổi" · còn lại → "Vé thường".
<details><summary>Lời giải</summary>

```swift
func classify(age: Int?, isStudent: Bool) -> String {
    switch (age, isStudent) {
    case (nil, _):                         return "Không rõ"
    case (let a?, _) where a < 0:          return "Không hợp lệ"
    case (.some(0...5), _):                return "Miễn phí"
    case (.some(6...22), true):            return "Vé SV"
    case (let a?, _) where a >= 60:        return "Vé người cao tuổi"
    default:                               return "Vé thường"
    }
}
```
- `let a?` là pattern "Optional có giá trị, bind vào a" (viết tắt của `.some(let a)`).
- **Bẫy:** viết `case (0...5, _)` sẽ **lỗi compile** — range `ClosedRange<Int>` không match được với `Int?`. Phải bọc vào `.some(0...5)` vì `age` là Optional.
</details>

**Bài 8 — inout & function type (T7)**
a) Viết `swapValues(_ a: inout Int, _ b: inout Int)`. Thử gọi `swapValues(&x, &x)` — đọc thông báo lỗi và giải thích.
b) Viết `func makeCounter() -> () -> Int` trả về hàm mà mỗi lần gọi tăng 1. (Đây là closure — tuần 3 học kỹ, giờ chỉ cần chạy được.)

**🏋️ Bài tập lớn (T7) — Quản lý điểm sinh viên (console)**
Viết file `grades.swift`:
- `typealias Student = (name: String, scores: [Double])`
- Hàm tính điểm trung bình (trả `Double?` nếu không có điểm)
- Hàm xếp loại bằng `switch` trên range: Giỏi ≥ 8, Khá ≥ 6.5, TB ≥ 5, Yếu
- Nhóm sinh viên theo xếp loại → `[String: [String]]`
- In top 3 theo điểm trung bình, bỏ qua sinh viên không có điểm
- Dùng ít nhất: `guard let`, `??`, `compactMap`, `sorted(by:)`, `Dictionary(grouping:by:)`, `switch`

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ `let` và `var` khác nhau thế nào? `let` với class có bất biến không?**
<details><summary>Đáp án gợi ý</summary>

`let` là hằng, chỉ gán một lần; `var` là biến. Với **value type** (struct, enum), `let` làm toàn bộ giá trị bất biến, kể cả thuộc tính bên trong. Với **class**, `let` chỉ cố định *tham chiếu* — vẫn sửa được các thuộc tính `var` của object. Nên ưu tiên `let` vì an toàn, dễ suy luận, thread-safe hơn và compiler tối ưu tốt hơn.
</details>

**Q2 ⭐ Optional là gì? Được cài đặt như thế nào?**
<details><summary>Đáp án gợi ý</summary>

Optional biểu diễn việc một giá trị có thể vắng mặt. Nó là một `enum Optional<Wrapped>` với 2 case `.none` và `.some(Wrapped)`. `nil` là `.none`. Nhờ vậy compiler buộc ta xử lý trường hợp nil một cách tường minh, tránh lỗi null pointer kiểu Objective-C/Java.
</details>

**Q3 ⭐ Các cách unwrap Optional? Khi nào dùng `guard let` thay vì `if let`?**
<details><summary>Đáp án gợi ý</summary>

`if let`, `guard let`, `??`, optional chaining `?.`, force unwrap `!`, ngoài ra `switch`/`if case let x?`, `map`/`flatMap`. Dùng `guard let` khi giá trị là **điều kiện tiên quyết** để hàm chạy tiếp — thoát sớm, tránh "kim tự tháp" lồng nhau và biến unwrap dùng được ở phần còn lại của scope. Dùng `if let` khi chỉ một nhánh logic cần giá trị.
</details>

**Q4 Khi nào được dùng force unwrap `!`?**
<details><summary>Đáp án gợi ý</summary>

Khi nil nghĩa là **lỗi lập trình** chứ không phải tình huống runtime hợp lệ: `URL(string:)` với chuỗi hằng, `@IBOutlet`, `dequeueReusableCell` cast sang cell đã đăng ký... Crash sớm giúp phát hiện bug. Không dùng cho dữ liệu từ network, user input, file. Có thể thay `!` bằng `guard ... else { fatalError("lý do") }` để có thông báo rõ hơn.
</details>

**Q5 Implicitly unwrapped optional (`String!`) là gì? Vì sao `@IBOutlet` hay dùng nó?**
<details><summary>Đáp án gợi ý</summary>

Là Optional nhưng được tự động unwrap khi sử dụng; nếu nil thì crash. IBOutlet được gán bởi storyboard **sau** `init` nhưng **trước** `viewDidLoad`, nên không thể là non-optional (init chưa có giá trị), nhưng khi dùng thì luôn có → `!` tiện hơn `?`.
</details>

**Q6 `map` và `flatMap` trên Optional khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`map` áp hàm `(Wrapped) -> U` và trả `U?`. `flatMap` áp hàm `(Wrapped) -> U?` và trả `U?` (làm phẳng), tránh `U??`. Ví dụ `"5" as String?` → `.flatMap { Int($0) }` cho `Int?`, còn `.map` cho `Int??`.
</details>

**Q7 ⭐ Vì sao String trong Swift không cho `str[0]`?**
<details><summary>Đáp án gợi ý</summary>

Vì `Character` là extended grapheme cluster, có độ dài byte thay đổi (emoji, ký tự có dấu tổ hợp). Để tới ký tự thứ n phải duyệt từ đầu → O(n). Swift dùng `String.Index` để tránh ảo tưởng truy cập O(1). Cũng vì thế `count` là O(n). Khi cần truy cập ngẫu nhiên nhiều lần, chuyển sang `Array(str)`.
</details>

**Q8 `String` và `Substring` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`Substring` là một "lát cắt" dùng chung bộ nhớ với String gốc nên tạo ra rất rẻ, nhưng giữ String gốc sống. Phù hợp xử lý tạm; nếu lưu lâu dài nên chuyển sang `String` để giải phóng bộ nhớ gốc. Cả hai cùng conform `StringProtocol`.
</details>

**Q9 ⭐ So sánh Array, Set, Dictionary. Khi nào dùng Set thay Array?**
<details><summary>Đáp án gợi ý</summary>

Array có thứ tự, cho trùng, `contains` O(n). Set không thứ tự, không trùng, `contains`/`insert`/`remove` O(1) trung bình, phần tử phải Hashable. Dictionary là cặp key–value, tra theo key O(1) trung bình. Dùng Set khi cần kiểm tra tồn tại nhanh, loại trùng, hoặc phép toán tập hợp (giao, hợp, hiệu). Cả 3 là value type có copy-on-write.
</details>

**Q10 Hashable là gì? Quan hệ với Equatable?**
<details><summary>Đáp án gợi ý</summary>

`Hashable` kế thừa `Equatable` và yêu cầu `hash(into:)`. Quy tắc bắt buộc: hai giá trị `==` phải có cùng hash. Dùng làm key Dictionary và phần tử Set. Struct/enum có mọi thành phần Hashable thì compiler tự sinh conformance.
</details>

**Q11 Độ phức tạp của `append` trên Array? Vì sao gọi là "amortized O(1)"?**
<details><summary>Đáp án gợi ý</summary>

Array dùng vùng nhớ liên tục có `capacity`. Khi đầy, nó cấp phát vùng mới gấp đôi và copy sang → lần đó O(n). Nhưng vì capacity tăng gấp đôi, tổng chi phí n lần append là O(n) → trung bình O(1) mỗi lần. Có thể dùng `reserveCapacity(_:)` nếu biết trước kích thước.
</details>

**Q12 ⭐ `switch` của Swift khác C/Java ở điểm nào?**
<details><summary>Đáp án gợi ý</summary>

Phải exhaustive; không tự fallthrough (không cần `break`); match được range, tuple, enum associated value, `where`, type casting (`case let x as String`); mỗi case phải có ít nhất một câu lệnh.
</details>

**Q13 ⭐ `inout` hoạt động như thế nào?**
<details><summary>Đáp án gợi ý</summary>

Ngữ nghĩa copy-in copy-out: giá trị được copy vào tham số khi gọi, hàm sửa bản copy, khi hàm trả về thì giá trị được ghi ngược lại biến gốc. Compiler có thể tối ưu thành truyền tham chiếu. Phải truyền `var` kèm `&`. Swift áp dụng *Law of Exclusivity*: không được truy cập cùng một biến chồng chéo (ví dụ truyền một biến vào 2 tham số inout) → lỗi compile hoặc crash runtime.
</details>

**Q14 Integer overflow trong Swift xử lý thế nào?**
<details><summary>Đáp án gợi ý</summary>

Mặc định **crash runtime** (trap) để tránh lỗi âm thầm. Muốn wrap-around dùng toán tử `&+`, `&-`, `&*`. Có thể kiểm tra bằng `addingReportingOverflow(_:)`.
</details>

**Q15 `==` và `===` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`==` so sánh giá trị, yêu cầu `Equatable`. `===` so sánh định danh — hai biến có trỏ tới **cùng một instance** class hay không; không dùng được với struct/enum.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–8 không xem lời giải trước
- [ ] Làm xong Bài tập lớn `grades.swift`
- [ ] Trả lời được ≥ 12/15 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút giải thích: "Optional là gì và các cách unwrap"
- [ ] Ghi âm 2 phút giải thích: "Vì sao String không có `str[0]`"
- [ ] Làm xong 6 bài DSA tuần 1
- [ ] Side project Milestone 1 xong (tạo repo, project Xcode)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [The Swift Programming Language — The Basics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/)
- [The Swift Programming Language — Strings and Characters](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/)
- [The Swift Programming Language — Collection Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/collectiontypes/)
- [The Swift Programming Language — Control Flow](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/)
- [The Swift Programming Language — Functions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/functions/)
- Hacking with Swift — *100 Days of SwiftUI*, Day 1–9 (ôn nhanh cú pháp nếu thấy cần)
