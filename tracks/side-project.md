# 📱 Side project — RepoScope (09/11/2026 – 21/03/2027)

> **Chạy song song** với các file tuần · Thời lượng: chủ yếu **Thứ 7 (2–3 giờ)**, thêm 1–2 buổi tối nếu còn sức
> **RepoScope** là app khám phá repository GitHub: tìm kiếm repo, xem chi tiết, avatar chủ sở hữu, (tuỳ chọn) đọc README, lưu **favorites** dùng được **offline**.
> Mỗi tuần app lớn thêm một chút, **đúng bằng kiến thức tuần đó**. Đến tuần 18, đây là thứ bạn **mở ra trên điện thoại cho interviewer xem** và là nguồn của 3 câu chuyện STAR.

---

## 0. Vì sao làm project này, và luật chơi

### 0.1. Vì sao
- CV 1,5–2 năm thường chỉ có "dự án công ty" mà bạn **không show code được**. Một repo public sạch sẽ, có test, có CI, có README tốt là **bằng chứng** bạn tự viết được code — đặc biệt quan trọng khi bạn từng phụ thuộc AI.
- Câu hỏi phỏng vấn kiểu *"Em làm image cache thế nào?"*, *"Pagination xử lý ra sao?"*, *"Test networking thế nào?"* — bạn sẽ trả lời bằng **code của chính mình**.
- GitHub API công khai, miễn phí, không cần backend, có pagination + rate limit + ảnh → đủ "đau" để học thật.

### 0.2. Luật chơi (quan trọng nhất)
| ✅ Được | ❌ Không |
|---|---|
| Đọc Apple docs, Swift book, bài blog, Stack Overflow | Nhờ AI viết code cho project |
| Hỏi AI **"vì sao code của tôi lỗi X"** sau 20' tự debug, **dán lỗi, không dán cả file** | Bật Copilot / Predictive Code Completion trong Xcode |
| Nhờ AI **review** một PR bạn đã tự viết xong (ghi rõ trong commit "reviewed with AI") | Copy skeleton trong file này mà không gõ lại |
| Xem skeleton trong file này, **gõ lại bằng tay**, đổi tên, hiểu từng dòng | Thêm thư viện bên thứ 3 cho phần cốt lõi (Alamofire, Kingfisher, SDWebImage…) |

> Lý do cấm thư viện: interviewer sẽ hỏi *"không có Kingfisher thì em làm image cache thế nào?"*. Project này là câu trả lời.
> Ngoại lệ chấp nhận được (tuần 11–12 trở đi, nếu cần): một thư viện render Markdown cho README, vì đó không phải trọng tâm.

### 0.3. Thói quen Git
- Mỗi milestone ít nhất **1 branch → 1 PR → tự review → merge** (tập quy trình làm việc thật).
- Commit nhỏ, message theo [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `refactor:`, `test:`, `chore:`, `docs:`.
- Gắn **tag** cuối mỗi milestone: `git tag m03-protocols` → sau này kể chuyện "tuần đó em refactor từ… sang…" có bằng chứng.

---

## 1. GitHub REST API dùng trong project

**Base URL:** `https://api.github.com` · Header nên gửi: `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`. (`URLSession` tự gửi `User-Agent`, GitHub bắt buộc header này.)

| Mục đích | Endpoint | Ghi chú |
|---|---|---|
| Tìm repo | `GET /search/repositories?q={query}&sort=stars&order=desc&page={n}&per_page={30}` | `per_page` tối đa **100** (mặc định 30). Search trả tối đa **1.000 kết quả** cho mỗi truy vấn → page × per_page > 1000 sẽ lỗi `422`. Query hỗ trợ qualifier: `swift language:swift stars:>1000` |
| Chi tiết repo | `GET /repos/{owner}/{repo}` | Trả `Repository` đầy đủ (`subscribers_count`, `default_branch`, …) |
| README | `GET /repos/{owner}/{repo}/readme` | JSON có `content` là **Base64** (có xuống dòng `\n`); hoặc gửi `Accept: application/vnd.github.raw+json` để nhận raw Markdown |
| Thông tin user (tuỳ chọn) | `GET /users/{username}` | Avatar, bio, số follower |
| Kiểm tra quota | `GET /rate_limit` | **Không bị tính** vào quota — dùng để debug |
| Avatar | URL trong `owner.avatar_url` (`https://avatars.githubusercontent.com/u/...`) | Thêm `&s=128` để lấy ảnh nhỏ hơn (GitHub hỗ trợ tham số size) |

**Rate limit** (theo docs GitHub tại thời điểm viết, 11/2026 — kiểm tra lại link cuối file):
| Loại | Không token | Có Personal Access Token |
|---|---|---|
| REST API chung | **60 request/giờ**, tính theo IP | **5.000 request/giờ** |
| Search API | **10 request/phút** | **30 request/phút** |

- Header trả về: `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-used`, `x-ratelimit-reset` (epoch giây, UTC), `x-ratelimit-resource`.
- Vượt giới hạn → **`403` hoặc `429`**, với `x-ratelimit-remaining: 0`. App phải hiển thị *"Hết lượt, thử lại sau mm:ss"* thay vì "Lỗi không xác định".
- → Đây là lý do **search phải debounce** (tuần 8) và **phải có cache** (tuần 5, 16): gõ "swiftui" từng chữ không debounce = 7 request = gần hết quota/phút.

**JSON mẫu (rút gọn) của `/search/repositories`:**
```json
{
  "total_count": 1,
  "incomplete_results": false,
  "items": [{
    "id": 44838949,
    "name": "swift",
    "full_name": "swiftlang/swift",
    "owner": { "login": "swiftlang", "id": 42816656,
               "avatar_url": "https://avatars.githubusercontent.com/u/42816656?v=4",
               "html_url": "https://github.com/swiftlang" },
    "html_url": "https://github.com/swiftlang/swift",
    "description": null,
    "stargazers_count": 68000,
    "forks_count": 10400,
    "open_issues_count": 7000,
    "language": "C++",
    "topics": ["swift"],
    "updated_at": "2026-11-09T08:00:00Z"
  }]
}
```
Chú ý: `description`, `language` có thể **`null`** → model phải Optional, nếu không decode sẽ **fail cả trang**.

---

## 2. Tổng quan milestone

| Tuần | Ngày (Thứ 7) | Milestone | Kiến thức tuần |
|---|---|---|---|
| 1 | 14/11 | M1 · Setup repo, Xcode project, `.gitignore`, README | Swift cơ bản |
| 2 | 21/11 | M2 · Model layer `Repository`/`Owner`, mock data, value semantics | Struct/class |
| 3 | 28/11 | M3 · `RepositoryServiceProtocol` + `MockRepositoryService`, extensions | Protocol, extension |
| 4 | 05/12 | M4 · Codable từ file JSON trong bundle, `APIError`, `APIClient` generic (skeleton) | Generics, Codable, error |
| 5 | 12/12 | M5 · Kiểm tra leak bằng Memory Graph, `ImageCache` với `NSCache` | ARC |
| 6 | 19/12 | M6 · Tải ảnh bằng GCD + cache thread-safe | GCD |
| 7 | 26/12 | M7 · async/await, `URLSession` thật + GitHub API, `actor` ImageCache/ImageLoader | Concurrency |
| 8 | 02/01 | M8 · Swift 6 language mode, sửa lỗi Sendable, debounce search | Sendable, AsyncSequence/Combine |
| 9 | 09/01 | M9 · Màn list UIKit (code, không storyboard), VC lifecycle, điều hướng sang detail | UIKit lifecycle |
| 10 | 16/01 | M10 · `UICollectionView` compositional + diffable, pagination, prefetch, downsampling | List performance |
| 11 | 23/01 | M11 · Màn detail SwiftUI với `@Observable` ViewModel | SwiftUI |
| 12 | 30/01 | M12 · `NavigationStack` + interop `UIHostingController` | SwiftUI nâng cao |
| 13–14 | 06/02, 13/02 | 🧧 Nghỉ · README có ảnh chụp màn hình | Ôn tập |
| 15 | 20/02 | M15 · Refactor MVVM + Clean-ish layers + DI + Coordinator | Kiến trúc |
| 16 | 27/02 | M16 · Favorites bằng SwiftData (hoặc Core Data), offline cache, ≥ 15 unit test (có URLProtocol stub) | Persistence, testing |
| 17 | 06/03 | M17 · GitHub Actions CI, Instruments profiling, token trong Keychain (tuỳ chọn), release config | Release, security, performance |
| 18 | 13/03 | M18 · Polish, sơ đồ kiến trúc trong README, chuẩn bị 3 "story" | System design, STAR |
| 19 | 20/03 | M19 · 🧊 Đóng băng — chỉ sửa bug, không thêm tính năng | Phỏng vấn |

> **Mốc M5 của README tổng (07/03):** "Side project chạy được: network + cache + test + kiến trúc rõ ràng, đẩy lên GitHub" = hoàn thành M1–M17.

---

## 3. Cấu trúc thư mục đích (sau tuần 17)

```
RepoScope/
├── .github/workflows/ci.yml
├── .gitignore
├── README.md
├── docs/
│   ├── architecture.png            ← sơ đồ (tuần 18)
│   └── screenshots/
├── RepoScope.xcodeproj
├── RepoScope/
│   ├── App/
│   │   ├── AppDelegate.swift
│   │   ├── SceneDelegate.swift
│   │   ├── AppContainer.swift       ← DI container (tuần 15)
│   │   └── AppCoordinator.swift
│   ├── Domain/
│   │   ├── Models/                  ← Repository, Owner (thuần Swift, không import UIKit)
│   │   └── Protocols/               ← RepositoryServiceProtocol, FavoritesStoreProtocol
│   ├── Data/
│   │   ├── Networking/              ← Endpoint, APIClient, APIError
│   │   ├── Services/                ← GitHubRepositoryService, MockRepositoryService
│   │   ├── Persistence/             ← FavoriteEntity (SwiftData), FavoritesStore, ResponseCache
│   │   └── Security/                ← KeychainStore
│   ├── Features/
│   │   ├── Search/                  ← UIKit: SearchViewController, SearchViewModel, RepositoryCell
│   │   ├── Detail/                  ← SwiftUI: RepositoryDetailView, RepositoryDetailViewModel
│   │   └── Favorites/               ← SwiftUI: FavoritesView, FavoritesViewModel
│   ├── Shared/
│   │   ├── ImageLoading/            ← ImageCache, ImageLoader, Downsampler
│   │   ├── Extensions/
│   │   └── UI/                      ← component dùng chung (LoadingView, ErrorView)
│   └── Resources/
│       ├── Assets.xcassets
│       └── Mock/search_repositories.json
├── RepoScopeTests/
│   ├── Support/                     ← URLProtocolStub, fixtures JSON, factory model
│   ├── Networking/
│   ├── Features/
│   └── Persistence/
└── RepoScopeUITests/                ← tuỳ chọn: 1–2 UI test smoke
```

---

## 4. Code skeleton đã kiểm tra

> Các đoạn dưới đây đã được **compile và chạy thử** bằng Swift 6.3 ở **Swift 6 language mode** (phần UIKit được typecheck với iOS 17 SDK). Chúng là **khung**, không phải đáp án: **đọc → đóng file → tự gõ lại → so sánh**.

### 4.1. Model với `CodingKeys` (tuần 2 tạo struct, tuần 4 thêm Codable)

```swift
import Foundation

struct Owner: Codable, Hashable, Sendable {
    let id: Int
    let login: String
    let avatarURL: URL
    let htmlURL: URL

    enum CodingKeys: String, CodingKey {
        case id, login
        case avatarURL = "avatar_url"
        case htmlURL = "html_url"
    }
}

struct Repository: Codable, Hashable, Identifiable, Sendable {
    let id: Int
    let name: String
    let fullName: String
    let owner: Owner
    let description: String?          // GitHub trả null nếu repo không có mô tả
    let htmlURL: URL
    let stargazersCount: Int
    let forksCount: Int
    let openIssuesCount: Int
    let language: String?
    let topics: [String]?
    let updatedAt: Date

    enum CodingKeys: String, CodingKey {
        case id, name, owner, description, language, topics
        case fullName = "full_name"
        case htmlURL = "html_url"
        case stargazersCount = "stargazers_count"
        case forksCount = "forks_count"
        case openIssuesCount = "open_issues_count"
        case updatedAt = "updated_at"
    }
}

struct SearchResponse<Item: Decodable & Sendable>: Decodable, Sendable {
    let totalCount: Int
    let incompleteResults: Bool
    let items: [Item]

    enum CodingKeys: String, CodingKey {
        case items
        case totalCount = "total_count"
        case incompleteResults = "incomplete_results"
    }
}
```
- Cách khác: `decoder.keyDecodingStrategy = .convertFromSnakeCase` rồi bỏ `CodingKeys`. **Bẫy:** `avatar_url` sẽ thành `avatarUrl` (không phải `avatarURL`) → nếu đặt tên `avatarURL` thì decode fail. Viết `CodingKeys` tường minh để chủ động — và để trả lời được câu hỏi *"CodingKeys dùng làm gì?"*.
- JSON có nhiều field hơn model (ví dụ `watchers_count`) → **không sao**, Decodable bỏ qua field thừa. Model có field mà JSON **thiếu** và không Optional → **lỗi** `keyNotFound`.
- `Sendable` chỉ cần từ tuần 8; tuần 2–4 có thể chưa thêm.

### 4.2. `APIError` + `Endpoint`

```swift
enum APIError: Error, Equatable, Sendable {
    case invalidURL
    case transport(URLError.Code)          // mất mạng, timeout...
    case invalidResponse
    case httpStatus(Int)                   // 4xx/5xx khác
    case rateLimited(resetAt: Date?)       // 403/429 + x-ratelimit-remaining = 0
    case decoding(String)
}

struct Endpoint: Sendable {
    var path: String
    var queryItems: [URLQueryItem] = []
    var method: String = "GET"

    static let baseURL = URL(string: "https://api.github.com")!

    func makeRequest(token: String? = nil) throws(APIError) -> URLRequest {
        var components = URLComponents(url: Self.baseURL, resolvingAgainstBaseURL: false)
        components?.path = path
        components?.queryItems = queryItems.isEmpty ? nil : queryItems
        guard let url = components?.url else { throw .invalidURL }

        var request = URLRequest(url: url)
        request.httpMethod = method
        request.timeoutInterval = 15
        request.setValue("application/vnd.github+json", forHTTPHeaderField: "Accept")
        request.setValue("2022-11-28", forHTTPHeaderField: "X-GitHub-Api-Version")
        if let token { request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization") }
        return request
    }
}

extension Endpoint {
    static func searchRepositories(query: String, page: Int, perPage: Int = 30) -> Endpoint {
        Endpoint(path: "/search/repositories", queryItems: [
            URLQueryItem(name: "q", value: query),
            URLQueryItem(name: "sort", value: "stars"),
            URLQueryItem(name: "order", value: "desc"),
            URLQueryItem(name: "page", value: String(page)),
            URLQueryItem(name: "per_page", value: String(perPage)),
        ])
    }

    static func repository(owner: String, name: String) -> Endpoint {
        Endpoint(path: "/repos/\(owner)/\(name)")
    }

    static func readme(owner: String, name: String) -> Endpoint {
        Endpoint(path: "/repos/\(owner)/\(name)/readme")
    }
}
```
- `URLComponents` tự percent-encode query: `"swift language:swift"` → `q=swift%20language:swift`. **Đừng** ghép chuỗi URL bằng tay.
- `throws(APIError)` là **typed throws** (Swift 6). Tuần 4 có thể viết `throws` thường; nâng cấp ở tuần 8.

### 4.3. Tuần 4 — `APIClient` skeleton đọc JSON từ bundle (completion handler)

```swift
protocol LegacyAPIClient {
    func send<T: Decodable>(_ endpoint: Endpoint, as type: T.Type,
                            completion: @escaping (Result<T, APIError>) -> Void)
}

struct BundleAPIClient: LegacyAPIClient {
    var bundle: Bundle = .main
    /// map path của endpoint → tên file JSON trong bundle
    var fileForPath: (String) -> String? = { path in
        path == "/search/repositories" ? "search_repositories" : nil
    }

    func send<T: Decodable>(_ endpoint: Endpoint, as type: T.Type,
                            completion: @escaping (Result<T, APIError>) -> Void) {
        guard let name = fileForPath(endpoint.path),
              let url = bundle.url(forResource: name, withExtension: "json"),
              let data = try? Data(contentsOf: url) else {
            completion(.failure(.httpStatus(404)))
            return
        }
        do {
            let decoder = JSONDecoder()
            decoder.dateDecodingStrategy = .iso8601
            completion(.success(try decoder.decode(T.self, from: data)))
        } catch {
            completion(.failure(.decoding(String(describing: error))))
        }
    }
}
```
Cách lấy file JSON thật: mở `https://api.github.com/search/repositories?q=swift&per_page=5` trên trình duyệt → lưu thành `search_repositories.json` → kéo vào `Resources/Mock/` (tick "Add to target").

### 4.4. Tuần 7 — `APIClient` async/await (thay `BundleAPIClient`)

```swift
protocol APIClientProtocol: Sendable {
    func send<T: Decodable & Sendable>(_ endpoint: Endpoint, as type: T.Type) async throws(APIError) -> T
}

struct APIClient: APIClientProtocol {
    private let session: URLSession
    private let tokenProvider: @Sendable () -> String?

    init(session: URLSession = .shared, tokenProvider: @escaping @Sendable () -> String? = { nil }) {
        self.session = session
        self.tokenProvider = tokenProvider
    }

    static let decoder: JSONDecoder = {
        let d = JSONDecoder()
        d.dateDecodingStrategy = .iso8601        // "2026-11-09T08:00:00Z"
        return d
    }()

    func send<T: Decodable & Sendable>(_ endpoint: Endpoint, as type: T.Type) async throws(APIError) -> T {
        let request = try endpoint.makeRequest(token: tokenProvider())

        let data: Data, response: URLResponse
        do {
            (data, response) = try await session.data(for: request)
        } catch let error as URLError {
            throw .transport(error.code)
        } catch {
            throw .transport(.unknown)
        }

        guard let http = response as? HTTPURLResponse else { throw .invalidResponse }

        switch http.statusCode {
        case 200..<300:
            break
        case 403, 429:
            // 403 có thể là "hết quota" hoặc "không có quyền" → phải nhìn header
            let remaining = http.value(forHTTPHeaderField: "x-ratelimit-remaining")
            guard remaining == "0" || http.statusCode == 429 else { throw .httpStatus(http.statusCode) }
            let reset = http.value(forHTTPHeaderField: "x-ratelimit-reset")
                .flatMap(TimeInterval.init)
                .map(Date.init(timeIntervalSince1970:))
            throw .rateLimited(resetAt: reset)
        default:
            throw .httpStatus(http.statusCode)
        }

        do {
            return try Self.decoder.decode(T.self, from: data)
        } catch {
            throw .decoding(String(describing: error))
        }
    }
}

protocol RepositoryServiceProtocol: Sendable {
    func search(query: String, page: Int) async throws -> [Repository]
    func detail(owner: String, name: String) async throws -> Repository
}

struct GitHubRepositoryService: RepositoryServiceProtocol {
    let client: APIClientProtocol

    func search(query: String, page: Int) async throws -> [Repository] {
        try await client.send(.searchRepositories(query: query, page: page),
                              as: SearchResponse<Repository>.self).items
    }

    func detail(owner: String, name: String) async throws -> Repository {
        try await client.send(.repository(owner: owner, name: name), as: Repository.self)
    }
}
```
> **Bẫy đã gặp khi viết skeleton này:** ban đầu viết `case 403, 429 where remaining == "0":` — compiler cảnh báo *"'where' only applies to the second pattern match"*: `where` **chỉ áp cho 429**, mọi lỗi 403 đều bị coi là rate limit. Đọc kỹ warning! Đây là câu chuyện nhỏ hay để kể khi được hỏi "bug thú vị bạn từng gặp".

### 4.5. Tuần 3 — `MockRepositoryService` + dữ liệu mẫu

```swift
struct MockRepositoryService: RepositoryServiceProtocol {
    var repositories: [Repository] = Repository.samples
    var error: Error? = nil

    func search(query: String, page: Int) async throws -> [Repository] {
        if let error { throw error }
        guard !query.isEmpty else { return repositories }
        return repositories.filter { $0.name.localizedCaseInsensitiveContains(query) }
    }

    func detail(owner: String, name: String) async throws -> Repository {
        if let error { throw error }
        guard let repo = repositories.first(where: { $0.owner.login == owner && $0.name == name }) else {
            throw APIError.httpStatus(404)
        }
        return repo
    }
}
// Repository.samples: khai báo trong extension Repository { static let samples: [Repository] = [...] }
```
> Tuần 3 bạn **chưa học async** → viết phiên bản đồng bộ trước: `func search(query: String) -> [Repository]`. Tuần 7 đổi chữ ký protocol sang `async throws` — compiler sẽ chỉ ra mọi chỗ phải sửa. Đó chính là lợi ích của protocol.

### 4.6. Tuần 5–7 — Image cache & loader

```swift
import UIKit
import ImageIO

// W5 — ImageCache bọc NSCache (NSCache đã thread-safe sẵn)
final class ImageCache: @unchecked Sendable {   // @unchecked: NSCache tự đồng bộ nội bộ
    static let shared = ImageCache()
    private let cache = NSCache<NSURL, UIImage>()

    init(countLimit: Int = 200, totalCostLimit: Int = 50 * 1024 * 1024) {
        cache.countLimit = countLimit
        cache.totalCostLimit = totalCostLimit       // ~50 MB
    }

    func image(for url: URL) -> UIImage? { cache.object(forKey: url as NSURL) }

    func insert(_ image: UIImage, for url: URL) {
        let cost = image.cgImage.map { $0.bytesPerRow * $0.height } ?? 0
        cache.setObject(image, forKey: url as NSURL, cost: cost)
    }
}
```

```swift
// W6 — cache thread-safe tự viết bằng concurrent queue + barrier (reader–writer)
final class ThreadSafeCache<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "repo.scope.cache", attributes: .concurrent)

    func value(for key: Key) -> Value? {
        queue.sync { storage[key] }                         // đọc song song
    }

    func set(_ value: Value, for key: Key) {
        queue.async(flags: .barrier) { self.storage[key] = value }   // ghi độc quyền
    }
}
```

```swift
// W7 — actor ImageLoader: gộp request trùng (in-flight dedup) + cache
// W10 — thêm downsampling
actor ImageLoader {
    static let shared = ImageLoader()

    private let session: URLSession
    private let cache: ImageCache
    private var inFlight: [URL: Task<UIImage, Error>] = [:]

    init(session: URLSession = .shared, cache: ImageCache = .shared) {
        self.session = session
        self.cache = cache
    }

    func image(for url: URL, targetSize: CGSize, scale: CGFloat) async throws -> UIImage {
        if let cached = cache.image(for: url) { return cached }
        if let running = inFlight[url] { return try await running.value }   // đã có người tải

        let session = self.session
        let task = Task<UIImage, Error> {
            let (data, response) = try await session.data(from: url)
            guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw URLError(.badServerResponse) }
            guard let image = Self.downsample(data: data, to: targetSize, scale: scale) else {
                throw URLError(.cannotDecodeContentData)
            }
            return image
        }
        inFlight[url] = task
        defer { inFlight[url] = nil }            // chạy sau khi await xong (kể cả throw)

        let image = try await task.value
        cache.insert(image, for: url)
        return image
    }

    // Downsampling bằng ImageIO: không decode ảnh full-size vào RAM
    nonisolated static func downsample(data: Data, to pointSize: CGSize, scale: CGFloat) -> UIImage? {
        let sourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
        guard let source = CGImageSourceCreateWithData(data as CFData, sourceOptions) else { return nil }
        let maxPixel = max(pointSize.width, pointSize.height) * scale
        let options = [
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            kCGImageSourceShouldCacheImmediately: true,      // decode ngay, ở background
            kCGImageSourceCreateThumbnailWithTransform: true,
            kCGImageSourceThumbnailMaxPixelSize: maxPixel,
        ] as CFDictionary
        guard let cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, options) else { return nil }
        return UIImage(cgImage: cgImage)
    }
}
```
- **Actor reentrancy** (câu hỏi phỏng vấn hay): sau mỗi `await` trong actor, trạng thái có thể đã đổi. Vì vậy phải **ghi `inFlight[url]` trước khi `await`**, để lời gọi thứ hai (đến trong lúc lời gọi đầu đang chờ) thấy task và dùng chung.
- Cache key là URL **không gồm kích thước** → nếu cùng một avatar hiển thị 2 cỡ, nên đưa kích thước vào key. Ghi lại thành "known limitation" trong README.

### 4.7. Tuần 8 — Debounce search

```swift
// Cách 1: Task + sleep + cancel — đơn giản, không cần Combine
@MainActor
final class SearchDebouncer {
    private var task: Task<Void, Never>?
    private let delay: Duration

    init(delay: Duration = .milliseconds(400)) { self.delay = delay }

    func submit(_ action: @escaping @MainActor () async -> Void) {
        task?.cancel()                                   // huỷ lần gõ trước
        task = Task { [delay] in
            try? await Task.sleep(for: delay)
            guard !Task.isCancelled else { return }
            await action()
        }
    }
}

// Cách 2: AsyncStream — biến text field thành một luồng giá trị
@MainActor
final class SearchQueryStream {
    let stream: AsyncStream<String>
    private let continuation: AsyncStream<String>.Continuation

    init() {
        (stream, continuation) = AsyncStream.makeStream(of: String.self, bufferingPolicy: .bufferingNewest(1))
    }
    func send(_ query: String) { continuation.yield(query) }
    deinit { continuation.finish() }
}
```
Cách 3 (Combine): `@Published var query` → `.debounce(for: .milliseconds(400), scheduler: RunLoop.main).removeDuplicates().sink { ... }`. **Làm cả cách 1 và 3**, rồi viết vào README vì sao chọn cách cuối cùng.

### 4.8. Tuần 16 — `URLProtocolStub` cho unit test networking

```swift
final class URLProtocolStub: URLProtocol, @unchecked Sendable {
    struct Stub: Sendable { let data: Data; let statusCode: Int; let headers: [String: String] }
    nonisolated(unsafe) static var stub: Stub?          // chỉ dùng trong test, chạy tuần tự
    nonisolated(unsafe) static var lastRequest: URLRequest?

    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }

    override func startLoading() {
        Self.lastRequest = request
        guard let stub = Self.stub, let url = request.url else {
            client?.urlProtocol(self, didFailWithError: URLError(.badServerResponse)); return
        }
        let response = HTTPURLResponse(url: url, statusCode: stub.statusCode,
                                       httpVersion: nil, headerFields: stub.headers)!
        client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
        client?.urlProtocol(self, didLoad: stub.data)
        client?.urlProtocolDidFinishLoading(self)
    }
    override func stopLoading() {}

    static func makeSession() -> URLSession {
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [URLProtocolStub.self]
        return URLSession(configuration: config)
    }
}
```

Ví dụ test (Swift Testing, Xcode 16+):
```swift
import Testing
@testable import RepoScope

@Suite(.serialized)          // stub là static → không chạy song song
struct GitHubRepositoryServiceTests {
    let sut = GitHubRepositoryService(client: APIClient(session: URLProtocolStub.makeSession()))

    @Test func search_whenRateLimited_throwsRateLimited() async throws {
        URLProtocolStub.stub = .init(data: Data(), statusCode: 403,
                                     headers: ["x-ratelimit-remaining": "0", "x-ratelimit-reset": "1800000000"])
        await #expect(throws: APIError.rateLimited(resetAt: Date(timeIntervalSince1970: 1_800_000_000))) {
            try await sut.search(query: "swift", page: 1)
        }
    }

    @Test func search_buildsCorrectQuery() async throws {
        URLProtocolStub.stub = .init(data: Data(#"{"total_count":0,"incomplete_results":false,"items":[]}"#.utf8),
                                     statusCode: 200, headers: [:])
        let repos = try await sut.search(query: "swift", page: 3)
        #expect(repos.isEmpty)
        let url = try #require(URLProtocolStub.lastRequest?.url)
        #expect(url.path == "/search/repositories")
        #expect(url.query?.contains("page=3") == true)
    }
}
```
> Trong skeleton đã chạy thử, các case sau đều được kiểm tra: decode JSON mẫu (có `description: null`), URL search có `per_page=30`, 403 + `remaining=0` → `.rateLimited`, body sai → `.decoding`, 404 → `.httpStatus(404)`, mock lọc theo tên.

### 4.9. Tuần 17 — Keychain tối thiểu cho token

```swift
enum KeychainStore {
    static let service = "com.example.RepoScope"

    static func save(_ value: String, account: String) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
        ]
        SecItemDelete(query as CFDictionary)              // xoá bản cũ nếu có
        var attributes = query
        attributes[kSecValueData as String] = Data(value.utf8)
        attributes[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlock
        return SecItemAdd(attributes as CFDictionary, nil) == errSecSuccess
    }

    static func read(account: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
        ]
        var result: AnyObject?
        guard SecItemCopyMatching(query as CFDictionary, &result) == errSecSuccess,
              let data = result as? Data else { return nil }
        return String(data: data, encoding: .utf8)
    }
}
```
> **Tuyệt đối không commit token** vào repo (kể cả trong `xcconfig`). Token chỉ được người dùng **tự nhập** trong màn Settings của app → lưu Keychain. Nếu lỡ commit: **revoke token ngay trên GitHub**, xoá khỏi lịch sử không đủ.

---

## 5. Chi tiết từng milestone

Mỗi milestone gồm: 🎯 mục tiêu · 📋 việc cần làm · ✅ Definition of Done · 💡 gợi ý · 🎤 chủ đề phỏng vấn chứng minh được · 📝 commit gợi ý.

---

### M1 — Tuần 1 (T7 14/11/2026): Setup repo, Xcode project

🎯 Có repo public trên GitHub, project chạy được màn hình trống.

📋 Việc cần làm
1. Tạo repo `RepoScope` trên GitHub (public, chưa thêm README từ web để tránh conflict).
2. Xcode → New Project → App · Interface: **Storyboard** (sẽ xoá ở tuần 9) · Language: Swift · tick **Include Tests**. Bundle ID: `com.<tên-bạn>.RepoScope`. Minimum deployment: **iOS 17**.
3. Tạo `.gitignore` (mẫu dưới) **trước** commit đầu tiên.
4. Tạo `README.md` 5 dòng: tên app, mô tả 1 câu, mục "Roadmap" liệt kê M1–M18, mục "Rules: no AI-generated code".
5. Tắt Predictive Code Completion: Xcode → Settings → Text Editing → Editing.
6. `git init`, commit, `git remote add origin ...`, `git push -u origin main`.

```gitignore
# Xcode
DerivedData/
build/
*.xcuserstate
xcuserdata/
*.xcscmblueprint
*.moved-aside
# SwiftPM
.build/
.swiftpm/
# macOS
.DS_Store
# Secrets — không bao giờ commit
*.xcconfig.local
Secrets.swift
```

✅ DoD
- [ ] Repo public có ≥ 2 commit, `.gitignore` đúng (không có `xcuserdata` trong repo — kiểm tra bằng `git ls-files | grep xcuserdata` ra rỗng).
- [ ] App chạy được trên simulator, hiển thị label "RepoScope".
- [ ] README có roadmap.

💡 Gợi ý: nếu lỡ commit `xcuserdata`, dùng `git rm -r --cached <path>` rồi commit lại.
🎤 Chứng minh: Git cơ bản, cấu trúc project Xcode (target, scheme, bundle ID).
📝 `chore: initial Xcode project with gitignore and README`

---

### M2 — Tuần 2 (T7 21/11): Model layer, mock data, value semantics

🎯 Có `Repository`, `Owner` là **struct**; hiểu vì sao chọn struct.

📋 Việc cần làm
1. Tạo `Domain/Models/Owner.swift`, `Repository.swift` (như §4.1 nhưng **chưa cần** `Codable`/`CodingKeys`).
2. `extension Repository { static let samples: [Repository] }` với 5 repo thật (copy số liệu từ github.com).
3. Thêm computed property: `formattedStars` (`"68k"`, `"1.2k"`, `"950"`), `isPopular` (`stars > 10_000`).
4. Viết **3 unit test đầu tiên**: `formattedStars` cho 3 khoảng giá trị.
5. Thử nghiệm value semantics trong test: copy một `Repository`, đổi… — bạn sẽ thấy `let` không cho đổi. Tạm đổi 1 property thành `var`, chứng minh bản copy không ảnh hưởng bản gốc. Ghi kết quả vào comment.

✅ DoD
- [ ] Model là struct, property `let` (trừ khi có lý do), conform `Hashable`, `Identifiable`.
- [ ] 3 test xanh (`⌘U`).
- [ ] Viết được 3 câu trong README mục "Decisions": *vì sao model là struct*.

💡 Gợi ý: `Identifiable` cần `id` — GitHub đã có `id: Int`. Format số: `Double(stars) / 1000` + `String(format: "%.1fk", …)` hoặc `formatted(.number.notation(.compactName))` (iOS 15+).
🎤 Chứng minh: struct vs class, value vs reference, `let` với struct, Hashable/Identifiable, copy-on-write (mảng `samples`).
📝 `feat(model): add Repository and Owner value types with samples`

---

### M3 — Tuần 3 (T7 28/11): Protocol + Mock service, extensions

🎯 Màn hình (tạm) gọi dữ liệu **qua protocol**, không biết dữ liệu từ đâu.

📋 Việc cần làm
1. `Domain/Protocols/RepositoryServiceProtocol.swift` — bản đồng bộ tuần này: `func search(query: String) -> [Repository]`.
2. `Data/Services/MockRepositoryService.swift` lọc `samples` theo tên (không phân biệt hoa thường).
3. `ViewController` giữ `let service: RepositoryServiceProtocol` (inject qua init hoặc property) — in kết quả ra console là đủ.
4. Extensions: `extension Repository: CustomStringConvertible`, `extension Int { var compactFormatted: String }` (chuyển logic `formattedStars` ra đây), `extension Array where Element == Repository { func sortedByStars() -> [Repository] }`.
5. Dùng closure: `sorted(by:)`, `filter`, `map` — ít nhất 1 chỗ viết closure đầy đủ, 1 chỗ dùng `$0`.
6. 3 test mới cho `MockRepositoryService` (query rỗng, có kết quả, không có kết quả).

✅ DoD
- [ ] Không chỗ nào ngoài `MockRepositoryService` biết tới `Repository.samples`.
- [ ] Đổi sang mock khác (ví dụ `EmptyRepositoryService`) chỉ cần sửa 1 dòng.
- [ ] 6 test xanh.

💡 Gợi ý: protocol trong Swift không có default argument → nếu muốn `page: Int = 1`, viết extension protocol với hàm overload.
🎤 Chứng minh: protocol-oriented programming, dependency inversion, extension có điều kiện (`where`), mock để test.
📝 `feat(data): add RepositoryServiceProtocol and MockRepositoryService`

---

### M4 — Tuần 4 (T7 05/12): Codable, APIError, APIClient generic

🎯 Dữ liệu đi từ **file JSON thật của GitHub** → model, qua một `APIClient` generic.

📋 Việc cần làm
1. Lưu JSON thật (xem §4.3) vào `Resources/Mock/search_repositories.json`.
2. Thêm `Codable` + `CodingKeys` cho model (§4.1), `SearchResponse<Item>` generic.
3. `APIError` enum (§4.2) — tuần này có thể chưa cần `rateLimited`.
4. `Endpoint` + `BundleAPIClient` (§4.3) với chữ ký generic `send<T: Decodable>`.
5. `BundleRepositoryService: RepositoryServiceProtocol` dùng `BundleAPIClient`.
6. Test: decode file JSON thật → `items.count > 0`; decode JSON hỏng → `.decoding`; decode JSON có `description: null` → không lỗi.

✅ DoD
- [ ] App in ra 30 repo đọc từ bundle.
- [ ] `APIClient` không biết gì về `Repository` (generic thật sự).
- [ ] 9+ test xanh.

💡 Gợi ý: lỗi decode khó đọc → in `DecodingError` chi tiết: `case .keyNotFound(let key, let ctx)` → `ctx.codingPath`. Ngày dạng `"2026-11-09T08:00:00Z"` → `.iso8601`.
🎤 Chứng minh: generics + constraint, Codable/CodingKeys, error handling bằng enum, `Result`.
📝 `feat(networking): decode GitHub search JSON via generic APIClient`

---

### M5 — Tuần 5 (T7 12/12): Memory Graph, ImageCache với NSCache

🎯 Chứng minh app **không leak**, có cache ảnh trong bộ nhớ.

📋 Việc cần làm
1. Tạo tạm `DetailViewController` (UIKit, hiển thị tên repo) và push từ màn hiện tại.
2. **Cố ý tạo retain cycle**: VC giữ closure `onFinish`, closure dùng `self` mạnh. Push/pop 5 lần → mở **Debug Memory Graph** → thấy 5 instance `DetailViewController` → chụp màn hình lưu vào `docs/`.
3. Sửa bằng `[weak self]` → kiểm tra lại còn 0 instance. Thêm `deinit { print("DetailVC deinit") }`.
4. `ImageCache` với `NSCache` (§4.6), có `countLimit` và `totalCostLimit`.
5. Tải avatar **đồng bộ tạm thời** bằng `Data(contentsOf:)` (sẽ thấy UI giật — ghi lại, sửa ở M6).

✅ DoD
- [ ] Ảnh chụp Memory Graph trước/sau trong `docs/memory/`.
- [ ] `deinit` in ra khi pop.
- [ ] Test `ImageCache`: insert rồi get trả về cùng instance (`===`).

💡 Gợi ý: bật **Malloc Stack Logging** (Scheme → Diagnostics) để Memory Graph cho biết object được tạo ở đâu. `NSCache` key phải là class → dùng `NSURL`.
🎤 Chứng minh: ARC, retain cycle với closure, `weak` vs `unowned`, Memory Graph, `NSCache` vs `Dictionary` (tự evict khi memory warning, thread-safe).
📝 `fix(detail): break retain cycle in onFinish closure` · `feat(image): add NSCache-backed ImageCache`

---

### M6 — Tuần 6 (T7 19/12): Tải ảnh bằng GCD + cache thread-safe

🎯 Ảnh tải ở background, cập nhật UI ở main thread, không race.

📋 Việc cần làm
1. `ImageLoaderGCD`: `func load(_ url: URL, completion: @escaping (UIImage?) -> Void)` — tải bằng `URLSession.dataTask` (hoặc `DispatchQueue.global().async`), **gọi completion trên main**.
2. Viết `ThreadSafeCache` (§4.6) bằng concurrent queue + barrier để lưu **metadata** (ví dụ URL đã lỗi → không thử lại). So sánh: vì sao `NSCache` không cần barrier?
3. Gộp request trùng: nếu URL đang tải, lưu thêm completion vào `[URL: [(UIImage?) -> Void]]` (đây là phần dễ race nhất — bảo vệ bằng serial queue).
4. Bật **Thread Sanitizer** (Scheme → Diagnostics → TSan), chạy, đảm bảo không cảnh báo.
5. Test: gọi `load` 100 lần song song cùng URL (dùng `DispatchQueue.concurrentPerform`) → server stub chỉ nhận 1 request.

✅ DoD
- [ ] Scroll/thao tác không giật khi tải avatar.
- [ ] TSan sạch.
- [ ] Không có `DispatchQueue.main.sync` nào gọi từ main (deadlock).

💡 Gợi ý: cell bị reuse → khi ảnh về, kiểm tra URL hiện tại của cell còn khớp không trước khi gán.
🎤 Chứng minh: GCD, serial vs concurrent, barrier, race condition, deadlock, Thread Sanitizer, main thread cho UI.
📝 `feat(image): load images on background queue with thread-safe cache`

---

### M7 — Tuần 7 (T7 26/12): async/await + GitHub API thật + actor

🎯 App gọi **GitHub API thật**; networking và image loading viết bằng async/await.

📋 Việc cần làm
1. Đổi `RepositoryServiceProtocol` sang `async throws` (§4.4). Để compiler dẫn đường sửa mọi chỗ gọi.
2. `APIClient` thật với `URLSession.data(for:)`, xử lý status code + rate limit.
3. `GitHubRepositoryService`; giữ `MockRepositoryService` cho preview/test.
4. `ImageLoader` thành `actor` với in-flight dedup (§4.6) — bỏ code GCD (giữ lại ở branch/tag `m06-gcd` để so sánh khi phỏng vấn).
5. Gọi từ VC: `Task { @MainActor in ... }`; huỷ task khi VC biến mất (`viewDidDisappear` hoặc lưu `Task` và `cancel()`).
6. Tải song song detail + README bằng `async let`.

✅ DoD
- [ ] Tìm "swift" ra kết quả thật từ API.
- [ ] Tắt mạng (Network Link Conditioner hoặc tắt Wi-Fi máy Mac) → hiện thông báo lỗi tiếng Việt dễ hiểu, không crash.
- [ ] Hết quota → hiện "Thử lại sau HH:mm".
- [ ] Không còn completion handler trong tầng networking.

💡 Gợi ý: gọi `GET /rate_limit` để xem còn bao nhiêu lượt. Giới hạn search không token là 10 request/phút — test nhẹ tay.
🎤 Chứng minh: async/await vs completion handler, `Task`, cancellation, `async let`, actor + reentrancy, `@MainActor`.
📝 `feat(networking): migrate to async/await with live GitHub API` · `refactor(image): convert ImageLoader to actor`

---

### M8 — Tuần 8 (T7 02/01/2027): Swift 6 language mode, Sendable, debounce

🎯 Project build ở **Swift 6 language mode** không warning concurrency; search có debounce.

📋 Việc cần làm
1. Build Settings → **Swift Language Version = Swift 6** (hoặc trước đó bật *Strict Concurrency Checking = Complete* để thấy warning).
2. Sửa từng lỗi: thêm `Sendable` cho model, `@MainActor` cho VC/VM, `@Sendable` cho closure, xem xét `@unchecked Sendable` **chỉ** khi chứng minh được an toàn (ghi comment lý do).
3. Debounce search 400 ms (§4.7) — làm bằng Task, sau đó thử Combine.
4. Huỷ request cũ khi query mới đến (nếu không, kết quả cũ có thể về **sau** kết quả mới và ghi đè — race điển hình).
5. Bỏ query rỗng / trùng query trước đó.

✅ DoD
- [ ] 0 warning, 0 error trong Swift 6 mode.
- [ ] Gõ "swiftui" nhanh → chỉ 1 request (kiểm tra bằng log hoặc Proxyman/Charles).
- [ ] Ghi trong README "Concurrency notes": mỗi `@unchecked Sendable` vì sao an toàn.

💡 Gợi ý: lỗi `"Capture of 'self' with non-sendable type in a @Sendable closure"` → thường do class chưa được isolate (`@MainActor`) hoặc chưa Sendable.
🎤 Chứng minh: Sendable, data race safety của Swift 6, actor isolation, AsyncStream, Combine `debounce`, cancellation.
📝 `chore: enable Swift 6 language mode and fix Sendable diagnostics` · `feat(search): debounce queries and cancel stale requests`

---

### M9 — Tuần 9 (T7 09/01): Màn list UIKit (programmatic), lifecycle, điều hướng

🎯 Màn Search hoàn toàn bằng code, có search bar, list cơ bản, bấm vào sang detail.

📋 Việc cần làm
1. **Xoá storyboard**: xoá `Main.storyboard`, xoá key trong Info.plist (`UISceneStoryboardFile`, "Main storyboard file base name"), tạo `window` trong `SceneDelegate`.
2. `SearchViewController` + `UISearchController` gắn vào `navigationItem`.
3. Tạm dùng `UITableView` (tuần 10 đổi sang collection view) với cell tuỳ chỉnh `RepositoryCell` (Auto Layout bằng code: `translatesAutoresizingMaskIntoConstraints = false`, `NSLayoutConstraint.activate`).
4. Log mọi method lifecycle (`viewDidLoad`, `viewWillAppear`, `viewIsAppearing`, `viewDidAppear`, `viewWillDisappear`, `viewDidDisappear`, `deinit`) → push/pop và ghi lại thứ tự vào `docs/lifecycle.md`.
5. Trạng thái màn hình: `enum ViewState { case idle, loading, loaded([Repository]), empty, error(String) }`.
6. `didSelectRowAt` → push `DetailViewController` tạm (tuần 11 thay bằng SwiftUI).

✅ DoD
- [ ] Không còn storyboard nào ngoài LaunchScreen.
- [ ] Đủ 5 trạng thái hiển thị đúng (loading spinner, empty "Không tìm thấy", error có nút "Thử lại").
- [ ] Bảng thứ tự lifecycle viết tay trong `docs/lifecycle.md`.

💡 Gợi ý: `viewIsAppearing` (iOS 17, back-deploy iOS 13) là nơi tốt để cập nhật UI phụ thuộc trait/kích thước.
🎤 Chứng minh: VC lifecycle, App/Scene lifecycle, Auto Layout bằng code, navigation, state-driven UI.
📝 `feat(search): programmatic UIKit search screen with state handling`

---

### M10 — Tuần 10 (T7 16/01): Collection view compositional + diffable, pagination, prefetch, downsampling

🎯 List cuộn mượt 60/120 fps, tự tải trang tiếp, ảnh không ngốn RAM.

📋 Việc cần làm
1. Đổi sang `UICollectionView` + `UICollectionViewCompositionalLayout` (`.list(using:)` hoặc layout tự định nghĩa).
2. `UICollectionViewDiffableDataSource<Section, Repository.ID>` — dùng **ID** làm item identifier, lấy model từ dictionary/store (khuyến nghị của Apple; tránh lỗi khi model đổi nội dung). Cập nhật bằng `snapshot.reconfigureItems`.
3. **Pagination**: khi hiển thị item gần cuối (`willDisplay` hoặc prefetch), gọi trang `page + 1`. Chặn gọi trùng bằng cờ `isLoadingNextPage`; dừng khi `items.count < perPage` hoặc chạm 1.000 kết quả.
4. **Prefetch**: `UICollectionViewDataSourcePrefetching` → bắt đầu tải avatar sớm; `cancelPrefetching` → huỷ task.
5. **Downsampling** avatar về kích thước hiển thị (§4.6).
6. Cell: `prepareForReuse` huỷ task ảnh, xoá ảnh cũ.
7. Đo bằng Instruments → **Allocations**: so sánh bộ nhớ trước/sau downsampling, ghi số liệu vào README.

✅ DoD
- [ ] Cuộn 300 item (10 trang) không giật, không lặp item, không hiện ảnh sai cell.
- [ ] Có footer "Đang tải thêm…" và trạng thái lỗi ở cuối list có nút thử lại.
- [ ] Số liệu bộ nhớ trước/sau downsampling trong README.

💡 Gợi ý: GitHub đôi khi trả **cùng repo ở 2 trang** (do thứ hạng thay đổi) → diffable crash vì ID trùng! Lọc trùng trước khi `appendItems`. Đây là một **bug thật đáng kể** trong phỏng vấn.
🎤 Chứng minh: cell reuse, diffable data source, compositional layout, pagination, prefetching, image downsampling, Instruments.
📝 `feat(search): compositional layout with diffable data source and pagination` · `perf(image): downsample avatars to display size`

---

### M11 — Tuần 11 (T7 23/01): Màn detail SwiftUI với `@Observable`

🎯 Màn detail viết bằng SwiftUI, state rõ ràng, có preview dùng mock.

📋 Việc cần làm
1. `RepositoryDetailViewModel` dùng macro `@Observable` (iOS 17), `@MainActor`, có `state: LoadState<Repository>` và `func load() async`.
2. `RepositoryDetailView`: header (avatar, tên, owner), thống kê (stars, forks, issues, language), topics (dạng chip), nút "Mở trên GitHub" (`Link`/`openURL`), nút ⭐ yêu thích (tạm lưu in-memory).
3. `.task { await viewModel.load() }` — tự huỷ khi view biến mất.
4. Avatar: thử `AsyncImage`, rồi viết `RemoteImage` view dùng `ImageLoader` của bạn — so sánh (`AsyncImage` không có memory cache ảnh đã decode, không downsample; mỗi lần view xuất hiện lại sẽ tải/decode lại, may lắm thì trúng `URLCache`).
5. `#Preview` với `MockRepositoryService` cho 3 trạng thái: loading, loaded, error.
6. (Tuỳ chọn) README: decode Base64 `content`, hiển thị bằng `Text(AttributedString(markdown:))` cho Markdown đơn giản.

✅ DoD
- [ ] 3 preview chạy được không cần mạng.
- [ ] Không có `@State` giữ dữ liệu lấy từ network ở view (thuộc về VM).
- [ ] Tạm thời mở từ UIKit bằng `UIHostingController` (chuẩn bị cho M12).

💡 Gợi ý: `@State private var viewModel: RepositoryDetailViewModel` khi view **sở hữu** VM; truyền vào bằng init.
🎤 Chứng minh: SwiftUI data flow, `@Observable` vs `ObservableObject`, `.task`, preview-driven development.
📝 `feat(detail): SwiftUI detail screen backed by @Observable view model`

---

### M12 — Tuần 12 (T7 30/01): NavigationStack + interop UIKit/SwiftUI

🎯 App có 2 tab: **Search (UIKit)** và **Favorites (SwiftUI)**, điều hướng 2 chiều.

📋 Việc cần làm
1. `UITabBarController`: tab 1 `UINavigationController(SearchViewController)`, tab 2 `UIHostingController(rootView: FavoritesView())`.
2. `FavoritesView` dùng `NavigationStack(path:)` + `navigationDestination(for: Repository.ID.self)` → mở `RepositoryDetailView`.
3. Từ UIKit push SwiftUI: `navigationController?.pushViewController(UIHostingController(rootView: RepositoryDetailView(...)), animated: true)`.
4. Ngược lại: nhúng một `UIView` UIKit vào SwiftUI bằng `UIViewRepresentable` (ví dụ `UIActivityIndicatorView` hoặc `WKWebView` để xem README đầy đủ).
5. Deep link nhỏ: `reposcope://repo/{owner}/{name}` → mở thẳng detail (xử lý trong `SceneDelegate`).
6. Kiểm tra identity: trong `ForEach` dùng `id: \.id`, không dùng `\.self` hay index.

✅ DoD
- [ ] Điều hướng UIKit → SwiftUI → UIKit không lỗi back button, không leak (Memory Graph).
- [ ] Deep link mở đúng màn (`xcrun simctl openurl booted "reposcope://repo/apple/swift"`).

💡 Gợi ý: SwiftUI view mới tạo lại liên tục; giữ VM trong `@State` để không bị reset khi parent re-render.
🎤 Chứng minh: `NavigationStack` path-based, `UIHostingController`, `UIViewRepresentable`, view identity, deep link.
📝 `feat(app): tab bar with UIKit search and SwiftUI favorites` · `feat(app): handle reposcope:// deep links`

---

### M13–14 — Tuần 13–14 (06/02, 13/02): 🧧 Nghỉ Tết · README có ảnh chụp màn hình

🎯 Không viết tính năng. Làm repo **trông chuyên nghiệp**.

📋 Việc cần làm
1. Chụp 4 ảnh (Search, Search đang tải thêm, Detail, Favorites) — light + dark — `xcrun simctl io booted screenshot docs/screenshots/search.png`.
2. Quay 1 GIF/video ngắn 10–15 giây cuộn list.
3. Cập nhật README theo template ở §6 (chưa cần phần Architecture/CI).
4. Ghi mục **"Known issues"** — bug bạn biết mà chưa sửa (interviewer đánh giá cao sự trung thực).
5. Dọn `TODO`, code comment thừa, file không dùng.

✅ DoD
- [ ] Người lạ đọc README trong 1 phút hiểu app làm gì, chạy thế nào.
- [ ] Không còn warning trong build.

🎤 Chứng minh: khả năng viết tài liệu, tự đánh giá.
📝 `docs: add screenshots and update README`

---

### M15 — Tuần 15 (T7 20/02): MVVM + Clean-ish layers + DI + Coordinator

🎯 Kiến trúc rõ ràng, mỗi lớp có một trách nhiệm, phụ thuộc đi **một chiều**.

📋 Việc cần làm
1. Sắp lại thư mục theo §3: `Domain` (model + protocol, không import UIKit/SwiftUI) · `Data` (implementation) · `Features` (UI + VM) · `App` (lắp ráp).
2. `SearchViewModel` (UIKit dùng): VC chỉ render state + chuyển event; logic search/pagination/debounce nằm trong VM. VM **không import UIKit**.
3. **DI bằng constructor**: `AppContainer` tạo `APIClient`, `GitHubRepositoryService`, `ImageLoader`, `FavoritesStore` một lần, truyền xuống qua init. Không gọi `.shared` trong Feature.
4. **Coordinator**: `AppCoordinator` → `SearchCoordinator` quyết định điều hướng (`showDetail(repo)`); VC/VM báo sự kiện qua closure/delegate (`onSelectRepository`), không tự `push`.
5. (Tuỳ chọn, Clean-ish) 1 use case: `SearchRepositoriesUseCase` gom logic lọc trùng + pagination → test dễ hơn.
6. Viết **ADR** ngắn `docs/adr/001-mvvm-coordinator.md`: bối cảnh, lựa chọn (MVC/MVVM/VIPER/TCA), quyết định, hệ quả.

✅ DoD
- [ ] `grep -r "import UIKit" RepoScope/Domain` → rỗng.
- [ ] Mỗi VM có ít nhất 2 test chạy với mock service (không mạng).
- [ ] Không VC nào gọi `pushViewController` trực tiếp (trừ coordinator).

💡 Gợi ý: Coordinator giữ `UINavigationController` **strong**; VC giữ coordinator (nếu có) phải **weak** → tránh retain cycle. Đừng over-engineer: 2 màn thì không cần VIPER.
🎤 Chứng minh: MVVM, SOLID (đặc biệt SRP, DIP), DI, Coordinator, trade-off kiến trúc.
📝 `refactor: introduce MVVM view models and coordinators` · `refactor(app): constructor-based DI via AppContainer`

---

### M16 — Tuần 16 (T7 27/02): Favorites SwiftData, offline cache, ≥ 15 unit test

🎯 Favorites lưu bền vững và xem được **khi không có mạng**; có bộ test đáng tin.

📋 Việc cần làm
1. `@Model final class FavoriteRepository` (SwiftData): lưu đủ field để hiển thị offline (id, fullName, description, stars, language, avatarURL, savedAt).
2. `FavoritesStoreProtocol` (Domain) + `SwiftDataFavoritesStore` (Data) — Feature chỉ biết protocol. Nếu chọn Core Data, giữ nguyên protocol, đổi implementation.
3. Offline cache cho search: lưu kết quả trang 1 của 5 query gần nhất (file JSON trong `Caches/` hoặc `URLCache` với `.returnCacheDataElseLoad` khi offline). Khi mất mạng hiển thị banner "Đang xem dữ liệu đã lưu".
4. Mạng: dùng `NWPathMonitor` để biết online/offline.
5. **Test ≥ 15**, gợi ý phân bổ:
   - Networking (URLProtocolStub, §4.8): 200 OK, 404, 403 rate limit, 403 không phải rate limit, JSON hỏng, mất mạng (`URLError.notConnectedToInternet`), query encode đúng → **7**
   - `SearchViewModel`: loading → loaded, empty, error, pagination không gọi trùng, lọc ID trùng giữa 2 trang, debounce chỉ gọi 1 lần → **6**
   - `FavoritesStore` với `ModelConfiguration(isStoredInMemoryOnly: true)`: thêm, xoá, toggle, không thêm trùng → **4**
   - Model: `formattedStars` (đã có) → **3**

✅ DoD
- [ ] Thêm favorite → tắt app → bật chế độ máy bay → mở app → vẫn thấy favorites.
- [ ] ≥ 15 test xanh, chạy < 10 giây, **không gọi mạng thật** (rút cáp vẫn xanh).
- [ ] Code coverage tầng Data + VM ≥ 60% (Scheme → Test → Options → Code Coverage).

💡 Gợi ý: test async VM — inject `MockRepositoryService` và `delay` có thể cấu hình; debounce test thì inject `Duration` nhỏ (ví dụ 10ms) thay vì 400ms.
🎤 Chứng minh: SwiftData/Core Data, chiến lược offline, test double (mock/stub/fake), URLProtocol, testable architecture.
📝 `feat(favorites): persist favorites with SwiftData` · `feat(search): offline cache for recent queries` · `test: add networking and view model tests`

---

### M17 — Tuần 17 (T7 06/03): CI, Instruments, Keychain, release config

🎯 Mỗi PR tự chạy test; app đã được profile; cấu hình release đúng.

📋 Việc cần làm
1. **GitHub Actions** `.github/workflows/ci.yml`:
   ```yaml
   name: CI
   on:
     pull_request:
     push:
       branches: [main]
   jobs:
     test:
       runs-on: macos-15
       steps:
         - uses: actions/checkout@v4
         - name: Show Xcode & simulators
           run: |
             xcodebuild -version
             xcrun simctl list devices available | head -40
         - name: Test
           run: |
             xcodebuild test \
               -project RepoScope.xcodeproj \
               -scheme RepoScope \
               -destination 'platform=iOS Simulator,name=iPhone 16' \
               CODE_SIGNING_ALLOWED=NO
   ```
   Tên simulator (`iPhone 16`) phụ thuộc phiên bản Xcode trên runner — nếu job báo không tìm thấy destination, xem output bước "Show Xcode & simulators" rồi sửa. Thêm badge CI vào README.
2. **Instruments pass** (ghi kết quả vào `docs/performance.md`):
   - *Time Profiler*: cuộn list — hàm nào nặng nhất trên main thread?
   - *Allocations* / *Leaks*: push/pop detail 10 lần — bộ nhớ có quay về?
   - *Network* (hoặc log): số request khi gõ search.
3. **Token tuỳ chọn**: màn Settings cho người dùng dán Personal Access Token (fine-grained, chỉ quyền đọc public repo) → `KeychainStore` (§4.9) → `APIClient(tokenProvider:)`. Hiển thị quota còn lại (`x-ratelimit-remaining`).
4. **Release config**: `Debug`/`Release` khác nhau ở log (`#if DEBUG`), `os.Logger` thay `print`, tắt log nhạy cảm; Build Settings: `SWIFT_OPTIMIZATION_LEVEL = -O` cho Release (mặc định); thử **Archive** thành công (không cần upload).
5. Thêm **SwiftLint** (tuỳ chọn) chạy trong CI.

✅ DoD
- [ ] PR mới có check xanh trên GitHub.
- [ ] `docs/performance.md` có ít nhất 3 phát hiện + 1 sửa đổi có số liệu trước/sau.
- [ ] `git log -p | grep -i "ghp_\|github_pat_"` → rỗng (không lộ token).
- [ ] Archive Release thành công.

💡 Gợi ý: profile trên **thiết bị thật** + Release build nếu có thể; simulator không phản ánh đúng hiệu năng.
🎤 Chứng minh: CI/CD, Instruments, Keychain vs UserDefaults, build configuration, bảo mật secrets.
📝 `ci: run unit tests on GitHub Actions` · `feat(settings): optional GitHub token stored in Keychain` · `perf: fix main-thread hotspot found with Time Profiler`

---

### M18 — Tuần 18 (T7 13/03): Polish, sơ đồ kiến trúc, 3 câu chuyện

🎯 Repo sẵn sàng để gửi kèm CV; bạn kể được 3 câu chuyện kỹ thuật từ nó.

📋 Việc cần làm
1. Polish: Dynamic Type (font `.preferredFont`), Dark Mode, VoiceOver label cho nút ⭐ và ảnh, empty state có minh hoạ, haptic khi thêm favorite.
2. **Sơ đồ kiến trúc** (draw.io / Excalidraw / Mermaid trong README):
   ```mermaid
   flowchart LR
     subgraph Features
       SVC[SearchViewController] --> SVM[SearchViewModel]
       DV[RepositoryDetailView] --> DVM[DetailViewModel]
       FV[FavoritesView] --> FVM[FavoritesViewModel]
     end
     subgraph Domain
       RSP[[RepositoryServiceProtocol]]
       FSP[[FavoritesStoreProtocol]]
     end
     subgraph Data
       GRS[GitHubRepositoryService] --> AC[APIClient] --> US[(URLSession)]
       SDS[SwiftDataFavoritesStore] --> SD[(SwiftData)]
       IL[ImageLoader actor] --> IC[(NSCache)]
     end
     SVM --> RSP
     DVM --> RSP
     DVM --> FSP
     FVM --> FSP
     GRS -. implements .-> RSP
     SDS -. implements .-> FSP
     AppCoordinator --> SVC
     AppContainer --> GRS
     AppContainer --> SDS
   ```
3. **Chuẩn bị 3 story** theo STAR (chi tiết cách kể ở [week18](../weeks/week18-system-design-mock.md)). Gợi ý chọn từ:
   - **Story A — Hiệu năng list:** ảnh avatar full-size làm bộ nhớ tăng X MB → downsampling + prefetch + huỷ task khi reuse → giảm còn Y MB (số liệu từ M10/M17).
   - **Story B — Concurrency:** request search cũ về sau ghi đè kết quả mới / ID trùng giữa 2 trang làm diffable crash → cancel + lọc trùng + test tái hiện.
   - **Story C — Kiến trúc & test:** refactor từ "VC làm hết" sang MVVM + DI → viết được 15+ test không cần mạng, CI chạy mỗi PR.
   - (Dự phòng) Story D — Rate limit: thiết kế debounce + cache + thông báo thời gian reset.
4. Viết mỗi story ≤ 200 chữ vào `docs/stories.md` (file này có thể để private — đừng commit nếu không muốn).
5. Đưa link repo lên CV, LinkedIn.

✅ DoD
- [ ] README đủ các mục ở §6, có sơ đồ.
- [ ] Kể mỗi story trong 2 phút không nhìn giấy (ghi âm lại).
- [ ] Cài app lên **iPhone thật** để mở cho interviewer xem nếu phỏng vấn trực tiếp (Xcode + Apple ID miễn phí, app hết hạn sau 7 ngày — cài lại trước buổi phỏng vấn).

🎤 Chứng minh: mọi thứ — đây là "portfolio".
📝 `docs: architecture diagram and polish` · `feat: accessibility and dynamic type support`

---

### M19 — Tuần 19 (20/03): 🧊 Đóng băng

🎯 Không thêm tính năng. Repo ở trạng thái ổn định, CI xanh.

📋 Việc cần làm
- Chỉ sửa bug nghiêm trọng. Mọi ý tưởng mới ghi vào mục "Roadmap / Future work" trong README (đây cũng là câu trả lời cho *"nếu có thêm thời gian em sẽ làm gì?"*).
- Đọc lại toàn bộ code 1 lượt như thể bạn là interviewer: chỗ nào bạn **không giải thích được** → hoặc hiểu nó, hoặc đơn giản hoá nó.
- Gắn tag `v1.0`.

✅ DoD
- [ ] CI xanh trên `main`, tag `v1.0`.
- [ ] Giải thích được **mọi dòng** trong `APIClient`, `ImageLoader`, `SearchViewModel`.

📝 `chore(release): v1.0`

---

## 6. README template cho repo

````markdown
# RepoScope

![CI](https://github.com/<username>/RepoScope/actions/workflows/ci.yml/badge.svg)
![Swift 6](https://img.shields.io/badge/Swift-6-orange) ![iOS 17+](https://img.shields.io/badge/iOS-17%2B-blue)

Explore GitHub repositories: search, view details, save favorites for offline use.
Built **without third-party libraries** and **without AI-generated code** as a learning project.

| Search | Detail | Favorites (offline) |
|---|---|---|
| ![](docs/screenshots/search.png) | ![](docs/screenshots/detail.png) | ![](docs/screenshots/favorites.png) |

## Features
- Repository search with debounce, infinite scrolling and prefetching
- Detail screen in SwiftUI; search screen in UIKit (compositional layout + diffable data source)
- Image loading: custom actor-based loader, in-flight request de-duplication, NSCache, ImageIO downsampling
- Favorites persisted with SwiftData, available offline
- Handles GitHub rate limit (shows reset time), offline mode, empty and error states
- Optional personal access token stored in Keychain

## Architecture
MVVM + Coordinator, constructor-based dependency injection, layered as Domain / Data / Features.
<diagram here>

Key decisions (see `docs/adr/`):
- Why MVVM + Coordinator instead of VIPER/TCA
- Why a custom image loader instead of Kingfisher
- Why UIKit for the list and SwiftUI for detail

## Tech
Swift 6 (strict concurrency) · async/await · actors · UIKit · SwiftUI (@Observable, NavigationStack) · SwiftData · Swift Testing / XCTest · GitHub Actions

## Testing
- 20+ unit tests, no real network: `URLProtocol` stub for networking, in-memory SwiftData store
- Run: `⌘U` or `xcodebuild test -scheme RepoScope -destination 'platform=iOS Simulator,name=iPhone 16'`

## Performance notes
| Change | Before | After |
|---|---|---|
| Avatar downsampling | xx MB | yy MB |
| ... | | |

## Getting started
1. Xcode 16+, iOS 17+ simulator
2. Open `RepoScope.xcodeproj`, run
3. (Optional) Settings → paste a GitHub token to raise the rate limit (60 → 5000 req/hour)

## Known issues / Future work
- Image cache key does not include target size
- README rendering supports basic Markdown only
- ...

## What I learned
3–5 bullet points, honest.
````

---

## 7. Checklist tổng (tick khi xong milestone)

| Milestone | Tuần | Xong | Tag | Ghi chú |
|---|---|---|---|---|
| M1 Setup | 1 | [ ] | `m01-setup` | |
| M2 Model | 2 | [ ] | `m02-models` | |
| M3 Protocol + Mock | 3 | [ ] | `m03-protocols` | |
| M4 Codable + APIClient | 4 | [ ] | `m04-codable` | |
| M5 Memory + NSCache | 5 | [ ] | `m05-memory` | |
| M6 GCD | 6 | [ ] | `m06-gcd` | |
| M7 async/await + API thật | 7 | [ ] | `m07-async` | |
| M8 Swift 6 + debounce | 8 | [ ] | `m08-swift6` | |
| M9 UIKit list | 9 | [ ] | `m09-uikit` | |
| M10 Collection view + pagination | 10 | [ ] | `m10-collection` | |
| M11 SwiftUI detail | 11 | [ ] | `m11-swiftui` | |
| M12 Navigation + interop | 12 | [ ] | `m12-navigation` | |
| M13–14 README + screenshots | 13–14 | [ ] | — | |
| M15 Architecture | 15 | [ ] | `m15-architecture` | |
| M16 Persistence + tests | 16 | [ ] | `m16-persistence` | |
| M17 CI + profiling + Keychain | 17 | [ ] | `m17-ci` | |
| M18 Polish + stories | 18 | [ ] | `m18-polish` | |
| M19 Freeze | 19 | [ ] | `v1.0` | |

> **Bị trễ?** Thứ tự ưu tiên giữ lại: M7 (API thật) → M10 (list + pagination) → M15 (kiến trúc) → M16 (test) → M11 (SwiftUI). Có thể bỏ: M6 (giữ phần đọc lý thuyết), deep link M12, token M17, README rendering.

---

## 📖 Tài liệu

- [GitHub REST API — Search repositories](https://docs.github.com/en/rest/search/search#search-repositories)
- [GitHub REST API — Repositories](https://docs.github.com/en/rest/repos/repos#get-a-repository) · [Contents (README)](https://docs.github.com/en/rest/repos/contents#get-a-repository-readme)
- [GitHub REST API — Rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) — **kiểm tra lại con số trước khi nói trong phỏng vấn**
- Apple — [Modern Collection Views (sample code)](https://developer.apple.com/documentation/uikit/views_and_controls/collection_views/implementing_modern_collection_views)
- WWDC18 — *Images and Graphics Best Practices* (downsampling)
- WWDC21 — *Use async/await with URLSession* · WWDC21 — *Protect mutable state with Swift actors*
- WWDC23 — *Meet SwiftData* · *Discover Observation in SwiftUI*
- WWDC24 — *Migrate your app to Swift 6* · *Meet Swift Testing*
