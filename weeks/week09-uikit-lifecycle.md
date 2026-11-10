# Tuần 9 — UIKit: App lifecycle · VC lifecycle · RunLoop · Responder chain · Render pipeline (04/01 – 10/01/2027)

> **Giai đoạn 3 · UI** · Thời lượng: ~14 giờ
> Bắt đầu phần UI. Tuần này **không** học cách dựng giao diện đẹp — học **UIKit chạy bên trong thế nào**: khi nào hàm nào được gọi, theo thứ tự nào, trên thread nào, vì sao Timer đứng khi scroll, touch đi đường nào để tới đúng nút, khung hình được vẽ ra màn hình ra sao. Đây là nhóm câu hỏi "phân loại" ứng viên 2 năm: người chỉ biết dùng sẽ trả lời mơ hồ, người hiểu sẽ nói được **thứ tự** và **lý do**.
> Nếu M2 (tuần 8) chưa đạt, dùng T2–T3 tuần này để bù, dồn §1–§3 sang T4.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Vẽ được sơ đồ 5 trạng thái app (not running / inactive / active / background / suspended) và nói callback nào của **SceneDelegate** chạy ở mỗi chuyển trạng thái
- [ ] Phân biệt vai trò AppDelegate vs SceneDelegate
- [ ] Kể **đúng thứ tự** lifecycle của UIViewController từ `init` tới `deinit`, kể cả `viewIsAppearing`, và thứ tự xen kẽ giữa 2 VC khi push/pop/present
- [ ] Biết đặt code vào đâu: tạo view, gọi API, bắt đầu animation, đọc kích thước, huỷ việc
- [ ] Thêm/gỡ child view controller đúng 3 bước
- [ ] Phân biệt `setNeedsLayout` vs `layoutIfNeeded` vs `setNeedsDisplay`, biết khi nào `layoutSubviews` / `draw(_:)` chạy
- [ ] Trả lời **frame vs bounds** có ví dụ transform và scroll
- [ ] Giải thích RunLoop, mode, vì sao Timer dừng khi scroll và cách sửa
- [ ] Giải thích hit-testing + responder chain, viết được code mở rộng vùng chạm
- [ ] Phân biệt UIView vs CALayer, mô tả render loop (commit → render server → GPU), nêu nguyên nhân offscreen rendering và cách tránh
- [ ] Phân biệt push vs present, các `modalPresentationStyle` và ảnh hưởng tới lifecycle
- [ ] Làm xong 6 bài DSA tuần 9 (Trees — DFS)

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 04/01** | 2h | §1 **App lifecycle** (AppDelegate, SceneDelegate, 5 trạng thái). Bài tập 1 · DSA: Maximum Depth of Binary Tree | App `LifecycleLab` in log mọi callback |
| **T3 05/01** | 2h | §2 **UIViewController lifecycle** (phần quan trọng nhất tuần). Bài tập 2, 3 · DSA: Invert Binary Tree | Bảng thứ tự push/pop/present tự ghi lại |
| **T4 06/01** | 2h | §3 Child VC / containment. §4 **UIView lifecycle**: layout & display cycle. §5 **frame vs bounds**. Bài tập 4, 5 · DSA: Same Tree | `ContainerVC` chạy được |
| **T5 07/01** | 2h | §6 **RunLoop**. §7 **Hit-testing & responder chain**, gesture recognizer. Bài tập 6 · DSA: Path Sum | Demo Timer + scroll, nút mở rộng vùng chạm |
| **T6 08/01** | 1,5h | §8 **CALayer, render loop, offscreen rendering**. §9 Navigation. Bài tập 7 · DSA: Diameter of Binary Tree | Ảnh chụp "Color Off-screen Rendered" trước/sau |
| **T7 09/01** | 3,5h | **Bài tập lớn** LifecycleLab hoàn chỉnh. **Side project: Milestone 9** (màn list UIKit viết bằng code, log lifecycle, push sang detail). DSA: Lowest Common Ancestor of a Binary Tree | Commit side project |
| **CN 10/01** | 2h | Tự trả lời 20 câu phỏng vấn. Vẽ lại tay 3 sơ đồ (app states, VC lifecycle, hit-test). Nói lại 3 câu bằng voice memo. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

> Code UIKit không chạy được bằng `swift file.swift`. Tạo một project iOS App (Storyboard → xoá `Main.storyboard`, hoặc làm theo §1) tên `LifecycleLab` và gõ code vào đó. Mọi đoạn code dưới đây đã được typecheck với iOS SDK, Swift 6 mode; các **thứ tự log** trong file này là log thật, chạy trên iPhone simulator iOS 26.

### §1. App lifecycle: AppDelegate, SceneDelegate và 5 trạng thái

**5 trạng thái của app:**

```
                 user mở app / hệ thống launch (push, background fetch)
 ┌─────────────┐ ───────────────────────────────────────────────┐
 │ Not running │                                                ▼
 └─────────────┘                                        ┌──────────────┐
        ▲                                               │   Inactive   │  ← đang ở foreground nhưng
        │ hệ thống kill khi thiếu RAM                   └──────────────┘    KHÔNG nhận event
        │ (KHÔNG có callback nào!)                        ▲          │      (đang chuyển trạng thái,
 ┌─────────────┐                                          │          ▼       có cuộc gọi, kéo Control Center)
 │  Suspended  │                                    ┌──────────────┐
 │ (đông cứng, │                                    │    Active    │  ← foreground + nhận touch
 │ không chạy  │                                    └──────────────┘
 │  code)      │                                          │ về Home / chuyển app
 └─────────────┘                                          ▼
        ▲                                         (Inactive → Background)
        │ vài giây sau, nếu không xin thêm thời gian ┌──────────────┐
        └────────────────────────────────────────────│  Background  │  ← vẫn chạy code (giới hạn
                                                     └──────────────┘    thời gian ~ vài giây)
```

**Từ iOS 13: tách AppDelegate và SceneDelegate.** Lý do: iPad có thể mở **nhiều cửa sổ** (scene) của cùng một app; mỗi scene có trạng thái foreground/background riêng.

| | `UIApplicationDelegate` (AppDelegate) | `UIWindowSceneDelegate` (SceneDelegate) |
|---|---|---|
| Phạm vi | Cả **process** (một app = một AppDelegate) | Một **cửa sổ / UI instance** |
| Việc gì | Setup SDK, push notification token, background URLSession, cấu hình scene | Tạo `UIWindow`, root VC, xử lý foreground/background của UI, deep link, state restoration |
| Callback chính | `didFinishLaunchingWithOptions`, `configurationForConnecting`, `didRegisterForRemoteNotificationsWithDeviceToken` | `willConnectTo`, `sceneDidBecomeActive`, `sceneWillResignActive`, `sceneWillEnterForeground`, `sceneDidEnterBackground`, `sceneDidDisconnect`, `openURLContexts` |

> **Quan trọng:** khi app dùng scene, các callback `applicationDidBecomeActive`, `applicationDidEnterBackground`... của AppDelegate **không được gọi nữa** (đã kiểm chứng: log của AppDelegate không xuất hiện). Code cũ đặt logic ở đó sẽ "im lặng không chạy". Apple đã thông báo app build với SDK sau iOS 26 sẽ **bắt buộc** dùng scene lifecycle (TN3187).

```swift
// Project không dùng storyboard. Info.plist: UIApplicationSceneManifest
import UIKit

@main
final class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Setup cấp PROCESS: SDK analytics, crash reporter, DI container, appearance
        return true
    }

    func application(_ application: UIApplication,
                     configurationForConnecting connectingSceneSession: UISceneSession,
                     options: UIScene.ConnectionOptions) -> UISceneConfiguration {
        let config = UISceneConfiguration(name: "Default", sessionRole: connectingSceneSession.role)
        config.delegateClass = SceneDelegate.self
        return config
    }
}

final class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        let window = UIWindow(windowScene: windowScene)
        window.rootViewController = UINavigationController(rootViewController: RepoListViewController())
        window.makeKeyAndVisible()
        self.window = window                 // PHẢI giữ strong, không thì window bị giải phóng → màn hình đen
        // Deep link lúc cold launch nằm trong connectionOptions.urlContexts
    }
    func sceneDidBecomeActive(_ scene: UIScene) { /* tiếp tục animation, timer */ }
    func sceneWillResignActive(_ scene: UIScene) { /* tạm dừng: có cuộc gọi, kéo Control Center */ }
    func sceneWillEnterForeground(_ scene: UIScene) { /* refresh dữ liệu cũ */ }
    func sceneDidEnterBackground(_ scene: UIScene) { /* lưu state, giải phóng tài nguyên lớn */ }
    func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) { /* deep link khi app đang chạy */ }
}
```

**Log thật khi cold launch rồi về Home rồi mở lại** (root là `UINavigationController(rootViewController: A)`):
```
App willFinishLaunching
App didFinishLaunching
App configurationForConnecting
Scene willConnectTo
A init
Scene makeKeyAndVisible done          ← makeKeyAndVisible KHÔNG load view ngay
Scene willEnterForeground
A loadView                            ← view được load khi window chuẩn bị hiện
A viewDidLoad
A viewWillAppear
A viewIsAppearing
A viewWillLayoutSubviews
A viewDidLayoutSubviews
A viewDidAppear
Scene didBecomeActive                 ← app thật sự nhận touch
--- (bấm Home / chuyển app)
Scene willResignActive
Scene didEnterBackground
--- (mở lại)
Scene willEnterForeground
Scene didBecomeActive
```
Chú ý:
- Về Home **không** gọi `viewWillDisappear` của VC — VC vẫn "đang hiện" trong cấu trúc UI. Muốn biết app vào background, lắng nghe `UIScene.didEnterBackgroundNotification` (hoặc `UIApplication.didEnterBackgroundNotification`).
- Kéo Control Center / có cuộc gọi đến: chỉ `willResignActive` (Active → Inactive), **không** vào background.
- **Suspended → bị kill: không có callback.** Vì vậy phải lưu dữ liệu quan trọng ở `sceneDidEnterBackground`, đừng chờ `applicationWillTerminate` (chỉ chạy khi app đang chạy ở background mà bị terminate, hiếm).
- Background chỉ có **vài giây**. Cần thêm (upload dở) → `UIApplication.shared.beginBackgroundTask` (tuần 17 học kỹ background modes).

### §2. UIViewController lifecycle — thứ tự chính xác

```
 init(nibName:bundle:) / init(coder:)      ← view CHƯA tồn tại
        │   (truy cập `view` lần đầu, hoặc VC sắp hiện)
        ▼
 loadView()                                 ← tạo root view (mặc định: load từ nib/storyboard, hoặc UIView trống)
        ▼
 viewDidLoad()                              ← 1 LẦN DUY NHẤT trong đời VC
        ▼
 ┌─► viewWillAppear(_:)                     ← mỗi lần sắp hiện
 │      ▼
 │   viewIsAppearing(_:)       (iOS 17+, back-deploy iOS 13) ← đã vào view hierarchy, trait & size ĐÚNG
 │      ▼
 │   viewWillLayoutSubviews()  ┐
 │   (view.layoutSubviews)     ├ có thể chạy NHIỀU lần, bất cứ lúc nào bounds đổi
 │   viewDidLayoutSubviews()   ┘
 │      ▼
 │   viewDidAppear(_:)                      ← animation chuyển màn hình đã xong
 │      ▼
 │   viewWillDisappear(_:)
 │      ▼
 │   viewDidDisappear(_:)
 └──────┘ (hiện lại → lặp lại từ viewWillAppear, KHÔNG gọi lại viewDidLoad)
        ▼
 deinit                                     ← khi không còn ai giữ strong reference
```

**Làm gì ở đâu:**

| Callback | Nên làm | Không nên |
|---|---|---|
| `init` | Nhận dependency (ViewModel, service, id) | Đụng vào `view` (sẽ ép load view sớm) |
| `loadView` | Gán `view = MyCustomRootView()` khi làm UI bằng code | Gọi `super.loadView()` rồi gán view (thừa); dùng khi có storyboard |
| `viewDidLoad` | Tạo subview, constraint, đăng ký cell, bind ViewModel, gọi API **lần đầu** | Đọc `view.bounds` để tính layout (chưa đúng size) |
| `viewWillAppear` | Cập nhật dữ liệu có thể đổi khi quay lại, ẩn/hiện nav bar, bắt đầu observe | Việc nặng (làm chậm animation chuyển màn) |
| `viewIsAppearing` | Việc cần **trait collection & kích thước đúng** (chọn layout theo size class, scroll tới item) | — |
| `viewDidLayoutSubviews` | Chỉnh thứ phụ thuộc frame cuối: `shadowPath`, `cornerRadius = height/2`, gradient layer frame | Thêm/sửa constraint → dễ vòng lặp layout vô hạn |
| `viewDidAppear` | Bắt đầu animation, video, analytics "screen view", hiện bàn phím, request quyền | — |
| `viewWillDisappear` | Lưu nháp, `endEditing`, dừng observe | — |
| `viewDidDisappear` | Dừng timer/video, huỷ `Task` | — |
| `deinit` | Log để kiểm tra leak | Gọi method của view (VC đang biến mất) |

```swift
import UIKit

final class RepoDetailViewController: UIViewController {
    private let repoID: Int
    private let titleLabel = UILabel()
    private var loadTask: Task<Void, Never>?

    // 1. init — chỉ gán dependency. CHƯA có view.
    init(repoID: Int) {
        self.repoID = repoID
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }

    // 2. loadView — chỉ override khi muốn tự tạo root view. KHÔNG gọi super nếu đã gán view.
    override func loadView() {
        view = UIView()
    }

    // 3. viewDidLoad — gọi MỘT lần. Tạo subview, constraint, bind ViewModel.
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        titleLabel.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(titleLabel)
        NSLayoutConstraint.activate([
            titleLabel.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            titleLabel.centerYAnchor.constraint(equalTo: view.centerYAnchor),
        ])
    }

    // 4. viewWillAppear — MỖI lần sắp hiện. Cập nhật dữ liệu có thể đã đổi, nav bar style.
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        navigationController?.setNavigationBarHidden(false, animated: animated)
    }

    // 5. viewIsAppearing (iOS 17+, back-deploy iOS 13) — đã có trait collection & kích thước đúng.
    override func viewIsAppearing(_ animated: Bool) {
        super.viewIsAppearing(animated)
        titleLabel.text = "Repo #\(repoID) — width \(Int(view.bounds.width))"
    }

    // 6. viewWillLayoutSubviews / viewDidLayoutSubviews — có thể gọi NHIỀU lần (xoay, đổi size...).
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        // frame của subview đã chốt — dùng để chỉnh layer (cornerRadius theo chiều cao, shadowPath)
    }

    // 7. viewDidAppear — đã hiện xong. Bắt đầu animation, analytics "screen view", load nặng.
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        loadTask = Task { /* await viewModel.load() */ }
    }

    // 8. viewWillDisappear / viewDidDisappear — dừng việc chỉ có ý nghĩa khi đang hiển thị.
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        view.endEditing(true)
    }
    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadTask?.cancel()
        if isMovingFromParent || isBeingDismissed {
            // VC bị pop/dismiss thật sự (không phải bị che bởi màn khác)
        }
    }

    // 9. deinit — nếu không chạy sau khi pop → có retain cycle (tuần 5)
    deinit { print("RepoDetailViewController deinit") }
}
```

**Thứ tự xen kẽ giữa 2 VC — log thật (iOS 26 simulator):**

| PUSH B (từ A) | POP B (về A) | PRESENT C `.fullScreen` | PRESENT D `.pageSheet` |
|---|---|---|---|
| B init | B willMove(toParent: nil) | C init | D init |
| B willMove(toParent:) | B viewWillDisappear | C loadView | D loadView |
| B loadView | **A viewWillAppear** | C viewDidLoad | D viewDidLoad |
| B viewDidLoad | A viewIsAppearing | **A viewWillDisappear** | D viewWillAppear |
| **A viewWillDisappear** | B viewDidDisappear | C viewWillAppear | D viewIsAppearing |
| B viewWillAppear | B didMove(toParent: nil) | C viewIsAppearing | D viewWill/DidLayoutSubviews |
| B viewIsAppearing | **A viewDidAppear** | C viewWill/DidLayoutSubviews | D viewDidAppear |
| B viewWill/DidLayoutSubviews | B deinit | C viewDidAppear | *(A không nhận gì cả!)* |
| **A viewDidDisappear** | | **A viewDidDisappear** | |
| B viewDidAppear | | | |
| B didMove(toParent:) | | | |

Ba điều người phỏng vấn hay hỏi xoáy:
1. **B `viewDidLoad` chạy TRƯỚC A `viewWillDisappear`** — VC mới phải load view xong mới bắt đầu transition.
2. **"Will" của bên này xen giữa "will" và "did" của bên kia**: A willDisappear → B willAppear → … → A didDisappear → B didAppear.
3. **Present kiểu `.pageSheet`/`.formSheet` (mặc định từ iOS 13) và `.overFullScreen`/`.overCurrentContext`: VC bên dưới KHÔNG nhận `viewWillDisappear`/`viewDidDisappear`**, và khi dismiss cũng **không** nhận `viewWillAppear`. Đây là bug kinh điển: "dismiss sheet xong màn trước không refresh" — vì code refresh đặt ở `viewWillAppear`. Cách sửa: delegate/closure callback khi sheet đóng, hoặc `UIAdaptivePresentationControllerDelegate.presentationControllerDidDismiss` (khi user vuốt đóng).

**Vì sao có `viewIsAppearing`?** Trước iOS 17, ở `viewWillAppear` view **chưa** chắc đã nằm trong window → trait collection và kích thước có thể chưa đúng; còn `viewDidAppear` thì quá muộn (user thấy giao diện "nhảy"). `viewIsAppearing` chạy sau khi view đã được thêm vào hierarchy và đã có trait/size chuẩn, nhưng **trước** khi hiện lên màn hình. Log trên cho thấy ở `viewIsAppearing`: `window != nil` và bounds là size thật (402×874 full screen; 402×812 với page sheet).

**`isMovingFromParent` / `isBeingDismissed`:** trong `viewWillDisappear/DidDisappear`, dùng để phân biệt "bị pop/dismiss thật" với "bị màn khác che lên".

### §3. Child view controller / containment

Khi một màn hình gồm nhiều phần độc lập (tab trong trang profile, player nhỏ, banner), tách mỗi phần thành một VC con thay vì nhồi vào một "Massive View Controller". Container có sẵn: `UINavigationController`, `UITabBarController`, `UIPageViewController`, `UISplitViewController`.

**Vì sao không chỉ `addSubview(child.view)`?** Vì như thế child **không** nhận lifecycle (`viewWillAppear`, trait changes, xoay màn hình), không nằm trong responder chain đúng, `child.navigationController` = nil. Phải đăng ký quan hệ cha–con:

```swift
extension UIViewController {
    func embed(_ child: UIViewController, in container: UIView) {
        addChild(child)                                   // 1. tự gọi child.willMove(toParent: self)
        child.view.translatesAutoresizingMaskIntoConstraints = false
        container.addSubview(child.view)                  // 2. thêm view
        NSLayoutConstraint.activate([
            child.view.topAnchor.constraint(equalTo: container.topAnchor),
            child.view.bottomAnchor.constraint(equalTo: container.bottomAnchor),
            child.view.leadingAnchor.constraint(equalTo: container.leadingAnchor),
            child.view.trailingAnchor.constraint(equalTo: container.trailingAnchor),
        ])
        child.didMove(toParent: self)                     // 3. BẮT BUỘC tự gọi
    }

    func unembed() {
        guard parent != nil else { return }
        willMove(toParent: nil)                           // 1. BẮT BUỘC tự gọi
        view.removeFromSuperview()                        // 2.
        removeFromParent()                                // 3. tự gọi didMove(toParent: nil)
    }
}
```
Mẹo nhớ: **thêm** thì `addChild` tự lo `willMove`, ta lo `didMove`; **gỡ** thì `removeFromParent` tự lo `didMove`, ta lo `willMove`. Khi `addSubview` vào một view đang hiện, child nhận `viewWillAppear` → … → `viewDidAppear` tự động (appearance forwarding).

### §4. UIView lifecycle: layout cycle & display cycle

UIKit **không** vẽ lại ngay khi bạn đổi thuộc tính. Nó **đánh dấu** "bẩn" rồi gom hết lại xử lý **một lần** ở cuối vòng RunLoop hiện tại (§6). Đó là lý do bạn đổi `frame` 10 lần trong một hàm thì màn hình chỉ cập nhật 1 lần.

```
  Code của bạn đổi state  →  setNeedsLayout / setNeedsDisplay / đổi constraint  (đánh dấu, rẻ)
                                        │
                     cuối vòng RunLoop (CATransaction commit)
                                        ▼
  ① Update constraints   updateConstraints()  (từ lá lên gốc)
  ② Layout               layoutSubviews()     (từ gốc xuống lá) — tính frame
  ③ Display              draw(_:)             — chỉ view override draw và bị đánh dấu
                                        ▼
  Gửi layer tree sang Render Server (§8)
```

| Hàm | Ý nghĩa | Khi nào dùng |
|---|---|---|
| `setNeedsLayout()` | "Lần tới hãy layout lại tôi" — **bất đồng bộ**, trả về ngay | Khi dữ liệu đổi làm vị trí subview đổi |
| `layoutIfNeeded()` | "Nếu đang bẩn thì layout **ngay bây giờ**" — **đồng bộ** | Trong `UIView.animate` để animate constraint; khi cần đọc frame đúng ngay |
| `layoutSubviews()` | Nơi tính frame subview. **Không gọi trực tiếp** — override nó | View custom tự đặt frame con |
| `setNeedsDisplay()` | "Lần tới hãy gọi `draw(_:)` lại" | View có `draw(_:)` custom khi dữ liệu đổi |
| `draw(_:)` | Vẽ bằng Core Graphics vào backing store | Biểu đồ, hình đặc biệt. **Không gọi trực tiếp** |
| `setNeedsUpdateConstraints()` / `updateConstraints()` | Gom việc sửa constraint | Hiếm dùng; Apple khuyên chỉ dùng khi đo đạc thấy cần |

`layoutSubviews` được gọi khi: `setNeedsLayout` + chu kỳ tiếp theo, `bounds` đổi (kể cả xoay màn hình), thêm/bớt subview, scroll `UIScrollView` (vì `bounds.origin` đổi!), constraint thay đổi.

```swift
import UIKit

final class BadgeView: UIView {
    private let label = UILabel()
    var count = 0 {
        didSet {
            label.text = "\(count)"
            setNeedsLayout()          // đánh dấu "cần layout lại" — layoutSubviews sẽ chạy ở chu kỳ sau
            setNeedsDisplay()         // đánh dấu "cần vẽ lại" — draw(_:) sẽ chạy ở chu kỳ sau
        }
    }

    override init(frame: CGRect) {
        super.init(frame: frame)
        addSubview(label)
        label.textAlignment = .center
        label.textColor = .white
        backgroundColor = .clear
    }
    required init?(coder: NSCoder) { fatalError() }

    override func layoutSubviews() {
        super.layoutSubviews()        // với Auto Layout, super áp dụng constraint ở đây
        label.frame = bounds          // dùng bounds (hệ toạ độ của CHÍNH MÌNH), không dùng frame
    }

    override func draw(_ rect: CGRect) {
        // Chỉ override khi cần vẽ custom bằng Core Graphics. Tốn CPU + bộ nhớ (backing store).
        UIColor.systemRed.setFill()
        UIBezierPath(ovalIn: bounds).fill()
    }
}

final class AnimatedVC: UIViewController {
    private let panel = UIView()
    private var heightConstraint: NSLayoutConstraint!

    override func viewDidLoad() {
        super.viewDidLoad()
        panel.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(panel)
        heightConstraint = panel.heightAnchor.constraint(equalToConstant: 100)
        NSLayoutConstraint.activate([
            panel.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            panel.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            panel.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            heightConstraint,
        ])
    }

    func expand() {
        heightConstraint.constant = 300                  // 1. đổi constraint (chưa có gì xảy ra trên màn hình)
        UIView.animate(withDuration: 0.3) {
            self.view.layoutIfNeeded()                   // 2. ép layout NGAY trong animation block → frame mới được animate
        }
    }
}
```
→ Câu hỏi kinh điển: *"Vì sao animate constraint phải gọi `layoutIfNeeded()` trong block?"* — đổi `constant` chỉ đánh dấu bẩn; nếu để chu kỳ sau tự layout thì việc đổi frame xảy ra **ngoài** animation block → nhảy cóc. Gọi `layoutIfNeeded()` **trên superview** (thường là `view` của VC) để mọi view bị ảnh hưởng được layout trong block.

**UIView "lifecycle" khác:** `willMove(toSuperview:)`, `didMoveToSuperview()`, `willMove(toWindow:)`, `didMoveToWindow()` (view lên/xuống màn hình — chỗ tốt để start/stop animation của view), `traitCollectionDidChange` (đã deprecated iOS 17 → `registerForTraitChanges`).

### §5. frame vs bounds — câu hỏi kinh điển

- **`frame`**: hình chữ nhật của view **trong hệ toạ độ của superview** (vị trí + kích thước "nhìn từ cha").
- **`bounds`**: hình chữ nhật trong **hệ toạ độ của chính nó**. `bounds.origin` thường là (0,0); `bounds.size` là kích thước "thật" của view.
- **`center`**: tâm view, trong toạ độ superview.

**Log thật** (simulator iOS 26):
```swift
let v = UIView(frame: CGRect(x: 50, y: 100, width: 200, height: 100))
// frame  = (50, 100, 200, 100)   bounds = (0, 0, 200, 100)   center = (150, 150)

v.transform = CGAffineTransform(rotationAngle: .pi / 4)     // xoay 45°
// frame  ≈ (43.9, 43.9, 212.1, 212.1)   ← hình chữ nhật NHỎ NHẤT bao quanh view đã xoay
// bounds = (0, 0, 200, 100)             ← không đổi: view vẫn "nghĩ" nó 200×100
// center = (150, 150)                   ← không đổi: xoay quanh tâm

v.transform = CGAffineTransform(scaleX: 2, y: 2)
// frame  = (-50, 50, 400, 200)          bounds = (0, 0, 200, 100)

v.transform = .identity
let child = UIView(frame: CGRect(x: 10, y: 10, width: 20, height: 20))
v.addSubview(child)
v.bounds.origin = CGPoint(x: 0, y: 30)
// v.frame = (50, 100, 200, 100)         ← view KHÔNG di chuyển
// child.frame = (10, 10, 20, 20)        ← không đổi, nhưng hiển thị ở y = 10 - 30 = -20 (bị đẩy lên trên)
```

**Điểm phải nói được khi phỏng vấn:**
1. Khi có `transform` khác identity, **`frame` không còn ý nghĩa** (Apple ghi rõ: undefined, đừng đọc/ghi). Muốn đổi kích thước hãy dùng `bounds` + `center`.
2. **Đổi `bounds.origin` = dịch chuyển nội dung bên trong**, chính là cách `UIScrollView` hoạt động: `contentOffset` **chính là** `bounds.origin`. Scroll xuống 100pt → `bounds.origin.y = 100` → mọi subview hiện lên cao hơn 100pt. Đó là lý do scroll gọi `layoutSubviews` liên tục.
3. Trong `layoutSubviews`, đặt frame con dựa vào **`bounds`** của mình, không dùng `frame` (frame là toạ độ của cha).
4. Chuyển toạ độ giữa các view: `view.convert(point, to: otherView)` / `convert(_:from:)`.

### §6. RunLoop

**RunLoop** là vòng lặp sự kiện gắn với một thread: *"có việc thì làm, hết việc thì ngủ (không tốn CPU), có sự kiện thì thức dậy"*. Main thread của mọi app iOS có một RunLoop chạy suốt đời app (`UIApplicationMain` khởi động nó). Thread phụ (`Thread`) mặc định **không** có runloop chạy.

```
     ┌──────────────────────────────────────────────────────┐
     │  1. Báo observer: sắp xử lý timer / source           │
     │  2. Xử lý source0 (event nội bộ: touch đã chuyển tới)│
     │  3. Xử lý timer đến hạn (Timer, performSelector)     │
     │  4. Xử lý block của DispatchQueue.main               │
     │  5. Báo observer: SẮP NGỦ (beforeWaiting)            │──► Core Animation commit ở đây:
     │  6. Ngủ (mach_msg) chờ: source1 (touch từ hệ thống,  │    layout + display + gửi layer tree
     │     port), timer, dispatch main                      │    sang render server (§8)
     │  7. Thức dậy → quay lại 1                            │
     └──────────────────────────────────────────────────────┘
```

**Mode:** mỗi lần chạy, RunLoop chạy ở **một mode**, và chỉ xử lý source/timer **đã đăng ký cho mode đó**.

| Mode | Khi nào |
|---|---|
| `.default` (`NSDefaultRunLoopMode`) | Bình thường |
| `.tracking` (`UITrackingRunLoopMode`) | Khi user đang **kéo/scroll** UIScrollView — UIKit chuyển sang mode này để ưu tiên mượt |
| `.common` (`NSRunLoopCommonModes`) | Không phải mode thật; là **tập hợp** mode (default + tracking). Đăng ký vào `.common` = đăng ký cho cả hai |

**Vì sao Timer dừng khi scroll?** `Timer.scheduledTimer(...)` thêm timer vào mode `.default`. Khi scroll, main runloop chạy ở `.tracking` → timer không được xử lý cho tới khi buông tay. Đã kiểm chứng trong app chạy trên simulator, giữ runloop ở `.tracking` 0,5s với 2 timer chu kỳ 50ms:
```
tracking 0.5s: default timer 0, common timer 10
```

```swift
final class CountdownVC: UIViewController {
    private var timer: Timer?
    private var remaining = 60

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        // ❌ Timer.scheduledTimer(...) thêm vào mode .default → đứng yên khi user scroll
        let t = Timer(timeInterval: 1, repeats: true) { [weak self] _ in
            MainActor.assumeIsolated { self?.tick() }    // Timer chạy trên main runloop → đang ở main thread
        }
        RunLoop.main.add(t, forMode: .common)            // ✅ .common = default + tracking
        timer = t
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        timer?.invalidate()                              // RunLoop giữ strong Timer → phải invalidate
        timer = nil
    }

    private func tick() { remaining -= 1 }
}
```
Lựa chọn khác: `CADisplayLink` (đồng bộ với tần số màn hình, cho animation thủ công — cũng phải add vào `.common`), `DispatchSourceTimer` (không phụ thuộc runloop), hoặc `Task { while !Task.isCancelled { try await Task.sleep(for: .seconds(1)) } }`.

**Liên hệ thực tế khác:**
- `perform(_:with:afterDelay:)` cũng chỉ chạy ở mode default.
- Combine `receive(on: RunLoop.main)` bị hoãn khi scroll; `DispatchQueue.main` thì không (tuần 8).
- Gọi `Timer.scheduledTimer` trên thread nền (GCD) → **không bao giờ chạy** vì thread đó không có runloop đang chạy.
- RunLoop giữ strong `Timer`, Timer (API target/selector) giữ strong `target` → VC không deinit nếu không `invalidate` (tuần 5).

### §7. Hit-testing, responder chain, gesture recognizer

Hai giai đoạn khác nhau, đừng nhầm:
1. **Hit-testing** (đi **xuống**): tìm **view nào** bị chạm → view đó thành *first responder* cho touch.
2. **Responder chain** (đi **lên**): nếu view đó không xử lý event, chuyển lên `next` responder.

**Hit-testing** — `UIWindow` gọi `hitTest(_:with:)` trên root view, đệ quy:
```
hitTest(point):
  nếu isHidden || !isUserInteractionEnabled || alpha < 0.01  → return nil
  nếu !point(inside: point)                                   → return nil   (ngoài bounds)
  duyệt subviews TỪ TRÊN XUỐNG (cuối mảng = trên cùng):
      p' = convert(point, to: subview)
      nếu let v = subview.hitTest(p') → return v
  return self
```
Hệ quả quan trọng:
- **Subview nằm ngoài bounds của cha không nhận được chạm** (dù `clipsToBounds = false` vẫn *nhìn thấy*), vì cha đã `return nil` ở bước `point(inside:)`. Đã kiểm chứng: `parent.hitTest(điểm trong nút lòi ra ngoài)` → `nil`.
- `UIImageView`, `UILabel` mặc định `isUserInteractionEnabled = false` → gắn gesture vào mà không bật thì không chạy.
- View có `alpha = 0` không nhận chạm; muốn "vô hình mà vẫn bắt chạm" dùng `alpha = 0.02` hoặc `backgroundColor = .clear`.

```swift
// 1. Mở rộng vùng chạm của nút nhỏ (icon 20x20 nhưng Apple khuyến nghị vùng chạm ≥ 44x44)
final class ExpandedButton: UIButton {
    var touchInsets = UIEdgeInsets(top: -12, left: -12, bottom: -12, right: -12)
    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        bounds.inset(by: touchInsets).contains(point)
    }
}

// 2. View phủ lên trên nhưng "trong suốt" với touch — chỉ subview của nó nhận chạm
final class PassThroughView: UIView {
    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let hit = super.hitTest(point, with: event)
        return hit === self ? nil : hit                  // chạm vào nền → trả nil → touch rơi xuống view bên dưới
    }
}

// 3. Cho phép subview nằm NGOÀI bounds của cha vẫn nhận chạm (vd. nút "x" lòi ra góc card)
final class OverflowView: UIView {
    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        if let hit = super.hitTest(point, with: event) { return hit }
        guard isUserInteractionEnabled, !isHidden, alpha > 0.01 else { return nil }
        for sub in subviews.reversed() {
            let p = sub.convert(point, from: self)
            if let hit = sub.hitTest(p, with: event) { return hit }
        }
        return nil
    }
}
```
(Đã kiểm chứng trên simulator: `ExpandedButton` 20×20 nhận điểm (-5,-5) nhưng không nhận (-15,0); chạm vào nền `PassThroughView` rơi xuống nút bên dưới.)

**Responder chain** — `UIResponder` là lớp cha của `UIView`, `UIViewController`, `UIWindow`, `UIApplication`. Mỗi responder có `next`:
```
 UIButton → superview → … → root view của VC → UIViewController
          → superview của view VC (nếu VC là child) / parent VC → … → UIWindow
          → UIWindowScene → UIApplication → AppDelegate (nếu là UIResponder)
```
Dùng cho: touch không ai xử lý (`touchesBegan` gọi `super` để chuyển tiếp), **bàn phím** (`becomeFirstResponder` / `resignFirstResponder`), menu copy/paste, `UIKeyCommand`, và **gửi action với target = nil**:
```swift
@objc protocol CartActions { func addToCart(_ sender: Any?) }
final class ProductCell: UICollectionViewCell {
    @objc func buyTapped() {
        UIApplication.shared.sendAction(#selector(CartActions.addToCart(_:)), to: nil, from: self, for: nil)
        // to: nil → UIKit đi từ first responder (hoặc `from`) ngược lên chain tới ai implement addToCart
    }
}
```

**Gesture recognizer vs touches:**
- Touch được gửi cho **gesture recognizer gắn trên view bị hit và các view cha** **trước** khi tới `touchesBegan` của view.
- Khi một recognizer *nhận diện thành công*, mặc định nó **huỷ** touch của view (`cancelsTouchesInView = true` → view nhận `touchesCancelled`). Đây là lý do gắn `UITapGestureRecognizer` lên view cha làm **cell của table không nhận `didSelectRow`** → sửa: `tap.cancelsTouchesInView = false`.
- Trạng thái: discrete (tap, swipe) `possible → recognized`; continuous (pan, pinch) `possible → began → changed → ended/cancelled`.
- Mặc định 2 recognizer **không** chạy đồng thời; muốn vậy implement `shouldRecognizeSimultaneouslyWith`. Thứ tự ưu tiên: `require(toFail:)`.

```swift
final class PhotoViewController: UIViewController, UIGestureRecognizerDelegate {
    private let imageView = UIImageView()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(imageView)
        imageView.isUserInteractionEnabled = true        // UIImageView/UILabel mặc định = false!

        let doubleTap = UITapGestureRecognizer(target: self, action: #selector(handleDoubleTap))
        doubleTap.numberOfTapsRequired = 2
        let singleTap = UITapGestureRecognizer(target: self, action: #selector(handleSingleTap))
        singleTap.require(toFail: doubleTap)             // single chỉ chạy khi chắc chắn không phải double
        let pinch = UIPinchGestureRecognizer(target: self, action: #selector(handlePinch(_:)))
        pinch.delegate = self
        let rotate = UIRotationGestureRecognizer(target: self, action: #selector(handleRotate(_:)))
        rotate.delegate = self
        [doubleTap, singleTap, pinch, rotate].forEach(imageView.addGestureRecognizer)
    }

    @objc private func handleDoubleTap() {}
    @objc private func handleSingleTap() {}
    @objc private func handlePinch(_ g: UIPinchGestureRecognizer) {
        guard let v = g.view else { return }
        v.transform = v.transform.scaledBy(x: g.scale, y: g.scale)
        g.scale = 1
    }
    @objc private func handleRotate(_ g: UIRotationGestureRecognizer) {
        guard let v = g.view else { return }
        v.transform = v.transform.rotated(by: g.rotation)
        g.rotation = 0
    }

    // Cho pinch và rotate chạy cùng lúc (mặc định 2 recognizer không chạy đồng thời)
    func gestureRecognizer(_ g: UIGestureRecognizer,
                           shouldRecognizeSimultaneouslyWith other: UIGestureRecognizer) -> Bool { true }
}
```

### §8. CALayer, render loop và offscreen rendering

**UIView vs CALayer:**
| | `UIView` | `CALayer` |
|---|---|---|
| Framework | UIKit | Core Animation (QuartzCore) |
| Nhận touch / responder chain | ✅ | ❌ |
| Auto Layout, accessibility | ✅ | ❌ |
| Vẽ / hiển thị nội dung | Không tự làm — **uỷ cho layer** | ✅ (contents bitmap, border, shadow, cornerRadius, transform 3D) |
| Animation | API tiện (`UIView.animate`) — thực chất tạo `CAAnimation` | Implicit + explicit animation |
| Quan hệ | Mỗi view **có** một backing layer (`view.layer`) và là **delegate** của layer đó | Có thể dùng layer độc lập (nhẹ hơn): `CAShapeLayer`, `CAGradientLayer` |

`frame`/`bounds`/`center` của view thực chất đọc/ghi `layer.frame`/`bounds`/`position`. Layer của view **tắt implicit animation** (view trả `NSNull` trong `action(for:forKey:)`), còn layer độc lập thì đổi thuộc tính là tự animate 0,25s.

**Render loop — một khung hình đi từ code tới màn hình:**
```
 ┌───────────── APP PROCESS ─────────────┐   ┌──── RENDER SERVER (backboardd) ────┐   ┌─ GPU ─┐
 │ Event: touch, timer, network callback │   │                                    │   │       │
 │   → code đổi view/layer               │   │  Render prepare: decode layer tree │   │ vẽ    │
 │ Commit (cuối vòng RunLoop):           │──►│  Render execute: gửi lệnh vẽ ──────┼──►│ pixel │──► Display
 │   ① layout  ② display (draw)          │   │                                    │   │       │
 │   ③ prepare (decode ảnh)  ④ commit    │   └────────────────────────────────────┘   └───────┘
 └───────────────────────────────────────┘
      mỗi khung hình chỉ có 16,67ms (60Hz) hoặc 8,33ms (120Hz ProMotion)
```
- **Hitch** (giật) = khung hình không kịp hiển thị đúng hạn. Hai loại: **commit hitch** (app tốn quá lâu ở main thread: layout phức tạp, `draw` nặng, decode ảnh lớn trên main) và **render hitch** (render server/GPU quá tải: offscreen pass, blur, quá nhiều layer trong suốt chồng nhau).
- Dụng cụ: Instruments → **Animation Hitches**; trong simulator: Debug → **Color Off-screen Rendered** (vàng), **Color Blended Layers** (đỏ).

**Offscreen rendering** = GPU không thể vẽ layer thẳng vào framebuffer, phải tạo **buffer tạm** ở chỗ khác, vẽ vào đó rồi mới ghép lại → tốn bộ nhớ và thời gian chuyển ngữ cảnh. Nguyên nhân hay gặp:

| Nguyên nhân | Vì sao offscreen | Cách tránh |
|---|---|---|
| `shadow` **không có `shadowPath`** | Phải render cả cây layer con để biết hình dạng (alpha) rồi mới tính bóng | Gán `layer.shadowPath` (cập nhật trong `layoutSubviews`) |
| `cornerRadius` + `masksToBounds`/`clipsToBounds` trên layer **có sublayer/subview** | Phải render nội dung con rồi mới cắt (mask) | Chỉ bo layer cần bo; với ảnh: vẽ sẵn ảnh bo góc ở background; dùng `cornerRadius` không clip nếu chỉ có `backgroundColor` |
| `layer.mask` | Mask tuỳ ý → luôn cần buffer tạm | Hạn chế, hoặc dùng ảnh đã cắt sẵn |
| `group opacity` (alpha < 1 trên view có subview) | Phải ghép các con trước rồi mới áp alpha | Áp alpha cho từng con nếu được |
| `UIVisualEffectView` (blur) | Bản chất cần đọc nội dung phía sau | Dùng ít, không đặt trong cell scroll nhiều |
| `shouldRasterize = true` | Cố ý render offscreen **một lần** rồi cache bitmap | Chỉ hợp nội dung **tĩnh**; nội dung đổi liên tục → tệ hơn. Nhớ `rasterizationScale = UIScreen.main.scale` |

> Ghi chú: chỉ `cornerRadius` trên view **không có sublayer** (vd. view chỉ có `backgroundColor`) không cần clip; và iOS hiện đại tối ưu được `cornerRadius + clipsToBounds` trên `UIImageView` đơn giản. Hãy **đo** bằng Color Off-screen Rendered thay vì đoán.

```swift
final class CardView: UIView {
    private let contentContainer = UIView()

    override init(frame: CGRect) {
        super.init(frame: frame)
        // Shadow ở view ngoài (không clip), bo góc + clip ở view trong
        layer.shadowColor = UIColor.black.cgColor
        layer.shadowOpacity = 0.15
        layer.shadowRadius = 8
        layer.shadowOffset = CGSize(width: 0, height: 4)

        contentContainer.layer.cornerRadius = 12
        contentContainer.layer.cornerCurve = .continuous
        contentContainer.clipsToBounds = true            // = layer.masksToBounds
        addSubview(contentContainer)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func layoutSubviews() {
        super.layoutSubviews()
        contentContainer.frame = bounds
        // ✅ Cho Core Animation biết sẵn hình dạng bóng → không phải render offscreen để tính từ alpha
        layer.shadowPath = UIBezierPath(roundedRect: bounds, cornerRadius: 12).cgPath
    }
}
```
→ Mẫu "shadow + bo góc": **không thể** có cả shadow lẫn `clipsToBounds` trên cùng một layer (clip sẽ cắt luôn bóng). Dùng 2 lớp: ngoài đổ bóng, trong bo góc + clip.

### §9. Navigation: push vs present

| | Push | Present (modal) |
|---|---|---|
| Cần | `UINavigationController` | Bất kỳ VC nào |
| Ý nghĩa UX | Đi sâu vào phân cấp (list → detail) | Ngắt luồng: tạo mới, đăng nhập, bộ lọc, xác nhận |
| Quay lại | Nút Back, vuốt cạnh trái, `popViewController` | `dismiss(animated:)` (gọi trên VC được present **hoặc** presenter), vuốt xuống (sheet) |
| Quan hệ | `navigationController.viewControllers` (stack) | `presentingViewController` ↔ `presentedViewController` |

**`modalPresentationStyle`** — ảnh hưởng trực tiếp tới lifecycle VC bên dưới:
| Style | Hình dạng | VC bên dưới bị gỡ khỏi màn hình? | VC dưới nhận `viewWillDisappear`? |
|---|---|---|---|
| `.automatic` → `.pageSheet` (mặc định iOS 13+) | Card, thấy mép màn trước | Không | **Không** |
| `.formSheet` | Card nhỏ giữa màn (iPad) | Không | **Không** |
| `.fullScreen` | Che hết | **Có** (sau animation) | Có |
| `.overFullScreen` | Che hết nhưng **giữ** view bên dưới (cho nền trong suốt) | Không | **Không** |
| `.overCurrentContext` / `.currentContext` | Chỉ che VC có `definesPresentationContext = true` | Tuỳ | Tuỳ |
| `.popover` | Bong bóng (iPad), trên iPhone thích ứng thành sheet | Không | Không |

```swift
final class RepoListViewController: UIViewController {
    func showDetail(id: Int) {
        let detail = UIViewController()
        navigationController?.pushViewController(detail, animated: true)      // push: cần nằm trong UINavigationController
    }

    func showFilter() {
        let filter = UINavigationController(rootViewController: UIViewController())
        filter.modalPresentationStyle = .pageSheet                            // mặc định từ iOS 13 (card)
        if let sheet = filter.sheetPresentationController {
            sheet.detents = [.medium(), .large()]                             // iOS 15+: half sheet
            sheet.prefersGrabberVisible = true
        }
        filter.isModalInPresentation = true                                   // chặn vuốt xuống để đóng (vd. form chưa lưu)
        present(filter, animated: true)
    }

    func showPlayer() {
        let player = UIViewController()
        player.modalPresentationStyle = .fullScreen                           // presenting VC nhận viewWillDisappear/DidDisappear
        player.modalTransitionStyle = .crossDissolve
        present(player, animated: true)
    }

    func close() {
        if presentingViewController != nil { dismiss(animated: true) }       // dismiss do VC được present gọi (hoặc presenter)
        else { navigationController?.popViewController(animated: true) }
    }
}
```
Lưu ý: một VC chỉ present được **một** VC tại một thời điểm — present khi đang có VC khác được present → warning *"Attempt to present … which is already presenting …"* và không có gì xảy ra. Present trong `viewDidLoad` cũng thất bại (view chưa nằm trong window) — làm ở `viewDidAppear`.

---

## ⚠️ Bẫy hay gặp

1. **Truy cập `vc.view` trong `init` hoặc trước khi push** → ép `loadView` + `viewDidLoad` chạy sớm (cả khi VC chưa hiện). Kiểm tra bằng `isViewLoaded`.
2. **Đọc `view.bounds` trong `viewDidLoad` để tính layout** → kích thước chưa đúng (còn là size của storyboard/nib hoặc .zero). Dùng Auto Layout, hoặc `viewIsAppearing`/`viewDidLayoutSubviews`.
3. **Refresh dữ liệu đặt ở `viewWillAppear` nhưng màn con là page sheet** → dismiss xong không refresh (VC dưới không nhận appear callback).
4. **Quên gọi `super`** trong các callback lifecycle (đặc biệt `viewWillAppear`, `layoutSubviews` khi dùng Auto Layout).
5. **Thêm constraint trong `viewDidLayoutSubviews`/`layoutSubviews`** → vòng layout vô hạn hoặc sai ngẫu nhiên.
6. **Logic background đặt ở AppDelegate** trong app dùng scene → không bao giờ chạy.
7. **Đọc `frame` khi view đang có transform** → giá trị vô nghĩa. Dùng `bounds` + `center`.
8. **`Timer.scheduledTimer` đứng khi scroll** (mode `.default`), và **Timer giữ target** → leak nếu không `invalidate`.
9. **Gesture trên view cha "nuốt" `didSelectRow`** → `cancelsTouchesInView = false`.
10. **Nút lòi ra ngoài bounds cha không bấm được** — hit-test của cha trả nil trước.
11. **`shadow` + `clipsToBounds` trên cùng layer** → mất bóng. **Shadow không `shadowPath`** → offscreen rendering, giật khi scroll.
12. **`addSubview(child.view)` mà không `addChild`** → child không nhận lifecycle, không có `navigationController`.

---

## 🛠 Bài tập

> Tạo project iOS `practice/week09/LifecycleLab` (UIKit, không storyboard). Mọi bài trong tuần làm trong project này.

**Bài 1 — Log app lifecycle (T2)**
Xoá `Main.storyboard`, tự tạo window trong `SceneDelegate` như §1. Log mọi callback của AppDelegate và SceneDelegate. **Dự đoán trước trên giấy** log của 4 kịch bản, rồi chạy kiểm chứng: (a) cold launch, (b) về Home rồi mở lại, (c) kéo Control Center xuống rồi đóng, (d) mở App Switcher rồi vuốt app lên (kill).
<details><summary>Lời giải</summary>

(a), (b): như log ở §1.
(c) Control Center: `sceneWillResignActive` → `sceneDidBecomeActive` (không vào background).
(d) Mở App Switcher: `sceneWillResignActive`. Vuốt kill khi app đang suspended: **không có log nào** (process bị kill thẳng). Nếu app vẫn đang chạy ở background, có thể thấy `sceneDidDisconnect`/`applicationWillTerminate` — nhưng **không được dựa vào nó**.
Thêm: `applicationDidBecomeActive` của AppDelegate **không** in ra vì app dùng scene.
</details>

**Bài 2 — Lifecycle logger (T3)**
Viết `class LoggingViewController: UIViewController` override mọi callback (§2) và in `"\(name) <callback>"`, kể cả `willMove/didMove(toParent:)` và `deinit`. Dùng nó để **dự đoán rồi kiểm chứng** thứ tự cho: push B, pop B, present C `.fullScreen`, dismiss C, present D `.pageSheet`, dismiss D.
<details><summary>Lời giải</summary>

Bảng log thật ở §2. Dismiss C (`.fullScreen`): `C viewWillDisappear → A viewWillAppear → A viewIsAppearing → A viewDidAppear → C viewDidDisappear → C deinit`. Dismiss D (`.pageSheet`): `D viewWillDisappear → D viewDidDisappear → D deinit` — **A không nhận gì**.
Nếu `deinit` của B/C/D không in → bạn có retain cycle (closure giữ self, delegate strong...).
</details>

**Bài 3 — Đặt code vào đâu? (T3)**
Ghép mỗi việc với callback phù hợp nhất và giải thích 1 câu: (1) tạo `UITableView` và constraint; (2) gọi API lấy danh sách lần đầu; (3) bắt đầu video autoplay; (4) set `layer.cornerRadius = button.bounds.height / 2`; (5) cập nhật số lượng giỏ hàng mỗi lần quay lại màn; (6) gửi analytics "screen_view"; (7) huỷ request đang chạy khi rời màn; (8) chọn layout 1 hay 2 cột theo `traitCollection.horizontalSizeClass`.
<details><summary>Lời giải</summary>

(1) `viewDidLoad` — một lần. (2) `viewDidLoad` — tránh gọi lại mỗi lần quay về. (3) `viewDidAppear` — chỉ khi đã hiện thật. (4) `viewDidLayoutSubviews` (hoặc `layoutSubviews` của view con) — cần bounds cuối. (5) `viewWillAppear` — nhưng nhớ bẫy sheet. (6) `viewDidAppear` — user thực sự thấy màn. (7) `viewDidDisappear` (hoặc `viewWillDisappear` + kiểm tra `isMovingFromParent`). (8) `viewIsAppearing` — trait collection đã chính xác; trước iOS 17 người ta phải làm ở `viewWillLayoutSubviews` + cờ.
</details>

**Bài 4 — Container với segmented control (T4)**
`ProfileContainerViewController` có `UISegmentedControl` ["Repos", "Stars"] và một `container` view. Đổi segment → thay child VC tương ứng (đúng 3 bước add/remove). Xác nhận bằng log: child mới nhận `viewWillAppear`, child cũ nhận `viewDidDisappear`.
<details><summary>Lời giải</summary>

```swift
import UIKit

final class ProfileContainerViewController: UIViewController {
    private let segmented = UISegmentedControl(items: ["Repos", "Stars"])
    private let container = UIView()
    private lazy var tabs: [UIViewController] = [ReposTabViewController(), StarsTabViewController()]
    private var current: UIViewController?

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        [segmented, container].forEach {
            $0.translatesAutoresizingMaskIntoConstraints = false
            view.addSubview($0)
        }
        NSLayoutConstraint.activate([
            segmented.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 8),
            segmented.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            container.topAnchor.constraint(equalTo: segmented.bottomAnchor, constant: 8),
            container.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            container.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            container.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])
        segmented.addTarget(self, action: #selector(segmentChanged), for: .valueChanged)
        segmented.selectedSegmentIndex = 0
        show(tabs[0])
    }

    @objc private func segmentChanged() { show(tabs[segmented.selectedSegmentIndex]) }

    private func show(_ new: UIViewController) {
        guard new !== current else { return }
        if let old = current {                       // gỡ child cũ
            old.willMove(toParent: nil)
            old.view.removeFromSuperview()
            old.removeFromParent()
        }
        addChild(new)                                // gắn child mới
        new.view.frame = container.bounds
        new.view.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        container.addSubview(new.view)
        new.didMove(toParent: self)
        current = new
    }
}

final class ReposTabViewController: UIViewController {
    override func viewWillAppear(_ animated: Bool) { super.viewWillAppear(animated); print("Repos viewWillAppear") }
}
final class StarsTabViewController: UIViewController {
    override func viewWillAppear(_ animated: Bool) { super.viewWillAppear(animated); print("Stars viewWillAppear") }
}
```
Giữ các tab trong mảng `tabs` để không tạo lại (giữ state scroll). Đánh đổi: tốn RAM hơn.
</details>

**Bài 5 — frame vs bounds trên giấy (T4)**
View `v` có `frame = (20, 40, 100, 60)`, nằm trong superview. Tính (không chạy code): (a) `v.bounds`, `v.center`; (b) sau `v.transform = CGAffineTransform(scaleX: 0.5, y: 0.5)`: `frame`, `bounds`, `center`; (c) reset transform, đặt `v.bounds.origin = (0, 20)`: `frame` của `v` có đổi không? Subview có `frame = (0, 0, 10, 10)` hiển thị ở đâu trong `v`? Sau đó chạy trong simulator để kiểm tra.
<details><summary>Lời giải</summary>

(a) `bounds = (0, 0, 100, 60)`, `center = (70, 70)`.
(b) Scale quanh tâm: kích thước nhìn từ cha 50×30, tâm giữ (70, 70) → `frame = (45, 55, 50, 30)`; `bounds = (0, 0, 100, 60)` (không đổi); `center = (70, 70)`.
(c) `frame` **không** đổi. Subview vẫn có `frame = (0, 0, 10, 10)` nhưng hiển thị ở y = 0 − 20 = −20 trong vùng nhìn thấy của `v` (bị đẩy lên, bị che nếu `clipsToBounds`). Giống như `v` là scroll view đã cuộn xuống 20pt.
</details>

**Bài 6 — RunLoop & hit-test (T5)**
a) Đặt một `UILabel` đếm giây (Timer 1s) phía trên một `UITableView` 100 dòng. Scroll và giữ tay → số có đứng không? Sửa để không đứng. Rời màn hình → kiểm tra VC `deinit`.
b) Làm nút "♥" 20×20 trong cell, mở rộng vùng chạm lên 44×44 **mà không đổi kích thước hiển thị**.
c) Gắn `UITapGestureRecognizer` lên `view` của VC để ẩn bàn phím. Quan sát `didSelectRowAt` có còn chạy không? Sửa.
<details><summary>Lời giải</summary>

a) Dùng `Timer(timeInterval:repeats:block:)` + `RunLoop.main.add(t, forMode: .common)`; `invalidate()` ở `viewWillDisappear`; closure dùng `[weak self]` (code §6).
b) `ExpandedButton` với `touchInsets = UIEdgeInsets(top: -12, left: -12, bottom: -12, right: -12)` (20 + 24 = 44). Lưu ý vùng mở rộng vẫn phải nằm trong bounds của **cell** (cha), nếu không hit-test của cha đã loại trước.
c) Không chạy, vì tap recognizer nhận diện xong thì huỷ touch của table. Sửa: `tap.cancelsTouchesInView = false`.
</details>

**Bài 7 — Offscreen rendering (T6)**
Tạo `UICollectionView` 500 cell, mỗi cell là card có ảnh bo góc + shadow (cố ý **không** set `shadowPath`, và đặt `cornerRadius + clipsToBounds` + shadow trên **cùng** một view). Bật Simulator → Debug → **Color Off-screen Rendered**, chụp màn hình. Sửa theo mẫu `CardView` §8, chụp lại. Nếu có máy thật, đo bằng Instruments → Animation Hitches trước/sau.
<details><summary>Gợi ý</summary>

Trước khi sửa: bóng biến mất (bị clip) **hoặc** nếu bỏ clip thì card tô vàng (offscreen). Sau khi tách 2 lớp và có `shadowPath`: lớp vàng biến mất phần lớn. Ghi lại số hitch ms/s trước/sau nếu đo trên máy thật.
</details>

**🏋️ Bài tập lớn (T7) — LifecycleLab hoàn chỉnh**
Một app UIKit viết **hoàn toàn bằng code** (không storyboard), có:
- Màn **Home** (`UITableView` 5 dòng), mỗi dòng mở một thí nghiệm:
  1. *Push/Pop* — `LoggingViewController` push lồng 3 cấp, có nút "Pop to root".
  2. *Present styles* — 4 nút present `.pageSheet`, `.fullScreen`, `.overFullScreen`, sheet `.medium()`; hiển thị log lifecycle của **cả** VC dưới lẫn VC trên trong một `UITextView` ở cuối màn.
  3. *Container* — Bài 4.
  4. *RunLoop* — Bài 6a, có switch "Dùng .common" bật/tắt để so sánh.
  5. *Hit-test* — `PassThroughView` phủ lên nút, `ExpandedButton`, nút lòi ra ngoài card dùng `OverflowView`.
- Một `LifecycleLogger` singleton `@MainActor` gom log (thời gian + tên VC + callback), có nút "Clear".
- Kiểm tra mọi VC đều `deinit` khi rời màn (in ra log). Không có warning Swift 6 mode.

> **📱 Side project — Milestone 9 (T7):** trong RepoScope, dựng màn danh sách repo bằng **UIKit viết code** (`UITableView` đơn giản là đủ — tuần 10 sẽ đổi sang compositional layout), log lifecycle của VC list và detail, push sang detail khi chọn dòng, đảm bảo detail `deinit` khi pop. Chi tiết: [side-project](../tracks/side-project.md).

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Kể các trạng thái của một app iOS.**
<details><summary>Đáp án gợi ý</summary>

Not running; Inactive — ở foreground nhưng không nhận event (đang chuyển trạng thái, có cuộc gọi, kéo Control Center); Active — foreground và nhận event; Background — không hiển thị nhưng vẫn chạy code trong thời gian ngắn (hoặc lâu hơn nếu có background mode/task); Suspended — trong bộ nhớ nhưng không chạy code, hệ thống có thể kill bất cứ lúc nào **mà không báo**. Vì vậy dữ liệu quan trọng phải lưu khi vào background.
</details>

**Q2 ⭐ AppDelegate và SceneDelegate khác nhau thế nào? Vì sao có SceneDelegate?**
<details><summary>Đáp án gợi ý</summary>

Từ iOS 13, một app có thể có nhiều scene (nhiều cửa sổ trên iPad/Mac). AppDelegate xử lý sự kiện cấp process: launch, cấu hình scene, push token, background URLSession. SceneDelegate quản lý một instance UI: tạo window, foreground/background, deep link, state restoration của scene đó. Khi dùng scene, các callback trạng thái UI của AppDelegate như `applicationDidEnterBackground` không còn được gọi.
</details>

**Q3 ⭐ Kể lifecycle của UIViewController theo thứ tự.**
<details><summary>Đáp án gợi ý</summary>

`init` → `loadView` → `viewDidLoad` (một lần) → `viewWillAppear` → `viewIsAppearing` (iOS 17, back-deploy 13) → `viewWillLayoutSubviews` → `viewDidLayoutSubviews` (có thể nhiều lần) → `viewDidAppear` → `viewWillDisappear` → `viewDidDisappear` → `deinit`. View được load lười khi `view` được truy cập lần đầu. Hiện lại thì lặp từ `viewWillAppear`, không gọi lại `viewDidLoad`.
</details>

**Q4 ⭐ Push B từ A: thứ tự callback của A và B?**
<details><summary>Đáp án gợi ý</summary>

B init → B loadView → B viewDidLoad → A viewWillDisappear → B viewWillAppear → B viewIsAppearing → B layout → A viewDidDisappear → B viewDidAppear. Tức là B load view xong mới bắt đầu transition, các "will" xảy ra trước, "did" xảy ra sau khi animation kết thúc. Pop thì ngược lại: B viewWillDisappear → A viewWillAppear → B viewDidDisappear → A viewDidAppear → B deinit.
</details>

**Q5 ⭐ Present một VC dạng sheet, VC bên dưới có nhận `viewWillDisappear` không?**
<details><summary>Đáp án gợi ý</summary>

Không. Với `.pageSheet`/`.formSheet` (mặc định từ iOS 13) và `.overFullScreen`/`.overCurrentContext`, view của VC dưới vẫn nằm trong hierarchy nên không nhận disappear, và khi dismiss cũng không nhận `viewWillAppear`. Chỉ `.fullScreen` (và `.currentContext`) mới gỡ view dưới. Hệ quả: logic refresh đặt ở `viewWillAppear` sẽ không chạy sau khi đóng sheet; dùng callback/delegate hoặc `presentationControllerDidDismiss`.
</details>

**Q6 `viewIsAppearing` là gì, khác `viewWillAppear` thế nào?**
<details><summary>Đáp án gợi ý</summary>

Thêm ở iOS 17 (back-deploy tới iOS 13). Chạy sau `viewWillAppear`, khi view đã được thêm vào hierarchy và trait collection, kích thước đã chính xác, nhưng trước khi hiện lên. Ở `viewWillAppear` view có thể chưa trong window nên trait/size chưa đúng; `viewDidAppear` thì quá muộn, user thấy UI nhảy. Hợp để cấu hình UI dựa trên size/trait, scroll tới vị trí ban đầu.
</details>

**Q7 Gọi API ở `viewDidLoad` hay `viewWillAppear`?**
<details><summary>Đáp án gợi ý</summary>

Tải dữ liệu ban đầu ở `viewDidLoad` vì chỉ chạy một lần. `viewWillAppear` dùng cho dữ liệu cần làm mới mỗi lần quay lại, nhưng phải cẩn thận vì nó chạy nhiều lần (kể cả khi dismiss full-screen modal, quay lại từ tab khác) và không chạy khi đóng sheet. Thực tế logic tải nằm trong ViewModel, VC chỉ gọi trigger.
</details>

**Q8 Thêm/gỡ child view controller thế nào? Vì sao không chỉ `addSubview`?**
<details><summary>Đáp án gợi ý</summary>

Thêm: `addChild(child)` (tự gọi `willMove`), `addSubview(child.view)` + layout, `child.didMove(toParent: self)`. Gỡ: `child.willMove(toParent: nil)`, `child.view.removeFromSuperview()`, `child.removeFromParent()` (tự gọi `didMove(nil)`). Chỉ `addSubview` thì child không được forward lifecycle (appear/disappear, trait, xoay), không có `parent`/`navigationController`, responder chain sai.
</details>

**Q9 ⭐ `setNeedsLayout` vs `layoutIfNeeded` vs `layoutSubviews`?**
<details><summary>Đáp án gợi ý</summary>

`setNeedsLayout` chỉ đánh dấu view cần layout, trả về ngay; layout thật xảy ra ở chu kỳ cập nhật tiếp theo (cuối vòng runloop), gom nhiều thay đổi thành một lần. `layoutIfNeeded` ép layout **ngay lập tức** nếu có thay đổi đang chờ — dùng trong animation block để animate constraint, hoặc khi cần frame đúng ngay. `layoutSubviews` là nơi hệ thống tính frame — ta override, không gọi trực tiếp.
</details>

**Q10 `setNeedsDisplay` và `draw(_:)` dùng khi nào? Có nên override `draw` không?**
<details><summary>Đáp án gợi ý</summary>

`setNeedsDisplay` đánh dấu nội dung cần vẽ lại, chu kỳ sau hệ thống gọi `draw(_:)`. Override `draw` chỉ khi cần vẽ custom bằng Core Graphics. Nó tạo backing store bitmap (tốn RAM theo kích thước × scale²) và vẽ trên CPU ở main thread; nếu chỉ cần hình đơn giản thì dùng `CAShapeLayer`, `CAGradientLayer` hoặc view con sẽ hiệu quả hơn.
</details>

**Q11 ⭐ frame và bounds khác nhau thế nào? Khi xoay view 45° thì sao?**
<details><summary>Đáp án gợi ý</summary>

`frame` là vị trí, kích thước trong hệ toạ độ superview; `bounds` là trong hệ toạ độ của chính view (origin thường là 0,0). Xoay 45°: `bounds` giữ nguyên kích thước, `center` giữ nguyên, còn `frame` thành hình chữ nhật nhỏ nhất bao quanh view đã xoay — to hơn. Khi transform khác identity, Apple nói `frame` không xác định, không nên dùng. Đổi `bounds.origin` sẽ dịch nội dung con — đó là cách UIScrollView cuộn (`contentOffset` chính là `bounds.origin`).
</details>

**Q12 ⭐ RunLoop là gì? Vì sao Timer dừng khi scroll?**
<details><summary>Đáp án gợi ý</summary>

RunLoop là vòng lặp sự kiện gắn với thread: xử lý input source, timer, block của main queue, rồi ngủ khi không có việc. Mỗi lần chạy ở một mode và chỉ xử lý source/timer của mode đó. `Timer.scheduledTimer` đăng ký ở `.default`; khi scroll, main runloop chuyển sang `.tracking` nên timer không chạy. Sửa: add timer vào `.common` (gồm default + tracking), hoặc dùng `DispatchSourceTimer`/`Task.sleep`. Nhớ `invalidate` vì runloop giữ strong timer.
</details>

**Q13 Thread nền có RunLoop không? Gọi `Timer.scheduledTimer` trong `DispatchQueue.global().async` thì sao?**
<details><summary>Đáp án gợi ý</summary>

Mỗi thread có thể có một RunLoop, được tạo lười khi truy cập `RunLoop.current`, nhưng không tự chạy. Thread của GCD không chạy runloop nên timer được thêm vào đó không bao giờ fire. Cần timer nền thì dùng `DispatchSourceTimer`, hoặc schedule trên main runloop.
</details>

**Q14 ⭐ Hit-testing hoạt động thế nào? Làm sao mở rộng vùng chạm của một nút?**
<details><summary>Đáp án gợi ý</summary>

Window gọi `hitTest(_:with:)` từ root view, đệ quy xuống: view bị bỏ qua nếu hidden, không bật user interaction, alpha < 0.01, hoặc điểm nằm ngoài `point(inside:)`; duyệt subview từ trên cùng xuống, view sâu nhất chứa điểm được chọn. Mở rộng vùng chạm: override `point(inside:with:)` trả `bounds.inset(by: negativeInsets).contains(point)`. Lưu ý vùng mở rộng vẫn phải nằm trong bounds của view cha, nếu không cha sẽ loại trước.
</details>

**Q15 Responder chain là gì? Dùng vào việc gì?**
<details><summary>Đáp án gợi ý</summary>

Chuỗi các `UIResponder` nối bằng `next`: view → superview → … → view controller → window → scene → application → app delegate. Event không được xử lý sẽ chuyển lên responder tiếp theo. Dùng cho touch chưa xử lý, bàn phím (first responder), menu edit, key command, và `sendAction(_:to: nil, ...)` để cell gửi action lên VC mà không cần delegate. Khác với hit-testing (đi xuống tìm view), responder chain đi lên.
</details>

**Q16 Gesture recognizer và `touchesBegan` liên quan thế nào? Vì sao thêm tap gesture làm table không chọn được row?**
<details><summary>Đáp án gợi ý</summary>

Touch được gửi cho gesture recognizer của view bị hit và các view tổ tiên trước. Khi recognizer nhận diện thành công, mặc định `cancelsTouchesInView = true` nên view nhận `touchesCancelled` — table không hoàn tất selection. Sửa bằng `cancelsTouchesInView = false` hoặc delegate `gestureRecognizer(_:shouldReceive:)` để bỏ qua touch trong cell. Phối hợp nhiều recognizer dùng `require(toFail:)` và `shouldRecognizeSimultaneouslyWith`.
</details>

**Q17 ⭐ UIView và CALayer khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

CALayer (Core Animation) lo hiển thị: bitmap contents, border, cornerRadius, shadow, transform, animation; được render bởi render server và GPU. UIView (UIKit) bọc một backing layer, là delegate của nó, và bổ sung xử lý sự kiện (UIResponder), Auto Layout, accessibility. Layer nhẹ hơn nên dùng sublayer cho thứ chỉ để vẽ (gradient, shape). Layer của view tắt implicit animation; layer độc lập thì có.
</details>

**Q18 Mô tả render loop. Hitch là gì?**
<details><summary>Đáp án gợi ý</summary>

Mỗi khung hình: app xử lý event, thay đổi view/layer; cuối vòng runloop Core Animation commit transaction — layout, display (`draw`), prepare (decode ảnh), rồi gửi layer tree sang render server. Render server chuẩn bị và gửi lệnh cho GPU vẽ, rồi hiển thị. Với 60Hz mỗi khung chỉ có ~16,7ms (120Hz: ~8,3ms). Hitch là khi khung hình hiển thị trễ — commit hitch do main thread làm quá nhiều; render hitch do render server/GPU quá tải (offscreen, blur, blending). Đo bằng Instruments → Animation Hitches.
</details>

**Q19 ⭐ Offscreen rendering là gì? Kể các nguyên nhân và cách tránh.**
<details><summary>Đáp án gợi ý</summary>

GPU phải render layer vào một buffer tạm rồi mới ghép vào khung hình, tốn bộ nhớ và chuyển ngữ cảnh — gây giật khi scroll. Nguyên nhân: shadow không có `shadowPath`, `cornerRadius` + `masksToBounds` trên layer có nội dung con, `mask`, group opacity, blur, `shouldRasterize`. Tránh: set `shadowPath`; tách view đổ bóng và view bo góc; bo sẵn ảnh ở background; chỉ dùng `shouldRasterize` cho nội dung tĩnh. Kiểm tra bằng Simulator → Color Off-screen Rendered.
</details>

**Q20 Push vs present? Các `modalPresentationStyle` khác nhau ở đâu?**
<details><summary>Đáp án gợi ý</summary>

Push cần navigation controller, dùng khi đi sâu vào phân cấp, quay lại bằng back/vuốt cạnh. Present là modal, ngắt luồng (tạo mới, login, filter), đóng bằng `dismiss`. Mặc định iOS 13+ là `.pageSheet` dạng card, vuốt xuống để đóng (chặn bằng `isModalInPresentation`), hỗ trợ detent `.medium()`; `.fullScreen` che toàn bộ và gỡ view bên dưới; `.overFullScreen` che toàn bộ nhưng giữ view dưới để làm nền trong suốt. Style ảnh hưởng tới việc VC dưới có nhận appear/disappear hay không.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Bài tập lớn LifecycleLab chạy được, mọi VC `deinit` đúng
- [ ] Vẽ tay không nhìn: sơ đồ 5 trạng thái app, sơ đồ VC lifecycle, thuật toán hit-test
- [ ] Viết lại từ trí nhớ bảng thứ tự **push B** và giải thích bẫy **page sheet**
- [ ] Trả lời được ≥ 16/20 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "frame vs bounds" (có ví dụ xoay + scroll view)
- [ ] Ghi âm 2 phút: "Vì sao Timer dừng khi scroll" (RunLoop mode)
- [ ] Ghi âm 2 phút: "Offscreen rendering và cách tránh"
- [ ] Làm xong 6 bài DSA tuần 9 (Trees — DFS)
- [ ] Side project Milestone 9 xong, đã commit
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [Apple — Managing your app's life cycle](https://developer.apple.com/documentation/uikit/managing-your-app-s-life-cycle)
- [Apple — UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller) (phần *Responding to view-related events*) · [`viewIsAppearing(_:)`](https://developer.apple.com/documentation/uikit/uiviewcontroller/viewisappearing(_:))
- [Apple — Creating a custom container view controller](https://developer.apple.com/documentation/uikit/creating-a-custom-container-view-controller)
- [Apple — Using responders and the responder chain to handle events](https://developer.apple.com/documentation/uikit/using-responders-and-the-responder-chain-to-handle-events)
- [Apple — View Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/) (cũ nhưng phần *View Geometry* và *Windows* vẫn đúng)
- [Apple — Threading Programming Guide: Run Loops](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
- Apple Tech Note **TN3187** — *Migrating to the UIKit scene-based life cycle*
- WWDC19 — *Architecting Your App for Multiple Windows* (scene)
- WWDC23 — *What's new in UIKit* (`viewIsAppearing`)
- WWDC21 — *Explore UI animation hitches and the render loop* · Tech Talk — *Find and fix hitches in the commit phase* / *Demystify and eliminate hitches in the render phase*
- WWDC14 — *Advanced Graphics and Animations for iOS Apps* (offscreen rendering, blending — kinh điển)
- objc.io Issue #3 *Views* — "Getting Pixels onto the Screen" · Blog của ibireme — "iOS 保持界面流畅的技巧" (bản dịch tiếng Anh: *Tips for keeping the UI smooth*)
