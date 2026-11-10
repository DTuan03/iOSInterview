# Tuần 11 — SwiftUI nền tảng: View là gì · State management · Data flow (18/01 – 24/01/2027)

> **Giai đoạn 3 · UI** · Thời lượng: ~14 giờ
> Tuần 9–10 bạn đã ôn UIKit — thế giới **imperative**. Tuần này đổi tư duy sang **declarative**. Người phỏng vấn ở mức 2 năm hầu như **luôn** hỏi: *"`@StateObject` khác `@ObservedObject` thế nào?"*, *"View trong SwiftUI là gì, vì sao là struct?"*, *"`@Observable` khác `ObservableObject` ra sao?"*. Ai chỉ biết "gõ được màn hình" sẽ trượt ở các câu "vì sao".

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích khác biệt **imperative vs declarative** bằng một ví dụ cụ thể (UIKit counter vs SwiftUI counter)
- [ ] Giải thích **View là một value-type mô tả UI**, không phải "cái view trên màn hình"; `body`, `some View`, `@ViewBuilder` làm gì
- [ ] Giải thích vì sao **thứ tự modifier quan trọng** và vẽ được cây `ModifiedContent`
- [ ] Mô tả thuật toán layout 3 bước: **parent đề xuất → child tự chọn → parent đặt vị trí**
- [ ] Phân biệt `@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`, `@Observable`, `@Bindable`, `@Environment` — và **chọn đúng wrapper** cho từng tình huống
- [ ] Giải thích **single source of truth** và vẽ được sơ đồ data flow của một màn hình
- [ ] Dùng `List`/`ForEach` với `Identifiable` đúng cách, biết bẫy `id: \.self`
- [ ] DSA: 4–5 bài Heap / Priority Queue (xem [dsa-plan](../tracks/dsa-plan.md))
- [ ] Side project: màn hình detail của RepoScope bằng SwiftUI với view model `@Observable`

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 18/01** | 2h | Tạo `practice/week11/` (một project iOS App, interface SwiftUI, target iOS 17). Đọc §1 Declarative vs imperative, §2 View là gì. Bài tập 1 · DSA: Kth Largest Element in an Array | `Day1View.swift`, ghi chú "kiểu thật của body" |
| **T3 19/01** | 2h | §3 **Modifier & thứ tự**, §4 **Layout system**. Bài tập 2, 3 · DSA: Last Stone Weight | `MessageRow.swift` |
| **T4 20/01** | 2h | §5 **`@State` & `@Binding`**, source of truth. Bài tập 4 · DSA: K Closest Points to Origin | `OrderScreen.swift` |
| **T5 21/01** | 2h | §6 **Thời ObservableObject**: `@StateObject` vs `@ObservedObject` vs `@EnvironmentObject` (đọc kỹ — câu hỏi số 1 của tuần). Bài tập 5 · DSA: Top K Frequent Elements | Giải thích bug trong Bài 5 bằng lời |
| **T6 22/01** | 1,5h | §7 **Observation framework**: `@Observable`, `@Bindable`, `@Environment(Type.self)`. Bài tập 6 · DSA: Merge k Sorted Lists | `SearchScreen.swift` |
| **T7 23/01** | 4h | §8 **Data flow & bảng chọn wrapper**, §9 **List/ForEach**. Bài tập 7 + **Bài tập lớn** (Todo app). Side project: màn hình detail RepoScope bằng SwiftUI + `@Observable` view model | Todo app chạy trên simulator, commit side project |
| **CN 24/01** | 2h | Tự trả lời 18 câu phỏng vấn (viết ra trước). Ghi âm 2 câu. Vẽ tay sơ đồ data flow của Todo app. Tick checklist | Checklist ≥ 80% |

> **Mẹo tuần này:** mỗi khi viết xong một view, tự hỏi *"ai là chủ (owner) của dữ liệu này?"*. Trả lời được câu đó = hiểu 70% SwiftUI.

---

## 📚 Kiến thức cốt lõi

### §1. Imperative vs Declarative

**Imperative (UIKit):** bạn ra lệnh *từng bước* để **biến đổi** UI hiện tại thành UI mới. Bạn giữ tham chiếu tới view và tự nhớ cập nhật.

```swift
final class CounterViewController: UIViewController {
    private var count = 0
    private let label = UILabel()
    private let button = UIButton(type: .system)

    override func viewDidLoad() {
        super.viewDidLoad()
        label.text = "0"
        button.setTitle("Tăng", for: .normal)
        button.addTarget(self, action: #selector(didTap), for: .touchUpInside)
        // ... addSubview + Auto Layout
    }

    @objc private func didTap() {
        count += 1
        label.text = "\(count)"          // phải tự nhớ cập nhật UI
        label.textColor = count > 5 ? .red : .label   // quên dòng này = bug
    }
}
```

**Declarative (SwiftUI):** bạn mô tả *UI trông như thế nào ứng với một state*. Khi state đổi, framework gọi lại `body`, so sánh mô tả cũ và mới, rồi **tự** áp dụng phần khác biệt.

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("\(count)")
                .foregroundStyle(count > 5 ? .red : .primary)
            Button("Tăng") { count += 1 }
        }
    }
}
```

Công thức cần thuộc: **`UI = f(state)`**. `body` là hàm `f`. Bạn không bao giờ "set label.text"; bạn chỉ đổi state.

| | Imperative (UIKit) | Declarative (SwiftUI) |
|---|---|---|
| Bạn viết | Các bước chuyển UI từ A → B | UI ở trạng thái bất kỳ trông thế nào |
| Đồng bộ UI–state | Tự làm, dễ quên → UI lệch state | Framework làm (diffing) |
| Tham chiếu tới view | Giữ `UILabel`, `IBOutlet` | Không giữ, view là giá trị tạm |
| Số trạng thái phải nghĩ | Mọi cặp chuyển đổi (n²) | Mỗi trạng thái (n) |
| Kiểm soát chi tiết | Rất cao | Thấp hơn, đôi khi phải "lách" |

> **Câu trả lời ăn điểm:** "Với imperative, số đường chuyển giữa các trạng thái tăng theo n², bug thường là *quên cập nhật một chỗ*. Declarative chỉ cần mô tả n trạng thái, framework lo việc chuyển. Đổi lại ta mất bớt quyền kiểm soát và phải hiểu cách framework quyết định khi nào vẽ lại."

### §2. View là gì? `body`, `some View`, `@ViewBuilder`

#### 2.1. View là một *mô tả*, không phải một *đối tượng trên màn hình*

```swift
public protocol View {
    associatedtype Body: View
    @ViewBuilder @MainActor var body: Self.Body { get }
}
```
(Từ SDK iOS 18 / Xcode 16, toàn bộ protocol `View` được đánh dấu `@MainActor`.)

- Một `struct` conform `View` chỉ là **công thức (recipe)** nhẹ: "ở đây có một Text màu đỏ trong một VStack". Nó **không** phải `UIView`, không có frame, không có vòng đời riêng.
- SwiftUI tạo view struct **rất nhiều lần** (mỗi lần parent re-evaluate `body`), dùng xong vứt. Vì vậy:
  - View phải **rẻ để tạo** → **struct** (value type, nằm trên stack, không ARC, không cấp phát heap).
  - **Không** làm việc nặng trong `init` hay `body` (không gọi network, không parse JSON, không tạo `DateFormatter` mỗi lần).
  - State **không thể** lưu trong struct (vì struct bị tạo lại liên tục) → SwiftUI lưu state ở **bộ nhớ riêng do framework quản lý** (attribute graph), gắn với **identity** của view (học sâu tuần 12). `@State` chỉ là "tay cầm" trỏ tới vùng nhớ đó.
- Phía sau, SwiftUI giữ một cấu trúc dữ liệu bền (attribute graph / render tree) và tự tạo/cập nhật `UIView`/`CALayer` thật. Bạn chỉ thấy lớp mô tả.

Chứng minh view struct nhẹ:
```swift
struct V: View {
    let flag: Bool
    var body: some View { Text("A") }
}
print(MemoryLayout<V>.size)   // 1 — chỉ 1 byte cho Bool
```

#### 2.2. `some View` — opaque type (ôn tuần 4)

`body` trả `some View` nghĩa là: "tôi trả **một kiểu cụ thể duy nhất**, compiler biết chính xác kiểu đó, nhưng người gọi không cần biết". Kiểu thật thường rất dài:

```swift
struct V: View {
    let flag: Bool
    var body: some View {
        HStack { Text("A"); if flag { Image(systemName: "star") } }
    }
}
print(type(of: V(flag: true).body))
// HStack<TupleView<(Text, Optional<Image>)>>

print(type(of: Text("A").padding().background(Color.yellow)))
// ModifiedContent<ModifiedContent<Text, _PaddingLayout>, _BackgroundStyleModifier<Color>>
```
(Chạy được bằng `swift file.swift` trên macOS với `import SwiftUI`.)

Vì sao không dùng `any View` / `AnyView`? Vì kiểu tĩnh **mã hoá cấu trúc cây view**. SwiftUI đọc kiểu đó để biết "chỗ này luôn là một Text, chỗ kia là Optional<Image>" → diffing cực nhanh, không cần so sánh động. Đây là gốc của khái niệm **structural identity** (tuần 12).

```swift
// ❌ lỗi compile: function declares an opaque return type 'some View',
//    but the return statements in its body do not have matching underlying types
func icon(isOn: Bool) -> some View {
    if isOn { return Text("On") }
    else { return Image(systemName: "power") }
}
```

#### 2.3. `@ViewBuilder` — result builder

`body` được ngầm đánh dấu `@ViewBuilder`. Result builder biến **danh sách câu lệnh** thành **một giá trị**:

| Bạn viết trong builder | Kiểu được tạo ra |
|---|---|
| 2–10 view liên tiếp | `TupleView<(A, B, ...)>` |
| `if cond { A }` | `Optional<A>` |
| `if cond { A } else { B }` | `_ConditionalContent<A, B>` |
| `if #available(iOS 18, *) { A }` | `AnyView` (qua `buildLimitedAvailability`) |
| Không có gì | `EmptyView` |

```swift
struct W: View {
    let flag: Bool
    var body: some View {
        if flag { Text("A") } else { Image(systemName: "star") }
    }
}
print(type(of: W(flag: true).body))   // _ConditionalContent<Text, Image>
```

→ Lỗi compile ở 2.2 biến mất nếu thêm `@ViewBuilder` và **bỏ `return`**:
```swift
@ViewBuilder
func badge(for count: Int) -> some View {
    if count == 0 {
        EmptyView()
    } else if count < 100 {
        Text("\(count)")
    } else {
        Text("99+")
    }
}
```

**Tự viết container nhận `@ViewBuilder`** (pattern hay gặp trong design system):
```swift
struct Card<Content: View>: View {
    let title: String
    @ViewBuilder let content: Content

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(title).font(.headline)
            content
        }
        .padding()
        .background(.thinMaterial, in: RoundedRectangle(cornerRadius: 12))
    }
}

struct CardDemo: View {
    var body: some View {
        Card(title: "Thông tin") {
            Text("Dòng 1")
            Text("Dòng 2")
        }
    }
}
```

> **Lưu ý:** trong builder **không** viết được `for` thường, `let x = ...; x += 1`, hay `print(...)` như hàm bình thường. Muốn lặp → `ForEach`. Muốn debug → `let _ = print("...")` hoặc `let _ = Self._printChanges()`.

### §3. Modifier — vì sao thứ tự quan trọng

Mỗi modifier **không sửa** view cũ (view là struct bất biến) mà **bọc** nó vào một view mới: `ModifiedContent<Content, Modifier>`. Chuỗi modifier là **cây lồng nhau từ trong ra ngoài**, đọc từ trên xuống = từ trong ra ngoài.

```swift
struct ModifierOrder: View {
    var body: some View {
        VStack(spacing: 20) {
            Text("A").padding().background(.yellow)   // nền vàng phủ cả phần padding
            Text("B").background(.yellow).padding()   // nền vàng chỉ ôm chữ, padding trong suốt bên ngoài
            Text("C").frame(width: 200).background(.blue)   // nền xanh rộng 200
            Text("D").background(.blue).frame(width: 200)   // nền xanh chỉ ôm chữ, nằm giữa khung 200
        }
    }
}
```

Vẽ `Text("A").padding().background(.yellow)`:
```
background(.yellow)          ← lớp ngoài cùng: vẽ vàng bằng kích thước của "con" nó
 └─ padding()                ← con: Text + 16pt mỗi phía
     └─ Text("A")
```
Với B: `padding` nằm ngoài cùng, `background` chỉ biết kích thước Text → nền nhỏ.

**Quy tắc nhớ:**
- Modifier **layout** (`padding`, `frame`, `offset`...) thay đổi kích thước/vị trí mà các modifier *bên ngoài* nhìn thấy.
- `background`/`overlay` lấy kích thước của **view nó bọc**.
- `.offset` chỉ dời chỗ **vẽ**, không đổi vị trí layout (các view khác không "né"). `.padding` thì đổi layout.
- Một số modifier là **environment modifier** (`.font`, `.foregroundStyle`, `.tint`, `.disabled`) — truyền xuống **mọi view con** qua environment, và giá trị **gần view nhất thắng**:
  ```swift
  VStack {
      Text("To")            // .title
      Text("Nhỏ").font(.caption)   // .caption thắng vì gần hơn
  }
  .font(.title)
  ```

**Tự viết modifier tái sử dụng** (`ViewModifier`):
```swift
struct PrimaryButtonStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .font(.headline)
            .padding(.horizontal, 24)
            .padding(.vertical, 12)
            .background(.blue, in: Capsule())
            .foregroundStyle(.white)
    }
}

extension View {
    func primaryButton() -> some View { modifier(PrimaryButtonStyle()) }
}
// Text("Mua ngay").primaryButton()
```

### §4. Layout system — "parent đề xuất, child chọn, parent đặt"

Khác hẳn Auto Layout (giải hệ phương trình ràng buộc). SwiftUI layout là **một lượt đi xuống rồi đi lên**:

```
1. Parent ĐỀ XUẤT (propose) kích thước cho child:  "Mày có tối đa 300×100"
2. Child TỰ CHỌN kích thước của nó và báo lên:        "Tao chỉ cần 80×20"
3. Parent ĐẶT child vào vị trí (theo alignment):     "Vậy tao để mày ở giữa"
```
**Child có quyền quyết định cuối cùng về kích thước của chính nó.** Parent không ép được (đây là câu hay bị hỏi).

**Mỗi loại view "cư xử" khác nhau khi nhận đề xuất:**
| View | Hành vi |
|---|---|
| `Text` | Lấy vừa đủ chữ; nếu không đủ chỗ thì xuống dòng, rồi mới cắt (`...`) |
| `Image` (chưa `.resizable()`) | Luôn lấy đúng kích thước ảnh gốc, **phớt lờ** đề xuất |
| `Image().resizable()` | Lấy hết đề xuất (thường kèm `.scaledToFit()`) |
| `Shape` (`Rectangle`, `Circle`), `Color` | Lấy **hết** không gian được đề xuất |
| `Spacer` | Co giãn, lấp chỗ trống dọc theo trục của stack |
| `HStack`/`VStack` | Chia không gian cho con: con **kém linh hoạt nhất** được hỏi trước, phần còn lại chia cho con linh hoạt hơn |

**`frame` không phải "set kích thước cho view"**. `.frame(width: 200)` tạo một **view bọc** có width 200, đề xuất 200 cho con, con vẫn tự chọn (Text "Hi" chỉ rộng ~20), rồi frame đặt con vào giữa (mặc định `alignment: .center`). Vì vậy D ở §3 có nền xanh nhỏ.

**Công cụ layout hay dùng:**
```swift
struct LayoutDemo: View {
    var body: some View {
        VStack {
            HStack {
                Text("Tiêu đề rất rất dài có thể bị cắt bớt khi không đủ chỗ")
                    .lineLimit(1)
                Spacer()
                Text("12:30").layoutPriority(1)     // được chia chỗ trước → không bị cắt
            }
            Text("Ô vuông")
                .frame(maxWidth: .infinity, alignment: .leading)  // "giãn hết chiều ngang, căn trái"
                .frame(height: 44)
            Text("Không bao giờ bị cắt").fixedSize()   // bỏ qua đề xuất, lấy kích thước lý tưởng
        }
        .padding(16)
    }
}
```
- `.frame(maxWidth: .infinity)` — thay cho "leading/trailing = superview" của Auto Layout.
- `.layoutPriority(_:)` — mặc định 0; stack cho con có priority cao hơn lấy chỗ trước.
- `.fixedSize()` — "đừng đề xuất gì cho tao, tao lấy kích thước lý tưởng" (có thể tràn).
- `Spacer(minLength:)` — mặc định có khoảng tối thiểu nhỏ theo hệ thống.
- Stack có `spacing` mặc định (khác 0, tuỳ nền tảng) — muốn sát nhau phải ghi `spacing: 0`.

**GeometryReader — dùng thận trọng:**
```swift
struct ProgressBar: View {
    let progress: Double          // 0...1
    var body: some View {
        Capsule()
            .fill(.gray.opacity(0.2))
            .overlay(alignment: .leading) {
                GeometryReader { proxy in
                    Capsule().fill(.green)
                        .frame(width: proxy.size.width * progress)
                }
            }
            .frame(height: 8)
    }
}
```
Vì sao "thận trọng"?
1. `GeometryReader` **lấy toàn bộ** không gian được đề xuất (như `Color`) → bọc ngoài cùng là layout bị phình.
2. Nó đặt con ở **góc trên-trái**, không căn giữa như stack.
3. Đọc size rồi ghi vào `@State` trong `onAppear`/`onChange` dễ gây **vòng lặp layout** (size đổi → state đổi → layout lại).
4. Ưu tiên thứ khác trước: `frame(maxWidth:)`, `containerRelativeFrame` (iOS 17), `ViewThatFits`, `Layout` protocol. Mẹo phổ biến là đặt `GeometryReader` trong `.background`/`.overlay` (như trên) để nó lấy đúng kích thước view chính, không phá layout.

### §5. `@State` và `@Binding` — source of truth

**Source of truth** = nơi **sở hữu** và **lưu** một mẩu dữ liệu. Mỗi mẩu dữ liệu chỉ nên có **một** source of truth; mọi nơi khác chỉ **đọc** hoặc **tham chiếu** tới nó. Có hai bản copy của cùng dữ liệu = sớm muộn sẽ lệch nhau.

**`@State`** — source of truth **cục bộ, do view sở hữu**, cho value type đơn giản (Bool, String, struct nhỏ...) hoặc (từ iOS 17) object `@Observable`.

```swift
struct SettingsScreen: View {
    @State private var notificationsOn = false     // view này là chủ
    @State private var username = ""

    var body: some View {
        Form {
            NotificationRow(isOn: $notificationsOn) // $ → Binding<Bool>
            TextField("Tên", text: $username)
            Text(notificationsOn ? "Đang bật" : "Đang tắt")
        }
    }
}

struct NotificationRow: View {
    @Binding var isOn: Bool                        // KHÔNG sở hữu, chỉ đọc-ghi vào chỗ của cha
    var body: some View {
        Toggle("Thông báo", isOn: $isOn)
    }
}

#Preview {
    NotificationRow(isOn: .constant(true))         // Binding hằng cho preview
}
```

**Cơ chế bên trong `@State`:**
- `@State` là property wrapper `State<Value>`. Giá trị thật nằm trong **storage của SwiftUI** gắn với identity của view, không nằm trong struct. Vì vậy dù struct bị tạo lại 100 lần, `count` vẫn giữ.
- `count += 1` → setter của `wrappedValue` báo cho SwiftUI "dữ liệu này đổi" → SwiftUI đánh dấu các view **đã đọc** nó là dirty → gọi lại `body` ở frame kế tiếp.
- `$count` là `projectedValue` → `Binding<Int>`: một cặp **get/set closure** trỏ về storage gốc.
- **Giá trị khởi tạo chỉ dùng lần đầu**. `@State private var name = initialName` — nếu cha truyền `initialName` khác ở lần sau, `name` **không** đổi theo (bẫy kinh điển). Muốn reset → đổi identity bằng `.id(...)` (tuần 12) hoặc dùng `onChange`.
- Luôn khai `@State` là **`private`**: nó là của riêng view. Nếu cha cần truyền vào → đó là dấu hiệu dữ liệu nên là `let` hoặc `@Binding`.
- Thay đổi `@State` ngoài main thread là sai (runtime warning). Từ Swift 6, View là `@MainActor` nên compiler bắt phần lớn.

**Vì sao không gán được property thường trong view?**
```swift
struct BadCounter: View {
    var count = 0
    var body: some View {
        // ❌ lỗi compile: left side of mutating operator isn't mutable: 'self' is immutable
        Button("Tăng") { count += 1 }
    }
}
```
`body` là getter **non-mutating** của struct → `self` bất biến. `@State` lách được vì setter của nó ghi vào storage bên ngoài (`nonmutating set`).

**`@Binding` không phải bản copy**: nó là tham chiếu hai chiều tới source of truth của người khác. Truyền nhầm giá trị thay vì binding:
```swift
struct BadChild: View {
    @Binding var isOn: Bool
    // ❌ lỗi compile: cannot convert value 'isOn' of type 'Bool' to expected type 'Binding<Bool>', use wrapper instead
    var body: some View { Toggle("x", isOn: isOn) }   // phải là $isOn
}
```

### §6. Thời `ObservableObject` (iOS 13–16, vẫn gặp rất nhiều trong codebase công ty)

Khi dữ liệu là **reference type** (view model, store, service có state) thì `@State` (thời iOS 13–16) không theo dõi được thay đổi bên trong object. Giải pháp cũ dùng **Combine**:

```swift
final class CartStore: ObservableObject {
    @Published var items: [String] = []      // mỗi lần SẮP đổi → objectWillChange.send()
    @Published var isLoading = false
    var debugCounter = 0                     // không @Published → đổi không ai biết
    func add(_ item: String) { items.append(item) }
}
```
- `ObservableObject` có một publisher `objectWillChange`. Mỗi `@Published` khi **sắp** đổi (`willSet`) sẽ gọi `objectWillChange.send()`.
- View đăng ký nghe `objectWillChange`. **Bất kỳ** `@Published` nào đổi → **mọi view** đang quan sát object đó đều re-evaluate `body`, kể cả view không đọc property đó. (Đây là điểm yếu về hiệu năng mà Observation sửa — tuần 12 đi sâu.)

**Ba wrapper — câu hỏi số 1 của tuần:**

| Wrapper | Sở hữu object? | Ai tạo object? | Sống bao lâu |
|---|---|---|---|
| `@StateObject` | **Có** | View tự tạo (`= CartStore()`) | Theo **identity** của view — tạo **1 lần**, giữ qua mọi lần re-render |
| `@ObservedObject` | **Không** | Được truyền từ bên ngoài | Do người tạo quyết định; view chỉ "nghe" |
| `@EnvironmentObject` | Không | Tổ tiên inject bằng `.environmentObject(_:)` | Do tổ tiên giữ; con cháu lấy theo **kiểu** |

```swift
struct CartScreen: View {
    @StateObject private var store = CartStore()   // CHỦ: tạo 1 lần duy nhất
    var body: some View {
        VStack {
            CartList(store: store)                  // truyền xuống trực tiếp
            CartBadge()                             // lấy qua environment
        }
        .environmentObject(store)
    }
}

struct CartList: View {
    @ObservedObject var store: CartStore            // KHÔNG sở hữu, chỉ quan sát
    var body: some View {
        List(store.items, id: \.self) { Text($0) }
        Button("Thêm") { store.add("Táo") }
    }
}

struct CartBadge: View {
    @EnvironmentObject var store: CartStore         // 💥 crash nếu tổ tiên quên .environmentObject
    var body: some View { Text("\(store.items.count)") }
}
```

**Bug kinh điển — `@ObservedObject` tự tạo object:**
```swift
struct BuggyTimerScreen: View {
    @ObservedObject var store = CartStore()   // ⚠️ compile được nhưng SAI
    var body: some View { Text("\(store.items.count)") }
}
```
Mỗi khi **cha** re-render, struct `BuggyTimerScreen` được tạo lại → `CartStore()` chạy lại → **object mới, state mất sạch** (timer reset, list trống, request gọi lại). `@StateObject` tránh được vì nó nhận `wrappedValue` dưới dạng `@autoclosure` và **chỉ gọi closure một lần** cho mỗi identity, các lần sau trả lại object đã lưu.

**Quy tắc 1 dòng:** *Ai tạo thì dùng `@StateObject`, ai được đưa cho thì dùng `@ObservedObject`.*

**Khởi tạo `@StateObject` với tham số:**
```swift
struct LegacyDetail: View {
    @StateObject private var store: CartStore
    init(seed: [String]) {
        let s = CartStore()
        s.items = seed
        _store = StateObject(wrappedValue: s)   // closure vẫn chỉ được dùng lần đầu!
    }
    var body: some View { Text("\(store.items.count)") }
}
```
Lưu ý: nếu `seed` đổi ở các lần sau, store **không** được tạo lại (giống `@State`). Apple khuyến nghị dùng cách này hạn chế.

### §7. Observation framework — `@Observable` (iOS 17+)

Swift 5.9 thêm macro `@Observable` (module `Observation`). Đây là cách **mặc định** cho code mới target iOS 17+.

```swift
import Observation

@Observable
final class ProfileModel {
    var name = "An"                              // tự động được theo dõi, không cần @Published
    var age = 25
    var isLoading = false
    @ObservationIgnored var cache: [String: String] = [:]   // loại khỏi theo dõi

    func load() async {
        isLoading = true
        defer { isLoading = false }
        try? await Task.sleep(for: .seconds(1))
        name = "An Nguyễn"
    }
}
```

**Macro sinh ra gì?** (mở bằng chuột phải → *Expand Macro* trong Xcode)
- Một `ObservationRegistrar` ẩn trong class.
- Mỗi stored property được viết lại thành computed property: **getter** gọi `registrar.access(self, keyPath:)` ("có ai đó vừa đọc `name`"), **setter** bọc trong `registrar.withMutation(of:keyPath:)` ("`name` đang đổi").
- Class conform `Observable` (protocol đánh dấu).

**SwiftUI dùng nó thế nào?** Khi chạy `body`, SwiftUI bọc trong `withObservationTracking { body } onChange: { ... }`. Mọi property **thực sự được đọc** trong lúc chạy `body` được ghi lại. Chỉ khi **một trong các property đó** đổi, view mới bị đánh dấu cần vẽ lại.
→ **Theo dõi ở mức property**, không phải mức object như `ObservableObject`. View chỉ đọc `name` sẽ **không** re-render khi `age` đổi.

**Các wrapper trong thế giới Observation:**

| Cần | Thời ObservableObject | Thời Observation |
|---|---|---|
| View **tạo và sở hữu** model | `@StateObject var m = M()` | `@State private var m = M()` |
| View **được truyền** model, chỉ đọc | `@ObservedObject var m: M` | `let m: M` (hoặc `var`) — **không cần wrapper** |
| View được truyền model, cần **`$` binding** | `@ObservedObject var m: M` | `@Bindable var m: M` |
| Inject qua cây view | `.environmentObject(m)` + `@EnvironmentObject var m: M` | `.environment(m)` + `@Environment(M.self) var m` |
| Lấy binding từ model trong environment | (có sẵn `$m.x`) | `@Bindable var m = m` trong `body` |

```swift
struct ProfileScreen: View {
    @State private var model = ProfileModel()      // CHỦ
    var body: some View {
        VStack {
            NameLabel(model: model)
            EditNameForm(model: model)
        }
        .task { await model.load() }
        .environment(model)                         // inject cho con cháu
    }
}

struct NameLabel: View {
    let model: ProfileModel                         // chỉ đọc → không wrapper
    var body: some View { Text(model.name) }        // chỉ phụ thuộc name
}

struct EditNameForm: View {
    @Bindable var model: ProfileModel               // cần $model.name
    var body: some View {
        TextField("Tên", text: $model.name)
        Stepper("Tuổi: \(model.age)", value: $model.age)
    }
}

struct AgeEditor: View {
    @Environment(ProfileModel.self) private var model   // 💥 crash nếu không ai inject
    var body: some View {
        @Bindable var model = model                 // tạo binding cục bộ trong body
        Stepper("Tuổi", value: $model.age)
    }
}
```

- Muốn environment **không bắt buộc**: `@Environment(ProfileModel.self) private var model: ProfileModel?` → trả `nil` thay vì crash.
- `@Environment(\.keyPath)` vẫn dùng cho giá trị hệ thống: `\.dismiss`, `\.colorScheme`, `\.dynamicTypeSize`, `\.openURL`...

**Vì sao `@State` giữ được class?** `@State` chỉ đảm bảo **giữ tham chiếu ổn định** qua các lần re-render. Việc *theo dõi thay đổi* do Observation lo, không phải `@State`.

**⚠️ Khác biệt tinh tế `@State` vs `@StateObject`:** `@StateObject` nhận `@autoclosure` → `M()` chỉ chạy 1 lần. `@State` thì **không** → biểu thức `ProfileModel()` **chạy lại mỗi lần struct view được init**, SwiftUI chỉ giữ instance đầu tiên và vứt các instance sau. Hệ quả:
- `init` của model phải **rẻ**, không side effect (không gọi network, không start timer trong `init`).
- Load dữ liệu trong `.task { await model.load() }` thay vì trong `init` của model.

**Khởi tạo `@State` từ tham số:**
```swift
struct DetailScreen: View {
    @State private var model: ProfileModel
    init(name: String) {
        let m = ProfileModel()
        m.name = name
        _model = State(initialValue: m)
    }
    var body: some View { Text(model.name) }
}
```

**So sánh tổng quát:**
| | `ObservableObject` | `@Observable` |
|---|---|---|
| Yêu cầu | iOS 13+, Combine | iOS 17+, Swift 5.9 |
| Đánh dấu property | `@Published` từng cái | Tự động mọi stored property |
| Độ chi tiết theo dõi | Cả object (`objectWillChange`) | Từng property **được đọc trong body** |
| Wrapper trong view | `@StateObject`, `@ObservedObject`, `@EnvironmentObject` | `@State`, (không gì), `@Bindable`, `@Environment` |
| Theo dõi trong mảng/optional/computed | Không (chỉ `@Published` trực tiếp) | Có — đọc `model.items[0].name` qua chuỗi object `@Observable` vẫn được track |
| Struct dùng được? | Không (chỉ class) | Không (chỉ class) |

> **Thực tế công ty:** app hỗ trợ iOS 15–16 vẫn phải dùng `ObservableObject`. Phỏng vấn sẽ hỏi **cả hai**, và hay hỏi "migrate từ cái cũ sang cái mới thế nào".

### §8. Data flow — sơ đồ và bảng chọn wrapper

```
                     ┌──────────────────────────┐
                     │  @State / @StateObject    │  ← SOURCE OF TRUTH (sở hữu)
                     │  (ProfileScreen)          │
                     └─────┬─────────────┬───────┘
           truyền giá trị  │             │ .environment(model)
           hoặc $binding   ▼             ▼
                  ┌──────────────┐   ┌──────────────────────────┐
                  │ let / @Binding│   │ @Environment(Model.self)  │
                  │ / @Bindable   │   │ ở bất kỳ con cháu nào     │
                  └──────┬───────┘   └────────────┬─────────────┘
                         │ user action (tap, gõ)  │
                         ▼                        ▼
                  ghi vào binding / gọi method của model
                         │
                         ▼
           source of truth đổi → SwiftUI vẽ lại view đã ĐỌC dữ liệu đó
```
Dữ liệu chảy **xuống** (data down), sự kiện chảy **lên** (actions up). Đây là **unidirectional data flow**.

**Bảng quyết định — thuộc lòng:**

| Tình huống | Dùng |
|---|---|
| Value type đơn giản, chỉ view này dùng (toggle, text đang gõ, tab đang chọn) | `@State private var` |
| Con cần **đọc** giá trị của cha | `let value: T` (tham số thường) |
| Con cần **đọc + ghi** giá trị value type của cha | `@Binding var` |
| Class model, **view này tạo**, iOS 17+ | `@State private var model = Model()` (`@Observable`) |
| Class model, **view này tạo**, iOS 13–16 | `@StateObject private var model = Model()` (`ObservableObject`) |
| Class model **được truyền vào**, iOS 17+, chỉ đọc / gọi method | `let model: Model` |
| Class model được truyền vào, iOS 17+, cần `$model.prop` | `@Bindable var model: Model` |
| Class model được truyền vào, iOS 13–16 | `@ObservedObject var model: Model` |
| Dữ liệu dùng chung cho **nhiều tầng** (user session, theme, router) | `.environment(obj)` + `@Environment(T.self)` (hoặc `environmentObject` ở iOS cũ) |
| Giá trị hệ thống | `@Environment(\.dismiss)`, `\.colorScheme`... |
| Lưu vào `UserDefaults` tự động | `@AppStorage("key")` |
| Lưu state UI khi app bị kill & khôi phục (state restoration) | `@SceneStorage("key")` |

> **Không lạm dụng environment.** Nó tiện nhưng là *dependency ẩn*: đọc chữ ký `init` không biết view cần gì, và quên inject = crash lúc runtime. Chỉ dùng cho thứ thật sự "toàn cục" trong một nhánh cây.

### §9. List, ForEach và `Identifiable`

`ForEach` cần biết **danh tính ổn định** của từng phần tử để biết phần tử nào được thêm/xoá/di chuyển (phục vụ animation và giữ state của từng hàng).

```swift
struct Repo: Identifiable, Hashable {
    let id: Int            // id ổn định, đến từ server
    var name: String
    var stars: Int
}

struct RepoListView: View {
    @State private var repos = [
        Repo(id: 1, name: "swift", stars: 66000),
        Repo(id: 2, name: "alamofire", stars: 41000),
    ]
    var body: some View {
        List {
            ForEach(repos) { repo in             // dùng repo.id nhờ Identifiable
                HStack {
                    Text(repo.name)
                    Spacer()
                    Text("★ \(repo.stars)")
                }
            }
            .onDelete { repos.remove(atOffsets: $0) }
            .onMove { repos.move(fromOffsets: $0, toOffset: $1) }
        }
    }
}
```

**Binding tới từng phần tử** (iOS 15+):
```swift
struct EditableRepoList: View {
    @State private var repos: [Repo] = []
    var body: some View {
        List($repos) { $repo in                  // $repo: Binding<Repo>
            TextField("Tên", text: $repo.name)
        }
    }
}
```

**Ba cách cung cấp id và rủi ro:**
| Cách | Ổn? | Ghi chú |
|---|---|---|
| `Identifiable` với `id` từ server/DB | ✅ Tốt nhất | Ổn định qua reload |
| `id: \.self` | ⚠️ | Chỉ khi phần tử **duy nhất và bất biến** (vd. `[String]` tag không trùng). Trùng giá trị → SwiftUI log "ID occurs multiple times", hành vi không xác định |
| `ForEach(arr.indices, id: \.self)` | ❌ trên dữ liệu động | Xoá phần tử 0 thì phần tử cũ ở index 1 "mượn" identity của index 0 → state của hàng (toggle, text đang gõ) nhảy sang hàng khác, animation sai, đôi khi crash index out of range |
| `let id = UUID()` sinh trong `init` của model | ⚠️ | Ổn nếu object được tạo **một lần**. Nếu mỗi lần decode/map lại tạo UUID mới → toàn bộ list bị coi là "mới", mất animation và state |

`List` vs `ScrollView + LazyVStack`: `List` dựa trên `UICollectionView` (iOS 16+) — có tái sử dụng cell, swipe actions, separator, edit mode. `LazyVStack` linh hoạt về giao diện nhưng **không reuse** view như cell (chỉ tạo lười khi cần). Tuần 12 bàn hiệu năng.

---

## ⚠️ Bẫy hay gặp

1. **`@ObservedObject var vm = ViewModel()`** — compile được, nhưng mỗi lần cha re-render là mất state. Câu hỏi bẫy số 1.
2. **`@State` khởi tạo từ tham số chỉ ăn lần đầu** — cha truyền giá trị mới, view con vẫn hiển thị giá trị cũ.
3. **`@State` không `private`** — dấu hiệu thiết kế sai; người khác có thể truyền vào qua memberwise init và tưởng nó sẽ cập nhật.
4. **Gọi network trong `init` của view hoặc của model gán vào `@State`** — `init` view chạy rất nhiều lần; model trong `@State` bị tạo thừa nhiều lần.
5. **Nghĩ `.frame(width:)` ép kích thước con** — nó chỉ tạo khung bọc; con vẫn tự chọn size.
6. **Thứ tự modifier**: `.padding().background()` ≠ `.background().padding()`; `.onTapGesture` đặt trước `.padding` thì vùng padding không nhận tap.
7. **`ForEach(array.indices, id: \.self)`** với dữ liệu thay đổi — state nhảy hàng, crash.
8. **`@Environment(Model.self)` / `@EnvironmentObject` mà quên inject** → crash runtime, đặc biệt trong `#Preview` và trong `sheet` (ở iOS 13–14, `sheet` không kế thừa environment object — code cũ hay inject lại cho chắc).
9. **Dùng `@Published`/`@StateObject` lẫn với `@Observable`** — `@Observable` class không cần `@Published`, và **không** dùng `@StateObject`/`@ObservedObject` cho nó (`@StateObject` yêu cầu `ObservableObject` → lỗi compile *generic struct 'StateObject' requires that 'M' conform to 'ObservableObject'*).
10. **`ObservableObject` có `@Published` là struct lớn** — sửa 1 field của struct cũng bắn `objectWillChange`, mọi view quan sát đều render lại.

---

## 🛠 Bài tập

> Tạo project `practice/week11/` (iOS App, SwiftUI, iOS 17). Mỗi bài một file. Xem kết quả trên `#Preview` hoặc simulator. **Không xem lời giải trước khi tự làm ít nhất 20 phút.**

**Bài 1 — Đọc kiểu thật của `body` (T2)**
a) Viết lại `CounterViewController` (UIKit) ở §1 thành SwiftUI, thêm: nút "Reset" chỉ hiện khi `count > 0`.
b) Đoán kiểu của `body` trước khi chạy. Rồi kiểm chứng bằng file `types.swift` chạy trên macOS:
```swift
import SwiftUI
struct V: View {
    let flag: Bool
    var body: some View {
        VStack { Text("A"); if flag { Text("B") } else { Image(systemName: "x") } }
    }
}
print(type(of: V(flag: true).body))
```
<details><summary>Lời giải</summary>

`VStack<TupleView<(Text, _ConditionalContent<Text, Image>)>>`. `if/else` trong builder → `_ConditionalContent`. Nếu chỉ có `if` không `else` → `Optional<...>`. Phần a: thêm `if count > 0 { Button("Reset") { count = 0 } }` trong `VStack` — kiểu sẽ có thêm `Optional<Button<Text>>`.
</details>

**Bài 2 — Thứ tự modifier (T3)**
a) Không chạy code, vẽ ra giấy 4 view A, B, C, D ở §3. Sau đó chạy để kiểm tra.
b) Làm một "tag chip": chữ trắng, nền xanh bo tròn, có viền 1pt màu xanh đậm **bên ngoài** nền 4pt. Gợi ý: cần `padding` 2 lần.
c) Tại sao `Text("Tap").onTapGesture { }.padding(20)` thì bấm vào vùng padding không ăn, còn `.padding(20).contentShape(Rectangle()).onTapGesture { }` thì ăn?
<details><summary>Lời giải</summary>

b)
```swift
Text("swift")
    .foregroundStyle(.white)
    .padding(.horizontal, 10).padding(.vertical, 4)
    .background(.blue, in: Capsule())
    .padding(4)
    .overlay(Capsule().stroke(.blue.opacity(0.8), lineWidth: 1))
```
c) Gesture gắn vào view nó bọc; trường hợp 1 nó chỉ bọc `Text`, padding nằm ngoài. Trường hợp 2 gesture bọc cả padding, và `contentShape` khai vùng hit-test là cả hình chữ nhật (mặc định vùng trong suốt không nhận tap).
</details>

**Bài 3 — Layout một hàng tin nhắn (T3)**
Dựng `MessageRow(name:preview:time:)`: avatar tròn 44×44 có chữ cái đầu, tên (1 dòng) + preview (tối đa 2 dòng, màu phụ) xếp dọc, thời gian ở mép phải **không bao giờ bị cắt**. Thêm `TwoButtons`: 2 nút "Huỷ" / "Đồng ý" **rộng bằng nhau** chiếm hết chiều ngang.
<details><summary>Lời giải</summary>

```swift
struct MessageRow: View {
    let name: String
    let preview: String
    let time: String

    var body: some View {
        HStack(alignment: .top, spacing: 12) {
            Circle()
                .fill(.blue.gradient)
                .frame(width: 44, height: 44)
                .overlay(Text(String(name.prefix(1))).foregroundStyle(.white))
            VStack(alignment: .leading, spacing: 4) {
                Text(name).font(.headline).lineLimit(1)
                Text(preview).font(.subheadline).foregroundStyle(.secondary).lineLimit(2)
            }
            Spacer(minLength: 8)
            Text(time)
                .font(.caption)
                .foregroundStyle(.secondary)
                .fixedSize()                 // hoặc .layoutPriority(1)
        }
        .padding(.vertical, 8)
    }
}

struct TwoButtons: View {
    var body: some View {
        HStack(spacing: 12) {
            Button("Huỷ") {}
                .frame(maxWidth: .infinity, minHeight: 44)
                .background(.gray.opacity(0.2), in: RoundedRectangle(cornerRadius: 10))
            Button("Đồng ý") {}
                .frame(maxWidth: .infinity, minHeight: 44)
                .background(.blue, in: RoundedRectangle(cornerRadius: 10))
                .foregroundStyle(.white)
        }
        .padding()
    }
}
```
Hai nút cùng `maxWidth: .infinity` → HStack chia đều phần còn lại. Lưu ý: vùng bấm của `Button` chỉ là label; muốn cả khung bấm được, đặt `.frame` **bên trong** label: `Button { } label: { Text("Huỷ").frame(maxWidth: .infinity) }`.
</details>

**Bài 4 — `@State` & `@Binding` (T4)**
Viết component `QuantityStepper(title:value:range:)` gồm nút −, số, nút +; nút bị disable khi chạm biên. Màn `OrderScreen` có 2 stepper (Cà phê 35k, Bánh 25k) và dòng tổng tiền cập nhật tức thì. Trả lời: **source of truth** của số lượng nằm ở đâu?
<details><summary>Lời giải</summary>

```swift
struct QuantityStepper: View {
    let title: String
    @Binding var value: Int
    var range: ClosedRange<Int> = 0...10

    var body: some View {
        HStack {
            Text(title)
            Spacer()
            Button { value -= 1 } label: { Image(systemName: "minus.circle") }
                .disabled(value <= range.lowerBound)
            Text("\(value)").monospacedDigit().frame(minWidth: 28)
            Button { value += 1 } label: { Image(systemName: "plus.circle") }
                .disabled(value >= range.upperBound)
        }
        .buttonStyle(.borderless)   // trong Form/List, thiếu dòng này thì tap 1 nút = tap cả hàng
    }
}

struct OrderScreen: View {
    @State private var coffee = 1
    @State private var cake = 0

    var body: some View {
        Form {
            QuantityStepper(title: "Cà phê", value: $coffee)
            QuantityStepper(title: "Bánh", value: $cake, range: 0...5)
            Text("Tổng: \(coffee * 35 + cake * 25)k")
        }
    }
}
```
Source of truth là 2 `@State` trong `OrderScreen`. Stepper không sở hữu gì — nó là component "dumb", tái sử dụng được. Tổng tiền là **dữ liệu dẫn xuất** (derived) → tính trong `body`, **không** lưu thành `@State` thứ ba (sẽ lệch).
</details>

**Bài 5 — Bug hunt: state bị reset (T5)**
```swift
final class TimerStore: ObservableObject {
    @Published var seconds = 0
    private var timer: AnyCancellable?
    init() {
        timer = Timer.publish(every: 1, on: .main, in: .common)
            .autoconnect()
            .sink { [weak self] _ in self?.seconds += 1 }
    }
}

struct TimerScreen: View {
    @ObservedObject var store = TimerStore()
    var body: some View { Text("\(store.seconds)s") }
}

struct ParentScreen: View {
    @State private var tapCount = 0
    var body: some View {
        VStack {
            Button("Tap \(tapCount)") { tapCount += 1 }
            TimerScreen()
        }
    }
}
```
Chạy, để timer đếm tới 10 rồi bấm nút "Tap". Chuyện gì xảy ra? Vì sao? Sửa 1 từ. Sau đó viết lại bằng `@Observable`.
<details><summary>Lời giải</summary>

Mỗi lần bấm Tap, `ParentScreen.body` chạy lại → tạo struct `TimerScreen()` mới → `TimerStore()` mới → đồng hồ về 0 (và object cũ bị huỷ, timer cũ dừng nhờ `AnyCancellable` deinit). Sửa: `@StateObject private var store = TimerStore()` — autoclosure chỉ chạy 1 lần cho identity của `TimerScreen`.
Bản `@Observable`: class `@Observable final class TimerStore` (bỏ `@Published`, thêm `@ObservationIgnored` cho `timer`), view dùng `@State private var store = TimerStore()`. **Nhưng** chú ý: với `@State`, `TimerStore()` vẫn bị *gọi* mỗi lần init view (rồi bị vứt) → mỗi lần bấm Tap sẽ tạo một timer tạm rồi huỷ ngay. Cách sạch hơn: không start timer trong `init`, mà start trong `.task { await store.run() }` dùng `for await` / `Task.sleep` loop — `.task` tự huỷ khi view biến mất.
</details>

**Bài 6 — Migrate sang `@Observable` (T6)**
Viết `SearchModel` (`query`, `results: [String]`, `isLoading`, `func search() async` giả lập trễ 300ms) bằng `@Observable`. `SearchScreen` sở hữu model, `SearchBar` (view con) cần `TextField` bind vào `query` và gọi search khi submit. Sau đó viết lại **cùng chức năng** bằng `ObservableObject` và so sánh từng dòng wrapper.
<details><summary>Lời giải</summary>

```swift
@Observable
final class SearchModel {
    var query = ""
    var results: [String] = []
    var isLoading = false

    func search() async {
        isLoading = true
        defer { isLoading = false }
        try? await Task.sleep(for: .milliseconds(300))
        results = ["\(query) 1", "\(query) 2"]
    }
}

struct SearchScreen: View {
    @State private var model = SearchModel()            // ⇔ @StateObject
    var body: some View {
        VStack {
            SearchBar(model: model)
            if model.isLoading { ProgressView() }
            List(model.results, id: \.self) { Text($0) }
        }
    }
}

struct SearchBar: View {
    @Bindable var model: SearchModel                    // ⇔ @ObservedObject (cần $)
    var body: some View {
        TextField("Tìm...", text: $model.query)
            .onSubmit { Task { await model.search() } }
    }
}
```
Bản cũ: `final class SearchModel: ObservableObject` + `@Published` cho 3 property; `@StateObject private var model = SearchModel()`; `@ObservedObject var model: SearchModel`. Khác biệt hành vi: với bản cũ, gõ mỗi ký tự (`query` đổi) làm **cả** `SearchScreen` re-render vì nó quan sát cả object; với `@Observable`, `SearchScreen` không đọc `query` trong body nên không bị vẽ lại khi gõ.
</details>

**Bài 7 — List & identity (T7)**
a) Dựng danh bạ `[Contact]` (`name`, `isFavorite`) với `Toggle` cho từng người và vuốt để xoá. Có **2 người cùng tên "An"**.
b) Thử dùng `ForEach(contacts.map(\.name), id: \.self)` — quan sát console và hành vi toggle. Rồi thử `ForEach(contacts.indices, id: \.self)`, bật toggle hàng 2 rồi xoá hàng 1. Ghi lại điều xảy ra.
c) Sửa đúng cách.
<details><summary>Lời giải</summary>

```swift
struct Contact: Identifiable {
    let id = UUID()            // ổn vì mỗi Contact được tạo 1 lần và lưu trong @State
    var name: String
    var isFavorite = false
}

struct ContactsView: View {
    @State private var contacts = [Contact(name: "An"), Contact(name: "An"), Contact(name: "Bình")]
    var body: some View {
        List {
            ForEach($contacts) { $contact in
                Toggle(contact.name, isOn: $contact.isFavorite)
            }
            .onDelete { contacts.remove(atOffsets: $0) }
        }
    }
}
```
b) `id: \.self` trên tên trùng → SwiftUI cảnh báo ID trùng, 2 hàng "An" bị coi là một, toggle/animation sai. Dùng `indices`: sau khi xoá hàng 1, hàng cũ số 2 giờ có index 0 → identity đổi → animation xoá sai hàng, nếu dùng `$contacts[i]` có thể crash index out of range trong lúc animate.
</details>

**🏋️ Bài tập lớn (T7) — Todo app với `@Observable` + Environment**
Yêu cầu:
- `TodoItem: Identifiable` (`title`, `isDone`), `TodoStore` là `@Observable` với `items`, `filter` (Tất cả / Chưa xong / Đã xong), computed `visibleItems`, `remainingCount`, các hàm `add`, `toggle`, `delete`.
- Store được tạo **một lần** ở `App` và inject bằng `.environment(store)`.
- Màn hình list: ô nhập + nút Thêm (disable khi rỗng), `Picker` segmented để lọc (cần binding vào `store.filter`), header section "Còn N việc", vuốt để xoá, tap để đánh dấu xong (gạch ngang).
- `TodoRow` là view riêng, tự lấy store từ environment.
- Vẽ sơ đồ data flow (giống §8) cho app này vào notes.

<details><summary>Lời giải tham khảo</summary>

```swift
struct TodoItem: Identifiable, Equatable {
    let id = UUID()
    var title: String
    var isDone = false
}

enum TodoFilter: String, CaseIterable, Identifiable {
    case all = "Tất cả", active = "Chưa xong", done = "Đã xong"
    var id: Self { self }
}

@Observable
final class TodoStore {
    var items: [TodoItem] = []
    var filter: TodoFilter = .all

    var visibleItems: [TodoItem] {
        switch filter {
        case .all: items
        case .active: items.filter { !$0.isDone }
        case .done: items.filter(\.isDone)
        }
    }
    var remainingCount: Int { items.filter { !$0.isDone }.count }

    func add(_ title: String) {
        let trimmed = title.trimmingCharacters(in: .whitespaces)
        guard !trimmed.isEmpty else { return }
        items.append(TodoItem(title: trimmed))
    }
    func toggle(_ item: TodoItem) {
        guard let index = items.firstIndex(where: { $0.id == item.id }) else { return }
        items[index].isDone.toggle()
    }
    func delete(_ item: TodoItem) {
        items.removeAll { $0.id == item.id }
    }
}

@main
struct TodoApp: App {
    @State private var store = TodoStore()
    var body: some Scene {
        WindowGroup {
            NavigationStack { TodoListView() }
                .environment(store)
        }
    }
}

struct TodoListView: View {
    @Environment(TodoStore.self) private var store
    @State private var newTitle = ""          // state UI cục bộ: không đưa vào store

    var body: some View {
        @Bindable var store = store           // để có $store.filter
        List {
            Section {
                HStack {
                    TextField("Việc mới", text: $newTitle)
                        .onSubmit(addItem)
                    Button("Thêm", action: addItem)
                        .disabled(newTitle.trimmingCharacters(in: .whitespaces).isEmpty)
                }
            }
            Section {
                Picker("Lọc", selection: $store.filter) {
                    ForEach(TodoFilter.allCases) { Text($0.rawValue).tag($0) }
                }
                .pickerStyle(.segmented)
            }
            Section("Còn \(store.remainingCount) việc") {
                ForEach(store.visibleItems) { item in
                    TodoRow(item: item)
                        .swipeActions {
                            Button("Xoá", role: .destructive) { store.delete(item) }
                        }
                }
            }
        }
        .navigationTitle("Todo")
    }

    private func addItem() {
        store.add(newTitle)
        newTitle = ""
    }
}

struct TodoRow: View {
    @Environment(TodoStore.self) private var store
    let item: TodoItem

    var body: some View {
        Button {
            store.toggle(item)
        } label: {
            HStack {
                Image(systemName: item.isDone ? "checkmark.circle.fill" : "circle")
                Text(item.title).strikethrough(item.isDone)
            }
        }
        .foregroundStyle(item.isDone ? .secondary : .primary)
    }
}
```
Điểm cần tự kiểm: `newTitle` là `@State` cục bộ (không ai khác cần) — không nhét vào store; `visibleItems` là computed (derived), không lưu riêng; `TodoRow` nhận `item` là `let` (giá trị), hành động đi ngược lên store.
</details>

**📱 Side project (T7) — RepoScope**: làm màn hình **Repo Detail** bằng SwiftUI (tên, mô tả, số sao/fork, ngôn ngữ, nút mở trên GitHub bằng `@Environment(\.openURL)`). View model `@Observable final class RepoDetailViewModel` với state `idle / loading / loaded(RepoDetail) / failed(String)` (một `enum`), load trong `.task`. Xem [side-project](../tracks/side-project.md).

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Declarative UI khác imperative UI thế nào? Ưu nhược điểm của SwiftUI so với UIKit?**
<details><summary>Đáp án gợi ý</summary>

Imperative: ta viết các bước biến đổi UI (set text, ẩn/hiện view), tự giữ UI đồng bộ với state. Declarative: ta mô tả UI là hàm của state (`UI = f(state)`), framework tự diff và cập nhật. Ưu: ít code, ít bug "quên cập nhật", preview nhanh, state rõ ràng, animation dễ, đa nền tảng. Nhược: ít kiểm soát chi tiết, một số API còn thiếu/khác nhau theo phiên bản iOS (phải hỗ trợ iOS cũ), debug hiệu năng re-render khó hơn, đôi khi phải bọc UIKit qua `UIViewRepresentable`.
</details>

**Q2 ⭐ View trong SwiftUI là gì? Vì sao là struct chứ không phải class?**
<details><summary>Đáp án gợi ý</summary>

View là một **mô tả** nhẹ, bất biến về một phần UI, không phải đối tượng hiển thị. SwiftUI tạo và huỷ chúng liên tục mỗi khi tính lại `body`, nên chúng phải rẻ: struct nằm trên stack, không ARC, không kế thừa (không mang theo hàng trăm property như `UIView`). Value semantics giúp SwiftUI so sánh mô tả cũ/mới dễ dàng. State thật được SwiftUI lưu ở bộ nhớ riêng gắn với identity của view, không nằm trong struct.
</details>

**Q3 ⭐ `some View` nghĩa là gì? Vì sao không trả `View` hoặc `AnyView`?**
<details><summary>Đáp án gợi ý</summary>

`some View` là opaque return type: hàm trả một kiểu cụ thể duy nhất mà compiler biết nhưng ẩn với người gọi. Không trả `View` (existential `any View`) được vì `View` có associatedtype `Body`, và SwiftUI cần kiểu tĩnh để biết cấu trúc cây view (ví dụ `VStack<TupleView<(Text, Optional<Image>)>>`) → diff nhanh, structural identity. `AnyView` xoá kiểu, làm SwiftUI mất thông tin cấu trúc, diff kém hơn và dễ mất state/animation.
</details>

**Q4 `@ViewBuilder` là gì? `if/else` trong `body` biến thành gì?**
<details><summary>Đáp án gợi ý</summary>

Là một result builder: gom các câu lệnh trong closure thành một giá trị view. Nhiều view liên tiếp → `TupleView`; `if` không else → `Optional<V>`; `if/else` → `_ConditionalContent<A, B>`; rỗng → `EmptyView`. Nhờ đó trong `body` có thể viết nhiều view và rẽ nhánh mà không cần `return`, và vẫn giữ được kiểu tĩnh. Có thể dùng `@ViewBuilder` cho hàm, computed property và tham số closure của container tự viết.
</details>

**Q5 ⭐ Vì sao thứ tự modifier quan trọng? Cho ví dụ.**
<details><summary>Đáp án gợi ý</summary>

Mỗi modifier bọc view hiện tại thành một view mới (`ModifiedContent`), nên chuỗi modifier là cây lồng nhau. Modifier bên ngoài chỉ "thấy" kết quả của các modifier bên trong. `Text("A").padding().background(.yellow)` → nền phủ cả padding; `.background(.yellow).padding()` → nền chỉ ôm chữ. Tương tự `.frame(width:200).background()` vs `.background().frame(width:200)`, hay gesture đặt trước/sau padding thì vùng chạm khác nhau.
</details>

**Q6 ⭐ Layout trong SwiftUI hoạt động thế nào? Khác Auto Layout ra sao?**
<details><summary>Đáp án gợi ý</summary>

Ba bước: parent đề xuất kích thước cho child; child tự chọn kích thước và báo lên; parent đặt child vào vị trí theo alignment. Child có quyết định cuối cùng về size của mình. Stack chia không gian theo độ linh hoạt của con và `layoutPriority`. Auto Layout thì giải hệ ràng buộc tuyến tính (Cassowary) cho toàn bộ cây, có priority/hugging/compression resistance; xung đột ràng buộc gây lỗi runtime. SwiftUI layout đơn giản, một lượt, không có "ràng buộc mâu thuẫn".
</details>

**Q7 `GeometryReader` dùng khi nào? Có vấn đề gì?**
<details><summary>Đáp án gợi ý</summary>

Dùng khi cần biết kích thước/vị trí parent đề xuất để tính layout tỉ lệ (progress bar, chart). Vấn đề: nó chiếm toàn bộ không gian được đề xuất, đặt con ở góc trên-trái, làm layout khó đoán; đọc size rồi ghi vào state có thể gây vòng lặp layout. Nên đặt trong `.background`/`.overlay`, hoặc dùng thay thế: `frame(maxWidth:)`, `containerRelativeFrame`, `ViewThatFits`, `Layout` protocol, `onGeometryChange` (iOS 18).
</details>

**Q8 ⭐ `@State` hoạt động như thế nào? Vì sao view là struct mà vẫn giữ được state?**
<details><summary>Đáp án gợi ý</summary>

`@State` là property wrapper mà storage thật do SwiftUI quản lý, gắn với identity của view trong cây. Struct chỉ giữ tay cầm. Khi struct bị tạo lại, SwiftUI nối tay cầm với storage cũ nên giá trị còn nguyên. Setter `nonmutating` ghi vào storage và báo SwiftUI đánh dấu các view đọc nó cần vẽ lại. Giá trị khởi tạo chỉ dùng lần đầu tiên view xuất hiện. `$state` cho `Binding`. Nên khai `private` và chỉ dùng cho state do view sở hữu.
</details>

**Q9 ⭐ `@Binding` là gì? Khác truyền giá trị thường thế nào?**
<details><summary>Đáp án gợi ý</summary>

`Binding<T>` là tham chiếu hai chiều (cặp get/set) tới một source of truth nằm ở chỗ khác (thường `@State` của cha, hoặc property của model qua `@Bindable`). Con đọc và ghi được, ghi sẽ cập nhật thẳng source of truth. Truyền giá trị thường (`let`) chỉ là bản sao chỉ-đọc. Dùng binding cho component tái sử dụng (toggle row, stepper) để chúng không sở hữu state. Preview dùng `.constant(...)`.
</details>

**Q10 ⭐⭐ `@StateObject` khác `@ObservedObject` thế nào? Chuyện gì xảy ra nếu dùng `@ObservedObject var vm = ViewModel()`?**
<details><summary>Đáp án gợi ý</summary>

Cả hai đều quan sát `ObservableObject` và re-render khi `objectWillChange` bắn. `@StateObject` **sở hữu** object: nhận autoclosure, tạo object một lần cho mỗi identity view, giữ qua các lần re-render, giải phóng khi view rời cây. `@ObservedObject` **không sở hữu**: object do bên ngoài tạo và giữ. Nếu viết `@ObservedObject var vm = ViewModel()`, mỗi lần cha re-render, struct view được tạo lại → `ViewModel()` mới → mất state, request gọi lại, timer reset. Quy tắc: ai tạo thì `@StateObject`, ai được truyền thì `@ObservedObject`.
</details>

**Q11 `@EnvironmentObject` dùng khi nào? Rủi ro?**
<details><summary>Đáp án gợi ý</summary>

Khi một object cần dùng ở nhiều tầng sâu (session, settings, cart) mà không muốn truyền qua từng view trung gian (prop drilling). Tổ tiên gọi `.environmentObject(obj)`, con cháu khai `@EnvironmentObject var obj: T`, lookup theo kiểu. Rủi ro: dependency ẩn, quên inject → crash runtime (hay gặp trong preview, sheet, view tạo trong UIHostingController); khó test; mọi view dùng nó re-render khi bất kỳ `@Published` nào đổi.
</details>

**Q12 ⭐⭐ `@Observable` khác `ObservableObject` thế nào? Lợi ích chính?**
<details><summary>Đáp án gợi ý</summary>

`@Observable` (iOS 17, macro Observation) tự theo dõi mọi stored property, không cần `@Published`, không phụ thuộc Combine. Getter gọi `registrar.access`, setter gọi `withMutation`. SwiftUI chạy `body` trong `withObservationTracking`, ghi lại chính xác property nào được đọc → chỉ re-render view khi những property đó đổi (theo dõi mức property, chỉ những gì thật sự đọc trong body). `ObservableObject` bắn `objectWillChange` cho cả object → mọi view quan sát re-render dù không dùng property đổi. Ngoài ra Observation theo dõi được qua optional, mảng object, computed property; wrapper đơn giản hơn: `@State`, `let`, `@Bindable`, `@Environment(T.self)`.
</details>

**Q13 ⭐ Với `@Observable`, khi nào dùng `@State`, khi nào `@Bindable`, khi nào không cần wrapper?**
<details><summary>Đáp án gợi ý</summary>

`@State` khi view **tạo và sở hữu** model (thay `@StateObject`). Không cần wrapper (`let model: M`) khi được truyền vào và chỉ đọc/gọi method — tracking vẫn hoạt động vì nó dựa vào việc đọc property trong body. `@Bindable` khi được truyền vào và cần tạo `Binding` tới property (`$model.name` cho `TextField`). Lấy từ environment mà cần binding thì khai `@Bindable var model = model` trong `body`.
</details>

**Q14 Dùng `@State` cho một class `@Observable` có gì cần lưu ý so với `@StateObject`?**
<details><summary>Đáp án gợi ý</summary>

`@State` không có autoclosure: biểu thức khởi tạo `Model()` chạy mỗi lần struct view được init, SwiftUI chỉ giữ instance đầu và vứt các instance sau. Nên `init` của model phải nhẹ, không side effect (không network, không start timer, không đăng ký notification); công việc nặng đặt trong `.task`. `@StateObject` chỉ gọi closure một lần nên không có vấn đề này.
</details>

**Q15 ⭐ "Single source of truth" nghĩa là gì trong SwiftUI? Cho ví dụ vi phạm.**
<details><summary>Đáp án gợi ý</summary>

Mỗi mẩu dữ liệu chỉ có một nơi sở hữu (`@State`, `@StateObject`/model), mọi view khác đọc qua tham số hoặc binding. Vi phạm: copy giá trị của cha vào `@State` của con (`@State var name: String` khởi tạo từ tham số) → con sửa bản copy, cha không biết, và khi cha đổi thì con không cập nhật. Hoặc lưu dữ liệu dẫn xuất (tổng tiền, danh sách đã lọc) thành state riêng thay vì tính từ dữ liệu gốc.
</details>

**Q16 ⭐ Vì sao `ForEach` cần `Identifiable` hoặc `id:`? Dùng `id: \.self` có vấn đề gì?**
<details><summary>Đáp án gợi ý</summary>

SwiftUI cần identity ổn định của mỗi phần tử để biết phần tử nào thêm/xoá/di chuyển — phục vụ animation, giữ state từng hàng, cập nhật đúng hàng. `id: \.self` dùng chính giá trị làm id: nếu có giá trị trùng → id trùng, hành vi không xác định; nếu giá trị thay đổi (sửa tên) → bị coi là phần tử mới, mất state/animation. `indices` làm id còn tệ hơn với dữ liệu động vì xoá một phần tử làm id của các phần tử sau dịch chuyển. Nên dùng id ổn định từ server/DB.
</details>

**Q17 Truyền dữ liệu từ con lên cha trong SwiftUI có những cách nào?**
<details><summary>Đáp án gợi ý</summary>

`@Binding` (con ghi trực tiếp vào state của cha); closure callback (`onSelect: (Item) -> Void` — rõ ràng, dễ test); gọi method trên model chung (`@Observable`/`ObservableObject`) được truyền xuống hoặc lấy từ environment; `PreferenceKey` cho thông tin layout đi lên (tuần 12). Tránh: dùng `NotificationCenter` hay singleton cho luồng dữ liệu UI thường.
</details>

**Q18 Có nên làm việc nặng trong `init` của View hay trong `body`? Vì sao?**
<details><summary>Đáp án gợi ý</summary>

Không. `init` và `body` có thể chạy rất nhiều lần (mỗi lần cha re-render, mỗi lần state liên quan đổi), không theo thời điểm ta kiểm soát. Việc nặng trong đó làm giật UI và lặp request. Network/async đặt trong `.task`; tính toán nặng cache trong model; formatter tạo `static let`. `body` chỉ nên là phép biến đổi rẻ từ state sang mô tả view.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Làm xong Bài tập lớn Todo app, chạy trên simulator
- [ ] Trả lời được ≥ 14/18 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "`@StateObject` vs `@ObservedObject` — và bug khi dùng nhầm"
- [ ] Ghi âm 2 phút: "`@Observable` khác `ObservableObject` thế nào, vì sao hiệu năng tốt hơn"
- [ ] Vẽ tay được sơ đồ data flow + bảng chọn wrapper (§8) không nhìn tài liệu
- [ ] Làm xong 4–5 bài DSA Heap / Priority Queue
- [ ] Side project: màn hình Repo Detail bằng SwiftUI + `@Observable` view model, đã commit
- [ ] Viết 5 dòng tự nhận xét: phần nào còn mơ hồ (layout? wrapper?) → ghi lại để hỏi lại ở tuần 13–14

## 📖 Tài liệu đọc thêm

- [Apple — SwiftUI Tutorials (Introducing SwiftUI)](https://developer.apple.com/tutorials/swiftui)
- [Apple — Managing user interface state](https://developer.apple.com/documentation/swiftui/managing-user-interface-state)
- [Apple — Migrating from the Observable Object protocol to the Observable macro](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)
- WWDC23 — *Discover Observation in SwiftUI*
- WWDC20 — *Data Essentials in SwiftUI* (rất hay cho câu hỏi source of truth)
- WWDC19 — *Building Custom Views with SwiftUI* (layout: parent proposes, child chooses)
- [Swift by Sundell — A guide to SwiftUI's state management system](https://www.swiftbysundell.com/articles/swiftui-state-management-guide/)
- objc.io — *Thinking in SwiftUI* (sách, chương Layout và State rất sâu)
