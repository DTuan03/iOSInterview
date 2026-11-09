# Tuần 17 — Push · Background · Deep link · Security · Performance · Build/Release · CI/CD · Git (01/03 – 07/03/2027)

> **Giai đoạn 4 · Kiến trúc & Hệ thống** · Thời lượng: ~15 giờ · **Kết thúc mốc M5**
> Tuần này rộng chứ không sâu: nhiều chủ đề "đã từng làm nhưng nhờ người khác/AI setup". Người phỏng vấn dùng chúng để kiểm tra **bạn đã thật sự ship app chưa**: *"Push đi từ server đến máy thế nào?"*, *"Crash log không đọc được thì làm gì?"*, *"Provisioning profile là gì?"*.
> 📮 **Bắt đầu rải CV từ tuần này** (theo README): mỗi tối dành 15 phút gửi 2–3 CV. CV phải có link GitHub RepoScope.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Vẽ luồng **APNs** từ lúc xin quyền đến lúc notification hiện trên máy; phân biệt alert push và silent push; biết Notification Service Extension làm gì
- [ ] Nói được app có thể chạy nền **những cách nào** và giới hạn của từng cách
- [ ] Phân biệt **custom URL scheme** và **universal link**, biết file `apple-app-site-association`
- [ ] Lưu token bằng **Keychain** đúng cách; giải thích ATS, SSL pinning, vì sao "không có secret nào an toàn trong app"
- [ ] Giải thích launch time (pre-main / post-main), dùng Instruments: App Launch, Time Profiler, Hangs
- [ ] Đọc một crash log, giải thích **dSYM và symbolication**
- [ ] Giải thích certificate, App ID, provisioning profile, entitlements; Debug/Release, scheme, `.xcconfig`
- [ ] Mô tả pipeline CI/CD; trả lời câu hỏi Git: rebase vs merge, xử lý conflict, git flow
- [ ] **Hoàn thành mốc M5**: RepoScope có network + cache + test + kiến trúc rõ ràng, CI xanh, đẩy lên GitHub
- [ ] Làm xong 6 bài DSA tuần 17 — Intervals + Greedy (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 01/03** | 2h | §1 **Push notification** · §2 Background execution. Bài tập 1 · DSA: Merge Intervals · 📮 Gửi 3 CV đầu tiên | Sơ đồ APNs vẽ tay |
| **T3 02/03** | 2h | §3 Deep link & universal link · §4 **Security** (Keychain, ATS, pinning, biometrics). Bài tập 2, 3 · DSA: Insert Interval | `KeychainStore.swift` |
| **T4 03/03** | 2h | §5 **Performance** (launch, Time Profiler, hang, energy, size) · §6 Crash & symbolication. Bài tập 4 · DSA: Non-overlapping Intervals | 1 trace Instruments của RepoScope |
| **T5 04/03** | 2h | §7 **Build & release**: configuration, xcconfig, code signing, TestFlight, App Review. Bài tập 5 · DSA: Meeting Rooms II | `Debug.xcconfig`, `Release.xcconfig` |
| **T6 05/03** | 1,5h | §8 CI/CD · §9 **Git**. Bài tập 6 · DSA: Jump Game | Workflow `.github/workflows/ci.yml` |
| **T7 06/03** | 4h | Bài tập lớn. Side project: Milestone 17 (GitHub Actions CI, profiling pass, token trong Keychain, release config) · DSA: Gas Station | CI xanh trên GitHub |
| **CN 07/03** | 2h | **🏁 M5 self-test** (cuối file) · tự trả lời 20 câu phỏng vấn · tick checklist | M5 đạt, Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. Push notification

**Luồng APNs (Apple Push Notification service):**
```
 ┌───────────── iPhone ─────────────┐                                  ┌──────────────┐
 │ App                               │ ① requestAuthorization (alert,   │              │
 │  │ ② registerForRemote...         │    badge, sound) → user đồng ý   │              │
 │  ▼                                │                                  │              │
 │ iOS ────────③ xin token─────────────────────────────────────────────▶│     APNs     │
 │  │◀───────── ④ device token ◀─────────────────────────────────────── │   (Apple)    │
 │  ▼                                │                                  │              │
 │ didRegisterForRemote...           │                                  │              │
 │  WithDeviceToken ──⑤ gửi token──────────▶ ┌────────────────┐         │              │
 │                                   │       │ Backend của bạn │ ─⑥ HTTP/2 POST ─────▶ │
 │                                   │       │ (lưu token/user)│  token + payload       │
 │                                   │       └────────────────┘  + JWT (.p8 key)        │
 │ ⑧ hiện banner / đánh thức app ◀────────────────────────────── ⑦ APNs đẩy tới máy ◀───┘
 └───────────────────────────────────┘
```
1. Xin quyền hiển thị (alert/badge/sound) — **user có thể từ chối**.
2. `registerForRemoteNotifications()` — xin device token (làm được kể cả khi user từ chối hiển thị; silent push vẫn nhận được).
3–4. iOS liên hệ APNs, trả **device token** — định danh *app này trên máy này*. Token **có thể đổi** (cài lại app, restore máy) → gửi lên server **mỗi lần launch**, đừng cache vĩnh viễn.
5. App gửi token lên backend của bạn, gắn với user.
6. Backend gửi request HTTP/2 tới APNs với token + payload, xác thực bằng **token-based auth (.p8 key, JWT)** — một key dùng cho mọi app trong team, không hết hạn — hoặc certificate `.p12` cũ (hết hạn mỗi năm).
7–8. APNs đẩy tới máy (store-and-forward: máy offline thì APNs giữ **notification mới nhất** một thời gian).

Môi trường: **sandbox** (build Debug ký bằng development profile) và **production** (TestFlight/App Store). Token sandbox gửi lên server production → lỗi `BadDeviceToken` — lỗi rất hay gặp.

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        UNUserNotificationCenter.current().delegate = self   // set SỚM, trước khi launch xong
        return true
    }

    func requestPushPermission() async {
        let center = UNUserNotificationCenter.current()
        let granted = (try? await center.requestAuthorization(options: [.alert, .badge, .sound])) ?? false
        guard granted else { return }
        await MainActor.run { UIApplication.shared.registerForRemoteNotifications() }
    }

    func application(_ application: UIApplication,
                     didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        let token = deviceToken.map { String(format: "%02x", $0) }.joined()   // Data → hex string
        print("APNs token:", token)   // gửi lên backend
    }

    // App đang FOREGROUND: mặc định iOS không hiện banner — phải trả options
    nonisolated func userNotificationCenter(_ center: UNUserNotificationCenter,
                                            willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
        [.banner, .sound]
    }

    // User TAP vào notification
    nonisolated func userNotificationCenter(_ center: UNUserNotificationCenter,
                                            didReceive response: UNNotificationResponse) async {
        let userInfo = response.notification.request.content.userInfo
        guard let repo = userInfo["repo"] as? String else { return }
        await MainActor.run { print("Open repo", repo) }   // điều hướng UI trên main (qua Coordinator)
    }

    // SILENT push (content-available: 1)
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any]) async -> UIBackgroundFetchResult {
        // tải dữ liệu mới — có khoảng ~30 giây
        .newData
    }
}
```
> Swift 6: hai method delegate của `UNUserNotificationCenterDelegate` phải đánh dấu `nonisolated`, nếu không compiler báo *"non-Sendable parameter type 'UNNotification' cannot be sent ... into main actor-isolated implementation"* (vì `AppDelegate` là `@MainActor` còn hệ thống gọi delegate từ thread khác).

**Payload** (JSON, tối đa **4KB**):
```json
{
  "aps": {
    "alert": { "title": "apple/swift", "body": "Có release mới 6.3" },
    "badge": 3,
    "sound": "default",
    "thread-id": "repo-apple-swift",
    "category": "RELEASE",
    "mutable-content": 1
  },
  "repo": "apple/swift",
  "image-url": "https://example.com/banner.png"
}
```
- `aps` là key của Apple; các key khác (như `repo`) là dữ liệu tuỳ ý cho app.
- `thread-id`: gom nhóm notification. `category`: gắn action button (`UNNotificationCategory`).
- `mutable-content: 1` → đi qua **Notification Service Extension** trước khi hiển thị.

**Silent push** (`"aps": { "content-available": 1 }`, không có alert, header `apns-push-type: background`, `apns-priority: 5`): đánh thức app ở nền để đồng bộ dữ liệu. **Không đảm bảo được giao**: hệ thống giới hạn (APNs khuyến cáo không quá vài cái mỗi giờ), không đánh thức nếu user đã **force-quit** app từ app switcher, Low Power Mode cũng ảnh hưởng. → Không dùng silent push cho tính năng quan trọng.

**Notification Service Extension (NSE)** — target riêng, chạy ~30 giây **trước khi** notification hiện: tải ảnh để đính kèm, **giải mã nội dung mã hoá end-to-end** (app nhắn tin), sửa title, đếm badge.
```swift
final class NotificationService: UNNotificationServiceExtension {
    private var contentHandler: ((UNNotificationContent) -> Void)?
    private var bestAttempt: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.contentHandler = contentHandler
        bestAttempt = request.content.mutableCopy() as? UNMutableNotificationContent
        guard let bestAttempt else { return contentHandler(request.content) }
        bestAttempt.title = "[RepoScope] " + bestAttempt.title
        // tải ảnh từ userInfo["image-url"] → lưu file tạm → UNNotificationAttachment
        contentHandler(bestAttempt)
    }

    override func serviceExtensionTimeWillExpire() {   // sắp hết giờ → giao bản tốt nhất đang có
        if let contentHandler, let bestAttempt { contentHandler(bestAttempt) }
    }
}
```
Extension là **process riêng**, không truy cập được bộ nhớ của app → chia sẻ dữ liệu (token để giải mã, DB) qua **App Group** (container chung) và Keychain access group. Giới hạn RAM rất thấp (~24MB).

Khác: **Notification Content Extension** (UI tuỳ chỉnh khi nhấn giữ notification), **local notification** (`UNNotificationRequest` với trigger thời gian/lịch/vị trí, không cần server), **Live Activities** (ActivityKit, cập nhật qua push type `liveactivity`).

### §2. Background execution

iOS **không cho app chạy nền tuỳ ý** (bảo vệ pin). Khi user về Home: app → *background* (~vài giây đến ~30 giây) → *suspended* (đóng băng trong RAM, không chạy code) → có thể bị *terminated* khi thiếu RAM, **không có callback**.

| Cách | Dùng khi | Giới hạn |
|---|---|---|
| `beginBackgroundTask(expirationHandler:)` | Làm nốt việc đang dở khi user rời app (lưu dữ liệu, upload nhỏ) | ~30 giây; phải gọi `endBackgroundTask`, không gọi → app bị kill |
| `BGAppRefreshTask` (BackgroundTasks) | Cập nhật nội dung ngắn định kỳ (feed mới) | ~30 giây; **hệ thống quyết định khi nào** dựa trên thói quen dùng app, pin, mạng |
| `BGProcessingTask` | Việc nặng: dọn DB, train ML, backup | Vài phút; thường khi sạc + idle; có thể yêu cầu mạng/sạc |
| Background `URLSession` | Download/upload file lớn | Chạy trong process hệ thống, **sống sót cả khi app bị kill**; app được đánh thức khi xong |
| Silent push | Server báo có dữ liệu mới | Không đảm bảo, bị throttle |
| Background modes đặc biệt | Audio, location, VoIP (PushKit), Bluetooth | Phải đúng mục đích, App Review kiểm tra gắt |

**BGTaskScheduler:**
```swift
// 1. Info.plist: BGTaskSchedulerPermittedIdentifiers = ["com.reposcope.refresh"]
//    + Signing & Capabilities → Background Modes → "Background fetch"
// 2. Đăng ký TRƯỚC khi didFinishLaunching kết thúc
func registerBackgroundTasks() {
    BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.reposcope.refresh", using: nil) { task in
        self.handleAppRefresh(task as! BGAppRefreshTask)
    }
}

// 3. Lên lịch (ví dụ khi app vào background)
func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.reposcope.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)   // SỚM NHẤT 15 phút nữa, không phải "đúng lúc"
    do { try BGTaskScheduler.shared.submit(request) } catch { print("Submit failed:", error) }
}

// 4. Xử lý
func handleAppRefresh(_ task: BGAppRefreshTask) {
    scheduleAppRefresh()                         // task là 1 lần → lên lịch lần sau
    let work = Task {
        // try await syncFavorites()
        task.setTaskCompleted(success: !Task.isCancelled)
    }
    task.expirationHandler = { work.cancel() }  // hết giờ → huỷ, nếu không app bị kill và bị "phạt"
}
```
Debug: pause app trong Xcode rồi chạy lệnh LLDB `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.reposcope.refresh"]`.

**Background URLSession:** `URLSessionConfiguration.background(withIdentifier:)`, chỉ download/upload task (upload phải từ **file**), phải dùng **delegate** (không dùng completion handler/async). Khi xong lúc app không chạy, iOS launch app nền và gọi `application(_:handleEventsForBackgroundURLSession:completionHandler:)` → tạo lại session cùng identifier, nhận event, gọi completion handler.

### §3. Deep link & Universal link

| | Custom URL scheme | Universal link |
|---|---|---|
| Dạng | `reposcope://repo/apple/swift` | `https://reposcope.app/repo/apple/swift` |
| Cấu hình | Info.plist `CFBundleURLTypes` | Entitlement `applinks:reposcope.app` + file **`apple-app-site-association`** trên server |
| Chưa cài app | Lỗi / không làm gì | Mở website (fallback tự nhiên) |
| Bảo mật | **App khác đăng ký cùng scheme được** → bị cướp link | Chỉ app được domain xác nhận mới mở → an toàn |
| Dùng cho | Link nội bộ, callback OAuth cũ, mở từ app khác | Link chia sẻ, email, marketing |

**apple-app-site-association** (AASA) đặt tại `https://reposcope.app/.well-known/apple-app-site-association`, JSON không đuôi, HTTPS, **không redirect**:
```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["ABCDE12345.com.reposcope.app"],
        "components": [ { "/": "/repo/*" }, { "/": "/search", "?": { "q": "?*" } } ]
      }
    ]
  }
}
```
`ABCDE12345` là **Team ID**. Từ iOS 14, máy tải AASA qua **CDN của Apple** (không gọi thẳng server bạn) → sửa file có thể mất thời gian mới cập nhật. Lưu ý: universal link gõ trực tiếp vào Safari hoặc mở từ chính domain đó thì **không** mở app.

**Xử lý trong app (SceneDelegate):** parse URL thành enum, đưa cho Coordinator (tuần 15).
```swift
enum DeepLink: Equatable {
    case repo(owner: String, name: String)
    case search(query: String)

    init?(url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else { return nil }
        // reposcope://repo/apple/swift  hoặc  https://reposcope.app/repo/apple/swift
        let parts: [String]
        if components.scheme == "reposcope" {
            parts = [components.host ?? ""] + url.pathComponents.filter { $0 != "/" }   // "repo" là host!
        } else {
            parts = url.pathComponents.filter { $0 != "/" }
        }
        switch parts.first {
        case "repo" where parts.count == 3:
            self = .repo(owner: parts[1], name: parts[2])
        case "search":
            guard let q = components.queryItems?.first(where: { $0.name == "q" })?.value else { return nil }
            self = .search(query: q)
        default:
            return nil
        }
    }
}

print(DeepLink(url: URL(string: "reposcope://repo/apple/swift")!)!)            // repo(owner: "apple", name: "swift")
print(DeepLink(url: URL(string: "https://reposcope.app/repo/apple/swift")!)!)  // repo(owner: "apple", name: "swift")
print(DeepLink(url: URL(string: "reposcope://search?q=swift%20ui")!)!)         // search(query: "swift ui")
print(DeepLink(url: URL(string: "https://reposcope.app/unknown")!) as Any)     // nil
```
```swift
final class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    // App mở từ trạng thái KILLED bằng link → link nằm trong connectionOptions
    func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {
        if let url = connectionOptions.urlContexts.first?.url { handle(url) }
        if let activity = connectionOptions.userActivities.first,
           activity.activityType == NSUserActivityTypeBrowsingWeb, let url = activity.webpageURL { handle(url) }
    }

    // Custom scheme khi app đang chạy
    func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
        if let url = URLContexts.first?.url { handle(url) }
    }

    // Universal link khi app đang chạy
    func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
        guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
              let url = userActivity.webpageURL else { return }
        handle(url)
    }

    private func handle(_ url: URL) {
        guard let link = DeepLink(url: url) else { return }
        print("route to", link)   // appCoordinator.handle(link)
    }
}
```
**Bẫy:** quên xử lý trường hợp cold start (`connectionOptions`) → link chỉ hoạt động khi app đang chạy. SwiftUI: `.onOpenURL { url in }` xử lý cả hai loại. **Luôn validate** dữ liệu từ deep link (là input từ bên ngoài — đừng để link `reposcope://transfer?amount=...` thực hiện hành động mà không xác nhận).

### §4. Security

**Nguyên tắc nền:** app chạy trên máy của user → **mọi thứ trong app bundle đều đọc được** (strings, plist, asset). Bảo mật mobile là **nâng chi phí tấn công**, không phải tuyệt đối.

**Keychain đúng cách:**
```swift
enum KeychainError: Error { case unexpectedStatus(OSStatus), itemNotFound, invalidData }

struct KeychainStore {
    let service: String     // ví dụ "com.reposcope.auth"

    func save(_ value: String, account: String) throws {
        let data = Data(value.utf8)
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
        ]
        let attributes: [String: Any] = [
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
        ]
        // Update trước; chưa có thì Add — tránh lỗi errSecDuplicateItem (-25299)
        var status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)
        if status == errSecItemNotFound {
            let addQuery = query.merging(attributes) { $1 }
            status = SecItemAdd(addQuery as CFDictionary, nil)
        }
        guard status == errSecSuccess else { throw KeychainError.unexpectedStatus(status) }
    }

    func read(account: String) throws -> String {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
        ]
        var item: CFTypeRef?
        let status = SecItemCopyMatching(query as CFDictionary, &item)
        guard status != errSecItemNotFound else { throw KeychainError.itemNotFound }
        guard status == errSecSuccess else { throw KeychainError.unexpectedStatus(status) }
        guard let data = item as? Data, let value = String(data: data, encoding: .utf8) else {
            throw KeychainError.invalidData
        }
        return value
    }

    func delete(account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
        ]
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```
**`kSecAttrAccessible` — chọn đúng:**
| Giá trị | Đọc được khi | Dùng cho |
|---|---|---|
| `WhenUnlocked` (mặc định) | Máy đang mở khoá | Dữ liệu chỉ dùng khi app foreground |
| `AfterFirstUnlock` | Sau lần mở khoá đầu tiên kể từ khi bật máy | Token cần dùng **ở nền** (background refresh, NSE) |
| `WhenPasscodeSetThisDeviceOnly` | Máy có passcode và đang mở khoá; mất nếu tắt passcode | Dữ liệu rất nhạy cảm |
| Hậu tố `ThisDeviceOnly` | Không đi theo backup/chuyển máy | Token gắn thiết bị |

Bẫy: dùng `WhenUnlocked` cho token rồi gọi API trong background task → đọc Keychain thất bại (`errSecInteractionNotAllowed`) khi máy khoá. Keychain **còn sau khi xoá app** → lần chạy đầu sau cài lại (kiểm tra cờ trong UserDefaults) nên xoá token cũ. Chia sẻ với extension/app khác cùng team qua **Keychain access group**.

**Biometrics (Face ID / Touch ID) — LocalAuthentication:**
```swift
func authenticateWithBiometrics() async -> Bool {
    let context = LAContext()
    var error: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
        return false   // không có Face ID / chưa đăng ký → fallback passcode (.deviceOwnerAuthentication)
    }
    do {
        return try await context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                                                localizedReason: "Mở khoá RepoScope")
    } catch {
        return false   // user huỷ, sai nhiều lần...
    }
}
// Info.plist bắt buộc: NSFaceIDUsageDescription — thiếu là crash khi gọi Face ID
```
⚠️ `evaluatePolicy` chỉ trả **Bool** — trên máy jailbreak có thể bị hook thành `true`. Bảo vệ thật sự: lưu secret trong Keychain với **access control** yêu cầu biometrics — Secure Enclave chỉ giải phóng dữ liệu khi xác thực thành công:
```swift
let access = SecAccessControlCreateWithFlags(nil,
                                             kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                             .biometryCurrentSet,   // đổi vân tay/khuôn mặt → item mất hiệu lực
                                             nil)
// query thêm: kSecAttrAccessControl as String: access!
```

**ATS (App Transport Security):** mặc định iOS **chặn HTTP thường**, bắt buộc HTTPS + TLS 1.2+ + cipher mạnh. Ngoại lệ khai báo trong Info.plist (`NSAppTransportSecurity` → `NSExceptionDomains`). `NSAllowsArbitraryLoads = true` phải giải thích với App Review.

**SSL pinning:** HTTPS chống nghe lén, nhưng nếu kẻ tấn công cài được **CA giả** vào máy (proxy Charles/mitmproxy, máy công ty, malware) thì vẫn đọc được traffic. Pinning = app chỉ tin **certificate/public key cụ thể** của server mình.
```swift
final class PinningDelegate: NSObject, URLSessionDelegate {
    private let pinnedKeyHashes: Set<String>   // base64 SHA-256 của public key (nên có ≥ 2: key hiện tại + key dự phòng)
    init(pinnedKeyHashes: Set<String>) { self.pinnedKeyHashes = pinnedKeyHashes }

    func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge)
        async -> (URLSession.AuthChallengeDisposition, URLCredential?) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let trust = challenge.protectionSpace.serverTrust else {
            return (.performDefaultHandling, nil)
        }
        // 1. Vẫn đánh giá chain chuẩn của hệ thống (hết hạn, sai domain...)
        guard SecTrustEvaluateWithError(trust, nil),
              let chain = SecTrustCopyCertificateChain(trust) as? [SecCertificate],
              let leaf = chain.first,
              let key = SecCertificateCopyKey(leaf),
              let keyData = SecKeyCopyExternalRepresentation(key, nil) as Data? else {
            return (.cancelAuthenticationChallenge, nil)
        }
        // 2. So hash public key với danh sách pin
        let hash = Data(SHA256.hash(data: keyData)).base64EncodedString()
        return pinnedKeyHashes.contains(hash)
            ? (.useCredential, URLCredential(trust: trust))
            : (.cancelAuthenticationChallenge, nil)
    }
}
```
(Ví dụ minh hoạ ý tưởng. Chuẩn công nghiệp hash cả SubjectPublicKeyInfo gồm header ASN.1; từ iOS 14 có thể khai báo pin trong Info.plist bằng `NSPinnedDomains` mà không cần code.)
- **Pin public key** tốt hơn pin certificate: renew certificate giữ nguyên key thì app không gãy.
- **Rủi ro lớn nhất:** server đổi key mà app cũ chưa cập nhật → **toàn bộ user mất kết nối**. Luôn có pin dự phòng, có kế hoạch xoay key, cân nhắc remote config.
- Pinning **không chống được** người dùng máy jailbreak hook vào app (Frida, SSL Kill Switch) — nó chống MITM với user bình thường.

**Lưu secret (API key) trong app:** **không có cách nào an toàn tuyệt đối**. Hardcode, xcconfig, plist, obfuscate đều lấy ra được bằng `strings`/reverse engineering. Cách đúng: secret quan trọng ở **backend** (app gọi backend của bạn, backend gọi dịch vụ thứ 3); key phía client chỉ là key có thể public (giới hạn theo bundle ID, rate limit). Không commit secret vào Git — dùng `.xcconfig` bị `.gitignore` hoặc biến môi trường CI.

**Jailbreak detection:** kiểm tra file như `/Applications/Cydia.app`, thử ghi ra ngoài sandbox, kiểm tra dyld library lạ. **Đều bypass được** — chỉ là một lớp phòng thủ (app ngân hàng dùng để cảnh báo/hạn chế). Kết hợp **App Attest** (DeviceCheck framework) để server xác minh request đến từ app thật trên máy Apple thật.

**Checklist khác:** không log dữ liệu nhạy cảm (`os.Logger` với `privacy: .private`), che màn hình nhạy cảm khi vào app switcher, `isSecureTextEntry` cho ô mật khẩu, file nhạy cảm dùng Data Protection (`.completeFileProtection`), validate input từ deep link/push.

### §5. App performance

**Đo trước, tối ưu sau.** Luôn đo trên **máy thật, build Release** (Debug chậm hơn nhiều, simulator dùng CPU của Mac).

**Launch time** — Apple khuyến nghị frame đầu tiên trong **< 400ms**; quá ~20 giây watchdog kill app.
```
 tap icon ──▶ [ PRE-MAIN ]──────────────────▶ main() ──▶ [ POST-MAIN ]──────────────────────▶ frame đầu tiên
             • dyld load dylib/framework           • UIApplicationMain
             • rebase/bind symbol                  • didFinishLaunching (SDK init, DI, DB...)
             • ObjC runtime setup                  • tạo window, root VC, viewDidLoad
             • +load, static initializer           • layout + render frame đầu
```
| Giai đoạn | Làm chậm | Tối ưu |
|---|---|---|
| Pre-main | Nhiều **dynamic framework** (mỗi cái tốn thời gian load), `+load` ObjC, static initializer C++ | Gộp framework, dùng **static library/merge** (Xcode 15 mergeable libraries), bỏ dependency không cần |
| Post-main | Khởi tạo **mọi SDK** (analytics, ads, crash...) trong `didFinishLaunching` một cách đồng bộ, đọc DB/file lớn trên main, root VC nặng | Lazy init, chuyển việc không cần thiết ra sau frame đầu hoặc ra background, hiện UI cache trước |

**Cold launch** (process chưa có trong RAM — sau reboot hoặc lâu không mở) vs **warm launch** (một phần còn cache) vs **resume** (app đang suspend, không phải launch). Đo bằng: Instruments template **App Launch**, Xcode Organizer → Metrics (dữ liệu thật từ user), `XCTApplicationLaunchMetric` trong performance test, MetricKit.

**Instruments hay dùng:**
| Template | Tìm gì |
|---|---|
| **Time Profiler** | Hàm nào ăn CPU. Mẹo: bật *Invert Call Tree* + *Hide System Libraries* → thấy ngay code của bạn tốn nhất |
| **App Launch** | Từng pha launch, dylib, static init |
| **Hangs** / **Animation Hitches** | Main thread bị chặn ≥ 250ms (hang), frame trễ hạn (hitch) khi scroll |
| **Allocations** / **Leaks** | RAM tăng, object không được giải phóng (tuần 5) |
| **Network** | Request, thời gian, payload |
| **Energy Log** / **Power Profiler** | CPU, mạng, GPS, wakeup tốn pin |
| **SwiftUI** | Body nào được tính lại quá nhiều |

**Đánh dấu vùng code để thấy trong Instruments:**
```swift
let signposter = OSSignposter(subsystem: "com.reposcope", category: "Search")
func measuredSearch() async {
    let state = signposter.beginInterval("search")
    // ... work
    signposter.endInterval("search", state)     // hiện thành khoảng trên timeline (Points of Interest)
}
```

**Hang & hitch:** main thread làm việc nặng (decode JSON lớn, đọc file, `sync` lên queue đang bận, lock) → UI đơ. Xcode 14+ có **hang detection** ngay khi debug, Xcode Organizer → Hangs report từ máy thật. Sửa: đưa việc ra khỏi main (`Task.detached`, actor riêng), tránh `DispatchQueue.main.sync`.

**Energy:** tránh polling (dùng push), gom request mạng, ngừng location khi không cần, timer có `tolerance`, không để animation chạy khi view ẩn.

**App size:** App Store hiển thị **download size** (đã nén) và **install size**. Giảm: asset catalog (App Thinning chỉ tải ảnh đúng độ phân giải máy), nén ảnh / dùng PDF/SF Symbols, bỏ dependency nặng, On-Demand Resources. Đo: Xcode Organizer → Archive → *Distribute* → App Thinning Size Report. Giới hạn tải bằng mạng di động (200MB) ảnh hưởng conversion.

**MetricKit:** `MXMetricManager` gửi báo cáo hàng ngày từ máy user (launch time, hang, CPU, memory, crash diagnostics) — dữ liệu thật thay vì máy dev.

### §6. Crash reporting & symbolication

**Crash log** (`.ips`) gồm: thông tin thiết bị/OS/version app, **Exception Type** (lý do), **thread bị crash** với **backtrace** (chuỗi hàm gọi), danh sách binary image (UUID + địa chỉ load).

**Exception type hay gặp:**
| Exception | Thường do |
|---|---|
| `EXC_BREAKPOINT (SIGTRAP)` | Swift runtime trap: **force unwrap nil**, index out of range, `fatalError`, overflow, cast `as!` sai |
| `EXC_BAD_ACCESS (SIGSEGV/SIGBUS)` | Truy cập vùng nhớ không hợp lệ: object đã giải phóng (`unowned` sai), race condition, C pointer |
| `EXC_CRASH (SIGABRT)` | `abort()`: exception ObjC không bắt (`NSInvalidArgumentException`, `NSInternalInconsistencyException` khi update table view sai số dòng) |
| `EXC_RESOURCE` | Dùng quá CPU/RAM/wakeup |
| Termination code `0x8badf00d` ("ate bad food") | **Watchdog** kill: main thread treo quá lâu khi launch/resume |
| `Jetsam` (không có crash log thông thường) | Hệ thống kill vì **hết RAM** |

**Symbolication:** build Release bị strip tên hàm → backtrace chỉ có địa chỉ:
```
3  RepoScope   0x0000000102a3c1f4  0x102a20000 + 115188
```
**dSYM** (debug symbol file) là file chứa bảng ánh xạ **địa chỉ → tên hàm, file, số dòng**, sinh ra khi build với `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym`. Mỗi build có **UUID** riêng; dSYM phải khớp UUID với binary đã crash. Sau symbolicate:
```
3  RepoScope   0x0000000102a3c1f4  RepoListViewModel.didSelectRow(at:) + 212 (RepoListViewModel.swift:58)
```
- Xcode Organizer (crash từ App Store/TestFlight) tự symbolicate nếu bạn archive trên máy đó (dSYM nằm trong `.xcarchive`).
- **Firebase Crashlytics / Sentry**: phải **upload dSYM** (thêm build phase script, hoặc fastlane `upload_symbols_to_crashlytics`) — quên upload là thấy crash "missing dSYM".
- Thủ công: `atos -arch arm64 -o RepoScope.app.dSYM/Contents/Resources/DWARF/RepoScope -l 0x102a20000 0x102a3c1f4`.
- Kiểm tra UUID: `dwarfdump --uuid RepoScope.app.dSYM`.

**Quy trình xử lý crash:** xem crash-free rate và crash nào ảnh hưởng nhiều user nhất → đọc exception + thread crash → tìm frame **đầu tiên thuộc code của bạn** → xem breadcrumb/log trước khi crash → tái hiện → viết test cho case đó → sửa.

### §7. Build & release

**Target, Scheme, Configuration:**
- **Target**: một sản phẩm build (app, extension, test bundle, framework).
- **Build configuration**: tập build setting — mặc định **Debug** (không tối ưu `-Onone`, có debug symbol, `DEBUG` flag) và **Release** (tối ưu `-O`, strip). Thêm được **Staging**.
- **Scheme**: "công thức" chọn target nào build, chạy Run/Test/Profile/Archive với configuration nào (Run → Debug, Archive → Release), environment variable, launch argument.

Môi trường Dev/Staging/Prod: tạo configuration + scheme cho mỗi môi trường, mỗi cái trỏ tới một **`.xcconfig`**:
```
// Configs/Base.xcconfig
PRODUCT_NAME = RepoScope
SWIFT_VERSION = 6.0

// Configs/Debug.xcconfig
#include "Base.xcconfig"
PRODUCT_BUNDLE_IDENTIFIER = com.reposcope.app.dev
API_BASE_URL = https:/$()/api.github.com
APP_DISPLAY_NAME = RepoScope Dev
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG

// Configs/Release.xcconfig
#include "Base.xcconfig"
PRODUCT_BUNDLE_IDENTIFIER = com.reposcope.app
API_BASE_URL = https:/$()/api.github.com
APP_DISPLAY_NAME = RepoScope
```
- `//` trong xcconfig là comment → URL phải viết `https:/$()/...` (mẹo kinh điển).
- Đưa giá trị vào app: Info.plist thêm key `APIBaseURL = $(API_BASE_URL)`, đọc bằng `Bundle.main.object(forInfoDictionaryKey: "APIBaseURL")`.
- Bundle ID khác nhau → cài **song song** bản Dev và Prod trên cùng máy.
- Ưu điểm so với chỉnh trong UI Xcode: diff được trong Git, ít conflict `project.pbxproj`.
- `#if DEBUG ... #endif` cho code chỉ có ở Debug (menu debug, log chi tiết).

**Code signing — phần nhiều người "nhờ AI setup" nhất:**
```
  Certificate (.cer + private key trong Keychain của Mac)  ── "AI ký"   (Development / Distribution)
  App ID (Bundle ID: com.reposcope.app + capabilities)      ── "ký CÁI GÌ"
  Devices (UDID)                                            ── "chạy trên MÁY NÀO" (chỉ dev/ad hoc)
  Entitlements (push, App Groups, Keychain sharing, ...)     ── "được phép LÀM GÌ"
            │
            ▼
  Provisioning Profile = gói tất cả lại, Apple ký xác nhận
     • Development  : cert dev + App ID + danh sách device       → chạy từ Xcode lên máy test
     • Ad Hoc       : cert distribution + App ID + device list   → phát cho ≤100 máy/loại không qua store
     • App Store    : cert distribution + App ID (không device)  → TestFlight + App Store
     • Enterprise   : tài khoản Enterprise, phát nội bộ công ty
```
- **Certificate** chứng minh danh tính developer; phần quan trọng là **private key** nằm trên máy tạo ra nó — đổi máy mà không export `.p12` thì "mất" cert.
- **Entitlements** phải khớp giữa file `.entitlements` của app và profile; bật Push trong Xcode mà profile cũ chưa có → lỗi ký.
- **Automatic signing** (Xcode tự tạo/quản lý) tiện cho cá nhân; team/CI thường dùng **manual signing** + **fastlane match** (lưu cert/profile mã hoá trong repo Git riêng, mọi máy/CI dùng chung).
- Lỗi hay gặp: *"No profiles for 'com.x' were found"*, *"doesn't include the aps-environment entitlement"*, cert hết hạn (1 năm), device chưa có trong profile dev.

**Version & build number:** `CFBundleShortVersionString` (`MARKETING_VERSION`, ví dụ `1.2.0`, user thấy) và `CFBundleVersion` (`CURRENT_PROJECT_VERSION`, build number, **phải tăng mỗi lần upload** cùng version). CI thường đặt build number = số lần chạy pipeline.

**Quy trình phát hành:**
```
Archive (Release) ──▶ Upload App Store Connect ──▶ xử lý (~15–30') ──▶ TestFlight
   ├─ Internal testers (≤100 người trong team, không cần review, có ngay)
   └─ External testers (≤10.000, cần Beta App Review lần đầu mỗi version, link public)
──▶ Submit for Review (metadata, screenshot, privacy) ──▶ App Review (thường 24–48h)
──▶ Release: thủ công / tự động / Phased Release (7 ngày: 1% → 2% → 5% → 10% → 20% → 50% → 100%)
```
Build TestFlight hết hạn sau **90 ngày**. Phased release cho phép **tạm dừng** khi thấy crash tăng (nhưng user tự cập nhật thủ công vẫn nhận được).

**Lý do bị reject hay gặp:**
| Guideline | Nội dung |
|---|---|
| 2.1 App Completeness | Crash khi review, link chết, placeholder, **thiếu tài khoản demo** cho reviewer |
| 2.3 Accurate Metadata | Screenshot/mô tả không khớp app |
| 4.2 Minimum Functionality | App chỉ là website bọc WebView, quá sơ sài |
| 5.1.1 Privacy / Data Collection | Thiếu privacy policy, xin quyền không giải thích (`NS...UsageDescription` chung chung), bắt đăng nhập khi không cần, **có tạo tài khoản mà không có chức năng xoá tài khoản** |
| 5.1.2 / App Tracking Transparency | Tracking mà không xin ATT |
| 3.1.1 In-App Purchase | Bán nội dung số không qua IAP |
| 4.8 Login Services | Có login bên thứ 3 (Google, Facebook) mà không có lựa chọn tương đương bảo vệ quyền riêng tư (thường là Sign in with Apple) |
| Privacy Manifest | Từ 2024: dùng "required reason API" (UserDefaults, file timestamp...) hoặc SDK phổ biến phải có `PrivacyInfo.xcprivacy` |

Bị reject: đọc kỹ guideline được trích, trả lời trong Resolution Center, sửa hoặc giải thích; có thể kháng nghị.

### §8. CI/CD

**CI (Continuous Integration):** mỗi push/PR → máy chủ tự build + chạy test + lint → báo đỏ/xanh trên PR. Mục đích: phát hiện lỗi sớm, "chạy được trên máy tôi" không còn là cớ, ép test luôn xanh.
**CD (Continuous Delivery/Deployment):** merge vào `main`/tag → tự tăng build number, ký, upload TestFlight, (tuỳ chọn) submit review.

**GitHub Actions** cho RepoScope:
```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

concurrency:                       # PR push liên tục → huỷ lần chạy cũ
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: macos-15
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.4.app
      - name: Cache SPM
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: spm-${{ hashFiles('**/Package.resolved') }}
      - name: Build & Test
        run: |
          set -o pipefail
          xcodebuild test \
            -project RepoScope.xcodeproj \
            -scheme RepoScope \
            -destination 'platform=iOS Simulator,name=iPhone 16' \
            -enableCodeCoverage YES \
            -resultBundlePath TestResults.xcresult \
            | xcbeautify
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: TestResults.xcresult
```
(Tên Xcode/simulator trên runner thay đổi theo thời gian — kiểm tra danh sách phần mềm của image `macos-15` trước khi dùng. `set -o pipefail` để lỗi của `xcodebuild` không bị `xcbeautify` nuốt.)

**fastlane** — công cụ Ruby tự động hoá các việc lặp lại:
```ruby
# fastlane/Fastfile
default_platform(:ios)

platform :ios do
  lane :test do
    run_tests(scheme: "RepoScope", device: "iPhone 16")
  end

  lane :beta do
    setup_ci                                      # tạo keychain tạm trên CI
    match(type: "appstore", readonly: true)       # lấy cert + profile từ repo mã hoá
    increment_build_number(build_number: ENV["GITHUB_RUN_NUMBER"])
    build_app(scheme: "RepoScope", export_method: "app-store")
    upload_to_testflight(skip_waiting_for_build_processing: true)
  end
end
```
Xác thực App Store Connect trên CI bằng **App Store Connect API key** (`.p8`) lưu trong GitHub Secrets, không dùng Apple ID + mật khẩu.

**Xcode Cloud:** CI/CD của Apple tích hợp trong Xcode và App Store Connect — tự quản signing, không cần máy Mac riêng, có giờ miễn phí hàng tháng; kém linh hoạt hơn GitHub Actions + fastlane. Biết các lựa chọn khác: Bitrise, CircleCI, Jenkins tự host.

Pipeline điển hình: `PR → lint (SwiftLint) → build → unit test → (snapshot test) → review → merge → build Release → TestFlight internal → QA → App Store`.

### §9. Git workflow

**Merge vs Rebase:**
```
Ban đầu:        A──B──C        main
                    └──D──E    feature

git merge main  (đứng ở feature):      A──B──C──────┐
                                            └──D──E──M    ← merge commit, giữ nguyên lịch sử thật

git rebase main (đứng ở feature):      A──B──C──D'──E'    ← "chép lại" D, E lên đầu main: lịch sử thẳng,
                                                             nhưng D', E' là commit MỚI (hash khác)
```
| | Merge | Rebase |
|---|---|---|
| Lịch sử | Giữ đúng thực tế, có merge commit | Thẳng, dễ đọc |
| Viết lại lịch sử | Không | **Có** (hash mới) |
| An toàn trên branch chung | ✅ | ❌ **Không rebase branch người khác đang dùng** |
| Conflict | Giải một lần | Có thể phải giải ở **từng commit** |

**Quy tắc vàng:** rebase branch **của riêng bạn** để cập nhật từ `main` và dọn commit trước khi mở PR (`git rebase -i` để squash/fixup); không rebase branch đã push mà người khác dựa vào. Sau rebase branch đã push phải `git push --force-with-lease` (an toàn hơn `--force`: từ chối nếu remote có commit mới bạn chưa thấy). Nhiều team chọn **squash merge** PR: mỗi PR = 1 commit trên main.

**Xử lý conflict:**
```
<<<<<<< HEAD
let timeout = 30
=======
let timeout = 60
>>>>>>> feature/longer-timeout
```
1. `git status` xem file nào conflict. 2. Hiểu **ý định** của cả hai bên (đọc PR/commit message, hỏi người viết) — không chọn bừa. 3. Sửa, xoá marker. 4. **Build + chạy test**. 5. `git add` → `git commit` (merge) hoặc `git rebase --continue`. Muốn bỏ: `git merge --abort` / `git rebase --abort`.

Conflict trong **`project.pbxproj`** là nỗi đau riêng của iOS: giảm bằng cách dùng thư mục đồng bộ của Xcode 16 (*buildable folder*), SPM cho module, XcodeGen/Tuist sinh project từ file YAML/Swift (không commit `.xcodeproj`), sắp xếp file thường xuyên.

**Git flow & các mô hình branch:**
- **Git Flow:** `main` (production), `develop` (tích hợp), `feature/*`, `release/*` (chuẩn bị phát hành, chỉ fix bug), `hotfix/*` (sửa gấp từ main, merge về cả main và develop). Hợp với app mobile có **chu kỳ release** và review của Apple.
- **GitHub Flow:** chỉ `main` + feature branch ngắn hạn + PR. Đơn giản.
- **Trunk-based:** commit nhỏ vào `main` thường xuyên, tính năng chưa xong ẩn sau **feature flag**. Cần CI mạnh. Các công ty lớn hay dùng.

**Lệnh hay hỏi:** `git cherry-pick <hash>` (lấy 1 commit sang branch khác — hay dùng cho hotfix), `git stash` (cất tạm thay đổi), `git reset --soft/--mixed/--hard` (lùi HEAD; `--hard` mất thay đổi chưa commit), `git revert <hash>` (tạo commit đảo ngược — an toàn trên branch chung), `git reflog` (cứu commit "đã mất" sau reset/rebase), `git bisect` (tìm commit gây bug bằng chia đôi).

---

## ⚠️ Bẫy hay gặp

1. **Cache device token vĩnh viễn** — token có thể đổi; gửi lên server mỗi lần launch.
2. **Token sandbox gửi tới APNs production** (hoặc ngược lại) → `BadDeviceToken`.
3. **Dựa vào silent push / BGAppRefresh cho tính năng quan trọng** — không đảm bảo chạy, không chạy nếu user force-quit.
4. **Set `UNUserNotificationCenter.delegate` muộn** → bỏ lỡ sự kiện tap notification khi app mở từ trạng thái killed.
5. **Deep link chỉ xử lý khi app đang chạy**, quên `connectionOptions` lúc cold start.
6. **Custom URL scheme cho dữ liệu nhạy cảm** (OAuth callback) — app khác chiếm scheme được. Dùng universal link / `ASWebAuthenticationSession`.
7. **Keychain `WhenUnlocked` + dùng ở background** → đọc thất bại khi máy khoá.
8. **Nghĩ rằng obfuscate/xcconfig giấu được API key.**
9. **SSL pinning không có pin dự phòng** → đổi cert là cả app chết.
10. **Đo performance trên simulator/Debug** → số liệu vô nghĩa.
11. **Quên upload dSYM** (Bitcode đã bị bỏ từ Xcode 14, nhưng vẫn phải upload dSYM cho Crashlytics/Sentry).
12. **Không tăng build number** → upload bị từ chối.
13. **`git push --force` lên branch chung** → xoá commit của đồng nghiệp. Dùng `--force-with-lease`, và chỉ trên branch của mình.
14. **Commit secret vào Git** — xoá ở commit sau **không đủ**, nó vẫn trong lịch sử; phải **thu hồi (rotate)** key ngay.

---

## 🛠 Bài tập

> Code đặt trong `practice/week17/` hoặc thẳng trong RepoScope. Push/universal link cần tài khoản Apple Developer trả phí để chạy thật — nếu chưa có, làm phần code + dùng **Simulator push**: kéo file `.apns` vào simulator hoặc `xcrun simctl push booted com.reposcope.app payload.apns`.

**Bài 1 — Push (T2)**
a) Vẽ lại sơ đồ APNs không nhìn tài liệu.
b) Trong RepoScope: xin quyền notification, in device token, xử lý `willPresent` và `didReceive` (payload có `"repo": "apple/swift"` → mở màn detail qua Coordinator).
c) Tạo file `payload.apns` và gửi vào simulator bằng `simctl push`.
<details><summary>Gợi ý c)</summary>

```json
{
  "Simulator Target Bundle": "com.reposcope.app",
  "aps": { "alert": { "title": "apple/swift", "body": "Release 6.3" }, "sound": "default" },
  "repo": "apple/swift"
}
```
`xcrun simctl push booted com.reposcope.app payload.apns` (có key `Simulator Target Bundle` thì có thể bỏ bundle ID trong lệnh). Simulator **không** có device token APNs thật — chỉ test được hiển thị và xử lý tap.
</details>

**Bài 2 — Deep link (T3)**
Viết `DeepLink` parser cho 3 dạng: `reposcope://repo/{owner}/{name}`, `https://reposcope.app/repo/{owner}/{name}`, `reposcope://search?q=...`. Viết **6 unit test** (3 hợp lệ, 3 không hợp lệ: thiếu name, host lạ, search thiếu `q`). Thử từ terminal: `xcrun simctl openurl booted "reposcope://repo/apple/swift"`.
<details><summary>Gợi ý</summary>

Bẫy: với `reposcope://repo/apple/swift`, `"repo"` là **host**, `pathComponents` chỉ là `["/", "apple", "swift"]`. Với `https://reposcope.app/repo/apple/swift`, host là domain và `"repo"` nằm trong path. Xem cách ghép `parts` ở §3.
</details>

**Bài 3 — Keychain (T3)**
Viết `KeychainStore` (save/read/delete) không nhìn §4. Test: save 2 lần cùng account không lỗi `-25299`; read sau delete ném `.itemNotFound`. (Chạy trong unit test của app trên simulator — test bundle cần host app để có Keychain.)
Câu hỏi thêm: token GitHub của RepoScope cần đọc trong `BGAppRefreshTask` → chọn `kSecAttrAccessible` nào?
<details><summary>Đáp án câu hỏi thêm</summary>

`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`: đọc được ở nền khi máy đang khoá (sau lần mở khoá đầu tiên từ lúc bật máy), không đi theo backup sang máy khác.
</details>

**Bài 4 — Profiling (T4)**
Chạy RepoScope bằng **Product → Profile** (Release) trên máy thật nếu có:
a) **App Launch**: ghi lại thời gian tới frame đầu tiên. Có SDK nào khởi tạo đồng bộ trong `didFinishLaunching` không?
b) **Time Profiler** khi cuộn list search 30 giây: bật *Invert Call Tree* + *Hide System Libraries*, ghi 3 hàm của bạn tốn CPU nhất.
c) Cố tình thêm `Thread.sleep(forTimeInterval: 0.5)` khi tap cell → xem **Hangs** báo gì, rồi xoá.
d) Thêm `OSSignposter` quanh hàm search, thấy khoảng đó trên timeline.

**Bài 5 — Build config (T5)**
Tạo `Debug.xcconfig`, `Staging.xcconfig`, `Release.xcconfig` cho RepoScope: bundle ID khác nhau (`.dev`, `.staging`, không hậu tố), tên app khác nhau, `API_BASE_URL` đưa vào Info.plist và đọc được trong code. Cài **cả 3 bản cạnh nhau** trên simulator. Giải thích: vì sao URL trong xcconfig phải viết `https:/$()/`?
<details><summary>Đáp án</summary>

Trong `.xcconfig`, `//` bắt đầu comment → `https://api.github.com` bị cắt thành `https:`. `$()` là biến rỗng chèn giữa hai dấu `/` để phá chuỗi `//`.
</details>

**Bài 6 — Git (T6)**
Trong một repo nháp: tạo `main` có 3 commit, tạo `feature` sửa cùng dòng với một commit mới trên `main`.
a) Thử `git merge main` → giải conflict → xem `git log --graph --oneline`.
b) Reset, thử `git rebase main` → giải conflict → `git rebase --continue` → so sánh graph.
c) `git reset --hard HEAD~2` rồi dùng `git reflog` cứu lại commit.
d) Viết 3 câu: khi nào bạn chọn merge, khi nào rebase.

**🏋️ Bài tập lớn (T7) — RepoScope Milestone 17**
(Side project: Milestone 17)
- **GitHub Actions** chạy build + toàn bộ unit test trên mỗi PR và push lên `main`; thêm badge CI vào README
- **Profiling pass**: chạy App Launch + Time Profiler + Allocations, sửa ít nhất 1 vấn đề tìm được, ghi lại trước/sau trong README (mục "Performance")
- **Token GitHub tuỳ chọn** lưu bằng Keychain (màn Settings: nhập/xoá token; có token thì rate limit 5.000 req/giờ thay vì 60)
- **Release config**: xcconfig cho Debug/Release, `#if DEBUG` cho menu debug, version/build number đúng
- (Tuỳ chọn) Deep link `reposcope://repo/{owner}/{name}` đi qua Coordinator
- Archive thử một bản Release thành công (không cần upload)

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Mô tả luồng push notification từ server đến thiết bị.**
<details><summary>Đáp án gợi ý</summary>

App xin quyền hiển thị qua `UNUserNotificationCenter.requestAuthorization`, gọi `registerForRemoteNotifications`; iOS liên hệ APNs và trả device token qua `didRegisterForRemoteNotificationsWithDeviceToken`. App gửi token lên backend, gắn với user. Khi cần gửi, backend gửi HTTP/2 request tới APNs với token, payload JSON (≤4KB) và xác thực bằng JWT từ key .p8. APNs đẩy tới máy; iOS hiển thị hoặc đánh thức app. Foreground thì app quyết định qua `willPresent`, user tap thì vào `didReceive`. Token có thể đổi nên gửi lại mỗi lần launch; có hai môi trường sandbox/production.
</details>

**Q2 ⭐ Silent push là gì? Có đảm bảo được giao không?**
<details><summary>Đáp án gợi ý</summary>

Push có `content-available: 1`, không có alert, `apns-push-type: background`, priority 5 — đánh thức app ở nền khoảng 30 giây qua `didReceiveRemoteNotification` để đồng bộ dữ liệu. Không đảm bảo: hệ thống throttle theo pin/tần suất, không đánh thức app nếu user đã force-quit, nhiều push dồn lại có thể chỉ giao cái cuối. Không dùng cho tính năng bắt buộc.
</details>

**Q3 Notification Service Extension dùng để làm gì?**
<details><summary>Đáp án gợi ý</summary>

Chạy trong process riêng khi push có `mutable-content: 1`, có khoảng 30 giây để sửa nội dung trước khi hiển thị: tải và đính kèm ảnh/video, giải mã nội dung mã hoá end-to-end, đổi title, cập nhật badge. Hết giờ thì `serviceExtensionTimeWillExpire` giao bản tốt nhất hiện có. Chia sẻ dữ liệu với app qua App Group và Keychain access group; RAM rất giới hạn.
</details>

**Q4 ⭐ App có thể chạy ở background bằng những cách nào?**
<details><summary>Đáp án gợi ý</summary>

`beginBackgroundTask` để làm nốt việc ~30 giây khi rời app; BackgroundTasks framework với `BGAppRefreshTask` (cập nhật ngắn, hệ thống chọn thời điểm) và `BGProcessingTask` (việc nặng, thường khi sạc/idle); background `URLSession` cho download/upload chạy ngoài process, sống cả khi app bị kill; silent push; và background mode chuyên biệt như audio, location, VoIP, Bluetooth — phải đúng mục đích kẻo bị reject. Mọi cách đều do hệ thống kiểm soát, app phải xử lý expiration và gọi hoàn thành.
</details>

**Q5 ⭐ Universal link khác custom URL scheme thế nào?**
<details><summary>Đáp án gợi ý</summary>

Custom scheme (`myapp://`) khai báo trong Info.plist, đơn giản nhưng không xác thực — app khác đăng ký cùng scheme được, chưa cài app thì link hỏng. Universal link là link `https` bình thường, cần entitlement Associated Domains và file `apple-app-site-association` trên domain chứng minh quyền sở hữu; chưa cài app thì mở website, an toàn hơn vì chỉ app được domain xác nhận mới mở. Xử lý qua `scene(_:continue:)` với `NSUserActivityTypeBrowsingWeb`, cold start thì lấy trong `connectionOptions`.
</details>

**Q6 ⭐ Lưu access token ở đâu? Vì sao không lưu UserDefaults?**
<details><summary>Đáp án gợi ý</summary>

Keychain: dữ liệu mã hoá bởi hệ thống, kiểm soát được khi nào đọc được (`kSecAttrAccessible`), không đi theo backup nếu dùng `ThisDeviceOnly`, có thể yêu cầu biometrics qua access control. UserDefaults là plist không mã hoá trong sandbox — đọc được qua backup không mã hoá hoặc trên máy jailbreak. Lưu ý Keychain còn lại sau khi xoá app, nên lần chạy đầu sau cài lại cần dọn token cũ.
</details>

**Q7 SSL pinning là gì? Rủi ro?**
<details><summary>Đáp án gợi ý</summary>

App chỉ chấp nhận kết nối nếu certificate hoặc public key của server khớp với giá trị đã nhúng trong app, chống MITM kể cả khi máy đã tin một CA giả (proxy, malware). Implement trong `urlSession(_:didReceive:)` đánh giá trust rồi so hash public key, hoặc `NSPinnedDomains` trong Info.plist. Rủi ro: server đổi key mà app chưa cập nhật thì toàn bộ user mất kết nối — phải pin public key thay vì cert, có pin dự phòng, kế hoạch xoay key. Không chống được kẻ tấn công kiểm soát máy jailbreak.
</details>

**Q8 Làm sao giấu API key trong app?**
<details><summary>Đáp án gợi ý</summary>

Không giấu được tuyệt đối — mọi thứ trong bundle đều trích xuất được (strings, reverse engineering, theo dõi traffic). Cách đúng: secret quan trọng để ở backend, app gọi backend của mình; key bắt buộc phải ở client thì giới hạn quyền (theo bundle ID, rate limit, chỉ đọc), có thể xoay vòng. Obfuscate chỉ nâng độ khó. Không commit key vào Git; dùng xcconfig bị ignore và secret của CI. App Attest giúp backend xác minh request đến từ app thật.
</details>

**Q9 Face ID dùng LocalAuthentication có an toàn không?**
<details><summary>Đáp án gợi ý</summary>

`LAContext.evaluatePolicy` chỉ trả về Bool — đủ cho việc "mở khoá UI", nhưng trên máy jailbreak có thể bị hook thành true. Muốn bảo vệ dữ liệu thật, lưu secret trong Keychain với `SecAccessControl` cờ `.biometryCurrentSet`: Secure Enclave chỉ trả dữ liệu khi xác thực sinh trắc thành công, và vô hiệu nếu đăng ký lại vân tay/khuôn mặt. Cần `NSFaceIDUsageDescription` trong Info.plist và có fallback khi không có biometrics.
</details>

**Q10 ⭐ Làm sao giảm thời gian launch app?**
<details><summary>Đáp án gợi ý</summary>

Đo trước bằng Instruments App Launch và Organizer Metrics trên máy thật, build Release. Pre-main: giảm số dynamic framework (gộp, dùng static/mergeable library), bỏ `+load` và static initializer nặng. Post-main: không khởi tạo mọi SDK đồng bộ trong `didFinishLaunching` — lazy hoặc trì hoãn sau frame đầu; không đọc file/DB lớn hay gọi mạng đồng bộ trên main; root view controller nhẹ, hiện dữ liệu cache trước. Mục tiêu frame đầu dưới ~400ms.
</details>

**Q11 ⭐ App bị giật khi scroll, bạn debug thế nào?**
<details><summary>Đáp án gợi ý</summary>

Tái hiện trên máy thật bản Release, dùng Instruments: Animation Hitches/Hangs để thấy frame bị trễ, Time Profiler (invert call tree, hide system libraries) để thấy hàm nào chiếm main thread. Nguyên nhân thường gặp: decode ảnh lớn trên main, tính layout/attributed string nặng trong `cellForRow`, Auto Layout phức tạp, offscreen rendering (shadow không có `shadowPath`, cornerRadius + masksToBounds), đọc DB đồng bộ. Sửa: downsample ảnh trên background, cache chiều cao/dữ liệu đã format, prefetch, tránh offscreen render. Đo lại sau khi sửa.
</details>

**Q12 ⭐ dSYM là gì? Symbolication là gì?**
<details><summary>Đáp án gợi ý</summary>

Build Release bị strip symbol nên crash log chỉ có địa chỉ bộ nhớ. dSYM là file debug symbol sinh lúc build, ánh xạ địa chỉ sang tên hàm, file, số dòng, gắn với binary bằng UUID. Symbolication là quá trình dùng dSYM khớp UUID để chuyển địa chỉ trong backtrace thành tên hàm đọc được — Xcode Organizer làm tự động nếu có archive; Crashlytics/Sentry cần upload dSYM; thủ công dùng `atos`. Quên upload hoặc UUID không khớp thì không đọc được crash.
</details>

**Q13 Đọc crash log thấy `EXC_BAD_ACCESS` và `EXC_BREAKPOINT` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`EXC_BREAKPOINT`/SIGTRAP thường là Swift runtime cố ý dừng: force unwrap nil, index out of range, `fatalError`, cast `as!` sai, overflow — đọc frame đầu tiên của code mình là thấy dòng lỗi. `EXC_BAD_ACCESS` là truy cập vùng nhớ không hợp lệ: dùng object đã giải phóng (`unowned`, `Unmanaged`), race condition ghi đè dữ liệu, pointer C — khó hơn, dùng Address Sanitizer, Thread Sanitizer, Zombies để tái hiện. Thêm `0x8badf00d` là watchdog kill do main thread treo.
</details>

**Q14 ⭐ Certificate, provisioning profile, entitlements là gì?**
<details><summary>Đáp án gợi ý</summary>

Certificate chứng minh danh tính người ký (cặp khoá, private key nằm trong Keychain máy Mac) — loại development hoặc distribution. App ID là bundle identifier cùng capability. Entitlements là các quyền app yêu cầu (push, App Groups, Keychain sharing, Associated Domains). Provisioning profile gói certificate + App ID + entitlements (+ danh sách device với development/ad hoc) được Apple ký, cho biết app này do ai ký được chạy ở đâu với quyền gì. Loại: Development, Ad Hoc, App Store, Enterprise. Team thường dùng fastlane match để đồng bộ cert/profile.
</details>

**Q15 Scheme, target, build configuration khác nhau thế nào? Quản lý nhiều môi trường ra sao?**
<details><summary>Đáp án gợi ý</summary>

Target là một sản phẩm build (app, extension, test). Build configuration là tập build setting (Debug, Release, thêm Staging). Scheme chọn target nào và dùng configuration nào cho từng hành động Run/Test/Profile/Archive. Nhiều môi trường: mỗi môi trường một configuration + scheme + `.xcconfig` khai bundle ID, tên app, base URL; giá trị đưa vào Info.plist rồi đọc từ `Bundle.main`; `#if DEBUG` cho code chỉ có ở debug. Bundle ID khác nhau để cài song song.
</details>

**Q16 TestFlight hoạt động thế nào? Phased release là gì?**
<details><summary>Đáp án gợi ý</summary>

Upload build lên App Store Connect, sau khi xử lý có thể phát cho internal tester (thành viên team, tối đa 100, không cần review) và external tester (tối đa 10.000, cần Beta App Review). Build hết hạn sau 90 ngày, build number phải tăng mỗi lần upload. Phased release phát bản cập nhật tự động dần trong 7 ngày (1% → 100%) cho user bật auto-update, có thể tạm dừng nếu thấy crash; người cập nhật thủ công vẫn nhận ngay.
</details>

**Q17 Kể vài lý do App Store hay reject.**
<details><summary>Đáp án gợi ý</summary>

Crash hoặc bug khi review, thiếu tài khoản demo (2.1); metadata/screenshot không khớp (2.3); app quá sơ sài hoặc chỉ là WebView (4.2); vi phạm quyền riêng tư: thiếu privacy policy, mô tả xin quyền chung chung, không có chức năng xoá tài khoản, tracking không qua ATT (5.1); bán nội dung số không qua In-App Purchase (3.1.1); thiếu privacy manifest cho required-reason API. Xử lý: đọc guideline được trích, trả lời trong Resolution Center, sửa hoặc giải thích.
</details>

**Q18 ⭐ Bạn đã setup CI/CD cho iOS thế nào?**
<details><summary>Đáp án gợi ý</summary>

Mỗi PR: GitHub Actions trên runner macOS chạy SwiftLint, `xcodebuild test` trên simulator, xuất kết quả và coverage, PR chỉ merge khi xanh. Merge vào main hoặc tạo tag: fastlane lane `beta` — `match` lấy cert/profile, tăng build number theo số lần chạy CI, `build_app`, `upload_to_testflight`, xác thực bằng App Store Connect API key trong secrets. Cache SPM để build nhanh, huỷ run cũ khi có push mới. Lựa chọn khác là Xcode Cloud (Apple quản signing) hoặc Bitrise.
</details>

**Q19 ⭐ Rebase và merge khác nhau thế nào? Khi nào dùng cái nào?**
<details><summary>Đáp án gợi ý</summary>

Merge tạo merge commit nối hai nhánh, giữ nguyên lịch sử, không viết lại commit — an toàn với branch chung. Rebase chép lại các commit của branch mình lên đầu branch đích, lịch sử thẳng và sạch nhưng tạo commit mới (hash khác) — chỉ dùng trên branch riêng chưa ai dựa vào; sau đó push bằng `--force-with-lease`. Tôi rebase feature branch của mình lên main để cập nhật và dọn commit trước khi mở PR, còn tích hợp vào main thì merge hoặc squash merge theo quy ước team.
</details>

**Q20 Bạn xử lý merge conflict thế nào? Conflict trong `project.pbxproj` thì sao?**
<details><summary>Đáp án gợi ý</summary>

Xem file conflict bằng `git status`, đọc cả hai phía và hiểu ý định (đọc commit/PR, hỏi người viết nếu cần) chứ không chọn bừa một bên, sửa và xoá marker, build và chạy test, rồi `add` + `commit` hoặc `rebase --continue`; sai thì `--abort`. Với `project.pbxproj`: thường giữ cả hai phía khi chỉ là thêm file, mở lại Xcode kiểm tra project hợp lệ. Giảm từ gốc bằng buildable folder của Xcode 16, chia module SPM, hoặc sinh project bằng XcodeGen/Tuist; merge main thường xuyên để conflict nhỏ.
</details>

---

## 🏁 M5 self-test (Chủ Nhật 07/03)

> Mốc M5 trong README: **"Side project chạy được: network + cache + test + kiến trúc rõ ràng, đẩy lên GitHub."** Tự chấm trung thực. Đạt **≥ 12/15** mới coi là qua M5. Chưa đạt: dùng 2–3 ngày đầu tuần 18 để bù **trước khi** làm mock interview.

**A. Side project RepoScope (mở repo GitHub ra kiểm tra từng ý)**
- [ ] 1. Clone repo về máy sạch → build và chạy được ngay, không cần sửa gì (README có hướng dẫn)
- [ ] 2. Kiến trúc rõ: thư mục/module Domain · Data · Presentation; README có sơ đồ
- [ ] 3. Network qua `APIClient` + `Endpoint` của bạn, có map lỗi, xử lý rate limit
- [ ] 4. Có cache/offline: mất mạng vẫn thấy dữ liệu lần trước; favorites lưu bằng SwiftData/Core Data
- [ ] 5. ≥ 15 unit test xanh, có test network qua `URLProtocol` stub
- [ ] 6. GitHub Actions chạy test trên mỗi PR, badge xanh trên README
- [ ] 7. Không có secret/token nào trong lịch sử Git; token người dùng lưu Keychain

**B. Nói không cần tài liệu (bật đồng hồ, mỗi câu ≤ 2 phút, ghi âm)**
- [ ] 8. "Giới thiệu kiến trúc RepoScope và vì sao bạn chọn nó" — có trade-off
- [ ] 9. "Network layer của bạn xử lý 401 khi 3 request chạy song song thế nào?"
- [ ] 10. "Bạn test ViewModel và APIClient thế nào?"
- [ ] 11. "Core Data/SwiftData threading" — nêu được ít nhất 3 quy tắc
- [ ] 12. "Luồng push notification từ server tới máy"

**C. Viết code trên giấy / editor trắng (không autocomplete, ≤ 20 phút mỗi bài)**
- [ ] 13. `Endpoint<Response>` + `APIClient.send` có kiểm tra status code và decode
- [ ] 14. `actor` refresh token không refresh trùng
- [ ] 15. Một ViewModel `@MainActor` + mock service + 2 unit test

**Sau M5:** cập nhật CV (thêm RepoScope với 3 gạch đầu dòng có số liệu: số test, coverage, tính năng), ghim repo trên GitHub profile, và **tiếp tục gửi CV** — mục tiêu cuối tuần 17: ≥ 10 CV đã gửi.

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–6 không xem lời giải trước
- [ ] Bài tập lớn: CI xanh, profiling pass có ghi chép, token trong Keychain, release config
- [ ] Trả lời được ≥ 16/20 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Luồng push notification" và "Certificate vs provisioning profile"
- [ ] **M5 self-test đạt ≥ 12/15**
- [ ] Đã gửi ≥ 10 CV, có bảng theo dõi (công ty · ngày gửi · vị trí · trạng thái)
- [ ] Làm xong 6 bài DSA tuần 17 (Intervals + Greedy)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [Apple — Registering your app with APNs](https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns) · [Pushing background updates to your app](https://developer.apple.com/documentation/usernotifications/pushing-background-updates-to-your-app)
- [Apple — Choosing background strategies for your app](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)
- [Apple — Supporting universal links in your app](https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app)
- [Apple — Keychain services](https://developer.apple.com/documentation/security/keychain-services) · [Identity Pinning (NSPinnedDomains)](https://developer.apple.com/news/?id=g9ejcf8y)
- [Apple — Reducing your app's launch time](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time) · WWDC22 *Track down hangs with Xcode and on-device detection*
- [Apple — Diagnosing issues using crash reports and device logs](https://developer.apple.com/documentation/xcode/diagnosing-issues-using-crash-reports-and-device-logs) · [Adding identifiable symbol names to a crash report](https://developer.apple.com/documentation/xcode/adding-identifiable-symbol-names-to-a-crash-report)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [fastlane docs](https://docs.fastlane.tools) · [GitHub Actions — runner images](https://github.com/actions/runner-images)
- [Pro Git book — Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
