# Tuần 12 — SwiftUI nâng cao: Identity · Re-render · Navigation · Interop UIKit (25/01 – 31/01/2027)

> **Giai đoạn 3 · UI** · Thời lượng: ~14 giờ · **Kết thúc mốc M3**
> Tuần 11 trả lời "viết SwiftUI thế nào". Tuần này trả lời **"SwiftUI chạy bên trong thế nào"** — đây là chỗ phân biệt người 2 năm với người mới học: *"Vì sao state của tôi bị reset?"*, *"Vì sao view này render lại liên tục?"*, *"Nhúng SwiftUI vào app UIKit có sẵn thế nào?"*. Cuối tuần là **M3**: dựng list + detail bằng cả UIKit và SwiftUI không tra cứu.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích **view identity**: structural identity vs explicit identity (`.id`), và phân biệt **identity – lifetime – value**
- [ ] Giải thích **khi nào `body` được gọi lại** và Observation theo dõi dependency khác `objectWillChange` thế nào; dùng được `Self._printChanges()`
- [ ] Kể được 5 lỗi hiệu năng SwiftUI phổ biến (AnyView, conditional modifier, closure làm mất so sánh, list không lazy, việc nặng trong body) và cách sửa
- [ ] Dựng navigation bằng `NavigationStack(path:)` + `navigationDestination(for:)`, điều hướng bằng code (push, pop, pop to root, deep link)
- [ ] Phân biệt `.task` vs `.onAppear`, giải thích **cancellation** của `.task` và `.task(id:)`
- [ ] Viết custom `EnvironmentKey` và `PreferenceKey` đơn giản
- [ ] Phân biệt animation implicit / explicit, transition, transaction
- [ ] Nhúng SwiftUI vào UIKit (`UIHostingController`) và UIKit vào SwiftUI (`UIViewRepresentable` + `Coordinator`)
- [ ] Trả lời thuyết phục: "Dự án công ty nên chọn SwiftUI hay UIKit?"
- [ ] Qua **M3 self-test** cuối tuần
- [ ] DSA: 4–5 bài Backtracking cơ bản (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 25/01** | 2h | §1 **Identity & lifetime**, §2 **Khi nào body chạy lại**. Bài tập 1, 2 · DSA: Subsets | Ghi chú kết quả `_printChanges` |
| **T3 26/01** | 2h | §3 **Hiệu năng**: AnyView, conditional modifier, equatable, lazy stack. Bài tập 3 · DSA: Permutations | `PerformanceLab.swift` |
| **T4 27/01** | 2h | §4 **NavigationStack**, §5 **Sheet & presentation**. Bài tập 4 · DSA: Combination Sum | `DeepLinkRoot.swift` |
| **T5 28/01** | 2h | §6 **`.task` vs `.onAppear`**, §7 **Environment & PreferenceKey**. Bài tập 5, 6 · DSA: Letter Combinations of a Phone Number | `DebouncedSearch.swift` |
| **T6 29/01** | 1,5h | §8 **Animation**, §9 **Interop UIKit ↔ SwiftUI**. Bài tập 7 · DSA: Generate Parentheses | `MultilineTextView.swift` |
| **T7 30/01** | 4h | §10 **SwiftUI hay UIKit?** + **Bài tập lớn** (list + detail hybrid). Side project: NavigationStack trong SwiftUI + nhúng màn SwiftUI vào UIKit bằng `UIHostingController` | Commit side project |
| **CN 31/01** | 2h | **M3 self-test** (bấm giờ, không tra cứu). Tự trả lời 18 câu phỏng vấn. Tick checklist | Kết quả M3 ghi vào notes |

---

## 📚 Kiến thức cốt lõi

### §1. View identity — gốc rễ của mọi "bug state bị reset"

Tuần 11: view struct bị tạo lại liên tục, còn **state** thì SwiftUI lưu riêng và **gắn với identity**. Vậy SwiftUI biết "cái view này lần trước và lần này là **cùng một** view" bằng cách nào? → **Identity**.

Có 2 loại:

**1. Structural identity (ngầm định)** — xác định bằng **vị trí trong cây view** + **kiểu**. Nhớ lại kiểu của `body` là `VStack<TupleView<(Text, _ConditionalContent<A, B>)>>` — vị trí "phần tử thứ 2 của tuple, nhánh true" là một identity, "nhánh false" là identity **khác**.

**2. Explicit identity (tường minh)** — do bạn gán: `id` trong `ForEach`/`List`, hoặc modifier `.id(value)`.

```swift
struct CounterBox: View {
    @State private var count = 0
    var body: some View {
        Button("Đếm: \(count)") { count += 1 }
    }
}

struct StructuralIdentityDemo: View {
    @State private var isHighlighted = false
    var body: some View {
        VStack {
            // Cách 1: 2 nhánh if/else → 2 identity KHÁC NHAU
            if isHighlighted {
                CounterBox().border(.red)
            } else {
                CounterBox()
            }
            // Cách 2: 1 view, chỉ đổi tham số modifier → CÙNG identity
            CounterBox()
                .border(isHighlighted ? .red : .clear)

            Toggle("Highlight", isOn: $isHighlighted)
        }
    }
}
```
Bấm cả 2 counter lên 5, rồi bật Toggle:
- Counter 1 **về 0** — SwiftUI thấy nhánh `true` là view mới: huỷ view nhánh `false` (cùng state của nó), tạo view mới với `@State` khởi tạo lại. Animation mặc định là **fade** (remove + insert), không phải "đổi màu viền".
- Counter 2 **giữ 5** — cùng vị trí, cùng kiểu, chỉ đổi giá trị → cập nhật tại chỗ.

> **Quy tắc vàng:** *Ưu tiên một view với giá trị thay đổi (ternary trong modifier) hơn là `if/else` sinh ra hai view.* Chỉ dùng `if/else` khi bạn **thật sự muốn** hai view khác nhau (khác kiểu, khác state, muốn transition).

**`.id()` — reset state có chủ đích:**
```swift
struct ProfileForm: View {
    let userID: Int
    @State private var draft = ""
    var body: some View {
        TextField("Ghi chú cho user \(userID)", text: $draft)
    }
}

struct ExplicitIdDemo: View {
    @State private var userID = 1
    var body: some View {
        VStack {
            ProfileForm(userID: userID)
                .id(userID)                  // userID đổi → identity mới → draft reset về ""
            Button("User kế tiếp") { userID += 1 }
        }
    }
}
```
Bỏ `.id(userID)` đi: chuyển user nhưng `draft` của user cũ vẫn còn (đúng bẫy "`@State` chỉ khởi tạo lần đầu" tuần 11). `.id` giải quyết bằng cách nói với SwiftUI "đây là view khác".

⚠️ Đừng lạm dụng `.id(UUID())` hay `.id` đổi liên tục trên `List`/`ScrollView` lớn: mỗi lần đổi là **huỷ và dựng lại toàn bộ** cây con (mất scroll position, mất state, tốn CPU).

**Identity – Lifetime – Value (WWDC21 "Demystify SwiftUI"):**
| Khái niệm | Nghĩa | Ví dụ |
|---|---|---|
| **Value** | Struct view tại một thời điểm. Ngắn ngủi, tạo/huỷ liên tục | Mỗi lần `body` của cha chạy là một `CounterBox` value mới |
| **Identity** | "Đây là cái view nào" — ổn định qua các lần cập nhật | Vị trí trong cây, hoặc `.id`/`ForEach id` |
| **Lifetime** | Khoảng thời gian **identity** tồn tại trên cây: từ khi xuất hiện đến khi bị gỡ | `@State` sống bằng lifetime; `onAppear`/`onDisappear`, `.task` gắn với lifetime |

→ Câu chốt: *"State gắn với identity, nên sống theo lifetime của identity, không theo lifetime của struct value."*

### §2. Khi nào `body` được gọi lại?

SwiftUI duy trì một **dependency graph** (attribute graph). `body` của view V được tính lại khi **một dependency của V thay đổi**:

1. **Input từ cha thay đổi** — cha re-render và tạo V với giá trị property **khác** lần trước. SwiftUI so sánh từng stored property của struct (với kiểu đơn giản là so sánh bộ nhớ; `Equatable` nếu có). Giống hệt → **bỏ qua**, không gọi `body` của V.
2. **`@State` / `@Binding`** mà V **đọc trong body** thay đổi.
3. **`@Environment`** value mà V đọc thay đổi.
4. **Object quan sát** thay đổi — cách tính phụ thuộc vào cơ chế:

| | `ObservableObject` | `@Observable` |
|---|---|---|
| Đăng ký | View có `@StateObject`/`@ObservedObject`/`@EnvironmentObject` subscribe `objectWillChange` | SwiftUI chạy `body` trong `withObservationTracking`, ghi lại **từng keyPath được đọc** |
| Thông báo khi | **Bất kỳ** `@Published` nào sắp đổi | Chỉ khi property **đã được đọc trong lần body gần nhất** đổi |
| Hệ quả | View chỉ hiển thị `title` vẫn render lại khi `tick` đổi | View chỉ đọc `title` **không** render lại khi `tick` đổi |
| Phụ thuộc framework | Combine | Không cần Combine (module `Observation` của Swift) |
| Đi qua cấu trúc lồng | Không — đổi property trong object con không bắn `objectWillChange` của cha | Có — đọc `store.selected?.name` thì track cả `store.selected` và `.name` của object con `@Observable` |

Thực nghiệm (Bài tập 2):
```swift
@Observable
final class DashboardModel {
    var title = "Dashboard"
    var tick = 0
}

struct TitleView: View {
    let model: DashboardModel
    var body: some View {
        let _ = Self._printChanges()     // in lý do body chạy lại (chỉ dùng khi debug)
        Text(model.title)
    }
}

struct TickView: View {
    let model: DashboardModel
    var body: some View { Text("\(model.tick)") }
}

struct DashboardScreen: View {
    @State private var model = DashboardModel()
    var body: some View {
        VStack {
            TitleView(model: model)
            TickView(model: model)
            Button("Tick") { model.tick += 1 }
        }
    }
}
```
Bấm Tick: chỉ `TickView` chạy lại. `DashboardScreen` **không** chạy lại vì body của nó không đọc `model.tick` (closure của Button chỉ *ghi* lúc bấm, không đọc lúc tính body). `TitleView` không in gì sau lần đầu.

**`Self._printChanges()`** in ra các dạng:
- `TitleView: @self changed.` — struct value từ cha khác lần trước.
- `CounterBox: _count changed.` — `@State` tên `count` đổi (có dấu `_`).
- `TitleView: @dependencies changed.` — dependency Observation / environment đổi.
- `... @identity changed.` — view mới xuất hiện (identity mới).

Đây là công cụ #1 khi bị hỏi "làm sao debug view render quá nhiều?" — kèm Instruments → template **SwiftUI** (đếm số lần body chạy; Xcode 26 có thêm biểu đồ *Cause & Effect* cho biết thay đổi nào gây ra update).

> **Lưu ý:** "body chạy lại" ≠ "vẽ lại pixel". Body chạy lại tạo mô tả mới; SwiftUI diff và chỉ cập nhật phần thật sự khác. Body rẻ thì chạy lại cũng không sao — vấn đề là body **đắt** hoặc chạy lại **lan rộng** quá nhiều view.

### §3. Hiệu năng — những lỗi hay gặp và cách sửa

**3.1. `AnyView` — xoá kiểu**
```swift
func makeView(isImage: Bool) -> AnyView {
    if isImage { return AnyView(Image(systemName: "photo")) }
    return AnyView(Text("Không có ảnh"))
}

@ViewBuilder
func makeViewBetter(isImage: Bool) -> some View {
    if isImage { Image(systemName: "photo") } else { Text("Không có ảnh") }
}
```
`AnyView` giấu kiểu bên trong → SwiftUI không biết cấu trúc tĩnh, mỗi lần kiểu bên trong đổi phải **huỷ cả cây con và dựng lại**, không diff được, khó animate. Bản `@ViewBuilder` cho `_ConditionalContent<Image, Text>` — SwiftUI biết rõ 2 nhánh. Chấp nhận `AnyView` khi thật sự cần dị chủng động (ví dụ danh sách widget từ server, factory trả view theo config) — và nên đặt ở mức lá, không bọc cả màn hình.

**3.2. Conditional modifier — bẫy rất phổ biến trên mạng**
```swift
extension View {
    @ViewBuilder
    func `if`<Content: View>(_ condition: Bool, transform: (Self) -> Content) -> some View {
        if condition { transform(self) } else { self }
    }
}

struct ConditionalModifierTrap: View {
    @State private var isError = false
    var body: some View {
        VStack {
            TextField("Email", text: .constant(""))
                .if(isError) { $0.border(.red) }        // ❌ đổi isError = đổi identity → mất focus, state, animation
            TextField("Email", text: .constant(""))
                .border(isError ? .red : .clear)        // ✅ cùng identity, chỉ đổi giá trị
            TextField("Email", text: .constant(""))
                .opacity(isError ? 0.5 : 1)             // ✅ "inert modifier": giá trị trung tính thay vì bỏ modifier
        }
    }
}
```
Extension `.if` trông gọn nhưng sinh `_ConditionalContent` → chính là bẫy `if/else` ở §1. Người dùng đang gõ, validation bật `isError` → TextField **mất focus** vì bị thay bằng view mới. Câu hỏi phỏng vấn yêu thích ở level senior-ish.

**3.3. So sánh view & `Equatable`**
SwiftUI bỏ qua `body` của con nếu input giống lần trước. Nhưng **closure không so sánh được** → view có property là closure luôn bị coi là "đã đổi". Có thể tự định nghĩa so sánh:
```swift
struct ChartView: View, Equatable {
    let points: [Double]
    var onTap: () -> Void = {}

    static func == (lhs: ChartView, rhs: ChartView) -> Bool {
        lhs.points == rhs.points          // bỏ qua closure
    }
    var body: some View {
        HStack(alignment: .bottom, spacing: 2) {
            ForEach(points.indices, id: \.self) { i in      // points không thêm/xoá giữa chừng → indices chấp nhận được
                Rectangle().frame(width: 4, height: points[i])
            }
        }
        .onTapGesture(perform: onTap)
    }
}

struct ChartContainer: View {
    @State private var points: [Double] = [10, 20, 30]
    @State private var unrelated = 0
    var body: some View {
        VStack {
            ChartView(points: points).equatable()   // dùng == của mình
            Button("Unrelated \(unrelated)") { unrelated += 1 }
        }
    }
}
```
Chỉ dùng khi đo được là có vấn đề — `==` sai (quên field) sẽ gây bug "UI không cập nhật".

**3.4. Lazy stacks và List**
```swift
ScrollView {
    LazyVStack(spacing: 12) {                 // chỉ tạo hàng khi sắp hiện lên màn hình
        ForEach(0..<10_000, id: \.self) { i in
            Text("Row \(i)")
        }
    }
}
```
- `VStack` trong `ScrollView` tạo **tất cả** 10.000 view ngay → chậm, tốn RAM. `LazyVStack`/`LazyHStack`/`LazyVGrid` tạo dần khi cuộn tới.
- Lazy stack **không tái sử dụng** view như cell; view đã tạo có thể được giữ lại. `List` (dựa trên `UICollectionView`) có cơ chế reuse → dữ liệu cực lớn thì `List` thường tốt hơn.
- Đừng đặt `.id(...)` thay đổi lên `List`, đừng bọc `ForEach` trong `AnyView` hay `Group` điều kiện phức tạp — SwiftUI cần đếm số hàng nhanh từ `ForEach`. Mỗi phần tử `ForEach` nên sinh ra **số view cố định** (một hàng = một view).

**3.5. Checklist hiệu năng nhanh**
| Triệu chứng | Nguyên nhân thường gặp | Sửa |
|---|---|---|
| Cả màn hình render khi gõ 1 ký tự | `ObservableObject` lớn, mọi view quan sát nó | Chia nhỏ view, truyền giá trị thay vì cả object, hoặc chuyển `@Observable` |
| Mất focus/state khi đổi style | `if/else` hoặc `.if` modifier | Ternary trong modifier |
| Cuộn giật | `VStack` thay vì Lazy; ảnh decode trên main; body nặng | Lazy stack/List; downsample ảnh; cache computed data trong model |
| `body` chạy hàng trăm lần | Ghi state trong `body`/`onAppear` gây vòng lặp; GeometryReader → state → layout | `_printChanges`, bỏ ghi state trong vòng layout |
| `DateFormatter` tạo trong body | Tạo đối tượng đắt mỗi lần | `static let formatter` hoặc `Text(date, format: .dateTime)` |

**Chia nhỏ view** là kỹ thuật tối ưu rẻ nhất: view con chỉ nhận đúng dữ liệu nó cần → khi dữ liệu khác đổi, con không chạy lại body. (Tách thành **struct View riêng**, không phải computed property `var header: some View` — computed property vẫn là một phần body của cha.)

### §4. NavigationStack (iOS 16+)

`NavigationView` đã **deprecated**. `NavigationStack` mô hình hoá điều hướng thành **dữ liệu**: một mảng (path) các giá trị `Hashable`. Push = append, pop = removeLast, pop to root = removeAll. Đây là **navigation driven by state** — rất hợp với declarative.

```swift
struct Repo: Identifiable, Hashable {
    let id: Int
    let name: String
    let owner: String
}

enum Route: Hashable {
    case repo(Repo)
    case owner(String)
    case settings
}

@Observable
final class Router {
    var path: [Route] = []
    func push(_ route: Route) { path.append(route) }
    func pop() { _ = path.popLast() }
    func popToRoot() { path.removeAll() }
}

struct RootView: View {
    @State private var router = Router()
    let repos = [Repo(id: 1, name: "swift", owner: "apple")]

    var body: some View {
        NavigationStack(path: $router.path) {
            List(repos) { repo in
                NavigationLink(repo.name, value: Route.repo(repo))   // link chỉ mang GIÁ TRỊ
            }
            .navigationTitle("Repos")
            .toolbar {
                Button("Cài đặt") { router.push(.settings) }         // điều hướng bằng code
            }
            .navigationDestination(for: Route.self) { route in      // ánh xạ giá trị → màn hình
                switch route {
                case .repo(let repo): RepoDetailView(repo: repo)
                case .owner(let login): Text("Owner \(login)")
                case .settings: Text("Settings")
                }
            }
        }
        .environment(router)
    }
}

struct RepoDetailView: View {
    @Environment(Router.self) private var router
    let repo: Repo
    var body: some View {
        VStack {
            Text(repo.name)
            Button("Xem owner") { router.push(.owner(repo.owner)) }
            Button("Về đầu") { router.popToRoot() }
        }
    }
}
```

**Điểm quan trọng:**
- **Tách "đi đâu" (value) khỏi "hiển thị gì" (destination)**. `NavigationLink(value:)` không tạo view đích ngay (khác `NavigationLink(destination:)` cũ — tạo sẵn view đích cho **mọi hàng**, kể cả không bấm).
- `navigationDestination(for:)` đặt **bên trong** `NavigationStack`, gắn lên view gốc của stack — **không** đặt trong hàng của `List`/`LazyVStack` (view lười chưa được tạo thì destination chưa đăng ký → link không chạy, Xcode cảnh báo).
- Path là `[Route]` (đồng nhất, dễ đọc/ghi, dễ test) hoặc `NavigationPath` (dị chủng, type-erased, có thể `Codable` để lưu/khôi phục):
  ```swift
  struct HeterogeneousPath: View {
      @State private var path = NavigationPath()
      var body: some View {
          NavigationStack(path: $path) {
              VStack {
                  Button("Mở số 42") { path.append(42) }
                  Button("Mở chuỗi") { path.append("hello") }
              }
              .navigationDestination(for: Int.self) { Text("Số \($0)") }
              .navigationDestination(for: String.self) { Text("Chuỗi \($0)") }
          }
      }
  }
  ```
- **Deep link** = dựng path từ URL: `path = [.repo(...), .owner(...)]` (Bài tập 4).
- Truyền **id** thay vì cả object vào route nếu object lớn hoặc có thể thay đổi; màn đích tự load theo id. Đừng nhét object không `Hashable` hoặc closure vào route.
- `NavigationSplitView` cho iPad/Mac (sidebar–detail) — biết tên là đủ ở mức 2 năm.

### §5. Sheet & các kiểu presentation

```swift
struct SheetDemo: View {
    @State private var showSettings = false
    @State private var selectedRepo: Repo?
    let repos = [Repo(id: 1, name: "swift", owner: "apple")]

    var body: some View {
        List(repos) { repo in
            Button(repo.name) { selectedRepo = repo }
        }
        .toolbar { Button("Settings") { showSettings = true } }
        .sheet(isPresented: $showSettings) {           // Bool → sheet
            SettingsSheet()
        }
        .sheet(item: $selectedRepo) { repo in          // Optional<Identifiable> → sheet có dữ liệu
            Text(repo.name)
                .presentationDetents([.medium, .large]) // bottom sheet nửa màn hình (iOS 16)
        }
    }
}

struct SettingsSheet: View {
    @Environment(\.dismiss) private var dismiss
    var body: some View {
        NavigationStack {                               // sheet muốn có nav bar thì tự bọc stack
            Text("Settings")
                .toolbar { Button("Xong") { dismiss() } }
        }
    }
}
```
- Presentation cũng là **state**: sheet hiện khi `showSettings == true`; vuốt xuống đóng → SwiftUI **ghi lại** `false` qua binding.
- **Ưu tiên `sheet(item:)`** khi sheet cần dữ liệu. Bẫy kinh điển: `sheet(isPresented:)` + đọc `selectedRepo` riêng → lần đầu mở sheet thấy dữ liệu **rỗng/cũ** vì closure content được capture trước khi state kia cập nhật.
- `fullScreenCover`, `alert`, `confirmationDialog`, `popover` cùng một mô hình.
- `@Environment(\.dismiss)` đóng sheet **hoặc** pop màn trong NavigationStack — tuỳ ngữ cảnh view đang ở đâu.
- Environment `@Observable` inject ở cha **được** sheet kế thừa (iOS 17). Khi trình bày từ `UIHostingController` mới thì phải inject lại.

### §6. `.task` vs `.onAppear` — và cancellation

| | `.onAppear { }` | `.task { }` |
|---|---|---|
| Closure | Đồng bộ | `async` |
| Muốn gọi async | Phải tự `Task { }` | Viết `await` trực tiếp |
| Huỷ khi view biến mất | **Không** — `Task {}` tự tạo chạy tiếp | **Có** — SwiftUI tự `cancel()` khi view rời cây (lifetime kết thúc) |
| Chạy lại theo giá trị | Không (dùng `onChange`) | `.task(id: value)`: huỷ task cũ, chạy task mới mỗi khi `value` đổi |
| Actor | Main | Kế thừa `@MainActor` của view |

```swift
struct OnAppearBad: View {
    @State private var repos: [Repo] = []
    let service: RepoService
    var body: some View {
        List(repos) { Text($0.name) }
            .onAppear {
                Task { repos = (try? await service.search("swift")) ?? [] }   // ⚠️ không ai huỷ
            }
    }
}

struct TaskGood: View {
    @State private var repos: [Repo] = []
    let service: RepoService
    var body: some View {
        List(repos) { Text($0.name) }
            .task {                                                          // ✅ tự huỷ khi view biến mất
                repos = (try? await service.search("swift")) ?? []
            }
    }
}
```

**Search có debounce bằng `.task(id:)`** — pattern hay được hỏi:
```swift
@MainActor
@Observable
final class RepoSearchModel {
    var query = ""
    var repos: [Repo] = []
    var errorMessage: String?
    private let service: RepoService
    init(service: RepoService) { self.service = service }

    func search() async {
        guard !query.isEmpty else { repos = []; return }
        do {
            try await Task.sleep(for: .milliseconds(300))   // debounce: bị huỷ nếu user gõ tiếp
            repos = try await service.search(query)
        } catch is CancellationError {
            // bình thường — user đã gõ ký tự mới, bỏ qua
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

struct RepoSearchView: View {
    @State var model: RepoSearchModel
    var body: some View {
        List(model.repos) { Text($0.name) }
            .searchable(text: $model.query)
            .task(id: model.query) {          // mỗi ký tự: huỷ task cũ, tạo task mới
                await model.search()
            }
    }
}
```
Cơ chế: gõ "s", "sw", "swi" nhanh → task "s" và "sw" bị cancel trong lúc `Task.sleep` → `Task.sleep` ném `CancellationError` → không gọi API. Chỉ "swi" chạy hết. Lưu ý **cancellation là hợp tác** (tuần 7): `URLSession` tôn trọng cancel, nhưng code CPU-bound của bạn phải tự `try Task.checkCancellation()`.

⚠️ Khi `NavigationStack` push màn mới, màn bên dưới **vẫn còn trong cây** → `.task` của nó **không** bị huỷ. Còn `onAppear`/`onDisappear` có thể gọi nhiều lần (quay lại màn, đổi tab) → đừng đặt "load lần đầu" trong `onAppear` mà không có cờ.

### §7. Environment & PreferenceKey

**Environment** = dữ liệu chảy **xuống** cây (cha → mọi con cháu). Tự tạo key:
```swift
private struct AccentThemeKey: EnvironmentKey {
    static let defaultValue: Color = .blue
}
extension EnvironmentValues {
    var accentTheme: Color {
        get { self[AccentThemeKey.self] }
        set { self[AccentThemeKey.self] = newValue }
    }
}
// Xcode 16+: có thể viết gọn bằng macro
// extension EnvironmentValues { @Entry var accentTheme: Color = .blue }

struct ThemedButton: View {
    @Environment(\.accentTheme) private var accent
    var body: some View { Text("Mua").padding().background(accent) }
}

struct ThemeDemo: View {
    var body: some View {
        VStack {
            ThemedButton()                                   // xanh (default)
            ThemedButton().environment(\.accentTheme, .orange) // cam
        }
    }
}
```
Dùng cho: theme, dependency nhẹ (formatter, feature flags, closure hành động như `openURL`). Giá trị gần nhất trong cây thắng.

**PreferenceKey** = dữ liệu chảy **lên** (con → tổ tiên). Ví dụ kinh điển: căn các label có độ rộng bằng label dài nhất.
```swift
struct MaxWidthKey: PreferenceKey {
    static let defaultValue: CGFloat = 0
    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        value = max(value, nextValue())      // gộp giá trị từ nhiều con
    }
}

struct AlignedLabels: View {
    @State private var labelWidth: CGFloat = 0
    let rows = [("Tên", "An"), ("Email", "an@example.com"), ("Điện thoại", "0123")]
    var body: some View {
        VStack(alignment: .leading) {
            ForEach(rows, id: \.0) { label, value in
                HStack {
                    Text(label)
                        .background(GeometryReader { proxy in
                            Color.clear.preference(key: MaxWidthKey.self, value: proxy.size.width)
                        })
                        .frame(width: labelWidth == 0 ? nil : labelWidth, alignment: .leading)
                    Text(value)
                }
            }
        }
        .onPreferenceChange(MaxWidthKey.self) { labelWidth = $0 }
    }
}
```
Chính SwiftUI dùng preference cho `.navigationTitle` và `.toolbar`: view con khai báo, `NavigationStack` (tổ tiên) đọc lên để vẽ thanh điều hướng. Đó là lý do bạn gắn `.navigationTitle` **trên view con bên trong** stack chứ không gắn lên `NavigationStack`.

### §8. Animation cơ bản

Animation trong SwiftUI = **nội suy giữa hai trạng thái**. Bạn đổi state; nếu thay đổi đó mang theo một `Animation`, SwiftUI nội suy các thuộc tính animatable (frame, opacity, màu, offset, scale...).

```swift
struct AnimationDemo: View {
    @State private var isExpanded = false
    @State private var showBanner = false
    @State private var scale = 1.0

    var body: some View {
        VStack(spacing: 20) {
            // 1. IMPLICIT: gắn vào view, animate khi `value` đổi
            RoundedRectangle(cornerRadius: 12)
                .frame(width: isExpanded ? 300 : 100, height: 100)
                .animation(.spring(duration: 0.4), value: isExpanded)
            Button("Đổi kích thước") { isExpanded.toggle() }

            // 2. EXPLICIT + TRANSITION: bọc thay đổi state trong withAnimation
            if showBanner {
                Text("Đã lưu!")
                    .padding()
                    .background(.green, in: Capsule())
                    .transition(.move(edge: .top).combined(with: .opacity))
            }
            Button("Hiện banner") {
                withAnimation(.easeInOut(duration: 0.3)) { showBanner.toggle() }
            }

            // 3. TRANSACTION: tắt animation cho một thay đổi cụ thể
            Circle().frame(width: 50).scaleEffect(scale)
            Button("Không animate") {
                var transaction = Transaction()
                transaction.disablesAnimations = true
                withTransaction(transaction) { scale = scale == 1 ? 1.5 : 1 }
            }
        }
    }
}
```

| | Implicit `.animation(_:value:)` | Explicit `withAnimation { }` |
|---|---|---|
| Gắn ở đâu | Trên view | Quanh **thay đổi state** |
| Phạm vi | Các thay đổi trong subtree đó khi `value` đổi | **Mọi** view bị ảnh hưởng bởi thay đổi đó |
| Dùng khi | Một view tự animate theo một giá trị | Một hành động người dùng làm nhiều thứ cùng animate, insert/remove view |

- `.animation(_:)` **không có `value:`** đã deprecated vì nó animate mọi thứ, kể cả thay đổi không mong muốn.
- **Transition** chỉ chạy khi view được **chèn vào/gỡ ra** (identity xuất hiện/biến mất) — liên hệ §1: `if/else` = transition, ternary = animation thuộc tính.
- **Transaction**: mỗi lần cập nhật state, SwiftUI tạo một `Transaction` mang `animation` đi xuống cây. `withAnimation` thực chất là `withTransaction` với animation set. Modifier `.transaction { $0.animation = ... }` cho phép view con sửa/tắt animation nhận được.
- Bẫy: animation từ `withAnimation` lan cả sang view không mong muốn (ví dụ list nhảy) → dùng `.transaction { $0.animation = nil }` hoặc `.animation(nil, value:)` cục bộ.

### §9. Interop UIKit ↔ SwiftUI

Ở công ty, **app hybrid** là bình thường: codebase UIKit 5–10 năm, màn mới viết SwiftUI. Phỏng vấn hầu như chắc chắn hỏi phần này.

**9.1. SwiftUI trong UIKit — `UIHostingController`**
```swift
struct RepoDetailSwiftUIView: View {
    let repoName: String
    var body: some View { Text(repoName) }
}

// (a) Push như mọi view controller
final class RepoListViewController: UITableViewController {
    let repos = ["swift", "alamofire"]
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let detail = RepoDetailSwiftUIView(repoName: repos[indexPath.row])
        let hosting = UIHostingController(rootView: detail)
        hosting.title = repos[indexPath.row]
        navigationController?.pushViewController(hosting, animated: true)
    }
}

// (b) Nhúng một phần màn hình: child view controller containment (tuần 9)
final class ContainerViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let hosting = UIHostingController(rootView: RepoDetailSwiftUIView(repoName: "swift"))
        addChild(hosting)                                  // 1
        view.addSubview(hosting.view)                      // 2
        hosting.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hosting.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hosting.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hosting.view.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            hosting.view.heightAnchor.constraint(equalToConstant: 200),
        ])
        hosting.didMove(toParent: self)                    // 3
    }
}

// (c) SwiftUI trong cell UITableView/UICollectionView (iOS 16+)
final class LegacyCell: UITableViewCell {
    func configure(name: String) {
        contentConfiguration = UIHostingConfiguration {
            HStack {
                Image(systemName: "book")
                Text(name)
            }
        }
    }
}
```
- Muốn hosting view tự co theo nội dung trong Auto Layout: `hosting.sizingOptions = .intrinsicContentSize` (iOS 16).
- Truyền dữ liệu UIKit → SwiftUI: tham số `rootView`, hoặc một `@Observable`/`ObservableObject` model mà cả VC và view dùng chung (UIKit đổi model → SwiftUI tự cập nhật). SwiftUI → UIKit: closure callback (`onSelect`) do VC truyền vào.
- Quên `addChild`/`didMove` → hosting không nhận lifecycle, trait (dark mode, dynamic type) và safe area đúng.

**9.2. UIKit trong SwiftUI — `UIViewRepresentable`**

Vòng đời (thuộc lòng):
| Hàm | Gọi khi | Làm gì |
|---|---|---|
| `makeCoordinator()` | **Trước** `makeUIView`, 1 lần | Tạo object làm delegate/target |
| `makeUIView(context:)` | 1 lần cho mỗi identity | Tạo và cấu hình view **một lần**, gán delegate = coordinator |
| `updateUIView(_:context:)` | Lần đầu + **mỗi khi** input/state/environment liên quan đổi | Đẩy state SwiftUI → UIKit. Phải idempotent, so sánh trước khi set |
| `dismantleUIView(_:coordinator:)` (static) | Khi gỡ khỏi cây | Dọn dẹp (observer, timer) |

```swift
struct SearchField: UIViewRepresentable {
    @Binding var text: String
    var placeholder: String = ""
    var onSubmit: () -> Void = {}

    func makeUIView(context: Context) -> UITextField {
        let field = UITextField()
        field.placeholder = placeholder
        field.borderStyle = .roundedRect
        field.returnKeyType = .search
        field.delegate = context.coordinator
        field.addTarget(context.coordinator,
                        action: #selector(Coordinator.textChanged(_:)),
                        for: .editingChanged)
        return field
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        context.coordinator.parent = self             // giữ bản struct MỚI NHẤT (binding, closure)
        if uiView.text != text { uiView.text = text } // so sánh trước khi set: tránh nhảy con trỏ / vòng lặp
        uiView.placeholder = placeholder
    }

    func makeCoordinator() -> Coordinator { Coordinator(parent: self) }

    final class Coordinator: NSObject, UITextFieldDelegate {
        var parent: SearchField
        init(parent: SearchField) { self.parent = parent }

        @objc func textChanged(_ sender: UITextField) {
            parent.text = sender.text ?? ""           // UIKit → SwiftUI qua binding
        }
        func textFieldShouldReturn(_ textField: UITextField) -> Bool {
            parent.onSubmit()
            textField.resignFirstResponder()
            return true
        }
    }
}
```
**Vì sao cần Coordinator?** Struct representable bị tạo lại liên tục và là value type → không thể làm `delegate` (delegate phải là object `NSObject` sống lâu, và `weak var delegate` không giữ được struct). Coordinator là **class** sống cùng lifetime của UIView, làm cầu nối UIKit → SwiftUI (delegate, target-action, KVO).

**9.3. `UIViewControllerRepresentable`** — tương tự, cho view controller (picker ảnh, Mail compose, Safari...):
```swift
struct PhotoPicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> PHPickerViewController {
        var config = PHPickerConfiguration()
        config.filter = .images
        config.selectionLimit = 1
        let picker = PHPickerViewController(configuration: config)
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: PHPickerViewController, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator(parent: self) }

    final class Coordinator: NSObject, PHPickerViewControllerDelegate {
        let parent: PhotoPicker
        init(parent: PhotoPicker) { self.parent = parent }

        func picker(_ picker: PHPickerViewController, didFinishPicking results: [PHPickerResult]) {
            parent.dismiss()
            guard let provider = results.first?.itemProvider,
                  provider.canLoadObject(ofClass: UIImage.self) else { return }
            provider.loadObject(ofClass: UIImage.self) { object, _ in
                let image = object as? UIImage
                DispatchQueue.main.async { self.parent.image = image }   // callback ở background thread
            }
        }
    }
}
// Dùng: .sheet(isPresented: $showPicker) { PhotoPicker(image: $avatar) }
// (Thực tế iOS 16+ đã có PhotosPicker thuần SwiftUI — ví dụ này để học cơ chế.)
```

### §10. SwiftUI hay UIKit trong app công ty thật?

Không có câu trả lời "SwiftUI luôn tốt hơn". Người phỏng vấn muốn nghe bạn **cân nhắc trade-off**:

| Tiêu chí | Nghiêng SwiftUI | Nghiêng UIKit |
|---|---|---|
| Deployment target | iOS 16–17+ | Phải hỗ trợ iOS 14–15 |
| Loại màn hình | Form, settings, detail, onboarding, widget, màn mới | List cực lớn/phức tạp, editor văn bản, camera, custom gesture/transition phức tạp |
| Codebase | Mới, hoặc module độc lập | Cũ lớn, kiến trúc dựa trên VC/Coordinator |
| Team | Đã quen SwiftUI | Đa số chỉ biết UIKit |
| Nhu cầu kiểm soát | Chấp nhận mặc định hệ thống | Cần pixel-perfect, hành vi scroll/keyboard chi tiết |
| Nền tảng | Muốn dùng chung cho watchOS/macOS/widget | Chỉ iPhone |

**Câu trả lời mẫu (30 giây):** *"Ở dự án cũ của em, codebase chủ yếu UIKit với Coordinator. Màn mới như settings, detail em viết SwiftUI và nhúng qua `UIHostingController`, navigation vẫn do Coordinator UIKit quản lý để không phải viết lại flow. Những màn list nặng, cần tối ưu cuộn hoặc custom transition thì giữ UIKit. Cách này cho team tận dụng tốc độ phát triển của SwiftUI mà không phải rewrite rủi ro. Nếu là app mới target iOS 17+ thì em chọn SwiftUI làm mặc định, chỉ bọc UIKit qua representable cho phần SwiftUI còn thiếu."*

(Chỉ nói "ở dự án cũ" nếu đúng là bạn đã làm như vậy — nếu chưa, nói "nếu là em thì em sẽ...". Đừng bịa.)

---

## ⚠️ Bẫy hay gặp

1. **`if/else` hoặc extension `.if` để đổi style** → đổi identity → mất state, mất focus, animation thành fade.
2. **`.id(UUID())`** (tạo mới mỗi lần body chạy) → view bị dựng lại **mỗi lần** render.
3. **Nghĩ "body chạy lại = vẽ lại toàn bộ"** — body chạy lại chỉ tạo mô tả; nhưng body đắt hoặc lan rộng mới là vấn đề.
4. **Computed property `var header: some View`** không giúp tối ưu — nó vẫn thuộc body của cha. Muốn tách dependency phải tạo **struct View riêng**.
5. **`navigationDestination` đặt trong hàng của List/LazyVStack** → không đăng ký được, link không chạy.
6. **`NavigationLink(destination:)` kiểu cũ trong List dài** → khởi tạo view đích (và có thể cả view model) cho mọi hàng.
7. **`onAppear { Task { ... } }`** → không huỷ khi rời màn; `onAppear` gọi lại khi quay về → request lặp.
8. **`sheet(isPresented:)` + đọc state riêng** → dữ liệu rỗng ở lần mở đầu. Dùng `sheet(item:)`.
9. **`updateUIView` set lại mọi thứ không so sánh** → con trỏ TextField nhảy về cuối, vòng lặp delegate → binding → update.
10. **Coordinator giữ `parent` cũ** → closure/binding lỗi thời. Gán `context.coordinator.parent = self` trong `updateUIView`.
11. **Quên `addChild`/`didMove(toParent:)`** khi nhúng `UIHostingController`.
12. **`.animation(.default)` không có `value:`** — deprecated, animate cả những thứ không mong muốn.

---

## 🛠 Bài tập

> Tiếp tục trong project `practice/week11/` hoặc tạo `practice/week12/`. **Không xem lời giải trước khi tự làm ít nhất 20 phút.**

**Bài 1 — Đoán identity (T2)**
Gõ lại `StructuralIdentityDemo` (§1). **Trước khi chạy**, viết ra: bấm 2 counter lên 5, bật toggle — mỗi counter hiển thị số mấy? Animation trông thế nào nếu bọc toggle bằng `withAnimation`? Sau đó chạy để kiểm chứng. Cuối cùng làm `ExplicitIdDemo`: thử có và không có `.id(userID)`.
<details><summary>Lời giải</summary>

Counter 1 (if/else) về 0, counter 2 giữ 5. Với `withAnimation`, counter 1 **fade out/fade in** (transition mặc định `.opacity` vì là remove + insert), counter 2 chỉ đổi màu viền. Không có `.id(userID)`: đổi user nhưng text đang gõ vẫn giữ (state gắn với vị trí, không với userID). Có `.id`: đổi user là ô nhập trống.
</details>

**Bài 2 — Đo re-render: ObservableObject vs @Observable (T2)**
```swift
final class LegacyModel: ObservableObject {
    @Published var title = "A"
    @Published var tick = 0
}

struct LegacyTitle: View {
    @ObservedObject var model: LegacyModel
    var body: some View {
        let _ = Self._printChanges()
        Text(model.title)
    }
}

@Observable final class NewModel {
    var title = "A"
    var tick = 0
}

struct NewTitle: View {
    let model: NewModel
    var body: some View {
        let _ = Self._printChanges()
        Text(model.title)
    }
}

struct CompareScreen: View {
    @StateObject private var legacy = LegacyModel()
    @State private var new = NewModel()
    var body: some View {
        VStack {
            LegacyTitle(model: legacy)
            NewTitle(model: new)
            Button("Tick cả hai") {
                legacy.tick += 1
                new.tick += 1
            }
        }
    }
}
```
Chạy trên simulator, bấm "Tick cả hai" 3 lần, đọc console. Giải thích vì sao. Thêm `let _ = Self._printChanges()` vào `CompareScreen` — nó có chạy lại không? Vì sao?
<details><summary>Lời giải</summary>

Mỗi lần bấm, `LegacyTitle` in ra (lý do `_model changed`) dù chỉ hiển thị `title`: `tick` là `@Published` → `objectWillChange` bắn → mọi subscriber render lại. `NewTitle` **không** in vì body chỉ đọc `title`, Observation không ghi nhận `tick`.
`CompareScreen` **cũng** chạy lại: nó giữ `@StateObject legacy` nên cũng subscribe `objectWillChange` của `legacy`. Khi nó chạy lại, nó tạo `NewTitle(model: new)` mới nhưng input (cùng reference) không đổi → SwiftUI bỏ qua body của `NewTitle`. Bài học: với `ObservableObject`, sự thay đổi lan tới **mọi** view giữ object, kể cả view cha chỉ dùng để truyền xuống.
</details>

**Bài 3 — Phòng thí nghiệm hiệu năng (T3)**
a) Gõ lại `ConditionalModifierTrap` (§3.2), nhưng dùng `@State var email` thật cho cả 3 TextField và một nút toggle `isError`. Click vào TextField 1, gõ vài ký tự, bấm toggle (bằng phím tắt hoặc một nút đặt sao cho không cướp focus — ví dụ đổi `isError` khi `email.count > 5` trong `onChange`). Quan sát TextField nào mất focus.
b) So sánh `ScrollView { VStack { ForEach(0..<5000) ... } }` và `LazyVStack`: thêm `let _ = print("tạo \(i)")` trong row và đếm số dòng in ra khi mở màn.
c) Sửa hàm `makeView(isImage:) -> AnyView` thành `@ViewBuilder`.
<details><summary>Lời giải</summary>

a) Đổi `isError` khi `email.count > 5` bằng `.onChange(of: email) { isError = email.count > 5 }`. TextField dùng `.if` mất focus đúng lúc gõ ký tự thứ 6 (bị thay bằng view mới thuộc nhánh khác); hai TextField còn lại giữ focus.
b) `VStack` in 5000 dòng ngay khi mở; `LazyVStack` chỉ in ~15–20 dòng (số hàng vừa màn hình + phần đệm), in thêm khi cuộn.
c) Xem §3.1.
</details>

**Bài 4 — Navigation + deep link (T4)**
Viết `AppRoute` (`repo(id: Int)`, `owner(login: String)`) với `init?(url: URL)` parse các URL `reposcope://repo/42` và `reposcope://owner/apple` (URL sai trả `nil`). Màn gốc dùng `NavigationStack(path:)`; `.onOpenURL` thay path bằng route parse được. Viết unit test (hoặc chạy script `swift`) cho phần parse.
<details><summary>Lời giải</summary>

```swift
enum AppRoute: Hashable {
    case repo(id: Int)
    case owner(login: String)
}

extension AppRoute {
    init?(url: URL) {
        guard url.scheme == "reposcope" else { return nil }
        let parts = url.pathComponents.filter { $0 != "/" }
        switch (url.host(), parts.first) {
        case ("repo", let idString?):
            guard let id = Int(idString) else { return nil }
            self = .repo(id: id)
        case ("owner", let login?):
            self = .owner(login: login)
        default:
            return nil
        }
    }
}

struct DeepLinkRoot: View {
    @State private var path: [AppRoute] = []
    var body: some View {
        NavigationStack(path: $path) {
            Text("Home")
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .repo(let id): Text("Repo #\(id)")
                    case .owner(let login): Text("Owner \(login)")
                    }
                }
        }
        .onOpenURL { url in
            guard let route = AppRoute(url: url) else { return }
            path = [route]                // deep link = đặt lại path
        }
    }
}
```
Kết quả parse (đã chạy thử): `reposcope://repo/42` → `.repo(id: 42)`, `reposcope://owner/apple` → `.owner(login: "apple")`, `reposcope://repo/abc` → `nil`, `https://repo/1` → `nil`. Test trên simulator: thêm URL scheme `reposcope` trong target → Info → URL Types, rồi chạy `xcrun simctl openurl booted "reposcope://repo/42"`.
Ghi nhớ: URL `scheme://host/path` — "repo" là **host**, "42" là path component.
</details>

**Bài 5 — Search có debounce và huỷ (T5)**
Màn search với `.searchable`. Khi gõ: đợi 400ms không gõ thêm mới "gọi API" (giả lập trả 3 kết quả). Ghi log `"huỷ: <query>"` và `"gọi API: <query>"` vào một mảng để chứng minh các request trung gian bị huỷ. Không dùng Combine.
<details><summary>Lời giải</summary>

```swift
struct DebouncedSearch: View {
    @State private var query = ""
    @State private var results: [String] = []
    @State private var log: [String] = []

    var body: some View {
        List(results, id: \.self) { Text($0) }
            .searchable(text: $query)
            .task(id: query) {
                let q = query                      // chụp giá trị TẠI THỜI ĐIỂM task bắt đầu
                guard !q.isEmpty else { results = []; return }
                do {
                    try await Task.sleep(for: .milliseconds(400))
                } catch {
                    log.append("huỷ: \(q)")
                    return
                }
                log.append("gọi API: \(q)")
                results = (1...3).map { "\(q) #\($0)" }
            }
    }
}
```
Gõ nhanh "swift" → log dạng: `huỷ: s`, `huỷ: sw`, `huỷ: swi`, `huỷ: swif`, `gọi API: swift`. Vì sao phải `let q = query`? Đọc `query` (một `@State`) **sau** `await` sẽ trả giá trị **hiện tại** trong storage, không phải giá trị lúc task bắt đầu → task "s" bị huỷ sẽ log nhầm thành "huỷ: sw". So sánh: bản Combine dùng `$query.debounce(for: .milliseconds(400), scheduler: RunLoop.main)` (tuần 8).
</details>

**Bài 6 — Environment & Preference (T5)**
a) Tạo environment key `\.accentTheme`; màn có 3 nút, nút thứ 3 dùng màu cam bằng `.environment`. Viết lại bằng macro `@Entry` (Xcode 16+).
b) Làm `AlignedLabels` (§7). Thử bỏ `reduce` (để `value = nextValue()`) — chuyện gì xảy ra?
<details><summary>Lời giải</summary>

a) `extension EnvironmentValues { @Entry var accentTheme: Color = .blue }` thay cho cả struct key + computed property.
b) Không lấy max mà lấy giá trị cuối cùng → độ rộng chung = độ rộng label cuối ("Điện thoại" trong ví dụ, tình cờ lại dài nhất; đổi thứ tự rows để thấy label bị cắt). `reduce` quyết định cách gộp giá trị từ nhiều view con.
</details>

**Bài 7 — Bọc UITextView (T6)**
SwiftUI `TextEditor` thiếu nhiều tính năng (data detectors, attributed text...). Viết `MultilineTextView: UIViewRepresentable` bọc `UITextView`, có `@Binding var text: String` và `isEditable`. Dùng nó trong `NoteEditor` có dòng đếm ký tự cập nhật theo thời gian thực.
<details><summary>Lời giải</summary>

```swift
struct MultilineTextView: UIViewRepresentable {
    @Binding var text: String
    var isEditable = true

    func makeUIView(context: Context) -> UITextView {
        let view = UITextView()
        view.font = .preferredFont(forTextStyle: .body)
        view.delegate = context.coordinator
        return view
    }

    func updateUIView(_ uiView: UITextView, context: Context) {
        context.coordinator.text = $text
        if uiView.text != text { uiView.text = text }
        uiView.isEditable = isEditable
    }

    func makeCoordinator() -> Coordinator { Coordinator(text: $text) }

    final class Coordinator: NSObject, UITextViewDelegate {
        var text: Binding<String>
        init(text: Binding<String>) { self.text = text }
        func textViewDidChange(_ textView: UITextView) {
            text.wrappedValue = textView.text
        }
    }
}

struct NoteEditor: View {
    @State private var note = "Ghi chú..."
    var body: some View {
        VStack {
            MultilineTextView(text: $note).frame(height: 200)
            Text("\(note.count) ký tự")
        }
    }
}
```
Thử bỏ `if uiView.text != text` → gõ ở giữa đoạn văn, con trỏ nhảy về cuối (vì mỗi lần gõ: delegate → binding → `updateUIView` → set lại text).
</details>

**🏋️ Bài tập lớn (T7) — Repo list + detail: UIKit, SwiftUI và hybrid**
Làm **3 phiên bản** của cùng một tính năng (dữ liệu hard-code 10 repo: `id`, `name`, `stars`):
1. **Thuần SwiftUI**: `NavigationStack` + `List` + `NavigationLink(value:)` + `navigationDestination`. Detail là `Form` có Toggle "Đã star".
2. **Hybrid**: `UIViewController` chứa `UITableView` (dataSource/delegate, `dequeueReusableCell`, `defaultContentConfiguration`), bấm hàng → push `UIHostingController(rootView: RepoDetail(...))`.
3. **Thuần UIKit** (ôn tuần 10): detail là `UIViewController` với `UIStackView`.
Ghi lại thời gian làm mỗi bản và những chỗ phải tra cứu → đó là danh sách cần ôn.

<details><summary>Lời giải tham khảo (bản 1 và 2)</summary>

```swift
struct Repo: Identifiable, Hashable { let id: Int; let name: String }

struct RepoDetail: View {
    let repo: Repo
    @State private var isStarred = false
    var body: some View {
        Form {
            LabeledContent("Tên", value: repo.name)
            Toggle("Đã star", isOn: $isStarred)
        }
        .navigationTitle(repo.name)
    }
}

// Bản 1: thuần SwiftUI
struct SwiftUIRepoList: View {
    let repos = [Repo(id: 1, name: "swift"), Repo(id: 2, name: "vapor")]
    var body: some View {
        NavigationStack {
            List(repos) { repo in
                NavigationLink(value: repo) { Text(repo.name) }
            }
            .navigationTitle("Repos")
            .navigationDestination(for: Repo.self) { RepoDetail(repo: $0) }
        }
    }
}

// Bản 2: UIKit list → SwiftUI detail
final class ReposViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {
    private let repos = [Repo(id: 1, name: "swift"), Repo(id: 2, name: "vapor")]
    private let tableView = UITableView()

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Repos"
        tableView.frame = view.bounds
        tableView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        tableView.dataSource = self
        tableView.delegate = self
        view.addSubview(tableView)
    }

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { repos.count }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        var content = cell.defaultContentConfiguration()
        content.text = repos[indexPath.row].name
        cell.contentConfiguration = content
        cell.accessoryType = .disclosureIndicator
        return cell
    }

    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        let detail = UIHostingController(rootView: RepoDetail(repo: repos[indexPath.row]))
        navigationController?.pushViewController(detail, animated: true)
    }
}
```
Lưu ý ở bản 2: `.navigationTitle` bên trong SwiftUI vẫn hoạt động khi được push vào `UINavigationController` — `UIHostingController` chuyển nó thành `navigationItem.title`. Bản 1: `isStarred` là `@State` trong detail → quay lại rồi vào lại sẽ mất (vì view bị gỡ khỏi cây) — muốn giữ phải đưa lên model chung. Đây chính là câu hỏi "state sống bao lâu" của §1.
</details>

**📱 Side project (T7) — RepoScope**: (1) chuyển luồng list → detail sang `NavigationStack(path:)` với `enum Route` và một `Router` `@Observable`; (2) ở nhánh UIKit (nếu list đang là UIKit từ tuần 10), push màn detail SwiftUI tuần 11 bằng `UIHostingController`; (3) màn search dùng `.task(id:)` có debounce. Xem [side-project](../tracks/side-project.md).

---

## 🎯 M3 Self-test (CN 31/01) — "List + detail bằng UIKit và SwiftUI không tra cứu"

> Tắt mạng/AI, đóng tài liệu, bấm giờ. Làm trong project trống mới.

**Phần A — Code (90 phút)**
| # | Nhiệm vụ | Thời gian mục tiêu | Đạt khi |
|---|---|---|---|
| A1 | UIKit: `UITableView` hiển thị 20 item, cell dùng `defaultContentConfiguration`, bấm → push detail VC hiển thị tên + nút "Yêu thích" đổi icon | 35' | Chạy được, không crash, không sai reuse (cuộn không lẫn icon) |
| A2 | SwiftUI: cùng tính năng với `NavigationStack` + `navigationDestination`, trạng thái yêu thích lưu ở model `@Observable` dùng chung (quay lại list thấy icon cập nhật) | 25' | Chạy được, yêu thích giữ khi back |
| A3 | Hybrid: từ list UIKit (A1) push detail SwiftUI (A2) bằng `UIHostingController`, dùng chung model | 15' | Đổi yêu thích ở detail SwiftUI → list UIKit cập nhật khi quay lại (gợi ý: `reloadData` trong `viewWillAppear` hoặc closure callback) |
| A4 | Thêm pull-to-refresh vào SwiftUI list (`.refreshable`) và search (`.searchable`) | 15' | Chạy được |

**Phần B — Nói (20 phút, ghi âm)** — mỗi câu ≤ 2 phút:
1. Vòng đời `UIViewController` và khi nào `viewDidLoad` vs `viewWillAppear` (tuần 9)
2. Cell reuse hoạt động thế nào, bug hay gặp (tuần 10)
3. `@StateObject` vs `@ObservedObject`; `@Observable` khác gì (tuần 11)
4. View identity và vì sao state bị reset (tuần 12)
5. Nhúng SwiftUI vào UIKit và ngược lại (tuần 12)

**Chấm điểm:** A1–A3 đạt + nói trôi chảy ≥ 4/5 câu → **qua M3** ✅. Chưa đạt → dùng tuần 13–14 (Tết) để làm lại phần thiếu, ghi rõ vào notes là thiếu phần nào.

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ View identity trong SwiftUI là gì? Structural vs explicit identity?**
<details><summary>Đáp án gợi ý</summary>

Identity là cách SwiftUI xác định "view lần này và lần trước là cùng một view" để giữ state, quyết định cập nhật tại chỗ hay huỷ-tạo mới, và chạy transition. Structural identity: xác định bằng vị trí và kiểu trong cây view (kiểu tĩnh của body, ví dụ nhánh true/false của `_ConditionalContent`). Explicit identity: do ta gán qua `id` trong `ForEach`/`List` hoặc `.id(_:)`. Identity đổi → view cũ bị gỡ (state mất, `onDisappear`, task bị huỷ), view mới được tạo.
</details>

**Q2 ⭐ Vì sao `@State` của tôi bị reset bất ngờ? Kể các nguyên nhân.**
<details><summary>Đáp án gợi ý</summary>

Vì identity của view đổi: view nằm trong nhánh `if/else` khác (hoặc extension `.if`), `.id()` đổi giá trị (đặc biệt `.id(UUID())`), `ForEach` dùng id không ổn định (indices, UUID sinh mới mỗi lần map), cha dùng `AnyView` với kiểu bên trong thay đổi, view bị gỡ khỏi cây rồi thêm lại (đóng sheet, đổi tab trong một số cấu trúc). Với object: dùng `@ObservedObject` tự tạo thay vì `@StateObject`.
</details>

**Q3 Phân biệt identity, lifetime và value của một view.**
<details><summary>Đáp án gợi ý</summary>

Value là struct view tại một thời điểm, ngắn ngủi, tạo lại mỗi lần body của cha chạy. Identity là danh tính ổn định của view qua các lần cập nhật. Lifetime là khoảng thời gian một identity tồn tại trên cây, từ lúc xuất hiện đến khi bị gỡ. State (`@State`, `@StateObject`) và `.task` gắn với lifetime của identity, không với value.
</details>

**Q4 ⭐ Khi nào `body` của một view được tính lại?**
<details><summary>Đáp án gợi ý</summary>

Khi một dependency của view đổi: input từ cha khác giá trị cũ (SwiftUI so sánh các stored property; nếu bằng thì bỏ qua), `@State`/`@Binding` mà body đọc đổi, giá trị environment mà view đọc đổi, object quan sát thay đổi (`ObservableObject`: bất kỳ `@Published` nào; `@Observable`: chỉ property được đọc trong body). Body chạy lại chỉ tạo mô tả mới; SwiftUI diff và chỉ cập nhật phần khác trên màn hình. Debug bằng `Self._printChanges()` và Instruments SwiftUI.
</details>

**Q5 ⭐ Observation tracking khác `objectWillChange` như thế nào về hiệu năng?**
<details><summary>Đáp án gợi ý</summary>

`ObservableObject` thông báo ở mức object: mọi `@Published` đổi đều bắn `objectWillChange`, mọi view giữ object (kể cả cha chỉ để truyền xuống) đều tính lại body. `@Observable` thông báo ở mức property: SwiftUI chạy body trong `withObservationTracking`, ghi lại các keyPath được đọc; chỉ khi chúng đổi mới invalidate view đó. Kết quả là ít re-render hơn, không cần chia nhỏ object chỉ để tối ưu, và tracking xuyên qua object lồng nhau, mảng, optional.
</details>

**Q6 ⭐ Vì sao nên tránh `AnyView`?**
<details><summary>Đáp án gợi ý</summary>

`AnyView` xoá kiểu tĩnh, SwiftUI không biết cấu trúc bên trong nên không diff hiệu quả: khi kiểu bên trong đổi, nó phải huỷ và dựng lại toàn bộ cây con, mất state và animation. Kiểu tĩnh (qua `some View` và `@ViewBuilder` → `_ConditionalContent`) cho phép SwiftUI so sánh nhanh. Chỉ dùng `AnyView` khi thật sự cần dị chủng động, đặt ở cấp nhỏ nhất.
</details>

**Q7 ⭐ Extension "conditional modifier" `.if(condition) { $0.xxx }` có vấn đề gì?**
<details><summary>Đáp án gợi ý</summary>

Nó dùng `if/else` trong `@ViewBuilder`, sinh `_ConditionalContent` → khi điều kiện đổi, view chuyển sang nhánh khác, tức identity khác: mất state (focus TextField, scroll position, `@State` con), animation thành transition fade thay vì nội suy thuộc tính. Cách đúng: luôn áp modifier và đổi tham số (`.border(isError ? .red : .clear)`, `.opacity(x ? 1 : 0)`) — "inert modifier".
</details>

**Q8 `LazyVStack` khác `VStack` và `List` thế nào?**
<details><summary>Đáp án gợi ý</summary>

`VStack` tạo toàn bộ con ngay lập tức — ổn với ít phần tử, tệ với hàng nghìn. `LazyVStack` (trong `ScrollView`) chỉ tạo con khi sắp hiển thị, nhưng không reuse như cell và giữ lại view đã tạo. `List` dựa trên `UICollectionView`, có reuse, separator, swipe actions, edit mode, selection — phù hợp danh sách lớn và chuẩn hệ thống; `LazyVStack` linh hoạt về giao diện tuỳ biến.
</details>

**Q9 ⭐ `NavigationStack` khác `NavigationView` như thế nào? Điều hướng bằng code ra sao?**
<details><summary>Đáp án gợi ý</summary>

`NavigationView` (deprecated) điều hướng bằng `NavigationLink(destination:isActive:)` rải rác, khó điều khiển bằng code và deep link. `NavigationStack` mô hình hoá ngăn xếp màn hình thành dữ liệu: path là mảng giá trị `Hashable` (`[Route]`) hoặc `NavigationPath`. `NavigationLink(value:)` chỉ append giá trị; `navigationDestination(for:)` ánh xạ giá trị thành view. Push = append, pop = removeLast, pop to root = removeAll, deep link = gán path. Path có thể giữ trong router `@Observable` và test được.
</details>

**Q10 Làm deep link `myapp://repo/42` mở thẳng màn detail thế nào?**
<details><summary>Đáp án gợi ý</summary>

Khai URL scheme trong Info.plist (hoặc Universal Links với associated domains). Trong SwiftUI dùng `.onOpenURL { url in ... }` (UIKit: `scene(_:openURLContexts:)`). Parse URL thành route enum (`init?(url:)`, kiểm tra scheme/host/path, trả nil nếu sai). Gán `path = [.repo(id: 42)]` cho `NavigationStack`. Màn detail nhận id và tự load dữ liệu. Parse tách riêng để unit test.
</details>

**Q11 ⭐ `.task` khác `.onAppear` thế nào? `.task(id:)` dùng làm gì?**
<details><summary>Đáp án gợi ý</summary>

`.task` nhận closure async, chạy khi view xuất hiện và **tự huỷ** khi view rời cây (gắn với lifetime). `.onAppear` là closure đồng bộ; muốn async phải tự `Task {}` và task đó không bị huỷ khi rời màn, dễ rò rỉ/cập nhật state của màn đã đóng. `.task(id:)` chạy lại mỗi khi `id` đổi, huỷ task cũ trước — dùng cho search debounce, load lại khi filter đổi. Cancellation là hợp tác: `Task.sleep`, `URLSession` ném `CancellationError`; code tự viết phải kiểm tra `Task.isCancelled`.
</details>

**Q12 `sheet(isPresented:)` và `sheet(item:)` khác nhau thế nào? Khi nào dùng cái nào?**
<details><summary>Đáp án gợi ý</summary>

`isPresented` dựa trên Bool — dùng khi sheet không cần dữ liệu đầu vào. `item` dựa trên Optional `Identifiable` — sheet hiện khi khác nil và nhận chính giá trị đó trong closure, đóng thì gán nil. Dùng `item` khi sheet hiển thị dữ liệu được chọn, tránh lỗi đọc state riêng chưa cập nhật (sheet hiện dữ liệu rỗng lần đầu). Đổi item sang id khác khi đang mở → SwiftUI đóng sheet cũ, mở sheet mới.
</details>

**Q13 Environment và PreferenceKey khác nhau thế nào? Ví dụ thực tế?**
<details><summary>Đáp án gợi ý</summary>

Environment truyền dữ liệu xuống (tổ tiên → con cháu): `colorScheme`, `dismiss`, theme tự định nghĩa, object dùng chung. PreferenceKey truyền dữ liệu lên (con → tổ tiên), có `reduce` để gộp giá trị nhiều con: chính SwiftUI dùng cho `navigationTitle`, `toolbar`; ta dùng để đo kích thước con (căn label bằng nhau, header co giãn khi cuộn).
</details>

**Q14 Implicit và explicit animation khác nhau thế nào? Transition là gì?**
<details><summary>Đáp án gợi ý</summary>

Implicit `.animation(_:value:)` gắn trên view, animate các thay đổi trong subtree khi `value` đổi. Explicit `withAnimation { state change }` gắn animation vào chính thay đổi state, áp dụng cho mọi view bị ảnh hưởng. Bên dưới đều là `Transaction` mang animation đi xuống cây. Transition là animation khi view được chèn vào hoặc gỡ ra (identity xuất hiện/biến mất), ví dụ `.transition(.move(edge:).combined(with: .opacity))`; transition cần thay đổi xảy ra trong một animation.
</details>

**Q15 ⭐ Nhúng SwiftUI vào app UIKit có sẵn như thế nào?**
<details><summary>Đáp án gợi ý</summary>

Dùng `UIHostingController(rootView:)`: push/present như VC thường, hoặc nhúng vào một phần màn hình bằng child view controller containment (`addChild`, add view + constraint, `didMove(toParent:)`); `sizingOptions = .intrinsicContentSize` để tự co theo nội dung. Cell: `UIHostingConfiguration` (iOS 16). Dữ liệu: truyền qua `rootView`, hoặc chia sẻ model `@Observable`/`ObservableObject`; sự kiện SwiftUI → UIKit qua closure callback để Coordinator/VC xử lý navigation.
</details>

**Q16 ⭐ `UIViewRepresentable` hoạt động thế nào? Coordinator để làm gì?**
<details><summary>Đáp án gợi ý</summary>

`makeUIView` tạo UIKit view một lần cho mỗi identity; `updateUIView` gọi mỗi khi state/input liên quan đổi để đẩy dữ liệu SwiftUI → UIKit (phải so sánh trước khi set để tránh vòng lặp và nhảy con trỏ); `dismantleUIView` dọn dẹp. Coordinator là class do `makeCoordinator` tạo, sống cùng UIView, làm delegate/target vì struct representable bị tạo lại liên tục và không thể làm delegate. Coordinator đẩy dữ liệu UIKit → SwiftUI qua binding/closure; nên cập nhật `coordinator.parent = self` trong `updateUIView` để không dùng binding/closure cũ.
</details>

**Q17 ⭐ Dự án mới ở công ty, bạn chọn SwiftUI hay UIKit? Vì sao?**
<details><summary>Đáp án gợi ý</summary>

Phụ thuộc: deployment target (SwiftUI thực sự ổn định từ iOS 16–17), loại màn hình (form, detail, settings → SwiftUI; list cực lớn, editor, camera, gesture/transition phức tạp → UIKit), kinh nghiệm team, codebase sẵn có. Cách thực tế: app mới iOS 17+ thì SwiftUI làm mặc định, bọc UIKit khi thiếu; app cũ thì viết màn mới bằng SwiftUI nhúng qua `UIHostingController`, giữ navigation/Coordinator UIKit, không rewrite hàng loạt. Nêu rủi ro: khác biệt hành vi giữa các phiên bản iOS, debug hiệu năng re-render.
</details>

**Q18 Làm sao debug một màn SwiftUI bị giật khi cuộn?**
<details><summary>Đáp án gợi ý</summary>

Đo trước: Instruments (template SwiftUI + Time Profiler, Hangs/Animation Hitches), `Self._printChanges()` để xem view nào render lại và vì sao. Nguyên nhân hay gặp: `VStack` thay vì lazy/List, body nặng (format date, filter/sort mảng lớn trong body), decode/resize ảnh trên main thread, `ObservableObject` lớn làm cả màn render, identity không ổn định (`id: \.self`, UUID mới) làm hàng bị dựng lại, `AnyView`. Sửa: lazy container, đưa tính toán vào model và cache, downsample ảnh ở background, chia nhỏ view, chuyển `@Observable`, id ổn định.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Làm xong Bài tập lớn (3 phiên bản list + detail), ghi thời gian từng bản
- [ ] **M3 self-test**: A1–A3 đạt, nói trôi ≥ 4/5 câu
- [ ] Trả lời được ≥ 14/18 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "View identity là gì và vì sao state bị reset"
- [ ] Ghi âm 2 phút: "Chọn SwiftUI hay UIKit cho dự án công ty"
- [ ] Làm xong 4–5 bài DSA Backtracking
- [ ] Side project: NavigationStack + Router, màn SwiftUI nhúng vào UIKit qua `UIHostingController`, đã commit
- [ ] Ghi danh sách chủ đề tuần 1–12 còn yếu → mang sang tuần 13–14 ôn

## 📖 Tài liệu đọc thêm

- WWDC21 — *Demystify SwiftUI* (identity, lifetime, dependencies — **bắt buộc xem**)
- WWDC23 — *Demystify SwiftUI performance*
- WWDC22 — *The SwiftUI cookbook for navigation*
- WWDC23 — *Wind your way through advanced animations in SwiftUI* / WWDC23 *Explore SwiftUI animation*
- WWDC22 — *Use SwiftUI with UIKit*
- [Apple — Migrating to new navigation types](https://developer.apple.com/documentation/swiftui/migrating-to-new-navigation-types)
- [Apple — UIViewRepresentable](https://developer.apple.com/documentation/swiftui/uiviewrepresentable)
- objc.io — *Thinking in SwiftUI* (chương View Trees, Animations)
- [SwiftUI Lab — Inspecting the View Tree (PreferenceKey)](https://swiftui-lab.com/communicating-with-the-view-tree-part-1/)
