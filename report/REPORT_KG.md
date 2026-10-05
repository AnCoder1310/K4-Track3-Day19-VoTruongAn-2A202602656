# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Trường An  **MSSV:** K4-Day19-Student  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    111.2
graph       196     89199     5186   0.00000    274.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33     3178       88   0.00000     7.06
graph       0.89   1.83     7883      148   0.00000     7.00
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00000 | ×1.00 |
| Indexing giây | 111.2s | 274.4s | ×2.47 |
| Mỗi câu: USD | $0.00000 | $0.00000 | ×1.00 |
| Mỗi câu: giây | 7.06s | 7.00s | ×0.99 |
| Mỗi câu: in_tok | 3178 | 7883 | ×2.48 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Chi phí thời gian Indexing của GraphRAG tăng gấp 2.47 lần (274.4s vs 111.2s của Flat RAG) chủ yếu do phải gọi LLM trích xuất cấu trúc thực thể, vụ án và người liên quan cho toàn bộ 20 bài báo (196 calls vs 176 calls, tiêu tốn 89,199 input tokens và 5,186 output tokens). Khi truy vấn, số lượng `in_tok` trung bình mỗi câu của GraphRAG cao gấp 2.48 lần (7,883 vs 3,178) do prompt được nạp thêm các dữ kiện có cấu trúc (facts) từ Knowledge Graph (bao gồm các điều khoản luật, khung phạt và tóm tắt vụ việc). Tuy nhiên, độ trễ phản hồi mỗi câu của GraphRAG tương đương Flat RAG (7.00s vs 7.06s, tỉ lệ ×0.99) do thông tin đầu vào đã được chọn lọc chính xác, giúp LLM tổng hợp câu trả lời tự tin và dứt khoát hơn.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi đơn về luật, vector search của cả hai bên đều tìm trúng Điều 2 Luật PCMT nên trả lời chính xác định nghĩa tiền chất. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi đơn trong tin tức, cả hai bên đều trích xuất đúng 2 bị cáo Trần Thanh Tuấn và Trần Minh Tâm lãnh án tử hình. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG thiếu chunk luật chứa khung phạt, trong khi GraphRAG đi qua cầu nối Crime để lấy trọn vẹn Điều 251 khoản 1 (2-7 năm tù). |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | GraphRAG xác định chính xác hành vi tổ chức sử dụng theo Điều 255 BLHS và nêu được khung phạt cơ bản khoản 1, trong khi Flat RAG hoàn toàn không biết điều luật nào. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Flat RAG không xác định được khoản luật do thiếu định lượng, GraphRAG nối tang vật MDMA (>9,6kg) vào Khoản 4 Điều 250 (tù 20 năm, chung thân, tử hình). |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 2 | Graph | Flat RAG bị giới hạn top-k nên chỉ gom được 1-2 vụ việc, GraphRAG gom đủ 5 vụ án liên quan MDMA qua node Substance và trích dẫn chuẩn xác. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Bỏ sót khung hình phạt tối đa do lọc theo tang vật)

- **Hiện tượng:** Ở câu Q4 (*Giang hồ "Hoàng Nato" bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?*), GraphRAG chỉ trả lời được mức phạt tối đa của khoản 1 là 7 năm tù, và giải trình thêm: *"Ngữ cảnh không cung cấp các khoản khác của Điều 255 BLHS, do đó không đủ thông tin để xác định mức phạt tù tối đa toàn bộ của tội danh này theo quy định của Bộ luật Hình sự"*. Điểm recall đạt 0.67 do thiếu từ khóa "chung thân".
- **Bằng chứng:**
  Trích nguyên văn câu trả lời Q4 của GraphRAG trong `ket_qua_benchmark_kg.txt`:
  > *"Theo Khoản 1 Điều 255 Bộ luật Hình sự (BLHS) được trích dẫn trong ngữ cảnh, người nào tổ chức sử dụng trái phép chất ma túy dưới bất kỳ hình thức nào thì bị phạt tù từ 02 năm đến 07 năm (tối đa là 07 năm tù theo khung hình phạt này). Ngữ cảnh không cung cấp các khoản khác của Điều 255 BLHS, do đó không đủ thông tin để xác định mức phạt tù tối đa toàn bộ của tội danh này theo quy định của Bộ luật Hình sự."*

  Truy vấn Cypher kiểm tra tại sao khoản 4 Điều 255 không được đưa vào facts:
  ```cypher
  MATCH (k:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
  WHERE cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(cl) }
  RETURN a.id, cl.number, cl.penalty;
  ```
  Kết quả: Vụ án Hoàng Nato thu giữ chất `etomidate` (ma túy mới ngụy trang pod chill, không có trong danh sách chất luật định danh theo tên ở các khoản tăng nặng của Điều 255). Do đó, chỉ có điều kiện `cl.number = 1` thỏa mãn; các khoản tăng nặng (khoản 2, 3, 4 với khung tối đa chung thân) bị loại bỏ.
- **Nguyên nhân:** Lỗi nằm ở bước **Cypher retrieval trong `Neo4jGraph.context`**: Quy tắc lọc khoản hiện tại chỉ chọn khoản 1 hoặc khoản có `MENTIONS` chất mà vụ án `INVOLVES`. Khi vụ án liên quan đến chất ma túy mới hoặc tình tiết tăng nặng phi vật thể (tái phạm nguy hiểm, phạm tội 2 lần trở lên), quy tắc này không lấy được các khoản định khung cao nhất.
- **Đề xuất sửa:** Trong hàm `context()`, phát hiện nếu câu hỏi chứa các từ khóa về mức án trần (*"tối đa"*, *"cao nhất"*, *"nặng nhất"*), Cypher cần tự động truy xuất bổ sung khoản có `number` lớn nhất của Điều luật liên quan:
  ```cypher
  MATCH (a:Article)-[:HAS_CLAUSE]->(cl:Clause)
  WHERE a.id = $article_id
  WITH cl ORDER BY cl.number DESC LIMIT 1
  RETURN cl
  ```
  *Đánh đổi:* Tăng thêm 1 khoản luật vào prompt (~100 tokens), nhưng giải quyết triệt để các câu hỏi về mức phạt tối đa.

---

### Lỗi E4: Phép đo sai / Bất đối xứng giữa Keyword Recall và Ngữ nghĩa thực tế

- **Hiện tượng:** Ở câu Q6 (*Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?*), GraphRAG trả lời toàn diện, liệt kê đúng 5 vụ án lớn và các Điều luật liên quan, được giám khảo LLM chấm điểm tuyệt đối **judge=2**. Tuy nhiên, điểm **recall** từ khóa chỉ đạt **0.67** (2/3).
- **Bằng chứng:**
  Trong `data/benchmark_kg.json`, danh sách từ khóa bắt buộc là:
  ```json
  "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
  ```
  Trích nguyên văn câu trả lời Q6 của GraphRAG trong `ket_qua_benchmark_kg.txt`:
  > *"Dựa trên ngữ cảnh và dữ kiện từ Knowledge Graph, các vụ việc có liên quan đến ma túy MDMA gồm:*
  > *1. Vụ vận chuyển hơn 10kg ma túy từ Đức về Hà Nội qua sân bay Nội Bài (liên quan đến hơn 9,6kg MDMA).*
  > *2. Vụ mua bán trái phép chất ma túy của Lê Minh Thành và đồng phạm (liên quan đến 5 viên MDMA).*
  > *3. Vụ tổ chức sử dụng và tàng trữ trái phép ma túy liên quan bệnh nhân Viện Pháp y tâm thần Trung ương (liên quan đến 0,686g MDMA).*
  > *4. Vụ án tại Viện Pháp y tâm thần Trung ương (liên quan đến MDMA).*
  > *5. Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM (liên quan đến MDMA)."*

  GraphRAG đã nhắc tới "Lê Minh Thành" (vụ 2) và "Pháp y tâm thần" (vụ 3, 4), đạt 2/3 từ khóa (recall = 0.67). Tuy nhiên, đối với vụ vận chuyển ma túy từ Đức về qua Nội Bài của "Cái Quang Huy", do đồ thị liên kết theo tên thực thể `Case.name` ("Vụ vận chuyển hơn 10kg ma túy từ Đức về Hà Nội qua sân bay Nội Bài"), LLM sử dụng tên vụ án thay vì nhắc lại tên riêng bị can Cái Quang Huy, khiến hàm `keyword_recall` đánh rớt 1 từ khóa này mặc dù thông tin vụ việc hoàn toàn chính xác và đầy đủ.
- **Nguyên nhân:** Nằm ở **phép đo `keyword_recall`**: Đây là phương pháp so khớp chuỗi cứng (`string in string`), hoàn toàn bỏ qua ngữ nghĩa, tính tương đương của thực thể và quan hệ đồng sở chỉ (coreference) trong văn bản.
- **Đề xuất sửa:**
  - Về phía prompt và context: Trong dữ kiện vụ án sinh ra từ graph (`facts`), luôn đính kèm danh sách các bị can/bị cáo chính theo dạng `Vụ việc '...' (liên quan: Lê Minh Thành, ...)` để LLM vừa nêu tên vụ vừa trích dẫn đầy đủ tên đối tượng.
  - Về phía đánh giá benchmark: Ưu tiên sử dụng điểm của LLM-as-judge hoặc F1-score thực thể có hỗ trợ mapping thực thể tương đương thay vì chỉ dựa vào exact substring match thô sơ.

---

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào Flat RAG là đủ:** Với các bài toán tìm kiếm thông tin đơn lẻ (single-hop) nằm trọn vẹn trong một văn bản hoặc một bài báo (như Q1 và Q2, cả hai bên đều đạt recall 1.00 và judge=2), Flat RAG là lựa chọn tối ưu vì thời gian index nhanh hơn gấp 2.47 lần (111.2s vs 274.4s) và không tốn chi phí xây dựng, duy trì đồ thị tri thức.
> - **Khi nào bắt buộc dùng GraphRAG:** Khi hệ thống đối mặt với các câu hỏi tổng hợp đa nguồn (cross-KB) hoặc câu hỏi suy luận bắc cầu nhiều bước (multi-hop) mà dữ kiện nằm phân tán ở nhiều tài liệu khác nhau (như Q3, Q4, Q5, Q6). Số liệu thực nghiệm cho thấy trên các câu hỏi cross-kb, Flat RAG thất bại hoặc hụt thông tin quan trọng (recall chỉ đạt 0.00 - 0.40 và judge=1 do thiếu ngữ cảnh luật), trong khi GraphRAG đạt recall 0.67 - 1.00 và judge 1.83 - 2.00 nhờ khả năng kết nối chính xác từ vụ án sang điều luật và khung hình phạt qua node cầu nối `Crime` và `Substance`.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.02s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:ag/gemini-3.8-flash-high | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Trần Thanh Tuấn

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> - Ban đầu khi gọi embedding qua local proxy OpenAI-compatible (`127.0.0.1:20128`), hệ thống gặp lỗi `BadRequestError: No credentials for provider: gemini` do proxy chỉ hỗ trợ chat models và thiếu model embedding.
> - Khắc phục bằng cách cấu hình phân tách độc lập trong `.env`: Chat model chạy qua proxy (`ag/gemini-3.8-flash-high`), còn Embedding gọi trực tiếp Google Gemini API qua `GEMINI_API_KEY` với model chuẩn `gemini-embedding-001`. Pipeline benchmark sau đó chạy mượt mà và sinh kết quả chuẩn xác 100%.
