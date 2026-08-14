# Phân công đội 3 người — Agent Arena (120 phút)

Tài liệu này chia việc theo đúng bảng giờ ở `README.md` §2. Mục tiêu: cả
**5 layer** trong `harness/layers/` phải chạy được, đúng luật (không sửa
`claim["text"]`, không đọc `Doc.tags`, không try/except trong hook), và
đội có **hai lần kiểm tra tích hợp** trước khi freeze ở phút 95.

Nguyên tắc phân việc: **mỗi người sở hữu file riêng, không ai sửa file của
người khác** → gần như zero merge conflict vì `harness/layers/*.py` là 5
file độc lập. Thứ tự lồng hook đã được `scripts/run_practice.py` wire sẵn
đúng theo `middleware.py`:

```
[injection_guard, critic, citation_checker, budget_policy, retry]
```

Không ai cần sửa thứ tự này — chỉ cần điền đúng thân hàm.

---

## 1. Vai trò — ai giữ file nào

| Người | Layer phụ trách | Hook cần viết | Cỡ ước tính | Vì sao ghép cặp này |
|---|---|---|---|---|
| **A — Grounding chính** | `critic.py` (§2) | `after_agent` | ~10–25 dòng | Điểm nặng nhất bài (honesty 15đ trên MỌI brief + phần lớn precision). Cần người cẩn thận nhất, vì đây cũng là layer dễ vô tình sửa `claim["text"]` nhất (case ghép câu mâu thuẫn). |
| **B — Grounding phụ + Control flow** | `citation_checker.py` (§11) rồi `budget_policy.py` (§3) | `after_agent` · `_spent` + `before_model` + `wrap_tool_call` | ~10–25 + ~10–14 dòng | Cùng nhóm "chỉnh report/claims" như A nhưng độc lập về logic (re-attribute doc_id, không xoá/sửa chữ). Sau khi xong sớm, chuyển sang `budget_policy` — logic đơn giản, nhiều dòng là bookkeeping. |
| **C — Safety & Reliability** | `injection_guard.py` (§10) rồi `retry.py` (§7) | `wrap_tool_call` + `after_agent` · `wrap_tool_call` | ~10–19 + ~8–12 dòng | Cả hai đều là "lớp biên" bọc quanh tool call, cùng một nhịp tư duy (đọc `ToolResult`, không đụng model). `retry` phụ thuộc hiểu `is_degraded` — làm sau `injection_guard` để đã quen cấu trúc `wrap_tool_call`. |

Không có vai trò "chỉ test" — cả 3 người đều code. Nhưng **A** là người
chốt quyết định cuối cùng nếu G (grounding) và S (safety) giằng co nhau ở
bước tích hợp, vì `critic` chạm vào cả hai.

---

## 2. Mốc thời gian

### Phút 0–15 — Orientation (làm cùng nhau, không tách việc)

- [ ] Cả 3 đọc `README.md` §3–§6 (ranh giới `harness/` vs `arena/`, cách
      chấm điểm, 6 hook) — đã đọc xong bản thân file này rồi thì lướt lại.
- [ ] Chạy cùng nhau, mỗi người trên máy mình:
  ```bash
  python3 -m pytest -q
  python3 scripts/run_practice.py --layers none
  python3 scripts/verify.py
  ```
- [ ] Mỗi người đọc kỹ **docstring đầu file** của layer mình phụ trách
      (không đọc lướt — README nói thẳng "nó đã trả lời gần hết câu hỏi").
- [ ] Thống nhất: ai là A/B/C theo bảng trên. Ưu tiên người chắc tay nhất
      giữ `critic` vì đây là layer dễ mất điểm nhất nếu làm sai (§8.1–8.2).
- [ ] Thống nhất quy ước: **commit nhỏ, thường xuyên, `git pull` trước mỗi
      lần push** — vì 5 file độc lập nên conflict gần như không xảy ra,
      nhưng vẫn nên đồng bộ ở mỗi checkpoint dưới đây.

### Phút 15–50 — Vòng build 1 (làm riêng)

Mỗi người điền TODO trong file của mình, theo đúng docstring. Vài điểm
bắt buộc nhớ khi code (rút từ README):

- **A (`critic`)**: chỉ được xoá claim / cắt substring / tách câu ghép
  giữ nguyên hai nửa chữ gốc — **tuyệt đối không thêm/sửa một ký tự nào**
  trong `claim["text"]`. Dùng `ctx.saw(text)` hoặc `text in
  ctx.observed_text` làm tín hiệu duy nhất.
- **B (`citation_checker`)**: so khớp theo **từng dòng** (`splitlines`),
  không phải `in doc.body` cả khối — khác biệt này đã đo được gây sai điểm.
  Chỉ gắn lại `doc_id`, không sửa `text`. Chỉ trỏ vào tài liệu đã thực sự
  quan sát nguyên vẹn (`doc.body in ctx.observed_text`).
- **B (`budget_policy`)**: nhớ cả `before_model` (bơm sentinel) **lẫn**
  `wrap_tool_call` (chặn cứng khi cạn ngân sách) — thiếu một trong hai là
  đã đo được 34/120 lượt chạy vượt ngân sách trong bài gốc.
- **C (`injection_guard`)**: `wrap_tool_call` phải xử lý cả trường hợp
  thiếu dấu mốc đóng (`BLOCK_END`) do bị cắt giữa chừng. `after_agent` chỉ
  sửa `report["answer"]`, không đụng `claims`.
- **C (`retry`)**: nhớ điều kiện dừng theo ngân sách
  (`ctx.tools.calls >= ctx.max_tool_calls - reserve`) ngay trong vòng lặp
  thử lại — `budget_policy` không nhìn thấy các lượt gọi lại bên trong.

Lệnh test riêng từng người, chạy nhiều lần trong lúc code:

```bash
python3 -m pytest tests/test_layers_stubs.py -k critic -q      # (đổi tên layer)
python3 scripts/run_practice.py --layers critic                # đổi tên layer mình
```

### Phút 50–55 — Checkpoint tích hợp #1 (5 phút, cả 3 dừng tay)

```bash
git pull
python3 scripts/run_practice.py --layers all
python3 scripts/leaderboard.py runs/*.json
```

- Layer nào **chưa xong** cứ để nguyên no-op (mặc định không làm gì) —
  README đảm bảo agent vẫn chạy được, không vỡ layer khác.
- So điểm với baseline (`--layers none`, ~24/100) để chắc GAP đang tăng,
  không phải chỉ tổng điểm tăng ngẫu nhiên.
- Ghi nhanh: layer nào đang kéo điểm xuống / có cảnh báo `⚠ Không có
  FINAL đọc được`? Phân công người rảnh tay nhất hỗ trợ ngay.

### Phút 55–80 — Vòng build 2: hoàn thiện + tự test sâu

- A tiếp tục tinh chỉnh `critic`, đặc biệt case brief mâu thuẫn
  (`pub-04-lam-viec-tu-xa`) và brief `is_absent`.
- B hoàn thiện `budget_policy`, quay lại soát `citation_checker` với các
  brief `pub-08`/`pub-09` (đáp án KHÔNG nằm trong top-k truy vấn gốc —
  đây là bài kiểm tra thật theo `phases/README.md`).
- C hoàn thiện `retry`, kiểm tra bằng leave-one-out (retry một mình có
  thể KHÔNG cho thấy điểm tăng rõ — đó là bình thường, xem README §10).
- Mỗi người tự chạy `python3 scripts/selfeval.py --brief <tên brief mình
  đang gỡ>` để đọc chẩn đoán chi tiết thay vì đoán mò từ điểm tổng.

```bash
python3 scripts/run_practice.py                      # full 9 brief, 5 layer
python3 scripts/selfeval.py                           # đọc "SỬA GÌ TIẾP THEO"
```

### Phút 80–90 — Checkpoint tích hợp #2 (10 phút, cả 3 cùng ngồi)

```bash
git pull
python3 scripts/run_practice.py --layers all --entry final-check --out runs/final-check.json
python3 scripts/selfeval.py --run runs/final-check.json --summary
```

- Chạy **leave-one-out** cho từng layer để xác nhận cả 5 layer đều thật
  sự có tác dụng (đúng khuyến nghị ở README §10):
  ```bash
  python3 scripts/run_practice.py --layers injection_guard,citation_checker,budget_policy,retry
  python3 scripts/run_practice.py --layers injection_guard,critic,budget_policy,retry
  python3 scripts/run_practice.py --layers injection_guard,critic,citation_checker,retry
  python3 scripts/run_practice.py --layers injection_guard,critic,citation_checker,budget_policy
  python3 scripts/run_practice.py --layers critic,citation_checker,budget_policy,retry
  ```
  Rút layer nào ra mà điểm không tụt (hoặc tụt rất ít, trừ `retry` — xem
  ghi chú phương sai ở README §10) thì quay lại sửa ngay trong 10 phút
  còn lại của khối build.
- Kiểm tra `gate_passed` trong `runs/final-check.json` — phải luôn `true`.
- Kiểm tra không ai vô tình động vào `arena/`:
  ```bash
  python3 scripts/verify.py
  git status arena/   # phải sạch, không có gì để commit
  ```

### Phút 90–95 — Polish cuối + chuẩn bị freeze

- Chạy lại toàn bộ test suite: `python3 -m pytest -q` — phải xanh hết.
- Xoá code debug/print thừa trong `ctx.state` nếu có (không bắt buộc,
  nhưng tránh nhiễu khi giảng viên đọc code).
- `git add -A && git status` — review kỹ danh sách file thay đổi, đảm bảo
  **chỉ** có `harness/` (và có thể `runs/` nếu muốn nộp kèm, không bắt
  buộc). Tuyệt đối không có gì trong `arena/` hay `data/`.

### Phút 95–105 — FREEZE & SUBMIT

- **Dừng sửa `harness/` ngay lập tức**, kể cả đang debug dở.
- Một người (đề xuất: A, vì ít bị vướng thao tác cuối nhất) chạy:
  ```bash
  git add -A
  git commit -m "Agent Arena — <tên đội>"
  git push
  ```
- Hai người còn lại xác nhận: `git log -1`, `git status` sạch, remote đã
  nhận commit (`git log origin/main -1` hoặc xem trên host git).

### Phút 105–120 — Vòng tính điểm

- Không sửa gì nữa. Ngồi xem. Đây là lúc tốt để cả 3 cùng đọc lại
  `arena/scorer.py` hoặc bàn xem nếu có thêm lab tương tự thì sẽ làm khác
  đi chỗ nào.

---

## 3. Rủi ro chung cần cả 3 người nhớ (không thuộc về riêng ai)

- **Không ai sửa `MAX_STEPS`, `arena.model.parse_output`, hay bọc
  `try/except` quanh code trong hook** — cả ba đều đã bị README cấm rõ
  ràng và mỗi cái đều có hậu quả đo được (điểm 0 hoặc report giả).
- **Không ai đọc `Doc.tags`** để nhận diện tài liệu bẫy — nhãn luôn rỗng
  qua `ctx.corpus`, kể cả lúc luyện tập. Chỉ file JSON trên đĩa mới còn
  nhãn, và đó là bẫy hard-code, không phải lối tắt hợp lệ.
- **Không ai sửa file trong `arena/`**, kể cả để debug tạm — vòng tính
  điểm chạy với `arena/` đã băm hash, sửa một dòng là huỷ bài.
- Nếu một hook raise exception, **cả lượt chạy chết và điểm về 0** — khi
  gỡ lỗi, ưu tiên đọc traceback thật thay vì đoán, vì không có gì nuốt
  lỗi giúp bạn.

---

## 4. Bảng tóm tắt phân công (dán lên bảng/note chung)

| Ai | Việc | Deadline mềm |
|---|---|---|
| A | `harness/layers/critic.py` | Bản chạy được: phút 50; hoàn thiện: phút 80 |
| B | `harness/layers/citation_checker.py` → `budget_policy.py` | citation_checker: phút 50; budget_policy: phút 80 |
| C | `harness/layers/injection_guard.py` → `retry.py` | injection_guard: phút 50; retry: phút 80 |
| Cả 3 | Checkpoint tích hợp | phút 50–55 và phút 80–90 |
| A (đại diện) | `git commit && git push` | phút 95–105 |
