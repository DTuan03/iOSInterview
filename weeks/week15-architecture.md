# Tuần 15 — Kiến trúc: MVC · MVVM · MVP · VIPER · Clean · TCA · SOLID · DI · Coordinator (15/02 – 21/02/2027)

> **Giai đoạn 4 · Kiến trúc & Hệ thống** · Thời lượng: ~14 giờ
> Sau Tết, đây là tuần **phân loại ứng viên 1 năm và 2 năm**. Người phỏng vấn không hỏi "MVVM là gì" để nghe định nghĩa — họ hỏi để xem bạn **có hiểu vì sao phải tách lớp, tách thế nào, và trả giá gì**. Câu trả lời ăn điểm luôn có chữ *"tuỳ"* kèm lý do cụ thể.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích **vì sao cần kiến trúc** bằng 3 từ khoá: testability, separation of concerns, khả năng làm việc nhóm
- [ ] Chỉ ra "Massive View Controller" trong một đoạn code và **refactor nó sang MVVM** trên giấy
- [ ] Biết 3 cách binding ViewModel → View (closure, Combine, `@Observable`) và ưu/nhược mỗi cách
- [ ] Vẽ được sơ đồ VIPER (5 vai trò) và Clean Architecture (3 lớp + dependency rule)
- [ ] Giải thích TCA ở mức khái niệm: State, Action, Reducer, Effect, Store
- [ ] Nêu **5 nguyên tắc SOLID**, mỗi nguyên tắc có một ví dụ Swift tự nghĩ ra
- [ ] Giải thích 3 kiểu Dependency Injection, ưu/nhược DI container
- [ ] Viết Coordinator có child coordinator mà **không bị leak**
- [ ] Trả lời "team 3 người chọn kiến trúc gì? team 15 người?" có lập luận
- [ ] Làm xong 6 bài DSA tuần 15 — Graph BFS/DFS (xem [dsa-plan](../tracks/dsa-plan.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 15/02** | 2h | §1 Vì sao cần kiến trúc · §2 MVC & Massive View Controller. Bài tập 1 · DSA: Number of Islands | `day1.md` — liệt kê trách nhiệm trong 1 VC cũ của bạn |
| **T3 16/02** | 2h | §3 **MVVM** (đọc kỹ nhất tuần): binding, cái gì để trong VM, input/output. Bài tập 2, 3 · DSA: Max Area of Island | `RepoListViewModel.swift` + 3 unit test |
| **T4 17/02** | 2h | §4 MVP · §5 VIPER · §6 **Clean Architecture** · §7 TCA. Bài tập 4 · DSA: Clone Graph | Sơ đồ Clean vẽ tay (chụp ảnh lưu lại) |
| **T5 18/02** | 2h | §8 So sánh + chọn kiến trúc · §9 **SOLID**. Bài tập 5 · DSA: Rotting Oranges | `solid.swift` chạy được |
| **T6 19/02** | 1,5h | §10 **Dependency Injection** · §12 Modularization SPM. Bài tập 6 · DSA: Course Schedule | `di.swift` |
| **T7 20/02** | 4h | §11 **Coordinator**. Bài tập 7 + Bài tập lớn. Side project: Milestone 15 (refactor RepoScope sang MVVM + Clean-ish + DI + Coordinator) · DSA: Pacific Atlantic Water Flow | Repo RepoScope có commit refactor |
| **CN 21/02** | 2h | Tự trả lời 18 câu phỏng vấn (viết trước, mở đáp án sau). Ghi âm 2 câu ⭐. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. Vì sao cần kiến trúc?

Kiến trúc **không phải để code đẹp**. Nó giải quyết 3 vấn đề thực tế:

| Vấn đề | Khi không có kiến trúc | Kiến trúc giúp gì |
|---|---|---|
| **Testability** | Logic nằm trong `UIViewController` → muốn test phải dựng view, gọi network thật | Logic nằm trong class thuần Swift, inject mock → test trong mili-giây |
| **Separation of concerns** | Sửa format ngày tháng phải đụng vào file 1.500 dòng có cả network lẫn animation | Mỗi lớp một trách nhiệm → sửa chỗ nào biết chỗ đó |
| **Làm việc nhóm** | 3 người cùng sửa 1 file → conflict liên tục | Ranh giới rõ → mỗi người một module/feature |

Ngoài ra: **thay thế được** (đổi Alamofire → URLSession, Core Data → SwiftData mà UI không biết), **onboard người mới nhanh** (mọi màn hình giống nhau về cấu trúc).

**Cái giá phải trả:** nhiều file hơn, nhiều protocol hơn, nhiều "boilerplate" hơn, người mới phải học quy ước. → Kiến trúc là **trade-off**, không phải "càng nhiều lớp càng tốt". Đây là câu bạn phải nói được khi bị hỏi.

> Một câu hay dùng: *"Architecture is about making the code easy to change, not easy to write."*

### §2. MVC và Massive View Controller

**MVC theo Apple (Cocoa MVC):**
```
   ┌────────┐  user action   ┌──────────────────┐   update    ┌───────┐
   │  View  │ ─────────────▶ │    Controller    │ ──────────▶ │ Model │
   │        │ ◀───────────── │ (UIViewController)│ ◀────────── │       │
   └────────┘   update UI    └──────────────────┘   notify    └───────┘
```
- **Model**: dữ liệu + business logic (struct `Repo`, class lưu trữ...).
- **View**: hiển thị, không biết Model (`UILabel`, `UITableViewCell`...).
- **Controller**: trung gian.

**Vấn đề:** trong UIKit, `UIViewController` **vừa là Controller vừa dính chặt với View** (nó sở hữu `view`, nhận lifecycle `viewDidLoad`...). Không có chỗ nào khác "hợp lý" để đặt code → mọi thứ dồn vào VC → **Massive View Controller**.

Ví dụ thật — màn hình tìm repo GitHub viết kiểu MVC "tự nhiên":

```swift
final class RepoListMVCViewController: UITableViewController {
    private var repos: [Repo] = []
    private let query = "swift"

    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        load()
    }

    private func load() {
        // ① Networking ngay trong VC
        let url = URL(string: "https://api.github.com/search/repositories?q=\(query)")!
        URLSession.shared.dataTask(with: url) { [weak self] data, _, error in
            guard let data, error == nil else { return }          // ② lỗi bị nuốt
            struct Response: Decodable { let items: [Repo] }
            let decoder = JSONDecoder()                            // ③ parsing trong VC
            decoder.keyDecodingStrategy = .convertFromSnakeCase
            guard let res = try? decoder.decode(Response.self, from: data) else { return }
            let sorted = res.items.sorted { $0.stargazersCount > $1.stargazersCount } // ④ business logic
            DispatchQueue.main.async {
                self?.repos = sorted
                self?.tableView.reloadData()
            }
        }.resume()
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { repos.count }

    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        let repo = repos[indexPath.row]
        let stars = repo.stargazersCount >= 1000                  // ⑤ presentation logic
            ? String(format: "%.1fk", Double(repo.stargazersCount) / 1000)
            : "\(repo.stargazersCount)"
        var config = cell.defaultContentConfiguration()
        config.text = repo.name
        config.secondaryText = "★ \(stars)"
        cell.contentConfiguration = config
        return cell
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let detail = UIViewController()                           // ⑥ navigation: VC biết VC tiếp theo
        detail.title = repos[indexPath.row].name
        navigationController?.pushViewController(detail, animated: true)
    }
}
```

**Đếm trách nhiệm:** ① networking ② error handling ③ parsing ④ business rule (sort) ⑤ format hiển thị ⑥ điều hướng ⑦ quản lý table view. 7 việc trong 1 class. Hậu quả:
- Muốn test "sort theo sao giảm dần" và "1500 → 1.5k" → **không test được** mà không gọi API thật và dựng table view.
- Đổi API → sửa VC. Đổi UI → sửa VC. Đổi luồng màn hình → sửa VC.

**MVC không có lỗi — dùng sai mới có lỗi.** MVC vẫn ổn cho màn hình nhỏ (Settings, About). Có thể "cứu" MVC bằng cách tách ra: `RepoService`, `RepoCellConfigurator`, child view controller, `UITableViewDataSource` riêng... Nhưng vẫn khó test vì logic trạng thái vẫn ở VC.

### §3. MVVM — phần quan trọng nhất tuần

```
 ┌──────────────────┐  user intent (load, select)  ┌────────────────┐   gọi    ┌──────────────────┐
 │ View / VC         │ ───────────────────────────▶ │   ViewModel    │ ───────▶ │ Service / Model  │
 │ (chỉ hiển thị)    │ ◀─────────────────────────── │ (state, logic) │ ◀─────── │                  │
 └──────────────────┘   binding: state thay đổi    └────────────────┘          └──────────────────┘
```

**Quy tắc vàng:**
1. ViewModel **không import UIKit** (lý tưởng). Không giữ `UIViewController`, `UIColor`, `UIImage`... → test được trên mọi platform, không cần simulator.
2. ViewModel **không biết View**. View giữ VM (strong), VM không giữ View → không retain cycle; VM gửi thông tin ra ngoài qua binding.
3. View **"ngu"**: chỉ nhận state đã format sẵn và vẽ.

**Cái gì để trong ViewModel?**
| Để trong VM | Không để trong VM |
|---|---|
| State màn hình (`loading`, `loaded`, `error`) | `UIView`, `UIColor`, constraint, animation |
| Presentation logic: format số, ngày, text lỗi | Code gọi `URLSession` trực tiếp (để ở Service/Repository) |
| Gọi service / use case, xử lý kết quả | Logic nghiệp vụ dùng chung nhiều màn (để ở UseCase/Model) |
| Validation input (email hợp lệ?) | Điều hướng trực tiếp (`pushViewController`) — báo ra ngoài cho Coordinator |

**Cùng tính năng trên, viết lại bằng MVVM:**

```swift
// 1. Abstraction cho data source → inject được mock khi test
protocol RepoServiceProtocol: Sendable {
    func searchRepos(query: String) async throws -> [Repo]
}

// 2. Dữ liệu ĐÃ FORMAT cho 1 cell — View chỉ việc gán
struct RepoRowViewModel: Equatable {
    let title: String
    let starsText: String

    init(repo: Repo) {
        title = repo.name
        starsText = repo.stargazersCount >= 1000
            ? String(format: "★ %.1fk", Double(repo.stargazersCount) / 1000)
            : "★ \(repo.stargazersCount)"
    }
}

// 3. ViewModel: không import UIKit, chạy trên MainActor vì state phục vụ UI
@MainActor
final class RepoListViewModel {
    enum State: Equatable {
        case idle, loading
        case loaded([RepoRowViewModel])
        case failed(String)
    }

    var onStateChange: ((State) -> Void)?      // binding bằng closure
    var onSelectRepo: ((Repo) -> Void)?        // điều hướng → báo ra ngoài (Coordinator)

    private(set) var state: State = .idle {
        didSet { onStateChange?(state) }
    }
    private var repos: [Repo] = []
    private let service: RepoServiceProtocol

    init(service: RepoServiceProtocol) {       // constructor injection
        self.service = service
    }

    func load(query: String) async {
        state = .loading
        do {
            repos = try await service.searchRepos(query: query)
                .sorted { $0.stargazersCount > $1.stargazersCount }
            state = .loaded(repos.map(RepoRowViewModel.init))
        } catch {
            state = .failed("Không tải được dữ liệu")
        }
    }

    func didSelectRow(at index: Int) {
        guard repos.indices.contains(index) else { return }
        onSelectRepo?(repos[index])
    }
}

// 4. View Controller: chỉ còn render + chuyển sự kiện
final class RepoListViewController: UITableViewController {
    private let viewModel: RepoListViewModel
    private var rows: [RepoRowViewModel] = []

    init(viewModel: RepoListViewModel) {
        self.viewModel = viewModel
        super.init(style: .plain)
    }
    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }

    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        viewModel.onStateChange = { [weak self] state in   // weak: VC giữ VM, VM giữ closure → tránh cycle
            self?.render(state)
        }
        Task { await viewModel.load(query: "swift") }
    }

    private func render(_ state: RepoListViewModel.State) {
        switch state {
        case .idle: break
        case .loading: title = "Đang tải..."
        case .loaded(let rows):
            title = "Repos"
            self.rows = rows
            tableView.reloadData()
        case .failed(let message): title = message
        }
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { rows.count }

    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        var config = cell.defaultContentConfiguration()
        config.text = rows[indexPath.row].title
        config.secondaryText = rows[indexPath.row].starsText
        cell.contentConfiguration = config
        return cell
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        viewModel.didSelectRow(at: indexPath.row)
    }
}
```

**Thắng được gì?** Bây giờ test được toàn bộ logic mà không cần UI, không cần mạng:
```swift
final class MockRepoService: RepoServiceProtocol, @unchecked Sendable {
    var result: Result<[Repo], Error> = .success([])
    func searchRepos(query: String) async throws -> [Repo] { try result.get() }
}

@MainActor
final class RepoListViewModelTests: XCTestCase {
    func test_load_sortsByStarsDescending_andFormatsStars() async {
        let service = MockRepoService()
        service.result = .success([Repo(id: 1, name: "a", stargazersCount: 10),
                                   Repo(id: 2, name: "b", stargazersCount: 1500)])
        let sut = RepoListViewModel(service: service)

        await sut.load(query: "swift")

        XCTAssertEqual(sut.state, .loaded([
            RepoRowViewModel(repo: Repo(id: 2, name: "b", stargazersCount: 1500)),  // "★ 1.5k"
            RepoRowViewModel(repo: Repo(id: 1, name: "a", stargazersCount: 10)),    // "★ 10"
        ]))
    }
}
```
(Tuần 16 học kỹ testing. Giờ chỉ cần thấy: *MVVM tồn tại để có được đoạn test này*.)

#### 3 cách binding ViewModel → View

**a) Closure / delegate** (như ví dụ trên) — đơn giản, không cần framework, dễ debug. Nhược: mỗi property một closure → dài; phải nhớ `[weak self]`; chỉ 1 subscriber.

**b) Combine** — `@Published` + `sink`:
```swift
@MainActor
final class SearchViewModel {
    @Published var query = ""                       // View đẩy text vào đây
    @Published private(set) var results: [String] = []
    private var cancellables = Set<AnyCancellable>()

    init() {
        $query
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)  // gõ xong 300ms mới tìm
            .removeDuplicates()
            .map { q in q.isEmpty ? [] : ["\(q)-1", "\(q)-2"] }           // thật: gọi service
            .sink { [weak self] in self?.results = $0 }
            .store(in: &cancellables)
    }
}
// VC: viewModel.$results.receive(on: DispatchQueue.main).sink { [weak self] in ... }.store(in: &bag)
```
Mạnh khi có luồng sự kiện phức tạp (debounce, combineLatest). Nhược: learning curve, stack trace khó đọc, dễ quên `store(in:)` (subscription bị huỷ ngay).

**c) `@Observable` (iOS 17+, Observation framework)** — cách hiện đại, nhất là với SwiftUI:
```swift
@MainActor
@Observable
final class RepoListObservableViewModel {
    private(set) var rows: [RepoRowViewModel] = []
    private(set) var isLoading = false
    private let service: RepoServiceProtocol
    init(service: RepoServiceProtocol) { self.service = service }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        rows = (try? await service.searchRepos(query: "swift"))?.map(RepoRowViewModel.init) ?? []
    }
}
// SwiftUI: @State private var vm = RepoListObservableViewModel(service: ...)
// View chỉ re-render khi property nó THỰC SỰ đọc thay đổi (khác ObservableObject: mọi @Published đổi đều re-render)
```
UIKit cũng tự track `@Observable`: iOS 18 bật bằng key `UIObservationTrackingEnabled` trong Info.plist (đọc property trong `viewWillLayoutSubviews`/`layoutSubviews` → tự gọi lại khi property đổi); iOS 26 bật mặc định và thêm `updateProperties()`.

| Binding | Hợp khi | Nhược |
|---|---|---|
| Closure/delegate | UIKit, team mới, màn hình đơn giản | Verbose, 1 subscriber |
| Combine | Luồng sự kiện phức tạp, cần operator | Khó debug, Apple ít đầu tư thêm |
| `@Observable` | SwiftUI, iOS 17+ | Cần iOS 17; với UIKit cần iOS 18+ |

#### Kiểu Input/Output (hay gặp ở công ty dùng RxSwift/Combine)
ViewModel là một **hàm thuần**: nhận stream input, trả stream output. Mọi thứ VM cần/nhả ra nằm ở 2 struct → đọc là hiểu.
```swift
@MainActor
final class LoginViewModel {
    struct Input {
        let email: AnyPublisher<String, Never>
        let password: AnyPublisher<String, Never>
    }
    struct Output {
        let isLoginEnabled: AnyPublisher<Bool, Never>
    }

    func transform(_ input: Input) -> Output {
        let enabled = Publishers.CombineLatest(input.email, input.password)
            .map { email, password in email.contains("@") && password.count >= 8 }
            .removeDuplicates()
            .eraseToAnyPublisher()
        return Output(isLoginEnabled: enabled)
    }
}
```
Ưu: contract rõ, test bằng cách đẩy input giả. Nhược: boilerplate nặng cho màn hình đơn giản.

**Nhược điểm của MVVM (phải nói được):** ViewModel có thể thành "Massive ViewModel"; không quy định điều hướng (cần thêm Coordinator); binding tự viết dễ leak; không có chuẩn chung → mỗi team một kiểu.

### §4. MVP

```
View (VC, passive) ◀──protocol──  Presenter  ──▶ Model/Service
     │ user action ───────────────▶   │
```
- **View** implement một protocol (`RepoListView`), hoàn toàn thụ động: `func show(rows:)`, `func showError(_:)`.
- **Presenter** giữ `weak var view: RepoListView?`, gọi **trực tiếp** method của view (không binding).
- Khác MVVM: MVVM **không biết View** (View tự quan sát), MVP **biết View qua protocol** và ra lệnh cho nó.
- Ưu: dễ test (mock view protocol, assert `show(rows:)` được gọi với gì), không cần framework binding. Nhược: protocol view phình to, Presenter phải `weak` view.

```swift
protocol RepoListView: AnyObject {
    func show(rows: [RepoRowViewModel])
    func showError(_ message: String)
}

@MainActor
final class RepoListPresenter {
    weak var view: RepoListView?          // weak! VC giữ presenter strong
    private let service: RepoServiceProtocol
    init(service: RepoServiceProtocol) { self.service = service }

    func viewDidLoad() async {
        do {
            let repos = try await service.searchRepos(query: "swift")
            view?.show(rows: repos.map(RepoRowViewModel.init))
        } catch {
            view?.showError("Không tải được dữ liệu")
        }
    }
}
```

### §5. VIPER

Tách mỗi màn hình thành 5 vai trò:

```
            ┌──────────┐
            │  Router  │  ← điều hướng, tạo module (wireframe)
            └────▲─────┘
                 │
┌──────┐   ┌─────┴─────┐   ┌────────────┐   ┌────────┐
│ View │◀─▶│ Presenter │◀─▶│ Interactor │──▶│ Entity │
└──────┘   └───────────┘   └────────────┘   └────────┘
 hiển thị   presentation     business logic    model thuần
            logic, điều phối  (use case), gọi service
```
| Vai trò | Trách nhiệm | Biết ai |
|---|---|---|
| **V**iew | Hiển thị, chuyển sự kiện | Presenter |
| **I**nteractor | Business logic của use case | Entity, Service; báo kết quả cho Presenter qua output protocol |
| **P**resenter | Nhận sự kiện từ View, gọi Interactor, format dữ liệu cho View, nhờ Router điều hướng | View (weak), Interactor, Router |
| **E**ntity | Model dữ liệu | Không ai |
| **R**outer | Tạo module, push/present màn hình khác | ViewController (weak) |

- **Ưu:** trách nhiệm cực kỳ rõ; test từng phần; team lớn chia việc dễ; điều hướng tách riêng.
- **Nhược:** **rất nhiều boilerplate** — 1 màn hình "About" cũng 5 file + 5–8 protocol; khó với SwiftUI (SwiftUI tự quản view state); nhiều tham chiếu chéo → dễ retain cycle (Presenter–View, Presenter–Router phải có `weak`).
- Thường dùng code generator (template Xcode, Generamba) để giảm gõ tay.

### §6. Clean Architecture

Đề xuất bởi Uncle Bob. Trong iOS thường gọn lại thành 3 lớp:

```
┌─────────────────────────────────────────────────────────────┐
│ PRESENTATION   View, ViewController, ViewModel              │
│                      │ gọi                                   │
│                      ▼                                       │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ DOMAIN   Entity (Repo) · UseCase (SearchReposUseCase)   │ │
│ │          Repository PROTOCOL (RepoRepository)           │ │
│ │          ← thuần Swift, không import UIKit/CoreData      │ │
│ └─────────────────────────────────────────────────────────┘ │
│                      ▲ implement protocol                    │
│                      │                                       │
│ DATA     RepoRepositoryImpl · APIClient · DTO · Core Data    │
└─────────────────────────────────────────────────────────────┘
   DEPENDENCY RULE: mũi tên phụ thuộc (import / biết đến) chỉ hướng VÀO Domain.
   Domain không biết gì về UI, network hay database.
```

**Dependency rule** là điểm cốt lõi: lớp trong (Domain) không biết lớp ngoài. Data layer **implement** protocol do Domain định nghĩa → đây chính là **Dependency Inversion** (chữ D trong SOLID).

```swift
// ===== DOMAIN =====
struct Repo: Equatable, Sendable {
    let id: Int
    let name: String
    let stars: Int
    var isFavorite: Bool = false
}

protocol RepoRepository: Sendable {          // Domain định nghĩa "cần gì"
    func search(query: String) async throws -> [Repo]
    func favoriteIDs() async -> Set<Int>
}

struct SearchReposUseCase: Sendable {         // 1 use case = 1 hành động nghiệp vụ
    let repository: RepoRepository

    func execute(query: String) async throws -> [Repo] {
        let trimmed = query.trimmingCharacters(in: .whitespaces)
        guard trimmed.count >= 2 else { return [] }               // business rule
        async let repos = repository.search(query: trimmed)       // chạy song song
        async let favorites = repository.favoriteIDs()
        let favs = await favorites
        return try await repos
            .map { repo in var r = repo; r.isFavorite = favs.contains(r.id); return r }
            .sorted { $0.stars > $1.stars }
    }
}

// ===== DATA =====
struct RepoDTO: Decodable {                   // DTO = hình dạng JSON của server
    let id: Int
    let full_name: String
    let stargazers_count: Int
    func toDomain() -> Repo { Repo(id: id, name: full_name, stars: stargazers_count) }
}
// final class RepoRepositoryImpl: RepoRepository { gọi APIClient + Core Data, map DTO → Repo }

// ===== TEST: thay Data bằng stub =====
struct StubRepoRepository: RepoRepository {
    var repos: [Repo]
    var favs: Set<Int>
    func search(query: String) async throws -> [Repo] { repos }
    func favoriteIDs() async -> Set<Int> { favs }
}

let useCase = SearchReposUseCase(repository: StubRepoRepository(
    repos: [Repo(id: 1, name: "a", stars: 10), Repo(id: 2, name: "b", stars: 99)],
    favs: [1]))
let result = try await useCase.execute(query: "  sw ")
print(result.map { "\($0.name):\($0.isFavorite)" })   // ["b:false", "a:true"]
print(try await useCase.execute(query: "s").count)    // 0
```

**Vì sao tách DTO và Entity?** Server đổi tên field (`full_name` → `fullName`) chỉ sửa DTO + mapper; Domain và UI không đổi. Entity có thể có field server không trả (`isFavorite` lấy từ DB local).

**Repository** = "nơi lấy dữ liệu", che giấu *từ đâu* (API, cache, DB). UseCase không biết dữ liệu đến từ mạng hay disk → dễ làm offline-first (tuần 16).

**Khi nào Clean là quá tay?** App 5 màn hình CRUD đơn giản: UseCase chỉ gọi thẳng repository (`execute() { repository.fetch() }`) — chỉ thêm file mà không thêm giá trị. Thực tế nhiều team cho phép **ViewModel gọi thẳng Repository** khi không có business logic, chỉ tạo UseCase khi có logic thật hoặc dùng chung → "Clean-ish". Đây là câu trả lời thực tế được đánh giá cao.

### §7. TCA (The Composable Architecture) — mức khái niệm

Thư viện của Point-Free, lấy cảm hứng từ Redux/Elm. **Unidirectional data flow:**

```
        ┌──────────── send(Action) ◀──────────── View
        ▼                                          ▲
   ┌─────────┐   (inout State, Action) → Effect    │ State mới
   │ Reducer │ ──────────────────────────────────▶ Store
   └─────────┘                                     │
        ▲                                          ▼
        └──────── Action (kết quả) ◀───────── Effect (API, timer...)
```
- **State**: struct mô tả toàn bộ dữ liệu màn hình.
- **Action**: enum mọi sự kiện (user tap, kết quả API).
- **Reducer**: hàm thuần `(inout State, Action) -> Effect` — chỗ DUY NHẤT được đổi state.
- **Effect**: side effect (network, timer), khi xong gửi lại Action.
- **Store**: giữ state, nhận action, chạy reducer, chạy effect.

Mô phỏng ý tưởng bằng Swift thuần (không phải API thật của TCA):
```swift
struct CounterState: Equatable { var count = 0; var fact: String? }
enum CounterAction { case increment, decrement, factButtonTapped, factResponse(String) }
enum Effect { case none, fetchFact(Int) }

func counterReducer(_ state: inout CounterState, _ action: CounterAction) -> Effect {
    switch action {
    case .increment: state.count += 1; return .none
    case .decrement: state.count -= 1; return .none
    case .factButtonTapped: state.fact = nil; return .fetchFact(state.count)  // không gọi API ở đây, chỉ MÔ TẢ
    case .factResponse(let text): state.fact = text; return .none
    }
}

var state = CounterState()
_ = counterReducer(&state, .increment)
_ = counterReducer(&state, .increment)
let effect = counterReducer(&state, .factButtonTapped)
print(state.count, effect)    // 2 fetchFact(2)
```
- **Ưu:** state dự đoán được, test cực mạnh (TestStore assert từng action → state), compose feature con, dependency management tích hợp.
- **Nhược:** phụ thuộc thư viện bên thứ 3 lớn, API đổi nhiều qua các version, learning curve dốc, performance cần chú ý với state lớn, tuyển người khó hơn.

### §8. So sánh và chọn kiến trúc

| | MVC | MVVM | MVP | VIPER | Clean (+MVVM) | TCA |
|---|---|---|---|---|---|---|
| Số file/màn hình | 1–2 | 2–3 | 3 | 5+ | 4–6 | 2–3 |
| Testability | Thấp | Tốt | Tốt | Rất tốt | Rất tốt | Rất tốt |
| Boilerplate | Ít | Vừa | Vừa | Rất nhiều | Nhiều | Vừa (+thư viện) |
| Điều hướng | Trong VC | Không quy định → Coordinator | Không quy định | Router | Coordinator | Trong state |
| Hợp SwiftUI | — | ✅ Rất hợp | Kém | Kém | ✅ | ✅ |
| Learning curve | Thấp | Thấp–vừa | Thấp | Cao | Vừa–cao | Cao |

**"Team 3 người, startup, cần ship nhanh — chọn gì?"**
> MVVM + Coordinator (UIKit) hoặc MVVM với `@Observable` (SwiftUI), có tầng Service/Repository qua protocol để test. Không cần UseCase cho mọi thứ. Lý do: ít boilerplate, ai cũng biết, vẫn test được logic chính. Khi app lớn dần thì tách module.

**"Team 15 người, app lớn, nhiều feature team?"**
> Clean Architecture (Domain/Data/Presentation) + MVVM ở presentation + Coordinator, **modularize theo feature bằng SPM**, DI rõ ràng. Lý do: ranh giới module giảm conflict, mỗi team sở hữu feature, build nhanh hơn (chỉ build module đổi), Domain test độc lập. Có thể cân nhắc TCA nếu cả team chấp nhận đầu tư học và phụ thuộc thư viện; VIPER nếu codebase UIKit cũ đã dùng sẵn.

Điều người phỏng vấn muốn nghe: **bạn chọn dựa trên team size, độ phức tạp, deadline, UIKit/SwiftUI, kinh nghiệm team** — và **nhất quán** quan trọng hơn kiến trúc "tốt nhất".

### §9. SOLID với ví dụ Swift

**S — Single Responsibility:** mỗi type chỉ có **một lý do để thay đổi**.
```swift
// ❌ Một class vừa tính tiền, vừa format, vừa lưu file, vừa gửi email
// ✅ Tách:
struct ReceiptFormatter { func format(total: Double) -> String { String(format: "Tổng: %.0f đ", total) } }
// PriceCalculator tính tiền, ReceiptStorage lưu, EmailSender gửi
print(ReceiptFormatter().format(total: 350_000))   // Tổng: 350000 đ
```

**O — Open/Closed:** mở để mở rộng, đóng để sửa đổi. Thêm loại mới **không phải sửa code cũ**.
```swift
// ❌ switch discountType { case .student: ... case .blackFriday: ... } — thêm loại = sửa hàm
protocol DiscountRule { func apply(to price: Double) -> Double }
struct StudentDiscount: DiscountRule { func apply(to price: Double) -> Double { price * 0.8 } }
struct BlackFridayDiscount: DiscountRule { func apply(to price: Double) -> Double { price - 50_000 } }

struct PriceCalculator {
    let rules: [DiscountRule]
    func finalPrice(_ base: Double) -> Double { rules.reduce(base) { $1.apply(to: $0) } }
}
print(PriceCalculator(rules: [StudentDiscount(), BlackFridayDiscount()]).finalPrice(500_000))  // 350000.0
// Thêm VoucherDiscount: tạo struct mới, PriceCalculator không đổi một dòng.
```

**L — Liskov Substitution:** subclass thay được cho superclass **mà không phá kỳ vọng** của code dùng nó.
```swift
class Bird { func fly() -> String { "bay" } }
class Penguin: Bird {
    override func fly() -> String { fatalError("Chim cánh cụt không bay") }  // ❌ vi phạm LSP (crash runtime)
}
// Code nhận [Bird] gọi fly() sẽ crash khi gặp Penguin.
// ✅ Sửa: tách khả năng ra protocol: protocol Flyable { func fly() }, chỉ chim biết bay mới conform.
```
Ví dụ kinh điển khác: `Square: Rectangle` — set `width` làm đổi `height` → phá kỳ vọng của code dùng `Rectangle`.

**I — Interface Segregation:** đừng bắt client phụ thuộc vào method nó không dùng. Protocol nhỏ, tập trung.
```swift
// ❌ protocol Storage { func read(...); func write(...); func delete(...); func migrate(...) }
protocol Readable { func read(key: String) -> String? }
protocol Writable { func write(_ value: String, key: String) }
typealias ReadWrite = Readable & Writable      // ghép khi cần
// Màn hình chỉ đọc cache thì chỉ phụ thuộc Readable → mock chỉ 1 method.
```
Apple làm điều này: `UITableViewDataSource` tách khỏi `UITableViewDelegate`; `Codable = Encodable & Decodable`.

**D — Dependency Inversion:** module cấp cao không phụ thuộc module cấp thấp; **cả hai phụ thuộc abstraction**.
```swift
protocol Logger { func log(_ message: String) }
final class ConsoleLogger: Logger { func log(_ message: String) { print("[LOG] \(message)") } }

final class CheckoutService {                 // cấp cao chỉ biết protocol
    private let logger: Logger
    init(logger: Logger) { self.logger = logger }
    func checkout() { logger.log("checkout") }
}
// Test: inject SpyLogger ghi lại message. Prod: ConsoleLogger hoặc FirebaseLogger.
```
**Đừng nhầm:** Dependency **Inversion** (nguyên tắc thiết kế) ≠ Dependency **Injection** (kỹ thuật truyền dependency vào). Injection là một cách để đạt Inversion.

### §10. Dependency Injection

**Dependency** = thứ một object cần để làm việc (service, logger, session). **Injection** = truyền từ ngoài vào thay vì tự tạo bên trong.
```swift
// ❌ Tự tạo — không thay được khi test, ẩn phụ thuộc
final class BadViewModel { private let service = GitHubRepoService(session: .shared) }
// ❌ Singleton ẩn — mọi test dùng chung state
// ... NetworkManager.shared.fetch()
```

**3 kiểu injection:**
```swift
// 1. Constructor injection — ƯU TIÊN: phụ thuộc bắt buộc, không thể quên, dùng được `let`
final class RepoListViewModel { init(service: RepoServiceProtocol) { ... } }

// 2. Property injection — khi không kiểm soát init (VC từ storyboard), hoặc phụ thuộc tuỳ chọn
final class ProfileViewController: UIViewController {
    var analytics: AnalyticsTracking?      // nhược: có thể quên gán, phải Optional hoặc !
}

// 3. Method injection — phụ thuộc chỉ cần cho 1 lần gọi
func export(report: Report, using formatter: ReportFormatter) { ... }
```

**Protocol-based DI cho testing:** inject protocol → test truyền mock/stub/spy. Thêm default value giúp code production gọn mà test vẫn thay được:
```swift
init(service: RepoServiceProtocol = GitHubRepoService(session: .shared)) { ... }
```

**Composition Root:** nơi duy nhất "lắp ráp" object graph — thường là `SceneDelegate`/`App`/`AppCoordinator`. Mọi class khác chỉ nhận dependency qua init.

**DI Container** — một registry tạo object theo type:
```swift
final class Container {
    private var factories: [ObjectIdentifier: () -> Any] = [:]
    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        factories[ObjectIdentifier(type)] = factory
    }
    func resolve<T>(_ type: T.Type) -> T {
        guard let factory = factories[ObjectIdentifier(type)], let value = factory() as? T else {
            fatalError("Chưa register \(T.self)")          // lỗi phát hiện lúc RUNTIME
        }
        return value
    }
}

let container = Container()
container.register(Logger.self) { ConsoleLogger() }
container.register(CheckoutService.self) { CheckoutService(logger: container.resolve(Logger.self)) }
container.resolve(CheckoutService.self).checkout()    // [LOG] checkout
```
| | DI thủ công (pure DI) | DI container (Swinject, Factory, Resolver) |
|---|---|---|
| Lỗi thiếu dependency | **Compile time** | Runtime crash |
| Boilerplate | Nhiều khi graph lớn | Ít |
| Scope (singleton, per-screen) | Tự quản | Có sẵn |
| Dễ đọc/debug | Rất rõ | "Magic", khó lần theo |
| Nguy cơ | — | Thành **Service Locator** nếu gọi `container.resolve` khắp nơi (ẩn phụ thuộc) |

→ App nhỏ–vừa: DI thủ công qua init + composition root là đủ. App lớn nhiều module: container hoặc thư viện compile-time safe (Needle, Factory).

### §11. Coordinator pattern

**Vấn đề:** VC tự `push` VC khác → VC A phải biết VC B, cách tạo B, dependency của B → khó tái sử dụng A, khó đổi luồng (A/B testing, deep link).

**Giải pháp:** tách điều hướng ra object riêng. VC/VM chỉ báo "user chọn repo X", Coordinator quyết định đi đâu.

```
               AppCoordinator (giữ window)
                ├── childCoordinators
                │     ├── AuthCoordinator      (login → otp)        ── xong thì remove
                │     └── MainTabCoordinator
                │            ├── RepoCoordinator     (list → detail)
                │            └── SettingsCoordinator
```

```swift
@MainActor
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func start()
}

extension Coordinator {
    func addChild(_ child: Coordinator) { childCoordinators.append(child) }
    func removeChild(_ child: Coordinator) {
        childCoordinators.removeAll { $0 === child }
    }
}

@MainActor
final class AppCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    private let window: UIWindow
    private let navigationController = UINavigationController()
    private let service: RepoServiceProtocol

    init(window: UIWindow, service: RepoServiceProtocol) {
        self.window = window
        self.service = service
    }

    func start() {
        window.rootViewController = navigationController
        window.makeKeyAndVisible()
        let repoCoordinator = RepoCoordinator(navigationController: navigationController, service: service)
        repoCoordinator.onFinish = { [weak self, weak repoCoordinator] in
            guard let self, let repoCoordinator else { return }
            self.removeChild(repoCoordinator)        // ← QUAN TRỌNG: không remove = leak
        }
        addChild(repoCoordinator)                    // ← QUAN TRỌNG: không giữ = bị dealloc ngay
        repoCoordinator.start()
    }
}

@MainActor
final class RepoCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    var onFinish: (() -> Void)?
    private let navigationController: UINavigationController
    private let service: RepoServiceProtocol

    init(navigationController: UINavigationController, service: RepoServiceProtocol) {
        self.navigationController = navigationController
        self.service = service
    }

    func start() {
        let vm = RepoListViewModel(service: service)
        vm.onSelectRepo = { [weak self] repo in      // weak: coordinator → nav → VC → VM → closure → coordinator
            self?.showDetail(repo)
        }
        navigationController.pushViewController(RepoListViewController(viewModel: vm), animated: false)
    }

    private func showDetail(_ repo: Repo) {
        let detail = UIViewController()
        detail.title = repo.name
        navigationController.pushViewController(detail, animated: true)
    }
}

// SceneDelegate (composition root):
// appCoordinator = AppCoordinator(window: window, service: GitHubRepoService(session: .shared))
// appCoordinator.start()      ← SceneDelegate phải GIỮ strong appCoordinator
```

**Các vấn đề bộ nhớ hay gặp:**
1. **Quên giữ child** → coordinator bị giải phóng ngay sau `start()`, closure `[weak self]` thành nil, bấm cell không có gì xảy ra.
2. **Quên remove child** khi luồng kết thúc → leak cả cây coordinator.
3. **User bấm nút Back hệ thống / swipe back** → coordinator không biết màn đã đóng. Cách xử lý: làm coordinator là `UINavigationControllerDelegate`, trong `navigationController(_:didShow:animated:)` kiểm tra `fromVC` bị pop (`!navigationController.viewControllers.contains(fromVC)`) rồi gọi `onFinish`.
4. Closure VM → coordinator phải `[weak self]` vì coordinator giữ nav → nav giữ VC → VC giữ VM → VM giữ closure.

**Coordinator + deep link:** `AppCoordinator.handle(.repo(owner:name:))` → chọn tab, push đúng màn. Đây là lợi ích lớn mà VC tự điều hướng không làm được gọn.

**SwiftUI:** không cần Coordinator kiểu UIKit; dùng `NavigationStack(path:)` với một `Router` `@Observable` giữ `path: [Route]` — cùng ý tưởng: điều hướng là **state** tách khỏi view.

### §12. Modularization với Swift Package Manager (giới thiệu)

Khi app lớn: chia code thành nhiều **local Swift package** trong cùng repo.
```
RepoScope/
├── App/                      ← target app: chỉ có composition root, AppCoordinator
└── Packages/
    ├── Core/                 ← Networking, Persistence, DesignSystem (không phụ thuộc feature)
    ├── Domain/               ← Entity, UseCase, Repository protocol
    ├── FeatureSearch/        ← phụ thuộc Domain, Core
    └── FeatureFavorites/     ← phụ thuộc Domain, Core — KHÔNG phụ thuộc FeatureSearch
```
```swift
// Packages/FeatureSearch/Package.swift
// swift-tools-version:5.9
import PackageDescription
let package = Package(
    name: "FeatureSearch",
    platforms: [.iOS(.v17)],
    products: [.library(name: "FeatureSearch", targets: ["FeatureSearch"])],
    dependencies: [.package(path: "../Domain"), .package(path: "../Core")],
    targets: [
        .target(name: "FeatureSearch", dependencies: ["Domain", "Core"]),
        .testTarget(name: "FeatureSearchTests", dependencies: ["FeatureSearch"]),
    ]
)
```
**Lợi ích:** compiler **ép** dependency rule (feature A không `import` được feature B nếu không khai báo); access control `internal` thật sự có ý nghĩa (chỉ `public` mới lộ ra); build song song + incremental nhanh hơn; test từng module riêng (`swift test`), preview SwiftUI nhanh hơn; ownership rõ ràng cho team.
**Chi phí:** phải nghĩ API `public`, quản lý graph phụ thuộc, feature cần gọi nhau phải qua interface/coordinator ở App.

---

## ⚠️ Bẫy hay gặp

1. **"MVVM là kiến trúc tốt nhất"** — sai tư duy. Luôn nói trade-off và bối cảnh.
2. **ViewModel import UIKit, giữ `UIImage`/`UIColor`** — mất lợi ích test được độc lập; trả về `String`, enum trạng thái, hoặc `Data`.
3. **VM giữ strong reference tới VC** (ví dụ `var view: RepoListViewController`) → retain cycle. Trong MVP thì `weak var view`.
4. **Quên `[weak self]` trong closure binding** — VC giữ VM, VM giữ closure, closure giữ VC → leak cả màn hình.
5. **Massive ViewModel** — chuyển hết code từ VC sang VM không phải là refactor. Tách tiếp: UseCase, Formatter, Validator.
6. **Coordinator quên add/remove child** → bị dealloc sớm hoặc leak. Back gesture hệ thống không đi qua coordinator.
7. **Nhầm Dependency Inversion với Dependency Injection.**
8. **DI container dùng như Service Locator** (`Container.shared.resolve` khắp nơi) — phụ thuộc bị ẩn, giống singleton.
9. **Over-engineering:** UseCase chỉ có 1 dòng gọi repository cho mọi tính năng, protocol cho mọi class kể cả không bao giờ mock. Nói được "khi nào KHÔNG cần" là dấu hiệu senior.
10. **VIPER với SwiftUI** — SwiftUI view đã tự quản state; VIPER ép vào thường gây vướng víu.
11. **`@Observable` vẫn cần `@MainActor`** cho VM phục vụ UI, nếu không Swift 6 sẽ báo lỗi khi cập nhật từ task nền.

---

## 🛠 Bài tập

> Code đặt trong `practice/week15/`. Code không dùng UIKit chạy bằng `swift file.swift`; code UIKit dùng project RepoScope hoặc Playground iOS.

**Bài 1 — Mổ xẻ Massive VC (T2)**
Mở một VC dài nhất bạn từng viết (hoặc đoạn MVC ở §2). Liệt kê **mọi trách nhiệm** của nó, đánh dấu cái nào chuyển sang: Service, ViewModel, Formatter, Coordinator. Viết ra `day1.md`.
<details><summary>Gợi ý</summary>

Với §2: networking + parsing → `RepoService`; sort → ViewModel (hoặc UseCase nếu dùng nhiều nơi); format sao → `RepoRowViewModel`; navigation → Coordinator; error handling → ViewModel đổi sang `State.failed`; table view data source → giữ ở VC (đó là việc của View).
</details>

**Bài 2 — Refactor sang MVVM (T3)**
Không nhìn §3, tự viết `RepoListViewModel` (không import UIKit) cho tính năng: tìm repo, sắp xếp sao giảm dần, format `1500 → "★ 1.5k"`, state `idle/loading/loaded/failed`. Viết **3 unit test**: thành công + sort đúng, lỗi → `.failed`, format `999 → "★ 999"` và `1000 → "★ 1.0k"`.
<details><summary>Lời giải (phần test format)</summary>

```swift
func test_rowViewModel_formatsStars() {
    XCTAssertEqual(RepoRowViewModel(repo: Repo(id: 1, name: "a", stargazersCount: 999)).starsText, "★ 999")
    XCTAssertEqual(RepoRowViewModel(repo: Repo(id: 1, name: "a", stargazersCount: 1000)).starsText, "★ 1.0k")
}
```
Chú ý test đặt tên theo hành vi `test_<unit>_<điều kiện>_<kết quả>` — tuần 16 sẽ học kỹ.
</details>

**Bài 3 — Binding 3 kiểu (T3)**
Viết cùng một `CounterViewModel` (tăng/giảm, `text: "Count: 3"`) theo 3 cách: closure `onChange`, Combine `@Published`, `@Observable`. Với mỗi cách, viết 3 dòng nhận xét: View subscribe thế nào, có cần `[weak self]` không, nhiều subscriber được không.
<details><summary>Gợi ý</summary>

- Closure: `var onChange: ((String) -> Void)?` — 1 subscriber, cần `[weak self]` ở VC.
- Combine: `@Published private(set) var text` — nhiều subscriber, cần `[weak self]` trong `sink` + giữ `AnyCancellable`.
- `@Observable`: SwiftUI đọc trực tiếp `vm.text`, không cần weak; ngoài SwiftUI dùng `withObservationTracking` (chỉ báo 1 lần, phải đăng ký lại).
</details>

**Bài 4 — Clean layers (T4)**
Viết `FavoriteRepoUseCase` với rule: không cho favorite quá 50 repo (throw `FavoriteError.limitReached`), không favorite trùng. Domain định nghĩa `protocol FavoritesRepository { func all() async -> [Repo]; func add(_: Repo) async throws }`. Viết `InMemoryFavoritesRepository` (actor) cho test. Test 3 trường hợp.
<details><summary>Lời giải</summary>

```swift
enum FavoriteError: Error, Equatable { case limitReached, duplicate }

protocol FavoritesRepository: Sendable {
    func all() async -> [Repo]
    func add(_ repo: Repo) async throws
}

actor InMemoryFavoritesRepository: FavoritesRepository {
    private var storage: [Repo] = []
    init(_ initial: [Repo] = []) { storage = initial }
    func all() -> [Repo] { storage }
    func add(_ repo: Repo) { storage.append(repo) }
}

struct FavoriteRepoUseCase {
    let repository: FavoritesRepository
    let limit = 50

    func execute(_ repo: Repo) async throws {
        let current = await repository.all()
        guard !current.contains(where: { $0.id == repo.id }) else { throw FavoriteError.duplicate }
        guard current.count < limit else { throw FavoriteError.limitReached }
        try await repository.add(repo)
    }
}
```
Business rule nằm ở UseCase, không ở ViewModel, không ở Core Data → đổi DB không ảnh hưởng rule.
</details>

**Bài 5 — SOLID (T5)**
Cho class sau, chỉ ra nguyên tắc SOLID nào bị vi phạm và refactor:
```swift
final class ReportManager {
    func generate(type: String, data: [Int]) -> String {
        if type == "csv" { return data.map(String.init).joined(separator: ",") }
        else if type == "json" { return "[" + data.map(String.init).joined(separator: ",") + "]" }
        return ""
    }
    func save(_ text: String) { try? text.write(toFile: "/tmp/report", atomically: true, encoding: .utf8) }
    func email(_ text: String) { print("send email", text) }
}
```
<details><summary>Lời giải</summary>

- **S**: 3 trách nhiệm (format, lưu, gửi) → 3 type.
- **O**: thêm `xml` phải sửa `if/else` → protocol `ReportFormatter` với `CSVFormatter`, `JSONFormatter`.
- **D**: `save` dùng file system cứng → inject `ReportStorage` protocol.
```swift
protocol ReportFormatter { func format(_ data: [Int]) -> String }
struct CSVFormatter: ReportFormatter { func format(_ d: [Int]) -> String { d.map(String.init).joined(separator: ",") } }
struct JSONFormatter: ReportFormatter { func format(_ d: [Int]) -> String { "[" + CSVFormatter().format(d) + "]" } }
protocol ReportStorage { func save(_ text: String) throws }
protocol ReportSender { func send(_ text: String) }

struct ReportService {
    let formatter: ReportFormatter
    let storage: ReportStorage
    let sender: ReportSender
    func run(_ data: [Int]) throws {
        let text = formatter.format(data)
        try storage.save(text)
        sender.send(text)
    }
}
```
</details>

**Bài 6 — DI (T6)**
a) Viết `Container` ở §10, thêm hỗ trợ **singleton scope**: `register(_:scope: .singleton)` trả về cùng một instance mỗi lần resolve.
b) Trả lời: vì sao khi `resolve` thiếu type thì app crash lúc runtime, còn constructor injection thuần thì không?
<details><summary>Lời giải a)</summary>

```swift
enum Scope { case transient, singleton }

final class Container {
    private var factories: [ObjectIdentifier: () -> Any] = [:]
    private var singletons: [ObjectIdentifier: Any] = [:]

    func register<T>(_ type: T.Type, scope: Scope = .transient, factory: @escaping () -> T) {
        let key = ObjectIdentifier(type)
        switch scope {
        case .transient:
            factories[key] = factory
        case .singleton:
            factories[key] = { [unowned self] in
                if let existing = self.singletons[key] { return existing }
                let instance = factory()
                self.singletons[key] = instance
                return instance
            }
        }
    }

    func resolve<T>(_ type: T.Type) -> T {
        guard let value = factories[ObjectIdentifier(type)]?() as? T else { fatalError("Chưa register \(T.self)") }
        return value
    }
}
```
b) Container tra cứu bằng dictionary lúc chạy — compiler không biết key đã đăng ký chưa. Constructor injection thì thiếu tham số init là lỗi compile.
</details>

**Bài 7 — Coordinator (T7)**
Viết `AuthCoordinator` (Login → OTP) là child của `AppCoordinator`. Khi OTP xong: `AuthCoordinator` gọi `onFinish`, `AppCoordinator` remove child và chuyển root sang `MainCoordinator`. Đặt `deinit { print("AuthCoordinator deinit") }` và xác nhận nó được in.
<details><summary>Gợi ý</summary>

- `AppCoordinator.showAuth()`: tạo child, `child.onFinish = { [weak self, weak child] in self?.removeChild(child!); self?.showMain() }`, `addChild`, `start()`.
- Nếu `deinit` không in: kiểm tra closure VM → coordinator có `[weak self]` chưa, đã `removeChild` chưa. Dùng Memory Graph Debugger (tuần 5) để thấy ai còn giữ.
</details>

**🏋️ Bài tập lớn (T7) — Refactor RepoScope (Side project: Milestone 15)**
Refactor app RepoScope hiện có:
- **Domain**: `Repo`, `RepoRepository` protocol, `SearchReposUseCase`
- **Data**: `RepoRepositoryImpl` dùng `APIClient` (tạm thời có thể dùng URLSession đơn giản, tuần 16 làm lại)
- **Presentation**: `RepoListViewModel`, `RepoDetailViewModel` (không import UIKit nếu dùng UIKit; dùng `@Observable` nếu SwiftUI)
- **Coordinator** (UIKit) hoặc `Router` `@Observable` + `NavigationStack(path:)` (SwiftUI)
- **Composition root** trong `SceneDelegate`/`App`: chỗ duy nhất tạo object thật
- ≥ 5 unit test cho ViewModel + UseCase dùng stub repository
- Vẽ sơ đồ kiến trúc vào `README.md` của repo (ASCII là đủ) — **nhà tuyển dụng sẽ đọc nó**

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 ⭐ Vì sao cần kiến trúc? Kiến trúc tốt là gì?**
<details><summary>Đáp án gợi ý</summary>

Để code dễ **thay đổi**, dễ **test** và nhiều người làm cùng lúc được. Kiến trúc tốt: mỗi thành phần một trách nhiệm rõ ràng, phụ thuộc qua abstraction nên thay thế/mock được, logic nghiệp vụ test được không cần UI, và nhất quán trong toàn app. Nó luôn có giá (nhiều file, boilerplate) nên phải chọn mức độ phù hợp với kích thước app và team.
</details>

**Q2 ⭐ Massive View Controller là gì? Bạn xử lý thế nào?**
<details><summary>Đáp án gợi ý</summary>

Là khi VC ôm networking, parsing, business logic, format dữ liệu, điều hướng, data source... vì trong UIKit, VC gắn chặt với view và lifecycle nên mọi thứ dễ dồn vào đó. Hậu quả: file nghìn dòng, không test được, conflict liên tục. Xử lý: tách networking ra Service/Repository, logic trạng thái + format ra ViewModel, điều hướng ra Coordinator, data source/cell config ra class riêng, chia màn lớn thành child VC. VC chỉ còn bind và render.
</details>

**Q3 ⭐ MVVM là gì? ViewModel chứa gì và không chứa gì?**
<details><summary>Đáp án gợi ý</summary>

Model–View–ViewModel: View hiển thị và chuyển sự kiện; ViewModel giữ state của màn hình, presentation logic, gọi service, và expose dữ liệu đã format cho View qua binding; Model là dữ liệu và nghiệp vụ. VM chứa state (loading/loaded/error), text đã format, validation, xử lý action của user. VM **không** chứa UIKit (UIView, UIColor), không giữ reference tới View, không tự push màn hình — báo ra cho Coordinator. Nhờ vậy VM test được bằng unit test thuần.
</details>

**Q4 ⭐ Các cách binding giữa View và ViewModel? Bạn chọn cách nào?**
<details><summary>Đáp án gợi ý</summary>

Closure/delegate (đơn giản, không phụ thuộc framework, nhưng verbose và 1 subscriber), Combine `@Published` (mạnh với luồng sự kiện như debounce/combineLatest, nhưng khó debug, phải quản lý `AnyCancellable`), RxSwift (tương tự, thư viện ngoài), và `@Observable` iOS 17 (SwiftUI tự track đúng property được đọc, ít boilerplate). Với SwiftUI mới tôi chọn `@Observable`; UIKit đơn giản tôi chọn closure; nếu có tìm kiếm debounce hoặc form validation nhiều field thì Combine.
</details>

**Q5 MVVM khác MVP thế nào?**
<details><summary>Đáp án gợi ý</summary>

Trong MVP, Presenter **biết View qua protocol** (weak) và gọi trực tiếp `view.show(...)` — View thụ động. Trong MVVM, ViewModel **không biết View**; View tự quan sát state của VM qua binding. MVP dễ test bằng mock view protocol, không cần cơ chế binding; MVVM hợp với framework reactive/SwiftUI hơn và VM tái sử dụng được cho nhiều view.
</details>

**Q6 ⭐ VIPER gồm những gì? Ưu nhược?**
<details><summary>Đáp án gợi ý</summary>

View (hiển thị), Interactor (business logic của use case), Presenter (nhận sự kiện từ View, gọi Interactor, format dữ liệu, nhờ Router điều hướng), Entity (model), Router (tạo module và điều hướng). Ưu: trách nhiệm rất rõ, test từng phần, hợp team lớn. Nhược: boilerplate cực nhiều (5 file + nhiều protocol cho một màn đơn giản), nhiều tham chiếu chéo cần `weak`, không hợp SwiftUI, learning curve cho người mới.
</details>

**Q7 ⭐ Clean Architecture là gì? Dependency rule là gì?**
<details><summary>Đáp án gợi ý</summary>

Chia app thành các lớp đồng tâm: Domain (Entity, UseCase, Repository protocol — Swift thuần), Data (implement repository: API, DB, DTO mapping), Presentation (View, ViewModel). Dependency rule: phụ thuộc chỉ hướng vào trong — Domain không biết gì về UI, network, database. Data implement protocol mà Domain định nghĩa (dependency inversion). Lợi ích: nghiệp vụ test độc lập, đổi nguồn dữ liệu hay UI framework không ảnh hưởng domain.
</details>

**Q8 UseCase và Repository khác nhau thế nào? Lúc nào bỏ UseCase?**
<details><summary>Đáp án gợi ý</summary>

Repository trả lời "lấy/lưu dữ liệu ở đâu" — che giấu nguồn (API, cache, DB) và mapping DTO → Entity. UseCase trả lời "nghiệp vụ là gì" — kết hợp một hoặc nhiều repository và áp rule (giới hạn 50 favorite, merge cờ favorite vào kết quả search). Nếu UseCase chỉ là một dòng gọi thẳng repository thì nó không thêm giá trị; nhiều team cho ViewModel gọi repository trực tiếp và chỉ tạo UseCase khi có logic thật hoặc dùng lại ở nhiều nơi.
</details>

**Q9 TCA là gì? Khác MVVM ở đâu?**
<details><summary>Đáp án gợi ý</summary>

The Composable Architecture của Point-Free, theo mô hình unidirectional data flow (giống Redux): State là struct, Action là enum sự kiện, Reducer là hàm duy nhất được đổi state và trả về Effect cho side effect, Store điều phối. Khác MVVM: state chỉ đổi qua một chỗ nên dự đoán được, effect được mô hình hoá rõ, test bằng TestStore kiểm tra từng action, feature compose được. Nhược: phụ thuộc thư viện bên thứ ba lớn, API thay đổi nhiều, learning curve cao.
</details>

**Q10 ⭐ Team 3 người và team 15 người, bạn chọn kiến trúc gì?**
<details><summary>Đáp án gợi ý</summary>

Team 3 người, cần ship nhanh: MVVM (+Coordinator nếu UIKit), Service/Repository qua protocol để test, không ép UseCase. Team 15 người: Clean-ish (Domain/Data/Presentation) với MVVM, Coordinator, modularize theo feature bằng SPM để giảm conflict và tăng tốc build, DI rõ ràng; TCA nếu team sẵn sàng đầu tư. Tiêu chí: kích thước team, độ phức tạp nghiệp vụ, UIKit hay SwiftUI, kinh nghiệm hiện có của team — và nhất quán quan trọng hơn chọn "đúng".
</details>

**Q11 ⭐ Giải thích SOLID, mỗi nguyên tắc một ví dụ iOS.**
<details><summary>Đáp án gợi ý</summary>

- **S**: một type một lý do thay đổi — tách `ImageDownloader`, `ImageCache` thay vì một `ImageManager` làm tất cả.
- **O**: mở rộng không sửa — thêm loại discount bằng struct mới conform `DiscountRule` thay vì thêm `case` vào `switch`.
- **L**: subclass thay được superclass — `Penguin: Bird` mà `fly()` crash là vi phạm; tách `Flyable`.
- **I**: protocol nhỏ — `UITableViewDataSource` tách `UITableViewDelegate`; `Codable = Encodable & Decodable`.
- **D**: phụ thuộc abstraction — ViewModel phụ thuộc `RepoServiceProtocol` chứ không phải `URLSession`, inject mock khi test.
</details>

**Q12 Dependency Injection là gì? Có mấy kiểu? Dùng kiểu nào nhiều nhất?**
<details><summary>Đáp án gợi ý</summary>

Là truyền dependency từ bên ngoài vào thay vì object tự tạo — để thay thế được (mock khi test, đổi implementation) và phụ thuộc hiện rõ. Ba kiểu: constructor (ưu tiên — bắt buộc, dùng `let`, compiler đảm bảo đủ), property (khi không kiểm soát init như VC từ storyboard, hoặc phụ thuộc tuỳ chọn; dễ quên gán), method (dependency chỉ dùng cho một lần gọi). Object graph được lắp ở một composition root như SceneDelegate/AppCoordinator.
</details>

**Q13 Dependency Inversion và Dependency Injection khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Inversion là **nguyên tắc thiết kế**: module cấp cao và cấp thấp cùng phụ thuộc abstraction; abstraction do phía cấp cao (domain) định nghĩa. Injection là **kỹ thuật** truyền dependency vào object. Có thể inject một class cụ thể (có Injection nhưng không có Inversion); thường dùng Injection để đạt Inversion.
</details>

**Q14 DI container có ưu nhược gì? Singleton có xấu không?**
<details><summary>Đáp án gợi ý</summary>

Container (Swinject, Factory) giảm boilerplate khi graph lớn, quản lý scope (singleton, per-screen). Nhược: lỗi thiếu đăng ký phát hiện lúc runtime, "magic" khó lần theo, dễ biến thành Service Locator nếu gọi `resolve` khắp nơi. Singleton không xấu tự thân (`URLSession.shared`, `UserDefaults.standard`) — xấu khi **truy cập ẩn** bên trong class vì không thay được khi test và state chia sẻ giữa các test. Cách dung hoà: vẫn có thể có một instance duy nhất nhưng **inject** nó qua init (thường qua protocol).
</details>

**Q15 ⭐ Coordinator pattern giải quyết vấn đề gì? Quản lý bộ nhớ thế nào?**
<details><summary>Đáp án gợi ý</summary>

Tách điều hướng khỏi VC: VC/VM chỉ báo sự kiện ("chọn repo X"), coordinator quyết định tạo và hiển thị màn tiếp theo, inject dependency cho nó. VC tái sử dụng được, luồng đổi dễ, deep link xử lý tập trung. Về bộ nhớ: parent giữ strong `childCoordinators`; khi luồng con xong phải remove child, nếu không sẽ leak; nếu không add thì child bị dealloc ngay. Closure từ VM về coordinator phải `[weak self]`. Back gesture của hệ thống không qua coordinator nên cần `UINavigationControllerDelegate` để phát hiện VC bị pop và dọn child.
</details>

**Q16 Trong SwiftUI có cần Coordinator không?**
<details><summary>Đáp án gợi ý</summary>

Không cần kiểu UIKit. SwiftUI điều hướng bằng state: `NavigationStack(path:)` với một `Router` `@Observable` chứa `path: [Route]` (enum Hashable) và `.navigationDestination(for:)`. Ý tưởng giống coordinator — tách quyết định điều hướng khỏi view, deep link chỉ việc gán path. Với app hybrid UIKit+SwiftUI, nhiều team vẫn dùng coordinator UIKit để host `UIHostingController`.
</details>

**Q17 Modularization bằng SPM mang lại gì?**
<details><summary>Đáp án gợi ý</summary>

Ranh giới được compiler ép (module không khai báo phụ thuộc thì không import được), access control có ý nghĩa (`internal` ẩn chi tiết, chỉ `public` lộ ra), build incremental/song song nhanh hơn, test từng module độc lập, SwiftUI preview nhanh, ownership rõ cho từng team. Chi phí: phải thiết kế API public, quản lý graph phụ thuộc, tránh phụ thuộc vòng (feature gọi nhau qua interface hoặc qua tầng App).
</details>

**Q18 Làm sao biết một ViewModel đã quá lớn? Xử lý thế nào?**
<details><summary>Đáp án gợi ý</summary>

Dấu hiệu: hàng trăm dòng, nhiều dependency trong init (>4–5), test setup phức tạp, nhiều phần state không liên quan nhau. Xử lý: tách business logic ra UseCase, format ra Formatter/RowViewModel, validation ra Validator, chia màn hình thành nhiều component con mỗi cái một VM nhỏ (child VM), dùng state enum thay vì nhiều Bool rời rạc.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Bài tập lớn: RepoScope đã refactor (Domain/Data/Presentation + DI + Coordinator/Router), có ≥ 5 unit test, README có sơ đồ kiến trúc
- [ ] Trả lời được ≥ 14/18 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Refactor một Massive VC sang MVVM — bạn làm từng bước thế nào?"
- [ ] Ghi âm 2 phút: "Team 3 người vs 15 người chọn kiến trúc gì?"
- [ ] Vẽ được sơ đồ VIPER và Clean trên giấy trắng trong 2 phút
- [ ] Làm xong 6 bài DSA tuần 15 (Graph BFS/DFS)
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [Apple — Observation framework](https://developer.apple.com/documentation/observation)
- [objc.io — Lighter View Controllers](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/) (bài kinh điển về Massive VC)
- [Soroush Khanlou — The Coordinator](https://khanlou.com/2015/01/the-coordinator/) và [Coordinators Redux](https://khanlou.com/2015/10/coordinators-redux/)
- [Uncle Bob — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [objc.io — Architecting iOS Apps with VIPER](https://www.objc.io/issues/13-architecture/viper/)
- [Point-Free — The Composable Architecture (GitHub)](https://github.com/pointfreeco/swift-composable-architecture) — chỉ đọc README + ví dụ Counter
- [Swift.org — Package Manager](https://www.swift.org/documentation/package-manager/)
- Sách: *Advanced iOS App Architecture* (Kodeco) — chương MVVM, Elements of Architecture
