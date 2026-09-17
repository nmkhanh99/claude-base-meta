# Workflow Agent Budget

## Activation
- Model Decision (kích hoạt trước mỗi lần gọi Workflow tool, kể cả khi đang ở `/effort ultracode`)

## Rules

### 1. Nguyên tắc gốc: một agent = một nhiệm vụ RIÊNG BIỆT
- Chỉ tách agent khi hai agent đó **đọc dữ liệu khác nhau** hoặc **trả lời câu hỏi khác nhau**.
- **KHÔNG** tách agent chỉ vì có nhiều phần tử trong một danh sách cùng loại (nhiều phát hiện, nhiều file, nhiều dòng cần kiểm).
- **KHÔNG** tách agent chỉ vì muốn nhiều "lăng kính"/"góc nhìn" trên cùng một đối tượng. Nhiều góc nhìn phải nằm trong **một prompt của một agent**, trả về nhiều trường trong cùng schema.

### 2. Chống nổ agent ở pha phản biện/verify — bắt buộc gộp lô
Đây là chỗ nổ agent thường gặp nhất. Anti-pattern **CẤM**:
```js
// ❌ CẤM: 35 phát hiện × 2 lăng kính = 70 agent, mỗi agent chỉ soi 1 phát hiện
pipeline(findings, (f) => parallel(LENSES.map((lens) => () => agent(`...${f}...${lens.ask}`))))
```
Mẫu **BẮT BUỘC** thay thế:
```js
// ✅ 35 phát hiện → 4 agent: mỗi agent nhận 1 LÔ và tự soi ĐỦ các lăng kính
const BATCH = 9
const batches = chunk(findings, BATCH)          // tự viết chunk() 3 dòng
const judged = (await parallel(batches.map((group, i) => () =>
  agent(`${CONTEXT}

BẠN NHẬN ${group.length} PHÁT HIỆN. Với TỪNG phát hiện, soi ĐỦ 2 góc rồi trả verdict riêng:
  (a) ĐƯỜNG CODE: cố bác bỏ bằng cách tự đọc lại code, tìm guard/branch bị bỏ sót.
  (b) RỦI RO VỠ: nếu làm theo, có vỡ test/hợp đồng/luồng đang chạy không.
Mặc định refuted=true nếu KHÔNG tự xác minh được bằng code.

${group.map((f, k) => `[${k}] ${f.title}\n    ${f.detail}\n    Bằng chứng: ${f.evidence}`).join('\n')}`,
    { label: `phản biện:lô-${i + 1}`, phase: 'Phản biện', schema: BATCH_VERDICT_SCHEMA }
  )
))).flatMap((r) => r.verdicts)
```
- Kích thước lô mặc định **8–10 phần tử/agent**. Danh sách càng dài thì lô càng to, **không** tăng số agent.
- Trước khi gộp lô, phải **khử trùng lặp và xếp theo mức nghiêm trọng**; nếu vẫn quá dài thì `.slice()` giữ phần nghiêm trọng nhất và ghi rõ đã bỏ bao nhiêu mục.

### 3. Trần an toàn: 12 agent mỗi workflow
- Tổng agent thực thi của một workflow **không vượt 12**, tính gộp mọi phase. Đây là lưới an toàn, không phải mục tiêu.
- Vượt 12 thì **phải hỏi user trước**, nêu con số và lý do. Không tự nới.
- Phân bổ tham chiếu: ≤6 agent khảo sát (mỗi agent một vùng code khác nhau) + ≤4 agent phản biện theo lô + ≤2 agent tổng hợp/quyết định.

### 4. Đếm agent theo số lần THỰC THI, không theo số dòng `agent(`
- `parallel(arr.map(...))` → số agent = `arr.length`. `pipeline` 2 tầng → `Σ (tầng 1 × tầng 2)`.
- Bẫy đã xảy ra thật: script chỉ có 4 lệnh `agent()` trong code nhưng chạy ra **85 agent**, trong đó 70 agent nằm ở một pha duy nhất.
- Mọi mảng động (findings, file list, kết quả grep) đưa vào `parallel`/`pipeline` phải đi qua `chunk()` hoặc `.slice(0, N)` với hằng số khai báo ở đầu script.

### 5. Báo số agent trước khi gọi Workflow
- Ngay trước lệnh gọi Workflow, nêu một dòng: số agent tối đa và phân bổ theo phase, ví dụ `Dự kiến tối đa 11 agent: 5 khảo sát + 4 phản biện theo lô + 2 tổng hợp`.
- Nếu số agent thực tế vượt dự kiến, phải báo lại cho user sau khi chạy xong.

### 6. Ưu tiên chiều sâu thay vì chiều rộng
- Trước khi thêm agent, hãy tăng phạm vi việc của agent hiện có (gộp nhiều câu hỏi liên quan vào một prompt) — rẻ hơn, ít nhiễu hơn, và tránh nhiều agent đọc lại cùng một file.
- Không tạo agent riêng cho việc mà agent chính có thể làm bằng một lần đọc file hoặc một lần grep.

## References
- Karpathy Guidelines (Simplicity First): .claude/rules/karpathy-guidelines.md
- Subagent chỉ được đọc, không sửa worktree: memory `feedback_subagents_never_mutate_worktree`
- Cấu hình liên quan: `workflowSizeGuideline` trong `~/.claude/settings.json` (small <5 · medium <15 · large <50 · unrestricted)
