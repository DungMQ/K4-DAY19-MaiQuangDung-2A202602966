# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Mai Quang Dũng  **MSSV:** 2A202602966  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu dưới đây khớp 100% với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     57.2
graph       196     91958     4704   0.00933    124.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.57
graph       0.63   1.33     3329       64   0.00053     2.01
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00933 | ×8.33 |
| Indexing giây | 57.2s | 124.4s | ×2.17 |
| Mỗi câu: USD | $0.00013 | $0.00053 | ×4.08 |
| Mỗi câu: giây | 1.57s | 2.01s | ×1.28 |
| Mỗi câu: in_tok | 694 | 3329 | ×4.80 |

**Chi phí tăng thêm đến từ đâu?**
> - **Ở khâu Indexing:** Chi phí của GraphRAG tăng gấp 8.33 lần USD và 2.17 lần thời gian là do hệ thống phải gọi thêm 20 lượt gọi LLM (`gpt-4o-mini`) ở chế độ `json_mode` để trích xuất thực thể/quan hệ từ 20 bài báo tin tức (tiêu tốn thêm 35.886 input tokens và 4.704 output tokens), cộng với thời gian thực thi các truy vấn Cypher `MERGE` và ràng buộc duy nhất trong Neo4j, trong khi Flat RAG chỉ gọi API embedding văn bản thuần túy.
> - **Ở khâu Querying:** Chi phí mỗi câu hỏi của GraphRAG cao hơn 4.08 lần (USD) và số lượng input token tăng gấp 4.80 lần (3.329 so với 694 tokens). Nguyên nhân do prompt gửi lên LLM của GraphRAG ngoài top-3 chunks từ vector search còn phải nhồi thêm toàn bộ danh sách `facts` sinh ra từ quá trình duyệt đồ thị tri thức (multi-hop traversal). Độ trễ tăng nhẹ thêm 0.44s (+28%) tương ứng với thời gian chạy các truy vấn Cypher trên Neo4j.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | :---: | :---: | :---: | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một điều luật của Luật PCMT 2021 nên Flat RAG tìm đúng chunk là trả lời đủ ý. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Hai bị cáo lãnh án tử hình nằm trọn trong 1 bài báo xét xử của TAND TP.HCM, vector search bắt đúng ngữ cảnh. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat RAG trả về "Không đủ thông tin" do không có chunk nào chứa cả mức án (báo) lẫn khung luật (BLHS), trong khi GraphRAG đi qua cầu nối Crime lấy chuẩn xác Điều 251 khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 0.00 / 0 | Hòa | Dữ kiện graph đã trích xuất đúng Điều 255 khoản 1-4 nhưng LLM từ chối kết luận khung tối đa do bài báo chưa nêu chi tiết tình tiết định khung của vụ án. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | **Graph** | GraphRAG có recall vượt trội (0.80 so với 0.60), đối soát chính xác loại ma túy MDMA với khoản 4 Điều 250 BLHS để đưa ra khung phạt tù 20 năm, chung thân hoặc tử hình. |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai pipeline đều gom đúng 3 vụ án liên quan MDMA, nhưng recall = 0 do phép đo bắt buộc đối soát tên đối tượng trong khi mô hình trả về tên vụ việc. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Broken Bridge)

- **Hiện tượng:** Vụ án `"Vụ tông cảnh sát giao thông ở An Giang"` (`news-100260926112415229`) là node `Case` mồ côi trong đồ thị, hoàn toàn không có cạnh `[:CHARGED_WITH]` nối sang bất kỳ node `Crime` nào để đi sang KB Luật.
- **Bằng chứng:** Truy vấn Cypher kiểm tra các case bị gãy cầu nối:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() 
RETURN k.name AS case_name, k.doc_id AS doc_id;
```

```
╒═══════════════════════════════════════════╤══════════════════════════╕
│case_name                                  │doc_id                    │
╞═══════════════════════════════════════════╪══════════════════════════╡
│"Vụ tông cảnh sát giao thông ở An Giang"   │"news-100260926112415229" │
└───────────────────────────────────────────┴──────────────────────────┘
```

- **Nguyên nhân:** Nằm ở khâu trích xuất tin tức bằng LLM và hàm `link_entity`. Bài báo này phản ánh sự việc đối tượng đang vận chuyển ma túy thì tông xe vào CSGT khi bị dừng kiểm tra. Do vụ việc đang ở giai đoạn tạm giữ ban đầu, bài báo chưa có cụm từ định danh tội danh chính thức theo Bộ luật Hình sự. LLM trả về tên hành vi tự do hoặc chuỗi tội danh không đủ độ tương đồng 0.8 so với danh mục chuẩn BLHS, dẫn đến `link_entity` trả về `None` và không tạo được quan hệ `CHARGED_WITH`.
- **Đề xuất sửa:** Trong hàm `extract_news_cases`, nếu không trích xuất được tội danh chính xác, cho phép tạo một cạnh suy diễn tạm thời qua node `Substance` thu giữ được (`[:INVOLVES]`) để liên kết sang các Điều luật ma túy phổ biến, hoặc bổ sung cơ chế fallback gắn tội danh nghi vấn mặc định dựa trên hành vi (ví dụ: vận chuyển trái phép chất ma túy).

---

### Lỗi E3: Trùng thực thể (Entity Duplication)

- **Hiện tượng:** Cùng một chất ma túy ngoài đời thực nhưng bị tạo thành nhiều node `Substance` khác nhau trong đồ thị (như `'Ketamine'` và `'ketamine'`, `'Methamphetamine'` và `'methamphetamine'`, `'MDMA'` và `'thuốc lắc'`).
- **Bằng chứng:** Truy vấn Cypher kiểm tra danh sách node `Substance`:

```cypher
MATCH (s:Substance) 
RETURN s.name AS name 
ORDER BY toLower(s.name);
```

```
['Amphetamine', 'chất ma túy', 'Cocaine', 'côca', 'cần sa', 'etomidate', 'Heroine', 'Ketamine', 'ketamine', 'ma túy', 'ma túy tổng hợp', 'MDMA', 'Methamphetamine', 'methamphetamine', 'thuốc lắc', 'thuốc phiện', 'XLR-11']
```

- **Nguyên nhân:** Nằm ở khâu thiết kế ontology và hàm ghi dữ liệu vào Neo4j:
  1. Ràng buộc duy nhất `CONSTRAINT FOR (n:Substance) REQUIRE n.name IS UNIQUE` trong Neo4j có tính phân biệt chữ hoa - chữ thường (case-sensitive). KB Luật tạo ra `'Ketamine'`, còn LLM khi đọc tin tức lại trích xuất `'ketamine'` chữ thường, dẫn đến lệnh `MERGE` tạo thành 2 node riêng biệt.
  2. Hệ thống chưa có tầng chuẩn hóa từ đồng nghĩa (Synonym Resolution) giữa tên khoa học trong Luật (`MDMA`) và tên lóng ngoài đời trong báo chí (`thuốc lắc`).
- **Đề xuất sửa:** 
  1. Trước khi `MERGE` node `Substance` vào Neo4j, chuẩn hóa `s.name` về chữ hoa chuẩn tắc bằng hàm `normalize_substance` (tương tự như `link_entity`).
  2. Bổ sung một bảng tra cứu từ đồng nghĩa (alias map): ví dụ map `"thuốc lắc"` $\to$ `"MDMA"`, `"đá"` $\to$ `"Methamphetamine"`, và gộp các từ chung chung như `"ma túy"`, `"ma túy tổng hợp"` vào một nhóm tổng quát.

---

### Lỗi E4: Phép đo sai / Cứng nhắc trong đánh giá (Evaluation Metric Mismatch)

- **Hiện tượng:** Ở câu hỏi Q6 (`aggregation`), cả Flat RAG và GraphRAG đều trả lời chính xác và đầy đủ 3 vụ án có liên quan đến ma túy MDMA, được giám khảo LLM chấm điểm đạt (`judge = 1`), tuy nhiên chỉ số `recall` từ khóa máy móc lại bị tính là `0.00`.
- **Bằng chứng:** Trích nguyên văn kết quả câu Q6 trong `ket_qua_benchmark_kg.txt`:

```
--- Q6 [aggregation] graph recall=0.00 judge=1 2.43s
Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:

1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Tổng khối lượng hơn 9,6kg MDMA.
2. Vụ góp tiền mua ma túy tại Hà Nội: Bao gồm 5 viên MDMA.
3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Có liên quan đến 0,686g MDMA.

Ngoài ra, Điều 252 BLHS cũng đề cập đến ma túy MDMA.
```

So với `must_include` trong `data/benchmark_kg.json`:
```json
"must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
```

- **Nguyên nhân:** Lỗi nằm ở chính thiết kế của phép đo `keyword_recall`. Câu hỏi Q6 hỏi: *"Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"*. GraphRAG đã tổng hợp và trả lời trực diện theo tên các vụ án thực tế (Vụ chuyển từ Đức về VN, Vụ góp tiền mua ma túy ở Hà Nội, Vụ Sầm Sơn), hoàn toàn đúng ngữ nghĩa của câu hỏi. Tuy nhiên, metric `keyword_recall` lại so khớp chuỗi cứng nhắc với tên của các bị cáo cụ thể (`Cái Quang Huy`, `Lê Minh Thành`) và địa danh `Pháp y tâm thần`, khiến câu trả lời đúng bị chấm 0% recall.
- **Đề xuất sửa:** 
  - Điều chỉnh `must_include` trong bộ benchmark linh hoạt hơn (chấp nhận cả tên vụ án hoặc từ khóa tương đương).
  - Hoặc trong prompt của Agent, bổ sung chỉ dẫn: *"Khi liệt kê các vụ việc, hãy nêu kèm họ tên của các đối tượng/bị can chủ chốt có trong vụ án"*.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào Flat RAG là đủ:** Với các bài toán tìm kiếm và hỏi đáp cục bộ (single-hop), nơi thông tin cần trả lời nằm gọn trong một văn bản cụ thể (như Q1 hỏi định nghĩa trong Luật đạt `recall = 1.00, judge = 2`, Q2 hỏi mức án trong một bài báo đạt `recall = 1.00, judge = 2`), Flat RAG là giải pháp tối ưu vượt trội. Nó nhanh hơn 28%, rẻ hơn gấp 4.08 lần chi phí truy vấn mỗi câu ($0.00013 so với $0.00053) và tiết kiệm đến 88% chi phí dựng hệ thống ban đầu ($0.00112 so với $0.00933) mà không cần duy trì hạ tầng cơ sở dữ liệu đồ thị phức tạp.
> - **Khi nào nên dùng Knowledge Graph (GraphRAG):** Knowledge Graph trở nên bắt buộc và hoàn toàn đáng tiền khi câu hỏi đòi hỏi **tổng hợp tri thức xuyên nguồn (cross-silo/cross-document)** mà không một đoạn văn bản đơn lẻ nào chứa đầy đủ thông tin (điển hình như Q3: ghép nối mức án trong bài báo với điều luật và khung hình phạt trong BLHS). Ở kịch bản này, Flat RAG hoàn toàn thất bại (`recall = 0.00, judge = 0` do chunking bị phân mảnh), trong khi GraphRAG đạt `recall = 1.00, judge = 2` nhờ khả năng truy vết đường đi qua node cầu nối `Crime`. Nhờ đó, GraphRAG nâng recall trung bình toàn bài từ 0.43 lên 0.63 và điểm judge từ 1.00 lên 1.33.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.08s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j đã chụp và lưu tại:
- `report/img/kg_count.png`: Ảnh bảng đếm node theo loại (Q-A), hiển thị đầy đủ 7 labels.
- `report/img/kg_cross_kb.png`: Ảnh đồ thị trực quan đường nối 2 KB qua node cầu nối Crime (Q-B), hiển thị cả thanh truy vấn và Results overview.
- `report/img/kg_my_case.png`: Ảnh đồ thị truy vấn đường đi của một vụ án cụ thể (Q-D).

Người đã chọn cho `kg_my_case.png`: **Dương Minh Tuấn** (biệt danh Hoàng Nato, liên kết tới 4 vụ án và các Điều 249, Điều 251, Điều 255 BLHS).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: Không có lỗi nghẽn nào chưa xử lý; hệ thống đã hoàn thành 100% các bước theo chuẩn yêu cầu của Lab Guide.
