# 🧮 DSA — Lộ trình thuật toán 19 tuần (09/11/2026 – 21/03/2027)

> **Chạy song song** với các file tuần · Thời lượng: ~30'/tối T2–T6 (≈ 2,5 giờ/tuần)
> Mục tiêu **không phải** cày 300 bài. Mục tiêu là **nhận ra pattern** và **tự viết được lời giải sạch bằng Swift trong 25–30 phút, vừa viết vừa nói**.

---

## 0. Đọc trước khi bắt đầu

### 0.1. Thực tế phỏng vấn iOS ~2 năm ở Việt Nam
- Đa số công ty (outsource, product vừa, startup) hỏi **Easy → Medium**, thường **1–2 bài trong 30–45 phút**, nhiều nơi chỉ cho làm trên Google Docs / CoderPad / giấy, **không có autocomplete**.
- Một số nơi **không hỏi thuật toán**, thay bằng live coding iOS (dựng list, gọi API) hoặc take-home. Một số product/công ty nước ngoài hỏi kỹ hơn (Medium, đôi khi follow-up tối ưu).
- Bài hay gặp nhất: Two Sum, Valid Parentheses, Reverse Linked List, Merge Intervals, Group Anagrams, Longest Substring Without Repeating Characters, Binary Search, Level Order Traversal, Climbing Stairs, Number of Islands, **LRU Cache** (rất hợp iOS vì liên quan cache ảnh).
- → **Ưu tiên pattern hơn số lượng.** 6 bài/tuần làm kỹ + làm lại > 20 bài/tuần làm lướt.

### 0.2. Quy trình làm 1 bài (không dùng AI)

```
⏱ 0'  ─ Đọc đề, tự viết ra 2–3 ví dụ + edge case (mảng rỗng, 1 phần tử, trùng, số âm)
⏱ 5'  ─ Nói to ý tưởng brute force + độ phức tạp. Rồi tìm cách tốt hơn: "pattern nào?"
⏱ 10' ─ Code. KHÔNG chạy thử liên tục — trace bằng tay 1 ví dụ trước khi Run
⏱ 25' ─ Hết giờ mà chưa ra → mở HINT (trong <details> của file này)
⏱ 35' ─ Vẫn chưa ra → đọc lời giải (NeetCode video / Editorial), hiểu, ĐÓNG LẠI, tự gõ lại từ đầu
⏱ +3 ngày ─ Làm lại bài đó từ trang trắng. Không ra trong 15' = chưa thuộc pattern → đánh dấu 🔁
```

**Quy tắc:**
1. **Tắt Copilot / AI completion** trong Xcode và trên LeetCode (LeetCode có nút AI — đừng bấm).
2. Code trong **file `.swift` chạy bằng terminal** (`swift practice/dsa/w01_two_sum.swift`) hoặc trực tiếp trên LeetCode (chọn ngôn ngữ Swift). Ít nhất 1 bài/tuần viết trên **Google Docs/giấy** để quen không có compiler.
3. Mỗi bài xong, viết 1 dòng vào **sổ pattern**: `#1 Two Sum — Hashing: lưu "phần bù" vào dict — O(n)/O(n)`.
4. Được hỏi AI **sau khi đã có lời giải**: "lời giải của tôi còn edge case nào sai? có cách nào gọn hơn?".
5. Chủ Nhật: làm lại 1–2 bài 🔁 của tuần trước.

### 0.3. Khung nói khi phỏng vấn (think aloud)
1. **Nhắc lại đề + hỏi làm rõ**: "Input có thể rỗng không? Có số âm không? Có trùng không? Kết quả trả theo thứ tự nào?"
2. **Brute force trước**: "Cách đơn giản là 2 vòng lặp, O(n²). Em có thể tối ưu bằng…"
3. **Chốt độ phức tạp trước khi code**: "Dùng dictionary → O(n) time, O(n) space."
4. **Code + giải thích**, đặt tên biến rõ (`left`, `right`, `seen`, không `a`, `b`, `x1`).
5. **Tự test** bằng ví dụ + edge case, trace từng bước.

---

## 1. Swift cho phỏng vấn coding — những thứ phải thuộc

### 1.1. String → luôn chuyển sang `Array`
`String` không truy cập `s[i]` được (xem tuần 1 §3). Trong bài thuật toán:
```swift
let s = "hello"
let chars = Array(s)              // [Character], truy cập chars[i] O(1)
let bytes = Array(s.utf8)         // [UInt8] — nhanh nhất khi đề nói "chỉ chữ thường a-z"
let str = String(chars)           // [Character] → String
let word = String(chars[1...3])   // "ell" — ArraySlice<Character> → String
```

### 1.2. Character "arithmetic" bằng `asciiValue`
```swift
let ch: Character = "c"
let index = Int(ch.asciiValue! - Character("a").asciiValue!)   // 2  → dùng cho mảng đếm 26 phần tử
let back = Character(UnicodeScalar(UInt8(97 + 3)))             // "d" (97 = "a")
ch.isLetter; ch.isNumber; ch.isUppercase; ch.lowercased()      // lowercased() trả String
Character("7").wholeNumberValue                                // Optional(7)
```
Với `Array(s.utf8)` thì mỗi phần tử đã là `UInt8`: `Int(byte) - 97`.

### 1.3. Mảng 2 chiều, khởi tạo, swap
```swift
var grid = [[Int]](repeating: [Int](repeating: 0, count: cols), count: rows)
var count = [Int](repeating: 0, count: 26)
nums.swapAt(i, j)                         // KHÔNG dùng swap(&nums[i], &nums[j]) → lỗi exclusive access
(a, b) = (b, a)                           // swap 2 biến
for i in stride(from: n - 1, through: 0, by: -1) { }   // duyệt ngược
for (i, x) in nums.enumerated() { }
```

### 1.4. Dictionary đếm & nhóm
```swift
var freq: [Character: Int] = [:]
for c in s { freq[c, default: 0] += 1 }
var groups: [String: [String]] = [:]
groups[key, default: []].append(word)
let sortedByCount = freq.sorted { $0.value > $1.value }   // [(key, value)]
```

### 1.5. Không có Stack/Queue/Deque/Heap sẵn
- **Stack**: dùng `Array` — `append` / `popLast()` / `last`. Tất cả O(1).
- **Queue**: `removeFirst()` trên Array là **O(n)** → BFS lớn sẽ chậm. 3 cách:
  1. **Thay cả tầng** (BFS theo level): `queue = nextLevel` — gọn nhất, dùng ở tuần 10.
  2. **Con trỏ `head`**: không xoá, chỉ tăng chỉ số.
  3. Struct `Queue` bên dưới. (Thư viện `swift-collections` có `Deque` nhưng phỏng vấn **không** được import.)

```swift
struct Queue<T> {
    private var items: [T] = []
    private var head = 0
    var isEmpty: Bool { head == items.count }
    var count: Int { items.count - head }
    mutating func enqueue(_ x: T) { items.append(x) }
    mutating func dequeue() -> T? {
        guard head < items.count else { return nil }
        let x = items[head]
        head += 1
        if head > 64 && head * 2 > items.count {   // dọn rác định kỳ để không phình bộ nhớ
            items.removeFirst(head); head = 0
        }
        return x
    }
}
```

- **Heap / Priority Queue**: phải **tự viết** (~40 dòng). Thuộc lòng — bị yêu cầu viết lại là chuyện bình thường ở tuần 11:

```swift
struct Heap<T> {
    private var items: [T] = []
    private let areSorted: (T, T) -> Bool   // true nếu a phải đứng TRÊN b

    init(sort: @escaping (T, T) -> Bool) { self.areSorted = sort }

    var count: Int { items.count }
    var isEmpty: Bool { items.isEmpty }
    var peek: T? { items.first }

    mutating func push(_ value: T) {                 // O(log n)
        items.append(value)
        siftUp(from: items.count - 1)
    }

    mutating func pop() -> T? {                      // O(log n)
        guard !items.isEmpty else { return nil }
        items.swapAt(0, items.count - 1)
        let top = items.removeLast()
        siftDown(from: 0)
        return top
    }

    private mutating func siftUp(from index: Int) {
        var child = index
        var parent = (child - 1) / 2
        while child > 0 && areSorted(items[child], items[parent]) {
            items.swapAt(child, parent)
            child = parent
            parent = (child - 1) / 2
        }
    }

    private mutating func siftDown(from index: Int) {
        var parent = index
        while true {
            let left = 2 * parent + 1, right = left + 1
            var candidate = parent
            if left < items.count && areSorted(items[left], items[candidate]) { candidate = left }
            if right < items.count && areSorted(items[right], items[candidate]) { candidate = right }
            if candidate == parent { return }
            items.swapAt(parent, candidate)
            parent = candidate
        }
    }
}
// var minHeap = Heap<Int>(sort: <)      var maxHeap = Heap<Int>(sort: >)
// var byDistance = Heap<[Int]>(sort: { $0[0] < $1[0] })
```
> Mẹo phỏng vấn: nếu thời gian gấp, nói *"Swift không có heap built-in; em sẽ giả sử có `Heap` với push/pop O(log n), nếu anh/chị muốn em viết luôn"* — đa số interviewer chấp nhận. Nhưng phải **viết được** nếu bị yêu cầu.

### 1.6. `inout`, nested function, closure đệ quy
```swift
func dfs(_ grid: inout [[Character]], _ r: Int, _ c: Int) { ... }   // sửa grid tại chỗ
dfs(&grid, 0, 0)

// Cách gọn hơn trong phỏng vấn: nested function bắt (capture) biến bên ngoài
func numIslands(_ grid: [[Character]]) -> Int {
    var grid = grid                  // copy để sửa (tham số hàm là let)
    func sink(_ r: Int, _ c: Int) { /* sửa grid trực tiếp */ }
    ...
}
```
- Tham số hàm là **`let`** → muốn sửa phải `var copy = param` hoặc `inout`.
- Nested function capture biến `var` của hàm ngoài → dùng cho backtracking/DFS rất gọn.

### 1.7. Đệ quy & giới hạn stack
- Stack có giới hạn: trên iOS main thread khoảng **1 MB**, thread phụ mặc định **512 KB** (macOS main thread 8 MB; môi trường chấm của LeetCode thường rộng hơn). Đệ quy sâu ~10⁴–10⁵ tầng có thể **crash stack overflow** (`EXC_BAD_ACCESS`), đặc biệt khi mỗi frame lớn.
- Cây/linked list có thể "thoái hoá" thành đường thẳng 10⁵ node → nói với interviewer: *"Đệ quy O(h) stack; nếu cây lệch có thể tràn stack, em có thể chuyển sang iterative với stack tường minh."*
- Memo hoá đệ quy bằng `[Int: Int]` hoặc mảng.

### 1.8. Bẫy Swift khác
| Bẫy | Đúng |
|---|---|
| `nums[0]` khi mảng có thể rỗng → crash | `guard !nums.isEmpty` hoặc `nums.first` |
| `for i in 0..<n` với `n = 0` OK, nhưng `0...n-1` với `n = 0` → **crash** (range ngược) | Dùng `0..<n` |
| `for i in 1...n` với `n = 0` → crash | `guard n >= 1` trước |
| `Int.max + 1` crash | Dùng `Int.max` làm "vô cực" nhưng đừng cộng thêm; hoặc dùng `amount + 1` |
| `mid = (lo + hi) / 2` | `lo + (hi - lo) / 2` (thói quen tốt, Swift tràn số sẽ crash) |
| `s.count` trong vòng lặp | Gán `let n = chars.count` 1 lần |
| `sorted()` trả mảng mới, `sort()` sửa tại chỗ | — |
| `Set`/`Dictionary` không có thứ tự | Đề yêu cầu thứ tự → sort trước khi trả |
| `ListNode`/`TreeNode` là **class** | So sánh node bằng `===`, không `==` |
| `stack.popLast()` trả Optional, `removeLast()` crash nếu rỗng | Dùng `popLast()` khi không chắc |

### 1.9. Template class LeetCode dùng
```swift
public class ListNode {
    public var val: Int
    public var next: ListNode?
    public init(_ val: Int, _ next: ListNode? = nil) { self.val = val; self.next = next }
}
public class TreeNode {
    public var val: Int
    public var left: TreeNode?
    public var right: TreeNode?
    public init(_ val: Int, _ left: TreeNode? = nil, _ right: TreeNode? = nil) {
        self.val = val; self.left = left; self.right = right
    }
}
// Helper tự viết để test local:
func makeList(_ a: [Int]) -> ListNode? {
    var head: ListNode? = nil
    for v in a.reversed() { head = ListNode(v, head) }
    return head
}
func toArray(_ head: ListNode?) -> [Int] {
    var out: [Int] = []; var cur = head
    while let n = cur { out.append(n.val); cur = n.next }
    return out
}
```

### 1.10. Bảng Big-O "đọc ngược" từ ràng buộc
| `n` tối đa | Độ phức tạp chấp nhận | Gợi ý pattern |
|---|---|---|
| ≤ 10–20 | O(2ⁿ), O(n!) | Backtracking |
| ≤ 10³ | O(n²) | 2 vòng lặp, DP 2D |
| ≤ 10⁵–10⁶ | O(n log n), O(n) | Sort, hashing, two pointers, sliding window, heap |
| ≤ 10⁹ | O(log n) | Binary search |

---

## 2. Lịch tổng quan

| Tuần | Ngày | Pattern | Số bài | Bài bắt buộc (⭐) |
|---|---|---|---|---|
| 1 | 09/11 – 15/11 | Array & Hashing cơ bản | 6 | Valid Anagram, Two Sum, Contains Duplicate |
| 2 | 16/11 – 22/11 | Array & Hashing | 6 | Group Anagrams, Top K Frequent |
| 3 | 23/11 – 29/11 | Two Pointers | 6 | Valid Palindrome, 3Sum |
| 4 | 30/11 – 06/12 | Sliding Window | 6 | Best Time to Buy and Sell Stock, Longest Substring… |
| 5 | 07/12 – 13/12 | Stack | 6 | Valid Parentheses, Daily Temperatures |
| 6 | 14/12 – 20/12 | Binary Search | 6 | Binary Search, Search in Rotated Sorted Array |
| 7 | 21/12 – 27/12 | Linked List | 6 | Reverse Linked List, Merge Two Sorted Lists |
| 8 | 28/12 – 03/01 | Linked List + Recursion | 6 | Swap Nodes in Pairs, Palindrome Linked List |
| 9 | 04/01 – 10/01 | Trees — DFS | 6 | Maximum Depth, Validate BST |
| 10 | 11/01 – 17/01 | Trees — BFS | 6 | Level Order Traversal, Right Side View |
| 11 | 18/01 – 24/01 | Heap / Priority Queue | 6 | Kth Largest Element, Last Stone Weight |
| 12 | 25/01 – 31/01 | Backtracking cơ bản | 6 | Subsets, Permutations |
| 13 | 01/02 – 07/02 | 🧧 Ôn lại (redo) | 3 | Two Sum, Valid Parentheses, Reverse Linked List |
| 14 | 08/02 – 14/02 | 🧧 Ôn lại (redo) | 3 | Best Time to Buy and Sell Stock, Merge Intervals, Number of Islands |
| 15 | 15/02 – 21/02 | Graph BFS/DFS | 6 | Number of Islands, Course Schedule |
| 16 | 22/02 – 28/02 | DP 1D | 6 | Climbing Stairs, House Robber, Coin Change |
| 17 | 01/03 – 07/03 | Intervals + Greedy | 6 | Merge Intervals, Jump Game |
| 18 | 08/03 – 14/03 | Mixed — timed | 6 | LRU Cache |
| 19 | 15/03 – 21/03 | Mixed — timed (mock) | 6 | — |

**Phân bổ mỗi tuần:** T2 đọc pattern + bài 1 · T3 bài 2 · T4 bài 3 · T5 bài 4 · T6 bài 5 · T7 (nếu còn sức) bài 6 · CN làm lại bài 🔁.
Nếu tuần đó file tuần đã ghi cụ thể bài nào vào ngày nào (ví dụ tuần 1: T4 Valid Anagram, T5 Two Sum, T6 Contains Duplicate) thì **theo file tuần**.

**Mức độ:** 🟢 Easy · 🟡 Medium. Không có bài Hard — với mức 2 năm, thời gian dành cho Hard nên chuyển sang iOS.

---

## Tuần 1 (09/11 – 15/11) — Array & Hashing cơ bản

### Pattern: Hashing
**Nhận diện:** đề có chữ "tồn tại / trùng / đếm số lần / tìm cặp / anagram / nhóm theo…". Brute force là 2 vòng lặp O(n²) → dùng `Set`/`Dictionary` để tra cứu O(1) → O(n).

**Template:**
```swift
// 1) Kiểm tra đã gặp chưa
var seen = Set<Int>()
for x in nums {
    if seen.contains(x) { /* tìm thấy */ }
    seen.insert(x)
}
// 2) Đếm tần suất
var freq: [Int: Int] = [:]
for x in nums { freq[x, default: 0] += 1 }
// 3) Lưu "giá trị → index" để tìm phần bù
var indexOf: [Int: Int] = [:]
```

### Bài tập
| # | Bài | Mức | Ngày gợi ý |
|---|---|---|---|
| 1 | [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) ⭐ | 🟢 | T6 |
| 2 | [242. Valid Anagram](https://leetcode.com/problems/valid-anagram/) ⭐ | 🟢 | T4 |
| 3 | [1. Two Sum](https://leetcode.com/problems/two-sum/) ⭐ | 🟢 | T5 |
| 4 | [1480. Running Sum of 1d Array](https://leetcode.com/problems/running-sum-of-1d-array/) | 🟢 | T2 (khởi động) |
| 5 | [383. Ransom Note](https://leetcode.com/problems/ransom-note/) | 🟢 | T3 |
| 6 | [169. Majority Element](https://leetcode.com/problems/majority-element/) | 🟢 | T7/CN |

<details><summary>💡 Hint từng bài</summary>

- **217:** `Set`; gặp phần tử đã có → `true`. Một dòng: `Set(nums).count != nums.count` — nói được cả 2 cách và ưu điểm của cách vòng lặp (dừng sớm).
- **242:** Đếm tần suất `s` (+1) và `t` (−1), cuối cùng tất cả phải bằng 0. Chỉ chữ thường → mảng 26 phần tử. Cách khác: `s.sorted() == t.sorted()` O(n log n).
- **1:** Duyệt 1 lần, với mỗi `x` hỏi "`target - x` đã gặp chưa?" — dict lưu giá trị → index. **Kiểm tra trước, insert sau** để không tự ghép với chính mình.
- **1480:** `result[i] = result[i-1] + nums[i]` — đây là **prefix sum**, sẽ gặp lại.
- **383:** Đếm ký tự của `magazine`, trừ dần bằng `ransomNote`; âm → `false`.
- **169:** Cách 1: dict đếm, trả phần tử có count > n/2. Cách 2 (follow-up O(1) space): **Boyer–Moore voting** — giữ `candidate` và `count`, gặp giống +1, khác −1, về 0 thì đổi candidate.
</details>

### Lời giải mẫu

<details><summary>✅ 1. Two Sum — lời giải + phân tích</summary>

```swift
func twoSum(_ nums: [Int], _ target: Int) -> [Int] {
    var seen: [Int: Int] = [:]          // giá trị → index
    for (i, x) in nums.enumerated() {
        if let j = seen[target - x] { return [j, i] }
        seen[x] = i
    }
    return []
}
// twoSum([2,7,11,15], 9) == [0,1]   twoSum([3,3], 6) == [0,1]
```
- **Time O(n)**: duyệt 1 lần, mỗi lần tra/ghi dict O(1) trung bình. **Space O(n)**: dict.
- Brute force 2 vòng lặp: O(n²) time, O(1) space → nói ra để thể hiện trade-off **thời gian ↔ bộ nhớ**.
- Edge case: `[3,3]` target 6 → nhờ "kiểm tra trước, insert sau" mà không trả `[0,0]`.
</details>

<details><summary>✅ 242. Valid Anagram — 2 cách</summary>

```swift
// Cách 1: chỉ chữ thường a–z (đúng ràng buộc đề) — mảng 26 phần tử
func isAnagram(_ s: String, _ t: String) -> Bool {
    guard s.count == t.count else { return false }
    var freq = [Int](repeating: 0, count: 26)
    for ch in s.utf8 { freq[Int(ch) - 97] += 1 }      // 97 = "a"
    for ch in t.utf8 { freq[Int(ch) - 97] -= 1 }
    return freq.allSatisfy { $0 == 0 }
}

// Cách 2: follow-up "nếu input có Unicode?" — Dictionary theo Character
func isAnagram2(_ s: String, _ t: String) -> Bool {
    var freq: [Character: Int] = [:]
    for ch in s { freq[ch, default: 0] += 1 }
    for ch in t { freq[ch, default: 0] -= 1 }
    return freq.values.allSatisfy { $0 == 0 }
}
```
- Time O(n), Space O(1) (26 phần tử cố định) cho cách 1; O(k) với k số ký tự khác nhau cho cách 2.
- Follow-up Unicode chính là câu hỏi Swift hay: `Character` là grapheme cluster nên cách 2 đúng với emoji/tiếng Việt, cách 1 thì không.
</details>

---

## Tuần 2 (16/11 – 22/11) — Array & Hashing

### Pattern: Hashing nâng cao + Prefix
**Nhận diện:**
- "Nhóm các phần tử có cùng đặc điểm" → **tạo key chuẩn hoá** (sort chuỗi, mảng đếm) rồi `Dictionary(grouping:)`.
- "Top K phổ biến nhất" → đếm + **bucket sort** (O(n)) hoặc heap (O(n log k), tuần 11).
- "Tích/tổng các phần tử trừ chính nó", "tổng đoạn con" → **prefix/suffix**.
- "Dãy liên tiếp dài nhất trong mảng không sắp xếp" → `Set` + chỉ bắt đầu đếm từ phần tử "đầu chuỗi".

**Template prefix sum:**
```swift
var prefix = [Int](repeating: 0, count: nums.count + 1)
for i in 0..<nums.count { prefix[i + 1] = prefix[i] + nums[i] }
// tổng đoạn [l, r] = prefix[r + 1] - prefix[l]
```

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [724. Find Pivot Index](https://leetcode.com/problems/find-pivot-index/) | 🟢 |
| 2 | [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/) ⭐ | 🟡 |
| 3 | [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) ⭐ | 🟡 |
| 4 | [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | 🟡 |
| 5 | [36. Valid Sudoku](https://leetcode.com/problems/valid-sudoku/) | 🟡 |
| 6 | [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **724:** Tổng toàn mảng `total`; duyệt giữ `leftSum`; pivot khi `leftSum == total - leftSum - nums[i]`.
- **49:** Key = `String(s.sorted())`. Follow-up nhanh hơn: key = mảng đếm 26 phần tử chuyển thành String.
- **347:** Đếm → `buckets[count]` chứa các số có tần suất đó → duyệt bucket từ cao xuống. O(n).
- **238:** `answer[i] = (tích bên trái i) * (tích bên phải i)`. Lượt 1 đi xuôi ghi tích trái, lượt 2 đi ngược nhân tích phải. Không dùng phép chia.
- **36:** 3 nhóm `Set`: theo hàng, theo cột, theo ô 3×3 (`(r / 3) * 3 + c / 3`).
- **128:** Bỏ hết vào `Set`. Chỉ bắt đầu đếm từ `x` nếu `x - 1` **không** có trong set → mỗi phần tử được duyệt tối đa 2 lần → O(n).
</details>

<details><summary>✅ 49. Group Anagrams</summary>

```swift
func groupAnagrams(_ strs: [String]) -> [[String]] {
    var groups: [String: [String]] = [:]
    for s in strs {
        let key = String(s.sorted())       // "eat" → "aet"
        groups[key, default: []].append(s)
    }
    return Array(groups.values)
}
```
- n chuỗi, độ dài tối đa k: **Time O(n · k log k)** (sort mỗi chuỗi), **Space O(n · k)**.
- Viết gọn: `Array(Dictionary(grouping: strs, by: { String($0.sorted()) }).values)` — nói được cả 2 cách là điểm cộng.
</details>

<details><summary>✅ 347. Top K Frequent Elements — bucket sort</summary>

```swift
func topKFrequent(_ nums: [Int], _ k: Int) -> [Int] {
    var freq: [Int: Int] = [:]
    for x in nums { freq[x, default: 0] += 1 }
    // buckets[i] = các số xuất hiện đúng i lần (tối đa n lần)
    var buckets = [[Int]](repeating: [], count: nums.count + 1)
    for (num, c) in freq { buckets[c].append(num) }
    var result: [Int] = []
    for c in stride(from: nums.count, through: 1, by: -1) {
        for num in buckets[c] {
            result.append(num)
            if result.count == k { return result }
        }
    }
    return result
}
```
- **Time O(n), Space O(n).** Cách "dễ nghĩ": `freq.sorted { $0.value > $1.value }.prefix(k)` → O(n log n) — chấp nhận được, nhưng interviewer sẽ hỏi "nhanh hơn được không?".
</details>

---

## Tuần 3 (23/11 – 29/11) — Two Pointers

### Pattern
**Nhận diện:** mảng/chuỗi **đã sắp xếp** (hoặc sort được), tìm cặp/bộ ba thoả điều kiện, palindrome, đảo ngược tại chỗ, gộp 2 mảng, "di chuyển phần tử tại chỗ O(1) space".

**2 dạng:**
```swift
// A) Hai đầu đi vào giữa (mảng sorted, palindrome)
var left = 0, right = arr.count - 1
while left < right {
    if /* điều kiện thoả */ { ... }
    else if /* cần lớn hơn */ { left += 1 }
    else { right -= 1 }
}

// B) Nhanh–chậm cùng chiều (lọc/ghi đè tại chỗ)
var write = 0
for read in 0..<arr.count where /* giữ arr[read] */ {
    arr[write] = arr[read]; write += 1
}
```

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) ⭐ | 🟢 |
| 2 | [283. Move Zeroes](https://leetcode.com/problems/move-zeroes/) | 🟢 |
| 3 | [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/) | 🟢 |
| 4 | [167. Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | 🟡 |
| 5 | [15. 3Sum](https://leetcode.com/problems/3sum/) ⭐ | 🟡 |
| 6 | [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **125:** Lọc chữ/số + lowercase → `Array`, hai đầu so sánh. Follow-up O(1) space: không lọc trước, mà skip ký tự không hợp lệ trong vòng lặp.
- **283:** Dạng B: con trỏ `write` cho phần tử khác 0, cuối cùng điền 0. Hoặc `swapAt(write, read)`.
- **88:** Ghi **từ cuối** mảng `nums1` (chỗ trống ở cuối) → không đè dữ liệu chưa xét.
- **167:** Dạng A: tổng nhỏ → `left += 1`, lớn → `right -= 1`. Chú ý đề trả index **1-based**.
- **15:** Sort, cố định `i`, chạy Two Sum II trên phần còn lại. **Bỏ trùng** ở cả `i` và `left`.
- **11:** Diện tích = `min(h[l], h[r]) * (r - l)`. Luôn dịch cột **thấp hơn** (dịch cột cao không bao giờ làm tăng diện tích).
</details>

<details><summary>✅ 125. Valid Palindrome</summary>

```swift
func isPalindrome(_ s: String) -> Bool {
    let chars = Array(s.lowercased().filter { $0.isLetter || $0.isNumber })
    var l = 0, r = chars.count - 1
    while l < r {
        if chars[l] != chars[r] { return false }
        l += 1; r -= 1
    }
    return true
}
```
Time O(n), Space O(n) (mảng đã lọc). Chuỗi rỗng → `r = -1`, vòng lặp không chạy → `true` (đúng đề).
</details>

<details><summary>✅ 15. 3Sum</summary>

```swift
func threeSum(_ nums: [Int]) -> [[Int]] {
    let a = nums.sorted()
    var result: [[Int]] = []
    for i in 0..<a.count {
        if i > 0 && a[i] == a[i - 1] { continue }          // bỏ trùng phần tử đầu
        var l = i + 1, r = a.count - 1
        while l < r {
            let sum = a[i] + a[l] + a[r]
            if sum < 0 { l += 1 }
            else if sum > 0 { r -= 1 }
            else {
                result.append([a[i], a[l], a[r]])
                l += 1; r -= 1
                while l < r && a[l] == a[l - 1] { l += 1 }   // bỏ trùng phần tử thứ 2
            }
        }
    }
    return result
}
// [-1,0,1,2,-1,-4] → [[-1,-1,2],[-1,0,1]]   [0,0,0,0] → [[0,0,0]]
```
- **Time O(n²)** (sort O(n log n) + vòng ngoài × two pointers O(n)). **Space O(n)** cho mảng sorted (hoặc O(log n)–O(n) tuỳ sort nếu sort tại chỗ).
- Bẫy hay sai: quên bỏ trùng → kết quả có bộ ba lặp lại.
</details>

---

## Tuần 4 (30/11 – 06/12) — Sliding Window

### Pattern
**Nhận diện:** "đoạn con **liên tiếp** (subarray/substring) dài nhất / ngắn nhất / có tổng… / chứa…". Brute force xét mọi đoạn O(n²); cửa sổ trượt mỗi phần tử vào 1 lần, ra 1 lần → O(n).

**Template cửa sổ thay đổi kích thước:**
```swift
var left = 0, best = 0
var window: [Character: Int] = [:]           // trạng thái cửa sổ
for right in 0..<chars.count {
    window[chars[right], default: 0] += 1    // 1. mở rộng phải
    while /* cửa sổ không hợp lệ */ false {  // 2. co trái tới khi hợp lệ
        window[chars[left]]! -= 1
        left += 1
    }
    best = max(best, right - left + 1)        // 3. cập nhật kết quả
}
```
**Cửa sổ cố định k:** cộng phần tử mới vào, trừ phần tử `i - k` ra.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) ⭐ | 🟢 |
| 2 | [643. Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i/) | 🟢 |
| 3 | [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) ⭐ | 🟡 |
| 4 | [209. Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum/) | 🟡 |
| 5 | [424. Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | 🟡 |
| 6 | [567. Permutation in String](https://leetcode.com/problems/permutation-in-string/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **121:** Giữ `minPrice` đã thấy; lợi nhuận tại ngày i = `price - minPrice`.
- **643:** Cửa sổ cố định k: `sum += nums[i] - nums[i - k]`.
- **3:** Dict ký tự → vị trí gần nhất; gặp trùng **trong cửa sổ** thì nhảy `left = prev + 1`.
- **209:** Số dương → mở rộng tới khi `sum >= target`, rồi co trái tối đa, cập nhật độ dài nhỏ nhất.
- **424:** Cửa sổ hợp lệ nếu `(độ dài) - (tần suất ký tự nhiều nhất) <= k`.
- **567:** Cửa sổ cố định độ dài `s1.count`, so mảng đếm 26 phần tử của cửa sổ với `s1`.
</details>

<details><summary>✅ 121. Best Time to Buy and Sell Stock</summary>

```swift
func maxProfit(_ prices: [Int]) -> Int {
    var minPrice = Int.max, best = 0
    for p in prices {
        minPrice = min(minPrice, p)
        best = max(best, p - minPrice)
    }
    return best
}
```
Time O(n), Space O(1). Đây là "cửa sổ" ngầm: `left` = ngày giá thấp nhất, `right` = hôm nay.
</details>

<details><summary>✅ 3. Longest Substring Without Repeating Characters</summary>

```swift
func lengthOfLongestSubstring(_ s: String) -> Int {
    let chars = Array(s)
    var lastIndex: [Character: Int] = [:]   // ký tự → vị trí gần nhất
    var left = 0, best = 0
    for (right, ch) in chars.enumerated() {
        if let prev = lastIndex[ch], prev >= left {
            left = prev + 1                  // co cửa sổ qua khỏi ký tự trùng
        }
        lastIndex[ch] = right
        best = max(best, right - left + 1)
    }
    return best
}
// "abcabcbb" → 3, "bbbbb" → 1, "pwwkew" → 3, "" → 0, "abba" → 2
```
- **Time O(n), Space O(min(n, bảng chữ cái)).**
- Bẫy: thiếu điều kiện `prev >= left` → test `"abba"` sai (left bị kéo **lùi**). Luôn tự test `"abba"`.
</details>

---

## Tuần 5 (07/12 – 13/12) — Stack

### Pattern
**Nhận diện:** ngoặc khớp nhau, "phần tử gần nhất lớn hơn/nhỏ hơn" (**monotonic stack**), tính biểu thức, undo/back (như navigation stack của `UINavigationController`!).

```swift
// Monotonic stack (giảm dần): tìm "phần tử lớn hơn tiếp theo"
var stack: [Int] = []                        // lưu INDEX
for (i, x) in nums.enumerated() {
    while let top = stack.last, nums[top] < x {
        stack.removeLast()
        // x là "next greater" của nums[top]
    }
    stack.append(i)
}
```
Mỗi phần tử push 1 lần, pop tối đa 1 lần → **O(n)** dù có `while` lồng.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) ⭐ | 🟢 |
| 2 | [232. Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) | 🟢 |
| 3 | [155. Min Stack](https://leetcode.com/problems/min-stack/) | 🟡 |
| 4 | [150. Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/) | 🟡 |
| 5 | [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) ⭐ | 🟡 |
| 6 | [496. Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/) | 🟢 |

<details><summary>💡 Hint từng bài</summary>

- **20:** Dict `đóng → mở`; gặp dấu đóng thì `popLast()` phải bằng dấu mở tương ứng; cuối cùng stack rỗng.
- **232:** 2 stack `inStack`, `outStack`; chỉ đổ từ in sang out khi out rỗng → amortized O(1).
- **155:** Stack thứ 2 lưu min tại mỗi thời điểm (hoặc lưu tuple `(value, minSoFar)`).
- **150:** Gặp số → push; gặp toán tử → pop 2 (chú ý thứ tự: `b = pop, a = pop, push(a op b)`), chia lấy phần nguyên về 0 (Swift `/` đã làm vậy).
- **739:** Monotonic stack lưu index, nhiệt độ giảm dần.
- **496:** Monotonic stack trên `nums2` tạo dict `số → next greater`, rồi tra cho `nums1`.
</details>

<details><summary>✅ 20. Valid Parentheses</summary>

```swift
func isValid(_ s: String) -> Bool {
    let pairs: [Character: Character] = [")": "(", "]": "[", "}": "{"]
    var stack: [Character] = []
    for ch in s {
        if let open = pairs[ch] {                 // ch là dấu đóng
            guard stack.popLast() == open else { return false }
        } else {
            stack.append(ch)
        }
    }
    return stack.isEmpty
}
// "()[]{}" true · "(]" false · "([)]" false · "(" false · "]" false
```
Time O(n), Space O(n). `popLast()` trả `nil` khi stack rỗng → so với `open` là `false` → xử lý gọn case `"]"`.
</details>

<details><summary>✅ 739. Daily Temperatures</summary>

```swift
func dailyTemperatures(_ temps: [Int]) -> [Int] {
    var answer = [Int](repeating: 0, count: temps.count)
    var stack: [Int] = []                     // lưu INDEX, nhiệt độ giảm dần
    for (i, t) in temps.enumerated() {
        while let top = stack.last, temps[top] < t {
            stack.removeLast()
            answer[top] = i - top
        }
        stack.append(i)
    }
    return answer
}
// [73,74,75,71,69,72,76,73] → [1,1,4,2,1,1,0,0]
```
Time O(n) (mỗi index vào/ra stack 1 lần), Space O(n).
</details>

---

## Tuần 6 (14/12 – 20/12) — Binary Search

### Pattern
**Nhận diện:** dữ liệu **sắp xếp** (hoặc "đơn điệu": có điểm mà trước đó điều kiện sai, sau đó đúng), đề yêu cầu O(log n), n rất lớn.

```swift
// Dạng 1: tìm chính xác
var lo = 0, hi = nums.count - 1
while lo <= hi {
    let mid = lo + (hi - lo) / 2
    if nums[mid] == target { return mid }
    if nums[mid] < target { lo = mid + 1 } else { hi = mid - 1 }
}
// Dạng 2: tìm vị trí ĐẦU TIÊN thoả điều kiện (lower bound) — khoảng [lo, hi)
var lo = 0, hi = nums.count
while lo < hi {
    let mid = lo + (hi - lo) / 2
    if condition(mid) { hi = mid } else { lo = mid + 1 }
}
// lo = vị trí đầu tiên condition đúng (có thể = count nếu không có)
```
> Thuộc **1 template** và dùng nhất quán. Đa số bug binary search là lẫn `<` với `<=` và `hi = mid` với `hi = mid - 1`.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [704. Binary Search](https://leetcode.com/problems/binary-search/) ⭐ | 🟢 |
| 2 | [35. Search Insert Position](https://leetcode.com/problems/search-insert-position/) | 🟢 |
| 3 | [278. First Bad Version](https://leetcode.com/problems/first-bad-version/) | 🟢 |
| 4 | [74. Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/) | 🟡 |
| 5 | [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) | 🟡 |
| 6 | [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) ⭐ | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **704:** Dạng 1.
- **35:** Dạng 2 với `condition = nums[mid] >= target`.
- **278:** Dạng 2 với `condition = isBadVersion(mid)`, khoảng `[1, n]`. (Trên LeetCode Swift, hàm có sẵn trong class `VersionControl`.)
- **74:** Coi ma trận m×n là mảng 1 chiều: `row = mid / n`, `col = mid % n`.
- **153:** So `nums[mid]` với `nums[hi]`: lớn hơn → min ở bên phải (`lo = mid + 1`), ngược lại `hi = mid`.
- **33:** Tại mỗi bước, **ít nhất một nửa đã sắp xếp** — xác định nửa đó, xem target có nằm trong nửa đó không.
</details>

<details><summary>✅ 704 & 35 — 2 template</summary>

```swift
func search(_ nums: [Int], _ target: Int) -> Int {
    var lo = 0, hi = nums.count - 1
    while lo <= hi {
        let mid = lo + (hi - lo) / 2
        if nums[mid] == target { return mid }
        if nums[mid] < target { lo = mid + 1 } else { hi = mid - 1 }
    }
    return -1
}

func searchInsert(_ nums: [Int], _ target: Int) -> Int {
    var lo = 0, hi = nums.count          // [lo, hi)
    while lo < hi {
        let mid = lo + (hi - lo) / 2
        if nums[mid] < target { lo = mid + 1 } else { hi = mid }
    }
    return lo
}
```
Cả hai: Time O(log n), Space O(1).
</details>

<details><summary>✅ 33. Search in Rotated Sorted Array</summary>

```swift
func searchRotated(_ nums: [Int], _ target: Int) -> Int {
    var lo = 0, hi = nums.count - 1
    while lo <= hi {
        let mid = lo + (hi - lo) / 2
        if nums[mid] == target { return mid }
        if nums[lo] <= nums[mid] {                       // nửa trái đã sắp xếp
            if nums[lo] <= target && target < nums[mid] { hi = mid - 1 } else { lo = mid + 1 }
        } else {                                         // nửa phải đã sắp xếp
            if nums[mid] < target && target <= nums[hi] { lo = mid + 1 } else { hi = mid - 1 }
        }
    }
    return -1
}
// [4,5,6,7,0,1,2], 0 → 4   ·   3 → -1   ·   [3,1], 1 → 1
```
Time O(log n), Space O(1). Bẫy: `nums[lo] <= nums[mid]` phải có dấu `=` (trường hợp `lo == mid`, ví dụ `[3,1]`).
</details>

---

## Tuần 7 (21/12 – 27/12) — Linked List

### Pattern
**Nhận diện:** đề cho `ListNode`. Kỹ thuật lặp đi lặp lại:
1. **Dummy node** — tránh xử lý riêng trường hợp đầu danh sách.
2. **Fast & slow pointer** — tìm giữa, phát hiện vòng, phần tử thứ n từ cuối.
3. **Đảo ngược** — 4 dòng `next / node.next = prev / prev = node / cur = next`.

```swift
let dummy = ListNode(0)
var tail = dummy
// ... tail.next = node; tail = node
return dummy.next

var slow = head, fast = head
while fast?.next != nil { slow = slow?.next; fast = fast?.next?.next }
// slow ở giữa (với list chẵn: giữa-phải)
```
> Liên hệ iOS: `ListNode` là **class** → ARC. Danh sách có vòng (cycle) bằng strong reference = **retain cycle** → leak. Câu chuyện hay để kể khi nói về tuần 5.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) ⭐ | 🟢 |
| 2 | [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) ⭐ | 🟢 |
| 3 | [876. Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list/) | 🟢 |
| 4 | [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | 🟢 |
| 5 | [83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list/) | 🟢 |
| 6 | [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **206:** Iterative 4 bước (dưới). Làm thêm bản **đệ quy**: `let newHead = reverse(head.next); head.next.next = head; head.next = nil`.
- **21:** Dummy + `tail`; cuối cùng nối phần còn lại `tail.next = a ?? b`.
- **876:** Fast/slow; fast đi 2 bước, slow 1 bước.
- **141:** Floyd: nếu có vòng, fast sẽ đuổi kịp slow. So sánh bằng `===`.
- **83:** Nếu `cur.val == cur.next.val` thì `cur.next = cur.next.next`, ngược lại tiến.
- **19:** Dummy; cho `fast` đi trước n+1 bước, rồi cả hai cùng đi tới khi `fast == nil`; `slow.next = slow.next.next`.
</details>

<details><summary>✅ 206. Reverse Linked List</summary>

```swift
func reverseList(_ head: ListNode?) -> ListNode? {
    var prev: ListNode? = nil
    var cur = head
    while let node = cur {
        let next = node.next      // 1. giữ phần còn lại
        node.next = prev          // 2. đảo mũi tên
        prev = node               // 3. tiến prev
        cur = next                // 4. tiến cur
    }
    return prev
}
```
Time O(n), Space O(1). Vẽ 3 ô `1 → 2 → 3` ra giấy và trace — interviewer rất thích thấy bạn vẽ.
</details>

<details><summary>✅ 21. Merge Two Sorted Lists</summary>

```swift
func mergeTwoLists(_ l1: ListNode?, _ l2: ListNode?) -> ListNode? {
    let dummy = ListNode(0)
    var tail = dummy
    var a = l1, b = l2
    while let x = a, let y = b {
        if x.val <= y.val { tail.next = x; a = x.next }
        else { tail.next = y; b = y.next }
        tail = tail.next!
    }
    tail.next = a ?? b
    return dummy.next
}
```
Time O(n + m), Space O(1) (tái sử dụng node có sẵn).
</details>

---

## Tuần 8 (28/12 – 03/01) — Linked List + Recursion

### Pattern: Đệ quy
**Nhận diện:** bài toán chia được thành "bài toán nhỏ hơn cùng dạng". Viết đệ quy theo 3 câu hỏi:
1. **Base case** là gì? (list rỗng / 1 node / n = 0)
2. **Giả sử** hàm đã đúng với phần nhỏ hơn ("leap of faith").
3. Dùng kết quả phần nhỏ để **ghép** ra kết quả hiện tại.

```swift
func solve(_ node: ListNode?) -> Result {
    guard let node else { return baseValue }     // 1
    let sub = solve(node.next)                   // 2
    return combine(node, sub)                    // 3
}
```
**Memoization**: đệ quy có bài con lặp lại (Fibonacci) → lưu kết quả vào dict → tiền đề cho DP tuần 16.
**Chi phí**: mỗi lời gọi tốn 1 stack frame → space O(độ sâu). Xem §1.7.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number/) | 🟢 |
| 2 | [24. Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/) ⭐ | 🟡 |
| 3 | [234. Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/) ⭐ | 🟢 |
| 4 | [160. Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/) | 🟢 |
| 5 | [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers/) | 🟡 |
| 6 | [143. Reorder List](https://leetcode.com/problems/reorder-list/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **509:** 3 phiên bản: đệ quy thuần O(2ⁿ) → memo O(n) → 2 biến O(1) space. Nói được cả 3 = hiểu DP.
- **24:** Base: < 2 node. Đổi 2 node đầu, `first.next = swapPairs(second.next)`.
- **234:** Fast/slow tìm giữa → đảo nửa sau → so sánh từng cặp. O(1) space.
- **160:** Hai con trỏ; con trỏ nào hết list thì nhảy sang đầu list kia → gặp nhau sau tối đa `m + n` bước (`===`).
- **2:** Dummy + biến `carry`; lặp khi `l1 != nil || l2 != nil || carry > 0`.
- **143:** Kết hợp 3 kỹ thuật: tìm giữa + đảo nửa sau + trộn xen kẽ.
</details>

<details><summary>✅ 24. Swap Nodes in Pairs (đệ quy) + 509 Fibonacci (memo)</summary>

```swift
func swapPairs(_ head: ListNode?) -> ListNode? {
    guard let first = head, let second = first.next else { return head }   // base case
    first.next = swapPairs(second.next)    // tin rằng đệ quy xử lý đúng phần còn lại
    second.next = first
    return second
}
// [1,2,3,4] → [2,1,4,3]   [1,2,3] → [2,1,3]

func fib(_ n: Int) -> Int {
    var memo: [Int: Int] = [:]
    func go(_ n: Int) -> Int {
        if n < 2 { return n }
        if let cached = memo[n] { return cached }
        let value = go(n - 1) + go(n - 2)
        memo[n] = value
        return value
    }
    return go(n)
}
```
- `swapPairs`: Time O(n), Space O(n) stack (n/2 tầng). Bản iterative với dummy được O(1) space — nói ra trade-off.
- `fib` memo: Time O(n), Space O(n). Nested function `go` **capture** `memo` — liên hệ closure capture tuần 3.
</details>

<details><summary>✅ 234. Palindrome Linked List</summary>

```swift
func isPalindromeList(_ head: ListNode?) -> Bool {
    // 1. tìm giữa  2. đảo nửa sau  3. so sánh
    var slow = head, fast = head
    while fast?.next != nil { slow = slow?.next; fast = fast?.next?.next }
    var right = reverseList(slow), left = head
    while let r = right, let l = left {
        if r.val != l.val { return false }
        right = r.next; left = l.next
    }
    return true
}
```
Time O(n), Space O(1). (Hàm này làm thay đổi list đầu vào — nói với interviewer; nếu cần, đảo lại sau khi so sánh.)
</details>

---

## Tuần 9 (04/01 – 10/01) — Trees: DFS

### Pattern
**Nhận diện:** đề cho `TreeNode`, hỏi chiều sâu, đường đi, so sánh cây, kiểm tra BST, "từ gốc tới lá".
**DFS đệ quy** = hầu hết bài cây Easy/Medium. Hỏi: "tại 1 node, nếu biết kết quả của cây con trái và phải, mình tính được kết quả của node không?"

```swift
func dfs(_ node: TreeNode?) -> Int {
    guard let node else { return 0 }        // base: cây rỗng
    let l = dfs(node.left)
    let r = dfs(node.right)
    return combine(node.val, l, r)           // post-order: con trước, cha sau
}
// Thứ tự duyệt: pre-order (gốc-trái-phải), in-order (trái-gốc-phải → BST cho ra dãy tăng), post-order (trái-phải-gốc)
```
- Truyền thông tin **từ trên xuống** → thêm tham số (`low`, `high`, `currentSum`).
- Thu thông tin **từ dưới lên** → giá trị trả về.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) ⭐ | 🟢 |
| 2 | [226. Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/) | 🟢 |
| 3 | [100. Same Tree](https://leetcode.com/problems/same-tree/) | 🟢 |
| 4 | [112. Path Sum](https://leetcode.com/problems/path-sum/) | 🟢 |
| 5 | [543. Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | 🟢 |
| 6 | [98. Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) ⭐ | 🟡 |
| + | [235. Lowest Common Ancestor of a BST](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/) (thêm nếu còn thời gian) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **104:** `1 + max(trái, phải)`.
- **226:** Đổi `left`/`right` rồi đệ quy hai bên. Swift: `(root.left, root.right) = (invert(root.right), invert(root.left))`.
- **100:** Cả hai nil → true; một nil → false; so `val` và đệ quy hai bên.
- **112:** Truyền `targetSum - node.val` xuống; tại **lá** kiểm tra bằng 0.
- **543:** Tại mỗi node, đường dài nhất đi qua nó = `depth(left) + depth(right)`. Cập nhật biến `best` bên ngoài, hàm trả về chiều sâu.
- **98:** Truyền khoảng `(low, high)` xuống. **Không** chỉ so node với con trực tiếp. Hoặc in-order phải tăng nghiêm ngặt.
- **235:** Cả p, q nhỏ hơn node → sang trái; cả hai lớn hơn → sang phải; còn lại → node chính là LCA.
</details>

<details><summary>✅ 104. Maximum Depth & 226. Invert</summary>

```swift
func maxDepth(_ root: TreeNode?) -> Int {
    guard let root else { return 0 }
    return 1 + max(maxDepth(root.left), maxDepth(root.right))
}

func invertTree(_ root: TreeNode?) -> TreeNode? {
    guard let root else { return nil }
    (root.left, root.right) = (invertTree(root.right), invertTree(root.left))
    return root
}
```
Time O(n), Space O(h) — h là chiều cao (O(log n) cây cân bằng, O(n) cây lệch).
</details>

<details><summary>✅ 98. Validate BST</summary>

```swift
func isValidBST(_ root: TreeNode?) -> Bool {
    func check(_ node: TreeNode?, _ low: Int?, _ high: Int?) -> Bool {
        guard let node else { return true }
        if let low, node.val <= low { return false }
        if let high, node.val >= high { return false }
        return check(node.left, low, node.val) && check(node.right, node.val, high)
    }
    return check(root, nil, nil)
}
```
- Time O(n), Space O(h).
- Dùng `Int?` làm cận (nil = không giới hạn) thay vì `Int.min/Int.max` → không bị bẫy khi node có giá trị đúng bằng `Int.min`.
- Bẫy kinh điển: cây `5 → (4, 6 → (3, 7))` — node 3 nhỏ hơn cha 6 (hợp lệ cục bộ) nhưng nằm ở cây con phải của 5 → **không** phải BST.
</details>

---

## Tuần 10 (11/01 – 17/01) — Trees: BFS

### Pattern
**Nhận diện:** "theo từng tầng/level", "nhìn từ bên phải", "độ sâu nhỏ nhất", "đường đi ngắn nhất" (cây/đồ thị không trọng số).

```swift
// Template BFS theo tầng — tránh removeFirst() O(n) bằng cách thay cả tầng
var queue: [TreeNode] = [root]
while !queue.isEmpty {
    var next: [TreeNode] = []
    for node in queue {
        // xử lý node
        if let l = node.left { next.append(l) }
        if let r = node.right { next.append(r) }
    }
    queue = next
}
```
DFS vs BFS: DFS dùng stack (đệ quy), bộ nhớ O(h); BFS dùng queue, bộ nhớ O(độ rộng lớn nhất). BFS tìm **ngắn nhất** trước tiên.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [102. Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) ⭐ | 🟡 |
| 2 | [637. Average of Levels in Binary Tree](https://leetcode.com/problems/average-of-levels-in-binary-tree/) | 🟢 |
| 3 | [111. Minimum Depth of Binary Tree](https://leetcode.com/problems/minimum-depth-of-binary-tree/) | 🟢 |
| 4 | [101. Symmetric Tree](https://leetcode.com/problems/symmetric-tree/) | 🟢 |
| 5 | [199. Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view/) ⭐ | 🟡 |
| 6 | [103. Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **102:** Template trên, mỗi tầng 1 mảng.
- **637:** Như 102, lấy trung bình mỗi tầng (`Double`).
- **111:** BFS, gặp **lá đầu tiên** thì trả độ sâu ngay. (DFS cũng được nhưng bẫy: node chỉ có 1 con không phải lá.)
- **101:** BFS với cặp `(left, right)` hoặc DFS `isMirror(a, b)`.
- **199:** Phần tử **cuối** của mỗi tầng.
- **103:** Như 102, tầng lẻ thì `reversed()`.
</details>

<details><summary>✅ 102. Level Order + 199. Right Side View</summary>

```swift
func levelOrder(_ root: TreeNode?) -> [[Int]] {
    guard let root else { return [] }
    var result: [[Int]] = []
    var queue = [root]
    while !queue.isEmpty {
        var level: [Int] = []
        var next: [TreeNode] = []
        for node in queue {                 // duyệt hết 1 tầng
            level.append(node.val)
            if let l = node.left { next.append(l) }
            if let r = node.right { next.append(r) }
        }
        result.append(level)
        queue = next                        // trick: thay cả mảng, không removeFirst
    }
    return result
}

func rightSideView(_ root: TreeNode?) -> [Int] {
    levelOrder(root).compactMap { $0.last }
}
```
Time O(n), Space O(w) với w là độ rộng tầng lớn nhất (≤ n/2).
</details>

<details><summary>✅ 111. Minimum Depth — dùng struct Queue</summary>

```swift
func minDepth(_ root: TreeNode?) -> Int {
    guard let root else { return 0 }
    var q = Queue<(TreeNode, Int)>()
    q.enqueue((root, 1))
    while let (node, depth) = q.dequeue() {
        if node.left == nil && node.right == nil { return depth }   // lá đầu tiên gặp = nông nhất
        if let l = node.left { q.enqueue((l, depth + 1)) }
        if let r = node.right { q.enqueue((r, depth + 1)) }
    }
    return 0
}
```
Time O(n) xấu nhất, nhưng thường dừng sớm — đó là lý do chọn BFS thay DFS.
</details>

---

## Tuần 11 (18/01 – 24/01) — Heap / Priority Queue

### Pattern
**Nhận diện:** "**k** lớn nhất / nhỏ nhất / gần nhất / phổ biến nhất", "luôn lấy phần tử lớn nhất hiện tại", trộn k danh sách đã sắp xếp, median luồng dữ liệu.

**Mẹo đảo ngược:** muốn **k lớn nhất** → dùng **min-heap kích thước k** (đỉnh là phần tử nhỏ nhất trong top-k, gặp số lớn hơn thì đẩy nó ra). O(n log k) thay vì sort O(n log n).

Dùng `struct Heap` ở §1.5 — **tuần này phải tự gõ lại Heap từ trí nhớ ít nhất 2 lần** (T2 và CN).

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [1046. Last Stone Weight](https://leetcode.com/problems/last-stone-weight/) ⭐ | 🟢 |
| 2 | [703. Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/) | 🟢 |
| 3 | [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) ⭐ | 🟡 |
| 4 | [973. K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | 🟡 |
| 5 | [692. Top K Frequent Words](https://leetcode.com/problems/top-k-frequent-words/) | 🟡 |
| 6 | [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) (làm lại bằng heap) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **1046:** Max-heap; pop 2 viên nặng nhất, push hiệu nếu khác 0.
- **703:** Min-heap kích thước k; `add` → push, nếu > k thì pop; trả `peek`.
- **215:** Như 703. Follow-up: Quickselect trung bình O(n) — chỉ cần **nói** ý tưởng.
- **973:** Max-heap theo khoảng cách bình phương (không cần `sqrt`), giữ kích thước k.
- **692:** Đếm → heap với comparator **2 tiêu chí**: tần suất, rồi thứ tự từ điển. Cẩn thận chiều so sánh.
- **347:** Min-heap theo tần suất, kích thước k. So với bucket sort tuần 2.
</details>

<details><summary>✅ 215. Kth Largest + 1046. Last Stone Weight</summary>

```swift
func findKthLargest(_ nums: [Int], _ k: Int) -> Int {
    var minHeap = Heap<Int>(sort: <)     // min-heap kích thước k
    for x in nums {
        minHeap.push(x)
        if minHeap.count > k { _ = minHeap.pop() }
    }
    return minHeap.peek!
}
// [3,2,1,5,6,4], k=2 → 5    [3,2,3,1,2,4,5,5,6], k=4 → 4

func lastStoneWeight(_ stones: [Int]) -> Int {
    var maxHeap = Heap<Int>(sort: >)
    for s in stones { maxHeap.push(s) }
    while maxHeap.count > 1 {
        let y = maxHeap.pop()!, x = maxHeap.pop()!
        if y != x { maxHeap.push(y - x) }
    }
    return maxHeap.peek ?? 0
}
```
- 215: **Time O(n log k), Space O(k)**. Cách "lười" `nums.sorted()[nums.count - k]` O(n log n) — nói ra rồi tối ưu.
- 1046: Time O(n log n), Space O(n).
</details>

<details><summary>✅ 973. K Closest Points — heap với comparator tuỳ biến</summary>

```swift
func kClosest(_ points: [[Int]], _ k: Int) -> [[Int]] {
    func dist(_ p: [Int]) -> Int { p[0] * p[0] + p[1] * p[1] }
    var maxHeap = Heap<[Int]>(sort: { dist($0) > dist($1) })
    for p in points {
        maxHeap.push(p)
        if maxHeap.count > k { _ = maxHeap.pop() }
    }
    var out: [[Int]] = []
    while let p = maxHeap.pop() { out.append(p) }
    return out
}
```
Time O(n log k), Space O(k).
</details>

---

## Tuần 12 (25/01 – 31/01) — Backtracking cơ bản

### Pattern
**Nhận diện:** "liệt kê **tất cả**" tập con / hoán vị / tổ hợp / cách đặt; n nhỏ (≤ 10–20).
Hình dung **cây quyết định**: mỗi tầng là một lựa chọn. Backtracking = DFS trên cây đó, **chọn → đi tiếp → bỏ chọn**.

```swift
var result: [[Int]] = []
var path: [Int] = []
func backtrack(_ start: Int) {
    if /* đủ điều kiện */ true { result.append(path) /* ; return */ }
    for i in start..<nums.count {
        path.append(nums[i])      // chọn
        backtrack(i + 1)          // đi tiếp (i: được dùng lại; i + 1: không)
        path.removeLast()         // bỏ chọn
    }
}
```
- `path` là **Array (value type)** → `result.append(path)` tự copy, không bị "tất cả kết quả giống nhau" như Java/JS. Một câu hay để nói về value semantics!
- Độ phức tạp thường O(2ⁿ · n) (subsets) hoặc O(n! · n) (permutations).

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [78. Subsets](https://leetcode.com/problems/subsets/) ⭐ | 🟡 |
| 2 | [46. Permutations](https://leetcode.com/problems/permutations/) ⭐ | 🟡 |
| 3 | [39. Combination Sum](https://leetcode.com/problems/combination-sum/) | 🟡 |
| 4 | [17. Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) | 🟡 |
| 5 | [22. Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | 🟡 |
| 6 | [79. Word Search](https://leetcode.com/problems/word-search/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **78:** Mỗi node trên cây là 1 subset → append ở **đầu** hàm.
- **46:** Mảng `used[Bool]`; đủ độ dài thì append.
- **39:** Gọi `backtrack(i, remain - c)` (được dùng lại phần tử i); `remain < 0` thì cắt nhánh. Sort trước để `break` sớm.
- **17:** Dict số → chữ; mỗi tầng là 1 chữ số.
- **22:** Chỉ thêm `(` nếu `open < n`; chỉ thêm `)` nếu `close < open`.
- **79:** DFS 4 hướng từ mỗi ô; đánh dấu ô đang đi (`board[r][c] = "#"`) rồi khôi phục khi quay lui.
</details>

<details><summary>✅ 78. Subsets + 46. Permutations</summary>

```swift
func subsets(_ nums: [Int]) -> [[Int]] {
    var result: [[Int]] = []
    var path: [Int] = []
    func backtrack(_ start: Int) {
        result.append(path)                    // mỗi nút trên cây là 1 subset
        for i in start..<nums.count {
            path.append(nums[i])               // chọn
            backtrack(i + 1)                   // đi tiếp
            path.removeLast()                  // bỏ chọn (quay lui)
        }
    }
    backtrack(0)
    return result
}

func permute(_ nums: [Int]) -> [[Int]] {
    var result: [[Int]] = []
    var path: [Int] = []
    var used = [Bool](repeating: false, count: nums.count)
    func backtrack() {
        if path.count == nums.count { result.append(path); return }
        for i in 0..<nums.count where !used[i] {
            used[i] = true; path.append(nums[i])
            backtrack()
            used[i] = false; path.removeLast()
        }
    }
    backtrack()
    return result
}
```
- Subsets: 2ⁿ tập, mỗi tập copy O(n) → **Time O(n · 2ⁿ)**, Space O(n) cho đệ quy (không tính output).
- Permutations: **Time O(n · n!)**, Space O(n).
</details>

<details><summary>✅ 39. Combination Sum</summary>

```swift
func combinationSum(_ candidates: [Int], _ target: Int) -> [[Int]] {
    var result: [[Int]] = []
    var path: [Int] = []
    func backtrack(_ start: Int, _ remain: Int) {
        if remain == 0 { result.append(path); return }
        if remain < 0 { return }
        for i in start..<candidates.count {
            path.append(candidates[i])
            backtrack(i, remain - candidates[i])   // i (không phải i+1): được dùng lại
            path.removeLast()
        }
    }
    backtrack(0, target)
    return result
}
// [2,3,6,7], 7 → [[2,2,3],[7]]
```
`start` đảm bảo không sinh trùng tổ hợp khác thứ tự (`[2,3,2]`).
</details>

---

## Tuần 13 (01/02 – 07/02) — 🧧 Ôn lại (3 bài, redo)

Tết — nhẹ nhàng. **Không làm bài mới.** Làm lại 3 bài kinh điển **từ trang trắng, bấm giờ 20'**, không mở lời giải cũ; viết ít nhất 1 bài trên giấy/Google Docs (không compiler).

| Ngày | Bài | Pattern cần nhớ lại |
|---|---|---|
| T2 01/02 | [1. Two Sum](https://leetcode.com/problems/two-sum/) | Hashing: dict "giá trị → index", kiểm tra phần bù trước khi insert |
| T3 02/02 | [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Stack + dict "đóng → mở", `popLast()` |
| T4 03/02 | [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | 4 bước `next / node.next = prev / prev = node / cur = next` — viết thêm bản đệ quy |

Sau mỗi bài: viết lại **1 dòng pattern** vào sổ mà không nhìn sổ cũ, rồi so sánh. Còn sức → làm thêm 1 bài bạn đánh dấu 🔁 trong bảng tiến độ.

## Tuần 14 (08/02 – 14/02) — 🧧 Ôn lại (3 bài, redo)

Cùng quy tắc như tuần 13:

| Ngày | Bài | Pattern cần nhớ lại |
|---|---|---|
| T3 09/02 | [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Sliding window ngầm: giữ `minPrice` |
| T5 11/02 | [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Sort theo start; sửa `result[result.count - 1]`, không sửa `last` (bản copy) |
| T6 12/02 | [200. Number of Islands](https://leetcode.com/problems/number-of-islands/) | DFS "nhấn chìm" đảo trên lưới, nested function |

> Merge Intervals (tuần 17) và Number of Islands (tuần 15) **chưa tới lượt học** theo lịch chính. Ở tuần 14, coi chúng là bài **xem trước**: đọc pattern ở mục tuần 15/17 của file này 10', rồi tự code 20'. Không ra → đọc lời giải, hiểu, đóng lại, gõ lại — và đánh dấu để làm lại khi tới tuần 15/17.

Thêm (tuỳ chọn): **gõ lại `struct Heap` từ trí nhớ trong 10'**.

---

## Tuần 15 (15/02 – 21/02) — Graph BFS/DFS

### Pattern
**Nhận diện:** lưới 2D (đảo, vùng, lan truyền), quan hệ phụ thuộc (khoá học, build order — giống dependency giữa các module/target!), "số bước ít nhất".

**Biểu diễn đồ thị:**
```swift
// Danh sách kề từ danh sách cạnh
var graph = [[Int]](repeating: [], count: n)
for e in edges { graph[e[0]].append(e[1]) }      // có hướng
var visited = [Bool](repeating: false, count: n)

// Lưới: 4 hướng
let dirs = [(1, 0), (-1, 0), (0, 1), (0, -1)]
for (dr, dc) in dirs {
    let nr = r + dr, nc = c + dc
    guard nr >= 0, nr < rows, nc >= 0, nc < cols else { continue }
}
```
- **DFS** (đệ quy): đếm vùng, đánh dấu vùng, phát hiện chu trình.
- **BFS** (queue): đường ngắn nhất, lan truyền theo "phút" (multi-source BFS).
- **Topological sort (Kahn)**: đếm `indegree`, bắt đầu từ node indegree 0.
- **Luôn đánh dấu visited** khi **đưa vào** queue (không phải khi lấy ra) để tránh thêm trùng.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [733. Flood Fill](https://leetcode.com/problems/flood-fill/) | 🟢 |
| 2 | [200. Number of Islands](https://leetcode.com/problems/number-of-islands/) ⭐ | 🟡 |
| 3 | [695. Max Area of Island](https://leetcode.com/problems/max-area-of-island/) | 🟡 |
| 4 | [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) | 🟡 |
| 5 | [133. Clone Graph](https://leetcode.com/problems/clone-graph/) | 🟡 |
| 6 | [207. Course Schedule](https://leetcode.com/problems/course-schedule/) ⭐ | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **733:** DFS từ ô xuất phát; nếu màu mới == màu cũ thì trả luôn (tránh lặp vô hạn).
- **200:** Duyệt từng ô; gặp `"1"` thì `count += 1` và "nhấn chìm" cả đảo bằng DFS.
- **695:** Như 200, DFS trả về diện tích.
- **994:** **Multi-source BFS**: cho tất cả cam thối vào queue cùng lúc, mỗi tầng = 1 phút. Cuối cùng còn cam tươi → -1.
- **133:** Dict `node gốc → node copy` (dùng `ObjectIdentifier` làm key nếu class không Hashable) + DFS/BFS.
- **207:** Topological sort Kahn; số khoá học lấy ra được < n → có chu trình.
</details>

<details><summary>✅ 200. Number of Islands (DFS)</summary>

```swift
func numIslands(_ grid: [[Character]]) -> Int {
    var grid = grid                                  // copy để đánh dấu
    let rows = grid.count, cols = grid[0].count
    func sink(_ r: Int, _ c: Int) {                  // DFS: "nhấn chìm" cả hòn đảo
        guard r >= 0, r < rows, c >= 0, c < cols, grid[r][c] == "1" else { return }
        grid[r][c] = "0"
        sink(r + 1, c); sink(r - 1, c); sink(r, c + 1); sink(r, c - 1)
    }
    var count = 0
    for r in 0..<rows {
        for c in 0..<cols where grid[r][c] == "1" {
            count += 1
            sink(r, c)
        }
    }
    return count
}
```
- **Time O(m·n)** (mỗi ô thăm tối đa 1 lần), **Space O(m·n)** (copy + stack đệ quy xấu nhất).
- Nói thêm: lưới 10⁶ ô toàn "1" thì đệ quy có thể tràn stack → chuyển sang BFS với `Queue`.
- `for ... where grid[r][c] == "1"` — điều kiện `where` được đánh giá **mỗi vòng**, nên thấy được thay đổi do `sink` gây ra.
</details>

<details><summary>✅ 207. Course Schedule (BFS / Kahn)</summary>

```swift
func canFinish(_ numCourses: Int, _ prerequisites: [[Int]]) -> Bool {
    var graph = [[Int]](repeating: [], count: numCourses)
    var indegree = [Int](repeating: 0, count: numCourses)
    for p in prerequisites {                 // [a, b]: học b trước a → cạnh b → a
        graph[p[1]].append(p[0])
        indegree[p[0]] += 1
    }
    var queue = Queue<Int>()
    for i in 0..<numCourses where indegree[i] == 0 { queue.enqueue(i) }
    var taken = 0
    while let course = queue.dequeue() {
        taken += 1
        for next in graph[course] {
            indegree[next] -= 1
            if indegree[next] == 0 { queue.enqueue(next) }
        }
    }
    return taken == numCourses              // còn sót → có chu trình
}
```
**Time O(V + E), Space O(V + E).** Liên hệ thực tế: Xcode/SPM sắp thứ tự build target theo đúng thuật toán này và báo lỗi "cycle in dependencies".
</details>

---

## Tuần 16 (22/02 – 28/02) — DP 1D

### Pattern
**Nhận diện:** "số **cách**…", "giá trị **lớn nhất/nhỏ nhất**…", lựa chọn ở bước i phụ thuộc kết quả các bước trước, đệ quy có bài con trùng lặp.

**Quy trình 4 bước (nói to khi phỏng vấn):**
1. **Trạng thái:** `dp[i]` nghĩa là gì? (viết ra bằng lời!)
2. **Công thức chuyển:** `dp[i]` tính từ `dp[i-1]`, `dp[i-2]`… thế nào?
3. **Base case:** `dp[0]`, `dp[1]`.
4. **Thứ tự tính + tối ưu bộ nhớ:** chỉ cần 2 giá trị trước → 2 biến, O(1) space.

Đường đi: đệ quy → đệ quy + memo (top-down) → bảng (bottom-up) → 2 biến.

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) ⭐ | 🟢 |
| 2 | [746. Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs/) | 🟢 |
| 3 | [198. House Robber](https://leetcode.com/problems/house-robber/) ⭐ | 🟡 |
| 4 | [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) | 🟡 |
| 5 | [322. Coin Change](https://leetcode.com/problems/coin-change/) ⭐ | 🟡 |
| 6 | [139. Word Break](https://leetcode.com/problems/word-break/) | 🟡 |
| + | [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) (thêm) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **70:** `dp[i] = dp[i-1] + dp[i-2]` — chính là Fibonacci.
- **746:** `dp[i] = cost[i] + min(dp[i-1], dp[i-2])`; đáp án `min(dp[n-1], dp[n-2])`.
- **198:** `dp[i] = max(dp[i-1], dp[i-2] + nums[i])` — bỏ nhà i hoặc cướp nhà i.
- **53:** Kadane: `cur = max(x, cur + x)` — đoạn đang xét có đáng giữ không?
- **322:** `dp[a] = min(dp[a - c] + 1)` với mọi đồng c; khởi tạo "vô cực" = `amount + 1`.
- **139:** `dp[i] = true` nếu tồn tại `j < i` với `dp[j] && s[j..<i] ∈ dict`. Dùng `Set` cho dict, `Array(s)` cho chuỗi.
- **300:** `dp[i] = 1 + max(dp[j])` với `j < i, nums[j] < nums[i]` — O(n²) là đủ cho mức này.
</details>

<details><summary>✅ 70. Climbing Stairs + 198. House Robber</summary>

```swift
func climbStairs(_ n: Int) -> Int {
    if n <= 2 { return n }
    var prev2 = 1, prev1 = 2           // dp[1], dp[2]
    for _ in 3...n { (prev2, prev1) = (prev1, prev1 + prev2) }
    return prev1
}

func rob(_ nums: [Int]) -> Int {
    var prev2 = 0, prev1 = 0           // dp[i-2], dp[i-1]
    for x in nums {
        let cur = max(prev1, prev2 + x)  // bỏ nhà này, hoặc cướp nhà này + dp[i-2]
        prev2 = prev1; prev1 = cur
    }
    return prev1
}
// climbStairs(5) == 8   rob([2,7,9,3,1]) == 12
```
Cả hai: Time O(n), Space O(1). Chú ý guard `n <= 2` trước `3...n` — nếu không, `n = 1` tạo range `3...1` → **crash**.
</details>

<details><summary>✅ 322. Coin Change</summary>

```swift
func coinChange(_ coins: [Int], _ amount: Int) -> Int {
    let inf = amount + 1
    var dp = [Int](repeating: inf, count: amount + 1)   // dp[a] = số xu ít nhất để đạt a
    dp[0] = 0
    if amount == 0 { return 0 }
    for a in 1...amount {
        for c in coins where c <= a {
            dp[a] = min(dp[a], dp[a - c] + 1)
        }
    }
    return dp[amount] == inf ? -1 : dp[amount]
}
// [1,2,5], 11 → 3   [2], 3 → -1   [1], 0 → 0
```
- **Time O(amount × số loại xu), Space O(amount).**
- Vì sao không greedy (lấy xu lớn nhất trước)? Phản ví dụ: `coins = [1,3,4]`, `amount = 6` → greedy `4+1+1` (3 xu), tối ưu `3+3` (2 xu).
- Dùng `amount + 1` thay `Int.max` để `dp[a - c] + 1` không tràn số (Swift sẽ crash).
</details>

<details><summary>✅ 53. Maximum Subarray (Kadane)</summary>

```swift
func maxSubArray(_ nums: [Int]) -> Int {
    var cur = nums[0], best = nums[0]
    for x in nums.dropFirst() {
        cur = max(x, cur + x)
        best = max(best, cur)
    }
    return best
}
```
Time O(n), Space O(1). Đề đảm bảo `nums` không rỗng; nếu không chắc thì `guard let first = nums.first`.
</details>

---

## Tuần 17 (01/03 – 07/03) — Intervals + Greedy

### Pattern
**Intervals — nhận diện:** danh sách `[start, end]`: gộp, chèn, đếm chồng lấn, lịch họp. **Gần như luôn sort theo `start` trước.**
Hai đoạn `a`, `b` (đã sort theo start) **chồng lấn** khi `b.start <= a.end`.

**Greedy — nhận diện:** chọn lựa "tốt nhất tại chỗ" mà không cần quay lại. Khó nhất là **chứng minh** greedy đúng — với mức 2 năm, đưa được **phản ví dụ** hoặc lý luận "đổi chỗ" là đủ.

```swift
let sorted = intervals.sorted { $0[0] < $1[0] }
var result: [[Int]] = []
for iv in sorted {
    if let last = result.last, iv[0] <= last[1] { /* chồng lấn */ }
    else { result.append(iv) }
}
```

### Bài tập
| # | Bài | Mức |
|---|---|---|
| 1 | [455. Assign Cookies](https://leetcode.com/problems/assign-cookies/) | 🟢 |
| 2 | [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/) | 🟡 |
| 3 | [55. Jump Game](https://leetcode.com/problems/jump-game/) ⭐ | 🟡 |
| 4 | [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/) ⭐ | 🟡 |
| 5 | [57. Insert Interval](https://leetcode.com/problems/insert-interval/) | 🟡 |
| 6 | [435. Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | 🟡 |

<details><summary>💡 Hint từng bài</summary>

- **455:** Sort cả hai; two pointers — bánh nhỏ nhất đủ cho đứa trẻ "dễ tính" nhất.
- **122:** Cộng mọi khoản tăng `max(0, p[i] - p[i-1])`.
- **55:** Giữ `farthest` có thể tới; nếu `i > farthest` → không tới được.
- **56:** Template trên; chồng lấn thì `last.end = max(last.end, iv.end)`.
- **57:** 3 pha: thêm các đoạn kết thúc trước `newInterval`; gộp các đoạn chồng lấn; thêm phần còn lại.
- **435:** Sort theo **end**; giữ đoạn kết thúc sớm nhất, đếm số đoạn phải bỏ.
</details>

<details><summary>✅ 56. Merge Intervals</summary>

```swift
func merge(_ intervals: [[Int]]) -> [[Int]] {
    let sorted = intervals.sorted { $0[0] < $1[0] }
    var result: [[Int]] = []
    for iv in sorted {
        if let last = result.last, iv[0] <= last[1] {
            result[result.count - 1][1] = max(last[1], iv[1])   // chồng lấn → nới end
        } else {
            result.append(iv)
        }
    }
    return result
}
// [[1,3],[2,6],[8,10],[15,18]] → [[1,6],[8,10],[15,18]]   [[1,4],[4,5]] → [[1,5]]   [[1,4],[2,3]] → [[1,4]]
```
- **Time O(n log n)** (sort), **Space O(n)**.
- Swift: `result.last` là **bản copy** (value type) → phải sửa qua `result[result.count - 1][1]`, sửa `last` không có tác dụng. Đây là câu hỏi value semantics rất hay.
- Bẫy: `[[1,4],[2,3]]` — quên `max` sẽ ra `[1,3]`.
</details>

<details><summary>✅ 55. Jump Game</summary>

```swift
func canJump(_ nums: [Int]) -> Bool {
    var farthest = 0
    for (i, step) in nums.enumerated() {
        if i > farthest { return false }   // không tới được ô i
        farthest = max(farthest, i + step)
    }
    return true
}
```
Time O(n), Space O(1). Greedy đúng vì: nếu tới được ô i thì tới được mọi ô ≤ i.
</details>

---

## Tuần 18 (08/03 – 14/03) — Mixed, bấm giờ

Không ghi pattern cho từng bài — **tự nhận diện** là một phần bài tập. Mỗi bài: **25' bấm giờ, nói to, viết trên Google Docs/CoderPad** (không compiler), sau đó mới chạy thử.

| # | Bài | Mức | Ghi chú |
|---|---|---|---|
| 1 | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) ⭐ | 🟡 | Rất hay hỏi iOS (liên hệ `NSCache`, image cache) |
| 2 | [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/) | 🟡 | |
| 3 | [75. Sort Colors](https://leetcode.com/problems/sort-colors/) | 🟡 | |
| 4 | [236. Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) | 🟡 | |
| 5 | [62. Unique Paths](https://leetcode.com/problems/unique-paths/) | 🟡 | |
| 6 | [394. Decode String](https://leetcode.com/problems/decode-string/) | 🟡 | |

<details><summary>💡 Pattern từng bài (mở sau khi đã đoán)</summary>

- **146:** Hash map + **doubly linked list** (dummy head/tail). `get`/`put` O(1).
- **560:** Prefix sum + dict đếm `prefix → số lần`; tại mỗi i cộng `count[prefix - k]`. Không dùng sliding window được vì có số âm.
- **75:** Two pointers 3 vùng (Dutch National Flag): `low`, `mid`, `high`.
- **236:** DFS post-order: nếu node là p hoặc q → trả node; nếu trái và phải đều khác nil → node là LCA.
- **62:** DP 2D `dp[r][c] = dp[r-1][c] + dp[r][c-1]` (tối ưu 1 hàng).
- **394:** Stack lưu `(chuỗi trước đó, số lần lặp)` khi gặp `[`.
</details>

<details><summary>✅ 146. LRU Cache — HashMap + Doubly Linked List</summary>

```swift
final class LRUCache {
    private final class Node {
        let key: Int
        var value: Int
        var prev: Node?
        var next: Node?
        init(_ key: Int, _ value: Int) { self.key = key; self.value = value }
    }
    private let capacity: Int
    private var map: [Int: Node] = [:]
    private let head = Node(0, 0)     // dummy: phía "mới dùng nhất"
    private let tail = Node(0, 0)     // dummy: phía "lâu nhất chưa dùng"

    init(_ capacity: Int) {
        self.capacity = capacity
        head.next = tail
        tail.prev = head
    }
    deinit {   // các Node trỏ 2 chiều bằng strong → tự cắt để không leak
        var cur = head.next
        while let n = cur { cur = n.next; n.prev = nil; n.next = nil }
        head.next = nil
    }
    private func remove(_ node: Node) {
        node.prev?.next = node.next
        node.next?.prev = node.prev
    }
    private func insertFront(_ node: Node) {
        node.next = head.next
        node.prev = head
        head.next?.prev = node
        head.next = node
    }
    func get(_ key: Int) -> Int {
        guard let node = map[key] else { return -1 }
        remove(node); insertFront(node)       // vừa dùng → đưa lên đầu
        return node.value
    }
    func put(_ key: Int, _ value: Int) {
        if let node = map[key] {
            node.value = value
            remove(node); insertFront(node)
            return
        }
        let node = Node(key, value)
        map[key] = node
        insertFront(node)
        if map.count > capacity, let lru = tail.prev {
            remove(lru)
            map[lru.key] = nil
        }
    }
}
```
- `get`/`put` **O(1)**, Space O(capacity).
- **Điểm ăn tiền khi nói với interviewer iOS:**
  - `prev`/`next` đều strong → **retain cycle** giữa các node. Hai cách: `weak var prev` (chi phí weak reference nhỏ) hoặc cắt liên kết trong `deinit` như trên. Nói được điều này = nối DSA với ARC (tuần 5).
  - Class không thread-safe — dùng thật trong app thì bọc bằng `actor` hoặc serial queue (tuần 6–7).
  - `NSCache` của Apple **không** đảm bảo LRU chính xác; nó evict theo chính sách riêng + khi bộ nhớ thấp.
</details>

---

## Tuần 19 (15/03 – 21/03) — Mixed, mô phỏng phỏng vấn

Mỗi buổi **chọn ngẫu nhiên 2 bài**, 45' cho cả hai, có người (hoặc AI đóng vai interviewer — chỉ hỏi, không gợi ý code) ngồi nghe bạn nói. Kết thúc mỗi bài tự chấm theo bảng ở cuối file.

| # | Bài | Mức |
|---|---|---|
| 1 | [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | 🟡 |
| 2 | [424. Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | 🟡 |
| 3 | [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) | 🟡 |
| 4 | [230. Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) | 🟡 |
| 5 | [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) | 🟡 |
| 6 | [139. Word Break](https://leetcode.com/problems/word-break/) | 🟡 |

**Bộ "cấp cứu" đêm trước phỏng vấn** (chỉ đọc lại lời giải đã viết, không làm mới): Two Sum · Valid Parentheses · Reverse Linked List · Merge Intervals · Level Order · Number of Islands · Climbing Stairs · LRU Cache.

<details><summary>💡 Hint</summary>

- **230:** In-order DFS đếm tới k (in-order BST cho dãy tăng).
- Các bài còn lại: xem hint ở tuần 2, 4, 7, 15, 16.
</details>

---

## ➕ Bài bổ sung được nhắc trong file tuần

Một số file tuần gợi ý thêm bài ngoài 6 bài chính ở trên. Đây là bài **tuỳ chọn**: làm nếu còn thời gian, không tính vào checklist "6 bài".

| Tuần | Bài bổ sung |
|---|---|
| 5 | [394. Decode String](https://leetcode.com/problems/decode-string/) 🟡 |
| 6 | [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/) 🟡 · [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/) 🟡 |
| 7 | [143. Reorder List](https://leetcode.com/problems/reorder-list/) 🟡 |
| 9 | [236. Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) 🟡 |
| 10 | [116. Populating Next Right Pointers in Each Node](https://leetcode.com/problems/populating-next-right-pointers-in-each-node/) 🟡 |
| 11 | [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) 🔴 |
| 15 | [417. Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow/) 🟡 |
| 16 | [91. Decode Ways](https://leetcode.com/problems/decode-ways/) 🟡 |
| 17 | [134. Gas Station](https://leetcode.com/problems/gas-station/) 🟡 · [253. Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) 🟡 (bài Premium — có thể làm bản miễn phí trên LintCode #919) |

## 3. Bảng tiến độ

Đánh dấu: ✅ tự làm được < 25' · 🟨 phải xem hint · ❌ phải xem lời giải · 🔁 cần làm lại. Cột "Redo" ghi ngày làm lại (+3 ngày).

| Tuần | Pattern | Bài 1 | Bài 2 | Bài 3 | Bài 4 | Bài 5 | Bài 6 | Redo xong? | Ghi chú |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Hashing cơ bản | 217 | 242 | 1 | 1480 | 383 | 169 | | |
| 2 | Hashing | 724 | 49 | 347 | 238 | 36 | 128 | | |
| 3 | Two Pointers | 125 | 283 | 88 | 167 | 15 | 11 | | |
| 4 | Sliding Window | 121 | 643 | 3 | 209 | 424 | 567 | | |
| 5 | Stack | 20 | 232 | 155 | 150 | 739 | 496 | | |
| 6 | Binary Search | 704 | 35 | 278 | 74 | 153 | 33 | | |
| 7 | Linked List | 206 | 21 | 876 | 141 | 83 | 19 | | |
| 8 | LL + Recursion | 509 | 24 | 234 | 160 | 2 | 143 | | |
| 9 | Trees DFS | 104 | 226 | 100 | 112 | 543 | 98 | | |
| 10 | Trees BFS | 102 | 637 | 111 | 101 | 199 | 103 | | |
| 11 | Heap | 1046 | 703 | 215 | 973 | 692 | 347 | | |
| 12 | Backtracking | 78 | 46 | 39 | 17 | 22 | 79 | | |
| 13 | 🧧 Redo | 1 | 20 | 206 | — | — | — | | |
| 14 | 🧧 Redo | 121 | 56 | 200 | — | — | — | | |
| 15 | Graph | 733 | 200 | 695 | 994 | 133 | 207 | | |
| 16 | DP 1D | 70 | 746 | 198 | 53 | 322 | 139 | | |
| 17 | Intervals + Greedy | 455 | 122 | 55 | 56 | 57 | 435 | | |
| 18 | Mixed | 146 | 560 | 75 | 236 | 62 | 394 | | |
| 19 | Mixed (mock) | 238 | 424 | 19 | 230 | 994 | 139 | | |

**Tổng:** ~100 lượt bài (≈ 95 bài khác nhau; vài bài như 347, 238, 139 cố tình lặp lại ở tuần sau để làm theo cách khác hoặc bấm giờ). Hoàn thành ≥ 70% với ✅/🟨 là đủ tự tin cho phỏng vấn mức 2 năm.

### Tự chấm sau mỗi bài (dùng ở tuần 18–19)
| Tiêu chí | 0 | 1 | 2 |
|---|---|---|---|
| Hỏi làm rõ + ví dụ trước khi code | Không | Có, sơ sài | Có, nêu edge case |
| Nêu brute force + độ phức tạp | Không | Một trong hai | Cả hai |
| Nhận ra pattern tối ưu | Không | Sau hint | Tự nhận ra |
| Code chạy đúng | Sai | Đúng sau sửa | Đúng ngay |
| Code sạch (tên biến, không lặp) | Rối | Tạm | Sạch |
| Tự test + trace | Không | Một ví dụ | Ví dụ + edge case |

**≥ 9/12 điểm** = sẵn sàng.

---

## 📖 Tài liệu

- [NeetCode Roadmap](https://neetcode.io/roadmap) — thứ tự pattern gần giống file này, mỗi bài có video giải thích (xem **sau** khi đã tự làm).
- [LeetCode — Explore cards](https://leetcode.com/explore/) (Array, Linked List, Binary Tree, Recursion I).
- [Swift Algorithm Club](https://github.com/kodecocodes/swift-algorithm-club) — cài đặt Heap, Queue, Linked List bằng Swift (đọc để tham khảo cách viết Swift-idiomatic).
- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/).
- Sách: *Cracking the Coding Interview* — chương 1–4 và phần "Interview process" (cách nói khi làm bài).
