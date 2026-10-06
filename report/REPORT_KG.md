# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đỗ Phúc Hùng  **MSSV:** 2A202602762  **Ngày:** 06/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small | top_k=3 | chunk_size=800 | chunks=176 | KG: 205 nodes / 382 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     40.3
graph       196     91958     4709   0.00933    101.8

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.35
graph       0.83   1.67     5798       72   0.00091     2.08
```

| Chỉ số | Flat | Graph | Graph / Flat |
|---|---|---|---|
| Indexing USD | $0.00112 | $0.00933 | ×8.33 |
| Indexing giây | 40.3s | 101.8s | ×2.53 |
| Mỗi câu: USD | $0.00013 | $0.00091 | ×7.00 |
| Mỗi câu: giây | 1.35s | 2.08s | ×1.54 |
| Mỗi câu: in_tok | 694 | 5798 | ×8.35 |

**Chi phí tăng thêm đến từ đâu?**

Chi phí tăng đến từ hai nguồn chính. **Thứ nhất, phần indexing:** Graph cần thêm LLM gọi 20 lần (20 bài báo × 1 call/bài) để trích xuất Case, Person, Substance từ tin tức — tốn $0.00821 tiền LLM, so với Flat RAG chỉ embed các chunk sẵn có ($0.00112). **Thứ hai, phần querying:** GraphRAG đưa thêm facts từ graph vào prompt, khiến mỗi câu hỏi tiêu tốn ~8× token đầu vào (5798 so với 694). Facts từ graph chứa dữ kiện cấu trúc (Điều luật, khoản, mức phạt) — rất hữu ích cho câu cross-KB nhưng lãng phí cho câu single-hop. Với số câu hỏi lớn hơn, chi phí tăng thêm này có thể hòa vốn nếu mỗi câu trả lời đúng tiết kiệm được chi phí sửa chữa sai sót.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
|---|---|---|---|---|---|
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi về định nghĩa trong luật — chunk đã chứa đủ, graph không thêm giá trị |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên bị cáo và mức án nằm trong cùng bài báo, vector search đủ |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph thắng đậm | Flat không có cách nào suy ra Điều 251; Graph qua cầu Crime → Article → Clause |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph thắng (có lỗi) | Lấy được tội và Điều 255, nhưng chọn khoản 1 (7 năm) thay vì khoản 2 (chung thân) |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph thắng | Flat bỏ sót khoản 4 / tử hình; Graph lọc theo chất MDMA và đi đến khoản phù hợp |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Hòa (cả hai yếu) | Cả hai đều trả lời sai tên vụ; Graph có dữ kiện graph đúng nhưng LLM tạo tên khác |

**Quy luật:** Flat RAG đủ cho câu hỏi single-hop (Q1, Q2). GraphRAG vượt trội trên câu cross-KB (Q3–Q5) với recall 1.00 so với 0.00–0.60. Lỗi của GraphRAG (Q4, Q6) nằm ở bước suy luận ngưỡng khối lượng và trích xuất tên vụ, không phải ở graph.

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E2: Thiếu ngữ cảnh luật — chọn khoản sai

- **Hiện tượng:** GraphRAG trả lời đúng vụ án nhưng lấy **khoản có mức phạt thấp hơn** so với thực tế. Q4: câu trả lời ghi "07 năm" (khoản 1) trong khi gold answer yêu cầu "chung thân" (khoản 2).
- **Bằng chứng:**

Câu trả lời GraphRAG cho Q4:
> "Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa **07 năm** theo Điều 255 BLHS khoản 1."

Đáp án chuẩn Q4:
> "Dương Minh Tuấn (Hoàng Nato) bị bắt về hành vi tổ chộc sử dụng trái phép chất ma túy (Điều 255 BLHS); **khung cao nhất là tù 20 năm hoặc tù chung thân**."

Điều 255 có ít nhất 2 khoản: khoản 1 (cơ bản, phạt 1–5 năm + cải tạo không giam giữ 1–3 năm → tối đa ~8 năm, đúng là 07 năm trong trả lời), khoản 2 (tình tiết tăng nặng, phạt tù 5–10 năm + tù chung thân). Bản thân thuật ngữ "tổ chức sử dụng" là dấu hiệu tăng nặng, nên khoản 2 mới đúng.

```cypher
// Kiểm tra: Case "Hoàng Nato" liên quan chất gì?
MATCH (k:Case)-[:INVOLVES]->(s:Substance) WHERE k.name CONTAINS 'Hoàng Nato' OR k.summary CONTAINS 'Hoàng Nato'
RETURN k.name, s.name, k.summary;

// Kiểm tra: Điều 255 có những khoản nào, khoản nào MENTIONS chất nào?
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)-[r:MENTIONS]->(s:Substance)
RETURN cl.number, cl.penalty, s.name, type(r);
```

- **Nguyên nhân:** Thuật toán lọc khoản ở KG-3 chỉ giữ lại `khoản 1` + **các khoản MENTIONS một Substance mà vụ đó INVOLVES**. Vụ "Hoàng Nato" có thể không INVOLVES chất nào (hoặc chất không MENTIONS trong bất kỳ khoản nào của Điều 255). Kết quả: chỉ khoản 1 được thêm vào facts. LLM không có thông tin về ngưỡng khối lượng → tự suy ra mức phạt cơ bản. Lỗi nằm ở bước **KG-3 Cypher** chứ không phải ở thiết kế ontology.
- **Đề xuất sửa:** Bổ sung **tất cả** các khoản của Article vào facts khi câu hỏi hỏi về khung phạt, không chỉ khoản 1. Hoặc mở rộng điều kiện lọc: với câu hỏi chứa "tối đa", "cao nhất", "khung" → lấy khoản có `penalty` lớn nhất (hoặc khoản cuối cùng). Đánh đổi: prompt dài hơn, tốn thêm token nhưng đỡ thiếu ngữ cảnh.

---

### Lỗi E3: Trùng thực thể — Substance và Case không nhất quán

- **Hiện tượng:** Một chất trong thực tế xuất hiện dưới nhiều tên hơi khác nhau, tạo ra nhiều node `Substance` riêng biệt. Với Q6 (hỏi "vụ nào liên quan MDMA"), graph trả lời đúng cách xác định các vụ qua cạnh `INVOLVES`, nhưng tên vụ do LLM đặt **không khớp với tên trong benchmark** nên recall chỉ 0.33.
- **Bằng chứng:**

```cypher
// Tất cả Substance trong graph
MATCH (s:Substance) RETURN s.name ORDER BY s.name;

// Tất cả Case trong graph
MATCH (k:Case) RETURN k.name, k.doc_id ORDER BY k.name;
```

Output trích:
```
Substance: Amphetamine, Cocaine, Heroine, MDMA, cần sa, thuốc phiện...
Substance: Cần sa, Cần sa (viên nén màu trắng)   ← 2 node riêng cho cùng 1 chất!
Substance: Ketamine, MDMA, Methamphetamine...
```

Câu trả lời GraphRAG cho Q6:
> "Các vụ việc ... bao gồm: 1. Vụ vận chuyển ma túy từ Đức về Việt Nam - liên quan đến hơn 9,6kg MDMA. 2. Vụ góp tiền mua ma túy tại Hà Nội..."

Benchmark gold answer Q6:
> "Vụ Cái Quang Huy ... Vụ Lê Minh Thành ... Vụ Viện Pháp y tâm thần..."

**Cả hai đều đúng về bản chất** (cùng các vụ MDMA) nhưng tên khác → recall tính keyword đánh giá thấp.

- **Nguyên nhân:** `Substance` khóa theo `name` nhưng `find_substances` nhạy với hoa/thường và không gộp tên đồng nghĩa. Với Case, tên do LLM đặt hoàn toàn tự do — mỗi lần chạy có thể cho tên khác. `MERGE (k:Case {name: ...})` không nhận biết được 2 vụ là cùng 1 vụ nếu LLM đặt tên khác.
- **Đề xuất sửa:**
  - Với Substance: chuẩn hóa tên bằng `toLower(trim(name))` làm key, hoặc dùng danh sách đồng nghĩa.
  - Với Case: khóa theo tổ hợp `doc_id + date + location` thay vì `name`. Hoặc gán `doc_id` cho mọi node Case (đã làm), rồi khi trả lời câu hỏi tổng hợp, dùng `doc_id` thay vì `name` để định danh vụ.
  - Đánh đổi: thay đổi cách định danh vụ ảnh hưởng đến cấu trúc graph, cần cập nhật cả KG-2 và KG-3.

---

## 4. Kết luận (5 điểm)

**Khi nào dùng Knowledge Graph:**

GraphRAG đáng tiền khi câu hỏi **yêu cầu thông tin từ nhiều KB khác nhau** mà không có đoạn văn nào chứa đủ. Trong dataset này, Q3 (cross-KB đơn giản) Flat RAG gần như không trả lời được (recall=0.00, judge=0) trong khi GraphRAG đạt recall=1.00, judge=2. Với 6 câu hỏi, GraphRAG tăng chi phí trung bình mỗi câu thêm ~$0.00078 (= ×7). Nếu mỗi câu hỏi sai của Flat RAG gây thiệt hại > $0.00078 (ví dụ: cần hỏi lại, sửa chữa, mất uy tín), thì GraphRAG hòa vốn.

**Khi nào Flat RAG đủ:**

Câu hỏi **single-hop** (Q1, Q2) — câu trả lời nằm trong một đoạn văn duy nhất — Flat RAG đạt recall=1.00, judge=2, chi phí thấp. Thêm graph chỉ tăng chi phí mà không cải thiện chất lượng. Câu hỏi tổng hợp (Q6) — graph giúp xác định các vụ liên quan nhưng vẫn phụ thuộc LLM đặt tên.

**Số liệu dẫn chứng:**
- flat indexing: $0.00112 vs graph: $0.00933 (×8.3)
- flat/câu: $0.00013 vs graph/câu: $0.00091 (×7.0)
- flat mean judge: 1.00 vs graph mean judge: 1.67 (+67%)
- flat cross-KB recall trung bình: (0.00+0.00+0.60)/3 = 0.20 vs graph cross-KB: (1.00+0.67+1.00)/3 = 0.89

→ **Mỗi 1 câu cross-KB đúng của GraphRAG tránh được khoảng $0.00078 × số câu hỏi × tỉ lệ câu cross-KB** chi phí phát sinh. Với dataset 6 câu, 3/6 là cross-KB → chi phí tăng thêm đáng giá.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
48 passed in 0.17s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Trần Thanh Tuấn** (bị cáo trong vụ Q2, bên flat/câu Q2 đạt recall=1.00 nhờ chunk đã có đủ thông tin).

## Vấn đề gặp phải (không tính điểm)

Không có lỗi nghiêm trọng chưa giải quyết. Pipeline chạy end-to-end thành công. Sai số chủ yếu nằm ở E2 (chọn khoản) và E3 (trùng Substance) — đã phân tích ở mục 3.
