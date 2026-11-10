# Tuần 18 — Mobile system design · Kể chuyện dự án (STAR) · Mock interview (08/03 – 14/03/2027)

> **Giai đoạn 5 · Về đích** · Thời lượng: ~15 giờ
> Tuần này chuyển từ "biết kiến thức" sang **"trình bày kiến thức"**. Ở mức 2 năm, system design không đòi hỏi bạn thiết kế backend cho 100 triệu user — người ta muốn xem bạn **có cấu trúc khi suy nghĩ**, biết **trade-off**, và biết những thứ app thật cần (cache, offline, pagination, lỗi mạng). Phần behavioral thì quyết định **có muốn làm việc với bạn không**.
> Song song: **bắt đầu rải CV** (nếu tuần 17 chưa làm).

## 🎯 Mục tiêu tuần

Cuối tuần, không nhìn tài liệu, bạn phải:
- [ ] Trình bày một bài mobile system design trong **35–40 phút** theo khung 9 bước, có vẽ sơ đồ
- [ ] Tự thiết kế lại 3 bài mẫu: **news feed**, **chat offline**, **image loading library**
- [ ] Giải thích được trade-off: offset vs cursor pagination, polling vs WebSocket vs push, memory vs disk cache, SwiftData vs SQLite vs file
- [ ] Kể **3 câu chuyện STAR** (từ side project RepoScope + dự án công ty), mỗi câu ≤ 2 phút
- [ ] Trả lời trôi chảy 10 câu behavioral, đặc biệt "giới thiệu bản thân", "vì sao nghỉ", "dùng AI thế nào"
- [ ] Làm ≥ **2 buổi mock interview** (1 với người, 1 với AI), có ghi âm/ghi hình
- [ ] Nộp CV ≥ **10 vị trí** phù hợp
- [ ] DSA tuần 18 (xem [dsa-plan](../tracks/dsa-plan.md)) · Side project: Milestone 18 (xem [side-project](../tracks/side-project.md))

---

## 📅 Lịch từng ngày

| Ngày | Thời lượng | Nội dung | Đầu ra |
|---|---|---|---|
| **T2 08/03** | 2h | §1 Khung 9 bước + §2 các "khối kiến thức" (pagination, cache, offline…). Gửi CV đợt 1 (5 nơi) | Khung 9 bước chép tay 1 trang A4 |
| **T3 09/03** | 2h | §3.1 **News feed**: đọc → đóng file → tự vẽ lại trên giấy trong 40'. DSA: LRU Cache | Sơ đồ feed tự vẽ |
| **T4 10/03** | 2h | §3.2 **Chat offline** + §3.3 **Image loading library** (đọc, ghi ý chính). DSA: 1 bài mixed | 2 sơ đồ |
| **T5 11/03** | 2h | §4 **STAR** + §5 viết câu trả lời 10 câu behavioral ra giấy. §6 Cách nói về dự án công ty | `stories.md` nháp |
| **T6 12/03** | 1,5h | Mock #1 với AI (prompt §7.2): 1 system design + 3 behavioral. Ghi âm. Gửi CV đợt 2 (5 nơi) | File ghi âm + note tự chấm |
| **T7 13/03** | 3,5h | Side project: **Milestone 18** (polish, sơ đồ kiến trúc, 3 story). Mock #2 với **người thật** (bạn/đồng nghiệp cũ) | Repo hoàn chỉnh, feedback mock |
| **CN 14/03** | 2h | Xem lại ghi hình, sửa câu trả lời. Tự trả lời 12 câu 🎤. Tick checklist | Checklist ≥ 80% |

---

## 📚 Kiến thức cốt lõi

### §1. Mobile system design là gì và khung 9 bước

**Khác với backend system design:** bạn thiết kế **app client** — các module trong app, luồng dữ liệu, giao tiếp với server (API contract), lưu trữ cục bộ, hiệu năng, trải nghiệm khi mạng kém. Server được coi là "hộp đen" mà bạn được quyền **đề xuất API**.

Đề thường gặp: *"Thiết kế app giống Instagram feed / Messenger / Grab theo dõi tài xế / app đọc báo offline / thư viện tải ảnh như Kingfisher / tính năng upload video"*.

**Khung 9 bước (thời gian gợi ý cho buổi 45'):**

| # | Bước | Thời gian | Bạn nói/vẽ gì |
|---|---|---|---|
| 1 | **Làm rõ yêu cầu** | 5' | Functional (tính năng nào **trong phạm vi**), non-functional (offline? realtime? bao nhiêu user? thiết bị yếu?), **ngoài phạm vi** (login, thanh toán…) |
| 2 | **Thành phần high-level** | 5' | Vẽ hộp: UI → ViewModel → Repository → (API client, Local store, Cache) → Server |
| 3 | **Luồng dữ liệu** | 5' | Mở màn hình thì dữ liệu đi từ đâu tới đâu; "single source of truth" là gì |
| 4 | **Thiết kế API** | 5' | Endpoint, request/response JSON, REST vs GraphQL vs WebSocket |
| 5 | **Offline & caching** | 5' | Lưu gì, ở đâu (memory/disk/DB), khi nào hết hạn, đồng bộ thế nào |
| 6 | **Pagination** | 3' | Cursor hay offset, prefetch khi nào, xử lý trùng |
| 7 | **Image/media loading** | 3' | Cache 2 tầng, downsampling, huỷ khi cuộn qua |
| 8 | **Xử lý lỗi** | 3' | Retry + backoff, trạng thái UI, mất mạng giữa chừng, token hết hạn |
| 9 | **Mở rộng & trade-off** | 5' | Nếu 10× user? Nếu cần realtime? Theo dõi (analytics, crash), feature flag, test |

> **Quy tắc vàng:** **hỏi trước, vẽ sau**. Người phỏng vấn cố tình đưa đề mơ hồ để xem bạn có hỏi không. Một ứng viên 2 năm hỏi 5–6 câu làm rõ tốt thường được đánh giá cao hơn ứng viên vẽ luôn sơ đồ phức tạp.

**Câu hỏi làm rõ "mẫu" (dùng cho mọi đề):**
1. Tính năng chính nào cần trong phạm vi buổi này? (liệt kê 3–4, xin xác nhận)
2. Có cần dùng **offline** không? Mức nào — chỉ đọc, hay cả ghi?
3. Có cần **realtime** không? Độ trễ chấp nhận được?
4. Quy mô: số user, kích thước dữ liệu mỗi item (có ảnh/video không)?
5. Nền tảng: chỉ iOS? iOS tối thiểu bao nhiêu (ảnh hưởng chọn SwiftUI/SwiftData)?
6. Backend đã có sẵn hay mình được đề xuất API?

### §2. Các "khối kiến thức" dùng lại cho mọi đề

#### 2.1. Kiến trúc client tiêu chuẩn
```
┌──────────── UI Layer ────────────┐
│ View / ViewController            │  ← chỉ render state + gửi event
│ ViewModel (@MainActor)           │  ← state, logic trình bày
└───────────────┬──────────────────┘
                │ protocol
┌───────────────▼──────────────────┐
│ Repository (single source of truth)
│  ├─ RemoteDataSource (APIClient) │
│  ├─ LocalDataSource (DB)         │
│  └─ MemoryCache                  │
└───────────────┬──────────────────┘
        ┌───────┴────────┐
   URLSession        SwiftData / Core Data / SQLite / Files
```
- **Single source of truth (SSOT):** UI chỉ đọc từ **DB local**; network chỉ **ghi vào DB**. Lợi ích: offline tự nhiên, không lệch dữ liệu giữa các màn. Chi phí: phức tạp hơn, cần migration.
- Nếu không cần offline: Repository trả thẳng dữ liệu network + memory cache là đủ — **nói ra lựa chọn này** thay vì mặc định dùng DB.

#### 2.2. Pagination
| | Offset/page (`?page=3&limit=20`) | Cursor (`?after=abc123&limit=20`) |
|---|---|---|
| Ưu | Đơn giản, nhảy tới trang bất kỳ | Ổn định khi dữ liệu thêm/xoá liên tục |
| Nhược | Item mới chèn đầu → **trùng/mất item** khi sang trang | Không nhảy trang được |
| Hợp với | Kết quả tìm kiếm, danh sách ít đổi | **Feed, chat, notification** |

Client: tải trang tiếp khi còn ~5 item tới cuối (prefetch), cờ `isLoading` chống gọi trùng, **lọc trùng theo ID** (bài học M10 của RepoScope), `hasMore` từ server.

#### 2.3. Cache nhiều tầng
| Tầng | Công cụ iOS | Đặc điểm |
|---|---|---|
| Memory | `NSCache`, dictionary trong actor | Nhanh nhất, mất khi app bị kill, tự evict khi thiếu RAM (NSCache) |
| Disk (file) | `FileManager` trong `Caches/` | Hệ thống có thể xoá khi đầy bộ nhớ; hợp ảnh, JSON |
| HTTP | `URLCache` + header `Cache-Control`/`ETag` | Gần như miễn phí nếu server trả header đúng |
| Database | SwiftData / Core Data / SQLite (GRDB) | Truy vấn, quan hệ, SSOT cho offline |

Chiến lược: **cache-then-network** (hiện cache ngay, tải mới, cập nhật) cho feed; **network-first, fallback cache** cho dữ liệu cần tươi; **TTL** (hết hạn sau X phút).

#### 2.4. Giao tiếp realtime
| Cách | Khi nào | Chi phí |
|---|---|---|
| **Polling** (gọi định kỳ) | Cập nhật không gấp, đơn giản | Tốn pin/data, trễ |
| **Long polling / SSE** | Server đẩy 1 chiều | Ít phổ biến trên iOS |
| **WebSocket** (`URLSessionWebSocketTask`) | Chat, vị trí tài xế, typing indicator | Phải tự reconnect, heartbeat; **chỉ chạy khi app foreground** |
| **Push notification (APNs)** | App ở background/đã tắt | Không đảm bảo tới, giới hạn payload (4 KB), không dùng làm kênh dữ liệu chính |

#### 2.5. Xử lý lỗi & mạng
- Phân loại: **lỗi mạng** (retry được) · **4xx** (lỗi client, không retry, trừ 401 → refresh token, 429 → chờ) · **5xx** (retry có giới hạn) · **lỗi decode** (bug, log lên crash/analytics).
- **Exponential backoff + jitter**: chờ 1s, 2s, 4s… (+ ngẫu nhiên) tối đa 3–5 lần.
- `NWPathMonitor` để biết online/offline; khi có mạng lại thì **flush hàng đợi** việc chờ gửi.
- **Idempotency**: request gửi lại (retry) không được tạo 2 bản ghi → client tạo `clientId`/`Idempotency-Key` (UUID).
- Token hết hạn: một actor `AuthManager` đảm bảo **chỉ 1 lần refresh** dù 5 request cùng gặp 401.

#### 2.6. Những thứ "mức senior hơn" — nhắc tới ở bước 9 là ăn điểm
Analytics/logging · crash reporting · feature flag/remote config · A/B test · accessibility · localization · bảo mật (Keychain, certificate pinning, không log PII) · kiểm thử (unit, snapshot, UI) · modularization (SPM) khi team lớn · đo hiệu năng (MetricKit, Instruments).

---

### §3. Ba bài thiết kế mẫu

> **Cách học:** đọc 1 lần, **đóng file**, tự vẽ lại trên giấy trong 40 phút **có nói to**. Sau đó so sánh, ghi lại 3 chỗ bạn quên.

#### §3.1. Thiết kế news feed (kiểu Instagram/Facebook)

**Bước 1 — Làm rõ (giả định sau khi hỏi):**
- Trong phạm vi: xem feed (bài có ảnh + caption + số like), cuộn vô hạn, like, pull-to-refresh, xem được feed cũ khi offline.
- Ngoài phạm vi: đăng bài, comment, story, video, login.
- Non-functional: cuộn mượt 60 fps trên máy cũ (iPhone XR), mở app thấy nội dung **ngay** (< 1s, từ cache), tiết kiệm data.

**Bước 2 — Thành phần:**
```
FeedViewController (UICollectionView + diffable)
        │ observe state
FeedViewModel ── LikeService
        │
FeedRepository ─┬─ FeedAPI (APIClient)
                ├─ FeedStore (SwiftData/SQLite)   ← SSOT
                └─ ImageLoader (memory + disk cache)
```

**Bước 3 — Luồng dữ liệu:**
1. Mở app → ViewModel yêu cầu Repository → Repository **trả ngay** 20 post trong DB (nếu có) → UI hiển thị.
2. Song song, Repository gọi `GET /feed?limit=20` → ghi DB (upsert theo `id`) → DB phát thay đổi → UI cập nhật bằng diffable (animation mượt, không nhảy).
3. Cuộn gần cuối → `GET /feed?cursor=<nextCursor>` → append.
4. Pull-to-refresh → gọi trang đầu, thay phần đầu feed.

**Bước 4 — API:**
```http
GET /v1/feed?cursor=eyJpZCI6MTIzfQ&limit=20
200 OK
{
  "posts": [{
    "id": "p_981", "author": {"id": "u_1", "name": "An", "avatarUrl": "https://cdn/..."},
    "caption": "…", "createdAt": "2027-03-08T09:00:00Z",
    "media": [{"url": "https://cdn/p_981_1080.jpg", "width": 1080, "height": 1350,
               "thumbhash": "…"}],
    "likeCount": 120, "likedByMe": false
  }],
  "nextCursor": "eyJpZCI6MTAzfQ"      // null = hết
}

POST   /v1/posts/{id}/like          → 204
DELETE /v1/posts/{id}/like          → 204
```
- Trả `width/height` ảnh → client **đặt sẵn kích thước cell**, không bị nhảy layout khi ảnh tải xong.
- `thumbhash`/`blurhash` (chuỗi ngắn) → hiện ảnh mờ ngay lập tức.
- Nhiều kích thước ảnh trên CDN (`_320`, `_640`, `_1080`) → client chọn theo `bounds.width × scale`.

**Bước 5 — Offline & cache:**
- DB lưu ~200 post gần nhất (giới hạn để không phình), xoá cũ theo `createdAt`.
- Ảnh: memory (`NSCache`) + disk (`Caches/images`, LRU, giới hạn ~200 MB).
- Offline: hiển thị DB + banner "Bạn đang offline".

**Bước 6 — Pagination:** **cursor** (vì feed thay đổi liên tục, offset sẽ trùng/mất bài). Prefetch khi còn 5 item. Lọc trùng ID.

**Bước 7 — Ảnh:** `UICollectionViewDataSourcePrefetching` bắt đầu tải sớm; huỷ khi `cancelPrefetching`/`prepareForReuse`; **downsample** về kích thước cell ở background; decode trước khi gán lên main thread.

**Bước 8 — Lỗi:**
- Trang đầu lỗi + có cache → giữ cache, toast "Không tải được bài mới".
- Trang đầu lỗi + không cache → màn lỗi + nút thử lại.
- Trang sau lỗi → footer "Thử lại" (không xoá dữ liệu đang có).
- **Like optimistic**: đổi UI ngay (`likedByMe = true`, `likeCount += 1`), gọi API; lỗi → **rollback** + toast. Bấm like/unlike liên tục → debounce hoặc chỉ gửi trạng thái cuối.

**Bước 9 — Trade-off & mở rộng:**
- DB làm SSOT: + offline, + mở app nhanh; − phức tạp (migration, đồng bộ). Nếu yêu cầu không cần offline → chỉ memory cache.
- Bài mới xuất hiện khi đang xem: **không chèn tự động** (làm nhảy nội dung) → hiện nút "Có 5 bài mới ↑".
- Feed có video: autoplay chỉ cell hiện > 50%, 1 `AVPlayer` dùng lại, tắt tiếng mặc định.
- Đo: thời gian hiển thị bài đầu tiên (time-to-first-content), hitch rate khi cuộn (MetricKit/Instruments).

#### §3.2. Thiết kế chat app có offline (kiểu Messenger/Zalo, 1–1 + nhóm nhỏ)

**Bước 1 — Làm rõ:**
- Trong phạm vi: danh sách hội thoại, xem tin nhắn, gửi tin text (+ ảnh nếu kịp), trạng thái gửi (đang gửi/đã gửi/lỗi), **soạn và "gửi" khi offline**, nhận tin realtime khi mở app, push khi app ở background.
- Ngoài phạm vi: gọi thoại/video, mã hoá đầu-cuối (chỉ nhắc ở bước 9), reaction.
- Non-functional: tin nhắn **không bao giờ mất** và **không trùng**, thứ tự đúng, mở hội thoại tức thì.

**Bước 2 — Thành phần:**
```
ConversationList VC/View      ChatView (list đảo ngược)
          │                           │
ConversationListViewModel       ChatViewModel
          └──────────┬────────────────┘
               MessageRepository
     ┌──────────┬────┴───────┬──────────────┐
 ChatAPI(REST) SocketClient  MessageStore    OutboxSender
 (lịch sử)     (WebSocket)   (DB = SSOT)     (hàng đợi gửi, retry)
                                    ▲
                        PushHandler (APNs) → tải tin mới khi được đánh thức
```

**Bước 3 — Luồng dữ liệu:**
- **Nhận**: Socket nhận event `message.new` → ghi DB → UI (đang quan sát DB) tự cập nhật.
- **Gửi**:
  1. User bấm gửi → tạo message với `clientId = UUID()`, `status = .pending`, `createdAt` local → **ghi DB ngay** → UI hiện tin (mờ, có đồng hồ).
  2. `OutboxSender` lấy tin `pending` → gửi qua socket (hoặc `POST /messages`).
  3. Server ack `{clientId, serverId, serverTimestamp}` → cập nhật DB `status = .sent`, đổi sang thứ tự theo `serverTimestamp`.
  4. Lỗi/offline → giữ `pending`, retry với backoff khi có mạng; quá N lần → `.failed`, cho bấm "Gửi lại".
- **Mở lại app / reconnect**: `GET /conversations/{id}/messages?after=<lastServerId>` để lấp khoảng trống (sync), vì socket có thể đã rớt tin khi ngắt kết nối.

**Bước 4 — API:**
```http
GET  /v1/conversations?cursor=…                         → danh sách + lastMessage + unreadCount
GET  /v1/conversations/{id}/messages?before=<msgId>&limit=50   → tải lịch sử cũ hơn (cuộn lên)
GET  /v1/conversations/{id}/messages?after=<msgId>              → đồng bộ tin bị lỡ
POST /v1/conversations/{id}/messages   {"clientId":"…","text":"…"}  → idempotent theo clientId

WebSocket wss://chat.example.com/v1/socket
  ← {"type":"message.new","conversationId":"c1","message":{…}}
  → {"type":"message.send","clientId":"…","conversationId":"c1","text":"…"}
  ← {"type":"message.ack","clientId":"…","id":"m_77","sentAt":"…"}
  ↔ {"type":"ping"} / {"type":"pong"}       // heartbeat 30s
```

**Bước 5 — Offline & lưu trữ:**
- DB bảng `Conversation(id, title, lastMessageAt, unreadCount)` và `Message(id?, clientId, conversationId, senderId, text, status, createdAt, serverTimestamp?)`, index theo `(conversationId, serverTimestamp)`.
- Chọn DB: SwiftData/Core Data đủ cho mức này; app chat lớn thường dùng SQLite (GRDB) để kiểm soát truy vấn & hiệu năng — **nói ra trade-off**.
- Outbox **chính là** các message `status = pending` trong DB → app bị kill vẫn gửi tiếp khi mở lại.

**Bước 6 — Pagination:** cuộn lên tải tin cũ bằng cursor `before=<oldestId>`; giữ vị trí cuộn khi chèn lên đầu (tính lại `contentOffset`, hoặc dùng list đảo ngược).

**Bước 7 — Ảnh:** upload ảnh: nén/resize trước → tạo message `pending` với ảnh local (hiện ngay) → upload (có thể `URLSession` background để không chết khi app vào nền) → gửi message chứa URL. Hiển thị bằng ImageLoader như feed.

**Bước 8 — Lỗi & trường hợp khó:**
- **Trùng tin**: retry sau khi server đã nhận → server dedupe theo `clientId`; client upsert theo `clientId`.
- **Thứ tự**: hiển thị theo `serverTimestamp`; tin pending luôn ở cuối.
- **Socket rớt**: reconnect với backoff; khi nối lại → sync `after=lastId`.
- App ở background: socket bị hệ thống đóng → dựa vào **push**; push có thể là *silent/background push* để tải tin trước, nhưng hệ thống **không đảm bảo** giao background push → khi mở app luôn sync.

**Bước 9 — Mở rộng:**
- Trạng thái "đã xem", "đang gõ" (event socket nhẹ, không lưu DB).
- E2E encryption (Signal protocol) — chỉ nêu, không đi sâu.
- Notification Service Extension để sửa nội dung push (giải mã, thêm ảnh).
- Nhóm lớn: không fan-out mọi event typing; phân trang member.

#### §3.3. Thiết kế thư viện image loading (kiểu Kingfisher/SDWebImage/Nuke)

Đề này rất hợp với iOS 2 năm vì **bạn đã tự làm trong RepoScope** (M5–M10). Kể kèm code của mình.

**Bước 1 — Làm rõ:**
- API dùng thế nào? `imageView.setImage(url:)` cho UIKit + `RemoteImage(url:)` cho SwiftUI.
- Cần: memory cache, disk cache, gộp request trùng, huỷ, downsampling, placeholder. Có cần GIF/WebP, transform (bo góc), progressive JPEG? → ngoài phạm vi, nêu ở bước 9.

**Bước 2 — Thành phần (pipeline):**
```
setImage(url, size)
   │
   ▼
ImagePipeline (actor)
   1. Memory cache (NSCache<Key, UIImage>)  ── hit → trả ngay (main thread, đồng bộ nếu được)
   2. In-flight table [Key: Task]           ── đang tải → chờ chung
   3. Disk cache (Caches/, key = sha256(url)) ── hit → decode + downsample (background)
   4. Downloader (URLSession)               ── tải → ghi disk → decode + downsample
   5. Lưu memory cache → trả về
   │
   ▼
UIImageView extension: lưu "token" hiện tại; khi ảnh về, kiểm tra token còn khớp (cell reuse) rồi mới gán
```

**Bước 3 — Luồng & threading:** mọi bước nặng (I/O disk, decode, downsample) ở **background**; chỉ gán `image` ở **main**. Memory cache lookup đủ nhanh để làm đồng bộ → tránh "chớp" placeholder khi ảnh đã có.

**Bước 4 — "API" của thư viện:**
```swift
public protocol ImageLoading: Sendable {
    func image(for request: ImageRequest) async throws -> UIImage
    func prefetch(_ requests: [ImageRequest])
    func cancelPrefetch(_ requests: [ImageRequest])
}
public struct ImageRequest: Hashable, Sendable {
    public let url: URL
    public let targetSize: CGSize      // điểm (point)
    public let scale: CGFloat
    // cacheKey = url + targetSize → ảnh cùng URL khác cỡ là 2 entry
}
```

**Bước 5 — Cache:**
- **Memory**: `NSCache` với `totalCostLimit` (cost = `bytesPerRow × height` của ảnh **đã decode**). Ảnh 1080×1350 decode ra ≈ 1080 × 1350 × 4 byte ≈ **5,8 MB** dù file JPEG chỉ 200 KB → đây là lý do phải downsample.
- **Disk**: lưu **data gốc** (đã nén), giới hạn dung lượng (vd 300 MB) + thời hạn (7 ngày), dọn theo LRU (`contentAccessDate`) khi app vào background.
- Tôn trọng HTTP cache header nếu có (hoặc tắt `URLCache` để tránh lưu 2 lần).

**Bước 6 — Danh sách dài:** prefetch API cho collection view; **ưu tiên** request đang hiển thị cao hơn prefetch (`URLSessionTask.priority`); huỷ khi cell biến mất.

**Bước 7 — Downsampling:** ImageIO `CGImageSourceCreateThumbnailAtIndex` với `kCGImageSourceThumbnailMaxPixelSize` (code ở [side-project §4.6](../tracks/side-project.md)) — **không** dùng `UIGraphicsImageRenderer` vẽ lại ảnh to (vẫn phải decode full).

**Bước 8 — Lỗi:** URL 404 → lưu vào "negative cache" ngắn hạn để không thử lại liên tục; lỗi mạng → retry 1 lần; memory warning → `NSCache` tự dọn (có thể chủ động `removeAllObjects` khi nhận `didReceiveMemoryWarningNotification`).

**Bước 9 — Mở rộng & trade-off:**
- Transform (bo góc, blur) → cache **kết quả sau transform** với key gồm tên transform.
- Định dạng: HEIC/WebP (iOS 14+ hỗ trợ WebP qua ImageIO), GIF động → bộ decode riêng.
- Actor vs lock: actor dễ đúng hơn nhưng có chi phí hop; thư viện hiệu năng cao thường dùng lock (`OSAllocatedUnfairLock`/`Mutex`) cho memory cache.
- Đo: hit rate memory/disk, thời gian tải trung bình, bộ nhớ đỉnh.

---

### §4. Phương pháp STAR cho câu hỏi behavioral

**STAR** = **S**ituation (bối cảnh) · **T**ask (nhiệm vụ của **bạn**) · **A**ction (bạn **đã làm** gì — phần dài nhất) · **R**esult (kết quả, **có số liệu** nếu được, + bài học).

| Phần | Tỷ lệ | Mẹo |
|---|---|---|
| S | 15% | 1–2 câu: dự án gì, team mấy người, vấn đề gì |
| T | 10% | Vai trò **của bạn**, không phải của team |
| A | 55% | Dùng **"em"**, không phải "bọn em". Các bước cụ thể, công cụ cụ thể (Instruments, Memory Graph…), quyết định + lý do |
| R | 20% | Số liệu (crash giảm từ X → Y, thời gian tải giảm Z%), phản hồi từ team/user, **bài học** |

**Thời lượng:** 1,5–2 phút. Dài hơn → interviewer mất tập trung; họ sẽ hỏi thêm nếu muốn.

**Kho chuyện (story bank):** chuẩn bị 5–6 câu chuyện, mỗi câu dùng được cho nhiều câu hỏi:
| Story | Dùng cho câu hỏi |
|---|---|
| Bug khó nhất (crash / leak / race condition) | hardest bug, áp lực, học được gì |
| Tối ưu hiệu năng (list, ảnh, khởi động app) | thành tựu tự hào, kỹ thuật sâu |
| Bất đồng với đồng nghiệp/PM về giải pháp | conflict, teamwork, giao tiếp |
| Deadline gấp / thay đổi yêu cầu phút chót | áp lực, ưu tiên |
| Tự học công nghệ mới (Swift Concurrency, SwiftUI) | học hỏi, chủ động |
| Sai lầm của bản thân (bug lên production) | failure, trách nhiệm |

> **Nếu dự án công ty thiếu chuyện hay:** dùng **RepoScope**. Câu chuyện "ID trùng giữa 2 trang làm diffable crash" hay "ảnh avatar chiếm X MB → downsample còn Y MB" là chuyện thật, có code, có số liệu — mạnh hơn chuyện công ty kể mơ hồ.

**Ví dụ STAR hoàn chỉnh (hardest bug):**
> **S:** Ở công ty cũ, app thương mại điện tử của bọn em có crash rate tăng lên ~1,2% sau một bản release, chủ yếu ở màn danh sách sản phẩm.
> **T:** Em được giao tìm nguyên nhân và sửa trong sprint đó.
> **A:** Em đọc stack trace trên Firebase Crashlytics, thấy crash `NSInternalInconsistencyException` trong `performBatchUpdates`. Em tái hiện bằng cách bật Network Link Conditioner cho mạng chậm và cuộn nhanh lúc đang pull-to-refresh: hai request trả về lệch thứ tự và cùng cập nhật data source. Em chuyển sang diffable data source, để mọi cập nhật đi qua một chỗ trên main actor, huỷ request cũ khi có request mới, và viết unit test tái hiện thứ tự trả về ngược.
> **R:** Bản sau crash rate màn đó về gần 0, tổng crash-free users lên ~99,8%. Bài học của em là với bug khó tái hiện, việc đầu tiên là tìm cách **tái hiện chắc chắn** trước khi sửa.

*(Đây là ví dụ cấu trúc — **thay bằng chuyện và số liệu thật của bạn**, không bịa số. Không nhớ số chính xác thì nói "khoảng", "theo dashboard lúc đó".)*

---

### §5. 10 câu behavioral hay gặp + câu trả lời mẫu (dev iOS ~2 năm, Việt Nam)

> Mẫu dưới đây để tham khảo **cấu trúc**. **Viết lại bằng câu chuyện của bạn.** Câu trả lời học thuộc nghe rất "giả" — hãy thuộc **ý**, không thuộc **chữ**.

**B1 ⭐ "Giới thiệu về bản thân" (Tell me about yourself)**
<details><summary>Câu trả lời mẫu (~90 giây)</summary>

Công thức: **Hiện tại → Quá khứ (2 điểm nhấn) → Tương lai (vì sao vị trí này)**.

> "Em là Tuấn, lập trình viên iOS gần 2 năm kinh nghiệm. Hiện em đang làm ở công ty X, phát triển app [lĩnh vực, ví dụ: ví điện tử / thương mại điện tử] với khoảng [N] người dùng, trong team 4 dev iOS.
> Công việc chính của em là phát triển tính năng mới bằng Swift, UIKit và gần đây là SwiftUI, tích hợp API, và xử lý bug/crash. Hai việc em thấy mình làm tốt nhất là [ví dụ: tối ưu màn danh sách bị giật — giảm thời gian load] và [ví dụ: chuyển phần networking sang async/await].
> Ngoài giờ, em tự xây app RepoScope — app khám phá GitHub, viết bằng Swift 6, có cache ảnh tự viết, offline, unit test và CI — để củng cố nền tảng mà công việc hằng ngày chưa đụng tới nhiều.
> Em đang tìm môi trường [product / có quy trình review code / làm sản phẩm lớn hơn] để phát triển lên mid-level, và vị trí bên mình phù hợp vì [lý do cụ thể từ JD]."

Tránh: kể lại CV từ thời đại học, kể chuyện cá nhân không liên quan, nói quá 2 phút.
</details>

**B2 ⭐ "Bug khó nhất bạn từng gặp?"**
<details><summary>Câu trả lời mẫu</summary>

Dùng ví dụ STAR ở §4, hoặc chuyện từ RepoScope:
> **S:** Trong app cá nhân RepoScope, màn search dùng diffable data source thỉnh thoảng crash khi cuộn nhanh xuống trang 4–5.
> **T:** Em cần tìm ra nguyên nhân vì crash không ổn định.
> **A:** Log crash là "Supplied item identifiers are not unique". Em log ID từng trang và phát hiện GitHub API trả **cùng một repo ở 2 trang liên tiếp** do thứ hạng thay đổi giữa 2 lần gọi. Em thêm bước lọc trùng theo ID trước khi `appendItems`, viết unit test với 2 trang giả lập có item trùng, và ghi chú vào README.
> **R:** Hết crash; em hiểu rõ hơn vì sao offset pagination không ổn định với dữ liệu thay đổi, và đó là lý do cursor pagination hay được dùng cho feed.

Điểm cộng: nêu **công cụ** (log, Crashlytics, Memory Graph, TSan, Instruments), **cách tái hiện**, **test chống tái phát**.
</details>

**B3 ⭐ "Kể về một lần bất đồng với đồng nghiệp / PM"**
<details><summary>Câu trả lời mẫu</summary>

> **S:** PM muốn thêm tính năng lọc nâng cao vào bản release 1 tuần nữa; em ước lượng cần ~2 tuần để làm đúng (cần đổi API và cache).
> **T:** Em cần thống nhất phạm vi mà không làm chậm release hay làm ẩu.
> **A:** Em không từ chối ngay mà chia tính năng thành 3 phần, ước lượng từng phần và chỉ ra phần nào phụ thuộc backend. Em đề xuất bản đầu chỉ lọc theo 2 tiêu chí phổ biến nhất (dựa trên số liệu analytics PM có), phần còn lại làm ở sprint sau. Em cũng nói rõ rủi ro nếu làm hết trong 1 tuần: không có thời gian test, dễ lỗi trên production.
> **R:** PM đồng ý, bản đầu ra đúng hạn, ít bug; phần còn lại ra 2 tuần sau. Em học được là **đưa ra lựa chọn kèm dữ liệu** hiệu quả hơn nói "không kịp".

Nguyên tắc: không đổ lỗi, không kể chuyện mình "thắng"; nhấn vào **lắng nghe, dữ liệu, mục tiêu chung**.
</details>

**B4 ⭐ "Vì sao bạn muốn nghỉ công ty hiện tại?"**
<details><summary>Câu trả lời mẫu</summary>

Nguyên tắc: **hướng tới cái mới**, không chê công ty cũ.
> "Em học được nhiều ở công ty hiện tại, đặc biệt là [quy trình release / làm việc với khách hàng]. Sau gần 2 năm, em muốn được làm ở môi trường [có sản phẩm riêng / có code review chặt hơn / quy mô người dùng lớn hơn / dùng Swift Concurrency, SwiftUI nhiều hơn] để phát triển chiều sâu kỹ thuật. Dự án hiện tại đang ở giai đoạn bảo trì nên cơ hội đó hạn chế."

Nếu lý do thật là lương: có thể nói nhẹ nhàng ở cuối — *"và em cũng mong mức đãi ngộ tương xứng với kỹ năng hiện tại"* — nhưng **không** để lương là lý do đầu tiên.
Tránh: "sếp tệ", "công ty làm ăn kém", "đồng nghiệp lười".
</details>

**B5 ⭐ "Bạn dùng AI (ChatGPT, Copilot, Claude…) trong công việc thế nào?"**
<details><summary>Câu trả lời mẫu — trung thực và tích cực</summary>

Câu này **ngày càng hay gặp** từ 2025. Interviewer muốn biết: bạn có hiểu code mình commit không, và có dùng công cụ hiệu quả không. **Đừng phủ nhận** (không ai tin), cũng **đừng** nói "AI viết hết".
> "Em có dùng AI hằng ngày, chủ yếu để: đọc hiểu nhanh code hoặc API lạ, sinh boilerplate (ví dụ model Codable từ JSON), gợi ý test case, và giải thích lỗi compiler. Nhưng giai đoạn đầu em đã **phụ thuộc vào nó quá nhiều** — em nhận ra mình review code AI sinh ra mà không đủ hiểu để phát hiện chỗ sai, ví dụ một closure gây retain cycle.
> Vì vậy mấy tháng gần đây em tự đặt quy tắc: phần cốt lõi tự viết, AI chỉ dùng để hỏi 'vì sao' sau khi đã tự thử, và em **không merge dòng nào mà em không giải thích được**. Side project RepoScope em viết hoàn toàn không dùng AI sinh code để luyện lại nền tảng. Kết quả là giờ em debug nhanh hơn và dùng AI hiệu quả hơn, vì em biết hỏi đúng câu và nhận ra khi nó sai."

Chuyện này **là thật** với bạn (19 tuần vừa rồi) → kể tự tin. Đây là điểm cộng: tự nhận thức + hành động cụ thể.
</details>

**B6 "Thành tựu bạn tự hào nhất?"**
<details><summary>Câu trả lời mẫu</summary>

Chọn chuyện có **số liệu** và **vai trò rõ của bạn**:
> "Em tự hào nhất là lần tối ưu màn [danh sách sản phẩm]. Màn đó cuộn giật trên iPhone đời cũ. Em dùng Time Profiler thấy decode ảnh full-size trên main thread và tính chiều cao cell bằng Auto Layout phức tạp. Em downsample ảnh ở background, cache chiều cao, và giảm số view lồng nhau. Hitch giảm rõ rệt [số liệu nếu có], được team QA và PM ghi nhận. Quan trọng hơn, em viết lại thành tài liệu ngắn để cả team áp dụng cho các màn khác."
</details>

**B7 "Kể về một lần bạn mắc lỗi / thất bại"**
<details><summary>Câu trả lời mẫu</summary>

Cấu trúc: lỗi thật (không phải "lỗi giả" kiểu "em quá cầu toàn") → bạn **chịu trách nhiệm** → sửa → **thay đổi quy trình** để không lặp lại.
> "Có lần em merge một thay đổi model Codable làm một field thành non-optional, nhưng API thỉnh thoảng trả null cho user cũ → decode lỗi cả danh sách, một phần user thấy màn trống. Em phát hiện qua log lỗi decode tăng sau release. Em hotfix chuyển field về optional với giá trị mặc định, rồi viết thêm test decode với JSON thiếu field. Sau đó em đề xuất team lưu sẵn vài JSON response thật làm fixture để test decode trước mỗi release. Từ đó em luôn hỏi backend về khả năng null của từng field."
</details>

**B8 "Bạn làm gì khi gặp vấn đề không biết giải quyết?" / "Bạn tự học thế nào?"**
<details><summary>Câu trả lời mẫu</summary>

> "Em thử tự tìm hiểu trong khoảng thời gian giới hạn (thường 30–60 phút): đọc tài liệu Apple, tìm issue tương tự, viết một đoạn code nhỏ tách riêng để tái hiện. Nếu vẫn bế tắc, em hỏi đồng nghiệp kèm những gì em đã thử để không tốn thời gian của họ. Để học công nghệ mới như Swift Concurrency, em đọc Swift book + xem WWDC, rồi áp dụng ngay vào side project — ví dụ em chuyển image loader từ GCD sang actor và tự gặp vấn đề actor reentrancy, nhờ vậy hiểu sâu hơn đọc lý thuyết."
</details>

**B9 "Bạn làm việc thế nào dưới áp lực deadline?"**
<details><summary>Câu trả lời mẫu</summary>

> "Em chia việc nhỏ, ước lượng lại và báo sớm nếu có rủi ro trễ — báo sớm 3 ngày tốt hơn báo trễ 1 giờ trước deadline. Em ưu tiên phần ảnh hưởng người dùng nhất, phần 'đẹp hơn' để sau. Ví dụ [chuyện cụ thể]: bọn em có 4 ngày để tích hợp cổng thanh toán mới; em đề xuất làm luồng chính + xử lý lỗi trước, phần UI tuỳ biến làm bản sau. Em không bỏ test cho luồng thanh toán vì đó là chỗ không được phép lỗi."
</details>

**B10 "Mục tiêu 2–3 năm tới?" / "Bạn có câu hỏi gì cho chúng tôi?"**
<details><summary>Câu trả lời mẫu</summary>

> "Trong 2–3 năm tới em muốn trở thành iOS developer mid-to-senior: tự thiết kế được một tính năng từ đầu đến cuối (kiến trúc, API contract với backend, test, release), và có thể review code, hướng dẫn bạn mới. Em cũng muốn hiểu sâu hơn về hiệu năng và kiến trúc app lớn (modularization). Em thấy vị trí này phù hợp vì team bên mình [điều cụ thể bạn tìm hiểu được]."

Phần "câu hỏi cho nhà tuyển dụng": xem [week19](week19-final-interview.md) §5.
</details>

---

### §6. Nói về dự án công ty (khi có NDA / code không public)

1. **Được nói:** lĩnh vực, quy mô (ước lượng: "hàng trăm nghìn user", "team 4 iOS"), công nghệ, vai trò của bạn, vấn đề kỹ thuật và cách giải quyết **ở mức khái niệm**.
2. **Không được nói:** tên khách hàng nếu có NDA, số liệu kinh doanh nội bộ, code, chi tiết bảo mật. Nói: *"Đây là dự án cho một khách hàng ngân hàng ở Nhật, em không nói tên được, nhưng em có thể kể về kiến trúc…"* — interviewer **đánh giá cao** việc bạn tôn trọng NDA.
3. **Chuẩn bị "thẻ dự án"** cho mỗi dự án (1 trang):
   - Sản phẩm làm gì, cho ai, quy mô
   - Kiến trúc (vẽ được trong 1 phút): MVC/MVVM? UIKit/SwiftUI? Networking? Lưu trữ? DI?
   - Phần **bạn** làm (2–3 tính năng cụ thể)
   - 1 vấn đề kỹ thuật + cách giải quyết (STAR)
   - 1 điều bạn **sẽ làm khác** nếu làm lại (thể hiện bạn trưởng thành)
4. **Kiểm tra trước:** mọi công nghệ ghi trong CV, bạn phải trả lời được 3 câu "đào sâu" về nó. Ghi "RxSwift" mà không giải thích được `flatMap` vs `flatMapLatest` → mất điểm nặng hơn không ghi.
5. **Outsource**: đừng ngại. Nhấn vào: làm việc với nhiều codebase, quy trình với khách hàng (estimate, demo, change request), giao tiếp tiếng Anh/Nhật nếu có.

---

### §7. Kế hoạch mock interview

#### 7.1. Lịch mock (tuần 18–19)
| # | Ngày | Với ai | Nội dung | Thời lượng |
|---|---|---|---|---|
| 1 | T6 12/03 | AI (prompt 7.2a) | 1 system design + 3 behavioral | 60' |
| 2 | T7 13/03 | Bạn/đồng nghiệp cũ làm iOS | Kỹ thuật iOS + 1 bài DSA | 60' |
| 3 | T3 16/03 | AI (prompt 7.2b) | Hỏi xoáy kỹ thuật toàn bộ 19 tuần | 45' |
| 4 | T5 18/03 | Người thật (mentor / cộng đồng) | Full loop: intro, kỹ thuật, design, behavioral | 75' |

Tìm người mock: đồng nghiệp cũ, bạn học, group Facebook **iOS Developers Vietnam**, cộng đồng Viblo/Discord, mentor trên LinkedIn (nhắn lịch sự, đề nghị mock chéo — bạn cũng mock lại cho họ).

#### 7.2. Prompt cho AI đóng vai interviewer
> Dùng **voice mode** nếu có — nói thành tiếng giống phỏng vấn thật hơn gõ phím. Không cho AI xem đáp án trong file này trước.

**a) System design:**
```text
Bạn là senior iOS engineer ở một công ty product tại Việt Nam, đang phỏng vấn tôi cho vị trí
iOS Developer (2+ năm kinh nghiệm). Hãy cho tôi một đề mobile system design (ví dụ: feed, chat,
đặt xe, đọc báo offline) nhưng ĐỪNG nói đề chi tiết — để tôi tự hỏi làm rõ.
Quy tắc:
- Mỗi lượt chỉ hỏi/trả lời 1 ý, như phỏng vấn thật. Không giảng bài, không gợi ý đáp án.
- Khi tôi bỏ sót một phần quan trọng (offline, pagination, lỗi mạng, ảnh...), hỏi xoáy vào đó.
- Đào sâu trade-off: "tại sao chọn X mà không phải Y?".
- Sau 40 phút (hoặc khi tôi nói "kết thúc"), chấm điểm 1–5 cho: làm rõ yêu cầu, cấu trúc,
  chiều sâu kỹ thuật iOS, trade-off, giao tiếp. Nêu 3 điểm cần cải thiện cụ thể.
Trả lời bằng tiếng Việt.
```

**b) Hỏi xoáy kỹ thuật:**
```text
Đóng vai interviewer iOS khó tính. Hỏi tôi lần lượt 15 câu, trộn các chủ đề: Swift (value/reference,
optional, generics, protocol), ARC & retain cycle, GCD & Swift Concurrency (actor, Sendable),
UIKit lifecycle, Auto Layout, UICollectionView, SwiftUI state, kiến trúc (MVVM, DI, Coordinator),
networking, persistence, testing, performance.
Quy tắc: hỏi 1 câu mỗi lượt; sau câu trả lời của tôi, hỏi tiếp 1 câu đào sâu ("vì sao?",
"nếu ... thì sao?") trước khi sang câu mới. Không nhận xét cho tới cuối.
Cuối buổi: liệt kê câu tôi trả lời sai/thiếu, đáp án đúng ngắn gọn, và mức độ sẵn sàng (1–10).
```

**c) Behavioral:**
```text
Bạn là engineering manager phỏng vấn vòng văn hoá/hành vi cho vị trí iOS 2 năm kinh nghiệm.
Hỏi tôi 6 câu behavioral (giới thiệu bản thân, bug khó nhất, bất đồng, lý do nghỉ việc,
cách dùng AI trong công việc, thất bại). Với mỗi câu trả lời, hỏi thêm 1 câu follow-up
để kiểm tra tôi có nói thật và hiểu sâu không. Cuối buổi chấm theo STAR: S/T/A/R có rõ không,
có số liệu không, có dùng "tôi" thay vì "chúng tôi" không, độ dài có hợp lý không.
```

**d) Chấm câu trả lời đã viết sẵn:**
```text
Đây là câu trả lời tôi đã tự viết cho câu "[câu hỏi]": [dán câu trả lời].
Đừng viết lại hộ tôi. Chỉ ra: chỗ nào mơ hồ, chỗ nào nghe như học thuộc, thiếu số liệu ở đâu,
interviewer có thể hỏi xoáy câu gì tiếp theo.
```

#### 7.3. Ghi âm / ghi hình chính mình
- Dùng QuickTime (Mac) hoặc Zoom record. Bật camera — phỏng vấn online là phổ biến.
- Xem lại và đếm: số lần "ờ", "kiểu là", "thì là"; lần im lặng > 10 giây không nói gì; câu trả lời > 3 phút.
- Kiểm tra: ánh sáng chiếu vào mặt, mắt nhìn camera khi nói ý chính, tiếng không vọng, nền gọn.
- Bảng tự chấm sau mỗi mock:

| Tiêu chí | 1–5 | Ghi chú |
|---|---|---|
| Hỏi làm rõ trước khi trả lời | | |
| Cấu trúc rõ (khung 9 bước / STAR) | | |
| Đúng kỹ thuật | | |
| Nêu trade-off | | |
| Ngắn gọn, không lan man | | |
| Tự tin, giọng rõ | | |

---

### §8. Bắt đầu nộp CV

**Ở đâu:**
| Kênh | Ghi chú |
|---|---|
| **Referral (người quen giới thiệu)** | Tỷ lệ được gọi cao nhất. Nhắn đồng nghiệp cũ, bạn học đang ở công ty bạn thích: *"Bên anh/chị có tuyển iOS không, em gửi CV nhờ refer được không?"* |
| **ITviec** | Nhiều job IT nhất, lọc theo "iOS", xem review công ty |
| **TopDev** | Job IT + báo cáo thị trường lương hằng năm |
| **LinkedIn** | Bật "Open to work" (chỉ recruiters thấy nếu sợ công ty hiện tại biết), cập nhật headline "iOS Developer · Swift · SwiftUI", kết nối với recruiter |
| **TopCV, VietnamWorks** | Thêm nguồn, nhiều công ty non-IT cần iOS |
| Trang tuyển dụng trực tiếp | Công ty product bạn thích (fintech, e-commerce, ride-hailing…) |
| Group Facebook / Discord | "iOS Developers Vietnam", nhóm tuyển dụng IT |

**Chiến lược:**
- Chia 3 nhóm: **luyện tay** (công ty bạn ít quan tâm — phỏng vấn trước để quen), **mục tiêu**, **mơ ước** (phỏng vấn sau cùng, khi đã "nóng máy").
- Mỗi CV **chỉnh nhẹ** theo JD: đưa từ khoá JD có (SwiftUI, Combine, CI…) mà bạn thực sự làm được lên đầu.
- Theo dõi bằng bảng: Công ty · Vị trí · Kênh · Ngày nộp · Trạng thái · Vòng tiếp theo · Ghi chú.
- **Không** nộp ồ ạt 50 nơi cùng ngày: không đủ sức chuẩn bị, dễ trùng lịch. 10–15 nơi/tuần là vừa.
- Phỏng vấn thường diễn ra trong giờ hành chính → chuẩn bị xin nghỉ phép/ra ngoài hợp lý; ưu tiên xin lịch online đầu giờ sáng hoặc cuối giờ chiều.
- Người hỏi "mong muốn lương" ngay ở form/cuộc gọi đầu → xem [week19](week19-final-interview.md) §6 trước khi trả lời.

---

## ⚠️ Bẫy hay gặp

1. **Vẽ sơ đồ ngay mà không hỏi yêu cầu** → thiết kế sai đề, mất 10 phút đầu.
2. **Thiết kế backend** (load balancer, sharding, Kafka) trong buổi mobile design → lạc đề. Chỉ nhắc backend ở mức API contract.
3. **Không nói trade-off**: "dùng Core Data" mà không nói vì sao không dùng file/SQLite → bị coi là học thuộc.
4. **Quên trạng thái lỗi & offline** — đây là phần phân biệt người làm app thật với người làm demo.
5. **Over-engineering**: VIPER + TCA + 5 module cho app 3 màn hình. Mức 2 năm: đơn giản, rõ ràng, đúng.
6. **Behavioral nói "bọn em"** suốt → interviewer không biết bạn làm gì. Dùng "em".
7. **Bịa số liệu** → bị hỏi "đo bằng gì?" là lộ. Không nhớ số thì nói khoảng + nguồn.
8. **Chê công ty cũ / sếp cũ** — dù đúng sự thật, vẫn bị trừ điểm.
9. **Phủ nhận dùng AI** hoặc ngược lại, nói "AI viết hết, em chỉ review" → cả hai đều xấu. Trả lời như B5.
10. **Im lặng khi suy nghĩ** — hãy nói "Cho em 30 giây suy nghĩ" rồi nghĩ to.

---

## 🛠 Bài tập

**Bài 1 — Khung 9 bước (T2)**
Không nhìn file, viết khung 9 bước lên 1 tờ A4 kèm 1–2 ý chính mỗi bước. Dán cạnh bàn học suốt tuần 18–19.

**Bài 2 — News feed (T3)**
Tự thiết kế feed trong 40 phút, **nói to + vẽ giấy**, ghi âm. So với §3.1, liệt kê 3 điểm thiếu.
<details><summary>Checklist tự chấm</summary>

- [ ] Hỏi ≥ 4 câu làm rõ, xác định rõ ngoài phạm vi
- [ ] Sơ đồ có ViewModel, Repository, API, DB/cache, ImageLoader
- [ ] Nêu SSOT (hoặc lý do không dùng)
- [ ] API có `cursor`, `nextCursor`, kích thước ảnh
- [ ] Cursor vs offset + lý do
- [ ] Downsampling + prefetch + huỷ khi reuse
- [ ] Like optimistic + rollback
- [ ] Trạng thái lỗi trang đầu / trang sau / offline
- [ ] ≥ 2 trade-off ở bước 9
</details>

**Bài 3 — Chat & image library (T4)**
Viết **1 trang** cho mỗi đề: sơ đồ + 5 quyết định quan trọng + 3 trade-off.
<details><summary>5 quyết định quan trọng nhất của bài chat</summary>

1. DB local là SSOT; UI quan sát DB.
2. Gửi tin: ghi DB `pending` trước (optimistic) với `clientId` → outbox gửi → ack cập nhật `sent`.
3. Idempotency bằng `clientId` → retry không tạo trùng.
4. WebSocket khi foreground + APNs khi background; luôn sync `after=lastId` khi mở app/reconnect.
5. Thứ tự theo `serverTimestamp`, tin pending ở cuối.
</details>

**Bài 4 — Bài design thứ 4 (CN, tuỳ chọn)**
Thiết kế **app đặt xe — màn theo dõi tài xế realtime** (kiểu Grab/Be). Gợi ý câu hỏi cần tự trả lời: tần suất cập nhật vị trí? WebSocket hay polling? Làm mượt marker giữa 2 điểm? Tiết kiệm pin? App vào background thì sao?
<details><summary>Ý chính</summary>

- WebSocket khi màn hình mở, nhận vị trí mỗi 2–5s; fallback polling khi socket lỗi.
- Nội suy (interpolate) vị trí marker giữa 2 điểm bằng animation để không "nhảy".
- Vào background: ngắt socket, dựa vào push cho sự kiện quan trọng (tài xế đã đến); Live Activity (iOS 16.1+) để hiện trạng thái trên màn khoá.
- Không lưu DB lịch sử vị trí; chỉ trạng thái chuyến đi hiện tại (để mở lại app khôi phục màn).
- Lỗi: mất kết nối → hiển thị "Đang kết nối lại…", giữ vị trí cuối cùng, không xoá bản đồ.
</details>

**Bài 5 — STAR (T5)**
Viết 3 story vào `docs/stories.md` (RepoScope) + 2 story từ công ty. Mỗi story ≤ 200 chữ, có **S/T/A/R** gạch chân, có ít nhất 1 con số.

**Bài 6 — Thẻ dự án (T5)**
Làm "thẻ dự án" (§6) cho 1–2 dự án công ty. Vẽ kiến trúc dự án trong 1 phút không nhìn.

**Bài 7 — Mock (T6, T7)**
Làm mock #1 (AI) và #2 (người). Điền bảng tự chấm §7.3. Ghi 3 hành động cụ thể để cải thiện cho mock #3–4 tuần sau.

**Bài 8 — Nộp CV (T2, T6)**
Tạo bảng theo dõi ứng tuyển, nộp ≥ 10 nơi (≥ 2 qua referral nếu có thể).

---

## 🎤 Câu hỏi phỏng vấn

> **Cách làm:** trả lời to thành tiếng (hoặc viết ra) trước khi mở đáp án. Câu có ⭐ rất hay gặp.

**Q1 ⭐ Bạn bắt đầu một bài mobile system design thế nào?**
<details><summary>Đáp án gợi ý</summary>

Hỏi làm rõ trước: tính năng trong/ngoài phạm vi, offline, realtime, quy mô, iOS tối thiểu, backend có sẵn chưa. Tóm tắt lại yêu cầu và xin xác nhận. Sau đó đi theo khung: thành phần high-level → luồng dữ liệu → API → offline/cache → pagination → media → lỗi → trade-off/mở rộng. Vừa làm vừa vẽ và hỏi interviewer muốn đào sâu phần nào.
</details>

**Q2 ⭐ Offset pagination và cursor pagination khác nhau thế nào? Feed nên dùng cái nào?**
<details><summary>Đáp án gợi ý</summary>

Offset (`page`, `limit`) đơn giản, nhảy trang được, nhưng khi dữ liệu mới chèn vào đầu, các trang sau bị dịch → trùng hoặc mất item. Cursor (`after=<id/token>`) trả về "từ sau item X", ổn định khi dữ liệu thay đổi, nhưng không nhảy trang được. Feed/chat/notification thay đổi liên tục → cursor. Client vẫn nên lọc trùng theo ID cho chắc.
</details>

**Q3 ⭐ "Single source of truth" trong app iOS là gì? Khi nào cần DB local?**
<details><summary>Đáp án gợi ý</summary>

Một nơi duy nhất giữ dữ liệu "đúng" mà UI quan sát — thường là DB local; network chỉ ghi vào DB. Tránh lệch dữ liệu giữa các màn (like ở màn detail, quay lại feed vẫn đúng), có offline tự nhiên. Cần khi: yêu cầu offline, dữ liệu dùng chung nhiều màn, mở app phải có nội dung ngay. Không cần khi dữ liệu tạm thời (kết quả search) — memory cache là đủ, tránh phức tạp migration.
</details>

**Q4 ⭐ Thiết kế cache ảnh cho list dài thế nào?**
<details><summary>Đáp án gợi ý</summary>

Pipeline: memory cache (`NSCache`, cost = bytes ảnh đã decode) → gộp request trùng đang tải → disk cache (data gốc trong `Caches/`, giới hạn dung lượng, LRU) → network. Decode + **downsample** bằng ImageIO ở background về đúng kích thước hiển thị, gán ở main. Prefetch khi sắp hiện, huỷ khi cell biến mất, kiểm tra cell reuse trước khi gán. Key gồm URL + kích thước. Xử lý memory warning.
</details>

**Q5 Gửi tin nhắn khi offline — làm sao không mất và không trùng?**
<details><summary>Đáp án gợi ý</summary>

Ghi tin vào DB với `status = pending` và `clientId` UUID **trước** khi gửi; UI hiển thị ngay. Outbox gửi khi có mạng (theo `NWPathMonitor`), retry với exponential backoff. Server dedupe theo `clientId` (idempotent), trả ack có `serverId` + timestamp → cập nhật `sent`. App bị kill vẫn còn tin pending trong DB → gửi tiếp khi mở lại. Quá số lần retry → `failed` + nút gửi lại.
</details>

**Q6 WebSocket, polling và push notification — chọn thế nào?**
<details><summary>Đáp án gợi ý</summary>

WebSocket cho realtime 2 chiều khi app foreground (chat, vị trí), phải tự reconnect + heartbeat; iOS sẽ đóng kết nối khi app vào background. Polling đơn giản, hợp dữ liệu không gấp, tốn pin/data hơn. Push (APNs) để báo khi app background/đã tắt, nhưng không đảm bảo giao và payload nhỏ → chỉ là "tín hiệu", dữ liệu thật vẫn sync qua API khi mở app.
</details>

**Q7 Xử lý lỗi mạng và retry trong app thế nào?**
<details><summary>Đáp án gợi ý</summary>

Phân loại lỗi: mạng/timeout/5xx → retry có giới hạn với exponential backoff + jitter; 4xx → không retry (401 → refresh token một lần qua actor, 429 → chờ theo header). Lỗi decode → log, không retry. UI: giữ dữ liệu cũ nếu có, thông báo rõ ràng, nút thử lại; mất mạng → banner offline, tự thử lại khi có mạng. Request ghi (POST) phải idempotent nếu retry.
</details>

**Q8 Optimistic update là gì? Rủi ro?**
<details><summary>Đáp án gợi ý</summary>

Cập nhật UI ngay như thể request đã thành công (like, gửi tin), rồi gọi API; lỗi thì rollback và báo. Lợi: app có cảm giác nhanh. Rủi ro: trạng thái lệch nếu rollback sai, nhiều thao tác liên tiếp (like/unlike nhanh) gây race → debounce hoặc chỉ gửi trạng thái cuối, hoặc xếp hàng theo thứ tự. Không dùng cho thao tác nghiêm trọng như thanh toán.
</details>

**Q9 ⭐ STAR là gì? Kể một câu chuyện theo STAR.**
<details><summary>Đáp án gợi ý</summary>

Situation – Task – Action – Result. Bối cảnh ngắn, vai trò của bạn, hành động cụ thể (phần dài nhất, dùng "em"), kết quả có số liệu + bài học. 1,5–2 phút. Kể một trong 3 story đã chuẩn bị (xem §4).
</details>

**Q10 ⭐ Vì sao bạn muốn nghỉ việc? / Vì sao chọn công ty chúng tôi?**
<details><summary>Đáp án gợi ý</summary>

Hướng về tương lai: muốn phát triển chiều sâu kỹ thuật/quy mô sản phẩm/quy trình mà môi trường hiện tại hạn chế; không chê công ty cũ. Vì sao chọn công ty: nêu **điều cụ thể** (sản phẩm bạn đã dùng, tech stack trong JD, bài blog kỹ thuật của công ty) — cho thấy bạn đã tìm hiểu.
</details>

**Q11 ⭐ Bạn dùng AI trong công việc như thế nào?**
<details><summary>Đáp án gợi ý</summary>

Trung thực + tích cực + có ranh giới: dùng để hiểu nhanh, boilerplate, gợi ý test, giải thích lỗi; từng phụ thuộc quá mức và nhận ra hạn chế; đặt quy tắc "không merge code mình không giải thích được"; tự viết side project không dùng AI để củng cố nền tảng; hiện dùng AI hiệu quả hơn vì biết kiểm chứng. (Xem B5.)
</details>

**Q12 Nếu app của bạn có 10 lần số user, phần nào phía client cần thay đổi?**
<details><summary>Đáp án gợi ý</summary>

Client ít bị ảnh hưởng trực tiếp bởi số user, nhưng bởi **dữ liệu lớn hơn và đa dạng thiết bị hơn**: phân trang & giới hạn cache chặt hơn, tối ưu khởi động, giảm request (cache HTTP, batch, debounce), xử lý rate limit/backoff tốt để không "DDoS" server khi server lỗi (thundering herd → thêm jitter), theo dõi bằng analytics/crash/MetricKit, feature flag để tắt tính năng lỗi từ xa, rollout theo % (phased release). Team lớn hơn → modularization, CI nhanh hơn.
</details>

---

## ✅ Checklist cuối tuần

- [ ] Viết khung 9 bước từ trí nhớ
- [ ] Tự thiết kế lại news feed trong 40' có ghi âm, đạt ≥ 7/9 mục checklist Bài 2
- [ ] Viết 1 trang cho chat offline và image loading library
- [ ] 3 story STAR (RepoScope) + 2 story (công ty) viết xong, mỗi story kể ≤ 2 phút
- [ ] Trả lời được 10 câu behavioral, đã ghi âm "Giới thiệu bản thân" và "Dùng AI thế nào"
- [ ] Thẻ dự án cho dự án công ty
- [ ] Mock #1 (AI) và #2 (người) xong, có bảng tự chấm
- [ ] Đã nộp ≥ 10 CV, có bảng theo dõi
- [ ] Trả lời được ≥ 10/12 câu 🎤 trước khi mở đáp án
- [ ] DSA tuần 18 xong (đặc biệt LRU Cache)
- [ ] Side project Milestone 18 xong (README có sơ đồ kiến trúc)

## 📖 Tài liệu đọc thêm

- [Mobile System Design (GitHub — weeeBox/mobile-system-design)](https://github.com/weeeBox/mobile-system-design) — khung trình bày + ví dụ, rất sát dạng phỏng vấn
- Alex Xu & Sahn Lam — *System Design Interview*: đọc chương "Design a news feed" và "Design a chat system" để hiểu phía server (chỉ đủ để thiết kế API hợp lý)
- Apple — [Images and Graphics Best Practices (WWDC18)](https://developer.apple.com/videos/play/wwdc2018/219/)
- Apple — [URLSessionWebSocketTask](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask) · [NWPathMonitor](https://developer.apple.com/documentation/network/nwpathmonitor)
- Kingfisher / Nuke source code — đọc phần `ImageCache`, `ImagePipeline` để so với thiết kế của bạn
- [The STAR method (The Muse)](https://www.themuse.com/advice/star-interview-method) — giải thích STAR kèm ví dụ
- Báo cáo thị trường IT Việt Nam của TopDev / ITviec — xem xu hướng tuyển dụng iOS trước khi nộp CV
