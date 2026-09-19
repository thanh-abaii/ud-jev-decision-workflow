# ud-jev-decision-workflow

> **Kiến trúc phân cấp quyết định & Quản trị luồng tự động hóa với TypeSafe / Jev (System One), Reasoning LLM và Con người.**

`ud-jev-decision-workflow` là một **Agent Skill** chuẩn hóa (tương thích hoàn toàn với công cụ CLI `npx skills` từ `vercel-labs/skills`), được thiết kế để định hướng cho các AI Coding Assistant (Codex, Antigravity, Claude Code, Cline, Gemini CLI...) cách xây dựng các hệ thống tự động hóa tác nghiệp an toàn, hiệu quả và tối ưu chi phí.

Skill này **ngồi một tầng kiến trúc bên trên** skill chính thức `typesafe-ai`. Kỹ năng không thay thế hay sao chép `typesafe-ai`, mà bổ sung quy chế phân định thẩm quyền, cơ chế định tuyến đa provider (Native TypeSafe vs OpenRouter) và ma trận kiểm soát rủi ro.

---

## 1. Triết lý cốt lõi (Core Philosophy)

> **"Code owns the workflow. AI supplies bounded semantic judgment where deterministic code is insufficient."**
> *(Code làm chủ quy trình tác nghiệp. AI chỉ cung cấp phán đoán ngữ nghĩa đóng khung khi code tất định không thể tự giải quyết.)*

Ứng dụng / Workflow engine (Node.js, Python backend, Temporal worker, n8n pipeline) phải luôn là nơi nắm giữ:
- Quản lý trạng thái (state) & quyền hạn (permissions).
- Các luật nghiệp vụ tất định và hạn mức (thresholds).
- Quyền thực thi các hành động không thể đảo ngược (irreversible actions).
- Nhật ký kiểm toán (audit trail) và chính sách leo thang (escalation policy).

Mô hình AI chỉ đóng vai trò là một khối phán đoán ngữ nghĩa trong chuỗi quy trình, không phải là "bộ não trung tâm" tự do quyết định tất cả.

---

## 2. Phễu 4 tầng quyết định (The 4-Tier Hierarchy)

Khi thiết kế bất kỳ tác vụ nào, hệ thống phải duyệt qua thứ tự ưu tiên từ tất định đến mở rộng:

```text
Tác vụ / Sự kiện đầu vào
  │
  ▼
Code tất định hoặc Business Rules có giải quyết được không?
  ├── [Có] ──► CHẾ ĐỘ A: DETERMINISTIC CODE / RULES
  │            (Regex, database lookup, schema validation, tính toán số học, RBAC)
  └── [Không]
        │
        ▼
Có phải là một phán đoán ngữ nghĩa đóng khung, nguyên tử không?
  ├── [Có] ──► CHẾ ĐỘ B: JEV / SYSTEM ONE
  │            (Choice, Noul, Score)
  │                   │
  │                   ▼
  │            Đánh giá: Bất định (Uncertainty) + Hệ quả (Consequence)
  │                   ├── Hệ quả thấp + Bằng chứng rõ ràng ──► TỰ ĐỘNG THỰC THI (Code)
  │                   ├── Bằng chứng mơ hồ / Không chắc chắn  ──► CHẾ ĐỘ C: REASONING LLM
  │                   └── Hệ quả cao / Rủi ro nghiệp vụ lớn ──► CHẾ ĐỘ D: HUMAN REVIEW
  │
  └── [Không]
        │
        ▼
Đòi hỏi tổng hợp đa văn bản, lập luận sâu, giải thích hoặc sinh ngôn ngữ tự nhiên?
        └───► CHẾ ĐỘ C: REASONING / FRONTIER LLM
```

---

## 3. Mối quan hệ với Official `typesafe-ai` Skill

```text
typesafe-ai (Official)
└── Ngữ nghĩa TypeSafe/Jev & Cú pháp SDK Native
    ├── Primitives: Choice, Noul, Score
    ├── Cấu trúc State, Instructions, Criteria
    └── Hợp đồng SDK Python / TypeScript và Live Docs

ud-jev-decision-workflow (Skill này)
└── Kiến trúc thượng tầng + Phân tầng quyết định + Quản trị rủi ro
    ├── Khi nào dùng Code vs Jev vs LLM vs Human Review
    ├── Abstraction định tuyến Provider: Native TypeSafe vs OpenRouter
    └── Ma trận thẩm quyền: typed output != truth & confidence != permission
```

Hai kỹ năng bổ trợ hoàn hảo cho nhau. Official `typesafe-ai` skill là nguồn chân lý duy nhất cho các primitives của System One.

---

## 4. Hướng dẫn cài đặt (Installation)

Yêu cầu môi trường: Node.js (hỗ trợ `npx`).

### A. Cài đặt toàn cục qua GitHub (Khuyến nghị cho mọi người dùng)

Cài đặt trực tiếp từ kho lưu trữ GitHub chính thức để mọi dự án và phiên làm việc của agent trên máy đều tự động nhận diện skill:

```powershell
# Dành riêng cho Codex
npx skills add thanh-abaii/ud-jev-decision-workflow --skill ud-jev-decision-workflow -a codex -g -y

# Dành cho tất cả các Agent trên máy (Codex, Antigravity, Claude Code, Cline...)
npx skills add thanh-abaii/ud-jev-decision-workflow --skill ud-jev-decision-workflow -a '*' -g -y
```

### B. Cài đặt theo từng dự án (Project-level)

Đứng từ thư mục dự án của bạn (consumer project) và chạy:

```powershell
npx skills add thanh-abaii/ud-jev-decision-workflow --skill ud-jev-decision-workflow -a codex -y
```
Skill sẽ được nạp vào thư mục `<your-project>/.agents/skills/ud-jev-decision-workflow`.

### C. Cài đặt từ mã nguồn cục bộ (Dành cho nhà phát triển)

Nếu bạn đang phát triển hoặc kiểm thử trực tiếp từ thư mục mã nguồn trên máy:

```powershell
npx skills add "d:\Scripts\ud-jev-decision-workflow" --skill ud-jev-decision-workflow -a codex -g -y
```

---

## 5. Cơ chế cập nhật (Update Workflow)

- **Khi cài từ Git/GitHub:**
  ```powershell
  # Cập nhật bản toàn cục
  npx skills update ud-jev-decision-workflow -g -y
  ```
- **Khi phát triển cục bộ:**
  Chỉ cần chạy lại lệnh cài đặt với cờ `-y`, `npx skills` sẽ tự động ghi đè nội dung mới nhất:
  ```powershell
  npx skills add "d:\Scripts\ud-jev-decision-workflow" --skill ud-jev-decision-workflow -a codex -g -y
  ```
- **Phát triển liên tục bằng Symbolic Link (Windows):**
  ```powershell
  New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.agents\skills\ud-jev-decision-workflow" -Target "d:\Scripts\ud-jev-decision-workflow" -Force
  ```

---

## 6. Định tuyến Provider: Native TypeSafe vs. OpenRouter

Kỹ năng cung cấp mô hình trừu tượng hóa gọn nhẹ, tách biệt hoàn toàn nghiệp vụ khỏi transport layer:

```typescript
// Interface gọn nhẹ:
judge(state, questions, provider?)
```

| Tiêu chí | Native TypeSafe | OpenRouter |
| :--- | :--- | :--- |
| **Mục đích** | Truy cập trực tiếp, độ trễ tối thiểu, hỗ trợ đầy đủ SDK | Tích hợp tiện lợi khi gom billing qua OpenRouter |
| **Biến môi trường** | `TYPESAFE_API_KEY` | `OPENROUTER_API_KEY` |
| **Model ID** | `jev-latest` | `typesafe/jev-latest` (hoặc `typesafe/jev-1.13`) |
| **Endpoint** | `POST https://api.typesafe.ai/v1/systemone` | `POST https://openrouter.ai/api/alpha/decisions` |
| **Payload Envelope** | `{ state, model: "jev-latest", questions: {...} }` | `{ model: "typesafe/jev-latest", state, questions }` |

> [!CAUTION]
> **Cấm trỏ chéo:** Tuyệt đối không trỏ TypeSafe Native SDK trực tiếp vào Base URL của OpenRouter vì hai bên sử dụng giao thức endpoint và cấu trúc phong bì khác nhau. Luôn sử dụng adapter pattern trong `references/provider-routing.md`.

---

## 7. Quy tắc quản trị & leo thang (Governance Axioms)

```text
typed output != truth
confidence != permission to act
```

1. **Tính đúng cú pháp không đồng nghĩa với chân lý:** Một kết quả `Choice` hợp lệ chỉ chứng minh schema khớp, không chứng minh nhận định thực tế là chính xác.
2. **Confidence không phải giấy phép hành động:** Điểm tin cậy `0.98` có thể đủ để tự động gắn nhãn một email thông thường, nhưng **không bao giờ** được phép tự động khóa tài khoản ngân hàng, xóa cơ sở dữ liệu hay từ chối giao dịch giá trị lớn.
3. **Hiệu chuẩn theo chi phí sai số:** Không dùng các con số cố định như `0.80`, `0.90` một cách vô căn cứ. Luôn hiệu chuẩn dựa trên chi phí của Dương tính giả ($C_{FP}$) so với Âm tính giả ($C_{FN}$).

---

## 8. Cấu trúc thư mục

```text
ud-jev-decision-workflow/
├── README.md                           # Hướng dẫn tổng quan & thao tác CLI
├── SKILL.md                            # Tệp chỉ dẫn trung tâm cho Agent
├── references/
│   ├── decision-architecture.md        # Phân tích 4 tầng quyết định & workflow pattern
│   ├── provider-routing.md             # Đặc tả Native & OpenRouter transport, adapter code
│   ├── governance-and-escalation.md    # Ma trận Uncertainty vs Consequence, 10 nguyên tắc
│   └── patterns.md                     # Speculative Fan-out, Cascades & 5 Anti-patterns
└── examples/
    ├── support-ticket-routing.md       # Ví dụ thực tế: Triage ticket IT qua n8n/Node.js
    └── fraud-triage.md                 # Ví dụ thực tế: Sàng lọc gian lận & guardrails
```

---

## 9. An toàn & Bảo mật (Security)

- **Zero Secrets:** Không bao giờ commit, log hoặc in các API key (`TYPESAFE_API_KEY`, `OPENROUTER_API_KEY`).
- **Server-Side Execution:** Các lời gọi đánh giá ngữ nghĩa với Jev phải luôn chạy ở tầng backend hoặc background worker, không bao giờ để lộ key ở client-side / browser.
- **Audit Trails:** Lưu trữ state đầu vào, câu hỏi, điểm xác suất và quyết định nghiệp vụ phục vụ việc giải trình và truy vết lỗi.

---

## 10. Giấy phép (License)

Phát hành theo giấy phép [MIT](LICENSE).
