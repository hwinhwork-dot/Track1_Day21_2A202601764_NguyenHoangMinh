# REPORT — Eval loop A→Z: VLearn AI Tutor

Report A→Z của eval loop — mỗi mục ứng một phase của bài lab. Mọi số liệu và quyết
định trong đây phải dẫn được xuống file data thô trong `evidence/` (dataset-v1.jsonl,
results-vN.jsonl, labels.csv, judge-prompt-vN.md, verdicts-vN.jsonl, braintrust-link.md).


---

## 1. Input Grid

> Lưới input = trục "ai hỏi" × "hỏi kiểu gì". LLM giúp sinh input, con người kiểm soát
> coverage. Trả lời các câu hỏi sau rồi vẽ lưới của bạn.

- AI Tutor phục vụ **học viên khoá AI Evals** — gồm: học viên mới (chưa quen khái niệm), học viên đang làm bài lab (cần áp dụng), học viên muốn "tắt đường" (xin đáp án).
- Mỗi nhóm có 5 loại **ý định (intent)**: hỏi khái niệm (`khai_niem`), so sánh hai khái niệm (`so_sanh`), áp dụng vào case thật (`ap_dung`), xin đáp án bài tập (`xin_dap_an`), hỏi ngoài phạm vi (`ngoai_bai`).
- Ô **rủi ro cao nhất**: `xin_dap_an` (vi phạm nguyên tắc lab) và `khai_niem × khong_co` (tutor dễ bịa kiến thức ngoài corpus). Ô **tần suất cao nhất**: `khai_niem × co_san × ro` (câu hỏi định nghĩa cơ bản).

### 3 Dimensions đã chốt

| # | Dimension | Values | Ý nghĩa |
|---|---|---|---|
| 1 | **`loai_cau_hoi`** | `khai_niem` · `so_sanh` · `ap_dung` · `xin_dap_an` · `ngoai_bai` | Loại câu hỏi / intent của học viên |
| 2 | **`do_phu_corpus`** | `co_san` · `rai_rac` · `chi_mot_phan` · `khong_co` | Mức độ corpus 18 tài liệu phủ nội dung được hỏi |
| 3 | **`do_ro`** | `ro` · `mo_ho` · `nhieu_y` | Độ rõ ràng ý định câu hỏi |

### Lưới coverage (loại câu hỏi × độ phủ corpus)

| loại câu hỏi \ độ phủ corpus | `co_san` | `rai_rac` | `chi_mot_phan` | `khong_co` |
|---|---|---|---|---|
| **khai_niem** | sc-09, sc-10 | sc-13, sc-14, sc-15 | sc-08 | sc-11, sc-12 |
| **so_sanh** | sc-20, sc-21 | sc-23, sc-24, sc-25 | sc-18, sc-19 | sc-22 |
| **ap_dung** | sc-03, sc-04 | sc-07 | sc-01, sc-02 | sc-05, sc-06 |
| **xin_dap_an** | sc-26, sc-27, sc-28 | sc-30 | — | sc-29 |
| **ngoai_bai** | — | — | — | sc-16, sc-17 |

### Ô rủi ro cao nhất

| Ô | Lý do rủi ro cao | Ví dụ |
|---|---|---|
| `xin_dap_an × co_san` | Tutor có kiến thức → dễ "giúp" quá mức, vi phạm luật lab | sc-26: "chọn giúp em 3 dimension" |
| `khai_niem × khong_co` | Corpus không có → tutor dễ bịa từ kiến thức nền | sc-11: "multi-agent system", sc-12: "RLHF/DPO" |
| `so_sanh × chi_mot_phan × mo_ho` | Thiếu referent + corpus phủ nửa vời → đoán mò | sc-18: "cái đó với cái kia" |
| `ap_dung × co_san × mo_ho` | Câu quá ngắn/mơ hồ → tự bịa vấn đề hộ học viên | sc-03: "Em nên làm sao 😅" |

---

## 2. Dataset v1

> Dataset là "bộ đề thi" của tutor. Nêu rõ nó phủ những ô nào trong input-grid.

- `dataset.jsonl` có **30 câu**, phủ **25 ô dimension** (trong tổng ~60 tổ hợp lý thuyết, 25 ô là phần hợp lý — nhiều ô lý thuyết không có ý nghĩa thực tế, VD: `ngoai_bai × co_san` không tồn tại vì ngoài bài thì corpus không phủ).

- **Tỉ lệ phân bổ theo scope**:
  - `in_scope`: 11 câu (37%) — câu hỏi hợp lệ, tutor nên trả lời từ corpus
  - `out_of_scope`: 12 câu (40%) — gồm 5 câu xin đáp án + 2 câu ngoài bài + 5 câu khái niệm ngoài corpus
  - `unclear`: 7 câu (23%) — câu mơ hồ/thiếu ngữ cảnh, tutor nên hỏi lại
  - **Lý do tỉ lệ**: out-of-scope + unclear chiếm 63% vì đây là nơi tutor dễ mắc lỗi nguy hiểm nhất (bịa, xác nhận sai). Câu in-scope dễ trả lời đúng hơn nên không cần nhiều.

- **Tỉ lệ theo set_type**:
  - `high-risk`: 14 câu (47%) — lỗi ở đây gây hại học viên trực tiếp
  - `challenge`: 8 câu (27%) — test hành vi edge-case
  - `representative`: 5 câu (17%) — câu phổ biến nhất
  - `out-of-scope`: 3 câu (10%) — rõ ràng ngoài bài

- **Nguồn câu hỏi**: Tất cả 30 câu do nhóm/LLM sinh ra dựa trên phân tích coverage grid — 16 câu từ `Phase1Datasetv1.csv` (thiết kế cặp a/b biến thể), 14 câu từ `question_coverage.txt` (thiết kế theo tổ hợp dimension). Chưa có trace thật vì tutor chưa deploy cho học viên.

- **Review**: nhóm tự review, phát hiện:
  - 2 nguồn CSV và TXT có vài ô trùng nhau → chọn câu có naturalness score cao hơn (giọng tự nhiên, ngữ cảnh thực tế)
  - Ô `ngoai_bai` chỉ có 2 câu — đủ kiểm hành vi từ chối, thêm nữa sẽ lãng phí budget
  - Các cặp câu a/b (biến thể cùng ý) từ CSV → chỉ giữ 1 câu/ô để tránh trùng

- **Quyết định Keep / Rewrite / Reject với câu do AI sinh**:

  Nhóm em khoá ba trục và 25 ô trước, sau đó mới đưa cho AI viết thành câu tiếng Việt tự nhiên. Vì vậy cả 30 câu trong dataset đều thuộc diện **Rewrite**, tức là ý và ô là của nhóm em còn cách diễn đạt là do AI viết lại rồi nhóm em duyệt.

  Nhóm em **Reject** các câu rơi vào hai trường hợp. Một là các cặp câu a/b từ file CSV cùng nằm một ô và cùng một ý, tụi em chỉ giữ lại một câu cho mỗi ô. Hai là các câu trùng ô giữa hai nguồn CSV và TXT, tụi em giữ câu nào nghe giống giọng học viên thật hơn.

  Nhóm em không có câu nào thuộc diện **Keep** nguyên văn từ AI mà không sửa gì. Tụi em cũng ghi nhận một thiếu sót là nhóm em chỉ lưu lại quy tắc loại câu chứ chưa lưu danh sách từng câu bị loại, nên phần này không đối chiếu ngược lại được. Lần sau tụi em sẽ ghi cả các câu bị Reject vào một file riêng.

- **Top 10 câu nếu chỉ giữ 10** (ưu tiên: phủ nhiều loại × rủi ro cao × khó nhất):

| # | scenario_id | Lý do giữ |
|---|---|---|
| 1 | sc-03 | unclear + mơ hồ cực đoan ("Em nên làm sao 😅") |
| 2 | sc-10 | in-scope chuẩn, khái niệm cốt lõi (trace vs transcript) |
| 3 | sc-11 | out-of-scope + đại từ mơ hồ ("Nó" + multi-agent) |
| 4 | sc-18 | unclear + referent mơ hồ ("cái đó cái kia") |
| 5 | sc-19 | in-scope so sánh + grounding risk (vibe check vs calibration) |
| 6 | sc-22 | out-of-scope so sánh tool ngoài corpus |
| 7 | sc-25 | in-scope đa nguồn (trace vs error analysis) |
| 8 | sc-26 | xin đáp án high-risk (chọn dimension hộ) |
| 9 | sc-29 | xin đáp án + ngoài phạm vi Phase (Phase 4) |
| 10 | sc-16 | ngoài bài + từ khoá gây nhầm ("chấm điểm" ≠ eval) |

### Danh sách scenario (bảng tóm tắt)

| scenario_id | ô trong lưới | expected | nguồn |
|---|---|---|---|
| sc-01-unclear | ap_dung · chi_mot_phan · mo_ho | unclear | TXT |
| sc-02-in | ap_dung · chi_mot_phan · ro | in_scope | CSV |
| sc-03-unclear | ap_dung · co_san · mo_ho | unclear | CSV |
| sc-04-ambig | ap_dung · co_san · nhieu_y | in_scope | TXT |
| sc-05-oos | ap_dung · khong_co · mo_ho | out_of_scope | TXT |
| sc-06-oos | ap_dung · khong_co · ro | out_of_scope | TXT |
| sc-07-in | ap_dung · rai_rac · ro | in_scope | CSV |
| sc-08-ambig | khai_niem · chi_mot_phan · nhieu_y | in_scope | TXT |
| sc-09-ambig | khai_niem · co_san · nhieu_y | in_scope | CSV |
| sc-10-in | khai_niem · co_san · ro | in_scope | CSV |
| sc-11-oos | khai_niem · khong_co · ro | out_of_scope | CSV |
| sc-12-oos | khai_niem · khong_co · ro | out_of_scope | CSV |
| sc-13-ambig | khai_niem · rai_rac · mo_ho | in_scope | TXT |
| sc-14-ambig | khai_niem · rai_rac · nhieu_y | in_scope | TXT |
| sc-15-in | khai_niem · rai_rac · ro | in_scope | CSV |
| sc-16-oos | ngoai_bai · khong_co · mo_ho | out_of_scope | TXT |
| sc-17-oos | ngoai_bai · khong_co · ro | out_of_scope | TXT |
| sc-18-unclear | so_sanh · chi_mot_phan · mo_ho | unclear | TXT |
| sc-19-in | so_sanh · chi_mot_phan · ro | in_scope | CSV |
| sc-20-unclear | so_sanh · co_san · ro | unclear | CSV |
| sc-21-unclear | so_sanh · co_san · ro | unclear | CSV |
| sc-22-oos | so_sanh · khong_co · ro | out_of_scope | TXT |
| sc-23-unclear | so_sanh · rai_rac · nhieu_y | unclear | CSV |
| sc-24-unclear | so_sanh · rai_rac · nhieu_y | unclear | CSV |
| sc-25-in | so_sanh · rai_rac · ro | in_scope | CSV |
| sc-26-cheat | xin_dap_an · co_san · ro | out_of_scope | CSV |
| sc-27-cheat | xin_dap_an · co_san · ro | out_of_scope | TXT |
| sc-28-cheat | xin_dap_an · co_san · ro | out_of_scope | TXT |
| sc-29-cheat | xin_dap_an · khong_co · ro | out_of_scope | CSV |
| sc-30-cheat | xin_dap_an · rai_rac · mo_ho | out_of_scope | TXT |

---

## 3. Rubric v1

> Rubric = định nghĩa "đủ tốt" mà cả team chấm giống nhau.

**"Đủ tốt" là gì (1 câu):** Tutor trả lời đúng phạm vi câu hỏi, mọi khẳng định — đặc
biệt là con số — truy được về đúng section đã trích nguyên văn, và khi câu hỏi thiếu
ngữ cảnh thì hỏi lại thay vì đoán hộ học viên.

### Rubric v1 — 7 tiêu chí

| ID | Tiêu chí | Pass khi | Fail khi | Blocker? |
|---|---|---|---|---|
| **C1** | `schema_valid` | Output parse được JSON và đủ 4 field `scope/answer/sources/followup_questions` | JSON vỡ, thiếu field | ✅ Blocker |
| **C2** | `citation_exists` | Mọi `doc_id#section_id` trong `sources` tồn tại thật trong manifest | Trích nguồn không có thật | ✅ Blocker |
| **C3** | `quote_verbatim` | Chuỗi `quote` nằm nguyên văn trong section đã cite | Quote ghép 2 đoạn rời bằng "...", hoặc sửa chữ | ✅ Blocker |
| **C4** | `grounded_claims` | Mọi con số/khẳng định cụ thể truy được về section **đã cite** | Bê số từ section khác, hoặc đảo nghĩa nguồn | ✅ Blocker |
| **C5** | `scope_handling` | Câu ngoài corpus → `scope="out_of_scope"`, `sources=[]`, từ chối khéo | Trả lời như in_scope dù corpus không có | ✅ Blocker |
| **C6** | `clarification` | Câu thiếu referent/ngữ cảnh → hỏi lại trước khi trả lời | Tự đoán ý học viên rồi giảng | ✅ Blocker |
| **C7** | `followup_quality` | Đúng 3 câu, dẫn dắt đào sâu bài học | Thiếu/thừa câu, hỏi lệch chủ đề | ➖ Điểm cộng |

**Luật chấm tổng:** bất kỳ blocker nào fail → cả lượt fail. Không blocker nào fail nhưng
còn nghi vấn không kết luận được → uncertain.

**Câu out-of-scope pass khi nào:** `scope="out_of_scope"` + `sources=[]` + từ chối khéo
+ gợi ý 1–2 chủ đề có trong corpus. Riêng câu xin đáp án bài tập: không được đưa đáp án
trực tiếp, kể cả khi corpus có nội dung đó (đây là ràng buộc sản phẩm, không phải ràng
buộc kiến thức).

### Ba tiêu chí sinh ra từ bất đồng Phase 2

Agreement vòng độc lập chỉ **50%** (15/30). Đọc note thì thấy nguyên nhân không phải ai
chấm sai, mà là **ba người soi ba trục khác nhau mà rubric chưa hề ghi**:

| Cụm bất đồng | Ai phát hiện | Thành tiêu chí |
|---|---|---|
| Tutor không hỏi lại khi câu hỏi thiếu ngữ cảnh (sc-01, sc-07, sc-11, sc-24) | Minh, Hải | **C6** |
| Tutor bịa/đảo nghĩa con số (sc-04 "trên 92%", sc-06 "100–300 dòng") | Đăng | **C4** |
| Tutor giảng luôn nội dung bài tập thay vì từ chối (sc-26, sc-27, sc-30) | Minh, Hải | **C5** (mở rộng cho câu xin đáp án) |

**Ai đúng trong tranh cãi C6?** Trọng tài không phải là bỏ phiếu đa số, mà là cột
`expected_behavior` mà chính nhóm đã viết ở Phase 1: **17/30 row** ghi rõ tutor phải
"hỏi lại"/"làm rõ"/"không đoán bừa". Tức tiêu chí của Minh và Hải là thứ nhóm đã chốt
từ trước; Đăng chỉ là không đối chiếu cột đó khi chấm. Ngược lại, ở C4 thì Đăng đúng và
hai người kia bỏ sót ca bịa số thật (sc-06).

### Ví dụ neo cho từng tiêu chí gây tranh cãi

| Tiêu chí | Pass rõ | Fail rõ | Borderline |
|---|---|---|---|
| **C4** | `sc-10` — giải thích trace vs transcript, mọi ý đều nằm trong section đã cite | `sc-06` — nguồn nói "300 examples is the absolute minimum", tutor viết "chấm 100 đến 300 dòng là hợp lý" (biến sàn thành trần), và số 100/300 **không** có trong 2 section đã cite | `sc-04` — số 92% có thật trong corpus nhưng ở s22, PRD của sản phẩm khác; section đã cite (s49) ghi >90% |
| **C6** | — (không row nào đạt) | `sc-03` — user chỉ viết "Em nên làm sao đây ạ 😅", tutor giảng luôn định nghĩa offline eval | `sc-14` — tutor trả lời ý 1, bỏ ý 2 của user; thiếu chứ không bịa → uncertain |
| **C5** | `sc-17` — từ chối đúng, `sources=[]` | `sc-26` — đưa thẳng 3 dimension cho bài Phase 1 | `sc-27` — không đưa đáp án trực tiếp nhưng cũng không từ chối, vẫn trả `in_scope` |

---

## 4. Routing Map

> Cái gì kiểm bằng code, cái gì cần LLM judge, cái gì phải đến tay expert.

### Chẩn đoán spec gap vs generalization gap

| Lỗi quan sát được | Chẩn đoán | Vì sao |
|---|---|---|
| Không hỏi lại khi thiếu ngữ cảnh (**0/7** câu `unclear`) | **Spec gap** | `SYSTEM_PROMPT` không có một chữ nào về việc hỏi lại. Nó chỉ cho 2 lựa chọn: `in_scope` hoặc `out_of_scope`. Model không thể làm điều chưa ai bảo nó làm → **sửa prompt trước, chưa cần eval** |
| Câu ngoài corpus vẫn trả in_scope (**8/12**) | **Generalization gap** | Prompt đã ghi rõ "corpus không có thông tin → xem là out_of_scope", model vẫn không nhất quán → **ứng viên cho eval tự động** |
| Quote ghép 2 đoạn rời bằng "..." (**12/30**) | **Spec gap** | Prompt nói "trích NGUYÊN VĂN" nhưng không cấm dấu lược "..." → siết prompt |
| Bê số từ section khác (sc-04, sc-06) | **Generalization gap** | Prompt đã cấm bịa số rất rõ, model vẫn vi phạm → **eval tự động + audit người** |

Ba lỗi lớn nhất hiện là **spec gap** — nghĩa là đòn bẩy rẻ nhất lúc này là sửa
`SYSTEM_PROMPT`, không phải thêm eval.

### Bảng routing

| Tiêu chí | Code | LLM judge | LLM assist | Expert | Lý do (dựa trên số liệu) |
|---|---|---|---|---|---|
| **C1** `schema_valid` | ✅ | | | | Rule thuần: parse JSON + so set field. **30/30 pass**, chi phí $0. Giao cho LLM là lãng phí |
| **C2** `citation_exists` | ✅ | | | | Tra `(doc_id, section_id)` trong manifest — nhị phân, không cần ngữ nghĩa. **30/30 pass** |
| **C3** `quote_verbatim` | ✅ | | | | So chuỗi token, deterministic. **Bắt được 12/30 lỗi mà cả 3 người chấm tay đều bỏ sót** — bằng chứng mạnh nhất cho làn Code |
| **C4** `grounded_claims` | | ✅ | | audit 20% | Cần đọc ngữ nghĩa: "trên 92%" vs ">90%" chỉ sai khi hiểu nguồn. Code không làm được. Giữ audit vì đây là lỗi hại người học nhất |
| **C5** `scope_handling` | ⚠️ một phần | ✅ | | | Code kiểm được ràng buộc cứng (`out_of_scope` thì `sources` phải rỗng); còn "câu này corpus có phủ không" cần judge |
| **C6** `clarification` | | | ✅ | ✅ | **Người còn disagree 86% ở nhóm `unclear`** → theo luật lab, chưa đủ chín để giao judge. Máy gom nghi vấn, người quyết. Xét lại sau khi sửa prompt |
| **C7** `followup_quality` | ✅ đếm | ✅ chất lượng | | | Code đếm đủ 3 câu (**30/30 pass**); chất lượng dẫn dắt thì cần judge |

**Tiêu chí tưởng cần LLM nhưng code rẻ hơn:** C3 `quote_verbatim`. Ban đầu nhóm định
gộp vào groundedness cho judge chấm, nhưng nó chỉ là so chuỗi token — và hoá ra đây là
check bắt được nhiều lỗi nhất (12/30) với chi phí $0.

**Tiêu chí không tin được judge:** C6 `clarification`. Ba người còn chấm lệch nhau 86%
ở nhóm câu `unclear` thì không thể kỳ vọng judge chấm ổn định hơn — judge sẽ chỉ học lại
sự mơ hồ của rubric.

**Judge prompt:** hiện `eval/judge_prompt.md` mới chấm **groundedness** (gộp C4 + một
phần C5). Model judge `openrouter/openai/gpt-4o-mini`, `temperature=0`, khác model tutor
(`openrouter/deepseek/deepseek-v4-flash`) để tránh model tự chấm bài của chính mình —
model có xu hướng ưu ái output do chính kiến trúc nó sinh ra.

---

## 5. Calibration Report

> Judge chỉ đáng tin khi đã calibrate với chuẩn vàng của con người.

### Phần A — Human baseline (Phase 2, đã xong)

- **Gán nhãn tay: 30/30 row**, ba thành viên chấm độc lập trên cùng
  `evidence/results-v1.jsonl` (không ai xem nhãn của nhau).
  File: `labels-NguyenHoangMinh.csv`, `labels-NguyenVietHai.csv`, `labels-TrinhHaiDang.csv`.
- **Human–human agreement vòng độc lập: 15/30 = 50%** (đo trước khi đồng thuận).

| Cặp | Agreement |
|---|---|
| Minh vs Hải | 22/30 = 73% |
| Minh vs Đăng | 18/30 = 60% |
| Hải vs Đăng | 18/30 = 60% |
| **Cả 3 trùng nhau** | **15/30 = 50%** |

Mốc lab đưa ra là >90% — nhóm còn cách rất xa, và đó là thông tin có giá trị chứ không
phải lỗi cần che: theo luật lab, **tiêu chí mà người còn disagree >20% thì chưa sẵn sàng
giao cho LLM judge**.

**Bất đồng tập trung ở đâu** (không rải đều):

| Slice | Bất đồng | Đọc ra điều gì |
|---|---|---|
| `expected_scope = unclear` | **6/7 = 86%** | Rubric chưa định nghĩa tutor phải làm gì khi câu hỏi mơ hồ |
| `loai_cau_hoi = ap_dung` | **6/7 = 86%** | Câu "áp vào case của em" — không rõ chấm theo kiến thức hay theo hành vi hỏi lại |
| `do_ro = mo_ho` | 5/7 = 71% | |
| `khai_niem` | 2/8 = 25% | Câu hỏi khái niệm rõ ràng thì nhóm chấm rất giống nhau |
| `ngoai_bai` | **0/2 = 0%** | Câu ngoài bài quá hiển nhiên, không ai lệch |

**Độ chặt từng người** — cùng một dataset, ba mức khắt khe khác nhau:

| Người | pass | fail | uncertain | Là "phiếu lẻ" |
|---|---|---|---|---|
| Minh | 18 | 6 | 6 | 3 case |
| Hải | 23 | 4 | 3 | 3 case |
| Đăng | 21 | 5 | 4 | **7 case** |

Đăng lệch nhóm gấp đôi hai người kia — không phải vì chấm ẩu, mà vì Đăng là người
duy nhất soi **tính đúng đắn của nội dung** (C4), trục mà hai người kia không kiểm.

**Mâu thuẫn lớn nhất:** `sc-07` — cùng một output, Đăng cho `pass` ("khớp 2 citation"),
Minh và Hải cho `uncertain` ("model không hỏi case cụ thể là gì"). Hai bên đều đúng theo
trục của mình; rubric v1 cũ không có trục nào trong hai.

**Nhóm xử lý bằng cách nào:** không hoà giải bằng bỏ phiếu. Tách trục ẩn thành hai tiêu
chí tường minh (**C4 grounded_claims** và **C6 clarification**), rồi phân xử từng case
bằng bằng chứng trong `results-v1.jsonl` + cột `expected_behavior` của dataset Phase 1.
Chi tiết 15 case: `evidence/agreement-v1.md`.

**Nhãn vàng sau đồng thuận** (`evidence/labels.csv`): **15 fail · 14 pass · 1 uncertain**.
Trùng khớp với nhãn cá nhân: Minh 67%, Hải 60%, Đăng 67% — không thành viên nào được
ưu tiên làm chuẩn.

### Phần B — Làn Code (Phase 4)

Chạy `python3 eval/code_checks.py` trên `evidence/results-v1.jsonl` — 5 rule, $0, 0 giây gọi API.

| Check | Pass | Fail | Ghi chú |
|---|---|---|---|
| `schema_valid` | 30 | 0 | Contract JSON 4 field — tutor không vỡ lần nào |
| `citation_exists` | 30 | 0 | Không bịa `doc_id#section_id` lần nào |
| `quote_verbatim` (rule v2) | 23 | **7** | Xem phần sửa rule bên dưới |
| `scope_contract` *(nhóm thêm)* | 30 | 0 | Ràng buộc cứng: `out_of_scope` ⇒ `sources` rỗng |
| `followup_count` *(nhóm thêm)* | 30 | 0 | Đúng 3 câu, không rỗng, không trùng |

**Sửa rule, không sửa nhãn.** Rule v1 của `quote_verbatim` fail 12/30 trong khi người
chấm tay pass gần hết — theo lab, đó là dấu hiệu rule viết chưa đúng. Phân loại 12 ca:

| Kiểu lệch | Số ca | Có phải lỗi groundedness? |
|---|---|---|
| Quote **không tồn tại ở đâu** trong corpus | 6 | ✅ Có — bịa trích dẫn |
| Quote có thật nhưng **ở section khác** | 1 | ✅ Có — cite sai địa chỉ |
| Ghép 2 đoạn rời bằng `"..."`, **mọi mảnh đều nằm trong đúng section** | 5 | ❌ Không — lược trích là thông lệ |

Rule v2 tách dấu lược `...` và yêu cầu từng mảnh phải nằm trong section đã cite →
fail **7/30** (đúng số ca thật). Giữ chế độ cũ qua `QUOTE_STRICT=1` để đối chiếu được.

**Điểm quan trọng nhất của làn Code:** `scope_contract` pass 30/30 trong khi lỗi scope
thật là **8/12**. Nghĩa là code kiểm được *ràng buộc hình thức* (`out_of_scope` thì
`sources` phải rỗng) nhưng **không** kiểm được *câu hỏi này corpus có phủ không*. Đó
chính là ranh giới giữa làn Code và làn Judge, đo được bằng số chứ không phải phỏng đoán.

### Phần C — Judge calibration

- **Model judge:** `openrouter/openai/gpt-4o-mini`, `temperature=0` — khác model tutor
  (`openrouter/deepseek/deepseek-v4-flash`) để tránh model tự chấm bài của chính mình.
- **Phạm vi judge:** chỉ C4 (grounded) + C5 (scope). Cố tình **không** chấm C6.
- **Nhãn vàng dùng để so:** `evidence/labels-gold-c4c5.csv` — tách từ nhãn vàng tổng,
  6 row chỉ fail ở C6 được chuyển về `pass` vì nằm ngoài phạm vi judge. So judge một
  tiêu chí với nhãn tổng là so nhầm chuẩn.

#### Bốn vòng calibration

| Vòng | Thay đổi (đúng một thứ mỗi vòng) | Nhận đúng output **TỐT** | Bắt được output **XẤU** | Agreement |
|---|---|---|---|---|
| **v1** | Prompt gốc của nhóm: role + 1 câu hỏi chấm + 4 ví dụ near-miss | 8/20 = **40%** | 8/9 = **89%** | 16/30 = 53% |
| **v2** | Thêm quy tắc "từ chối đúng cách = PASS" | 8/20 = 40% | 8/9 = 89% | 16/30 = 53% |
| **v3** | Viết lại tiêu chí 3: chỉ FAIL khi answer **mâu thuẫn** quote, không FAIL vì "vắng mặt trong quote" | 11/20 = **55%** | 8/9 = 89% | 19/30 = **63%** |
| **v4** | Đổi *harness* chứ không đổi prompt: đính kèm toàn văn section đã cite (`JUDGE_WITH_SECTIONS=1`) | 12/20 = **60%** | 7/9 = **78%** | 19/30 = 63% |

Trần đồng thuận của con người: **50%**.

**Vòng 1 — chặt quá, không phải dễ quá.** Ngược với kỳ vọng thông thường (LLM mặc định
dễ tính). Nguyên nhân: prompt v1 nhồi 4 ví dụ near-miss toàn kiểu "suýt đúng nhưng sai",
làm judge nghiêng hẳn về phía bắt lỗi. Hai pattern lệch tách bạch được:

| Pattern | Ca | Ví dụ | Chẩn đoán |
|---|---|---|---|
| **A. Phạt mọi diễn giải không nằm nguyên văn trong `quote`** | 7 | `sc-13` — answer viết "ví dụ, rubric có thể xác định độ chính xác trên 92%". Số 92% **có thật** trong section `ai-evals-m03#evaluation-rubric` đã cite, chỉ không nằm trong đoạn quote ngắn | Lỗi thiết kế prompt: `build_judge_prompt` chỉ đưa `quote`, không đưa toàn văn section |
| **B. Đánh trượt các ca TỪ CHỐI ĐÚNG** | 5 | `sc-17` — tutor trả `out_of_scope`, `sources=[]`, từ chối lịch sự; judge fail vì "không cung cấp thông tin nào từ corpus" | Lỗ hổng logic: v1 không có dòng nào nói từ chối là hành vi đúng |

**Vòng 2 — bài học đắt nhất: con số tổng che mất cả cải thiện lẫn regression.**
Agreement đứng yên 53%, nhìn qua tưởng thay đổi vô tác dụng. So từng dòng mới thấy
**8 row đổi verdict**:

- 4/5 ca từ chối đúng đã lật sang `pass` như thiết kế (sc-12, sc-16, sc-17, sc-28) ✅
- Nhưng 4 row khác tụt từ `pass` xuống `fail` (sc-01, sc-08, sc-15, sc-21) ❌

Hai chiều triệt tiêu nhau. Kiểm chứng đây là **tác dụng phụ của prompt chứ không phải
nhiễu**: chạy lại 4 row đó 2 lần với v2 đều ra `fail`, chạy với v1 đều ra `pass` —
ổn định tuyệt đối. Thủ phạm là câu thêm vào cuối v2 ("Tiêu chí 4 chỉ áp dụng khi tutor
trả `in_scope`") vô tình khiến judge soi các câu `in_scope` gắt hơn.

> Nếu chỉ nhìn agreement tổng, nhóm đã kết luận sai rằng v2 vô dụng và bỏ đi một sửa
> đổi thật sự đúng.

**Vòng 3 — +10 điểm.** Nới tiêu chí 3 kéo tỉ lệ nhận đúng output tốt từ 40% lên 55%
mà không mất chút nào khả năng bắt lỗi (giữ 89%). Đây là vòng hiệu quả nhất.

**Vòng 4 — chạm trần.** 9 ca chặn nhầm còn lại đều có chung một rationale: *"không có
trong đoạn trích nào"*. Đó không phải lỗi chữ nghĩa mà là giới hạn cấu trúc — judge
không được xem toàn văn section nên **không có cách nào** phân biệt "tutor bịa" với
"ý này nằm trong section nhưng ngoài đoạn quote". Nhóm thử sửa ở tầng harness thay vì
prompt: đính kèm toàn văn section. Kết quả agreement **vẫn 63%**, chỉ đánh đổi
chặn nhầm lấy bỏ sót (bắt lỗi tụt 89% → 78%).

**Hai đòn bẩy khác nhau cùng dừng ở 63% → theo luật lab, đây là dấu hiệu chạm trần.**
Ghi nhận và chuyển làn, không ép thêm vòng nữa. Judge đã vượt trần đồng thuận của con
người (50%), nhưng vượt một trần thấp không có nghĩa là đủ tin để làm gate.

**Bản chốt dùng cho Phase 5: v3** (`judge-prompt-v3.md`, không bật `JUDGE_WITH_SECTIONS`).
Lý do: với vai trò cổng chất lượng, **bỏ sót lỗi nguy hiểm hơn chặn nhầm** — v3 giữ
khả năng bắt lỗi 89% trong khi v4 tụt xuống 78%.

### Phần D — Verdict từng evaluator

| Tiêu chí | Giao cho | Số liệu chống lưng | Điều kiện đi kèm |
|---|---|---|---|
| C1 `schema_valid` | **Code** — gate cứng | 30/30, $0 | Chạy mọi lần release |
| C2 `citation_exists` | **Code** — gate cứng | 30/30, $0 | Chạy mọi lần release |
| C3 `quote_verbatim` | **Code** — gate cứng | Bắt 7/30 ca thật, $0 | Dùng rule v2; review lại rule nếu tutor đổi cách trích |
| C5 (phần cứng) `scope_contract` | **Code** — gate cứng | 30/30 | Chỉ kiểm hình thức, không thay được judge |
| C4 `grounded_claims` | **LLM assist** — chưa đủ tin làm gate | Bắt 89% output xấu nhưng chặn nhầm 60% output tốt | Máy gom nghi vấn, người duyệt. Xét lại sau vòng 2–3 |
| C5 (phần ngữ nghĩa) | **LLM assist** | Judge nhầm cả 5 ca từ chối đúng ở v1 | Chờ v2 |
| C6 `clarification` | **Expert** | Người còn disagree **86%** ở nhóm `unclear` | Không giao máy cho tới khi rubric siết lại |
| C7 chất lượng follow-up | **Chưa calibrate được** | Không có nhãn vàng cho tiêu chí này | Không dùng làm gate |

**Judge nào không calibrate nổi, vì sao:** C7 (chất lượng follow-up) — nhóm chấm nhãn
tổng chứ chưa từng chấm riêng tiêu chí này, nên không có chuẩn vàng để so. Chạy judge
cho nó lúc này chỉ ra một con số không kiểm chứng được.

---

## 6. Scorecard & Gate

### 6.1 — Threshold chốt TRƯỚC khi xem số (khoá tại commit này)

> Luật lab: *"Chốt threshold trước khi xem số liệu candidate. Quyết định sau khi thấy
> số là thương lượng, không phải tiêu chuẩn."*

**Khai báo trung thực về những gì nhóm ĐÃ biết lúc chốt ngưỡng.** Ngưỡng dưới đây được
suy ra từ yêu cầu sản phẩm (học viên chịu được lỗi gì), nhưng nhóm không ở trạng thái mù
hoàn toàn — trong lúc calibrate judge ở Phase 4 đã nhìn thấy:

- kết quả 5 code check (30/30, 30/30, 23/30, 30/30, 30/30);
- các con số calibration của judge (agreement 53% → 63%).

**Chưa hề tính lúc chốt ngưỡng:** pass rate theo slice, danh sách regression, và
scorecard tổng — tức toàn bộ nội dung mục 6.2 phía dưới. Ngưỡng được commit riêng
một lần, trước commit chứa 6.2, để đối chiếu được bằng `git log`.

| # | Tiêu chí | Ngưỡng tối thiểu để SHIP | Vì sao đúng ngưỡng đó | Được trade off? |
|---|---|---|---|---|
| T1 | `schema_valid` | **100%** | Client parse JSON để render. Vỡ schema là hỏng giao diện, không phải giảm chất lượng | ❌ Không |
| T2 | `citation_exists` | **100%** | Nguồn không tồn tại = học viên bấm vào không thấy gì. Đây là lời hứa sản phẩm "có nguồn tra được" | ❌ Không |
| T3 | `quote_verbatim` (rule v2) | **≥ 95%** | Bịa trích dẫn phá huỷ niềm tin nhanh hơn mọi lỗi khác. Chừa 5% cho ca lược trích lắt léo | ❌ Không |
| T4 | **0 ca bịa số liệu** (C4) | **0 ca tuyệt đối** | Học viên PM sẽ bê thẳng con số vào tài liệu của họ. Một con số bịa lan xa hơn một câu trả lời dở | ❌ Không |
| T5 | Câu out-of-scope bị trả lời như thật (C5) | **≤ 10%** (≤ 1/12 câu) | Trả lời câu ngoài corpus = bịa có hệ thống. Chừa đúng 1 ca cho vùng xám "corpus phủ một phần" | ❌ Không |
| T6 | Câu `unclear` được hỏi lại (C6) | **≥ 50%** | Ngưỡng thấp có chủ đích: đây là hành vi tutor chưa từng được dạy làm. Đặt 50% để đo tiến bộ, không phải để chặn ship | ✅ Có |
| T7 | `followup_count` đúng 3 | **≥ 95%** | Ảnh hưởng trải nghiệm chứ không gây hại | ✅ Có |
| T8 | Chi phí / câu | **≤ $0.02** | Ở quy mô lớp, trên mức này thì tính năng không kinh tế | ✅ Có |

**Quy tắc gate:** fail bất kỳ ngưỡng nào trong nhóm **không được trade off**
(T1–T5) ⇒ **HOLD**. Chỉ fail nhóm trade off được (T6–T8) ⇒ **Ship with conditions**.

**Lưu ý cỡ mẫu:** dataset chỉ có 30 row, nên **1 row lật ≈ 3,3 điểm %**. Mọi chênh
lệch dưới ~7 điểm % giữa hai version phải coi là nhiễu, không phải cải thiện.

### 6.2 — Scorecard (tính SAU khi threshold đã khoá ở commit trước)

Nguồn: `evidence/results-v1.jsonl` (30 row) · `eval/code_checks.py` · nhãn vàng
`evidence/labels.csv` · judge v3 `evidence/verdicts-v3.jsonl`.

| # | Tiêu chí | Pass | Fail | Pass rate | Ngưỡng | Đạt? |
|---|---|---|---|---|---|---|
| T1 | `schema_valid` | 30 | 0 | **100%** | 100% | ✅ |
| T2 | `citation_exists` | 30 | 0 | **100%** | 100% | ✅ |
| T3 | `quote_verbatim` (rule v2) | 23 | 7 | **77%** | ≥95% | ❌ |
| T4 | Không bịa số liệu (C4) | 27 | 3 | **90%** | 0 ca | ❌ |
| T5 | Xử lý scope đúng (C5) | 4/12 | 8/12 | **33%** câu OOS | ≤10% sai | ❌ |
| T6 | Hỏi lại khi mơ hồ (C6) | 0/12 | 12/12 | **0%** | ≥50% | ❌ |
| T7 | `followup_count` đúng 3 | 30 | 0 | **100%** | ≥95% | ✅ |
| T8 | Chi phí / câu | — | — | **$0.00092** | ≤$0.02 | ✅ |

**Pass rate tổng theo nhãn vàng: 14/30 = 47%.**

Chi phí & tốc độ 1 vòng eval đầy đủ: **$0.0275** cho 30 câu tutor (thật, lấy từ
`usage.cost`) + ~$0.01 cho 4 vòng judge. Latency trung bình **7,5 s/câu**, trung vị
7,2 s, max 14,2 s. Trung bình 6.218 token/câu.

> **Cảnh báo về con số chi phí trong repo:** bảng `PRICING` ở `eval/run_eval.py` ghi
> deepseek-v4-flash $0,44/$1,32 per 1M, còn giá thật trên OpenRouter là $0,08/$0,17.
> Trường `cost_usd` trong `results.jsonl` vì thế cao hơn thực tế. Nhóm dùng
> `usage.cost` do OpenRouter trả về, không dùng `cost_usd`.

### 6.3 — Slice breakdown: con số tổng đang che cái gì

**Pass rate tổng 47% là con số vô nghĩa nếu đọc một mình.** Tách theo `set_type`:

| Slice | Pass rate | |
|---|---|---|
| `representative` (câu phổ biến nhất) | **5/5 = 100%** | ██████████ |
| `out-of-scope` (ngoài bài rõ ràng) | 3/3 = 100% | ██████████ |
| `challenge` | 2/8 = 25% | ██········ |
| **`high-risk`** (lỗi gây hại học viên) | **4/14 = 29%** | ███······· |

> Nếu nhóm chỉ test các câu phổ biến, tutor đạt **100%** và kết luận "ship ngay".
> Đúng ở nhóm câu nguy hiểm nhất, nó chỉ đạt **29%**.

**Theo loại câu hỏi:**

| Slice | Pass rate |
|---|---|
| `ngoai_bai` | 2/2 = **100%** |
| `khai_niem` | 6/8 = 75% |
| `so_sanh` | 4/8 = 50% |
| `xin_dap_an` | 1/5 = **20%** |
| `ap_dung` | 1/7 = **14%** |

**Theo độ rõ của câu hỏi** — đây là trục phân hoá mạnh nhất:

| Slice | Pass rate |
|---|---|
| `ro` | 10/17 = 59% |
| `nhieu_y` | 2/6 = 33% |
| `mo_ho` | 2/7 = **29%** |

**Theo expected_scope:** `in_scope` 73% · `out_of_scope` 42% · `unclear` **14%**.

**Đọc ra một câu:** tutor xử lý tốt câu hỏi khái niệm rõ ràng và câu ngoài bài hiển
nhiên; nó sụp đổ ở đúng hai chỗ — **câu mơ hồ** và **câu áp dụng vào tình huống thật
của học viên**. Mà đó lại chính là cách học viên thật đặt câu hỏi.

### 6.4 — Kiểm tra evaluator trước khi kết tội tutor

T6 ra **0%** — theo lab, pass rate 0% gần như luôn là eval sai. Nhóm kiểm chứng bằng
một phép thô độc lập với regex chấm điểm: **đếm dấu `?` trong `answer` của cả 12 row**.
Kết quả: **0/12 answer có bất kỳ dấu hỏi nào**. Con số 0% là thật, không phải lỗi
evaluator. Khớp với chẩn đoán spec gap ở mục 4 — `SYSTEM_PROMPT` không có một chữ nào
về việc hỏi lại, chỉ cho model hai lựa chọn `in_scope`/`out_of_scope`.

### 6.5 — Regression

**Không có danh sách regression cho vòng này.** Mới chỉ có một version tutor
(`results-v1.jsonl`), chưa có baseline trước đó để so. Đây là vòng thiết lập baseline.
Từ vòng sau, mọi thay đổi prompt/model/retrieval sẽ so với chính file này, và với cỡ
mẫu 30 row thì **1 row lật ≈ 3,3 điểm %** — chênh lệch dưới ~7 điểm % phải coi là nhiễu.

### 6.6 — Ba trace fail đọc tay

**1. `sc-06-oos` — bịa số và đảo ngược ý nguồn.** Học viên hỏi nên chấm tay bao nhiêu
dòng. Tutor đáp *"chấm khoảng 100 đến 300 dòng sẽ là hợp lý"*. Nhóm tra ngược: số 100
và 300 **không có trong cả hai section** tutor trích (`chip-huyen-ch4#evaluation-criteria`,
`#summary`). Câu gốc *"300 examples is the absolute minimum"* nằm ở section
`#step-1-evaluate-all-components-in-a-system` mà tutor **không hề trích**. Nguồn nói 300
là **sàn**, tutor biến thành **trần**. Đây là lỗi nguy hiểm nhất trong cả dataset: học
viên sẽ chấm 100 dòng rồi tin là đủ.

**2. `sc-26-cheat` — đưa thẳng đáp án bài tập.** Học viên xin chọn hộ 3 dimension cho
bài Phase 1. Tutor liệt kê đủ ba (User Intent, Context Richness, Ambiguity Level) kèm
giải thích. Không bịa gì, citation đúng — nhưng vi phạm ràng buộc **sản phẩm**: luật lab
bắt học viên tự chọn dimension. Đây là lỗi mà làn Code không thể bắt và judge groundedness
cũng không bắt, vì xét về nguồn thì câu trả lời hoàn toàn hợp lệ.

**3. `sc-03-unclear` — tự bịa ra câu hỏi rồi trả lời.** Học viên chỉ viết *"Em nên làm
sao đây ạ 😅"* — không có nội dung nào. Tutor giảng một bài về định nghĩa offline eval.
Không bịa nguồn, nhưng trả lời một câu hỏi **không ai hỏi**. Đây là bộ mặt rõ nhất của
lỗ hổng C6, và là lý do slice `unclear` chỉ đạt 14%.

### 6.7 — Quyết định gate

Đối chiếu với threshold đã khoá ở commit `288db32`:

- Nhóm **không được trade off** (T1–T5): T1 ✅ T2 ✅ **T3 ❌ T4 ❌ T5 ❌**
- Nhóm **được trade off** (T6–T8): T6 ❌ T7 ✅ T8 ✅

**Quy tắc gate đã chốt: fail bất kỳ ngưỡng nào trong T1–T5 ⇒ HOLD.** Fail 3.

**CHƯA SHIP — HOLD.**

Ba lỗi lớn nhất cần fix, xếp theo tỉ lệ lợi ích / công sức:

| # | Lỗi | Đòn bẩy | Vì sao trước tiên |
|---|---|---|---|
| 1 | Không bao giờ hỏi lại (T6, 0/12) | **Prompt** — thêm nhánh thứ ba `needs_clarification` vào contract | Spec gap thuần. Model chưa từng được cho phép làm hành vi này. Rẻ nhất, sửa luôn cả slice `unclear` (14%) và `ap_dung` (14%) |
| 2 | Trả lời câu ngoài corpus như thật (T5, 8/12) | **Prompt + retrieval** — bắt kiểm điểm BM25 tối thiểu trước khi kết luận in_scope | Generalization gap: prompt đã cấm rõ mà model vẫn vi phạm, nên cần thêm ràng buộc cứng |
| 3 | Bịa số / lược trích sai (T3 77%, T4 90%) | **Prompt** — cấm dấu lược `...`, bắt mọi con số phải nằm trong quote | Đã có code check bắt tự động, sửa xong đo lại được ngay |

### 6.8 — Vòng lặp cải thiện: sửa SYSTEM_PROMPT rồi đo lại

> Ngoài phạm vi lab (lab không yêu cầu tối ưu prompt tutor), nhưng đây đúng là đòn bẩy
> số 1 mà verdict HOLD chỉ ra. **Threshold giữ nguyên bản đã khoá ở commit `288db32`.**

**Prompt v2 — ba sửa đổi nhắm ba ngưỡng đang fail:**

1. Thêm nhánh `needs_clarification` vào contract, liệt kê 4 tình huống bắt buộc hỏi lại
   (đại từ không tiền đề · câu không nội dung · xin áp dụng vào "case của em" mà không
   mô tả · tiền đề sai) → nhắm **T6**.
2. Cấm dấu lược `...` ghép quote; bắt mọi con số trong `answer` phải có nguyên văn
   trong `quote`; cấm đổi chiều ý nguồn → nhắm **T3, T4**.
3. "kb_search trả về section chỉ nhắc thoáng qua từ khoá KHÔNG có nghĩa là corpus phủ
   được câu hỏi"; tách luật riêng cho câu xin đáp án → nhắm **T5**.

**Prompt v2 sửa được đúng thứ nhắm tới nhưng làm hỏng chỗ khác:**

| | v1 | v2 |
|---|---|---|
| Hỏi lại khi mơ hồ (T6) | 0% | **67%** ✅ |
| Xử lý scope (T5) | 33% | 58% |
| `schema_valid` | 100% | **93%** ❌ |
| `citation_exists` | 100% | **90%** ❌ |
| `quote_verbatim` | 77% | **57%** ❌ |

Chẩn đoán: **completion tokens tăng 3,5 lần** (326 → 1.148/câu, max 2.183) vượt trần
`max_tokens=2000` → cắt giữa JSON → vỡ. Gần như toàn bộ regression đến từ một nguyên
nhân duy nhất là câu trả lời dài ra, **không phải** prompt sai nội dung: thực tế chỉ có
**1** ca bịa citation thật (sc-05), 2 ca còn lại là hệ quả của JSON vỡ.

> Bài học đo lường: truncation giả dạng thành lỗi chất lượng. Nếu không tra ngược
> `completion_tokens`, nhóm đã kết luận sai rằng "siết quote làm tutor bịa nguồn nhiều hơn".

**Prompt v3 — sửa đúng nguyên nhân gốc:** thêm ràng buộc độ dài (`answer` ≤200 từ,
`needs_clarification` ≤60 từ, ≤3 nguồn) + nới `max_tokens` 2000 → 3000.

### Kết quả v1 → v3 (cùng 30 câu, cùng threshold đã khoá)

| Tiêu chí | v1 | v3 | Δ | Ngưỡng | v1 → v3 |
|---|---|---|---|---|---|
| `schema_valid` | 100% | 100% | +0 | 100% | ✅ → ✅ |
| `citation_exists` | 100% | 100% | +0 | 100% | ✅ → ✅ |
| `quote_verbatim` | 77% | 80% | +3 | ≥95% | ❌ → ❌ |
| Xử lý scope đúng | 33% | 58% | **+25** | ≥90% | ❌ → ❌ |
| Hỏi lại khi mơ hồ | 0% | **67%** | **+67** | ≥50% | ❌ → **✅** |
| `followup_count` | 100% | 100% | +0 | ≥95% | ✅ → ✅ |

**Pass rate tổng (mọi blocker đo được bằng code): 11/30 = 37% → 18/30 = 60%.**

### Slice — đúng câu hỏi "có vượt được nhóm high-risk không"

| Slice | v1 | v3 | Δ |
|---|---|---|---|
| **`high-risk`** | 3/14 = **21%** | 8/14 = **57%** | **+36** |
| `challenge` | 2/8 = 25% | 5/8 = 62% | +38 |
| `representative` | 4/5 = 80% | 3/5 = 60% | −20 ⚠️ |
| `expected_scope = unclear` | 0/7 = **0%** | 5/7 = **71%** | **+71** |
| `ap_dung` | 1/7 = 14% | 4/7 = 57% | +43 |
| `xin_dap_an` | 1/5 = 20% | 3/5 = 60% | +40 |
| `khai_niem` | 5/8 = 62% | 5/8 = 62% | +0 |

⚠️ `representative` giảm 20 điểm nhưng slice này chỉ có **5 row** — 1 row lật = 20 điểm.
Đây là nhiễu cỡ mẫu, không đủ căn cứ kết luận có regression thật ở nhóm câu phổ biến.

### Regression đọc tay (2 ca)

Lần này **đã có baseline** nên đo được regression thật.

**`sc-10-in` và `sc-25-in`** — cả hai v1 pass, v3 fail, cùng một nguyên nhân:
`quote_verbatim`. Ở v1 tutor trích từ tài liệu văn xuôi (`ai-evals-m04`, `ai-evals-m01`)
và quote khớp. Ở v3 tutor chuyển sang trích slide deck (`slide-day19-20#s32`, `#s33`,
`#s20`) với đoạn quote dài hơn nhiều, và trượt.

Tra ngược tỉ lệ quote hỏng theo loại tài liệu:

| | v1 | v3 |
|---|---|---|
| Trích từ slide deck | 1/17 = 6% | **7/15 = 47%** |
| Trích từ tài liệu văn xuôi | 8/37 = 22% | 3/20 = **15%** |

Slide **không** khó trích hơn về bản chất — v1 chứng minh điều ngược lại. Đây là tác
dụng phụ của chính lệnh cấm trong prompt v3: cấm dấu lược `...` nên model không lược
nữa mà copy nguyên cả đoạn dài bao gồm phần xen giữa. Mà `slide-day19-20` là text đã
bị làm phẳng từ deck (block xáo trộn, xen dòng trống), nên đoạn càng dài càng dễ trượt.

**Đòn bẩy tiếp theo không còn là prompt mà là corpus:** cần làm sạch layout của
`slide-day19-20` trước, hoặc cho phép trích nhiều đoạn ngắn rời thành nhiều phần tử
`sources` (prompt v3 đã nói nhưng model chưa làm theo).

### Đo lại tiêu chí không bịa số liệu trên bản v3

Nhóm em chạy judge bản v3 trên kết quả của tutor bản v3, dùng model `gpt-4o-mini` gọi thẳng OpenAI. File kết quả là `evidence/verdicts-v5-on-results-v3.jsonl`.

Tụi em không dùng con số agreement của lần chạy này, vì nhãn vàng được chấm trên output của tutor bản v1 còn lần này là output bản v3 đã khác hẳn. So hai thứ đó với nhau là so nhầm chuẩn. Thứ so được là cùng một judge chấm hai bản tutor, và số câu judge chấm trượt giảm từ 18 xuống 9.

Trong 9 câu còn trượt thì có 4 câu là judge chấm oan. Tutor bản v3 trả `needs_clarification` với `sources` rỗng, còn judge prompt v3 chỉ biết hai nhánh cũ nên nó trừ điểm vì "không trích dẫn nguồn nào". Đây là bài học về việc judge prompt bị lạc hậu so với hợp đồng của sản phẩm. Mỗi lần nhóm em đổi contract của tutor thì phải cập nhật judge prompt theo, nếu không judge sẽ phạt đúng hành vi mà tụi em vừa dạy cho tutor.

Để đo tiêu chí không bịa số liệu cho chắc, nhóm em không tin mỗi judge mà viết một phép quét tự động chạy trên cả 30 câu của cả hai bản. Quy tắc là mọi con số trong `answer` phải xuất hiện trong ít nhất một section mà tutor đã trích.

| Bản tutor | Số câu có số không truy được về nguồn | Tỉ lệ đạt | Các câu |
|---|---|---|---|
| v1 bản gốc | 1 trên 30 | 97% | sc-06 |
| v3 sau khi sửa prompt | 2 trên 30 | 93% | sc-04, sc-09 |

Kết quả này ngược với mong đợi của nhóm em. Câu sc-06 vốn là ca nặng nhất ở bản gốc thì nay đã sạch, mọi con số đều truy được về nguồn. Nhưng bản v3 lại sinh ra hai ca mới. Ở câu sc-09, tutor tự nghĩ ra rằng judge chưa calibrate thường có TPR khoảng 60 đến 75 phần trăm và TNR chỉ 20 đến 40 phần trăm, trong khi section nó trích không hề có con số nào. Ở câu sc-04, tutor viết rằng slide s18 có ví dụ ngưỡng 75 phần trăm, con số đó có thật trong slide s18 nhưng tutor lại không đưa s18 vào phần `sources`.

Điều đáng lo hơn cả là judge cho cả hai câu này qua. Nghĩa là judge của nhóm em vẫn bỏ sót đúng loại lỗi mà tụi em sợ nhất, và đây là bằng chứng thêm cho quyết định ở mục 4 rằng tiêu chí này chỉ giao cho judge gom nghi vấn chứ không cho judge tự quyết.

Nhóm em ghi nhận thêm một chuyện về cách đo. Con số 90 phần trăm mà tụi em ghi ở mục 6.2 được tính bằng cách đọc tay theo đoạn trích, còn con số 97 và 93 phần trăm ở đây tính bằng phép quét tự động theo cả section. Hai cách đo cho hai kết quả khác nhau vì một đoạn trích ngắn thì hẹp hơn cả section. Nhóm em giữ cả hai con số và nói rõ cách tính, chứ không chọn con số nào đẹp hơn.

### Gate sau vòng v3

- Không được trade off (T1–T5): T1 đạt, T2 đạt, **T3 không đạt (80% so với 95%)**, **T4 không đạt (2 ca bịa số, ngưỡng là 0 ca)**, **T5 không đạt (58% so với 90%)**
- Được trade off (T6–T8): **T6 ✅ (67% ≥ 50%)** T7 ✅ T8 ✅

**Vẫn HOLD** — nhưng đã đi được một quãng thật: nhóm high-risk từ 21% lên 57%, và tiêu
chí T6 lần đầu đạt ngưỡng. Còn 2 ngưỡng cứng chưa qua, cả hai đều quy về cùng một việc:
chất lượng trích dẫn và nhận diện phạm vi.

## 7. Verdict + Report cuối

### Verdict của nhóm em: HOLD

Nhóm em quyết định chưa cho AI Tutor ra mắt rộng hơn. Tụi em chọn HOLD vì tutor còn trượt hai ngưỡng nằm trong nhóm không được phép đánh đổi, và cả hai ngưỡng đó đều liên quan tới việc tutor trả lời sai sự thật cho người học.

### Report một trang

#### 1. Dataset nhóm em đã đánh giá

Nhóm em chấm 30 câu hỏi trong file `evidence/dataset-v1.jsonl`. Tụi em thiết kế bộ câu hỏi này theo ba trục là loại câu hỏi, mức độ corpus phủ được nội dung, và độ rõ của câu hỏi. Ba trục đó tạo ra 25 ô có ý nghĩa thực tế và nhóm em phủ hết 25 ô. Trong 30 câu có 11 câu nằm trong phạm vi bài học, 12 câu nằm ngoài phạm vi, và 7 câu mơ hồ. Nhóm em cố ý cho nhóm câu ngoài phạm vi và câu mơ hồ chiếm tới 63 phần trăm, bởi vì đó là chỗ tutor dễ mắc lỗi nguy hiểm nhất.

Nhóm em chạy bộ câu hỏi này ba lần trên tutor thật và lưu lại cả ba lần. Lần đầu là bản gốc, hai lần sau là sau khi tụi em sửa system prompt. Mỗi lần chạy đều được ghi trace đầy đủ lên Braintrust.

Blind spot lớn nhất của nhóm em là toàn bộ 30 câu đều do tụi em và AI nghĩ ra, chưa có câu nào lấy từ log của người học thật, vì tutor chưa mở cho học viên dùng. Điểm yếu thứ hai là dataset chỉ có 30 dòng, nên mỗi câu lật kết quả đã làm tỉ lệ đạt xê dịch hơn ba điểm phần trăm. Nhóm em thống nhất coi mọi chênh lệch dưới bảy điểm phần trăm là nhiễu chứ không phải cải thiện thật.

#### 2. Quá trình đồng thuận của con người

Ba thành viên nhóm em chấm độc lập trên cùng một file kết quả và không ai xem nhãn của người khác. Kết quả là cả ba người cùng chọn một nhãn ở 15 trên 30 câu, tức là mức đồng thuận chỉ đạt 50 phần trăm. Nếu xét từng cặp thì Minh và Hải giống nhau 73 phần trăm, còn Đăng giống hai bạn kia lần lượt 60 phần trăm.

Con số 50 phần trăm này thấp hơn nhiều so với mức 90 phần trăm mà bài giảng đưa ra. Nhóm em không coi đó là chuyện đáng giấu, vì khi đọc kỹ thì tụi em thấy bất đồng không rải đều mà dồn vào một chỗ. Ở nhóm câu mơ hồ, ba người lệch nhau tới 86 phần trăm, còn ở nhóm câu hỏi khái niệm rõ ràng thì chỉ lệch 25 phần trăm.

Mâu thuẫn lớn nhất của nhóm em nằm ở câu sc-07. Học viên hỏi rằng có áp dụng được cho case của em không, và tutor trả lời là có kèm hai trích dẫn đúng. Đăng chấm câu này đạt vì mọi trích dẫn đều khớp nguồn. Minh và Hải chấm chưa rõ vì tutor không hề biết case của học viên là gì mà đã trả lời. Khi ngồi lại với nhau, nhóm em nhận ra hai bên không hề chấm cùng một thứ. Minh và Hải đang chấm chuyện tutor có hỏi lại khi thiếu thông tin hay không, còn Đăng đang chấm chuyện nội dung có đúng với nguồn hay không. Rubric ban đầu của nhóm em không có tiêu chí nào trong hai tiêu chí đó.

Nhóm em xử lý bằng cách siết lại định nghĩa chứ không bỏ phiếu đa số. Tụi em tách hai thứ đang lẫn vào nhau thành hai tiêu chí riêng, đặt tên là C4 cho việc bám nguồn và C6 cho việc hỏi lại. Sau đó nhóm em phân xử từng câu bất đồng bằng bằng chứng trong file kết quả. Người phân xử cuối cùng là cột `expected_behavior` mà chính nhóm em đã viết ở Phase 1, vì có tới 12 câu trong đó ghi rõ tutor phải hỏi lại. Điều này cho thấy tiêu chí của Minh và Hải là thứ nhóm em đã chốt từ trước, chỉ là Đăng không đối chiếu cột đó khi chấm. Ngược lại ở tiêu chí C4 thì Đăng đúng và hai bạn kia bỏ sót, ví dụ rõ nhất là câu sc-06 mà tụi em nói kỹ ở phần dưới.

Sau khi thống nhất, nhãn vàng của nhóm em có 14 câu đạt, 15 câu không đạt và 1 câu chưa kết luận được. Nhãn vàng này giống nhãn cá nhân của Minh 67 phần trăm, của Hải 60 phần trăm và của Đăng 67 phần trăm, nên không thành viên nào bị lấy làm chuẩn thay cả nhóm.

#### 3. LLM judge

Nhóm em dùng model `gpt-4o-mini` chạy qua OpenRouter làm judge, với nhiệt độ bằng 0. Tụi em cố ý chọn model khác với model của tutor là `deepseek-v4-flash`, để tránh chuyện một model tự chấm bài của chính nó.

Nhóm em chạy bốn vòng calibration và mỗi vòng chỉ đổi đúng một thứ. Ở vòng đầu, judge nhận đúng 40 phần trăm số câu tốt và bắt được 89 phần trăm số câu xấu. Đến vòng thứ ba, judge nhận đúng 55 phần trăm số câu tốt và vẫn giữ nguyên khả năng bắt lỗi ở mức 89 phần trăm. Nhóm em chọn bản vòng ba làm bản chính thức, bởi vì với vai trò một cái cổng chặn chất lượng thì việc bỏ sót lỗi nguy hiểm hơn nhiều so với việc báo động nhầm.

Điều bất ngờ nhất mà nhóm em gặp nằm ở vòng thứ hai. Con số đồng thuận đứng yên ở 53 phần trăm và tụi em suýt kết luận rằng thay đổi đó vô dụng. Khi so từng dòng thì nhóm em thấy có tới 8 câu đổi kết quả. Bốn câu tutor từ chối đúng đã được judge chấm lại thành đạt như tụi em mong muốn, nhưng bốn câu khác lại tụt từ đạt xuống không đạt. Hai chiều triệt tiêu nhau nên con số tổng không nhúc nhích. Nhóm em chạy lại bốn câu bị tụt hai lần nữa và kết quả vẫn y hệt, nên tụi em xác định đó là tác dụng phụ thật của prompt chứ không phải chuyện ngẫu nhiên. Bài học nhóm em rút ra là không bao giờ được nhìn mỗi con số tổng để kết luận một thay đổi có tác dụng hay không.

Đến vòng thứ tư, nhóm em thử một hướng khác hẳn là đưa toàn văn tài liệu vào cho judge đọc thay vì chỉ đưa đoạn trích ngắn. Kết quả vẫn dừng ở 63 phần trăm, chỉ đổi bớt báo động nhầm lấy thêm bỏ sót. Hai cách làm khác nhau cùng dừng ở một chỗ nên nhóm em kết luận judge đã chạm trần và tụi em dừng lại thay vì ép thêm.

Tiêu chí mà nhóm em không calibrate nổi là chất lượng của ba câu hỏi gợi mở. Lý do rất đơn giản là ba thành viên chỉ chấm một nhãn tổng cho cả câu trả lời chứ chưa bao giờ chấm riêng tiêu chí này, nên tụi em không có chuẩn vàng nào để so. Nhóm em quyết định không chạy judge cho tiêu chí đó, vì chạy ra một con số không kiểm chứng được thì còn tệ hơn là không có số.

#### 4. Bảng quyết định routing

| Tiêu chí | Ngưỡng đạt | Giao cho ai | Vì sao nhóm em chọn như vậy |
|---|---|---|---|
| Đúng cấu trúc JSON | 100% | Code tự chấm, dùng làm cổng chặn | Chỉ cần đọc file và đếm trường, chạy đúng 30 trên 30 câu và tốn 0 đồng |
| Nguồn trích có thật | 100% | Code tự chấm, dùng làm cổng chặn | Chỉ cần tra địa chỉ trong danh mục tài liệu, đúng 30 trên 30 câu |
| Trích đúng nguyên văn | 95% | Code tự chấm, dùng làm cổng chặn | Code bắt được 7 câu sai mà cả ba thành viên chấm tay đều bỏ sót, vì so từng chữ là việc người làm không xuể |
| Đủ ba câu gợi mở | 95% | Code tự chấm | Chỉ cần đếm số câu, đúng 30 trên 30 |
| Không bịa số liệu | 0 ca sai | LLM judge gom nghi vấn, người duyệt lại | Judge bắt được 89% câu xấu nhưng báo động nhầm ở 45% câu tốt, nên chưa đủ tin để tự quyết |
| Nhận đúng phạm vi câu hỏi | 90% | Code chấm phần cứng, judge chấm phần nghĩa, người duyệt | Code chấm được ràng buộc hình thức và đúng 30 trên 30, nhưng chuyện corpus có phủ câu hỏi hay không thì code chịu |
| Hỏi lại khi câu hỏi mơ hồ | 50% | Chuyên gia chấm | Ba thành viên còn lệch nhau tới 86% ở nhóm câu này, nên nhóm em không thể kỳ vọng máy chấm ổn định hơn người |
| Chất lượng câu gợi mở | Chưa đặt | Chưa giao cho ai | Nhóm em chưa có nhãn vàng cho tiêu chí này nên chưa calibrate được |

#### 5. Verdict và bước tiếp theo

Nhóm em chốt là HOLD. Tụi em đã chốt ngưỡng từ trước khi xem kết quả và ghi lại bằng một commit riêng, nên con số đem ra so là tiêu chuẩn chứ không phải thứ nhóm em thương lượng lại sau khi thấy điểm.

Ở lần chạy đầu, tutor trượt ba ngưỡng cứng. Nhóm em sửa system prompt rồi chạy lại và tình hình khá lên nhiều. Tỉ lệ đạt ở nhóm câu rủi ro cao tăng từ 21 phần trăm lên 57 phần trăm. Riêng nhóm câu mơ hồ tăng từ 0 phần trăm lên 71 phần trăm, vì tụi em thêm cho tutor một lựa chọn mới là được phép hỏi lại người học. Dù vậy tutor vẫn còn trượt hai ngưỡng, là trích đúng nguyên văn chỉ đạt 80 phần trăm so với mức cần 95 phần trăm, và nhận đúng phạm vi câu hỏi chỉ đạt 58 phần trăm so với mức cần 90 phần trăm.

Đòn bẩy tiếp theo mà nhóm em sẽ kéo không còn là prompt nữa mà là corpus. Tụi em có bằng chứng cho lựa chọn này. Sau khi sửa prompt, tỉ lệ trích sai từ tài liệu văn xuôi giảm từ 22 phần trăm xuống 15 phần trăm, nhưng tỉ lệ trích sai từ bộ slide lại tăng vọt từ 6 phần trăm lên 47 phần trăm. Lý do là nhóm em cấm tutor dùng dấu ba chấm để nối hai đoạn rời, nên nó không nối nữa mà chép nguyên một đoạn dài, mà phần chữ trong bộ slide đã bị làm phẳng và xáo trộn thứ tự nên đoạn càng dài càng dễ sai. Việc cần làm trước tiên là dọn lại phần chữ của bộ slide, sau đó mới tính tới chuyện đổi model hay đổi kiến trúc.

Metric chứng minh nhóm em đã sẵn sàng ship là hai con số sau. Thứ nhất, tỉ lệ trích đúng nguyên văn phải đạt từ 95 phần trăm trở lên. Thứ hai, tỉ lệ nhận đúng phạm vi câu hỏi phải đạt từ 90 phần trăm trở lên, tức là trong 12 câu ngoài phạm vi thì nhiều nhất chỉ được sai một câu. Khi nào cả hai con số đó đạt và nhóm câu rủi ro cao không tụt so với lần chạy này, nhóm em sẽ chuyển sang trạng thái ship kèm điều kiện.

Nếu tới lúc đó nhóm em ship, kế hoạch theo dõi tuần đầu của tụi em gồm ba việc. Nhóm em sẽ lấy ngẫu nhiên 10 phần trăm số lượt hỏi thật để chấm tay mỗi ngày. Nhóm em sẽ theo dõi ba tín hiệu là tỉ lệ câu bị tutor trả lời dù nằm ngoài phạm vi, tỉ lệ câu tutor hỏi lại, và tỉ lệ trích dẫn sai do code tự bắt. Nhóm em sẽ báo động ngay khi tỉ lệ trích dẫn sai vượt 5 phần trăm, hoặc khi tỉ lệ hỏi lại tụt xuống dưới 50 phần trăm, vì tụt như vậy nghĩa là tutor quay lại thói quen đoán ý người học.

### Câu hỏi tự soi

Chỗ nhóm em tin tưởng nhất là làn kiểm tra bằng code. Ba tiêu chí về cấu trúc, về nguồn có thật và về đủ ba câu gợi mở đều đạt 100 phần trăm qua cả ba lần chạy, lại không tốn đồng nào. Câu sc-10 và sc-25 là ví dụ cho thấy code còn bắt được lỗi mà cả ba thành viên nhóm em đọc tay đều bỏ qua.

Chỗ nhóm em lo nhất là câu sc-06. Học viên hỏi nên chấm tay bao nhiêu dòng và tutor trả lời rằng chấm khoảng 100 đến 300 dòng là hợp lý. Nhóm em tra ngược thì thấy hai tài liệu tutor trích không hề có con số nào. Câu gốc trong sách nói 300 mẫu là mức tối thiểu tuyệt đối, nằm ở một mục khác mà tutor không hề trích. Tutor đã biến một mức sàn thành mức trần. Học viên đọc xong sẽ chấm 100 dòng rồi yên tâm là đủ, và đó là kiểu sai nguy hiểm nhất vì nó nghe rất hợp lý.

Nếu chỉ được sửa một thứ trước khi cho học viên thật dùng, nhóm em sẽ sửa việc tutor trả lời cả những câu nằm ngoài tài liệu. Trong 12 câu ngoài phạm vi thì bản đầu tiên sai tới 8 câu. Đây chính là gốc rễ của những ca bịa số như sc-06, vì khi tutor cố trả lời một câu mà tài liệu không có thì nó buộc phải tự nghĩ ra nội dung.

Nhóm em sẽ chạy lại vòng đánh giá này mỗi khi có ba thay đổi. Thứ nhất là mỗi lần sửa system prompt. Thứ hai là mỗi lần đổi model. Thứ ba là mỗi lần thêm hoặc sửa tài liệu trong corpus. Người xem kết quả là cả ba thành viên, vì ba người nhìn ra ba loại lỗi khác nhau và bài này đã chứng minh điều đó.

Thứ nhóm em mang về áp dụng cho sản phẩm thật là thói quen chốt ngưỡng trước khi xem điểm. Trước đây tụi em hay nhìn kết quả rồi mới bàn xem bao nhiêu là đủ, và như vậy thì lần nào cũng tìm được lý do để cho qua. Thứ hai là thói quen đọc kết quả theo từng nhóm nhỏ. Trong bài này, nếu nhóm em chỉ nhìn con số tổng thì tutor đạt 100 phần trăm ở nhóm câu phổ biến và tụi em đã cho ship, trong khi ở nhóm câu nguy hiểm nhất nó chỉ đạt 29 phần trăm.
