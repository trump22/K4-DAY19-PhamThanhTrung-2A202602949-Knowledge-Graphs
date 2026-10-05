# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phạm Thanh Trung  **MSSV:** 2A2202602949  **Ngày:** 05/10/2026

Chạy `python bench_kg.py --judge` trên Python 3.11.9, Neo4j 5.26.31. Hai pipeline dùng cùng Gemini 3.5 Flash-Lite, Gemini Embedding 001, top_k=3, chunk_size=800 và 176 chunk từ 18 Điều luật + 20 bài báo. Graph cuối có **202 node / 379 cạnh**, gồm Article 18, Clause 99, Crime 13, Case 12, Person 44, Substance 11, Location 5. Ontology dùng gợi ý, không xét bonus.

## 1. Chi phí

Hai bảng dưới được chép nguyên từ `ket_qua_benchmark_kg.txt` của lần chạy thành công:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     34851        0   0.00523    155.0
graph       196     69470     5720   0.02991    236.9

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      728       77   0.00041     4.13
graph       0.94   1.83     5626      160   0.00208     4.63
```

| Chỉ số | Flat | Graph | Graph / Flat |
|---|---:|---:|---:|
| Indexing USD | 0.00523 | 0.02991 | ≈5.72× |
| Indexing giây | 155.0 | 236.9 | ≈1.53× |
| Mỗi câu: USD | 0.00041 | 0.00208 | ≈5.07× |
| Mỗi câu: giây | 4.13 | 4.63 | ≈1.12× |
| Mỗi câu: input token | 728 | 5626 | ≈7.73× |

Tỉ lệ tính từ số đã làm tròn trong bảng nên là xấp xỉ. Index Graph gồm index vector của Flat cộng 20 lần trích tin bằng LLM: tăng 34.619 input token, 5.720 output token và khoảng 0.02468 USD. Khi hỏi, prompt Graph có nhiều dữ kiện luật và vụ việc hơn nên input token trung bình tăng từ 728 lên 5.626; phần judge được chương trình đo riêng, không cộng vào chi phí pipeline.

USD là **ước tính theo giá Standard paid tier**, không phải hóa đơn thực tế: Gemini 3.5 Flash-Lite 0.30 USD/triệu input token, 2.50 USD/triệu output token; Embedding 001 0.15 USD/triệu input token. Nguồn: [bảng giá Google](https://ai.google.dev/gemini-api/docs/pricing) và [công bố Embedding 001](https://developers.googleblog.com/gemini-embedding-available-gemini-api/), đối chiếu ngày 05/10/2026. Free tier có thể không phát sinh khoản thanh toán này.

Endpoint embedding tương thích OpenAI trả `usage=None`; bộ đo gọi `countTokens` của Google để lấy token thay vì ghi 0. `calls` vẫn đếm các request embedding/chat sinh kết quả theo định nghĩa mẫu, không đếm request đếm token; thời gian thực bao gồm request đếm token. Chat được giãn tối thiểu 4.2 giây giữa các lần bắt đầu để đáp ứng 15 request/phút; latency có cả thời gian chờ quota, không phải latency thuần của model. Không sửa `bench_kg.py`, test hay logic base RAG.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
|---|---|---|---|---|---|
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa; Flat tiết kiệm hơn | Định nghĩa tiền chất nằm gọn trong chunk luật; Graph thêm số Điều nhưng không tăng điểm. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa; Flat tiết kiệm hơn | Cả hai tìm đúng Trần Thanh Tuấn và Trần Minh Tâm tử hình. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat chỉ biết 36 tháng và tội danh; Graph nối thêm Điều 251 và khung 02–07 năm. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph tăng recall, vẫn chưa đủ | Graph thêm Điều 255 nhưng thiếu khoản 4 để trả mức tối đa; cả hai judge=1. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Graph nối khối lượng MDMA với khoản 4 Điều 250 và khung 20 năm/chung thân/tử hình. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | Graph về độ bao phủ | Graph có tên đầy đủ và vụ Viện Pháp y; Flat chỉ gọi Huy/Thành, lặp hai kiện hàng và thiếu vụ Viện Pháp y. Judge=2 của Flat cần kiểm tra, xem E4. |

Trên Q3–Q5, recall trung bình tăng từ khoảng 0.36 lên 0.89. Đây là nhóm được lợi từ đường nối hai KB; hai câu single-hop không tăng điểm. Sáu câu chưa đủ để suy rộng cho mọi corpus; judge cũng có sai sót như E4.

## 3. Phân tích lỗi

### Lỗi E2: bỏ khoản luật về mức phạt tối đa

- **Hiện tượng:** Q4 Graph biết đúng hành vi và Điều 255 nhưng không trả được khung tối đa 20 năm hoặc tù chung thân. Điểm recall=0.67, judge=1.
- **Bằng chứng câu trả lời:** Q4, pipeline graph, trong file kết quả:

> **Mức phạt tù tối đa:** Ngữ cảnh không đủ thông tin để xác định mức phạt tù tối đa cụ thể (vì không nêu rõ các tình tiết định khung tăng nặng tại các khoản 2, 3, 4 của Điều luật).

Graph có đủ các khoản. Truy vấn:

```cypher
MATCH (:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS number, split(cl.text,'\n')[0] AS first_line
ORDER BY number;
```

| number | Phần đầu text trả về |
|---:|---|
| 1 | Người nào tổ chức sử dụng trái phép chất ma túy dưới bất kỳ hình thức nào, thì bị phạt tù từ 02 năm đến 07 năm. |
| 2 | Phạm tội thuộc một trong các trường hợp sau đây, thì bị phạt tù từ 07 năm đến 15 năm: |
| 3 | Phạm tội thuộc một trong các trường hợp sau đây, thì bị phạt tù từ 15 năm đến 20 năm: |
| 4 | Phạm tội thuộc một trong các trường hợp sau đây, thì bị phạt tù 20 năm hoặc tù chung thân: |
| 5 | Người phạm tội còn có thể bị phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng, phạt quản chế, cấm cư trú từ 01 năm đến 05 năm hoặc tịch thu một phần hoặc toàn bộ tài sản. |

Chạy đúng bộ lọc KG-3 cho các vụ của Hoàng Nato:

```cypher
MATCH (:Person {name:'Dương Minh Tuấn'})-[:INVOLVED_IN]->(k:Case)
-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(a:Article {id:'Điều 255 BLHS'})
-[:HAS_CLAUSE]->(cl:Clause)
WHERE cl.number=1 OR EXISTS {
  MATCH (k)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(cl)
}
RETURN DISTINCT cl.number AS kept_clause ORDER BY kept_clause;
```

```text
kept_clause
1
```

- **Nguyên nhân:** KG-3 giữ khoản 1 hoặc khoản có cạnh MENTIONS tới chất của vụ. Điều 255 khoản 4 nêu tình tiết định khung, không nêu tên chất nên bị loại. Đây là thiếu context ở Cypher, không phải thiếu dữ liệu luật hay bắt buộc LLM phải đoán. Câu hỏi hỏi trần hình phạt của tội danh, khác với xác định mức án thực tế của riêng người bị bắt.
- **Đề xuất sửa:** Trong `Neo4jGraph.context()`, nếu câu hỏi yêu cầu “tối đa/khung cao nhất”, lấy mọi khoản của Điều xác định qua Crime, hoặc riêng khoản có trần cao nhất bằng schema penalty có cấu trúc. Phương án lấy toàn bộ tăng token; phương án schema cần parse mức phạt chính xác. Đây là đề xuất sau phân tích, chưa thêm vào code của bài để giữ lựa chọn retrieval gợi ý và số liệu benchmark nhất quán.

### Lỗi E4: keyword recall và judge không phản ánh đầy đủ độ đúng

- **Hiện tượng:** Q6 Flat có recall=0.00 nhưng judge=2. Recall bỏ qua các tham chiếu tên ngắn; judge lại cho điểm đầy đủ dù thiếu một vụ trong đáp án chuẩn.
- **Bằng chứng:** Q6 Flat, nguyên văn các mục từ file kết quả:

> 1. **Vụ việc [1]:** Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh là MDMA (khối lượng gần 4,3kg) liên quan đến Đạt và Huy.
> 2. **Vụ việc [2]:** Thành bị bắt quả tang khi mang 5 viên ma túy đến điểm hẹn để bán, kết luận giám định xác định đây là ma túy MDMA.
> 3. **Vụ việc [3]:** Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng gửi bởi Huy là MDMA (khối lượng hơn 5,3kg) liên quan đến Đức.

`must_include` của Q6 là `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`. Không chuỗi nào xuất hiện nguyên văn nên hàm keyword_recall tính 0/3. Nhưng “Huy” và “Thành” vẫn có thông tin liên quan; điểm 0 không có nghĩa câu trả lời hoàn toàn không liên quan. Ngược lại, đáp án chuẩn còn có vụ Viện Pháp y tâm thần; Flat không nhắc vụ này, và hai kiện hàng của Huy được liệt kê thành hai vụ. Vì vậy judge=2 “đúng và đủ” không phù hợp khi đối chiếu thủ công. Graph Q6 có cả tên đầy đủ và Viện Pháp y, nhưng liệt kê các tên vụ gần nhau; không nên dùng judge=2 để kết luận entity resolution hoàn hảo.

- **Nguyên nhân:** Phép đo keyword là kiểm tra chuỗi con, không chuẩn hóa alias hoặc đối sánh thực thể. Prompt judge chỉ yêu cầu mức 0/1/2 tổng quát, không buộc kiểm tra từng vụ trong gold; LLM judge bỏ qua thiếu sót và trùng vụ.
- **Đề xuất sửa:** Ở vòng đánh giá sau, chuẩn hóa entity/alias và đo recall theo vụ thay vì chuỗi tên; yêu cầu judge trả bảng “có/thiếu/sai” cho từng ý gold trước điểm tổng. Có thêm chi phí và phải xử lý đồng danh. Không sửa test hay `bench_kg.py` để nâng điểm của lần nộp này; giữ nguyên hai tín hiệu và ghi rõ kiểm tra thủ công.

## 4. Kết luận

Knowledge Graph đáng cân nhắc khi câu hỏi phải nối mức án trong tin với Điều/khoản trong luật: Q3 và Q5 tăng judge từ 1 lên 2; recall Q3–Q5 trung bình tăng khoảng 0.36 lên 0.89. Trong lần đo này, đổi lại chi phí indexing khoảng 5.72×, chi phí mỗi câu khoảng 5.07×, input token mỗi câu khoảng 7.73×; thời gian hỏi tăng khoảng 1.12× dưới quota đang dùng. Graph không luôn tốt hơn: Q4 vẫn thiếu khoản tối đa.

Flat RAG đủ cho Q1/Q2 khi đáp án nằm trọn trong một chunk: cùng recall=1, judge=2 và rẻ hơn. Cần xét loại câu hỏi và chất lượng ontology/Cypher thay vì dùng graph cho mọi câu. Không có điểm hòa vốn chỉ theo chi phí ở cấu hình này, vì Graph vừa tốn thêm chi phí dựng vừa tốn thêm mỗi lần hỏi; muốn biện minh phải tính cả giá trị của câu trả lời đầy đủ và chi phí xử lý lỗi, chưa đo trong lab.

## 5. Tự kiểm

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Check hợp đồng chạy trước benchmark; trường USD=0 của check được ghi nguyên từ terminal lúc bảng giá chưa có model 3.5. Sau đó đã bổ sung giá và token counter, chạy smoke test bộ đo thành công và sinh lại benchmark đầy đủ bằng code cuối. Không dùng trường USD cũ của check để so chi phí. Không chạy lại `--check` sau benchmark vì lệnh này xóa graph đầy đủ để dựng graph nhỏ.

Ảnh chụp từ graph **202 node / 379 cạnh** sau bước dựng của lần benchmark thành công; chạy `:clear` trước mỗi truy vấn, không cắt ảnh:

- [Q-A — đếm node](img/kg_count.png): bảng đủ bảy label; Query/Browser bản mới trình bày count ở Table.
- [Q-B — cầu nối hai KB](img/kg_cross_kb.png): Graph và Results overview có Person, Case, Crime, Article và ba loại cạnh trên đường nối.
- [Q-D — vụ tự chọn](img/kg_my_case.png): chọn **Cái Quang Huy**, không dùng Lê Minh Thành; có Person, Case, Crime, Article, Substance, Location và Results overview.

`.env` và `.venv` được Git bỏ qua. Không thay đổi test, benchmark, chunking, store hay agent Flat. Chỉ hoàn thành `src/graph.py`, sửa đo chi phí/giãn request Gemini cần cho lần chạy thật trong `src/llm.py`, và tạo các đầu ra được yêu cầu.

## Vấn đề gặp phải

Lần chạy đầu dừng ở request trả lời Q1 sau khi dựng graph, với HTTP 429: `GenerateRequestsPerMinutePerProjectPerModel-FreeTier`, limit=15, model=`gemini-3.5-flash-lite`, retryDelay=36s. Thêm giãn request 4.2 giây trong client rồi chạy lại nguyên `python bench_kg.py --judge` thành công. Endpoint embedding trả usage=None nên bổ sung countTokens và giá Standard thay vì báo sai chi phí bằng 0. Các lần thử/setup này không nằm trong chi phí của lần benchmark thành công ở mục 1.
