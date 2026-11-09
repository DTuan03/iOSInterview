# Tuần 16 — Networking · Persistence · Unit test & mock (22/02 – 28/02/2027)

> **Giai đoạn 4 · Kiến trúc & Hệ thống** · Thời lượng: ~15 giờ
> Ba chủ đề "ngày nào cũng đụng" ở công việc thật. Người phỏng vấn mức 2 năm sẽ hỏi: *"Thiết kế network layer của bạn thế nào?"*, *"Core Data threading ra sao?"*, *"Bạn test code async/network thế nào?"*. Tuần này nặng — nếu thiếu giờ, **ưu tiên §3 (network layer), §8 (Core Data), §11–§13 (testing)**.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích HTTP method, status code, header quan trọng, **idempotency**, REST
- [ ] Biết các loại `URLSessionConfiguration` và task (data/download/upload/background)
- [ ] **Tự viết từ đầu** một network layer generic: `Endpoint`, `APIClient`, map lỗi, decode, retry, refresh token bằng `actor`
- [ ] Giải thích cache HTTP: `URLCache`, `Cache-Control`, `ETag`/`304`
- [ ] Làm pagination không bị gọi trùng trang; image loading có cache + chống request trùng
- [ ] Biết lưu gì ở đâu: UserDefaults, file (Documents/Caches/Application Support/tmp), Keychain, DB
- [ ] Vẽ Core Data stack, nói được **quy tắc threading** và `NSFetchedResultsController`
- [ ] Dùng SwiftData (`@Model`, `ModelContainer`, `@Query`) và so sánh với Core Data, Realm, GRDB
- [ ] Viết unit test bằng **XCTest và Swift Testing**, mock qua protocol, test async, stub network bằng `URLProtocol`
- [ ] Làm xong 6 bài DSA tuần 16 — Dynamic Programming 1D (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 22/02** | 2h | §1 HTTP cơ bản · §2 URLSession. Bài tập 1 · DSA: Climbing Stairs | `http-notes.md` (bảng status code tự viết) |
| **T3 23/02** | 2h | §3 **Network layer generic** (đọc kỹ nhất tuần) — gõ lại toàn bộ. Bài tập 2 · DSA: House Robber | `network.swift` chạy được |
| **T4 24/02** | 2h | §4 Cache · §5 Pagination · §6 Image loading. Bài tập 3 · DSA: Coin Change | `paginator.swift` |
| **T5 25/02** | 2h | §7 UserDefaults, FileManager, Keychain · §8 **Core Data** · §9 SwiftData · §10 So sánh & offline-first. Bài tập 4 · DSA: Decode Ways | Sơ đồ Core Data stack vẽ tay |
| **T6 26/02** | 2h | §11 XCTest & Swift Testing · §12 Mock, async test · §13 URLProtocol stub · §14 UI/snapshot test, coverage. Bài tập 5, 6 · DSA: Word Break | Package test chạy `swift test` xanh |
| **T7 27/02** | 4h | Bài tập lớn. Side project: Milestone 16 (favorites bằng SwiftData, offline cache, ≥ 15 unit test có network stub) · DSA: Longest Increasing Subsequence | RepoScope có test xanh |
| **CN 28/02** | 2h | Tự trả lời 20 câu phỏng vấn. Ghi âm 2 câu ⭐. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. HTTP cơ bản

Một request HTTP gồm: **method**, **URL**, **headers**, **body** (tuỳ chọn). Response gồm: **status code**, **headers**, **body**.

```
GET /search/repositories?q=swift&page=1 HTTP/1.1
Host: api.github.com
Accept: application/vnd.github+json
Authorization: Bearer ghp_xxx

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: private, max-age=60
ETag: "a1b2c3"
X-RateLimit-Remaining: 9

{"total_count": 1234, "items": [...]}
```

**Methods & idempotency**
| Method | Dùng để | Safe (không đổi dữ liệu server) | Idempotent (gọi N lần = gọi 1 lần) |
|---|---|---|---|
| GET | Đọc | ✅ | ✅ |
| POST | Tạo mới / hành động | ❌ | ❌ (gọi 2 lần → 2 đơn hàng!) |
| PUT | Thay **toàn bộ** resource | ❌ | ✅ |
| PATCH | Sửa **một phần** | ❌ | ❌ (không đảm bảo) |
| DELETE | Xoá | ❌ | ✅ (xoá lần 2 vẫn là "đã xoá", dù có thể trả 404) |

**Vì sao idempotency quan trọng với mobile?** Mạng di động chập chờn → request timeout nhưng server **có thể đã xử lý**. Retry GET/PUT/DELETE an toàn; retry POST có thể tạo trùng. Giải pháp: server hỗ trợ header `Idempotency-Key: <uuid>` — client gửi cùng key khi retry, server nhận ra và không xử lý lại (Stripe làm vậy).

**Status code cần thuộc**
| Nhóm | Code | Ý nghĩa | App nên làm gì |
|---|---|---|---|
| 2xx | 200 OK, 201 Created, 204 No Content | Thành công | 204: không decode body |
| 3xx | 301/302 Redirect, **304 Not Modified** | 304: dùng cache | URLSession tự theo redirect |
| 4xx | 400 Bad Request | Request sai | Hiện lỗi, không retry |
| | **401 Unauthorized** | Chưa xác thực / token hết hạn | Refresh token rồi thử lại 1 lần, thất bại → logout |
| | 403 Forbidden | Không có quyền (GitHub: cả rate limit) | Không retry ngay |
| | 404 Not Found | Không tồn tại | Không retry |
| | 409 Conflict, 422 Unprocessable | Xung đột / validation | Hiện lỗi từ body |
| | **429 Too Many Requests** | Rate limit | Đợi theo `Retry-After` |
| 5xx | 500, 502, 503, 504 | Lỗi server / gateway | Retry có backoff |

**Header hay gặp:** `Content-Type` (body là gì), `Accept` (muốn nhận gì), `Authorization: Bearer <token>`, `Cache-Control`, `ETag` / `If-None-Match`, `Retry-After`, `User-Agent`.

**REST** (Representational State Transfer): API xoay quanh **resource** (danh từ, `/users/42/repos`), dùng HTTP method làm động từ, **stateless** (mỗi request tự mang đủ thông tin, ví dụ token), response cache được. Khác với GraphQL (1 endpoint, client chọn field) — biết tên là đủ.

### §2. URLSession

```
URLSessionConfiguration ──▶ URLSession ──▶ URLSessionTask (data / download / upload / stream / websocket)
   (timeout, cache,           (dùng lại,        (một request cụ thể)
    headers, protocol)         đừng tạo mới mỗi request)
```

**3 loại configuration:**
| Config | Đặc điểm | Dùng khi |
|---|---|---|
| `.default` | Cache trên disk, cookie, credential lưu lại | Hầu hết trường hợp |
| `.ephemeral` | Mọi thứ chỉ trong RAM, không lưu gì xuống disk | Chế độ ẩn danh, **unit test** |
| `.background(withIdentifier:)` | Hệ thống tiếp tục tải/đẩy file **khi app bị suspend hoặc bị kill** | Tải file lớn, upload video (tuần 17) |

```swift
let config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 30          // thời gian chờ giữa các gói dữ liệu
config.waitsForConnectivity = true             // đợi có mạng thay vì fail ngay
config.urlCache = URLCache(memoryCapacity: 20 * 1024 * 1024, diskCapacity: 100 * 1024 * 1024)
config.requestCachePolicy = .useProtocolCachePolicy
let session = URLSession(configuration: config)
```

**Các loại task (API async, iOS 15+):**
```swift
let (data, response) = try await session.data(for: request)            // data task: JSON API
let (fileURL, _) = try await session.download(from: url)                // download task: ghi ra file tạm
// ⚠️ fileURL bị xoá sau khi hàm trả về / app thoát → phải move sang thư mục của bạn ngay
let (_, _) = try await session.upload(for: request, from: bodyData)     // upload task
let (bytes, _) = try await session.bytes(from: url)                     // stream từng byte (server-sent events)
for try await line in bytes.lines { print(line) }
```
API cũ dùng completion handler: `session.dataTask(with:) { data, response, error in }.resume()` — **quên `.resume()` là lỗi kinh điển**, task không bao giờ chạy. Completion chạy trên **background queue** → cập nhật UI phải về main.

**Lưu ý:** `URLSession` **giữ strong delegate** đến khi `invalidateAndCancel()`/`finishTasksAndInvalidate()` → session tạo với delegate mà không invalidate là leak. Dùng một session sống suốt app.

`URLError` cho lỗi tầng mạng: `.notConnectedToInternet`, `.timedOut`, `.networkConnectionLost`, `.cancelled`... **4xx/5xx KHÔNG phải error** với URLSession — `data(for:)` vẫn trả thành công, bạn phải tự kiểm tra `statusCode`.

### §3. Network layer generic — phần quan trọng nhất tuần

Mục tiêu: ViewModel/Repository chỉ cần viết `try await client.send(.searchRepos(query: "swift", page: 1))`, mọi thứ còn lại (URL, header, token, lỗi, decode, retry) nằm ở một chỗ.

```
 Repository ──send(Endpoint<T>)──▶ APIClient ──▶ URLSession
                                     │ 1. Endpoint → URLRequest
                                     │ 2. gắn token (AuthManager actor)
                                     │ 3. gửi, kiểm tra status code → APIError
                                     │ 4. 401 → refresh token 1 lần, gửi lại
                                     │ 5. lỗi tạm thời → retry backoff
                                     │ 6. decode JSON → T
```

**Bước 1 — Endpoint mang luôn kiểu Response (phantom generic):**
```swift
enum HTTPMethod: String { case get = "GET", post = "POST", put = "PUT", patch = "PATCH", delete = "DELETE" }

struct Endpoint<Response: Decodable> {
    var path: String
    var method: HTTPMethod = .get
    var query: [URLQueryItem] = []
    var body: Data? = nil
    var headers: [String: String] = [:]

    func makeRequest(baseURL: URL) throws -> URLRequest {
        guard var components = URLComponents(url: baseURL.appendingPathComponent(path),
                                             resolvingAgainstBaseURL: false) else {
            throw APIError.invalidURL
        }
        if !query.isEmpty { components.queryItems = query }   // URLComponents tự percent-encode
        guard let url = components.url else { throw APIError.invalidURL }
        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.httpBody = body
        headers.forEach { request.setValue($1, forHTTPHeaderField: $0) }
        return request
    }
}

struct SearchResponse: Decodable { let totalCount: Int; let items: [RepoDTO] }
struct RepoDTO: Decodable, Equatable { let id: Int; let fullName: String; let stargazersCount: Int }

extension Endpoint where Response == SearchResponse {
    static func searchRepos(query: String, page: Int, perPage: Int = 30) -> Endpoint {
        Endpoint(path: "search/repositories",
                 query: [URLQueryItem(name: "q", value: query),
                         URLQueryItem(name: "page", value: String(page)),
                         URLQueryItem(name: "per_page", value: String(perPage))])
    }
}

let req = try Endpoint<SearchResponse>.searchRepos(query: "swift ui", page: 2)
    .makeRequest(baseURL: URL(string: "https://api.github.com")!)
print(req.url!.absoluteString)
// https://api.github.com/search/repositories?q=swift%20ui&page=2&per_page=30
```
Nhờ `Endpoint<SearchResponse>`, gọi `client.send(.searchRepos(...))` **tự suy ra kiểu trả về** — không thể decode nhầm kiểu.

**Bước 2 — Lỗi có nghĩa với app, không phải lỗi thô:**
```swift
enum APIError: Error, Equatable {
    case invalidURL
    case transport(URLError.Code)         // mất mạng, timeout
    case unauthorized                     // 401
    case notFound                         // 404
    case rateLimited(retryAfter: TimeInterval?)
    case server(statusCode: Int)          // 5xx
    case http(statusCode: Int)            // 4xx khác
    case decoding(String)

    var isRetryable: Bool {
        switch self {
        case .transport(let code): return [.timedOut, .networkConnectionLost, .notConnectedToInternet].contains(code)
        case .server, .rateLimited: return true
        default: return false             // 400, 404, decode lỗi: thử lại cũng vô ích
        }
    }
}
```
ViewModel chỉ cần `switch` trên `APIError` để hiện thông báo phù hợp ("Kiểm tra kết nối mạng" vs "Không tìm thấy").

**Bước 3 — Refresh token bằng actor, tránh refresh trùng:**

Vấn đề: màn hình gọi 3 API song song, token hết hạn → cả 3 nhận 401 → **3 lần refresh** (server có thể vô hiệu hoá refresh token sau lần dùng đầu → 2 lần sau fail → user bị logout oan).

Giải pháp: `actor` giữ **một `Task` refresh đang chạy**; request nào đến sau thì `await` chung task đó.
```swift
actor AuthManager: TokenStore {
    private var token: String?
    private var refreshTask: Task<String, Error>?
    private(set) var refreshCount = 0
    private let refresher: @Sendable () async throws -> String

    init(token: String?, refresher: @escaping @Sendable () async throws -> String) {
        self.token = token
        self.refresher = refresher
    }

    func currentToken() -> String? { token }

    func refreshToken() async throws -> String {
        if let refreshTask { return try await refreshTask.value }   // đang refresh → chờ chung
        let task = Task { try await refresher() }
        refreshTask = task                                           // gán TRƯỚC khi await
        refreshCount += 1
        defer { refreshTask = nil }
        let newToken = try await task.value                          // ← actor reentrancy: request khác
        token = newToken                                             //   vào đây sẽ thấy refreshTask != nil
        return newToken
    }
}
```
Điểm mấu chốt: **actor reentrancy** (tuần 7) — trong lúc `await task.value`, actor nhận các lời gọi khác. Vì `refreshTask` đã được gán **trước** `await`, các lời gọi đó thấy task và chờ chung. Nếu bạn viết `token = try await refresher()` trực tiếp thì mỗi lời gọi lại tự refresh → vẫn trùng.

**Bước 4 — APIClient:**
```swift
protocol APIClientProtocol: Sendable {
    func send<R: Decodable>(_ endpoint: Endpoint<R>) async throws -> R
}

final class APIClient: APIClientProtocol, Sendable {
    private let baseURL: URL
    private let session: URLSession
    private let auth: AuthManager?
    private let maxRetries: Int

    init(baseURL: URL, session: URLSession = .shared, auth: AuthManager? = nil, maxRetries: Int = 2) {
        self.baseURL = baseURL
        self.session = session
        self.auth = auth
        self.maxRetries = maxRetries
    }

    func send<R: Decodable>(_ endpoint: Endpoint<R>) async throws -> R {
        var attempt = 0
        var didRefresh = false
        while true {
            do {
                var request = try endpoint.makeRequest(baseURL: baseURL)
                if let token = await auth?.currentToken() {
                    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
                }
                return try await perform(request)
            } catch APIError.unauthorized where auth != nil && !didRefresh {
                didRefresh = true                                    // chỉ refresh 1 lần, tránh vòng lặp vô hạn
                _ = try await auth?.refreshToken()
            } catch let error as APIError where error.isRetryable && attempt < maxRetries {
                attempt += 1
                let delay = 0.2 * pow(2, Double(attempt - 1))       // exponential backoff: 0.2s, 0.4s...
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }
    }

    private func perform<R: Decodable>(_ request: URLRequest) async throws -> R {
        let data: Data, response: URLResponse
        do {
            (data, response) = try await session.data(for: request)
        } catch let error as URLError {
            throw APIError.transport(error.code)
        }
        guard let http = response as? HTTPURLResponse else { throw APIError.http(statusCode: -1) }
        switch http.statusCode {
        case 200..<300: break
        case 401: throw APIError.unauthorized
        case 404: throw APIError.notFound
        case 403, 429:
            let retryAfter = http.value(forHTTPHeaderField: "Retry-After").flatMap(TimeInterval.init)
            throw APIError.rateLimited(retryAfter: retryAfter)
        case 500..<600: throw APIError.server(statusCode: http.statusCode)
        default: throw APIError.http(statusCode: http.statusCode)
        }
        do {
            let decoder = JSONDecoder()
            decoder.keyDecodingStrategy = .convertFromSnakeCase     // full_name → fullName
            return try decoder.decode(R.self, from: data)
        } catch {
            throw APIError.decoding(String(describing: error))
        }
    }
}
```
Chạy thử với URLProtocol stub (§13) cho kết quả:
```
apple/swift                 ← decode thành công
notFound                    ← 404 map đúng
calls: 3                    ← 500, 500, rồi 200: retry 2 lần
refreshCount: 1             ← 3 request song song cùng nhận 401 nhưng chỉ refresh 1 lần
decoding error
```

**Những thứ production thêm vào:** retry chỉ với method idempotent (hoặc có `Idempotency-Key`), tôn trọng `Retry-After`, jitter cho backoff (tránh mọi client retry cùng lúc), logging bằng `os.Logger` (ẩn token), interceptor/middleware (thêm header chung, đo thời gian), huỷ request khi màn hình đóng (`Task` bị cancel → `URLError.cancelled`).

**Có nên dùng Alamofire?** Alamofire cho sẵn retry/interceptor/multipart/reachability. Với async/await hiện nay, `URLSession` + 200 dòng như trên là đủ cho đa số app — ít phụ thuộc, hiểu rõ từng dòng. Phỏng vấn thường muốn thấy bạn **tự viết được**.

### §4. Caching HTTP

**`URLCache`** — URLSession tự cache response theo header của server khi `requestCachePolicy = .useProtocolCachePolicy`:
- `Cache-Control: max-age=60` → trong 60s, request lặp lại lấy từ cache, **không lên mạng**.
- `Cache-Control: no-store` → không cache. `no-cache` → cache nhưng phải hỏi lại server trước khi dùng.
- Response lớn hơn ~5% dung lượng disk cache sẽ không được cache — đặt `diskCapacity` đủ lớn.

**ETag / conditional request** — tiết kiệm băng thông:
```
Lần 1:  GET /repos/apple/swift          → 200 OK, ETag: "abc", body 20KB
Lần 2:  GET /repos/apple/swift
        If-None-Match: "abc"           → 304 Not Modified, body RỖNG → dùng bản đã lưu
```
URLCache tự làm việc này khi server gửi ETag. Tự làm thủ công (ví dụ cache lưu trong DB của bạn):
```swift
actor ETagCache {
    private var store: [URL: (etag: String, data: Data)] = [:]
    func fetch(_ url: URL, session: URLSession) async throws -> Data {
        var request = URLRequest(url: url)
        request.cachePolicy = .reloadIgnoringLocalCacheData   // tự quản, không để URLCache xen vào
        if let cached = store[url] { request.setValue(cached.etag, forHTTPHeaderField: "If-None-Match") }
        let (data, response) = try await session.data(for: request)
        let http = response as! HTTPURLResponse
        if http.statusCode == 304, let cached = store[url] { return cached.data }
        if let etag = http.value(forHTTPHeaderField: "ETag") { store[url] = (etag, data) }
        return data
    }
}
```
GitHub API: request có điều kiện trả 304 **không bị tính vào rate limit** — rất đáng làm cho RepoScope.

**Các tầng cache trong app:** memory (`NSCache`) → disk (`URLCache`, file, DB) → network. Cache HTTP phù hợp dữ liệu "đọc"; dữ liệu cần query/sửa offline thì lưu DB (§10).

### §5. Pagination

Hai kiểu: **offset/page** (`?page=2&per_page=30` — đơn giản nhưng trùng/sót khi dữ liệu thay đổi giữa các trang) và **cursor** (`?after=abc123` — server trả cursor trang sau, ổn định hơn; GitHub GraphQL, Twitter dùng). GitHub REST còn trả header `Link: <...page=3>; rel="next"`.

Vấn đề phỏng vấn hay hỏi: **scroll nhanh → gọi trùng trang 2 nhiều lần**. Giải pháp: cờ `isLoading` + `hasMore`, đặt trong actor (hoặc `@MainActor` VM):
```swift
actor RepoPaginator {
    private let client: APIClientProtocol
    private let query: String
    private var nextPage = 1
    private var isLoading = false
    private(set) var items: [RepoDTO] = []
    private(set) var hasMore = true

    init(client: APIClientProtocol, query: String) { self.client = client; self.query = query }

    func loadNextPage() async throws {
        guard hasMore, !isLoading else { return }      // chống gọi trùng
        isLoading = true
        defer { isLoading = false }
        let res = try await client.send(.searchRepos(query: query, page: nextPage, perPage: 2))
        items += res.items
        hasMore = items.count < res.totalCount && !res.items.isEmpty
        nextPage += 1                                  // chỉ tăng khi THÀNH CÔNG → lỗi thì thử lại đúng trang
    }
}
// 3 lần loadNextPage với total_count = 3, perPage = 2 → items: ["a", "b", "c"], hasMore: false
```
Trigger: trong `collectionView(_:willDisplay:forItemAt:)` (hoặc `onAppear` của item cuối trong SwiftUI) khi `indexPath.item >= items.count - 5` → prefetch sớm. Đổi query mới → huỷ task cũ, reset paginator.

### §6. Image loading

Yêu cầu: không tải lại ảnh đã có, không tải trùng khi 2 cell cùng URL, không hiển thị **ảnh sai** khi cell bị reuse.
```swift
actor ImageLoader {
    static let shared = ImageLoader()
    private let cache = NSCache<NSURL, UIImage>()            // tự giải phóng khi thiếu RAM
    private var inFlight: [URL: Task<UIImage, Error>] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let cached = cache.object(forKey: url as NSURL) { return cached }
        if let task = inFlight[url] { return try await task.value }   // đang tải → chờ chung
        let task = Task<UIImage, Error> {
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else { throw URLError(.cannotDecodeContentData) }
            return image
        }
        inFlight[url] = task
        defer { inFlight[url] = nil }
        let image = try await task.value
        cache.setObject(image, forKey: url as NSURL)
        return image
    }
}
```
Trong cell: lưu `Task` tải ảnh, **huỷ trong `prepareForReuse()`** và set `imageView.image = nil`; khi ảnh về kiểm tra cell vẫn đang hiển thị đúng URL. Ảnh lớn: **downsample** bằng `CGImageSourceCreateThumbnailAtIndex` về đúng kích thước hiển thị (ảnh 4000×3000 decode ra ~48MB RAM). Thực tế: Kingfisher, SDWebImage, Nuke làm sẵn tất cả. SwiftUI: `AsyncImage` **không cache** ngoài URLCache → list dài nên dùng loader riêng.

### §7. UserDefaults, FileManager, Keychain

**UserDefaults** — plist key–value, đọc **toàn bộ vào RAM** khi app khởi động.
```swift
UserDefaults.standard.set(true, forKey: "hasSeenOnboarding")
let seen = UserDefaults.standard.bool(forKey: "hasSeenOnboarding")   // không có key → false
// SwiftUI: @AppStorage("preferredSort") private var preferredSort = "stars"
```
| Nên lưu | KHÔNG lưu |
|---|---|
| Cờ (đã xem onboarding), setting nhỏ (theme, sort) | **Token, mật khẩu, thông tin nhạy cảm** — file plist không mã hoá, đọc được từ backup/thiết bị jailbreak |
| | Dữ liệu lớn (danh sách, ảnh, JSON response) — làm chậm launch, tốn RAM |
| | Dữ liệu cần query, quan hệ |

**FileManager & sandbox** — mỗi app có một "hộp" riêng:
| Thư mục | Dùng cho | Backup iCloud | Hệ thống tự xoá? |
|---|---|---|---|
| `Documents/` | Dữ liệu **user tạo** (file export, tài liệu) — có thể hiện trong app Files | ✅ | ❌ |
| `Library/Application Support/` | Dữ liệu app cần, user không thấy (DB SQLite, Core Data store) | ✅ | ❌ |
| `Library/Caches/` | Dữ liệu **tải lại được** (ảnh, JSON cache) | ❌ | ✅ khi thiếu dung lượng |
| `tmp/` | File tạm | ❌ | ✅ bất cứ lúc nào |

```swift
let caches = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
let fileURL = caches.appendingPathComponent("repos.json")
try Data("[]".utf8).write(to: fileURL, options: .atomic)   // .atomic: ghi file tạm rồi rename → không hỏng nửa chừng
let data = try Data(contentsOf: fileURL)
let appSupport = URL.applicationSupportDirectory            // iOS 16+
```
Đường dẫn tuyệt đối của sandbox **đổi giữa các lần cài/cập nhật** → chỉ lưu **đường dẫn tương đối**, ghép lại lúc chạy. File lớn tải lại được mà lưu ở Documents → App Review có thể reject (vi phạm iCloud backup guideline) → đặt `isExcludedFromBackup = true` hoặc dùng Caches.

**Keychain** — kho mã hoá của hệ thống, dành cho **token, mật khẩu, khoá**. Dữ liệu Keychain **còn lại sau khi xoá app** (cần xử lý khi cài lại: xoá token cũ nếu là lần chạy đầu). API C-style `SecItemAdd/CopyMatching/Update/Delete` — chi tiết và code ở **tuần 17**.

### §8. Core Data

Core Data **không phải database** — nó là framework **quản lý object graph** và persistence (thường lưu xuống SQLite). Có change tracking, undo, faulting (lazy load), migration.

**Stack:**
```
 NSPersistentContainer  (gói sẵn mọi thứ bên dưới, iOS 10+)
   ├── NSManagedObjectModel           ← schema (.xcdatamodeld): entity, attribute, relationship
   ├── NSPersistentStoreCoordinator   ← nối model với store
   │      └── NSPersistentStore       ← file SQLite (hoặc in-memory)
   ├── viewContext                    ← NSManagedObjectContext trên MAIN queue (cho UI)
   └── newBackgroundContext()         ← context trên PRIVATE queue (import, ghi nặng)

 NSManagedObjectContext = "nháp": tạo/sửa/xoá object trong context, gọi save() mới ghi xuống store
```
```swift
final class CoreDataStack: @unchecked Sendable {
    let container: NSPersistentContainer

    init(modelName: String = "RepoScope", inMemory: Bool = false) {
        container = NSPersistentContainer(name: modelName)
        if inMemory {   // cho unit test: không đụng file thật
            container.persistentStoreDescriptions.first?.url = URL(fileURLWithPath: "/dev/null")
        }
        container.loadPersistentStores { _, error in
            if let error { fatalError("Load store failed: \(error)") }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true    // nhận thay đổi từ background
        container.viewContext.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump
    }

    func saveFavorite(id: Int64, name: String) async throws {
        let context = container.newBackgroundContext()
        try await context.perform {                    // MỌI thao tác trên context phải trong perform
            let object = NSEntityDescription.insertNewObject(forEntityName: "FavoriteRepo", into: context)
            object.setValue(id, forKey: "id")
            object.setValue(name, forKey: "name")
            if context.hasChanges { try context.save() }
        }
    }
}
```

**Quy tắc threading — câu hỏi kinh điển:**
1. `NSManagedObjectContext` và `NSManagedObject` **không thread-safe**. Mỗi context gắn với một queue; chỉ dùng context/object trên queue của nó.
2. Truy cập context qua `perform { }` (async) hoặc `performAndWait { }` (sync). `viewContext` thì dùng trên main thread.
3. **Không truyền `NSManagedObject` giữa các thread/context.** Truyền **`NSManagedObjectID`** (thread-safe), rồi ở context bên kia `context.object(with: id)` hoặc `existingObject(with:)`.
4. Bật debug: launch argument `-com.apple.CoreData.ConcurrencyDebug 1` → crash ngay khi vi phạm (nên bật suốt lúc dev).
5. Import nặng (hàng nghìn record) ở background context, hoặc dùng `NSBatchInsertRequest` (ghi thẳng xuống SQLite, không qua context → nhanh, nhưng phải tự merge thay đổi vào viewContext).

**Merge policy:** khi 2 context sửa cùng object rồi save → conflict. Mặc định `NSErrorMergePolicy` → save **throw lỗi**. `mergeByPropertyObjectTrump`: thay đổi trong memory (của context đang save) thắng.

**NSFetchedResultsController (FRC)** — "cầu nối" Core Data ↔ table/collection view: chạy fetch request, theo dõi thay đổi trong context và báo cho UI cập nhật đúng dòng.
```swift
let request = NSFetchRequest<NSManagedObject>(entityName: "FavoriteRepo")
request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]   // BẮT BUỘC có sort
request.fetchBatchSize = 20                                                  // chỉ load 20 object/lần
let frc = NSFetchedResultsController(fetchRequest: request, managedObjectContext: context,
                                     sectionNameKeyPath: nil, cacheName: nil)
frc.delegate = self
try frc.performFetch()
// delegate (iOS 13+): controller(_:didChangeContentWith snapshot:) → apply snapshot vào diffable data source
```

**Faulting:** fetch về object "rỗng" (fault), chỉ load thuộc tính khi truy cập → tiết kiệm RAM. Nhưng truy cập trong vòng lặp → **N+1 query**; dùng `relationshipKeyPathsForPrefetching`.

**Migration:** đổi model (thêm attribute) → tạo version mới của `.xcdatamodeld`. Thay đổi đơn giản (thêm/xoá attribute, đổi tên có renaming ID) → **lightweight migration** tự động (`NSPersistentContainer` bật sẵn). Thay đổi phức tạp → mapping model / staged migration (iOS 17+).

### §9. SwiftData (iOS 17+)

SwiftData là lớp Swift-native **xây trên nền Core Data** (cùng SQLite bên dưới): khai báo model bằng macro, không cần file `.xcdatamodeld`.
```swift
@Model
final class FavoriteRepo {
    @Attribute(.unique) var repoID: Int                 // unique → insert trùng sẽ thành upsert
    var fullName: String
    var stars: Int
    var addedAt: Date
    @Relationship(deleteRule: .cascade) var notes: [RepoNote] = []

    init(repoID: Int, fullName: String, stars: Int, addedAt: Date = .now) {
        self.repoID = repoID
        self.fullName = fullName
        self.stars = stars
        self.addedAt = addedAt
    }
}

@Model
final class RepoNote {
    var text: String
    var repo: FavoriteRepo?                            // inverse relationship
    init(text: String) { self.text = text }
}
```
```swift
// @Model chỉ dùng được với class:
// @Model struct Fav { var id: Int }    // ❌ '@Model' cannot be applied to struct type 'Fav'
```

**Setup & dùng trong SwiftUI:**
```swift
@main
struct RepoScopeApp: App {
    var body: some Scene {
        WindowGroup { FavoritesView() }
            .modelContainer(for: FavoriteRepo.self)      // tạo ModelContainer, đưa mainContext vào environment
    }
}

struct FavoritesView: View {
    @Query(sort: \FavoriteRepo.addedAt, order: .reverse) private var favorites: [FavoriteRepo]
    @Environment(\.modelContext) private var context

    var body: some View {
        List {
            ForEach(favorites) { repo in Text(repo.fullName) }
                .onDelete { offsets in
                    for i in offsets { context.delete(favorites[i]) }   // @Query tự cập nhật list
                }
        }
    }
}
```
**Ngoài View (ViewModel/Repository, test):**
```swift
let config = ModelConfiguration(isStoredInMemoryOnly: true)                 // test
let container = try ModelContainer(for: FavoriteRepo.self, configurations: config)
let context = container.mainContext                                         // @MainActor
context.insert(FavoriteRepo(repoID: 1, fullName: "apple/swift", stars: 65000))
try context.save()                                                          // autosave có, nhưng save tường minh cho chắc

let popular = 1000
let descriptor = FetchDescriptor<FavoriteRepo>(
    predicate: #Predicate { $0.stars > popular },       // type-safe, compile-time check
    sortBy: [SortDescriptor(\.stars, order: .reverse)]
)
let result = try context.fetch(descriptor)
```
**Threading:** `ModelContext` và `@Model` object **không Sendable** (giống Core Data). Ghi nền dùng `@ModelActor`:
```swift
@ModelActor
actor FavoritesStore {                       // macro tạo sẵn modelContainer + modelContext riêng của actor
    func add(id: Int, name: String) throws {
        modelContext.insert(FavoriteRepo(repoID: id, fullName: name, stars: 0))
        try modelContext.save()
    }
}
// let store = FavoritesStore(modelContainer: container)
// Truyền giữa actor: dùng PersistentIdentifier (giống NSManagedObjectID), không truyền object.
```

### §10. So sánh lựa chọn persistence & offline-first

| | Core Data | SwiftData | Realm | GRDB / SQLite |
|---|---|---|---|---|
| Tối thiểu | iOS 3 | **iOS 17** | iOS cũ đều được | iOS 13+ (GRDB 7) |
| Nguồn | Apple | Apple (trên Core Data) | MongoDB (Atlas Device Sync đã ngừng, SDK còn mã nguồn mở) | Mã nguồn mở |
| Model | `.xcdatamodeld` hoặc code | Swift macro `@Model` | Class kế thừa `Object` | Struct + protocol `Record` |
| Threading | Context gắn queue, rắc rối | `@ModelActor`, `ModelContext` không Sendable | Object gắn thread (frozen để chia sẻ) | Struct value type → **dễ nhất** |
| Query | NSPredicate (string), `@FetchRequest` | `#Predicate`, `@Query` | API riêng | **SQL thật**, rất mạnh |
| Độ trưởng thành | Rất cao | Còn non (bug, thiếu tính năng ở iOS 17) | Cao | Cao |
| CloudKit sync | ✅ `NSPersistentCloudKitContainer` | ✅ | ❌ | ❌ (tự làm) |

**Chọn thế nào (câu trả lời mẫu):** app mới chỉ hỗ trợ iOS 17+ và model đơn giản → SwiftData; app phải hỗ trợ iOS cũ hoặc cần tính năng nâng cao/ổn định → Core Data; cần SQL phức tạp, kiểm soát hiệu năng, model value type → GRDB. Realm: cân nhắc kỹ vì phụ thuộc vendor. Dù chọn gì, **giấu sau Repository protocol** → đổi được.

**Offline-first** — UI luôn đọc từ DB local (single source of truth), network chỉ là cách **cập nhật DB**:
```
  View ◀── observe ── Local DB (SwiftData/Core Data) ◀── ghi ── Repository ── fetch ──▶ API
                                                                   │
                       action của user (favorite) ── ghi DB ngay ──┘ + đưa vào "outbox" → sync khi có mạng
```
Lợi ích: mở app thấy dữ liệu ngay, không mạng vẫn dùng được. Khó khăn: **conflict resolution** (sửa offline trên 2 máy), outbox/retry, xoá dữ liệu cũ, schema migration. Phỏng vấn mức 2 năm chỉ cần nói được ý tưởng + "stale-while-revalidate": hiện cache trước, gọi API nền, cập nhật khi về.

### §11. XCTest và Swift Testing

**Cấu trúc một unit test tốt — Given / When / Then (Arrange / Act / Assert):**

Các ví dụ dưới dùng một `RepoListViewModel` rút gọn: `Repo(id:name:stars:)`, `State` gồm `idle/loading/loaded([String])/failed(String)`, `load(query:)` gọi `RepoServiceProtocol` rồi sắp xếp tên theo sao giảm dần (giống tuần 15).
```swift
@MainActor
final class RepoListViewModelTests: XCTestCase {
    // Factory thay cho setUp: mỗi test tự tạo SUT mới, không chia sẻ state
    private func makeSUT(result: Result<[Repo], Error> = .success([]))
        -> (sut: RepoListViewModel, service: MockRepoService) {
        let service = MockRepoService()
        service.result = result
        return (RepoListViewModel(service: service), service)
    }

    func test_load_success_sortsByStarsDescending() async {
        // Given
        let (sut, service) = makeSUT(result: .success([
            Repo(id: 1, name: "a", stars: 1), Repo(id: 2, name: "b", stars: 5)
        ]))
        // When
        await sut.load(query: "swift")
        // Then
        XCTAssertEqual(sut.state, .loaded(["b", "a"]))
        XCTAssertEqual(service.receivedQueries, ["swift"])
    }

    func test_load_failure_setsFailedState() async {
        let (sut, _) = makeSUT(result: .failure(URLError(.notConnectedToInternet)))
        await sut.load(query: "swift")
        XCTAssertEqual(sut.state, .failed("Không tải được dữ liệu"))
    }
}
```
- **SUT** = System Under Test. Đặt tên test theo hành vi: `test_<hàm>_<điều kiện>_<kết quả mong đợi>`.
- XCTest: class kế thừa `XCTestCase`, method bắt đầu bằng `test`, `setUp()`/`tearDown()` chạy trước/sau mỗi test. **Mỗi test chạy trên một instance mới** của class.
- Assertion: `XCTAssertEqual`, `XCTAssertTrue`, `XCTAssertNil`, `XCTAssertThrowsError`, `XCTUnwrap` (unwrap hoặc fail), `XCTFail`.

**Swift Testing (Xcode 16+)** — framework mới, dùng macro:
```swift
import Testing

@MainActor
@Suite("RepoListViewModel")
struct RepoListViewModelSpec {
    let service = MockRepoService()                  // init của struct = setUp; mỗi test một instance mới

    @Test("Sắp xếp theo số sao giảm dần")
    func sortsByStars() async {
        service.result = .success([Repo(id: 1, name: "a", stars: 1), Repo(id: 2, name: "b", stars: 5)])
        let sut = RepoListViewModel(service: service)
        await sut.load(query: "swift")
        #expect(sut.state == .loaded(["b", "a"]))    // fail sẽ in rõ giá trị 2 vế
    }

    @Test(arguments: ["", " ", "swift ui"])          // parameterized: 1 test chạy 3 case
    func passesQueryThrough(query: String) async {
        let sut = RepoListViewModel(service: service)
        await sut.load(query: query)
        #expect(service.receivedQueries == [query])
    }

    @Test func requireUnwrap() throws {
        let first = try #require([1, 2, 3].first)    // như XCTUnwrap: nil → test dừng
        #expect(first == 1)
    }

    @Test func throwsError() {
        #expect(throws: URLError.self) { throw URLError(.badURL) }
    }
}
```
| | XCTest | Swift Testing |
|---|---|---|
| Khai báo | class `XCTestCase`, tên `test...` | `@Test` trên func bất kỳ, struct `@Suite` |
| Assert | ~40 hàm `XCTAssert*` | `#expect`, `#require` |
| Setup | `setUp`/`tearDown` | `init`/`deinit` |
| Parameterized | Tự viết vòng lặp | `@Test(arguments:)` |
| Song song | Theo process | **Mặc định song song** in-process (`.serialized` để tắt) |
| UI test, performance test (`measure`) | ✅ | ❌ (vẫn dùng XCTest) |

Hai framework **sống chung trong một target**. Code mới nên dùng Swift Testing; codebase cũ vẫn đầy XCTest → phải đọc được cả hai.

### §12. Mock qua protocol & test code async

**Test double — gọi đúng tên:**
| Loại | Làm gì | Ví dụ |
|---|---|---|
| **Dummy** | Chỉ để đủ tham số, không dùng | `nil` logger |
| **Stub** | Trả dữ liệu cố định | `StubRepoRepository` trả 2 repo |
| **Spy** | Stub + **ghi lại** đã được gọi thế nào | `receivedQueries` |
| **Mock** | Spy + tự verify kỳ vọng | `XCTAssertEqual(mock.calls, [...])` |
| **Fake** | Implementation thật nhưng đơn giản | DB in-memory, `InMemoryFavoritesRepository` |

```swift
final class MockRepoService: RepoServiceProtocol, @unchecked Sendable {
    var result: Result<[Repo], Error> = .success([])
    private(set) var receivedQueries: [String] = []

    func searchRepos(query: String) async throws -> [Repo] {
        receivedQueries.append(query)
        return try result.get()
    }
}
```
`@unchecked Sendable` chấp nhận được trong test vì mock chỉ dùng trên một luồng; code production thì không nên.

**Test code async:**
- Hàm `async` → test cũng `async`, gọi `await` trực tiếp (XCTest và Swift Testing đều hỗ trợ).
- Code dùng **callback/delegate** → XCTest dùng `expectation`:
```swift
func test_expectation_style() {
    let exp = expectation(description: "callback")
    DispatchQueue.global().async { exp.fulfill() }
    wait(for: [exp], timeout: 1)            // trong test async thì: await fulfillment(of: [exp], timeout: 1)
}
```
Swift Testing dùng `await confirmation { confirm in ... confirm() }` hoặc bọc callback bằng `withCheckedContinuation`.
- **Không dùng `sleep` trong test** để "đợi cho xong" → test chậm và flaky. Inject dependency kiểm soát thời gian (clock, scheduler) hoặc để hàm trả về khi xong việc.
- Code bắn `Task { }` bên trong (fire-and-forget) → khó test; tốt hơn là expose hàm `async` để test `await` được (như `load(query:)` ở trên).

### §13. Stub network bằng URLProtocol

Muốn test **cả APIClient** (build request, map status code, decode) mà không gọi mạng thật: đăng ký một `URLProtocol` tự viết vào `URLSessionConfiguration`. Mọi request của session đó đi qua class này.
```swift
final class StubURLProtocol: URLProtocol, @unchecked Sendable {
    nonisolated(unsafe) static var handler: (@Sendable (URLRequest) throws -> (HTTPURLResponse, Data))?

    override class func canInit(with request: URLRequest) -> Bool { true }   // chặn mọi request
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }

    override func startLoading() {
        guard let handler = Self.handler else {
            client?.urlProtocol(self, didFailWithError: URLError(.unknown)); return
        }
        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)       // giả lập mất mạng: throw URLError(.notConnectedToInternet)
        }
    }
    override func stopLoading() {}
}

func makeStubSession() -> URLSession {
    let config = URLSessionConfiguration.ephemeral
    config.protocolClasses = [StubURLProtocol.self]
    return URLSession(configuration: config)
}

func response(_ request: URLRequest, _ status: Int, _ json: String) -> (HTTPURLResponse, Data) {
    (HTTPURLResponse(url: request.url!, statusCode: status, httpVersion: nil, headerFields: nil)!, Data(json.utf8))
}
```
```swift
@Suite(.serialized)   // handler là static dùng chung → chạy tuần tự
struct APIClientTests {
    let client = APIClient(baseURL: URL(string: "https://api.github.com")!, session: makeStubSession())

    @Test func decodesSuccessResponse() async throws {
        StubURLProtocol.handler = { request in
            #expect(request.url?.path == "/search/repositories")      // kiểm tra request build đúng
            return response(request, 200, #"{"total_count":1,"items":[{"id":1,"full_name":"apple/swift","stargazers_count":5}]}"#)
        }
        let result = try await client.send(.searchRepos(query: "swift", page: 1))
        #expect(result.items.map(\.fullName) == ["apple/swift"])
    }

    @Test func maps404ToNotFound() async {
        StubURLProtocol.handler = { response($0, 404, "{}") }
        await #expect(throws: APIError.notFound) {
            try await client.send(.searchRepos(query: "x", page: 1))
        }
    }
}
```
**Hai lỗi thật đã gặp khi viết đoạn này:**
1. Handler là `static` → các test Swift Testing chạy **song song** sẽ ghi đè handler của nhau → dùng `.serialized`, hoặc map handler theo một header/URL riêng cho từng test.
2. Nếu khai báo handler **không có `@Sendable`**, Swift 6 suy ra closure viết trong context `@MainActor` là MainActor-isolated, nhưng URLSession gọi nó trên thread `com.apple.CFNetwork.CustomProtocols` → Swift 6 chèn kiểm tra isolation lúc chạy và có thể **crash runtime** (`_dispatch_assert_queue_fail`). Thêm `@Sendable` để closure không bị gắn vào MainActor.

Lưu ý: `httpBody` của request thường là `nil` bên trong URLProtocol (body bị chuyển thành `httpBodyStream`) → muốn kiểm tra body thì đọc stream.

**Mock ở tầng nào?** Test ViewModel → mock `RepoServiceProtocol`/Repository (nhanh, đơn giản). Test APIClient/Repository → URLProtocol stub. Không cần protocol bọc `URLSession`.

### §14. Snapshot test, UI test, code coverage, testable architecture

**Kim tự tháp test:** nhiều unit test (nhanh, ổn định) → ít integration test → rất ít UI test (chậm, flaky).

- **UI test (XCUITest):** chạy app thật trong process riêng, điều khiển qua accessibility: `app.buttons["Favorite"].tap()`, `XCTAssertTrue(app.staticTexts["apple/swift"].waitForExistence(timeout: 2))`. Đặt `accessibilityIdentifier` để không phụ thuộc text/ngôn ngữ. Truyền `app.launchArguments = ["-UITest"]` để app dùng stub server. Chậm, dễ flaky → chỉ cho luồng quan trọng (login, thanh toán).
- **Snapshot test** (thư viện `swift-snapshot-testing` của Point-Free): render view thành ảnh, so với ảnh tham chiếu đã commit → bắt lỗi UI vô tình (lệch layout, dark mode, Dynamic Type). Nhược: phụ thuộc simulator/OS version, ảnh tham chiếu phải cập nhật khi đổi UI có chủ đích.
- **Performance test:** `measure { }` trong XCTest.
- **Code coverage:** bật trong Scheme → Test → Options → "Code Coverage". Xem trong Report navigator. **Coverage cao ≠ test tốt** (gọi hàm mà không assert vẫn tính là covered). Mục tiêu hợp lý: phủ kỹ Domain/ViewModel/Networking (70–90%), không ép UI code.
- **Testable architecture:** dependency inject qua protocol, logic trong type thuần Swift (không UIKit), side effect (network, DB, thời gian `Date()`, `UUID()`) bị đẩy ra ngoài và inject được, hàm async trả về khi xong việc. Code khó test thường là dấu hiệu thiết kế dính chặt.

`@testable import RepoScope` cho phép test truy cập type `internal` (chỉ hoạt động khi build Debug có "Enable Testability").

---

## ⚠️ Bẫy hay gặp

1. **4xx/5xx không làm `URLSession` throw** — phải tự kiểm tra `statusCode`. Rất nhiều code "bắt lỗi" mà không bao giờ bắt được 500.
2. **Quên `.resume()`** với API completion handler → request không bao giờ chạy.
3. **Cập nhật UI từ completion handler** (chạy trên background queue) → Main Thread Checker báo lỗi. Với async/await, đặt ViewModel `@MainActor`.
4. **Tạo `URLSession` mới cho mỗi request** → mất connection reuse (HTTP/2), cache; session có delegate còn bị leak.
5. **Retry POST không có idempotency key** → tạo trùng đơn hàng/bình luận.
6. **Refresh token không đồng bộ** → nhiều request cùng refresh, user bị logout oan. Dùng actor + `Task` chia sẻ.
7. **Token trong UserDefaults** → đọc được dễ dàng. Dùng Keychain.
8. **Lưu đường dẫn tuyệt đối của file** → sau update app, UUID sandbox đổi, file "biến mất".
9. **Dùng `NSManagedObject` ở thread khác** context của nó → crash ngẫu nhiên, khó tái hiện. Truyền `NSManagedObjectID`. Bật `-com.apple.CoreData.ConcurrencyDebug 1`.
10. **`@Model` object/`ModelContext` truyền qua actor** → lỗi Sendable; dùng `PersistentIdentifier`.
11. **`downloadTask` file tạm bị xoá** khi hàm trả về → phải `moveItem` ngay.
12. **Test dùng `sleep`/thời gian thật** → flaky. Test chia sẻ state static (singleton, `StubURLProtocol.handler`) + chạy song song → fail ngẫu nhiên.
13. **Cell reuse + ảnh async** → hiển thị ảnh của cell cũ. Huỷ task trong `prepareForReuse` và kiểm tra URL khi ảnh về.

---

## 🛠 Bài tập

> Code đặt trong `practice/week16/`. Phần networking/test nên tạo **Swift package** (`swift package init --type library`) để chạy `swift test` không cần Xcode; package macOS vẫn dùng được `URLSession`, `URLProtocol`, Swift Testing.

**Bài 1 — HTTP (T2)**
Không nhìn §1, tự viết bảng: 5 method (safe? idempotent?), 12 status code hay gặp + app nên phản ứng thế nào. Rồi trả lời: *"Bấm nút Đặt hàng, request timeout. App có nên tự retry không?"*
<details><summary>Gợi ý câu hỏi cuối</summary>

POST không idempotent; timeout không có nghĩa là server chưa xử lý. Không tự retry mù quáng. Cách đúng: gửi kèm `Idempotency-Key` (UUID tạo khi user bấm, giữ nguyên khi retry) để server bỏ qua request trùng; hoặc trước khi retry gọi API kiểm tra trạng thái đơn hàng; UI disable nút trong lúc gửi để tránh double-tap.
</details>

**Bài 2 — Network layer từ đầu (T3)**
Đóng file này lại, tự viết `Endpoint<Response>`, `APIError`, `APIClient` (map status code, decode snake_case, retry tối đa 2 lần cho lỗi tạm thời với backoff). Dùng `StubURLProtocol` viết script kiểm tra: (a) 200 decode đúng, (b) 404 → `.notFound`, (c) 500, 500, 200 → thành công sau 3 lần gọi, (d) 400 → không retry (1 lần gọi).
<details><summary>Gợi ý (d)</summary>

Đếm số lần handler được gọi bằng một class counter có lock (handler là `@Sendable` nên không capture `var` được):
```swift
final class Counter: @unchecked Sendable {
    private let lock = NSLock(); private var value = 0
    func increment() -> Int { lock.lock(); defer { lock.unlock() }; value += 1; return value }
}
```
`APIError.http(statusCode: 400).isRetryable == false` → `send` ném lỗi ngay, counter = 1.
</details>

**Bài 3 — Refresh token & pagination (T4)**
a) Viết `AuthManager` actor. Test: 3 request song song đều nhận 401 khi token là `"old"`, 200 khi `"new"` → `refreshCount == 1`. Sau đó **cố tình sửa sai**: bỏ `refreshTask`, gọi thẳng `token = try await refresher()` — chạy lại, `refreshCount` là bao nhiêu? Vì sao?
b) Viết `RepoPaginator`; gọi `loadNextPage()` 5 lần **đồng thời** bằng `TaskGroup` khi đang ở trang 1 — đảm bảo chỉ 1 request được gửi.
<details><summary>Lời giải a)</summary>

Bản sửa sai cho `refreshCount == 3`: mỗi lời gọi vào actor, thấy chưa có ai refresh (không có trạng thái "đang refresh"), rồi `await refresher()` — tại điểm `await`, actor nhả ra cho lời gọi tiếp theo vào (**reentrancy**), lời gọi đó cũng tự refresh. Actor chỉ đảm bảo không có 2 đoạn code chạy *cùng lúc*, không đảm bảo một hàm async chạy *trọn vẹn* không bị xen. Phải lưu `Task` đang chạy trước `await` để các lời gọi sau chờ chung.
```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    for _ in 0..<3 { group.addTask { _ = try await authed.send(.searchRepos(query: "x", page: 1)) } }
    try await group.waitForAll()
}
print("refreshCount:", await auth.refreshCount)   // refreshCount: 1
```
</details>

**Bài 4 — Persistence (T5)**
a) Cho từng dữ liệu, chọn nơi lưu và giải thích: token GitHub, theme sáng/tối, 500 repo đã favorite, ảnh avatar đã tải, file CSV user export, dữ liệu tạm khi nén ảnh.
b) Viết `@Model FavoriteRepo` + một `FavoritesRepository` (protocol) có 2 implementation: `SwiftDataFavoritesRepository` và `InMemoryFavoritesRepository`. Viết test cho bản SwiftData dùng `ModelConfiguration(isStoredInMemoryOnly: true)`.
<details><summary>Lời giải a)</summary>

Token → Keychain. Theme → UserDefaults/`@AppStorage`. 500 favorite → SwiftData/Core Data (Application Support). Avatar → `Caches/` hoặc URLCache (tải lại được). CSV export → `Documents/` (user tạo, cần backup, có thể hiện trong app Files). Dữ liệu tạm khi nén → `tmp/`, xoá khi xong.
</details>

**Bài 5 — Test ViewModel (T6)**
Viết `SearchViewModel` có: `search(query:)` (query < 2 ký tự thì không gọi service), state `idle/loading/loaded/empty/failed`. Viết **6 test** bằng Swift Testing, trong đó có 1 test parameterized cho các query không hợp lệ `["", "a", " "]`, và 1 test kiểm tra `.empty` khi service trả mảng rỗng.
<details><summary>Gợi ý</summary>

```swift
@Test(arguments: ["", "a", " "])
func invalidQuery_doesNotCallService(query: String) async {
    let service = MockRepoService()
    let sut = SearchViewModel(service: service)
    await sut.search(query: query)
    #expect(service.receivedQueries.isEmpty)
    #expect(sut.state == .idle)
}
```
Chú ý `" "` — nhớ `trimmingCharacters(in: .whitespaces)` trước khi đếm.
</details>

**Bài 6 — Test Core Data threading (T6)**
Viết `CoreDataStack(inMemory: true)` với model tạo bằng code (`NSManagedObjectModel` + `NSEntityDescription`, không cần file `.xcdatamodeld`). Ghi 1.000 record ở background context, kiểm tra `viewContext` đếm được 1.000. Thử cố tình đọc `object.value(forKey:)` của object background **ngoài** `perform` với `-com.apple.CoreData.ConcurrencyDebug 1` — ghi lại chuyện gì xảy ra.

**🏋️ Bài tập lớn (T7) — RepoScope Milestone 16**
(Side project: Milestone 16)
- Thay networking của RepoScope bằng `APIClient` + `Endpoint` của bạn; xử lý `403/429` rate limit của GitHub (hiện thông báo, đọc `X-RateLimit-Reset`)
- Pagination khi cuộn danh sách search, không gọi trùng trang
- **Favorites** lưu bằng SwiftData (hoặc Core Data nếu bạn muốn hỗ trợ iOS < 17), giấu sau `FavoritesRepository`
- **Offline cache:** lưu kết quả search gần nhất; mở app khi mất mạng vẫn thấy danh sách + banner "Đang offline"
- **≥ 15 unit test**: ViewModel (mock repository), UseCase, `APIClient` qua `URLProtocol` stub (200, 404, 401→refresh, 500→retry, decode lỗi), paginator, favorites repository in-memory
- Bật code coverage, chụp màn hình % coverage của module Domain/Data đưa vào README

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ GET, POST, PUT, PATCH, DELETE khác nhau thế nào? Idempotent là gì?**
<details><summary>Đáp án gợi ý</summary>

GET đọc, POST tạo/hành động, PUT thay toàn bộ resource, PATCH sửa một phần, DELETE xoá. Idempotent: gọi N lần cho kết quả trên server như gọi 1 lần — GET, PUT, DELETE là idempotent; POST không (và PATCH không đảm bảo). Quan trọng với mobile vì mạng chập chờn: retry request idempotent thì an toàn, retry POST có thể tạo trùng → cần `Idempotency-Key` hoặc kiểm tra trạng thái trước khi retry.
</details>

**Q2 ⭐ 401 và 403 khác nhau thế nào? App xử lý ra sao?**
<details><summary>Đáp án gợi ý</summary>

401 Unauthorized: chưa xác thực hoặc token không hợp lệ/hết hạn → refresh token rồi thử lại một lần; refresh thất bại → đăng xuất. 403 Forbidden: đã xác thực nhưng không có quyền → không refresh, hiện thông báo không có quyền. (GitHub dùng 403/429 cho rate limit, cần đọc header `X-RateLimit-Remaining`/`Retry-After`.)
</details>

**Q3 ⭐ Bạn thiết kế network layer như thế nào?**
<details><summary>Đáp án gợi ý</summary>

Một `Endpoint<Response>` mô tả request (path, method, query, body, header) và mang luôn kiểu response để decode type-safe. Một `APIClient` qua protocol (để mock) nhận endpoint, build `URLRequest`, gắn token từ `AuthManager`, gọi `URLSession`, map status code thành `APIError` có nghĩa với app, decode bằng `JSONDecoder` cấu hình chung. Thêm retry có backoff cho lỗi tạm thời (timeout, 5xx), refresh token khi 401 bằng actor để không refresh trùng. Repository dùng client và map DTO sang domain model. Test bằng `URLProtocol` stub.
</details>

**Q4 ⭐ Nhiều request cùng lúc bị 401, làm sao chỉ refresh token một lần?**
<details><summary>Đáp án gợi ý</summary>

Đặt logic refresh trong một `actor` giữ `Task` refresh đang chạy. Request đầu tiên tạo `Task` và lưu lại **trước khi `await`**; các request sau vào actor thấy task đang tồn tại thì `await task.value` chung. Xong thì xoá task. Phải lưu task trước `await` vì actor reentrancy: trong lúc chờ, actor cho lời gọi khác chạy. Trước Swift Concurrency, người ta dùng serial queue + mảng completion handler đang chờ.
</details>

**Q5 URLSessionConfiguration có những loại nào?**
<details><summary>Đáp án gợi ý</summary>

`.default` (cache disk, cookie, credential lưu bền), `.ephemeral` (mọi thứ trong RAM, không ghi disk — hợp ẩn danh và unit test), `.background(withIdentifier:)` (hệ thống tiếp tục upload/download ở process riêng kể cả khi app suspend hoặc bị kill, chỉ hỗ trợ download/upload task từ file, kết quả báo qua delegate và `handleEventsForBackgroundURLSession`).
</details>

**Q6 ⭐ HTTP caching hoạt động thế nào? ETag là gì?**
<details><summary>Đáp án gợi ý</summary>

Server điều khiển cache qua header: `Cache-Control: max-age=N` cho phép dùng bản cache N giây không cần hỏi lại; `no-cache` phải xác nhận lại; `no-store` không lưu. `ETag` là "phiên bản" của resource; lần sau client gửi `If-None-Match: <etag>`, nếu không đổi server trả `304 Not Modified` không có body → tiết kiệm băng thông. `URLCache` với `.useProtocolCachePolicy` tự làm theo các header này. Cache trong app thường có tầng memory (`NSCache`), disk, rồi mới network.
</details>

**Q7 Làm pagination thế nào để không load trùng?**
<details><summary>Đáp án gợi ý</summary>

Giữ `nextPage`/`cursor`, `isLoading`, `hasMore` trong một nơi được cô lập (actor hoặc `@MainActor` ViewModel). Khi item gần cuối xuất hiện, gọi `loadNextPage`; hàm guard `!isLoading && hasMore`. Chỉ tăng page khi thành công để lỗi thì thử lại đúng trang. Đổi query thì huỷ task cũ và reset. Cursor-based ổn định hơn offset khi dữ liệu thay đổi giữa các trang.
</details>

**Q8 Load ảnh trong list thế nào cho mượt và đúng?**
<details><summary>Đáp án gợi ý</summary>

Cache 2 tầng (`NSCache` trong RAM, disk/URLCache), gộp request trùng URL đang tải, downsample ảnh lớn về kích thước hiển thị trên background, decode trước khi gán. Trong cell: huỷ task trong `prepareForReuse`, reset image, khi ảnh về kiểm tra cell vẫn hiển thị URL đó. Prefetch bằng `UICollectionViewDataSourcePrefetching`. Thực tế dùng Kingfisher/SDWebImage/Nuke.
</details>

**Q9 ⭐ UserDefaults, Keychain, file, Core Data — lưu gì ở đâu?**
<details><summary>Đáp án gợi ý</summary>

UserDefaults: setting nhỏ, cờ — không lưu dữ liệu nhạy cảm (plist không mã hoá) và không lưu dữ liệu lớn (load hết vào RAM khi launch). Keychain: token, mật khẩu, khoá — mã hoá, có access control, tồn tại sau khi xoá app. File: dữ liệu lớn/blob — `Documents` cho file user tạo, `Application Support` cho dữ liệu app, `Caches` cho thứ tải lại được, `tmp` cho tạm. Core Data/SwiftData/SQLite: dữ liệu có cấu trúc, cần query, quan hệ, nhiều record.
</details>

**Q10 ⭐ Core Data stack gồm những gì?**
<details><summary>Đáp án gợi ý</summary>

`NSManagedObjectModel` (schema), `NSPersistentStoreCoordinator` (nối model với store), `NSPersistentStore` (thường SQLite), và `NSManagedObjectContext` (vùng nháp làm việc với object, save mới ghi xuống). `NSPersistentContainer` gói tất cả, cung cấp `viewContext` (main queue, cho UI) và `newBackgroundContext()`/`performBackgroundTask` (private queue). Core Data là framework quản lý object graph có change tracking, faulting, migration — không phải database.
</details>

**Q11 ⭐ Quy tắc threading của Core Data?**
<details><summary>Đáp án gợi ý</summary>

Context và managed object không thread-safe, mỗi context gắn với một queue. Mọi thao tác phải trong `perform`/`performAndWait` (viewContext thì trên main). Không truyền `NSManagedObject` giữa thread — truyền `NSManagedObjectID` rồi `object(with:)` ở context kia. Ghi nặng ở background context, viewContext đặt `automaticallyMergesChangesFromParent = true` để nhận thay đổi, đặt merge policy để tránh lỗi conflict. Bật `-com.apple.CoreData.ConcurrencyDebug 1` khi dev.
</details>

**Q12 NSFetchedResultsController dùng để làm gì?**
<details><summary>Đáp án gợi ý</summary>

Quản lý kết quả một fetch request (bắt buộc có sort descriptor) cho table/collection view: chia section, load theo batch, và theo dõi thay đổi trong context để báo delegate — từ iOS 13 có thể nhận thẳng `NSDiffableDataSourceSnapshot` để apply, UI tự cập nhật đúng dòng thêm/xoá/sửa mà không reload toàn bộ.
</details>

**Q13 ⭐ SwiftData khác Core Data thế nào? Bạn chọn cái nào?**
<details><summary>Đáp án gợi ý</summary>

SwiftData xây trên Core Data, nhưng model khai báo bằng macro `@Model` trong code, query bằng `#Predicate` type-safe, tích hợp SwiftUI qua `.modelContainer` và `@Query`, background dùng `@ModelActor`. Nhược: chỉ iOS 17+, còn non (thiếu tính năng, bug ở bản đầu), ít kiểm soát hơn. Core Data trưởng thành, hỗ trợ iOS cũ, nhiều tính năng nâng cao nhưng API dài dòng, threading khó. App mới iOS 17+ và model đơn giản → SwiftData; cần ổn định/hỗ trợ cũ → Core Data. Luôn giấu sau Repository để đổi được.
</details>

**Q14 Offline-first là gì?**
<details><summary>Đáp án gợi ý</summary>

DB local là nguồn sự thật duy nhất: UI luôn đọc và quan sát DB, network chỉ dùng để đồng bộ DB. Mở app hiện dữ liệu ngay, gọi API nền, ghi vào DB, UI tự cập nhật. Thao tác của user ghi DB ngay và vào hàng đợi đồng bộ khi có mạng. Khó khăn: xử lý conflict, retry, migration, dọn dữ liệu cũ.
</details>

**Q15 ⭐ Bạn mock dependency trong unit test thế nào?**
<details><summary>Đáp án gợi ý</summary>

Class cần test phụ thuộc vào protocol, được inject qua init. Trong test tạo implementation giả conform protocol: stub trả dữ liệu cố định, spy ghi lại tham số được gọi, fake là bản in-memory. Không cần thư viện mock. Với network, mock ở tầng service/repository khi test ViewModel; khi test APIClient thì dùng `URLProtocol` stub để giữ nguyên `URLSession` thật.
</details>

**Q16 ⭐ Test code network mà không gọi server thật bằng cách nào?**
<details><summary>Đáp án gợi ý</summary>

Tạo subclass `URLProtocol` override `canInit` trả true và `startLoading` trả response/data giả (hoặc lỗi) qua `client`. Tạo `URLSession` với `URLSessionConfiguration.ephemeral` có `protocolClasses = [StubURLProtocol.self]`, inject session này vào APIClient. Kiểm tra được request build đúng (URL, header) và xử lý response đúng (status code, decode, retry). Chú ý handler static → chạy test tuần tự.
</details>

**Q17 Test code async thế nào?**
<details><summary>Đáp án gợi ý</summary>

Hàm async: test method đánh dấu `async` rồi `await` trực tiếp. Code callback: XCTest dùng `expectation` + `fulfill` + `wait(for:timeout:)` hoặc `await fulfillment(of:)`; Swift Testing dùng `confirmation` hoặc bọc bằng continuation. Tránh `sleep`; thiết kế code để expose hàm async trả về khi xong, inject clock/scheduler để điều khiển thời gian. Test chạm `@MainActor` thì đánh dấu test `@MainActor`.
</details>

**Q18 XCTest và Swift Testing khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

XCTest: class `XCTestCase`, method `test...`, nhiều hàm `XCTAssert*`, `setUp/tearDown`, hỗ trợ UI test và performance test. Swift Testing (Xcode 16): `@Test` trên function, `@Suite` struct, chỉ `#expect`/`#require` nhưng in chi tiết giá trị khi fail, test có tham số `@Test(arguments:)`, trait (`.serialized`, `.tags`, `.disabled`), chạy song song mặc định, setup bằng `init`. Hai framework dùng chung được trong một target; UI test vẫn phải dùng XCTest.
</details>

**Q19 Code coverage 100% có nghĩa là code không có bug?**
<details><summary>Đáp án gợi ý</summary>

Không. Coverage chỉ đo dòng nào đã được chạy, không đo có assert đúng hay không, không bao phủ hết tổ hợp input, race condition hay edge case. Nó hữu ích để tìm vùng **chưa** được test. Tập trung coverage vào logic quan trọng (domain, ViewModel, networking) thay vì đuổi theo con số.
</details>

**Q20 Thế nào là code "testable"?**
<details><summary>Đáp án gợi ý</summary>

Logic tách khỏi UI framework; dependency (network, DB, thời gian, random, UUID) inject qua protocol thay vì tạo bên trong hay gọi singleton ẩn; hàm có input/output rõ ràng, ít side effect; async code trả về khi xong việc để test `await` được; state quan sát được từ ngoài. Nếu viết test khó, thường là dấu hiệu thiết kế đang dính chặt.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–6 không xem lời giải trước
- [ ] Tự viết lại `APIClient` + `AuthManager` từ trí nhớ trong ≤ 30 phút
- [ ] Bài tập lớn: RepoScope có favorites SwiftData/Core Data, offline cache, ≥ 15 unit test xanh
- [ ] Trả lời được ≥ 16/20 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Thiết kế network layer của bạn"
- [ ] Ghi âm 2 phút: "Core Data threading và cách bạn tránh crash"
- [ ] Làm xong 6 bài DSA tuần 16 (DP 1D)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [Apple — URLSession](https://developer.apple.com/documentation/foundation/urlsession) và [Accessing cached data](https://developer.apple.com/documentation/foundation/url_loading_system/accessing_cached_data)
- [MDN — HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) · [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [GitHub REST API — Rate limits & conditional requests](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)
- [Apple — Core Data](https://developer.apple.com/documentation/coredata) · [Using Core Data in the background](https://developer.apple.com/documentation/coredata/using_core_data_in_the_background)
- [Apple — SwiftData](https://developer.apple.com/documentation/swiftdata) · WWDC23 *Meet SwiftData*, *Model your schema with SwiftData*
- [Apple — Swift Testing](https://developer.apple.com/documentation/testing) · WWDC24 *Meet Swift Testing*, *Go further with Swift Testing*
- WWDC21 *Use async/await with URLSession*
- [Point-Free — swift-snapshot-testing](https://github.com/pointfreeco/swift-snapshot-testing)
- [GRDB.swift README](https://github.com/groue/GRDB.swift) — phần "Why GRDB"
