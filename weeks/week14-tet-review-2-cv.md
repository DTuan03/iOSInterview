# Tuần 14 — 🧧 Ôn tập Giai đoạn 3 + Viết CV bản nháp (08/02 – 14/02/2027)

> **Tuần nhẹ** · Thời lượng: ~7 giờ cả tuần (đầu tuần vẫn là Tết: mùng 3 → mùng 9)
> Nửa đầu tuần: flashcard **UIKit + SwiftUI** (tuần 9–12), nhẹ nhàng giữa những buổi đi chúc Tết. Nửa sau: **viết CV bản nháp** — việc cần đầu óc tỉnh táo nhưng không cần học thêm kiến thức mới. Cuối tuần là **mốc M4**.
> Tuần 17–18 bạn sẽ bắt đầu rải CV, nên CV phải có **trước** đó vài tuần để sửa dần.

## 🎯 Mục tiêu tuần

- [ ] Đi hết **~56 flashcard** tuần 9–12, đi lại thẻ sai lần 2
- [ ] Ôn lại các thẻ còn sai từ tuần 13
- [ ] Viết xong **CV bản nháp tiếng Anh** (1 trang) theo template bên dưới
- [ ] Viết lại ít nhất **6 bullet** kinh nghiệm theo công thức *Action + Tech + Result*
- [ ] Dọn GitHub (pin repo, README RepoScope có screenshot) và LinkedIn
- [ ] DSA: làm lại 3 bài cũ (xem [dsa-plan](../tracks/dsa-plan.md))
- [ ] Qua **M4 self-test** (CV + flashcard Giai đoạn 1–3)

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 08/02** (mùng 3) | 0–30' | 🧧 Vẫn Tết. Tuỳ chọn: flashcard **Tuần 9** | — |
| **T3 09/02** (mùng 4) | 45' | Flashcard **Tuần 9–10**. DSA làm lại: Best Time to Buy and Sell Stock | Danh sách thẻ sai |
| **T4 10/02** (mùng 5) | 45' | Flashcard **Tuần 11–12**. Đi lại thẻ sai tuần 13 | Danh sách thẻ sai |
| **T5 11/02** (mùng 6) | 1h | Đọc §CV 1–3. **Brain dump**: liệt kê mọi việc đã làm ở công ty (xem §CV 3.1). DSA làm lại: Merge Intervals | File `cv-braindump.md` |
| **T6 12/02** (mùng 7) | 1h | §CV 4–5: viết bullet theo công thức, viết mục Projects cho RepoScope. DSA làm lại: Number of Islands | 6–8 bullet đã viết |
| **T7 13/02** (mùng 8) | 2,5h | Điền **CV template** đầy đủ (§CV 9), xuất PDF. §CV 6–7: dọn GitHub, LinkedIn. Side project: README RepoScope có screenshot + sơ đồ kiến trúc | `CV_v0.pdf`, README đã push |
| **CN 14/02** (mùng 9) | 1,5h | **M4 self-test**. Đi lại thẻ sai lần 2. Nhờ 1 người (bạn/đồng nghiệp cũ) đọc CV trong 30 giây và nói họ nhớ được gì. Tick checklist | Checklist ≥ 80% |

---

## 🃏 Flashcards Giai đoạn 3

> Cách dùng như tuần 13: nói to câu trả lời → mới mở đáp án → ghi số thẻ sai.

### Tuần 9 — App lifecycle · VC lifecycle · RunLoop · Responder chain · Render

**1.** Các trạng thái của app iOS?
<details><summary>Đáp án</summary>Not running → Inactive (đang chuyển, không nhận event: cuộc gọi đến, kéo Control Center) → Active (foreground, nhận event) → Background (chạy code giới hạn thời gian) → Suspended (trong RAM nhưng không chạy code, có thể bị hệ thống kill bất cứ lúc nào mà không báo).</details>

**2.** AppDelegate vs SceneDelegate?
<details><summary>Đáp án</summary>Từ iOS 13: AppDelegate lo sự kiện cấp **process** (launch, push token, cấu hình SDK). SceneDelegate lo vòng đời **từng cửa sổ/scene** (foreground/background, UI) — hỗ trợ multi-window trên iPad. SwiftUI dùng `App` + `@UIApplicationDelegateAdaptor` khi cần AppDelegate.</details>

**3.** Thứ tự vòng đời UIViewController?
<details><summary>Đáp án</summary>`init` → `loadView` → `viewDidLoad` → `viewWillAppear` → (`viewIsAppearing`) → `viewWillLayoutSubviews` → `viewDidLayoutSubviews` → `viewDidAppear` → … → `viewWillDisappear` → `viewDidDisappear` → `deinit` (nếu không còn ai giữ).</details>

**4.** Đặt code gì ở `viewDidLoad`, gì ở `viewWillAppear`?
<details><summary>Đáp án</summary>`viewDidLoad` (1 lần): setup view hierarchy, constraint, binding, gọi load dữ liệu ban đầu. `viewWillAppear` (mỗi lần hiện): refresh dữ liệu có thể đã đổi, cập nhật nav bar, bắt đầu observe. Geometry cuối cùng chưa chắc có ở cả hai — dùng `viewIsAppearing` (iOS 17, back-deploy iOS 13) hoặc `viewDidLayoutSubviews`.</details>

**5.** Khi nào override `loadView`? Lưu ý gì?
<details><summary>Đáp án</summary>Khi muốn tự tạo root view bằng code (gán `self.view = MyCustomView()`). **Không** gọi `super.loadView()` trong trường hợp này và không truy cập `view` trước khi gán.</details>

**6.** `frame` vs `bounds`?
<details><summary>Đáp án</summary>`frame`: vị trí + kích thước trong hệ toạ độ của **superview**. `bounds`: trong hệ toạ độ của **chính nó** (origin thường 0,0; đổi origin = cuộn nội dung như UIScrollView). Xoay view (transform) thì frame đổi, bounds không.</details>

**7.** RunLoop là gì? Vì sao `Timer` dừng khi cuộn?
<details><summary>Đáp án</summary>Vòng lặp xử lý sự kiện của một thread: chờ input source/timer, xử lý, ngủ. Main RunLoop chạy ở mode `.default`, khi cuộn chuyển sang `.tracking`. Timer đăng ký ở `.default` không chạy trong `.tracking` → thêm vào `.common` modes (`RunLoop.main.add(timer, forMode: .common)`).</details>

**8.** Responder chain là gì?
<details><summary>Đáp án</summary>Chuỗi `UIResponder` (view → superview → … → view controller → window → UIApplication → AppDelegate) mà event/action đi qua nếu responder hiện tại không xử lý. First responder nhận keyboard input. `sendAction(_:to: nil, ...)` gửi action lên chain.</details>

**9.** Hit-testing hoạt động thế nào?
<details><summary>Đáp án</summary>Từ window, `hitTest(_:with:)` đệ quy xuống subview (duyệt từ subview trên cùng), dùng `point(inside:with:)` kiểm tra. Bỏ qua view `isHidden`, `alpha < 0.01`, `isUserInteractionEnabled == false`, và phần nằm ngoài bounds của superview. View sâu nhất chứa điểm chạm thành hit-test view.</details>

**10.** Làm sao tăng vùng chạm của nút nhỏ mà không đổi giao diện?
<details><summary>Đáp án</summary>Override `point(inside:with:)` trả `bounds.insetBy(dx: -10, dy: -10).contains(point)`. (Hoặc tăng kích thước nút và giữ icon nhỏ.)</details>

**11.** `setNeedsLayout` vs `layoutIfNeeded` vs `layoutSubviews`?
<details><summary>Đáp án</summary>`setNeedsLayout`: đánh dấu cần layout, sẽ làm ở chu kỳ cập nhật kế tiếp (rẻ, gộp nhiều lần). `layoutIfNeeded`: nếu đang được đánh dấu thì layout **ngay** (dùng trong animation block). `layoutSubviews`: nơi override để tự tính frame; **không** gọi trực tiếp.</details>

**12.** Render pipeline của iOS gồm những bước nào?
<details><summary>Đáp án</summary>Trong app: Layout → Display (draw) → Prepare (decode ảnh) → Commit (gửi layer tree cho render server). Render server (process riêng) render bằng GPU → hiển thị. Phải xong trong ngân sách một frame (16,67ms ở 60Hz, 8,33ms ở 120Hz), trễ → **hitch** (giật).</details>

**13.** UIView vs CALayer?
<details><summary>Đáp án</summary>CALayer lo **hiển thị** (nội dung, border, shadow, animation) và được Core Animation render. UIView bọc layer và thêm xử lý **sự kiện** (responder), Auto Layout, accessibility. Mỗi UIView có một `layer`; view là delegate của layer.</details>

**14.** App vào background còn chạy code được bao lâu?
<details><summary>Đáp án</summary>Rất ngắn (vài giây). Xin thêm bằng `beginBackgroundTask` (khoảng ~30 giây, không đảm bảo). Việc dài hơn: `BGTaskScheduler` (BGAppRefresh / BGProcessing), background `URLSession`, hoặc các background mode đặc biệt (audio, location, VoIP).</details>

### Tuần 10 — Auto Layout · TableView/CollectionView · Diffable · Scroll performance

**15.** Mỗi constraint Auto Layout là phương trình gì?
<details><summary>Đáp án</summary>`item1.attribute = multiplier × item2.attribute + constant` (hoặc `≤`, `≥`), kèm priority. Auto Layout giải hệ phương trình (Cassowary) để ra frame.</details>

**16.** Intrinsic content size là gì?
<details><summary>Đáp án</summary>Kích thước "tự nhiên" view muốn có dựa trên nội dung (UILabel theo chữ, UIButton theo title + inset, UIImageView theo ảnh). Có nó thì không cần constraint width/height. View tự viết override `intrinsicContentSize` và gọi `invalidateIntrinsicContentSize()` khi đổi.</details>

**17.** Content hugging vs compression resistance?
<details><summary>Đáp án</summary>Hugging: chống bị **kéo giãn** lớn hơn intrinsic size (priority cao = "ôm sát"). Compression resistance: chống bị **ép nhỏ** hơn intrinsic size (priority cao = khó bị cắt). Ví dụ 2 label cạnh nhau: label thời gian có compression resistance cao để không bị cắt.</details>

**18.** Vì sao phải set `translatesAutoresizingMaskIntoConstraints = false`?
<details><summary>Đáp án</summary>Mặc định `true` với view tạo bằng code: UIKit tự sinh constraint từ frame/autoresizing mask → xung đột với constraint bạn thêm (lỗi "Unable to simultaneously satisfy constraints").</details>

**19.** Ambiguous layout vs unsatisfiable constraints?
<details><summary>Đáp án</summary>Ambiguous: **thiếu** constraint → nhiều nghiệm, view có thể ở vị trí lạ (dùng View Debugger, `hasAmbiguousLayout`). Unsatisfiable: constraint **mâu thuẫn** → hệ thống phá bớt một constraint, log lỗi trong console (đặt symbolic breakpoint `UIViewAlertForUnsatisfiableConstraints`).</details>

**20.** Animate thay đổi constraint thế nào?
<details><summary>Đáp án</summary>Đổi `constraint.constant` (hoặc activate/deactivate) **ngoài** block, rồi `UIView.animate(withDuration:) { self.view.layoutIfNeeded() }` — gọi trên superview chung.</details>

**21.** Cơ chế cell reuse hoạt động thế nào?
<details><summary>Đáp án</summary>TableView chỉ tạo đủ cell cho phần hiển thị (+ chút đệm). Cell cuộn ra khỏi màn hình được đưa vào **reuse pool** theo identifier; `dequeueReusableCell(withIdentifier:for:)` lấy cell từ pool (hoặc tạo mới nếu trống). Tiết kiệm bộ nhớ và thời gian tạo view.</details>

**22.** Bug "ảnh hiện sai hàng" khi cuộn nhanh — nguyên nhân và sửa?
<details><summary>Đáp án</summary>Cell bắt đầu tải ảnh async, bị reuse cho hàng khác trước khi tải xong → ảnh cũ gán vào cell mới. Sửa: reset ảnh trong `prepareForReuse` + huỷ request; khi tải xong kiểm tra cell vẫn đang hiển thị đúng item (so sánh id/URL đã lưu trong cell) trước khi gán.</details>

**23.** `prepareForReuse` nên làm gì và không nên làm gì?
<details><summary>Đáp án</summary>Nên: reset trạng thái tạm (ảnh, huỷ task tải ảnh, trạng thái selected/animation). Không nên: cấu hình nội dung mới (việc đó của `cellForRowAt`), việc nặng.</details>

**24.** Diffable data source giải quyết vấn đề gì?
<details><summary>Đáp án</summary>Thay `reloadData`/`performBatchUpdates` thủ công (dễ crash "invalid number of rows"). Bạn tạo `NSDiffableDataSourceSnapshot` (section + item identifier `Hashable`) rồi `apply` → hệ thống tự tính diff và animate. Một nguồn sự thật cho dữ liệu hiển thị.</details>

**25.** Item identifier trong diffable nên là gì? `reconfigureItems` vs `reloadItems`?
<details><summary>Đáp án</summary>Nên là **id ổn định** (vd. `Repo.ID`), không phải cả model — nếu model là identifier thì đổi 1 field = item mới (xoá + chèn). Nội dung đổi mà id giữ nguyên → `reconfigureItems` (iOS 15, cập nhật cell hiện có, rẻ) thay vì `reloadItems` (tạo/dequeue cell mới).</details>

**26.** Compositional layout gồm những tầng nào?
<details><summary>Đáp án</summary>Item → Group → Section → Layout. Kích thước bằng `NSCollectionLayoutDimension` (`.fractionalWidth`, `.absolute`, `.estimated`). Hỗ trợ cuộn ngang trong section (`orthogonalScrollingBehavior`), header/footer qua supplementary items.</details>

**27.** Self-sizing cell cần điều kiện gì?
<details><summary>Đáp án</summary>Constraint trong `contentView` nối **liên tục từ trên xuống dưới** (top → bottom), để Auto Layout tính được chiều cao; `rowHeight = UITableView.automaticDimension` (mặc định từ iOS 11) và ước lượng `estimatedRowHeight`.</details>

**28.** Kể các nguyên nhân scroll giật trong list UIKit.
<details><summary>Đáp án</summary>Việc nặng trên main trong `cellForRow` (format, parse, tính layout); decode ảnh lớn trên main; ảnh to hơn kích thước hiển thị (cần downsample); **offscreen rendering** (shadow không có `shadowPath`, `mask`, `cornerRadius` + `masksToBounds` trên nội dung phức tạp); view hierarchy quá sâu; Auto Layout phức tạp trong cell.</details>

**29.** Offscreen rendering là gì? Cách giảm?
<details><summary>Đáp án</summary>GPU phải render layer ra buffer riêng ngoài màn hình rồi mới ghép → tốn thêm pass. Do shadow không có path, mask, group opacity... Giảm: đặt `layer.shadowPath`, dùng ảnh đã bo góc sẵn, tránh mask; kiểm tra bằng Simulator → Debug → Color Off-screen Rendered.</details>

**30.** Prefetching API dùng làm gì?
<details><summary>Đáp án</summary>`UITableViewDataSourcePrefetching` / `UICollectionViewDataSourcePrefetching`: `prefetchRowsAt` báo trước các hàng sắp hiện → bắt đầu tải ảnh/dữ liệu sớm; `cancelPrefetchingForRowsAt` → huỷ khi người dùng đổi hướng cuộn.</details>

**31.** UITableView vs UICollectionView — chọn cái nào?
<details><summary>Đáp án</summary>TableView: danh sách một cột dọc, có sẵn style/swipe/section index. CollectionView: mọi layout (grid, ngang, phức hợp) qua compositional layout; từ iOS 14 có `UICollectionLayoutListConfiguration` làm list giống table. Code mới thường dùng CollectionView + diffable + compositional.</details>

### Tuần 11 — SwiftUI nền tảng

**32.** View trong SwiftUI là gì? Vì sao là struct?
<details><summary>Đáp án</summary>Mô tả nhẹ, bất biến về UI — không phải đối tượng trên màn hình. Bị tạo lại liên tục nên phải rẻ: struct, không ARC, không kế thừa. State thật được SwiftUI lưu riêng, gắn với identity.</details>

**33.** `some View` và kiểu thật của `body`?
<details><summary>Đáp án</summary>Opaque type: một kiểu cụ thể duy nhất mà compiler biết, ví dụ `VStack<TupleView<(Text, Optional<Image>)>>`. Kiểu tĩnh mã hoá cấu trúc cây → SwiftUI diff nhanh và xác định structural identity.</details>

**34.** `@ViewBuilder` biến `if` và `if/else` thành gì?
<details><summary>Đáp án</summary>`if` → `Optional<V>`; `if/else` → `_ConditionalContent<A, B>`; nhiều view liên tiếp → `TupleView`.</details>

**35.** Vì sao thứ tự modifier quan trọng?
<details><summary>Đáp án</summary>Mỗi modifier bọc view thành `ModifiedContent` mới; modifier ngoài chỉ thấy kết quả bên trong. `.padding().background()` nền phủ cả padding; `.background().padding()` thì không.</details>

**36.** Thuật toán layout của SwiftUI?
<details><summary>Đáp án</summary>Parent đề xuất kích thước → child tự chọn kích thước của nó → parent đặt child theo alignment. Child có quyết định cuối cùng về size.</details>

**37.** `.frame(width: 200)` có ép con rộng 200 không?
<details><summary>Đáp án</summary>Không. Nó tạo view bọc rộng 200, đề xuất 200 cho con; con (vd. Text ngắn) vẫn chọn size nhỏ hơn và được căn giữa trong khung.</details>

**38.** `@State` lưu giá trị ở đâu? Giá trị khởi tạo dùng khi nào?
<details><summary>Đáp án</summary>Trong storage do SwiftUI quản lý, gắn với identity của view — không nằm trong struct. Giá trị khởi tạo chỉ dùng lần đầu view xuất hiện; cha truyền giá trị mới sau đó không làm `@State` đổi.</details>

**39.** `@Binding` là gì?
<details><summary>Đáp án</summary>Tham chiếu hai chiều (get/set) tới source of truth ở nơi khác. Con đọc/ghi được nhưng không sở hữu. Tạo bằng `$state`; preview dùng `.constant(x)`.</details>

**40.** `@StateObject` vs `@ObservedObject`?
<details><summary>Đáp án</summary>`@StateObject` sở hữu: tạo object một lần (autoclosure) cho mỗi identity, giữ qua re-render. `@ObservedObject` không sở hữu: object được truyền vào. `@ObservedObject var vm = VM()` → mỗi lần cha render tạo VM mới, mất state.</details>

**41.** `@Observable` khác `ObservableObject`?
<details><summary>Đáp án</summary>Macro (iOS 17) tự theo dõi mọi stored property, không cần `@Published`/Combine; SwiftUI chỉ re-render view khi property **mà body đã đọc** thay đổi (mức property), thay vì mọi `@Published` (mức object).</details>

**42.** Wrapper tương ứng trong thế giới Observation?
<details><summary>Đáp án</summary>Sở hữu: `@State var m = M()`. Được truyền, chỉ đọc: `let m: M`. Được truyền, cần `$`: `@Bindable var m: M`. Environment: `.environment(m)` + `@Environment(M.self)`.</details>

**43.** Lưu ý khi dùng `@State` cho class `@Observable`?
<details><summary>Đáp án</summary>`@State` không có autoclosure → `M()` chạy lại mỗi lần struct view init (chỉ instance đầu được giữ). `init` của model phải rẻ, không side effect; load dữ liệu trong `.task`.</details>

**44.** Single source of truth là gì?
<details><summary>Đáp án</summary>Mỗi dữ liệu có đúng một nơi sở hữu; các view khác đọc qua tham số hoặc binding. Không copy vào `@State` của con, không lưu dữ liệu dẫn xuất thành state riêng.</details>

**45.** Vì sao `ForEach` cần id ổn định? Rủi ro của `id: \.self` và `indices`?
<details><summary>Đáp án</summary>Để biết phần tử nào thêm/xoá/di chuyển — giữ state hàng, animation đúng. `\.self` trùng giá trị → id trùng; giá trị đổi → bị coi là phần tử mới. `indices` → xoá một phần tử làm id các phần tử sau dịch, state nhảy hàng, có thể crash.</details>

### Tuần 12 — SwiftUI nâng cao

**46.** Structural vs explicit identity?
<details><summary>Đáp án</summary>Structural: theo vị trí + kiểu trong cây (nhánh `if/else` khác nhau = identity khác). Explicit: do ta gán qua `ForEach` id hoặc `.id()`. Identity đổi → view cũ bị gỡ, state mất, view mới tạo.</details>

**47.** Dùng `.id()` để làm gì? Lạm dụng thì sao?
<details><summary>Đáp án</summary>Reset state có chủ đích (đổi user → form trống). Lạm dụng (`.id(UUID())`, `.id` đổi liên tục trên List) → dựng lại toàn bộ cây con mỗi lần, mất scroll/state, tốn CPU.</details>

**48.** Identity – lifetime – value?
<details><summary>Đáp án</summary>Value: struct view tại một thời điểm, tạo lại liên tục. Identity: danh tính ổn định. Lifetime: thời gian identity tồn tại trên cây. State và `.task` gắn với lifetime của identity.</details>

**49.** Khi nào `body` chạy lại?
<details><summary>Đáp án</summary>Input từ cha khác trước, `@State`/`@Binding` được đọc thay đổi, environment được đọc thay đổi, object quan sát thay đổi (ObservableObject: mọi `@Published`; Observable: property đã đọc). Debug bằng `Self._printChanges()`.</details>

**50.** Vấn đề của extension `.if(condition) { ... }`?
<details><summary>Đáp án</summary>Sinh `_ConditionalContent` → điều kiện đổi = identity đổi → mất state/focus, animation thành fade. Dùng ternary trong modifier (`.border(x ? .red : .clear)`).</details>

**51.** Vì sao tránh `AnyView`?
<details><summary>Đáp án</summary>Xoá kiểu tĩnh → SwiftUI không diff được cấu trúc, kiểu bên trong đổi thì dựng lại toàn bộ, mất state/animation. Dùng `@ViewBuilder`/`some View`.</details>

**52.** `NavigationStack(path:)` hoạt động thế nào?
<details><summary>Đáp án</summary>Path là mảng giá trị `Hashable` (hoặc `NavigationPath`). `NavigationLink(value:)` append giá trị; `navigationDestination(for:)` ánh xạ giá trị → view. Push = append, pop = removeLast, pop to root = removeAll, deep link = gán path.</details>

**53.** `.task` vs `.onAppear`?
<details><summary>Đáp án</summary>`.task` async, tự huỷ khi view rời cây; `.task(id:)` huỷ & chạy lại khi id đổi (search debounce). `.onAppear` đồng bộ; `Task {}` bên trong không bị huỷ khi rời màn.</details>

**54.** `sheet(item:)` hơn `sheet(isPresented:)` ở đâu?
<details><summary>Đáp án</summary>Sheet nhận chính item được chọn trong closure → tránh lỗi đọc state riêng chưa cập nhật (sheet rỗng lần đầu). Đóng sheet → item thành nil.</details>

**55.** Environment vs PreferenceKey?
<details><summary>Đáp án</summary>Environment: dữ liệu đi **xuống** (theme, dismiss, model dùng chung). PreferenceKey: dữ liệu đi **lên** từ con tới tổ tiên, có `reduce` gộp giá trị (navigationTitle, đo kích thước con).</details>

**56.** Implicit vs explicit animation? Transition?
<details><summary>Đáp án</summary>Implicit `.animation(_:value:)` gắn trên view, animate khi value đổi. Explicit `withAnimation {}` gắn vào thay đổi state, ảnh hưởng mọi view liên quan. Transition: animation khi view được chèn/gỡ (identity xuất hiện/biến mất).</details>

**57.** Nhúng SwiftUI vào UIKit?
<details><summary>Đáp án</summary>`UIHostingController(rootView:)` — push/present, hoặc child VC containment (`addChild` → add view + constraint → `didMove(toParent:)`), `sizingOptions = .intrinsicContentSize`. Cell: `UIHostingConfiguration`.</details>

**58.** `UIViewRepresentable` và Coordinator?
<details><summary>Đáp án</summary>`makeUIView` tạo 1 lần; `updateUIView` đẩy state SwiftUI → UIKit mỗi khi đổi (so sánh trước khi set); `dismantleUIView` dọn dẹp. Coordinator là class làm delegate/target (struct không làm delegate được), đẩy dữ liệu UIKit → SwiftUI qua binding.</details>

**59.** SwiftUI hay UIKit cho dự án công ty?
<details><summary>Đáp án</summary>Tuỳ deployment target, loại màn hình, team, codebase. Thực tế: app mới iOS 17+ → SwiftUI mặc định, bọc UIKit khi thiếu; app cũ → màn mới SwiftUI nhúng qua `UIHostingController`, giữ navigation UIKit, không rewrite hàng loạt.</details>

---

## 📄 Viết CV — iOS Developer ~2 năm kinh nghiệm

### §CV 1. CV để làm gì (và không để làm gì)

- Recruiter đọc CV trung bình **dưới 30 giây** cho vòng lọc đầu. Mục tiêu duy nhất của CV: **được gọi phỏng vấn**.
- Trong phỏng vấn, **mỗi dòng trên CV là một câu hỏi tiềm năng**. Viết "Combine" thì phải giải thích được `sink` và retain cycle; viết "MVVM-C" thì phải vẽ được Coordinator. Với người từng dựa nhiều vào AI như bạn, đây là quy tắc sống còn: **chỉ ghi những gì bạn nói được 2 phút không vấp**.
- CV ngắn mà sâu > CV dài mà rỗng.

### §CV 2. Cấu trúc chuẩn (1 trang)

| Thứ tự | Mục | Nội dung | Ghi chú |
|---|---|---|---|
| 1 | **Header** | Tên, chức danh "iOS Developer", email, SĐT, thành phố, link GitHub, LinkedIn | Không cần địa chỉ nhà cụ thể |
| 2 | **Summary** | 2–3 dòng: bạn là ai, mạnh gì, đang tìm gì | Không viết sáo rỗng "hard-working, team player" |
| 3 | **Skills** | Nhóm theo loại, chỉ ghi thứ bạn thật sự dùng | Không thanh % / sao kỹ năng |
| 4 | **Experience** | Mỗi công ty: chức danh, thời gian (MM/YYYY), 3–5 bullet | Phần quan trọng nhất |
| 5 | **Projects** | RepoScope (+ app cá nhân nếu có) | Bù cho kinh nghiệm còn ngắn |
| 6 | **Education** | Trường, ngành, năm tốt nghiệp | GPA chỉ ghi nếu tốt và mới ra trường |
| 7 | (Tuỳ chọn) Languages / Certificates | Tiếng Anh (IELTS/TOEIC nếu có, hoặc mô tả "đọc tài liệu, giao tiếp công việc") | |

**Định dạng:** 1 cột, font rõ (Inter, Helvetica, Calibri 10–11pt), heading chuẩn ("Experience", "Skills") để hệ thống **ATS** (phần mềm lọc CV) đọc được. Tránh bảng phức tạp, icon thay chữ, 2 cột dày đặc, ảnh nền. Xuất **PDF**, tên file `NguyenVanA_iOS_Developer_CV.pdf`.

### §CV 3. Gom nguyên liệu trước khi viết

**3.1. Brain dump (T5)** — mở file trống, liệt kê **mọi thứ** đã làm ở công ty, không cần hay:
- Các app/feature đã làm (tên màn hình, luồng: login, thanh toán, chat, map, push...)
- Công nghệ đã chạm: UIKit/SwiftUI, Alamofire/URLSession, Realm/CoreData, Firebase, RxSwift/Combine, SPM/CocoaPods, Fastlane, CI...
- Vấn đề đã sửa: crash, leak, màn chậm, bug khó
- Việc "ngoài code": review code, viết tài liệu, hướng dẫn intern, làm việc với designer/BE/QA, release lên App Store
- **Con số** có thể kiểm chứng: số người dùng/lượt tải, rating App Store, crash-free rate (Crashlytics), số màn hình, thời gian build, % test coverage, thời gian tải màn hình trước/sau

**3.2. Lọc**: với mỗi mục, hỏi *"mình có giải thích được vì sao làm thế không?"*. Không → hoặc học lại cho chắc (tuần 15–17 còn kịp), hoặc bỏ khỏi CV.

### §CV 4. Viết bullet: Action + Tech + Result

**Công thức:** **Động từ mạnh** (đã làm gì) + **công nghệ/cách làm** (bằng gì) + **kết quả đo được hoặc tác động** (để làm gì, được gì).

Động từ gợi ý: *Built, Implemented, Developed, Refactored, Migrated, Reduced, Improved, Optimized, Integrated, Designed, Automated, Fixed, Led, Collaborated, Mentored.* Tránh: *Responsible for, Worked on, Helped with, Participated in.*

**Bad → Good:**

| ❌ Bad | ✅ Good |
|---|---|
| Responsible for developing features for the app. | Built the in-app checkout flow (cart → payment → order tracking) in UIKit with MVVM, used by ~20K monthly active users. |
| Fixed bugs. | Reduced crash rate from 1.8% to 0.4% by fixing retain cycles and force-unwrap crashes found via Crashlytics and Xcode Memory Graph. |
| Used SwiftUI. | Developed 6 new settings and profile screens in SwiftUI and embedded them into the existing UIKit app via UIHostingController. |
| Worked with API. | Implemented a reusable networking layer with URLSession and async/await, with protocol-based mocking that enabled unit tests for 4 core services. |
| Improved performance. | Improved feed scrolling to a stable 60 fps by moving image decoding off the main thread and adding image downsampling and prefetching. |
| Participated in releases. | Prepared and shipped 15+ App Store releases; automated TestFlight uploads with Fastlane, cutting release time from ~1 hour to ~15 minutes. |

**Về con số — trung thực tuyệt đối:**
- Ưu tiên số **thật** từ Crashlytics, App Store Connect, Jira, Git.
- Không có số chính xác thì dùng ước lượng **bảo vệ được** ("~", "15+") hoặc mô tả tác động định tính ("used by all store staff across 30 branches").
- **Không bịa số.** Người phỏng vấn tốt sẽ hỏi "đo bằng gì? trước đó là bao nhiêu?" — bịa là lộ ngay và mất điểm uy tín còn nặng hơn không có số.

**Bài tập (T6):** viết lại **ít nhất 6 bullet** từ brain dump. Với mỗi bullet, tự trả lời 3 câu: *Vì sao làm? Làm thế nào (chi tiết kỹ thuật)? Nếu làm lại sẽ làm khác gì?* — đó chính là 3 câu người phỏng vấn sẽ hỏi.

### §CV 5. Trình bày side project RepoScope

Side project là vũ khí chính để bù cho "kinh nghiệm chưa đủ 2 năm": nó chứng minh bạn **tự tay** viết được code chất lượng, và người phỏng vấn có thể mở code ra xem.

**Cách viết:**
```
RepoScope — GitHub repository explorer (iOS 17+)          github.com/<you>/RepoScope
Swift, UIKit + SwiftUI, async/await, MVVM + Coordinator, XCTest
• Built search, repository list and detail screens; list in UIKit with diffable data source,
  detail and settings in SwiftUI embedded via UIHostingController.
• Designed a protocol-based networking layer (URLSession, async/await) with pagination,
  error handling and request cancellation for search-as-you-type.
• Added offline cache and image loading with downsampling; unit-tested view models and
  networking with mocked services (XX% coverage).
```
(Điều chỉnh theo những gì bạn **thật sự** làm đến tuần 17. Thay XX bằng số thật.)

**README của RepoScope (làm T7)** phải có:
1. 1 câu mô tả + **3–4 screenshot** (hoặc GIF) — ảnh chụp simulator, đặt trong thư mục `docs/`.
2. Tính năng chính (bullet).
3. **Kiến trúc**: sơ đồ đơn giản (View → ViewModel → Service → API/Cache) và *vì sao chọn* (2–3 câu).
4. Tech stack, yêu cầu (Xcode/iOS version), cách chạy.
5. Những gì sẽ cải thiện tiếp (cho thấy bạn tự nhận thức được trade-off).

### §CV 6. Kinh nghiệm ~1,5–2 năm: trình bày trung thực

Đến tháng 3/2027 bạn có khoảng **gần 2 năm** kinh nghiệm. JD ghi "2+ years" **không** có nghĩa bạn bị loại — với ứng viên sát mức, người tuyển nhìn vào **độ sâu**.

- ✅ Ghi thời gian chính xác theo tháng: `07/2025 – Present`. Recruiter tự tính được.
- ✅ Summary viết **"iOS Developer with nearly 2 years of experience..."** hoặc không nêu số năm, để phần Experience tự nói.
- ❌ Không viết "2+ years" / "3 years" khi chưa đủ. Không kéo dài thời gian thực tập thành full-time. Không gộp thời gian học thành kinh nghiệm. Background check hoặc một câu hỏi "dự án đó bạn làm từ tháng mấy?" là đủ lộ.
- ✅ **Để độ sâu lên tiếng**: một bullet mô tả rõ việc sửa retain cycle bằng Memory Graph có giá trị hơn 3 năm kinh nghiệm ghi chung chung. Trong phỏng vấn, câu trả lời tốt về ARC, concurrency, SwiftUI identity (những gì bạn ôn 12 tuần qua) chính là bằng chứng.
- Nếu bị hỏi *"Bạn chưa đủ 2 năm?"*: trả lời bình tĩnh — *"Đúng, em có khoảng [1 năm 10 tháng — thay bằng số thật] làm iOS full-time. Trong thời gian đó em [ví dụ cụ thể]. Ngoài giờ em làm thêm RepoScope để luyện [kiến trúc/testing]. Em tin mức độ hiểu của em phù hợp với yêu cầu, và rất sẵn lòng làm bài test kỹ thuật."*

### §CV 7. Mục Skills, GitHub, LinkedIn

**Skills — nhóm rõ ràng, không liệt kê lan man:**
```
Languages:      Swift, Objective-C (reading/maintenance)
iOS:            UIKit, SwiftUI, Auto Layout, Core Data, URLSession, Push Notifications
Concurrency:    Swift Concurrency (async/await, actors), GCD, Combine
Architecture:   MVVM, Coordinator, Dependency Injection, Protocol-oriented design
Testing/Tools:  XCTest, Xcode Instruments, Git, SPM, CocoaPods, Fastlane, Firebase Crashlytics
```
- Chỉ ghi thứ bạn đã dùng thật + giải thích được. "Objective-C" chỉ ghi nếu đọc/sửa được.
- Không ghi kỹ năng chung chung (MS Office, "teamwork") vào mục này.
- Không dùng thanh tiến độ / "Swift 90%" — vô nghĩa và bị ATS bỏ qua.

**GitHub (T7):**
- Pin 2–4 repo tốt nhất (RepoScope đầu tiên). Ẩn/archive repo tutorial copy, repo trống.
- Mỗi repo pin có README tử tế. Tạo profile README ngắn (1–3 dòng giới thiệu).
- Commit message có nghĩa (`Add pagination to search` thay vì `update`, `fix`).
- **Kiểm tra không lộ secret**: GitHub token, API key trong code/lịch sử commit. Dùng `.xcconfig` + `.gitignore`. Nếu đã lỡ commit → thu hồi (revoke) key ngay, rồi mới dọn lịch sử.

**LinkedIn:**
- Headline: `iOS Developer | Swift, SwiftUI, UIKit` (không phải "Student at..." hay để trống).
- Ảnh đại diện rõ mặt, About 3–4 câu (có thể dùng lại Summary).
- Experience khớp **hoàn toàn** với CV (chức danh, thời gian).
- Bật "Open to work" ở chế độ **chỉ recruiter thấy** nếu chưa muốn công ty hiện tại biết.
- Viết bằng tiếng Anh.

### §CV 8. CV tiếng Việt hay tiếng Anh?

| Nơi nộp | Ngôn ngữ nên dùng |
|---|---|
| Công ty nước ngoài, product company, startup có team quốc tế | **Tiếng Anh** (bắt buộc) |
| Công ty Việt Nam lớn (tập đoàn, ngân hàng, fintech) | Tiếng Anh thường được ưu tiên; JD tiếng Việt thì có thể gửi tiếng Việt |
| Outsourcing (làm khách Nhật/Mỹ/EU) | Tiếng Anh (khách có thể xem CV) |
| JD hoàn toàn tiếng Việt, công ty nhỏ trong nước | Tiếng Việt hoặc tiếng Anh đều được |

**Khuyến nghị:** làm **bản tiếng Anh làm bản chính** (tuần này), bản tiếng Việt dịch lại sau khi bản Anh ổn định (tuần 16–17). Thuật ngữ kỹ thuật giữ tiếng Anh ở cả hai bản. CV tiếng Anh không cần ảnh, ngày sinh, tình trạng hôn nhân. CV tiếng Việt có thể có ảnh (phổ biến ở VN) nhưng không bắt buộc.

### §CV 9. Template CV đầy đủ (Markdown → PDF)

> Copy vào file `cv.md`, điền thông tin thật, xuất PDF (Typora/VS Code Markdown PDF/Google Docs). Thay mọi `[...]`. Xoá bullet nào bạn không bảo vệ được.

```markdown
# NGUYEN VAN A
**iOS Developer**
Ho Chi Minh City · [email@example.com] · [+84 9xx xxx xxx]
github.com/[username] · linkedin.com/in/[username]

---

## SUMMARY
iOS Developer with nearly 2 years of experience building and shipping [e-commerce / fintech / ...]
apps in Swift. Comfortable with UIKit and SwiftUI, Swift Concurrency, and MVVM. Focused on
app stability and clean, testable code. Looking for a product team where I can grow ownership
of features end to end.

## SKILLS
- **Languages:** Swift[, Objective-C (maintenance)]
- **iOS:** UIKit, SwiftUI, Auto Layout, [Core Data / Realm], URLSession, Push Notifications
- **Concurrency:** async/await, actors, GCD[, Combine]
- **Architecture:** MVVM, Coordinator, Dependency Injection
- **Testing & Tools:** XCTest, Instruments, Git, SPM[, CocoaPods, Fastlane, Firebase Crashlytics]

## EXPERIENCE

**iOS Developer** — [Company Name], [City]                                  [MM/YYYY] – Present
*[1 line about the product: e.g. "Food delivery app with ~50K MAU on the App Store"]*
- [Built ... using ... , resulting in / used by ...]
- [Reduced / Improved ... by ... (measured with ...)]
- [Migrated / Refactored ... from ... to ..., which ...]
- [Integrated ... (payment / maps / push) ...]
- [Collaborated with designers and backend to ...; shipped NN App Store releases]

**[Intern / Junior iOS Developer]** — [Company Name]                       [MM/YYYY] – [MM/YYYY]
- [Only if real and relevant. 1–2 bullets.]

## PROJECTS

**RepoScope** — GitHub repository explorer (iOS 17+) · github.com/[username]/RepoScope
*Swift, UIKit + SwiftUI, async/await, MVVM + Coordinator, XCTest*
- Built search, list and detail screens; UIKit list with diffable data source, SwiftUI detail
  embedded via UIHostingController.
- Designed a protocol-based networking layer with pagination, error handling and cancellation
  for search-as-you-type.
- Added offline cache and unit tests for view models and services with mocked dependencies
  ([XX]% coverage).

## EDUCATION
**[B.Sc. / Engineer] in [Major]** — [University Name]                                [YYYY]

## LANGUAGES
Vietnamese (native) · English ([IELTS x.x / TOEIC xxx] or "professional working proficiency")
```

**Checklist trước khi xuất PDF:**
- [ ] Đúng 1 trang, không lỗi chính tả (dán vào trình kiểm tra chính tả tiếng Anh)
- [ ] Mỗi bullet bắt đầu bằng động từ, thì quá khứ cho việc đã xong (`Built`), hiện tại cho việc đang làm
- [ ] Có ít nhất 3 con số **thật**
- [ ] Không có từ nào bạn không giải thích được trong 2 phút
- [ ] Thời gian làm việc ghi theo tháng, khớp LinkedIn
- [ ] Link GitHub/LinkedIn bấm được trong PDF
- [ ] Tên file chuẩn, không phải `CV final final (2).pdf`

---

## 🎯 M4 Self-test (CN 14/02) — "CV bản nháp xong; ôn xong flashcard Giai đoạn 1–3"

**Phần A — CV (đạt khi tick đủ 5/5)**
- [ ] CV tiếng Anh 1 trang, xuất PDF, đã qua checklist §CV 9
- [ ] ≥ 6 bullet theo công thức Action + Tech + Result, ≥ 3 con số thật
- [ ] Mục Projects có RepoScope; README RepoScope có screenshot và mô tả kiến trúc
- [ ] Với **mỗi** bullet, nói được 1 phút "vì sao – làm thế nào – học được gì" (ghi âm 3 bullet bất kỳ)
- [ ] Một người khác đọc CV 30 giây và nói lại được: bạn làm iOS, mạnh gì, đã làm sản phẩm gì

**Phần B — Flashcard tổng (30 phút, không mở đáp án trước)**
Chọn ngẫu nhiên **30 thẻ**: 15 thẻ từ tuần 13 (thẻ 1–92) + 15 thẻ từ tuần này (thẻ 1–59). Cách chọn: mỗi lần lấy số ngẫu nhiên (`swift -e 'print(Int.random(in: 1...92))'` hoặc app random bất kỳ).
- Đạt: **≥ 24/30** thẻ trả lời đúng ý chính.
- Thẻ sai → ghi vào file `weak-topics.md`, ôn lại khi rảnh ở tuần 15–16.

**Kết quả:** đạt cả A và B → **qua M4** ✅. Chưa đạt phần A → dành thêm 1–2 buổi tối tuần 15 cho CV (CV cần sẵn sàng **trước tuần 17**). Chưa đạt phần B → không lùi lịch, nhưng ôn thẻ sai 10 phút mỗi tối tuần 15–16.

---

## ✅ Checklist cuối tuần

- [ ] Đi hết flashcard tuần 9–12, đi lại thẻ sai lần 2
- [ ] Ôn lại các thẻ còn sai của tuần 13
- [ ] Brain dump kinh nghiệm xong (`cv-braindump.md`)
- [ ] CV tiếng Anh bản nháp v0 (PDF)
- [ ] GitHub: RepoScope được pin, README có screenshot + sơ đồ kiến trúc, không lộ token/API key
- [ ] LinkedIn: headline, About, Experience khớp CV
- [ ] Làm lại 3 bài DSA cũ
- [ ] **M4 self-test** đạt (phần A + phần B)
- [ ] Viết 5 dòng: sau 14 tuần, phần nào tự tin nhất, phần nào yếu nhất → điều chỉnh trọng tâm Giai đoạn 4

## 📖 Tài liệu đọc thêm

- [Google — "XYZ formula" cho bullet CV (Accomplished [X] as measured by [Y], by doing [Z])](https://www.inc.com/bill-murphy-jr/google-recruiters-say-these-5-resume-tips-including-x-y-z-formula-will-improve-your-odds-of-getting-hired-at-google.html)
- [The Tech Resume Inside Out — Gergely Orosz](https://thetechresume.com/) (sách ngắn, rất thực tế)
- Xem CV mẫu iOS trên GitHub: tìm "iOS developer resume" — chỉ tham khảo **cấu trúc**, không copy nội dung
- Tuần 18 sẽ luyện **kể chuyện dự án theo STAR** dựa trên chính các bullet trong CV này
