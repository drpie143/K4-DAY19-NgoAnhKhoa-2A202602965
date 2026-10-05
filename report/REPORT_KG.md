# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Ngô Anh Khoa  **MSSV:** 2A202602965  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     55.6
graph       196     91958     4670   0.00931    123.6

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.44
graph       0.67   1.50     4515       61   0.00071     1.78
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00931 | ×8.31 |
| Indexing giây | 55.6s | 123.6s | ×2.22 |
| Mỗi câu: USD | $0.00013 | $0.00071 | ×5.46 |
| Mỗi câu: giây | 1.44s | 1.78s | ×1.24 |
| Mỗi câu: in_tok | 694 | 4515 | ×6.51 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí Indexing tăng ~8.3 lần chủ yếu do phải gọi LLM trích xuất thực thể và quan hệ JSON từ 20 bài báo tin tức (thêm 20 lần gọi chat model và 4.670 output tokens, thay vì chỉ tốn embedding thuần túy như Flat RAG). Chi phí mỗi câu hỏi khi truy vấn tăng ~5.5 lần do prompt của GraphRAG phải gánh thêm danh sách các dữ kiện có cấu trúc (facts) từ đồ thị tri thức (input tokens trung bình tăng từ 694 lên 4.515 tokens/câu), tuy nhiên độ trễ chỉ tăng nhẹ 24% (1.44s lên 1.78s).

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 0.00 / 0 | Flat | Khái niệm định nghĩa nằm trọn vẹn trong một chunk luật, Flat RAG tìm trúng ngay trong khi GraphRAG không tìm thấy Case hạt giống nên không kích hoạt multi-hop. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin mức án nằm trong cùng một bài báo cụ thể, cả hai pipeline đều tìm được đoạn văn bản liên quan để trả lời chính xác. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Cần kết hợp tin tức (Lê Minh Thành 36 tháng tù) và luật (Điều 251 khoản 1: 2-7 năm), Flat RAG thiếu ngữ cảnh luật nên báo không đủ thông tin, GraphRAG đi xuyên node cầu nối Crime trả lời hoàn hảo. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Cần thông tin hành vi của Hoàng Nato từ tin tức và mức phạt tối đa từ Điều 255 BLHS, Flat RAG thất bại vì thiếu liên kết luật trong khi GraphRAG đi qua graph lấy chính xác khung cao nhất (chung thân). |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ nêu được khung phạt chung mà không chỉ ra số Điều 250 và khoản 4, trong khi GraphRAG đi qua graph xác định chính xác Điều 250 khoản 4 đối với khối lượng MDMA. |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai pipeline đều tổng hợp được 3 vụ liên quan MDMA nhưng điểm recall máy móc bằng 0 do câu trả lời dùng tên tóm tắt vụ việc thay vì lặp lại đúng từ khóa trong danh sách must_include. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (và cách khắc phục bộ lọc khoản hình phạt)

- **Hiện tượng:** Ở các câu hỏi yêu cầu xác định khung hình phạt tối đa (như Q4: Hoàng Nato) hoặc khung phạt dựa trên loại ma túy cụ thể (Q5: Cái Quang Huy), GraphRAG sẽ trả lời sai hoặc thiếu nếu thuật toán trích xuất facts chỉ giữ lại `khoản 1` và các khoản `MENTIONS` chất ma túy.
- **Bằng chứng:**
  Truy vấn kiểm tra các khoản của Điều 255 BLHS liên quan vụ Hoàng Nato:

```cypher
MATCH (k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number, cl.penalty, [(cl)-[:MENTIONS]->(s) | s.name] AS substances;
```

```
╒═══════════╤══════════════════════════════════════╤════════════╕
│"cl.number"│"cl.penalty"                          │"substances"│
╞═══════════╪══════════════════════════════════════╪════════════╡
│1          │"phạt tù từ 02 năm đến 07 năm"        │[]          │
│2          │"phạt tù từ 07 năm đến 15 năm"        │[]          │
│3          │"phạt tù từ 15 năm đến 20 năm"        │[]          │
│4          │"phạt tù 20 năm hoặc tù chung thân"   │[]          │
│5          │""                                    │[]          │
└───────────┴──────────────────────────────────────┴────────────┘
```

- **Nguyên nhân:** Nằm ở câu truy vấn Cypher trong hàm `context` (bước KG-3). Ở Điều 255 BLHS (Tội tổ chức sử dụng), các khoản tăng nặng (khoản 2, 3, 4) quy định dựa trên hậu quả (số người chết, tỷ lệ thương tật) chứ không liệt kê tên chất cụ thể, do đó danh sách `substances` của các khoản này là rỗng. Nếu chỉ lấy `cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(s)<-[:MENTIONS]-(cl) }`, hàm sẽ chỉ lấy được khoản 1 (phạt từ 02 đến 07 năm) và bỏ sót hoàn toàn khoản 4 (khung tối đa: chung thân).
- **Đề xuất sửa:** Mở rộng điều kiện lọc khoản trong Cypher để bổ sung các khoản có mức án kịch khung:
  `cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(cl) } OR cl.penalty CONTAINS 'chung thân' OR cl.penalty CONTAINS 'tử hình' OR cl.number >= 4`.
  Đánh đổi: Prompt sẽ dài thêm khoảng 150–200 tokens cho mỗi điều luật liên quan, nhưng đảm bảo LLM có đủ ngữ cảnh để trả lời chính xác khung phạt tối đa.

---

### Lỗi E4: Phép đo sai (Metric Keyword Recall quá cứng nhắc)

- **Hiện tượng:** Ở câu Q6 (loại aggregation), câu trả lời của GraphRAG được giám khảo LLM chấm đạt (judge = 1, xác định đủ 3 vụ án có liên quan MDMA), nhưng điểm keyword recall lại nhận giá trị 0.00.
- **Bằng chứng:**
  - *Must include keywords trong benchmark:* `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`.
  - *Câu trả lời thực tế của GraphRAG (trích từ `ket_qua_benchmark_kg.txt`):*
    ```
    Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:
    1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Tổng khối lượng hơn 9,6kg MDMA.
    2. Vụ góp tiền mua ma túy tại Hà Nội: Có liên quan đến 5 viên MDMA.
    3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Có liên quan đến 0,686g MDMA.
    Tất cả các vụ việc này đều có sự liên quan đến ma túy MDMA.
    ```
- **Nguyên nhân:** Nằm ở phép đo `keyword_recall` trong `bench_kg.py`. Hàm này thực hiện `k.lower() in answer.lower()` một cách cơ học. Cả 3 vụ việc GraphRAG liệt kê ở trên chính là 3 vụ việc tương ứng với 3 từ khóa chuẩn: vụ Cái Quang Huy (vận chuyển từ Đức về VN), vụ Lê Minh Thành (góp tiền mua tại Hà Nội), và vụ Viện Pháp y tâm thần (tổ chức sử dụng tại Sầm Sơn). Tuy nhiên, vì GraphRAG tổng hợp theo tên vụ án (`Case.name`) thay vì lặp lại tên bị can/cơ quan, hàm kiểm tra chuỗi thô đã đánh rớt toàn bộ điểm recall xuống 0.00.
- **Đề xuất sửa:** Thay vì chỉ so khớp chuỗi con cố định, cần chuẩn hóa danh sách thực thể hoặc bổ sung các alias/thuộc tính đại diện của vụ án (như tên vụ, địa điểm) vào tập từ khóa hợp lệ, hoặc sử dụng metric ngữ nghĩa (semantic similarity / LLM-as-a-judge) làm thước đo chính cho các câu hỏi tổng hợp.

---

## 4. Kết luận (5 điểm)

Từ số liệu đo lường thực tế trên bộ dữ liệu ma túy:
1. **Khi nào Flat RAG là đủ:** Đối với các câu hỏi tra cứu đơn nguồn (*single-hop*), nơi câu trả lời nằm trọn vẹn trong một điều luật hoặc một bài báo (như Q1 và Q2), Flat RAG đạt điểm tuyệt đối (recall 1.0, judge = 2) với chi phí cực rẻ ($0.00112 để index và $0.00013 mỗi câu hỏi) cùng độ trễ chỉ 1.44s. Lúc này việc dựng Knowledge Graph là lãng phí và không cần thiết.
2. **Khi nào nên dùng Knowledge Graph (GraphRAG):** Khi nghiệp vụ đòi hỏi trả lời các câu hỏi xuyên nguồn (*cross-kb* hoặc *multi-hop*) như Q3, Q4, Q5 (nối giữa dữ kiện hành vi trong tin tức với điều khoản quy định trong luật). Ở các câu hỏi này, Flat RAG hoàn toàn bất lực (recall = 0.00, judge = 0) do thông tin bị phân mảnh ở các tài liệu khác nhau. GraphRAG đã nâng recall từ 0.00 lên 1.00 và điểm judge từ 0 lên 2.
3. **Đánh đổi kinh tế:** GraphRAG tốn chi phí dựng hệ thống cao hơn gấp 8.3 lần ($0.00931 vs $0.00112) và chi phí mỗi câu hỏi cao hơn 5.5 lần ($0.00071 vs $0.00013) do kích thước prompt dài hơn. Do đó, chỉ nên đầu tư Knowledge Graph khi bài toán thực sự có cấu trúc dữ liệu liên kết chéo và yêu cầu tính chính xác cao về mặt suy luận quan hệ.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.05s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 24 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00077. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ án vận chuyển ma túy MDMA qua đường hàng không từ Đức về Việt Nam).

## Vấn đề gặp phải (không tính điểm)

- Ban đầu trên môi trường Windows PowerShell mặc định dùng mã hóa ký tự `cp1252` gây ra lỗi `UnicodeEncodeError` khi script in tiếng Việt ra màn hình. Đã giải quyết triệt để bằng cách thiết lập `$env:PYTHONIOENCODING="utf-8"`.
- Giao diện Neo4j Browser phiên bản 5.x sử dụng editor CodeMirror 6 (`.cm-content`) thay vì Monaco Editor, script chụp ảnh tự động đã được tinh chỉnh để focus và nhập truy vấn chuẩn xác thông qua Playwright.
