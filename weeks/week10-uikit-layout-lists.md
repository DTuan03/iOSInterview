# Tuần 10 — Auto Layout · UITableView/UICollectionView · Diffable · Scroll performance (11/01 – 17/01/2027)

> **Giai đoạn 3 · UI** · Thời lượng: ~14 giờ
> Gần như mọi app đều là "một cái list". Người phỏng vấn mức 2 năm sẽ hỏi: *constraint xung đột thì debug thế nào, hugging vs compression là gì, cell reuse hoạt động ra sao và vì sao ảnh hiện sai cell, diffable data source khác `reloadData` chỗ nào, list giật thì bạn làm gì?* Tuần này trả lời hết, và cuối tuần RepoScope có một list đúng chuẩn production: compositional layout + diffable + pagination + prefetch + downsampling.

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Giải thích Auto Layout giải hệ phương trình tuyến tính thế nào, constraint = `item1.attr = multiplier × item2.attr + constant`
- [ ] Giải thích **intrinsic content size**, **content hugging** vs **compression resistance**, priority (1000/750/250)
- [ ] Phân biệt **safe area** và **layout margins**
- [ ] Debug được constraint **ambiguous** và **unsatisfiable** (đọc log, symbolic breakpoint, identifier, View Debugger)
- [ ] Nêu trade-off **code vs Storyboard/XIB**
- [ ] Dùng `UIStackView` đúng (axis, alignment, distribution, ẩn view)
- [ ] Giải thích **cơ chế cell reuse**, sửa được bug "ảnh hiện sai cell", biết `prepareForReuse` nên/không nên làm gì
- [ ] Làm **self-sizing cell**
- [ ] Dựng `UICollectionViewCompositionalLayout` (item → group → section, orthogonal scrolling, header)
- [ ] Dùng **diffable data source**: snapshot, item identifier đúng cách, `reconfigureItems` vs `reloadItems`
- [ ] Làm **prefetching + pagination** không gọi trùng
- [ ] Kể ≥ 6 kỹ thuật tối ưu scroll, giải thích **downsampling bằng ImageIO** và vì sao ảnh tốn RAM
- [ ] Xử lý bàn phím che input (`keyboardLayoutGuide` và cách cổ điển)
- [ ] Làm xong 6 bài DSA tuần 10 (Trees — BFS)

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 11/01** | 2h | §1 **Auto Layout từ gốc**: constraint, intrinsic size, **hugging vs compression**, priority, safe area, margins. Bài tập 1 · DSA: Binary Tree Level Order Traversal | `HeaderView` viết bằng code |
| **T3 12/01** | 2h | §2 Debug ambiguous/unsatisfiable. §3 Code vs Storyboard. §4 UIStackView. Bài tập 2 · DSA: Binary Tree Right Side View | Ghi chú cách đọc log constraint |
| **T4 13/01** | 2h | §5 **UITableView / UICollectionView, cell reuse** (quan trọng nhất tuần), self-sizing. Bài tập 3 · DSA: Minimum Depth of Binary Tree | Sửa xong bug ảnh sai cell |
| **T5 14/01** | 2h | §6 **Compositional layout**. §7 **Diffable data source**. Bài tập 4, 5 · DSA: Average of Levels in Binary Tree | Màn grid + featured |
| **T6 15/01** | 1,5h | §8 Prefetching & pagination. §9 **Scroll performance + downsampling**. §10 Bàn phím. Bài tập 6, 7 · DSA: Binary Tree Zigzag Level Order Traversal | `downsample.swift` chạy được |
| **T7 16/01** | 3,5h | **Bài tập lớn** + **Side project: Milestone 10** (compositional + diffable + pagination + prefetch + downsampling). DSA: Populating Next Right Pointers in Each Node | Commit side project |
| **CN 17/01** | 2h | Tự trả lời 20 câu phỏng vấn. Nói lại 3 câu. Đo scroll RepoScope bằng Instruments (nếu có máy thật). Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

> Code UIKit gõ vào project `practice/week10/ListLab`. Mọi đoạn đã được typecheck với iOS SDK (Swift 6 mode, deployment iOS 17). Các con số "log thật" là chạy trên iPhone simulator iOS 26.

### §1. Auto Layout từ gốc

**Ý tưởng:** thay vì tính `frame` bằng tay, bạn mô tả **quan hệ** giữa các view. Mỗi constraint là một phương trình (hoặc bất phương trình) tuyến tính:

```
item1.attribute  (= | ≤ | ≥)  multiplier × item2.attribute + constant
   button.leading       =          1.0     × view.leading      + 16
```
Engine (thuật toán **Cassowary**) giải hệ phương trình này trong mỗi layout pass (tuần 9, §4: update constraints → layout) và ra `frame` cho từng view. Để một view có vị trí xác định, cần đủ thông tin cho **4 ẩn**: x, y, width, height.

| Hệ phương trình | Trạng thái | Biểu hiện |
|---|---|---|
| Đủ, không mâu thuẫn | ✅ Satisfiable & unambiguous | Đúng |
| **Thiếu** — nhiều nghiệm | ⚠️ **Ambiguous** | View nhảy lung tung, size 0, khác nhau giữa các lần chạy. **Không có log!** |
| **Thừa & mâu thuẫn** | ❌ **Unsatisfiable** | Log "Unable to simultaneously satisfy constraints", UIKit tự bẻ một constraint |

**`translatesAutoresizingMaskIntoConstraints`**: view tạo bằng code mặc định `true` → UIKit tự sinh constraint từ `frame` hiện tại → **xung đột** với constraint của bạn. Luôn đặt `false` cho view bạn tự viết constraint (view từ Interface Builder thì IB tự đặt).

**Intrinsic content size** — kích thước "tự nhiên" view biết từ nội dung: `UILabel` (theo text + font), `UIButton`, `UIImageView` (theo ảnh), `UISwitch`. Có intrinsic size thì chỉ cần constraint **vị trí**, không cần width/height. `UIView` trơn **không** có (trả `UIView.noIntrinsicMetric`). View custom: override `intrinsicContentSize` và gọi `invalidateIntrinsicContentSize()` khi nội dung đổi.

**Priority** — mỗi constraint có priority 1…1000:
- `1000` = `.required` (mặc định): **bắt buộc**.
- `< 1000` = optional: engine **cố gắng** thoả, càng cao càng được ưu tiên; không thoả được thì thoả "gần nhất có thể".
- Mốc có tên: `.defaultHigh` = 750, `.defaultLow` = 250, `.fittingSizeLevel` = 50.

**Content hugging vs compression resistance** — hai "constraint ngầm" sinh từ intrinsic size:

```
              Hugging ("ôm"): KHÔNG MUỐN BỊ KÉO GIÃN lớn hơn intrinsic size
              ──►│ [ Text ] │◄──           priority mặc định 250 (label: 251 khi tạo bằng IB)

    Compression resistance: KHÔNG MUỐN BỊ ÉP nhỏ hơn intrinsic size
              ◄──│[ Text ]│──►            priority mặc định 750
```
- Khi **thừa** chỗ → view có **hugging thấp hơn** bị kéo giãn.
- Khi **thiếu** chỗ → view có **compression resistance thấp hơn** bị ép (label bị cắt "…").

**Ví dụ kinh điển** — một hàng: `[tên repo dài ...........][★ 12k]` rộng 200pt. Log thật:
```
default priorities: hug 250.0 compress 750.0      ← hai label như nhau → engine TỰ CHỌN ai bị ép
before: title 192.0  badge 0.0                     ← badge "★ 12k" bị ép mất hẳn!
--- badge.setContentCompressionResistancePriority(.required, for: .horizontal)
--- badge.setContentHuggingPriority(.required, for: .horizontal)
after:  title 144.7  badge 47.3                    ← badge giữ đúng intrinsic width, title bị cắt "…"
```
Quy tắc: **phần thông tin quan trọng/ngắn** (badge, giá tiền, nút) → compression cao + hugging cao; **phần co giãn được** (tiêu đề dài) → để thấp.

**Safe area vs layout margins:**
| | Safe area (`safeAreaLayoutGuide`, `safeAreaInsets`) | Layout margins (`layoutMarginsGuide`, `directionalLayoutMargins`) |
|---|---|---|
| Là gì | Vùng **không bị che** bởi notch, Dynamic Island, status bar, nav bar, tab bar, home indicator | **Khoảng đệm nội dung** bên trong một view (padding) |
| Ai quyết định | Hệ thống + container (VC con thừa hưởng) | Bạn (mặc định 8pt; root view của VC theo `systemMinimumLayoutMargins` 16/20pt) |
| Dùng khi | Ghim nội dung chính của màn hình | Căn lề nhất quán bên trong card/cell |

`directionalLayoutMargins` dùng `leading/trailing` → tự đảo khi ngôn ngữ RTL (tiếng Ả Rập). Luôn dùng `leading/trailing` thay cho `left/right`.

```swift
import UIKit

final class RepoHeaderView: UIView {
    private let avatar = UIImageView()
    private let nameLabel = UILabel()
    private let starsLabel = UILabel()
    private let descriptionLabel = UILabel()

    override init(frame: CGRect) {
        super.init(frame: frame)
        [avatar, nameLabel, starsLabel, descriptionLabel].forEach {
            $0.translatesAutoresizingMaskIntoConstraints = false   // BẮT BUỘC khi tự viết constraint
            addSubview($0)
        }
        descriptionLabel.numberOfLines = 0                         // cho phép xuống dòng → intrinsic height thay đổi theo width
        directionalLayoutMargins = NSDirectionalEdgeInsets(top: 12, leading: 16, bottom: 12, trailing: 16)
        let margins = layoutMarginsGuide

        // Tên repo được phép bị cắt (…), số sao thì KHÔNG
        starsLabel.setContentCompressionResistancePriority(.required, for: .horizontal)
        starsLabel.setContentHuggingPriority(.required, for: .horizontal)

        NSLayoutConstraint.activate([
            avatar.leadingAnchor.constraint(equalTo: margins.leadingAnchor),
            avatar.topAnchor.constraint(equalTo: margins.topAnchor),
            avatar.widthAnchor.constraint(equalToConstant: 40),
            avatar.heightAnchor.constraint(equalTo: avatar.widthAnchor),

            nameLabel.leadingAnchor.constraint(equalTo: avatar.trailingAnchor, constant: 12),
            nameLabel.centerYAnchor.constraint(equalTo: avatar.centerYAnchor),

            starsLabel.leadingAnchor.constraint(greaterThanOrEqualTo: nameLabel.trailingAnchor, constant: 8),
            starsLabel.trailingAnchor.constraint(equalTo: margins.trailingAnchor),
            starsLabel.firstBaselineAnchor.constraint(equalTo: nameLabel.firstBaselineAnchor),

            descriptionLabel.topAnchor.constraint(equalTo: avatar.bottomAnchor, constant: 8),
            descriptionLabel.leadingAnchor.constraint(equalTo: margins.leadingAnchor),
            descriptionLabel.trailingAnchor.constraint(equalTo: margins.trailingAnchor),
            descriptionLabel.bottomAnchor.constraint(equalTo: margins.bottomAnchor),
        ])
    }
    required init?(coder: NSCoder) { fatalError() }
}

final class ScreenVC: UIViewController {
    private let header = RepoHeaderView()
    override func viewDidLoad() {
        super.viewDidLoad()
        header.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(header)
        let safe = view.safeAreaLayoutGuide                         // tránh notch, Dynamic Island, home indicator
        let preferredWidth = header.widthAnchor.constraint(equalToConstant: 600)
        preferredWidth.priority = .defaultHigh                      // 750: "muốn" rộng 600 nhưng nhường nếu không đủ chỗ
        NSLayoutConstraint.activate([
            header.topAnchor.constraint(equalTo: safe.topAnchor),
            header.centerXAnchor.constraint(equalTo: safe.centerXAnchor),
            header.leadingAnchor.constraint(greaterThanOrEqualTo: safe.leadingAnchor),   // required: không tràn
            preferredWidth,
        ])
    }
}
```
Điểm đáng chú ý:
- Chiều cao `RepoHeaderView` **không** được đặt — nó được suy ra từ chuỗi dọc `top → avatar → description → bottom` + intrinsic height của label. Đây chính là nguyên lý của **self-sizing cell** (§5).
- `preferredWidth` priority 750 + `leading ≥ safe.leading` (required): trên iPad rộng 600, trên iPhone co lại vừa màn. Mẫu "**required bất đẳng thức + optional đẳng thức**" rất hay dùng.
- `NSLayoutConstraint.activate([...])` hiệu quả hơn bật `isActive = true` từng cái.

### §2. Debug constraint ambiguous & unsatisfiable

**Unsatisfiable** — console in:
```
Unable to simultaneously satisfy constraints.
    Probably at least one of the constraints in the following list is one you don't want. ...
(
    "<NSLayoutConstraint:0x600... 'box.width.100' UIView:0x7f...width == 100   (active)>",
    "<NSLayoutConstraint:0x600... 'box.width.200' UIView:0x7f...width == 200   (active)>"
)
Will attempt to recover by breaking constraint
<NSLayoutConstraint:0x600... 'box.width.200' UIView:0x7f...width == 200   (active)>
Make a symbolic breakpoint at UIViewAlertForUnsatisfiableConstraints to catch this in the debugger.
```
Quy trình:
1. Đặt **`identifier`** cho constraint quan trọng (`w1.identifier = "box.width.100"`) và `accessibilityIdentifier` cho view → log dễ đọc hơn địa chỉ `0x7f...`.
2. Tìm các constraint mâu thuẫn trong danh sách. Thủ phạm hay gặp: **`NSAutoresizingMaskLayoutConstraint`** (quên `translatesAutoresizingMaskIntoConstraints = false`), `UIView-Encapsulated-Layout-Height/Width` (cell/table áp chiều cao cố định trong khi constraint bên trong đòi khác → hạ priority constraint dọc cuối cùng xuống 999), constraint chiều cao cố định + chuỗi dọc.
3. **Symbolic breakpoint** `UIViewAlertForUnsatisfiableConstraints` → dừng đúng lúc xảy ra; trong lldb: `po [[UIWindow keyWindow] _autolayoutTrace]` (Obj-C) để in cây view có đánh dấu `AMBIGUOUS LAYOUT`.
4. Sửa: bỏ constraint thừa, hoặc **hạ priority** một bên để nó thành optional.
5. Dán log vào [wtfautolayout.com](https://www.wtfautolayout.com) để xem dạng dễ đọc.

**Ambiguous** — **không có log**, chỉ có UI sai. Công cụ:
- **Xcode View Debugger** (Debug View Hierarchy): view ambiguous có biểu tượng **!** tím; Size Inspector hiện constraint đang áp dụng.
- `view.hasAmbiguousLayout` (chỉ dùng khi debug), `view.exerciseAmbiguityInLayout()` → view nhảy sang nghiệm khác, nhìn ra trục bị thiếu.
- `view.constraintsAffectingLayout(for: .horizontal)`.

Nguyên nhân ambiguous hay gặp: thiếu constraint một trục (có `top` nhưng không có `leading`/`centerX`); hai label cùng hugging/compression trên một hàng (như ví dụ §1 — engine tự chọn, kết quả "ổn" nhưng không xác định); `≥`/`≤` mà không có đẳng thức neo lại.

### §3. Code vs Storyboard/XIB

| | Storyboard | XIB | Code (programmatic) |
|---|---|---|---|
| Xem trực quan | ✅ Cả flow | ✅ Một view | ❌ (có `#Preview` cho UIKit từ Xcode 15) |
| **Merge conflict** | ❌ Rất tệ (XML lớn, nhiều người sửa) | ⚠️ Vừa | ✅ Diff dễ đọc |
| Review code | Khó | Khó | Dễ |
| Tái sử dụng | Kém | Tốt (view/cell) | Tốt nhất |
| Dependency injection | Khó (`init(coder:)`, phải dùng `instantiateViewController(identifier:creator:)` iOS 13+) | Như storyboard | ✅ `init(viewModel:)` tự nhiên |
| Lỗi runtime | Outlet mất kết nối → crash; segue identifier là string | Outlet | Compiler bắt lỗi |
| Tốc độ mở/build | Storyboard lớn mở chậm | Nhanh | Nhanh |

**Câu trả lời mẫu:** *"Team nhiều người mình chọn code (hoặc XIB cho vài cell phức tạp) vì merge conflict, review và dependency injection. Storyboard hợp prototype hoặc app nhỏ một người làm. Dự án cũ có storyboard thì tách nhỏ mỗi màn một storyboard, dùng `instantiateViewController(identifier:creator:)` để inject dependency."* Với SwiftUI ngày nay, UI mới thường là SwiftUI, phần UIKit còn lại viết bằng code.

### §4. UIStackView

`UIStackView` là view **không vẽ gì** (không có background trước iOS 14), chỉ tự sinh constraint cho các `arrangedSubviews`.

| Thuộc tính | Ý nghĩa |
|---|---|
| `axis` | `.horizontal` / `.vertical` — trục chính |
| `distribution` | Cách chia **trục chính**: `.fill` (dựa vào hugging/compression — ai hugging thấp nhất bị giãn), `.fillEqually`, `.fillProportionally`, `.equalSpacing`, `.equalCentering` |
| `alignment` | Cách căn **trục phụ**: `.fill`, `.leading/.top`, `.center`, `.trailing/.bottom`, `.firstBaseline` |
| `spacing`, `setCustomSpacing(_:after:)` | Khoảng cách |
| `isLayoutMarginsRelativeArrangement` | Dùng layout margins làm padding |

```swift
@MainActor func makeStatsRow() -> UIStackView {
    let stars = UILabel(); stars.text = "★ 12k"
    let forks = UILabel(); forks.text = "⑂ 1.2k"
    let lang = UILabel(); lang.text = "Swift"
    let stack = UIStackView(arrangedSubviews: [stars, forks, lang])
    stack.axis = .horizontal
    stack.spacing = 12
    stack.alignment = .firstBaseline        // căn theo trục phụ (dọc)
    stack.distribution = .fill              // chia theo trục chính: dựa vào intrinsic size + hugging/compression
    stack.isLayoutMarginsRelativeArrangement = true
    stack.directionalLayoutMargins = .init(top: 4, leading: 8, bottom: 4, trailing: 8)
    stack.setCustomSpacing(24, after: forks)
    return stack
}
// Ẩn một view trong stack: forks.isHidden = true → stack tự gỡ khỏi layout (không cần đổi constraint)
```
Bẫy: `addSubview` vào stack view **không** được sắp xếp — phải `addArrangedSubview`. `removeArrangedSubview` chỉ gỡ khỏi sắp xếp, view **vẫn còn** trong hierarchy → gọi thêm `removeFromSuperview()`. Stack lồng 4–5 tầng trong cell → nhiều constraint, layout chậm khi scroll; cell phức tạp nên viết constraint trực tiếp.

### §5. UITableView / UICollectionView và cell reuse

**Hai protocol:**
- **Data source** — *dữ liệu*: bao nhiêu section/row, cell nào ở index nào (`numberOfRowsInSection`, `cellForRowAt`).
- **Delegate** — *hành vi & hiển thị*: chọn row, chiều cao, `willDisplay`, swipe action, scroll (`UIScrollViewDelegate`).

**Cơ chế reuse:**
```
          màn hình hiển thị ~10 cell
   ┌────────────────────────┐
   │ cell #1 (đang hiện)    │   cuộn lên ↑
   │ ...                    │
   │ cell #10               │
   └────────────────────────┘
        │ cell #1 trượt khỏi màn hình
        ▼
   Reuse queue (theo reuseIdentifier): [cell #1] ──► prepareForReuse()
        │
        ▼  dequeueReusableCell(withIdentifier:for:) cho row 11
   cell cũ được LẤY RA, configure lại với dữ liệu row 11
```
- Tạo cell (alloc + dựng subview + constraint) **rất tốn** → list 10.000 dòng chỉ cần ~15 cell object.
- `dequeueReusableCell(withIdentifier:for:)` (bản có `for:`) **luôn** trả cell (tạo mới nếu queue rỗng) — yêu cầu đã `register`. Bản không có `for:` có thể trả `nil`.
- Hệ quả: **cell là object tái chế**, mang theo trạng thái cũ. Hai bug kinh điển:
  1. **Trạng thái "dính"**: `if repo.isStarred { star.isHidden = false }` mà không có `else` → cell tái chế từ row có sao vẫn hiện sao. → `configure` phải set **toàn bộ** trạng thái, mọi nhánh.
  2. **Ảnh hiện sai cell**: cell bắt đầu tải ảnh cho row 3, user cuộn, cell được tái sử dụng cho row 20, ảnh row 3 tải xong → gán vào cell đang hiển thị row 20. → **huỷ** request trong `prepareForReuse` **và** kiểm tra cell vẫn đại diện cho item đó trước khi gán.

`prepareForReuse()` nên: huỷ task/request async, reset thứ **không** được set lại trong configure (animation đang chạy, trạng thái chọn tuỳ chỉnh). **Không** nên dồn hết việc reset text/màu vào đây thay cho configure đầy đủ — Apple khuyến nghị reset ở `cellForRowAt`. Không làm việc nặng (nó được gọi rất nhiều).

```swift
import UIKit

struct Repo: Hashable, Sendable {
    let id: Int
    var name: String
    var stars: Int
    var avatarURL: URL
}

final class RepoCell: UITableViewCell {
    static let reuseID = "RepoCell"
    private var imageTask: Task<Void, Never>?
    private var representedID: Int?

    func configure(with repo: Repo, loader: ImageLoader) {
        representedID = repo.id
        var content = defaultContentConfiguration()
        content.text = repo.name
        content.secondaryText = "★ \(repo.stars)"
        content.image = UIImage(systemName: "photo")              // placeholder
        contentConfiguration = content

        imageTask = Task { [weak self] in
            guard let image = await loader.image(for: repo.avatarURL) else { return }
            guard let self, self.representedID == repo.id else { return }   // cell đã bị tái sử dụng cho repo khác?
            var updated = content
            updated.image = image
            self.contentConfiguration = updated
        }
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        imageTask?.cancel()                                        // huỷ request của "đời trước"
        imageTask = nil
        representedID = nil
        // KHÔNG reset text/màu ở đây làm nhiệm vụ chính — configure() phải set lại TOÀN BỘ trạng thái
    }
}

actor ImageLoader {
    private var cache: [URL: UIImage] = [:]
    func image(for url: URL) async -> UIImage? {
        if let cached = cache[url] { return cached }
        guard let (data, _) = try? await URLSession.shared.data(from: url),
              let image = UIImage(data: data) else { return nil }
        let prepared = await image.byPreparingForDisplay()        // decode ở nền, không decode trên main lúc vẽ
        cache[url] = prepared ?? image
        return prepared ?? image
    }
}

final class RepoTableViewController: UITableViewController {
    private var repos: [Repo] = []
    private let loader = ImageLoader()

    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.register(RepoCell.self, forCellReuseIdentifier: RepoCell.reuseID)
        // Self-sizing: mặc định rowHeight = automaticDimension từ iOS 11; chỉ cần constraint đầy đủ trong contentView
        tableView.rowHeight = UITableView.automaticDimension
        tableView.estimatedRowHeight = 72
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { repos.count }

    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: RepoCell.reuseID, for: indexPath) as! RepoCell
        cell.configure(with: repos[indexPath.row], loader: loader)
        return cell
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
    }
}
```
Ghi chú: `contentConfiguration` / `defaultContentConfiguration()` (iOS 14+) thay cho `textLabel`/`imageView` (deprecated). Dùng tốt cho cell đơn giản; cell custom thì subclass và tự dựng subview **trong `contentView`** (không add thẳng vào cell — sẽ không co lại khi vào edit mode/swipe).

**Self-sizing cell:**
1. `rowHeight = UITableView.automaticDimension` (mặc định), `estimatedRowHeight` > 0 (ước lượng càng sát, scrollbar càng ít nhảy).
2. Constraint trong `contentView` phải tạo **chuỗi dọc liền mạch** từ `contentView.top` tới `contentView.bottom`, và mọi view trong chuỗi có chiều cao xác định (intrinsic hoặc constraint).
3. Label nhiều dòng: `numberOfLines = 0`.
4. Nếu log báo xung đột với `UIView-Encapsulated-Layout-Height` (thường khi tính toán lần đầu với chiều cao ước lượng) → hạ priority constraint `bottom` xuống 999.
5. Nội dung trong cell đổi chiều cao sau khi đã hiện (expand "Xem thêm") → cập nhật model rồi `tableView.performBatchUpdates(nil)` (hoặc `reconfigureItems` với diffable) để table tính lại.

Với `UICollectionView`: compositional layout dùng `.estimated(...)` cho dimension là self-sizing (§6).

**Table vs Collection:** từ iOS 14, `UICollectionView` + `UICollectionLayoutListConfiguration` làm được mọi thứ của table (swipe, accessory, header) **cộng** grid/orthogonal. Code mới: ưu tiên collection view. Table view vẫn gặp nhiều trong codebase cũ và phỏng vấn.

### §6. UICollectionViewCompositionalLayout

Mô hình 4 tầng:
```
 Layout ─► Section ─► Group ─► Item
                       (horizontal / vertical / custom)
 Kích thước (NSCollectionLayoutDimension):
   .fractionalWidth(0.5)   — tỉ lệ so với container (của group/section)
   .absolute(160)          — điểm cố định
   .estimated(120)         — ước lượng, cell tự đo (self-sizing)
```
```swift
enum Section: Int, CaseIterable, Sendable { case featured, all }

@MainActor func makeLayout() -> UICollectionViewCompositionalLayout {
    UICollectionViewCompositionalLayout { sectionIndex, environment in
        switch Section(rawValue: sectionIndex)! {
        case .featured:
            // Card ngang cuộn riêng (orthogonal scrolling) — giống App Store
            let item = NSCollectionLayoutItem(layoutSize: .init(widthDimension: .fractionalWidth(1),
                                                                heightDimension: .fractionalHeight(1)))
            let group = NSCollectionLayoutGroup.horizontal(
                layoutSize: .init(widthDimension: .fractionalWidth(0.85), heightDimension: .absolute(160)),
                subitems: [item])
            let section = NSCollectionLayoutSection(group: group)
            section.orthogonalScrollingBehavior = .groupPagingCentered
            section.interGroupSpacing = 12
            section.contentInsets = .init(top: 8, leading: 16, bottom: 8, trailing: 16)
            let header = NSCollectionLayoutBoundarySupplementaryItem(
                layoutSize: .init(widthDimension: .fractionalWidth(1), heightDimension: .estimated(44)),
                elementKind: UICollectionView.elementKindSectionHeader, alignment: .top)
            section.boundarySupplementaryItems = [header]
            return section
        case .all:
            // Grid: 2 cột trên iPhone dọc, 4 cột khi màn rộng; chiều cao tự co giãn (estimated)
            let columns = environment.container.effectiveContentSize.width > 700 ? 4 : 2
            let item = NSCollectionLayoutItem(layoutSize: .init(widthDimension: .fractionalWidth(1.0 / CGFloat(columns)),
                                                                heightDimension: .estimated(120)))
            let group = NSCollectionLayoutGroup.horizontal(
                layoutSize: .init(widthDimension: .fractionalWidth(1), heightDimension: .estimated(120)),
                repeatingSubitem: item, count: columns)
            group.interItemSpacing = .fixed(8)
            let section = NSCollectionLayoutSection(group: group)
            section.interGroupSpacing = 8
            section.contentInsets = .init(top: 8, leading: 16, bottom: 8, trailing: 16)
            return section
        }
    }
}

// Danh sách kiểu UITableView bằng compositional layout (iOS 14+)
@MainActor func makeListLayout() -> UICollectionViewLayout {
    var config = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
    config.trailingSwipeActionsConfigurationProvider = { indexPath in
        UISwipeActionsConfiguration(actions: [UIContextualAction(style: .destructive, title: "Xoá") { _, _, done in done(true) }])
    }
    return UICollectionViewCompositionalLayout.list(using: config)
}
```
So với `UICollectionViewFlowLayout`: flow layout chỉ là "xếp lần lượt rồi xuống dòng", muốn section ngang cuộn riêng phải **lồng collection view trong cell** (phức tạp, khó reuse). Compositional làm được bằng `orthogonalScrollingBehavior`, mỗi section một layout riêng, tự thích ứng theo `environment` (size class, xoay màn).

### §7. Diffable data source

**Vấn đề của cách cũ:** dữ liệu nằm ở mảng của bạn, UI nằm trong collection view — hai "nguồn sự thật". Gọi `insertItems`/`deleteItems` sai số lượng → crash kinh điển:
```
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Invalid update:
invalid number of items in section 0. The number of items contained in an existing section after the
update (11) must be equal to the number of items contained in that section before the update (10),
plus or minus the number of items inserted or deleted from that section (0 inserted, 0 deleted)...'
```
Còn `reloadData()` thì an toàn nhưng **mất animation** và reload mọi cell.

**Diffable (iOS 13+):** bạn mô tả **trạng thái mong muốn** bằng một `NSDiffableDataSourceSnapshot<Section, Item>`, gọi `apply`, framework **tự tính diff** (insert/delete/move) và animate. Một nguồn sự thật duy nhất: snapshot.

- `SectionIdentifier` và `ItemIdentifier` phải **`Hashable`**, và **duy nhất** trong snapshot — ID trùng → crash (`Fatal: supplied item identifiers are not unique`).
- **Nên dùng ID (vd. `Repo.ID`) làm item identifier**, không dùng cả struct model. Lý do: nếu identifier là cả struct, đổi `stars` → hash đổi → diffable coi là **xoá item cũ + chèn item mới** (animation sai, mất selection, cell nhấp nháy). Dùng ID: identity ổn định; nội dung đổi thì báo bằng `reconfigureItems`.
- `apply` có thể gọi từ background queue (nhưng phải **luôn** cùng một queue) — đơn giản nhất là luôn ở main. Snapshot lớn (hàng chục nghìn item) thì diff tốn CPU.

| | `reloadItems` | `reconfigureItems` (iOS 15+) |
|---|---|---|
| Cách làm | Dequeue **cell mới** (hoặc cell khác) và configure lại | Gọi lại cell registration/provider trên **cell hiện có** |
| Chi phí | Cao hơn, có thể nháy | Rẻ, giữ trạng thái cell (đang animation, focus) |
| Dùng khi | Đổi **loại** cell / kích thước lớn | Đổi **nội dung** (số sao, tim, tiến độ) |

```swift
import UIKit

struct Repo: Identifiable, Sendable {
    let id: Int
    var name: String
    var stars: Int
    var isStarredByMe: Bool
}

enum Section: Hashable, Sendable { case main }

@MainActor
final class RepoStore {
    private(set) var reposByID: [Repo.ID: Repo] = [:]
    func upsert(_ repos: [Repo]) { repos.forEach { reposByID[$0.id] = $0 } }
    func repo(_ id: Repo.ID) -> Repo? { reposByID[id] }
}

final class RepoCollectionViewController: UIViewController, UICollectionViewDelegate {
    // Item identifier = Repo.ID (Int), KHÔNG phải cả struct Repo
    private typealias DataSource = UICollectionViewDiffableDataSource<Section, Repo.ID>
    private typealias Snapshot = NSDiffableDataSourceSnapshot<Section, Repo.ID>

    private let store = RepoStore()
    private var collectionView: UICollectionView!
    private var dataSource: DataSource!
    private var orderedIDs: [Repo.ID] = []

    override func viewDidLoad() {
        super.viewDidLoad()
        collectionView = UICollectionView(frame: view.bounds,
                                          collectionViewLayout: UICollectionViewCompositionalLayout.list(
                                            using: .init(appearance: .plain)))
        collectionView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        collectionView.delegate = self
        view.addSubview(collectionView)

        // Cell registration (iOS 14+): không cần reuse identifier string, không cần cast
        let registration = UICollectionView.CellRegistration<UICollectionViewListCell, Repo.ID> { [weak self] cell, _, id in
            guard let repo = self?.store.repo(id) else { return }
            var content = cell.defaultContentConfiguration()
            content.text = repo.name
            content.secondaryText = "★ \(repo.stars)"
            content.image = UIImage(systemName: repo.isStarredByMe ? "star.fill" : "star")
            cell.contentConfiguration = content
        }
        dataSource = DataSource(collectionView: collectionView) { collectionView, indexPath, id in
            collectionView.dequeueConfiguredReusableCell(using: registration, for: indexPath, item: id)
        }
    }

    // Trang đầu / pull-to-refresh: thay toàn bộ
    func show(_ repos: [Repo]) {
        store.upsert(repos)
        orderedIDs = repos.map(\.id)
        var snapshot = Snapshot()
        snapshot.appendSections([.main])
        snapshot.appendItems(orderedIDs, toSection: .main)
        dataSource.apply(snapshot, animatingDifferences: true)    // tự tính diff: insert/delete/move có animation
    }

    // Pagination: nối thêm
    func append(_ more: [Repo]) {
        store.upsert(more)
        let newIDs = more.map(\.id).filter { !orderedIDs.contains($0) }   // ID trùng trong snapshot → crash!
        orderedIDs += newIDs
        var snapshot = dataSource.snapshot()
        snapshot.appendItems(newIDs, toSection: .main)
        dataSource.apply(snapshot, animatingDifferences: false)
    }

    // Nội dung của 1 item đổi (user bấm star): ID không đổi → diff không thấy gì → phải báo
    func toggleStar(id: Repo.ID) {
        guard var repo = store.repo(id) else { return }
        repo.isStarredByMe.toggle()
        repo.stars += repo.isStarredByMe ? 1 : -1
        store.upsert([repo])
        var snapshot = dataSource.snapshot()
        snapshot.reconfigureItems([id])                           // iOS 15+: cập nhật cell HIỆN TẠI, không dequeue cell mới
        dataSource.apply(snapshot, animatingDifferences: false)
    }

    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        guard let id = dataSource.itemIdentifier(for: indexPath) else { return }
        toggleStar(id: id)
    }
}
```
Lưu ý: `orderedIDs.contains` là O(n) — với hàng nghìn item nên giữ thêm một `Set<Repo.ID>`. `CellRegistration` phải tạo **bên ngoài** closure cell provider (tạo trong closure → iOS 15+ báo lỗi/crash vì mỗi lần một registration mới).

### §8. Prefetching & pagination

**Prefetching** (`UITableViewDataSourcePrefetching` / `UICollectionViewDataSourcePrefetching`, iOS 10+): collection view báo trước các index path **sắp** hiện (theo hướng cuộn) → bắt đầu tải dữ liệu/ảnh sớm; `cancelPrefetchingForItemsAt` khi user đổi hướng. Prefetch chỉ là **gợi ý** — không được gọi cho mọi cell (vd. lần hiển thị đầu, VoiceOver, cuộn rất nhanh), nên `cellForItemAt` vẫn phải tự tải được.

Từ iOS 15 còn có **cell prefetching** (`isPrefetchingEnabled`, mặc định bật): collection view chuẩn bị cell sớm trong lúc rảnh → `cellForItemAt` có thể được gọi **trước** khi cell hiện một lúc; đừng giả định "cellForItemAt = cell đang hiện" (analytics impression nên đặt ở `willDisplay`).

**Pagination** — 3 điều kiện để không lỗi:
1. **Chặn gọi trùng**: cờ `isLoading`.
2. **Biết khi hết**: `hasMore` (trang rỗng hoặc API trả `next` = nil).
3. **Trigger sớm**: khi còn ~5 item nữa mới tới cuối (trong prefetch hoặc `willDisplay`), không đợi chạm đáy.
Thêm: lỗi thì **không** tăng số trang, hiện cell "Thử lại"; dùng cursor/`since` thay cho số trang nếu dữ liệu đổi liên tục (tránh trùng/sót item khi có item mới chen vào đầu).

```swift
import UIKit

struct Repo: Sendable { let id: Int; let name: String; let avatarURL: URL }

protocol RepoService: Sendable { func fetch(page: Int) async throws -> [Repo] }

actor ThumbnailPipeline {
    private var tasks: [URL: Task<UIImage?, Never>] = [:]
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL, pointSize: CGSize, scale: CGFloat) async -> UIImage? {
        if let img = cache[url] { return img }
        if let running = tasks[url] { return await running.value }     // đang tải → chờ chung, không tải 2 lần
        let task = Task<UIImage?, Never> {
            guard let (data, _) = try? await URLSession.shared.data(from: url) else { return nil }
            return await UIImage(data: data)?.byPreparingThumbnail(ofSize:
                CGSize(width: pointSize.width * scale, height: pointSize.height * scale))
        }
        tasks[url] = task
        let img = await task.value
        tasks[url] = nil
        if let img { cache[url] = img }
        return img
    }

    func prefetch(_ url: URL, pointSize: CGSize, scale: CGFloat) {
        guard cache[url] == nil, tasks[url] == nil else { return }
        Task { _ = await image(for: url, pointSize: pointSize, scale: scale) }
    }

    func cancel(_ url: URL) {
        tasks[url]?.cancel()
        tasks[url] = nil
    }
}

final class FeedViewController: UIViewController, UICollectionViewDataSourcePrefetching, UICollectionViewDelegate {
    private var collectionView: UICollectionView!
    private var repos: [Repo] = []
    private let pipeline = ThumbnailPipeline()
    private let service: RepoService
    private var nextPage = 1
    private var isLoading = false
    private var hasMore = true
    private let thumbSize = CGSize(width: 60, height: 60)

    init(service: RepoService) {
        self.service = service
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: UICollectionViewFlowLayout())
        collectionView.prefetchDataSource = self
        collectionView.delegate = self
        view.addSubview(collectionView)
        loadNextPage()
    }

    // MARK: Prefetching — gọi TRƯỚC khi cell xuất hiện (theo hướng cuộn)
    func collectionView(_ collectionView: UICollectionView, prefetchItemsAt indexPaths: [IndexPath]) {
        let scale = traitCollection.displayScale
        for ip in indexPaths where ip.item < repos.count {
            let url = repos[ip.item].avatarURL
            Task { await pipeline.prefetch(url, pointSize: thumbSize, scale: scale) }
        }
        if indexPaths.contains(where: { $0.item >= repos.count - 5 }) { loadNextPage() }
    }

    func collectionView(_ collectionView: UICollectionView, cancelPrefetchingForItemsAt indexPaths: [IndexPath]) {
        for ip in indexPaths where ip.item < repos.count {
            let url = repos[ip.item].avatarURL
            Task { await pipeline.cancel(url) }
        }
    }

    // MARK: Pagination — dự phòng khi prefetch tắt (VoiceOver, cuộn rất nhanh)
    func collectionView(_ collectionView: UICollectionView, willDisplay cell: UICollectionViewCell,
                        forItemAt indexPath: IndexPath) {
        if indexPath.item == repos.count - 1 { loadNextPage() }
    }

    private func loadNextPage() {
        guard !isLoading, hasMore else { return }          // chặn gọi trùng
        isLoading = true
        Task {
            defer { isLoading = false }
            do {
                let page = try await service.fetch(page: nextPage)
                hasMore = !page.isEmpty
                nextPage += 1
                repos += page
                // thực tế: apply snapshot diffable (append) thay vì reloadData
                collectionView.reloadData()
            } catch {
                // hiện cell "Thử lại" ở cuối; KHÔNG tăng nextPage
            }
        }
    }
}
```
Vì sao `isLoading` không bị race ở đây? `FeedViewController` là `UIViewController` → `@MainActor`; `Task { }` tạo trong nó cũng chạy trên MainActor → mọi đọc/ghi `isLoading` tuần tự trên main (tuần 7–8). `ThumbnailPipeline` là actor nên `tasks`/`cache` an toàn, và gom request trùng URL (giống bài M2).

### §9. Scroll performance

Mục tiêu: mỗi khung hình ≤ 16,7ms (60Hz) / 8,3ms (120Hz) — tuần 9 §8. Khi scroll, main thread phải: dequeue + configure cell, layout cell mới, decode ảnh, commit. Danh sách kỹ thuật (hãy thuộc ≥ 6):

| # | Kỹ thuật | Vì sao |
|---|---|---|
| 1 | **Downsampling** ảnh về đúng kích thước hiển thị | Ảnh 4000×3000 decode ra **48MB** RAM dù chỉ hiện ở ô 100pt |
| 2 | **Decode ảnh ở background** (`byPreparingForDisplay`, `byPreparingThumbnail`, ImageIO) | `UIImage(data:)` **chưa** decode; decode xảy ra lúc vẽ lần đầu **trên main** → hitch |
| 3 | **Cache** ảnh đã decode (`NSCache` — tự giải phóng khi thiếu RAM) + gom request trùng | Cuộn lên xuống không tải/decode lại |
| 4 | `cellForRowAt` **nhẹ**: không parse JSON, không format ngày (DateFormatter tạo 1 lần), không truy vấn DB | Được gọi mỗi lần cell xuất hiện |
| 5 | **Giảm layout**: hierarchy phẳng, tránh stack view lồng sâu, không đổi constraint mỗi lần configure (bật/tắt bằng `isHidden` hoặc giữ sẵn 2 bộ constraint) | Auto Layout giải hệ phương trình mỗi pass |
| 6 | **Cache chiều cao** cell (với tính tay) hoặc `estimatedRowHeight` sát thực tế | Tránh tính lại chiều cao, scrollbar nhảy |
| 7 | Tránh **offscreen rendering** (shadowPath, không mask/cornerRadius+clip trên view phức tạp) & blending (view **opaque**, `backgroundColor` không trong suốt) | Render hitch (tuần 9) |
| 8 | **Prefetching** dữ liệu & ảnh | Có sẵn trước khi cell hiện |
| 9 | Dùng `reconfigureItems` thay `reloadData` | Không dequeue/layout lại toàn bộ |
| 10 | **Đo** bằng Instruments: Animation Hitches, Time Profiler, Allocations; trên simulator: Color Blended/Off-screen | Đừng tối ưu theo cảm giác |

**Vì sao ảnh tốn RAM?** RAM của ảnh đã decode **không phụ thuộc dung lượng file** mà phụ thuộc **số pixel**: `width × height × 4 byte` (RGBA). File JPEG 2MB, kích thước 4000×3000 → 4000 × 3000 × 4 = **48MB** khi decode. 20 ảnh như thế trong list → gần 1GB → app bị hệ thống kill (jetsam). Downsample về ô 100×100pt @3x (300px cạnh dài) → 300×225×4 ≈ **270KB**.

**Downsampling bằng ImageIO** (WWDC18 *Image and Graphics Best Practices*). Code dưới **chạy được trên Mac** (`swift -swift-version 6 downsample.swift`):
```swift
import Foundation
import ImageIO
import CoreGraphics
import UniformTypeIdentifiers

/// Giải mã ảnh ở kích thước NHỎ ngay từ file/data, không bao giờ decode ảnh full-size.
/// - pointSize: kích thước hiển thị (point); scale: 2 hoặc 3 trên iPhone.
func downsample(imageAt url: URL, to pointSize: CGSize, scale: CGFloat) -> CGImage? {
    // 1. Không cache ảnh gốc đã decode khi tạo source
    let sourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let source = CGImageSourceCreateWithURL(url as CFURL, sourceOptions) else { return nil }

    let maxPixel = max(pointSize.width, pointSize.height) * scale
    let thumbOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,          // decode NGAY lúc tạo thumbnail (ở thread hiện tại = nền)
        kCGImageSourceCreateThumbnailWithTransform: true,    // tôn trọng EXIF orientation
        kCGImageSourceThumbnailMaxPixelSize: maxPixel,
    ] as CFDictionary
    return CGImageSourceCreateThumbnailAtIndex(source, 0, thumbOptions)
}

// --- Demo: tạo ảnh 4000x3000 rồi downsample về ô 100x100pt @3x ---
func makeBigPNG(at url: URL) {
    let w = 4000, h = 3000
    let ctx = CGContext(data: nil, width: w, height: h, bitsPerComponent: 8, bytesPerRow: 0,
                        space: CGColorSpaceCreateDeviceRGB(),
                        bitmapInfo: CGImageAlphaInfo.premultipliedLast.rawValue)!
    ctx.setFillColor(CGColor(red: 0.2, green: 0.5, blue: 0.9, alpha: 1))
    ctx.fill(CGRect(x: 0, y: 0, width: w, height: h))
    let dest = CGImageDestinationCreateWithURL(url as CFURL, UTType.png.identifier as CFString, 1, nil)!
    CGImageDestinationAddImage(dest, ctx.makeImage()!, nil)
    CGImageDestinationFinalize(dest)
}

let url = URL(fileURLWithPath: NSTemporaryDirectory()).appendingPathComponent("big.png")
makeBigPNG(at: url)
if let thumb = downsample(imageAt: url, to: CGSize(width: 100, height: 100), scale: 3) {
    let fullBytes = 4000 * 3000 * 4
    let thumbBytes = thumb.width * thumb.height * 4
    print("thumbnail:", thumb.width, "x", thumb.height)
    print("RAM nếu decode full:", fullBytes / 1_000_000, "MB — thumbnail:", thumbBytes / 1000, "KB")
}
// thumbnail: 300 x 225
// RAM nếu decode full: 48 MB — thumbnail: 270 KB
```
Trên iOS, bọc kết quả bằng `UIImage(cgImage: thumb, scale: scale, orientation: .up)`. Chạy hàm này **trên background** (trong actor/`Task.detached`/`@concurrent` func), không trên main. Data từ network: dùng `CGImageSourceCreateWithData`. Từ iOS 15 có đường tắt `UIImage.byPreparingThumbnail(ofSize:)` (async) — nhưng nó cần `UIImage(data:)` trước; ImageIO tiết kiệm hơn vì không giữ ảnh full.

### §10. Bàn phím

Bàn phím hiện lên **che** nội dung phía dưới; bạn phải tự xử lý.

```swift
import UIKit

// Cách hiện đại (iOS 15+): keyboardLayoutGuide
final class ChatViewController: UIViewController {
    private let tableView = UITableView()
    private let inputBar = UIView()

    override func viewDidLoad() {
        super.viewDidLoad()
        [tableView, inputBar].forEach { $0.translatesAutoresizingMaskIntoConstraints = false; view.addSubview($0) }
        tableView.keyboardDismissMode = .interactive                       // kéo list xuống để ẩn bàn phím
        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: view.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: inputBar.topAnchor),
            inputBar.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            inputBar.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            inputBar.heightAnchor.constraint(equalToConstant: 52),
            // Khi bàn phím ẩn, guide này nằm ở safe area bottom; khi hiện, ở mép trên bàn phím (có animation)
            inputBar.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor),
        ])
    }
}

// Cách cổ điển: lắng nghe notification, chỉnh contentInset của scroll view
final class FormViewController: UIViewController {
    private let scrollView = UIScrollView()
    private var observers: [NSObjectProtocol] = []

    override func viewDidLoad() {
        super.viewDidLoad()
        scrollView.frame = view.bounds
        view.addSubview(scrollView)
        let center = NotificationCenter.default
        observers.append(center.addObserver(forName: UIResponder.keyboardWillChangeFrameNotification,
                                            object: nil, queue: .main) { [weak self] note in
            guard let endFrame = note.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect else { return }
            MainActor.assumeIsolated { self?.adjust(for: endFrame) }
        })
        observers.append(center.addObserver(forName: UIResponder.keyboardWillHideNotification,
                                            object: nil, queue: .main) { [weak self] _ in
            MainActor.assumeIsolated { self?.adjust(for: .zero) }
        })
    }

    private func adjust(for keyboardFrameInScreen: CGRect) {
        // Frame bàn phím ở toạ độ MÀN HÌNH → đổi sang toạ độ view rồi tính phần bị che
        let frameInView = view.convert(keyboardFrameInScreen, from: nil)
        let overlap = max(0, view.bounds.maxY - frameInView.minY - view.safeAreaInsets.bottom)
        scrollView.contentInset.bottom = overlap
        scrollView.verticalScrollIndicatorInsets.bottom = overlap
    }

    deinit { observers.forEach(NotificationCenter.default.removeObserver) }
}
```
Điểm cần nói được: frame bàn phím trong notification là **toạ độ màn hình** → phải `convert(_:from: nil)`; dùng `keyboardWillChangeFrame` (bắt cả đổi chiều cao khi bật gợi ý/emoji, bàn phím nổi trên iPad); scroll tới ô đang nhập bằng `scrollRectToVisible`; ẩn bàn phím bằng `view.endEditing(true)` hoặc `keyboardDismissMode`. `UITableViewController` tự xử lý inset bàn phím; `UIViewController` thường thì không.

---

## ⚠️ Bẫy hay gặp

1. **Quên `translatesAutoresizingMaskIntoConstraints = false`** → log xung đột với `NSAutoresizingMaskLayoutConstraint`, view không xuất hiện đúng.
2. **Ambiguous layout không có log** — chỉ thấy UI sai/khác nhau giữa các lần chạy. Dùng View Debugger.
3. **Hai label cùng hàng không đặt priority** → engine tự chọn ai bị cắt; trên ngôn ngữ/độ dài khác sẽ sai.
4. **Dùng `left/right` thay cho `leading/trailing`** → giao diện hỏng với RTL.
5. **Thêm view vào stack bằng `addSubview`**; `removeArrangedSubview` mà quên `removeFromSuperview`.
6. **`configure` không set đủ mọi nhánh** → trạng thái "dính" từ lần dùng trước của cell.
7. **Ảnh async gán vào cell đã bị tái sử dụng** → ảnh sai, nhấp nháy. Huỷ trong `prepareForReuse` + kiểm tra ID.
8. **Add subview thẳng vào cell** thay vì `contentView`.
9. **Self-sizing không chạy** vì chuỗi constraint dọc bị đứt (thiếu `bottom`), hoặc label `numberOfLines = 1`.
10. **Diffable: item identifier trùng** → crash; **dùng cả struct làm identifier** → đổi nội dung thành xoá + chèn.
11. **Tạo `CellRegistration` bên trong cell provider closure** → lỗi trên iOS 15+.
12. **Gọi API trang tiếp nhiều lần** vì không có `isLoading`; lỗi mạng mà vẫn tăng `page` → mất trang.
13. **`UIImage(data:)` tưởng là đã decode** → decode lười trên main lúc vẽ; ảnh lớn = hitch + RAM.
14. **Đặt analytics impression ở `cellForItemAt`** → sai vì cell prefetching (iOS 15) gọi sớm. Dùng `willDisplay`.

---

## 🛠 Bài tập

> Project `practice/week10/ListLab` (UIKit, code thuần, iOS 17+). Bài 6 là file `.swift` chạy trên Mac.

**Bài 1 — Hugging & compression (T2)**
Dựng hàng `[avatar 40×40] [tên repo] [★ stars]` theo §1 **không nhìn code**. Thử tên repo cực dài và cực ngắn, xoay ngang. Yêu cầu: (a) số sao luôn hiện đủ; (b) khi tên ngắn, khoảng trống nằm **giữa** tên và số sao (số sao dính phải); (c) thêm description nhiều dòng bên dưới, view tự cao theo nội dung.
<details><summary>Lời giải</summary>

Chính là `RepoHeaderView` ở §1. Điểm mấu chốt:
- (a) `starsLabel` compression resistance `.required` (> 750 của tên).
- (b) `starsLabel.leading ≥ nameLabel.trailing + 8` (bất đẳng thức) + `starsLabel.trailing = margins.trailing`; hugging của `starsLabel` cao nên nó không bị kéo giãn — khoảng thừa nằm ở bất đẳng thức.
- (c) `descriptionLabel.numberOfLines = 0` và chuỗi dọc liền `top → avatar → description → bottom`, không đặt height cố định.
Thử bỏ dòng compression của `starsLabel` để thấy nó bị ép về 0 như log ở §1.
</details>

**Bài 2 — Đọc log xung đột (T3)**
Cố ý tạo 3 lỗi rồi sửa, ghi lại log của từng lỗi: (a) quên `translatesAutoresizingMaskIntoConstraints = false` cho một view có width/height constraint; (b) `box.width == 100` và `box.width == 200` cùng required (đặt `identifier` cho cả hai); (c) chỉ có `top`, `width`, `height` (thiếu trục ngang) — log có in gì không? Tìm bằng View Debugger. Đặt symbolic breakpoint `UIViewAlertForUnsatisfiableConstraints`.
<details><summary>Lời giải</summary>

(a) Log liệt kê `NSAutoresizingMaskLayoutConstraint:... h=--& v=--& ...` mâu thuẫn với width/height của bạn → đặt `false`.
(b) Log "Unable to simultaneously satisfy constraints" liệt kê `'box.width.100'` và `'box.width.200'`, "Will attempt to recover by breaking constraint" một trong hai. Sửa: bỏ một cái, hoặc hạ priority một cái (vd. 999).
(c) **Không có log** — đây là ambiguous. View Debugger đánh dấu "!" tím (Ambiguous Position). Sửa: thêm `leading` hoặc `centerX`.
</details>

**Bài 3 — Bug ảnh sai cell (T4)**
Code cell sau gây ảnh nhấp nháy/sai khi cuộn nhanh và sao bị "dính". Tìm 3 lỗi và sửa.
```swift
final class BuggyCell: UITableViewCell {
    let avatar = UIImageView()
    let star = UIImageView(image: UIImage(systemName: "star.fill"))
    func configure(repo: Repo, isStarred: Bool) {
        textLabel?.text = repo.name
        if isStarred { star.isHidden = false }
        Task {
            let (data, _) = try await URLSession.shared.data(from: repo.avatarURL)
            avatar.image = UIImage(data: data)
        }
    }
}
```
<details><summary>Lời giải</summary>

1. `if isStarred` thiếu nhánh `else` → cell tái chế từ row có sao vẫn hiện sao. Sửa: `star.isHidden = !isStarred`.
2. Task không bị huỷ và không kiểm tra cell còn đại diện cho repo đó → ảnh của row cũ gán vào row mới. Sửa: giữ `imageTask`, `cancel()` trong `prepareForReuse`, lưu `representedID` và kiểm tra trước khi gán (như `RepoCell` §5); reset `avatar.image = nil`/placeholder khi configure.
3. `UIImage(data:)` decode lười trên main + ảnh full size → dùng `byPreparingThumbnail(ofSize:)` hoặc downsample ImageIO ở nền, có cache. (Thêm: `textLabel` đã deprecated — dùng `contentConfiguration` hoặc label riêng; add `avatar`/`star` vào `contentView`.)
</details>

**Bài 4 — Compositional layout (T5)**
Dựng màn "Explore" 2 section: **Featured** (card ngang 85% chiều rộng, paging, có header "Nổi bật") và **All** (grid 2 cột iPhone dọc / 4 cột khi rộng > 700pt, cell tự cao). Xoay máy để kiểm tra số cột.
<details><summary>Lời giải</summary>

`makeLayout()` ở §6. Nhớ `register` supplementary view cho `elementKindSectionHeader` và gán `dataSource.supplementaryViewProvider` (diffable) hoặc implement `viewForSupplementaryElementOfKind`. Số cột đổi khi xoay vì section provider được gọi lại với `environment` mới.
</details>

**Bài 5 — Diffable (T5)**
a) Làm `RepoCollectionViewController` (§7) với 50 repo giả. Bấm vào cell → toggle star bằng `reconfigureItems`.
b) Đổi item identifier sang **cả struct `Repo`** (cho `Repo: Hashable`). Toggle star: quan sát animation khác gì? Giải thích.
c) Cố ý `appendItems` hai lần cùng một ID — đọc thông báo crash.
<details><summary>Lời giải</summary>

b) Struct đổi `stars`/`isStarredByMe` → hash khác → snapshot mới có item "khác" → diffable coi là xoá item cũ + chèn item mới: cell nhấp nháy/fade, mất trạng thái selection, và nếu không apply snapshot mới hoàn chỉnh thì `reconfigureItems([repoCũ])` crash vì item không tồn tại. Dùng ID ổn định + `reconfigureItems` là cách đúng.
c) Crash vì identifier không duy nhất (thông báo dạng "supplied item identifiers are not unique" kèm danh sách ID trùng). Luôn khử trùng khi phân trang (API có thể trả lại item đã có khi dữ liệu dịch chuyển).
</details>

**Bài 6 — Downsampling (T6)**
Chạy `downsample.swift` ở §9 trên Mac. Rồi: (a) đổi `scale` thành 2, `pointSize` 60×60 — kích thước thumbnail là bao nhiêu? (b) tính RAM khi decode full một ảnh 12MP (4032×3024) và khi downsample về 80pt @3x. (c) Vì sao `kCGImageSourceShouldCache: false` ở source nhưng `kCGImageSourceShouldCacheImmediately: true` ở thumbnail?
<details><summary>Lời giải</summary>

(a) Cạnh dài tối đa 60 × 2 = 120px → 120 × 90 (tỉ lệ 4:3).
(b) Full: 4032 × 3024 × 4 ≈ 48,8MB. Downsample: cạnh dài 240px → 240 × 180 × 4 ≈ 173KB.
(c) Source không cache để **không bao giờ** giữ bản decode full-size; thumbnail decode ngay lúc tạo (trên thread nền đang gọi) để khi gán vào `UIImageView` trên main không phải decode nữa.
</details>

**Bài 7 — Form & bàn phím (T6)**
Form 8 `UITextField` trong `UIScrollView`. Khi focus vào ô cuối, ô đó phải nằm trên bàn phím. Làm 2 cách: (a) notification + `contentInset` (§10) + `scrollRectToVisible`; (b) ghim `scrollView.bottom` vào `view.keyboardLayoutGuide.top`. Nút "Return" chuyển sang ô kế tiếp (`becomeFirstResponder` — responder chain tuần 9).

**🏋️ Bài tập lớn (T7) — "GitHub Explore" offline**
Dựng màn danh sách repo đầy đủ chuẩn production, dùng dữ liệu giả (1000 repo, ảnh từ `https://picsum.photos/id/<n>/2000/2000` — cố ý ảnh lớn):
- `UICollectionView` + compositional layout: section "Featured" ngang + section "All" dạng list.
- **Diffable** data source, item identifier là `Repo.ID`, toggle star bằng `reconfigureItems`.
- **Pagination** 30 item/trang qua `FakeRepoService` (delay 500ms, 10% lỗi ngẫu nhiên) — có cell loading/"Thử lại" ở cuối, không gọi trùng, không trùng ID.
- **Prefetch** ảnh + huỷ prefetch; **downsampling** bằng ImageIO ở background; cache bằng `NSCache`.
- Cell không bao giờ hiện sai ảnh (cuộn thật nhanh để kiểm tra).
- Swift 6 mode, 0 warning.
- Đo: mở Xcode Debug Navigator → Memory khi cuộn hết 1000 item, so sánh **có** và **không** downsampling (ghi số MB vào README).

> **📱 Side project — Milestone 10 (T7):** chuyển màn list của RepoScope sang `UICollectionView` compositional layout + diffable data source, thêm pagination theo GitHub Search API (`page`/`per_page`, đọc header `Link`), prefetch avatar và downsampling. Chi tiết: [side-project](../tracks/side-project.md).

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** viết câu trả lời ra giấy/notes trước, sau đó mới mở đáp án để so sánh. Câu có ⭐ là câu rất hay gặp.

**Q1 Auto Layout hoạt động thế nào?**
<details><summary>Đáp án gợi ý</summary>

Mỗi constraint là một phương trình/bất phương trình tuyến tính `item1.attr = multiplier × item2.attr + constant` với priority. Engine (Cassowary) giải hệ để ra frame của mọi view trong layout pass. Mỗi view cần đủ thông tin cho x, y, width, height — từ constraint hoặc intrinsic content size. Thiếu → ambiguous, mâu thuẫn → unsatisfiable và UIKit bẻ một constraint.
</details>

**Q2 ⭐ Content hugging và compression resistance khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Cả hai là priority sinh từ intrinsic content size. Hugging: chống bị **kéo giãn** lớn hơn intrinsic size (mặc định 250). Compression resistance: chống bị **ép** nhỏ hơn (mặc định 750). Khi thừa chỗ, view hugging thấp hơn bị giãn; khi thiếu chỗ, view compression thấp hơn bị ép/cắt. Ví dụ hàng tên repo + badge: tăng cả hai priority của badge để nó luôn đủ, tên repo bị cắt "…".
</details>

**Q3 Intrinsic content size là gì? View custom muốn có thì làm sao?**
<details><summary>Đáp án gợi ý</summary>

Kích thước tự nhiên view biết từ nội dung — label theo text/font, button, image view theo ảnh — nên chỉ cần constraint vị trí. UIView trơn không có. View custom override `intrinsicContentSize` và gọi `invalidateIntrinsicContentSize()` khi nội dung đổi để Auto Layout tính lại.
</details>

**Q4 Priority của constraint dùng để làm gì? Cho ví dụ.**
<details><summary>Đáp án gợi ý</summary>

Priority 1000 là bắt buộc; dưới 1000 là tuỳ chọn — engine cố thoả, không được thì thoả gần nhất. Mẫu hay dùng: "required bất đẳng thức + optional đẳng thức", ví dụ width = 600 priority 750 và leading ≥ safe area (required): iPad rộng 600, iPhone co vừa màn. Hoặc hạ constraint bottom của cell xuống 999 để tránh xung đột với `UIView-Encapsulated-Layout-Height`.
</details>

**Q5 Safe area và layout margins khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

Safe area là vùng không bị che bởi notch, Dynamic Island, status/nav/tab bar, home indicator — do hệ thống và container quyết định. Layout margins là khoảng đệm nội dung bên trong một view, do mình đặt (`directionalLayoutMargins`). Ghim nội dung chính màn hình vào safe area, căn lề trong card/cell bằng margins. Dùng directional (leading/trailing) để hỗ trợ RTL.
</details>

**Q6 ⭐ Debug constraint lỗi thế nào?**
<details><summary>Đáp án gợi ý</summary>

Unsatisfiable: đọc log "Unable to simultaneously satisfy constraints", tìm constraint mâu thuẫn — hay gặp `NSAutoresizingMaskLayoutConstraint` do quên `translatesAutoresizingMaskIntoConstraints = false`, hoặc xung đột với `UIView-Encapsulated-Layout-Height` trong cell. Đặt `identifier` cho constraint để log dễ đọc, symbolic breakpoint `UIViewAlertForUnsatisfiableConstraints`. Ambiguous thì không có log: dùng View Debugger (dấu ! tím), `hasAmbiguousLayout`, `exerciseAmbiguityInLayout`. Sửa bằng cách bỏ constraint thừa, thêm constraint thiếu hoặc chỉnh priority.
</details>

**Q7 ⭐ Storyboard hay code? Vì sao?**
<details><summary>Đáp án gợi ý</summary>

Team mình ưu tiên code: không merge conflict XML, review được, tái sử dụng tốt, inject dependency qua init, lỗi được compiler bắt. Storyboard trực quan, nhanh cho prototype/app nhỏ nhưng file lớn hay conflict, segue/identifier là string dễ crash runtime, khó DI (từ iOS 13 có `instantiateViewController(identifier:creator:)`). XIB là trung gian tốt cho từng view/cell phức tạp. UI mới thường làm SwiftUI.
</details>

**Q8 UIStackView `distribution` và `alignment` khác nhau thế nào?**
<details><summary>Đáp án gợi ý</summary>

`distribution` quyết định cách chia kích thước dọc theo **trục chính** (`fill` dựa vào hugging/compression, `fillEqually`, `equalSpacing`...). `alignment` quyết định căn ở **trục phụ** (`fill`, `center`, `leading`, `firstBaseline`...). Ẩn arranged subview thì stack tự gỡ khỏi layout. Phải dùng `addArrangedSubview`; `removeArrangedSubview` không xoá view khỏi hierarchy.
</details>

**Q9 ⭐ Cell reuse hoạt động thế nào? Vì sao cần?**
<details><summary>Đáp án gợi ý</summary>

Tạo cell tốn kém nên table/collection chỉ tạo đủ số cell lấp màn hình (+ vài cái). Khi cell trượt khỏi màn hình, nó vào reuse queue theo reuse identifier; `dequeueReusableCell(withIdentifier:for:)` lấy cell cũ ra (gọi `prepareForReuse`) hoặc tạo mới nếu queue rỗng, rồi ta configure với dữ liệu mới. Hệ quả: cell mang trạng thái cũ, nên configure phải set đủ mọi thuộc tính và phải xử lý tác vụ async cẩn thận.
</details>

**Q10 ⭐ Ảnh hiện sai cell khi cuộn nhanh — vì sao và sửa thế nào? `prepareForReuse` nên làm gì?**
<details><summary>Đáp án gợi ý</summary>

Cell bắt đầu tải ảnh cho item A, bị tái sử dụng cho item B, ảnh A về sau và gán vào cell đang hiển thị B. Sửa: trong `prepareForReuse` huỷ task tải và reset ảnh về placeholder; lưu ID item cell đang đại diện và kiểm tra trước khi gán; tốt hơn là có image pipeline có cache và gom request. `prepareForReuse` chỉ nên huỷ tác vụ, reset trạng thái tạm, nhẹ nhàng; còn nội dung thì configure trong `cellForRowAt` phải set lại toàn bộ.
</details>

**Q11 Làm self-sizing cell thế nào?**
<details><summary>Đáp án gợi ý</summary>

`rowHeight = automaticDimension` và `estimatedRowHeight` hợp lý (collection view: dimension `.estimated`). Trong `contentView` tạo chuỗi constraint dọc liền từ top tới bottom, các view có chiều cao xác định (intrinsic hoặc constraint), label `numberOfLines = 0`. Gặp xung đột với encapsulated height thì hạ priority bottom xuống 999. Khi nội dung đổi chiều cao sau khi hiển thị, gọi `performBatchUpdates` hoặc reconfigure để tính lại.
</details>

**Q12 ⭐ Compositional layout là gì? Ưu điểm so với flow layout?**
<details><summary>Đáp án gợi ý</summary>

Layout khai báo theo cấu trúc item → group → section, kích thước bằng fractional/absolute/estimated. Mỗi section có layout riêng, hỗ trợ `orthogonalScrollingBehavior` (section cuộn ngang trong list dọc, không cần lồng collection view), header/footer/badge supplementary, decoration, list configuration thay table view, và thích ứng theo environment (size class, xoay). Flow layout chỉ xếp tuần tự, layout phức tạp phải subclass hoặc lồng collection view.
</details>

**Q13 ⭐ Diffable data source giải quyết vấn đề gì?**
<details><summary>Đáp án gợi ý</summary>

Cách cũ có hai nguồn sự thật (mảng của mình và trạng thái của collection view), gọi insert/delete lệch số lượng là crash `NSInternalInconsistencyException`; còn `reloadData` thì mất animation. Diffable dùng snapshot làm nguồn sự thật duy nhất: mình dựng snapshot trạng thái mong muốn, `apply`, framework tự tính diff và animate. Identifier phải Hashable và duy nhất.
</details>

**Q14 ⭐ Item identifier nên là model hay ID? `reconfigureItems` vs `reloadItems`?**
<details><summary>Đáp án gợi ý</summary>

Nên là ID ổn định. Nếu dùng cả struct, đổi một thuộc tính làm hash đổi nên diffable coi là xoá và chèn — animation sai, mất selection. Dùng ID thì nội dung đổi phải báo: `reconfigureItems` (iOS 15) gọi lại configure trên cell hiện có, rẻ và giữ trạng thái cell; `reloadItems` dequeue/tạo lại cell, dùng khi đổi loại cell hoặc thay đổi lớn.
</details>

**Q15 Prefetching hoạt động thế nào? Có đảm bảo được gọi không?**
<details><summary>Đáp án gợi ý</summary>

`prefetchDataSource` nhận index path sắp hiện theo hướng cuộn để bắt đầu tải dữ liệu/ảnh sớm, và `cancelPrefetching` khi hướng đổi. Nó chỉ là gợi ý, không phải lúc nào cũng được gọi (lần hiển thị đầu, cuộn quá nhanh, VoiceOver), nên `cellForItemAt` vẫn phải tự tải được — prefetch chỉ làm ấm cache. Khác với cell prefetching iOS 15 (chuẩn bị cell sớm).
</details>

**Q16 ⭐ Làm pagination thế nào cho đúng?**
<details><summary>Đáp án gợi ý</summary>

Trigger khi còn vài item tới cuối (prefetch hoặc `willDisplay`), có cờ `isLoading` để không gọi trùng, `hasMore` để dừng khi hết. Lỗi thì không tăng trang và hiện cell "Thử lại". Nối kết quả bằng snapshot append, khử ID trùng. Dữ liệu thay đổi liên tục thì dùng cursor thay cho số trang để tránh trùng/sót. State này để trên MainActor (hoặc trong ViewModel `@MainActor`) để không race.
</details>

**Q17 ⭐ List bị giật khi scroll, bạn làm gì?**
<details><summary>Đáp án gợi ý</summary>

Đầu tiên đo bằng Instruments (Animation Hitches, Time Profiler) để biết commit hay render hitch. Commit hitch: làm `cellForRow` nhẹ, decode và downsample ảnh ở background với cache, giảm độ sâu view/stack view, không đổi constraint mỗi lần configure, cache chiều cao/estimated sát, dùng reconfigure thay reloadData, prefetch. Render hitch: tránh offscreen rendering (shadowPath, tránh mask), giảm blending (view opaque), hạn chế blur.
</details>

**Q18 ⭐ Vì sao một ảnh JPEG 2MB lại tốn ~48MB RAM? Downsampling là gì?**
<details><summary>Đáp án gợi ý</summary>

Khi hiển thị, ảnh phải được decode thành bitmap; RAM tính theo số pixel × 4 byte (RGBA), không theo dung lượng file nén. Ảnh 4000×3000 là 48MB. Downsampling là dùng ImageIO (`CGImageSourceCreateThumbnailAtIndex` với `kCGImageSourceThumbnailMaxPixelSize` = kích thước hiển thị × scale, `ShouldCache = false` ở source) để decode thẳng ra ảnh nhỏ mà không bao giờ decode ảnh full, chạy ở background. Ô 100pt @3x chỉ còn ~270KB.
</details>

**Q19 `UIImage(data:)` đã decode ảnh chưa? Làm sao decode ở background?**
<details><summary>Đáp án gợi ý</summary>

Chưa — nó chỉ giữ dữ liệu nén; decode xảy ra lười khi ảnh được render lần đầu, trên main thread, gây hitch. Decode trước ở nền bằng `byPreparingForDisplay()`/`prepareForDisplay` (iOS 15), `byPreparingThumbnail(ofSize:)`, hoặc ImageIO với `kCGImageSourceShouldCacheImmediately`, hoặc vẽ vào context ở background (cách cũ), rồi mới gán cho image view trên main.
</details>

**Q20 Xử lý bàn phím che ô nhập liệu thế nào?**
<details><summary>Đáp án gợi ý</summary>

iOS 15+ dùng `view.keyboardLayoutGuide`: ghim bottom của input bar/scroll view vào `keyboardLayoutGuide.topAnchor`, hệ thống tự animate. Cách cổ điển: lắng nghe `keyboardWillChangeFrame`/`keyboardWillHide`, lấy frame cuối (toạ độ màn hình), convert sang toạ độ view, tính phần bị che, chỉnh `contentInset` và `scrollIndicatorInsets` của scroll view, rồi `scrollRectToVisible` ô đang focus. Ẩn bàn phím bằng `endEditing(true)` hoặc `keyboardDismissMode = .interactive`.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Làm xong Bài tập 1–7 không xem lời giải trước
- [ ] Bài tập lớn "GitHub Explore" chạy được, cuộn nhanh không sai ảnh, đã ghi số MB có/không downsampling
- [ ] Giải thích bằng lời hugging vs compression với ví dụ title + badge
- [ ] Viết lại từ trí nhớ: hàm `downsample` ImageIO và các option
- [ ] Trả lời được ≥ 16/20 câu phỏng vấn trước khi mở đáp án
- [ ] Ghi âm 2 phút: "Cell reuse và bug ảnh sai cell"
- [ ] Ghi âm 2 phút: "Diffable data source, item identifier, reconfigure"
- [ ] Ghi âm 2 phút: "List giật — mình làm gì" (≥ 6 kỹ thuật)
- [ ] Làm xong 6 bài DSA tuần 10 (Trees — BFS)
- [ ] Side project Milestone 10 xong, đã commit
- [ ] Viết 5 dòng tự nhận xét: tuần này phần nào còn yếu?

## 📖 Tài liệu đọc thêm

- [Apple — Auto Layout Guide](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/) (cũ nhưng chuẩn: *Anatomy of a Constraint*, *Debugging Auto Layout*)
- [Apple — Positioning content relative to the safe area](https://developer.apple.com/documentation/uikit/positioning-content-relative-to-the-safe-area) · [Positioning content within layout margins](https://developer.apple.com/documentation/uikit/positioning-content-within-layout-margins)
- [Apple — Implementing modern collection views](https://developer.apple.com/documentation/uikit/implementing-modern-collection-views) (sample code: compositional + diffable + list)
- [Apple — Updating collection views using diffable data sources](https://developer.apple.com/documentation/uikit/updating-collection-views-using-diffable-data-sources)
- [Apple — Prefetching collection view data](https://developer.apple.com/documentation/uikit/prefetching-collection-view-data)
- [Apple — Adjusting your layout with keyboard layout guide](https://developer.apple.com/documentation/uikit/adjusting-your-layout-with-keyboard-layout-guide)
- WWDC18 — *High Performance Auto Layout* · *Image and Graphics Best Practices* (downsampling — **bắt buộc xem**)
- WWDC19 — *Advances in Collection View Layout* · *Advances in UI Data Sources*
- WWDC20 — *Modern cell configuration* · *Lists in UICollectionView*
- WWDC21 — *Make blazing fast lists and collection views* (reconfigure, cell prefetching, `prepareForDisplay`)
- WWDC16 — *What's New in UICollectionView in iOS 10* (prefetching)
- [wtfautolayout.com](https://www.wtfautolayout.com) — dịch log constraint sang dạng dễ đọc
- Blog: Antoine van der Lee (SwiftLee) — Auto Layout & diffable; Kodeco — *UICollectionView Tutorial: Prefetching APIs*
