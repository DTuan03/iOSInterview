# Tuần 19 — Mock cuối · Phỏng vấn thật · Deal lương · Checklist ngày phỏng vấn (15/03 – 21/03/2027)

> **Giai đoạn 5 · Về đích** · Thời lượng: ~12 giờ (nhẹ hơn — giữ sức cho phỏng vấn thật)
> Tuần này **không học kiến thức mới**. Việc của bạn: ôn nhanh bằng 50 câu hỏi, làm 2 buổi mock cuối, đi phỏng vấn thật, và biết cách **nhận offer đúng cách**. Ngủ đủ quan trọng hơn đọc thêm một bài blog.

## 🎯 Mục tiêu tuần

Cuối tuần, bạn phải:
- [ ] Trả lời được ≥ 40/50 câu rapid-fire trong ≤ 30 giây mỗi câu
- [ ] Làm xong mock #3 (AI, kỹ thuật) và mock #4 (người thật, full loop) — tổng ≥ 4 mock
- [ ] Đã đi ≥ 2 buổi phỏng vấn thật (hoặc đã có lịch)
- [ ] Biết quy trình live coding: làm rõ → ví dụ → brute force → tối ưu → code → test
- [ ] Có sẵn 5 câu hỏi để hỏi lại nhà tuyển dụng
- [ ] Có "con số lương" của riêng mình dựa trên dữ liệu, biết cách trả lời câu "mong muốn lương bao nhiêu?"
- [ ] Biết khi nào và cách nào nộp đơn nghỉ việc (chỉ sau khi có offer bằng văn bản)
- [ ] DSA tuần 19 (xem [dsa-plan](../tracks/dsa-plan.md)) · Side project: Milestone 19 — đóng băng (xem [side-project](../tracks/side-project.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 15/03** | 1,5h | §1 Rapid-fire câu 1–25 (nói to, bấm 30''/câu). Đánh dấu câu ấp úng. DSA: 1 bài mixed bấm giờ | Danh sách câu yếu |
| **T3 16/03** | 1,5h | Rapid-fire 26–50. **Mock #3** với AI (prompt 7.2b tuần 18) | Ghi âm + bảng chấm |
| **T4 17/03** | 1,5h | §2 Live coding + §3 Take-home. Ôn lại câu yếu từ T2–T3. §6 Lương: tra cứu dữ liệu, tính con số của bạn | Con số lương (khoảng) |
| **T5 18/03** | 2h | **Mock #4** với người thật — full loop 75'. §4–5 chuẩn bị câu hỏi ngược lại | Feedback mock #4 |
| **T6 19/03** | 1h | Nhẹ: đọc lại 3 story STAR, "bộ cấp cứu" DSA. §8 checklist trước ngày phỏng vấn | Đồ đạc sẵn sàng |
| **T7 20/03** | 2h | Side project: **Milestone 19** — đóng băng, tag `v1.0`. Phỏng vấn thật nếu có lịch | Repo `v1.0` |
| **CN 21/03** | 2h | Tổng kết 19 tuần. §9 Kế hoạch sau lộ trình. Tick checklist | Kế hoạch 3 tháng tiếp theo |

> Nếu tuần này có lịch phỏng vấn thật: **ưu tiên phỏng vấn thật**, đẩy mock xuống. Một buổi phỏng vấn thật = 2 buổi mock.

---

## 📚 Kiến thức cốt lõi

### §1. 50 câu hỏi hay gặp nhất — rapid-fire

> **Cách dùng:** đọc câu hỏi → **nói to** câu trả lời trong ≤ 30 giây → mở đáp án. Đánh dấu ❌ câu sai/ấp úng, quay lại tuần tương ứng ôn 15 phút. Đáp án dưới đây là bản **ngắn** — khi phỏng vấn thật, nói thêm 1 ví dụ.

#### Swift (tuần 1–4)

**1. Optional là gì?**
<details><summary>Đáp án</summary>`enum Optional<Wrapped> { case none, some(Wrapped) }`. Biểu diễn "có thể không có giá trị", compiler buộc xử lý nil. Unwrap: `if let`, `guard let`, `??`, `?.`, `!`, `switch`.</details>

**2. `struct` vs `class`?**
<details><summary>Đáp án</summary>Struct: value type, copy khi gán, không kế thừa, memberwise init, thường nằm trên stack/inline, thread-safe hơn. Class: reference type, chia sẻ instance, kế thừa, `deinit`, ARC, identity (`===`). Mặc định chọn struct; dùng class khi cần identity, kế thừa, interop ObjC, hoặc trạng thái chia sẻ.</details>

**3. Copy-on-write là gì?**
<details><summary>Đáp án</summary>Kỹ thuật cho collection (Array, Dictionary, Set, String): copy chỉ chia sẻ buffer, chỉ thật sự copy khi một bên bị sửa (kiểm tra `isKnownUniquelyReferenced`). Struct tự viết **không tự có** COW — phải tự cài nếu chứa buffer lớn.</details>

**4. `let` với class có bất biến không?**
<details><summary>Đáp án</summary>Không. `let` chỉ khoá tham chiếu; thuộc tính `var` của object vẫn sửa được. Với struct, `let` khoá toàn bộ giá trị.</details>

**5. Closure capture biến thế nào? Capture list là gì?**
<details><summary>Đáp án</summary>Closure capture **tham chiếu** tới biến (thấy thay đổi sau đó). Capture list `[x]` chụp **giá trị** tại thời điểm tạo; `[weak self]`/`[unowned self]` đổi kiểu tham chiếu để tránh retain cycle.</details>

**6. `@escaping` là gì?**
<details><summary>Đáp án</summary>Closure được giữ lại và gọi **sau khi** hàm trả về (completion handler, lưu vào property). Mặc định closure tham số là non-escaping → compiler tối ưu được và không cần `self.` tường minh.</details>

**7. `map`, `flatMap`, `compactMap` khác nhau?**
<details><summary>Đáp án</summary>`map` biến đổi từng phần tử. `compactMap` biến đổi và bỏ `nil`. `flatMap` biến đổi thành sequence rồi làm phẳng (`[[a]] → [a]`); trên Optional thì làm phẳng `T??` thành `T?`.</details>

**8. Protocol-oriented programming? Protocol extension dùng làm gì?**
<details><summary>Đáp án</summary>Thiết kế dựa trên protocol + extension thay vì kế thừa class: dùng được cho struct/enum, conform nhiều protocol, default implementation trong extension. Bẫy: method chỉ khai báo trong extension (không có trong requirement) dùng **static dispatch** → không bị "override" khi gọi qua kiểu protocol.</details>

**9. `some` vs `any`?**
<details><summary>Đáp án</summary>`some P` (opaque type): một kiểu cụ thể **cố định** mà compiler biết, người gọi không biết; static dispatch, tối ưu tốt (`some View`). `any P` (existential): hộp chứa **bất kỳ** kiểu nào conform P, đổi được lúc runtime, có chi phí (dynamic dispatch, có thể cấp phát heap).</details>

**10. `associatedtype` là gì?**
<details><summary>Đáp án</summary>Placeholder kiểu trong protocol, kiểu conform sẽ quyết định (vd `Collection.Element`). Protocol có associatedtype trước Swift 5.7 không dùng trực tiếp làm kiểu được; nay dùng `any P` hoặc `some P` / generic `<T: P>`.</details>

**11. Error handling trong Swift: `throws`, `Result`, `try?`, `try!`?**
<details><summary>Đáp án</summary>`throws` + `do/catch` cho lỗi đồng bộ hoặc `async throws`. `Result<Success, Failure>` khi cần lưu/truyền lỗi như giá trị (completion handler). `try?` biến lỗi thành nil (mất thông tin lỗi); `try!` crash nếu lỗi. Swift 6 có typed throws `throws(MyError)`.</details>

**12. Access control: `private`, `fileprivate`, `internal`, `public`, `open`?**
<details><summary>Đáp án</summary>`private`: trong scope khai báo (+ extension cùng file). `fileprivate`: cả file. `internal` (mặc định): cả module. `public`: dùng ngoài module, không subclass/override ngoài module. `open`: như public + cho subclass/override ngoài module.</details>

#### Bộ nhớ (tuần 5)

**13. ARC hoạt động thế nào?**
<details><summary>Đáp án</summary>Compiler chèn retain/release lúc compile; mỗi instance class có strong reference count, về 0 thì `deinit` và giải phóng. Không phải garbage collector — không có chu kỳ quét, nên **không tự phá retain cycle**.</details>

**14. Retain cycle là gì? Ví dụ hay gặp?**
<details><summary>Đáp án</summary>Hai (hoặc nhiều) object giữ strong lẫn nhau → count không về 0 → leak. Hay gặp: closure lưu trong property dùng `self` mạnh; delegate khai báo strong; `Timer` giữ target; parent ↔ child cùng strong; Coordinator ↔ VC.</details>

**15. `weak` vs `unowned`?**
<details><summary>Đáp án</summary>Cả hai không tăng strong count. `weak` là Optional, tự về nil khi object bị huỷ — an toàn. `unowned` không Optional, truy cập sau khi object huỷ → crash; dùng khi chắc chắn đối tượng sống lâu hơn hoặc bằng (vd closure của chính object, quan hệ customer–card).</details>

**16. Tìm memory leak bằng cách nào?**
<details><summary>Đáp án</summary>Đặt `deinit { print }`; Xcode **Debug Memory Graph** (bật Malloc Stack Logging để xem nơi cấp phát); Instruments **Leaks** + **Allocations** (mark generation, lặp thao tác, xem bộ nhớ có quay về). Test: `addTeardownBlock { [weak sut] in XCTAssertNil(sut) }`.</details>

**17. Có cần `[weak self]` trong mọi closure không?**
<details><summary>Đáp án</summary>Không. Chỉ khi closure được **giữ lại** bởi object mà self cũng (gián tiếp) giữ → cycle. `UIView.animate`, `DispatchQueue.main.async`, closure non-escaping không gây cycle (chỉ kéo dài vòng đời tạm thời). Với `Task {}` trong VC: không phải cycle nhưng giữ self tới khi task xong → cân nhắc huỷ task.</details>

#### Concurrency (tuần 6–8)

**18. Serial vs concurrent queue? sync vs async?**
<details><summary>Đáp án</summary>Serial: chạy lần lượt từng task; concurrent: nhiều task cùng lúc. `sync`: chặn thread gọi tới khi task xong; `async`: trả về ngay. `main.sync` từ main thread → **deadlock**.</details>

**19. Race condition là gì? Cách phòng?**
<details><summary>Đáp án</summary>Nhiều thread truy cập cùng dữ liệu, ít nhất 1 ghi, không đồng bộ → kết quả phụ thuộc thứ tự chạy, có thể crash. Phòng: serial queue, concurrent queue + barrier, lock (`NSLock`, `OSAllocatedUnfairLock`), **actor**. Phát hiện: Thread Sanitizer. Swift 6 bắt phần lớn lỗi lúc compile.</details>

**20. Deadlock là gì? Ví dụ?**
<details><summary>Đáp án</summary>Các task chờ nhau mãi mãi. Ví dụ: gọi `queue.sync` trên chính serial queue đang chạy (kể cả `DispatchQueue.main.sync` trên main); hai lock lấy theo thứ tự ngược nhau.</details>

**21. GCD vs OperationQueue?**
<details><summary>Đáp án</summary>GCD: API C nhẹ, closure-based. OperationQueue (dựa trên GCD): object `Operation` có **dependency**, **cancel**, giới hạn `maxConcurrentOperationCount`, KVO trạng thái. Nay với code mới, Swift Concurrency thường thay thế cả hai.</details>

**22. async/await giải quyết vấn đề gì so với completion handler?**
<details><summary>Đáp án</summary>Code tuần tự dễ đọc (hết "callback hell"), lỗi dùng `throws` thống nhất, compiler đảm bảo luôn "trả về" (completion có thể quên gọi hoặc gọi 2 lần), hỗ trợ **cancellation** có cấu trúc, tích hợp actor isolation.</details>

**23. `Task`, `async let`, `TaskGroup` khác nhau?**
<details><summary>Đáp án</summary>`async let`: số lượng tác vụ con **cố định** chạy song song, structured. `TaskGroup`: số lượng **động** (vd tải N ảnh). `Task {}`: tạo tác vụ unstructured, kế thừa actor context + priority; `Task.detached` không kế thừa. Tác vụ con structured tự huỷ khi cha huỷ.</details>

**24. Actor là gì? Actor reentrancy?**
<details><summary>Đáp án</summary>Reference type bảo vệ state bằng cách chỉ cho **một tác vụ truy cập tại một thời điểm** (isolation); gọi từ ngoài phải `await`. Reentrancy: tại mỗi `await` trong actor, actor có thể xử lý lời gọi khác → state có thể đã thay đổi sau `await`. Phải kiểm tra lại state sau await / ghi state trước khi await (ví dụ in-flight dictionary).</details>

**25. `@MainActor` là gì?**
<details><summary>Đáp án</summary>Global actor đại diện main thread. Đánh dấu class/hàm/property để đảm bảo chạy trên main — dùng cho UI, ViewModel. Gọi từ context khác phải `await`. Swift 6.2+ có chế độ "default MainActor isolation" cho target app.</details>

**26. `Sendable` là gì? Vì sao Swift 6 báo lỗi nhiều?**
<details><summary>Đáp án</summary>Đánh dấu kiểu **an toàn để chuyển qua ranh giới concurrency** (giữa các actor/task). Value type có thành phần Sendable, actor, class `final` bất biến là Sendable. Swift 6 language mode kiểm tra data race lúc compile → lỗi khi chuyển kiểu non-Sendable qua task/actor. `@unchecked Sendable` khi tự đảm bảo an toàn bằng lock (phải giải thích được).</details>

**27. Cancellation trong Swift Concurrency hoạt động thế nào?**
<details><summary>Đáp án</summary>Hợp tác (cooperative): `task.cancel()` chỉ đặt cờ; code phải kiểm tra `Task.isCancelled` / `try Task.checkCancellation()`. API hệ thống như `URLSession.data`, `Task.sleep` tự ném `CancellationError`. SwiftUI `.task` tự huỷ khi view biến mất.</details>

**28. Combine: Publisher, Subscriber, `AnyCancellable`? Combine còn dùng không?**
<details><summary>Đáp án</summary>Publisher phát giá trị theo thời gian, Subscriber nhận; operator (`map`, `debounce`, `combineLatest`…) ở giữa. `sink` trả `AnyCancellable` — phải giữ lại, huỷ khi deinit. Vẫn gặp nhiều ở codebase 2019–2023 và hợp cho luồng sự kiện UI (debounce search); code mới thường dùng async/await + AsyncSequence, `@Observable` thay `@Published`.</details>

#### UIKit (tuần 9–10)

**29. Vòng đời UIViewController?**
<details><summary>Đáp án</summary>`init` → `loadView` → `viewDidLoad` (1 lần) → `viewWillAppear` → `viewIsAppearing` (iOS 17, có trait & kích thước đúng) → `viewWillLayoutSubviews`/`viewDidLayoutSubviews` (nhiều lần) → `viewDidAppear` → `viewWillDisappear` → `viewDidDisappear` → `deinit`.</details>

**30. App lifecycle / Scene lifecycle?**
<details><summary>Đáp án</summary>Trạng thái: not running, inactive, active, background, suspended. Từ iOS 13: `AppDelegate` lo sự kiện cấp process (launch, push token), `SceneDelegate` lo từng window/scene (`sceneDidBecomeActive`, `sceneDidEnterBackground`…). SwiftUI: `@Environment(\.scenePhase)`.</details>

**31. `frame` vs `bounds`?**
<details><summary>Đáp án</summary>`frame`: vị trí + kích thước trong hệ toạ độ **của superview**. `bounds`: trong hệ toạ độ **của chính nó** (origin thường 0,0; đổi origin = cuộn nội dung như UIScrollView). Khi view bị xoay (transform), `frame` là hình chữ nhật bao, không còn đúng kích thước thật.</details>

**32. Auto Layout: intrinsic content size, hugging, compression resistance?**
<details><summary>Đáp án</summary>Intrinsic size: kích thước tự nhiên (label theo text). Hugging priority cao → không muốn **giãn** to hơn intrinsic. Compression resistance cao → không muốn bị **ép** nhỏ hơn. Dùng để quyết định view nào co/giãn khi 2 label nằm cạnh nhau.</details>

**33. Cell reuse hoạt động thế nào? Lỗi hay gặp?**
<details><summary>Đáp án</summary>Table/collection view chỉ tạo đủ cell cho màn hình + vài cái; cell cuộn ra được đưa vào hàng đợi tái sử dụng. Lỗi: ảnh tải async gán vào cell đã bị reuse cho item khác → hiển thị sai. Sửa: `prepareForReuse` huỷ task + reset, kiểm tra identifier trước khi gán.</details>

**34. Diffable data source có lợi gì?**
<details><summary>Đáp án</summary>Cập nhật bằng snapshot, hệ thống tự tính diff và animate — hết crash "invalid number of items" do `performBatchUpdates` sai. Identifier phải unique + Hashable (dùng ID, không dùng cả model nếu model thay đổi). `reconfigureItems` để cập nhật nội dung cell không reload.</details>

**35. Vì sao list cuộn giật? Cách tối ưu?**
<details><summary>Đáp án</summary>Main thread làm quá nhiều trong 16,7ms (60fps) / 8,3ms (120fps): decode ảnh lớn, tính layout phức tạp, offscreen rendering (shadow không có `shadowPath`, mask + cornerRadius), blending. Tối ưu: downsample ảnh ở background, prefetch, cache chiều cao, giảm view lồng nhau, `shadowPath`, đo bằng Instruments (Time Profiler, Animation Hitches).</details>

**36. Responder chain là gì?**
<details><summary>Đáp án</summary>Chuỗi `UIResponder` (view → superview → VC → window → application → app delegate) để chuyển sự kiện chưa được xử lý (touch, motion, action `sendAction(to: nil)`). Hit-testing tìm view nhận touch đầu tiên (`hitTest(_:with:)`, `point(inside:)`).</details>

#### SwiftUI (tuần 11–12)

**37. SwiftUI `View` là gì? Vì sao là struct?**
<details><summary>Đáp án</summary>Một **mô tả** UI (value nhẹ), không phải view thật trên màn hình. SwiftUI tạo lại struct thường xuyên, so sánh và chỉ cập nhật phần thay đổi trong render tree. Struct rẻ để tạo, không có state riêng lâu dài (state được SwiftUI lưu bên ngoài qua `@State`).</details>

**38. `@State`, `@Binding`, `@Observable`, `@Environment` dùng khi nào?**
<details><summary>Đáp án</summary>`@State`: state riêng của view (giá trị đơn giản, hoặc giữ instance `@Observable` mà view sở hữu). `@Binding`: tham chiếu 2 chiều tới state của cha. `@Observable` (iOS 17): class model, SwiftUI theo dõi **từng property được đọc**. `@Environment`: giá trị/đối tượng truyền ngầm xuống cây view.</details>

**39. `@Observable` khác `ObservableObject` + `@Published` thế nào?**
<details><summary>Đáp án</summary>`ObservableObject` báo thay đổi cả object (`objectWillChange`) → mọi view quan sát đều re-render kể cả property không dùng. `@Observable` theo dõi chi tiết property nào view thực sự đọc → ít re-render hơn; không cần `@Published`, `@StateObject`, `@ObservedObject` (dùng `@State`, `@Bindable`).</details>

**40. View identity trong SwiftUI? Vì sao `ForEach` cần `id`?**
<details><summary>Đáp án</summary>Identity xác định "đây có phải cùng một view không" → quyết định giữ state, animation. Structural identity (vị trí trong cây, `if/else` tạo 2 view khác nhau) và explicit identity (`id`). `ForEach` dùng `id` ổn định để diff; dùng index/`\.self` không ổn định → animation sai, state nhảy sang item khác. `.id(x)` đổi → view bị tạo mới, mất state.</details>

**41. `NavigationStack` khác `NavigationView`?**
<details><summary>Đáp án</summary>`NavigationStack` (iOS 16) điều hướng dựa trên dữ liệu: `path` (mảng/`NavigationPath`) + `navigationDestination(for:)` → push/pop/deep link bằng code dễ dàng. `NavigationView` đã deprecated, khó điều khiển bằng code.</details>

**42. Kết hợp UIKit và SwiftUI thế nào?**
<details><summary>Đáp án</summary>SwiftUI trong UIKit: `UIHostingController(rootView:)` (push, present, hoặc add child VC). UIKit trong SwiftUI: `UIViewRepresentable` / `UIViewControllerRepresentable` với `makeUIView`, `updateUIView`, `Coordinator` cho delegate. iOS 16+ còn có `UIHostingConfiguration` cho cell.</details>

#### Kiến trúc (tuần 15)

**43. MVC vs MVVM? Massive View Controller?**
<details><summary>Đáp án</summary>MVC của Apple: VC vừa điều khiển view vừa xử lý logic → phình to, khó test. MVVM: ViewModel giữ state + logic trình bày, không import UIKit → test được không cần UI; View chỉ bind/render. Trade-off: thêm lớp, cần cơ chế binding (closure, Combine, `@Observable`).</details>

**44. Dependency Injection là gì? Các kiểu?**
<details><summary>Đáp án</summary>Đối tượng nhận phụ thuộc từ bên ngoài thay vì tự tạo (`init(service: ServiceProtocol)`). Kiểu: constructor (ưu tiên), property, method. Lợi: test (inject mock), đổi implementation, giảm coupling. Singleton `.shared` gọi trực tiếp khắp nơi là ngược lại DI.</details>

**45. SOLID — giải thích 1–2 nguyên tắc bằng ví dụ iOS.**
<details><summary>Đáp án</summary>**S**ingle responsibility: VC không vừa gọi API vừa parse JSON vừa format ngày. **D**ependency inversion: ViewModel phụ thuộc `RepositoryServiceProtocol`, không phụ thuộc `URLSession`. (O: mở rộng bằng protocol/extension; L: subclass thay được class cha; I: protocol nhỏ, tách `DataSource` và `Delegate`.)</details>

**46. Coordinator pattern giải quyết gì?**
<details><summary>Đáp án</summary>Tách logic điều hướng khỏi VC: VC báo sự kiện (`onSelect`), coordinator quyết định push/present gì → VC tái sử dụng được, luồng dễ thay đổi, deep link tập trung. Lưu ý quản lý vòng đời child coordinator và tránh retain cycle (VC giữ coordinator phải weak).</details>

#### Networking, persistence, testing (tuần 16)

**47. Bạn tổ chức tầng networking thế nào?**
<details><summary>Đáp án</summary>`Endpoint` (path, method, query, headers) → `APIClient` generic (`send<T: Decodable>`), xử lý status code, map lỗi về `APIError`, decode → Service/Repository theo domain, trả model cho ViewModel qua protocol. Inject `URLSession` để test bằng `URLProtocol` stub. Token qua interceptor/actor, retry có backoff.</details>

**48. UserDefaults, Keychain, file, Core Data/SwiftData — lưu gì ở đâu?**
<details><summary>Đáp án</summary>UserDefaults: setting nhỏ, không nhạy cảm. Keychain: token, mật khẩu (mã hoá, có thể còn sau khi xoá app). File (`Documents`/`Caches`): ảnh, JSON lớn, cache. Core Data/SwiftData: dữ liệu có cấu trúc, quan hệ, truy vấn, offline. Không lưu token trong UserDefaults.</details>

**49. Unit test networking/ViewModel thế nào mà không gọi mạng thật?**
<details><summary>Đáp án</summary>ViewModel phụ thuộc protocol → inject mock/stub trả dữ liệu hoặc lỗi định sẵn. APIClient: `URLSessionConfiguration.protocolClasses = [URLProtocolStub.self]` để chặn request và trả response giả, kiểm tra cả request đã gửi (URL, header). Test async bằng `async` test function; test chạy nhanh, độc lập, không phụ thuộc thứ tự.</details>

#### Release & hệ thống (tuần 17)

**50. Push notification hoạt động thế nào? Quy trình đưa app lên App Store?**
<details><summary>Đáp án</summary>Push: app xin quyền → đăng ký APNs → nhận **device token** → gửi lên server → server gửi payload tới APNs (dùng key .p8/certificate) → APNs gửi tới máy. Silent push (`content-available: 1`) đánh thức app ở background nhưng không đảm bảo. Release: tăng version/build → Archive (Release config, signing với distribution certificate + provisioning profile) → upload App Store Connect → TestFlight → điền metadata, submit review → phát hành (có thể phased release).</details>

---

### §2. Vòng live coding — làm sao để "đậu" dù chưa giải xong

Người chấm live coding đánh giá **quá trình** nhiều hơn kết quả. Ứng viên giải xong nhưng im lặng 25 phút thường **thua** ứng viên giải 80% nhưng giao tiếp tốt.

**Quy trình 6 bước (45 phút):**
| Bước | Thời gian | Nói gì |
|---|---|---|
| 1. Nhắc lại đề + hỏi làm rõ | 3' | "Để em nhắc lại: input là…, output là… Input có rỗng được không? Có số âm/trùng không? Kích thước tối đa?" |
| 2. Ví dụ + edge case | 3' | Tự viết 1 ví dụ thường + 2 edge case lên màn hình |
| 3. Brute force | 3' | "Cách đơn giản nhất là…, O(n²). Em code cách này trước hay tối ưu luôn ạ?" |
| 4. Tối ưu | 5' | "Em thấy đang tìm kiếm lặp lại → dùng dictionary → O(n)". **Chốt độ phức tạp trước khi code** |
| 5. Code | 15–20' | Nói khi code: "Ở đây em dùng `guard` vì…". Tên biến rõ. Tách hàm nhỏ nếu dài |
| 6. Test | 5' | Trace bằng tay với ví dụ + edge case. Tự tìm bug trước khi interviewer chỉ |

**Mẹo:**
- **Bí?** Nói ra: *"Em đang phân vân giữa A và B vì…"*. Interviewer thường gợi ý — **nhận gợi ý một cách vui vẻ** là điểm cộng, không phải điểm trừ.
- **Không nhớ API?** *"Em không nhớ chính xác tên hàm, em sẽ giả sử có `xyz` làm việc…"* — chấp nhận được.
- **Live coding iOS** (dựng list gọi API trong 60–90'): ưu tiên **chạy được** trước (fetch → decode → hiển thị), rồi mới loading/error state, rồi kiến trúc. Giải thích những gì bạn **sẽ làm thêm nếu có thời gian** (test, cache, pagination). Luyện bằng cách dựng lại màn Search của RepoScope từ project trống trong 60'.
- **Swift trên CoderPad/Google Docs**: nhớ `Array(s)` cho String, `dict[key, default: 0]`, `popLast()`, `0..<n`, không có heap/queue sẵn (xem [dsa-plan §1](../tracks/dsa-plan.md)).
- Khi xong sớm: tự đề xuất follow-up — *"Nếu input rất lớn không vừa bộ nhớ thì…"*, *"Nếu cần thread-safe thì…"*.

**Câu chữ hay dùng (tiếng Anh nếu phỏng vấn bằng tiếng Anh):**
- "Let me restate the problem to make sure I understand…"
- "Can I assume the input fits in memory / is non-empty?"
- "The brute-force approach would be… which is O(n²). I think we can do better with a hash map."
- "Let me walk through this example to verify…"
- "One edge case I want to handle is…"

---

### §3. Take-home assignment — checklist nộp bài

Đề thường gặp: *"Làm app 2 màn: list + detail, gọi API X, có search/pagination/offline, 3–7 ngày"*. Đây chính là RepoScope thu nhỏ.

**Trước khi bắt đầu:**
- [ ] Đọc kỹ đề 2 lần, gạch chân **yêu cầu bắt buộc** vs **điểm cộng**.
- [ ] Hỏi lại những điểm mơ hồ qua email (iOS tối thiểu? UIKit hay SwiftUI? được dùng thư viện không?). Hỏi = chuyên nghiệp.
- [ ] Ước lượng thời gian, nói với nhà tuyển dụng nếu cần thêm thời gian **trước** deadline, không phải sau.

**Khi làm:**
- [ ] Commit nhỏ, message rõ ràng — người chấm **đọc lịch sử git**.
- [ ] Kiến trúc vừa đủ (MVVM + protocol + DI), không over-engineer.
- [ ] Xử lý đủ trạng thái: loading, empty, error (có retry), offline nếu yêu cầu.
- [ ] Không force unwrap bừa bãi, không `print` debug thừa, không code comment-out.
- [ ] **Unit test** cho ViewModel và tầng networking/parse (5–10 test chất lượng > 50 test rỗng).
- [ ] Hỗ trợ Dark Mode, Dynamic Type cơ bản, không vỡ layout trên iPhone SE và iPhone Pro Max.
- [ ] Không commit secret/API key; nếu cần key, hướng dẫn cấu hình trong README.
- [ ] Hạn chế thư viện bên thứ 3; nếu dùng, giải thích lý do trong README.

**README nộp bài phải có:**
- [ ] Cách chạy (Xcode version, iOS tối thiểu, bước cấu hình)
- [ ] Ảnh chụp/GIF
- [ ] Kiến trúc + lý do lựa chọn
- [ ] Những gì **đã làm** / **chưa làm** / **sẽ làm nếu có thêm thời gian**
- [ ] Giả định đã đặt ra
- [ ] Thời gian thực tế đã dùng (trung thực)

**Trước khi nộp:**
- [ ] Clone repo về thư mục mới, build từ đầu trên máy sạch — có chạy không?
- [ ] Chạy test xanh, không warning.
- [ ] Chuẩn bị **giải thích mọi dòng code** — vòng sau thường là "walk us through your code" và yêu cầu sửa trực tiếp.

> **Về AI trong take-home:** đọc kỹ đề — nhiều công ty nay ghi rõ có cho dùng AI hay không. Nếu không ghi, an toàn nhất là tự viết phần cốt lõi; nếu có dùng cho phần phụ thì ghi rõ trong README. Dù thế nào, bạn phải giải thích và sửa được mọi dòng trong buổi review.

---

### §4. Ngày trước và ngày phỏng vấn

#### Ngày trước (T-1)
- [ ] Đọc lại JD, gạch 5 từ khoá; chuẩn bị 1 ví dụ cho mỗi từ khoá.
- [ ] Tìm hiểu công ty: sản phẩm (tải app về dùng 15 phút, ghi 2 điểm thích + 1 điểm có thể cải thiện), tin tức gần đây, blog kỹ thuật, review trên ITviec.
- [ ] Tra tên interviewer trên LinkedIn (nếu có) — họ làm mảng nào.
- [ ] Đọc lại: 3 story STAR, "giới thiệu bản thân", 5 câu hỏi ngược, con số lương.
- [ ] Đọc lại "bộ cấp cứu" DSA (8 bài ở dsa-plan tuần 19) — **chỉ đọc**, không làm bài mới.
- [ ] Online: test Zoom/Meet/Teams, camera, mic, tai nghe, mạng (có 4G dự phòng), sạc laptop, đóng thông báo/Slack công ty, nền gọn, ánh sáng trước mặt.
- [ ] Offline: in 2 bản CV, xem đường đi + chỗ gửi xe, dự kiến đến sớm 15'. Quần áo gọn gàng (smart casual là đủ cho công ty IT).
- [ ] Sạc điện thoại, cài sẵn **RepoScope** lên iPhone (cài lại nếu đã quá 7 ngày với Apple ID miễn phí).
- [ ] **Ngủ trước 23h.** Không thức khuya ôn.

#### Ngày phỏng vấn (T-0)
- [ ] Ăn sáng nhẹ, uống nước. Không uống quá nhiều cà phê.
- [ ] 30' trước: đọc lại 1 trang tóm tắt (giới thiệu bản thân + 3 story + câu hỏi ngược). Không học thêm.
- [ ] Online: vào phòng sớm 5', camera ngang mắt, có giấy bút + nước bên cạnh, mở sẵn link GitHub RepoScope.
- [ ] Khi bắt đầu: chào, cảm ơn, mỉm cười. Hỏi lại tên người phỏng vấn nếu không nghe rõ.
- [ ] Trong lúc: không biết thì nói "Em chưa làm phần này, nhưng theo em hiểu thì…" — **không bịa**.
- [ ] Ghi chú tên người phỏng vấn + câu hỏi bạn trả lời chưa tốt (để ôn).
- [ ] Cuối buổi: hỏi câu hỏi ngược (§5), hỏi **bước tiếp theo và thời gian phản hồi**.

#### Sau phỏng vấn
- [ ] Trong 24h: email/tin nhắn cảm ơn ngắn gọn cho HR (2–3 câu, nhắc 1 điều thú vị từ buổi nói chuyện).
- [ ] Ghi lại **toàn bộ câu hỏi** đã gặp vào file `practice/interview-log.md` + tự chấm. Câu sai → ôn ngay.
- [ ] Cập nhật bảng theo dõi ứng tuyển.
- [ ] Quá hạn phản hồi 2–3 ngày → hỏi lại HR lịch sự một lần.

---

### §5. Câu hỏi để hỏi lại nhà tuyển dụng

Luôn hỏi **2–3 câu** — "Em không có câu hỏi" bị coi là thiếu quan tâm. Chọn theo người phỏng vấn:

**Hỏi kỹ sư / tech lead:**
1. "Team iOS hiện có bao nhiêu người, chia việc thế nào? Codebase đang dùng UIKit hay SwiftUI, tỷ lệ ra sao?"
2. "Quy trình từ lúc nhận task đến lúc release là gì — có code review, CI, test tự động không?"
3. "Thách thức kỹ thuật lớn nhất của app hiện tại là gì?"
4. "Team đã lên Swift 6 / Swift Concurrency chưa? Kế hoạch về kiến trúc sắp tới?"
5. "Một ngày làm việc điển hình của dev iOS trong team như thế nào?"

**Hỏi manager / HR:**
6. "Sau 3 tháng / 6 tháng, như thế nào thì anh/chị đánh giá em làm tốt vị trí này?"
7. "Lộ trình phát triển lên mid/senior ở đây ra sao, đánh giá bao lâu một lần?"
8. "Công ty có hỗ trợ học tập (khoá học, hội thảo, thiết bị) không?"
9. "Vì sao vị trí này đang mở — mở rộng team hay thay người?"
10. "Bước tiếp theo của quy trình là gì và khi nào em nhận được phản hồi?"

**Không nên hỏi ở vòng kỹ thuật:** lương, số ngày phép, có được làm remote mỗi ngày không (để dành cho vòng HR/offer). Không hỏi điều đã ghi rõ trên JD/website.

---

### §6. Đàm phán lương — iOS ~2 năm ở Việt Nam

> ⚠️ File này **cố ý không ghi con số cụ thể**. Mức lương thay đổi theo năm, thành phố, loại công ty (outsource / product / nước ngoài), kỹ năng tiếng Anh/Nhật. Con số trên mạng hay trong lời đồn nhanh lỗi thời. Hãy **tự nghiên cứu** theo phương pháp dưới đây, và **kiểm tra lại** trước khi đàm phán.

#### 6.1. Nghiên cứu: tìm "khoảng thị trường" của bạn
| Nguồn | Lấy gì |
|---|---|
| **Báo cáo lương IT hằng năm của ITviec, TopDev** (thường ra cuối năm/đầu năm) | Khoảng lương theo vị trí (Mobile/iOS) + số năm kinh nghiệm (1–3 năm) |
| **Tin tuyển dụng có ghi lương** trên ITviec, TopDev, TopCV, LinkedIn | Ghi 15–20 tin iOS 2–3 năm có ghi khoảng lương vào bảng → lấy trung vị |
| **Bạn bè/đồng nghiệp cũ cùng mức kinh nghiệm** | Hỏi khoảng, không hỏi con số chính xác; hỏi cả phúc lợi |
| **Recruiter / headhunter** | Họ biết khoảng thực tế của từng công ty — hỏi thẳng "khoảng ngân sách cho vị trí này là bao nhiêu?" |
| Glassdoor, nhóm Facebook ẩn danh | Tham khảo, độ tin cậy thấp hơn |

Ghi kết quả thành **3 con số** của riêng bạn (tham khảo, kiểm tra lại mỗi lần đàm phán):
- **Walk-away (tối thiểu):** dưới mức này bạn **ở lại công ty cũ** — thường = lương hiện tại + mức tăng bạn kỳ vọng nếu ở lại + bù rủi ro chuyển việc.
- **Target (mục tiêu):** khoảng giữa–trên của thị trường cho kỹ năng của bạn.
- **Anchor (mở lời):** cao hơn target khoảng 10–20%, vẫn nằm trong khoảng thị trường (để có chỗ thương lượng).

> Lưu ý thói quen tính lương ở Việt Nam: luôn hỏi rõ **gross hay net**, **tháng 13 (và thưởng khác) có cam kết trong hợp đồng không**, **lương thử việc bao nhiêu %** (luật quy định tối thiểu 85% lương chính thức).

#### 6.2. Trả lời câu "Mong muốn lương bao nhiêu?"
- **Vòng đầu (HR sàng lọc)** — cố gắng để họ nói trước:
  > "Em muốn tìm hiểu thêm về phạm vi công việc trước. Anh/chị có thể chia sẻ khoảng ngân sách cho vị trí này không ạ?"
- **Bắt buộc phải nói** → đưa **khoảng** với đáy ≈ target của bạn:
  > "Dựa trên báo cáo thị trường và kinh nghiệm của em với Swift Concurrency, SwiftUI và kiến trúc MVVM, em mong muốn trong khoảng **[A]–[B] gross**, tuỳ vào tổng phúc lợi."
- **Lương hiện tại?** Bạn có quyền không tiết lộ chính xác:
  > "Em xin phép không chia sẻ con số cụ thể, nhưng mức mong muốn của em là [khoảng], dựa trên khảo sát thị trường cho vị trí tương đương."
  Nếu đã chia sẻ thì phải **trung thực** (công ty có thể kiểm tra qua bảo hiểm xã hội).

#### 6.3. Khi nhận offer
1. **Cảm ơn, đừng nhận ngay** — *"Em cảm ơn, em rất vui. Em xin 2–3 ngày để xem xét kỹ, và nhờ anh/chị gửi offer letter qua email ạ."*
2. **Xem tổng gói (total package)**, không chỉ lương:
   | Hạng mục | Câu hỏi |
   |---|---|
   | Lương | Gross/net? Thử việc bao nhiêu %, bao lâu (tối đa 60 ngày với vị trí chuyên môn cao đẳng trở lên)? |
   | Thưởng | Tháng 13 có cam kết? Thưởng hiệu suất/dự án? |
   | Bảo hiểm | Đóng BHXH trên **lương thực tế hay lương cơ bản thấp**? Có bảo hiểm sức khoẻ tư nhân không? |
   | Tăng lương | Review mấy lần/năm, tháng nào? |
   | Thời gian | Số ngày phép, làm thứ 7?, hybrid/remote, OT có trả không? |
   | Khác | Thiết bị (MacBook), ngân sách học tập, phụ cấp ăn trưa/gửi xe |
3. **Counter-offer** (đề nghị lại) — một lần, có lý do, lịch sự:
   > "Em rất muốn gia nhập team. Dựa trên [khảo sát thị trường / offer khác / kỹ năng X phù hợp JD], em mong mức lương có thể ở [con số]. Nếu ngân sách lương khó điều chỉnh, em mong được xem xét [review sau 6 tháng thay vì 12 / thêm ngày phép / thưởng ký hợp đồng]."
   - Đưa **một con số**, không phải khoảng (khoảng → họ chọn đáy).
   - Có offer khác → nói thật, **không bịa** offer.
   - Họ từ chối tăng → bạn vẫn có thể nhận nếu ≥ walk-away. Đàm phán lịch sự **không làm mất offer** ở công ty chuyên nghiệp.
4. **Công ty cũ giữ chân (counter-offer từ công ty cũ):** cân nhắc kỹ — lý do bạn muốn đi (ngoài lương) có thay đổi không? Đã nhận offer bên mới rồi thì nên giữ lời.
5. Chỉ coi là "có offer" khi có **offer letter bằng văn bản (email)** ghi rõ: vị trí, lương gross, thời gian thử việc + % lương thử việc, ngày bắt đầu, phúc lợi chính.

---

### §7. Thời điểm nghỉ việc

1. **Chỉ nộp đơn nghỉ sau khi có offer bằng văn bản** và bạn đã **trả lời chấp nhận** (email). Lời hứa miệng "chắc chắn nhận em" chưa phải offer.
2. **Thời hạn báo trước** theo Bộ luật Lao động 2019 (Điều 35) — **tham khảo, kiểm tra lại văn bản hiện hành và hợp đồng của bạn**:
   | Loại hợp đồng | Báo trước ít nhất |
   |---|---|
   | Không xác định thời hạn | 45 ngày |
   | Xác định thời hạn 12–36 tháng | 30 ngày |
   | Xác định thời hạn dưới 12 tháng | 3 ngày làm việc |
   | Đang thử việc | Có thể nghỉ không cần báo trước (theo luật); xem thoả thuận thử việc |
   Hợp đồng/nội quy công ty có thể có điều khoản riêng (bàn giao, cam kết đào tạo, bảo mật). **Đọc hợp đồng của bạn trước khi hẹn ngày bắt đầu** với công ty mới.
3. **Ngày bắt đầu** ở công ty mới = ngày nộp đơn + thời hạn báo trước + 3–7 ngày nghỉ (khuyên nên có để nghỉ ngơi, xử lý giấy tờ). Thương lượng ngày này **trước khi** ký offer.
4. **Cách nghỉ chuyên nghiệp:**
   - Nói trực tiếp với quản lý trước, sau đó mới nộp đơn/email chính thức. Không để đồng nghiệp biết trước quản lý.
   - Lý do ngắn gọn, tích cực; không phàn nàn ở buổi exit interview.
   - Viết **tài liệu bàn giao**: task đang làm, kiến thức chỉ mình biết, tài khoản/quyền truy cập, chứng chỉ signing nếu bạn quản lý.
   - Không mang code/tài liệu công ty theo; xoá dữ liệu công ty khỏi máy cá nhân.
   - Hỏi rõ: chốt sổ BHXH, quyết định nghỉ việc, lương tháng cuối, phép năm chưa dùng.
5. Giữ quan hệ tốt — ngành iOS ở Việt Nam nhỏ, người quản lý cũ có thể là người được hỏi tham chiếu (reference check).

---

### §8. Kế hoạch sau lộ trình — tiếp tục học

Có offer hay chưa, 19 tuần này chỉ là **nền**. Để không "quên hết" lần nữa:

**Nếu chưa có offer sau tuần 19:**
- Bình thường — quy trình tuyển dụng thường mất 2–6 tuần. Tiếp tục nộp 10 nơi/tuần, mỗi buổi phỏng vấn thất bại = danh sách câu hỏi cần ôn.
- Mỗi tuần: 1 mock, 4 bài DSA (redo bài 🔁), ôn 1 tuần cũ theo câu hỏi phỏng vấn bị hỏi.
- Xin feedback sau khi bị loại: *"Anh/chị có thể chia sẻ em cần cải thiện điểm gì không ạ?"*

**3 tháng đầu ở công ty mới (hoặc tiếp tục ở công ty cũ):**
- Tháng 1: hiểu kiến trúc codebase, vẽ lại sơ đồ; hỏi nhiều; giao task nhỏ đúng hạn.
- Tháng 2–3: chủ động nhận 1 việc "khó" (bug lâu năm, tối ưu hiệu năng, viết test cho module chưa có test).

**Thói quen giữ kiến thức:**
- **Dùng AI như đồng nghiệp, không như người làm thay:** tự viết trước, nhờ AI review; mọi dòng merge phải giải thích được. Mỗi tuần viết ít nhất một phần code quan trọng không có AI.
- 2–3 bài DSA/tuần để giữ tay (dùng bảng tiến độ của dsa-plan, làm lại bài 🔁).
- Xem WWDC mỗi tháng 6 (chọn 5–6 session liên quan công việc), viết ghi chú 1 trang.
- Viết blog/ghi chú kỹ thuật (Viblo, Medium, GitHub) — mỗi tháng 1 bài về thứ bạn mới học: cách hiệu quả nhất để học sâu và xây thương hiệu cá nhân.
- Tiếp tục RepoScope hoặc dự án mới: thử modularization bằng SPM, snapshot test, Widget/Live Activity, App Intents.
- Hướng lên mid-level (1–2 năm tới): system design, mentor bạn mới, review code, sở hữu một module/tính năng từ đầu đến cuối.
- Đọc code mã nguồn mở chất lượng: Kingfisher, Nuke, Alamofire, swift-composable-architecture, swift-collections.

---

## ⚠️ Bẫy hay gặp

1. **Học kiến thức mới đêm trước phỏng vấn** → rối, ngủ ít. Chỉ ôn cái đã biết.
2. **Bịa câu trả lời** khi không biết → interviewer hỏi xoáy 1 câu là lộ. Nói "em chưa làm, nhưng em nghĩ…".
3. **Đưa con số lương trước khi nghiên cứu** → tự đặt "trần" cho mình.
4. **Nhận offer ngay tại chỗ** khi chưa xem tổng gói, chưa có văn bản.
5. **Nộp đơn nghỉ khi mới có offer miệng** → offer bị huỷ là thất nghiệp.
6. **Không đọc hợp đồng hiện tại** → hẹn ngày bắt đầu sai, vi phạm thời hạn báo trước.
7. **So sánh offer chỉ bằng lương gross** — bỏ qua BHXH đóng trên lương nào, tháng 13, review lương, OT.
8. **Live coding im lặng** — người chấm không đọc được suy nghĩ của bạn.
9. **Take-home over-engineer** (VIPER + 10 module) nhưng thiếu error state và test.
10. **"Em không có câu hỏi gì ạ"** ở cuối buổi.

---

## 🛠 Bài tập

**Bài 1 — Rapid-fire (T2–T3)**
Bấm giờ 30''/câu cho cả 50 câu §1. Ghi điểm: ✅ trôi chảy · 🟨 ấp úng · ❌ sai. Ôn lại mọi câu 🟨/❌ bằng file tuần tương ứng, làm lại ngày T5.
<details><summary>Bảng tra câu → tuần</summary>

| Câu | Tuần |
|---|---|
| 1–4 | 1–2 |
| 5–8 | 3 |
| 9–12 | 4 |
| 13–17 | 5 |
| 18–21 | 6 |
| 22–25 | 7 |
| 26–28 | 8 |
| 29–30, 36 | 9 |
| 31–35 | 10 |
| 37–39 | 11 |
| 40–42 | 12 |
| 43–46 | 15 |
| 47–49 | 16 |
| 50 | 17 |
</details>

**Bài 2 — Live coding iOS 60' (T4)**
Project trống → dựng màn danh sách repo GitHub (gọi `GET /search/repositories?q=swift`, decode, hiển thị tên + số sao, có loading và error) trong 60 phút, **nói to** suốt quá trình, quay màn hình.
<details><summary>Thứ tự ưu tiên nên theo</summary>

1. Model `Codable` tối thiểu (`id`, `name`, `stargazersCount`) → 2. Hàm `fetch` async → 3. `UITableView`/`List` hiển thị → 4. Loading + error + retry → 5. Tách ViewModel → 6. (Nếu còn giờ) ảnh avatar, pagination, test.
Mốc tự kiểm tra: phút 20 phải hiển thị được dữ liệu thật.
</details>

**Bài 3 — Con số lương (T4)**
Thu thập ≥ 15 tin tuyển dụng iOS 2–3 năm có ghi lương + báo cáo lương mới nhất của ITviec/TopDev. Tính trung vị. Viết ra 3 con số walk-away / target / anchor và 2 câu trả lời mẫu cho "mong muốn lương".

**Bài 4 — Đóng vai đàm phán (T5)**
Nhờ bạn bè (hoặc AI) đóng vai HR đưa offer thấp hơn target 15%. Thực hành: cảm ơn → xin thời gian → counter-offer có lý do → đề xuất phương án thay thế.
<details><summary>Prompt cho AI</summary>

```text
Đóng vai HR công ty phần mềm ở Việt Nam, đưa offer cho tôi (iOS developer 2 năm) với mức lương
thấp hơn mong muốn của tôi khoảng 15%. Hãy đàm phán thực tế: ban đầu từ chối tăng, viện dẫn
"ngân sách", "công bằng nội bộ". Chỉ nhượng bộ nếu tôi đưa lý do thuyết phục hoặc đề xuất phương án
thay thế hợp lý. Cuối cùng, nhận xét cách tôi đàm phán: lịch sự chưa, có lý do không, có để lộ
walk-away không, có hỏi tổng gói phúc lợi không.
```
</details>

**Bài 5 — Đọc hợp đồng (T4)**
Mở hợp đồng lao động hiện tại: loại hợp đồng, thời hạn báo trước, điều khoản bàn giao/bảo mật/cam kết đào tạo. Tính ngày bắt đầu sớm nhất nếu nộp đơn hôm nay.

**Bài 6 — Tổng kết 19 tuần (CN)**
Viết 1 trang: 5 điều bạn giờ làm được mà 19 tuần trước không làm được; 3 chủ đề còn yếu; kế hoạch 3 tháng tiếp theo (§8).

---

## 🎤 Câu hỏi phỏng vấn

> Vòng cuối/HR thường hỏi những câu dưới đây. Trả lời to trước khi mở đáp án.

**Q1 ⭐ Mong muốn mức lương của bạn là bao nhiêu?**
<details><summary>Đáp án gợi ý</summary>

Nếu được, hỏi khoảng ngân sách trước. Nếu phải nói: đưa **khoảng** có cơ sở (báo cáo thị trường + kỹ năng phù hợp JD), đáy khoảng ≈ target của bạn, nói rõ gross và "tuỳ tổng phúc lợi". Không nói con số chưa nghiên cứu.
</details>

**Q2 ⭐ Khi nào bạn có thể bắt đầu?**
<details><summary>Đáp án gợi ý</summary>

"Hợp đồng hiện tại của em cần báo trước [30/45] ngày, em cần thêm khoảng một tuần để bàn giao chu đáo, nên em có thể bắt đầu từ khoảng [ngày]." — thể hiện trách nhiệm với công ty cũ (công ty mới đánh giá cao điều này).
</details>

**Q3 Bạn đang phỏng vấn ở công ty nào khác không?**
<details><summary>Đáp án gợi ý</summary>

Trung thực, không cần nêu tên: "Em đang trong quá trình với 2–3 công ty khác, một nơi đã ở vòng cuối. Tuy nhiên bên mình là lựa chọn em ưu tiên vì…". Không bịa offer để ép giá.
</details>

**Q4 Nếu công ty hiện tại tăng lương giữ bạn lại thì sao?**
<details><summary>Đáp án gợi ý</summary>

"Lương không phải lý do chính em tìm cơ hội mới; em tìm môi trường [lý do phát triển]. Nên nếu em nhận offer bên mình, em sẽ giữ cam kết."
</details>

**Q5 ⭐ Điểm yếu lớn nhất của bạn?**
<details><summary>Đáp án gợi ý</summary>

Điểm yếu thật + đang cải thiện bằng hành động cụ thể. Ví dụ: "Trước đây em phụ thuộc AI để viết code, dẫn tới nền tảng không chắc. Mấy tháng qua em tự học lại có hệ thống, làm side project không dùng AI sinh code, và giờ em chỉ dùng AI để review/giải thích." Hoặc: "Em chưa có nhiều kinh nghiệm system design cho app lớn — em đang học qua [nguồn] và áp dụng trong RepoScope."
</details>

**Q6 Bạn sẽ làm gì trong 90 ngày đầu?**
<details><summary>Đáp án gợi ý</summary>

30 ngày: hiểu sản phẩm, codebase, quy trình; hoàn thành task nhỏ, hỏi nhiều. 60 ngày: tự nhận tính năng vừa, tham gia review code. 90 ngày: đóng góp cải tiến (test, hiệu năng, tài liệu), được team tin tưởng giao việc độc lập.
</details>

**Q7 Walk us through your code (take-home / side project).**
<details><summary>Đáp án gợi ý</summary>

Bắt đầu từ bức tranh lớn (sơ đồ: layer, luồng dữ liệu), rồi đi theo **một luồng người dùng** (gõ search → debounce → ViewModel → service → APIClient → decode → snapshot). Nói lý do các quyết định chính, chủ động nêu hạn chế và việc sẽ làm thêm. Sẵn sàng sửa code trực tiếp khi được yêu cầu.
</details>

**Q8 Bạn có câu hỏi gì cho chúng tôi không?**
<details><summary>Đáp án gợi ý</summary>

Luôn có 2–3 câu (§5), chọn theo người phỏng vấn; câu cuối hỏi bước tiếp theo và thời gian phản hồi.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Rapid-fire 50 câu: ≥ 40 câu ✅
- [ ] Mock #3 (AI) và #4 (người thật) xong — tổng ≥ 4 mock trong lộ trình (mốc M6)
- [ ] ≥ 2 buổi phỏng vấn thật đã diễn ra hoặc đã có lịch (mốc M6)
- [ ] Bài live coding iOS 60' xong, có quay màn hình
- [ ] Có 3 con số lương (walk-away/target/anchor) dựa trên dữ liệu
- [ ] Đã đọc hợp đồng hiện tại, biết thời hạn báo trước
- [ ] Danh sách câu hỏi ngược sẵn sàng
- [ ] Side project tag `v1.0`, CI xanh
- [ ] DSA tuần 19 xong; đọc lại "bộ cấp cứu"
- [ ] Viết tổng kết 19 tuần + kế hoạch 3 tháng tiếp theo

## 📖 Tài liệu đọc thêm

- Báo cáo thị trường IT/lương hằng năm của [ITviec](https://itviec.com/blog/) và [TopDev](https://topdev.vn/) — tìm bản mới nhất
- [Bộ luật Lao động 2019](https://thuvienphapluat.vn/van-ban/Lao-dong-Tien-luong/Bo-Luat-lao-dong-2019-333670.aspx) — Điều 24–27 (thử việc), Điều 35 (quyền đơn phương chấm dứt hợp đồng của người lao động). Kiểm tra văn bản sửa đổi mới nhất
- Haseeb Qureshi — *Ten Rules for Negotiating a Job Offer* (bài blog nổi tiếng về đàm phán offer)
- Gayle Laakmann McDowell — *Cracking the Coding Interview*, phần "Behavioral Questions" và "The Offer and Beyond"
- [Apple — WWDC videos](https://developer.apple.com/videos/) — sau khi có việc, xem session mới mỗi tháng 6
- Lại các file tuần: dùng mục 🎤 của từng tuần làm flashcard ôn định kỳ
