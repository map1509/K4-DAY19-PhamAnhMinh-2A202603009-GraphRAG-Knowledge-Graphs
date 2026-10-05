# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phạm Anh Minh  **MSSV:** 2A202603009  **Ngày:** 05/10/2026

## 1. Chi phí (10 điểm)

Kết quả cuối được sinh từ code sau sửa retrieval Q6; USD ước tính theo src/llm.py, không phải hóa đơn provider. File trước sửa lưu riêng tại report/benchmark_before_q6.txt.

```text
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small | top_k=3 | chunk_size=800 | chunks=176 | KG: 205 nodes / 384 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     41.5
graph       196     91958     4882   0.00943    120.6

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.41
graph       1.00   2.00     5126      136   0.00084     2.85

```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00943 | 8.42× |
| Indexing giây | 41.5 | 120.6 | 2.91× |
| Mỗi câu: USD | 0.00013 | 0.00084 | 6.46× |
| Mỗi câu: giây | 1.41 | 2.85 | 2.02× |
| Mỗi câu: in_tok | 694 | 5126 | 7.39× |

Tỉ lệ dựa trên số đã làm tròn. Graph indexing gồm vector index giống Flat cộng trích tin bằng LLM; luật dùng regex. Querying tăng do thêm dữ kiện graph và toàn văn khoản. Riêng câu liệt kê vụ, bản sửa xuất danh sách khớp trực tiếp thay vì thêm khoản luật không phục vụ câu hỏi.

Judge trong bench_kg.py chạy ngoài vùng metered mỗi câu, nên bảng Querying không tính token/tiền/thời gian judge. Không dùng tổng các bảng làm hóa đơn toàn phiên. Graph không có điểm hòa vốn về tiền API thuần nếu cả phí indexing và phí mỗi câu đều cao hơn; giá trị câu trả lời tốt hơn phải bù chênh lệch. Chỉ đo 6 câu, chưa đủ ngoại suy.

## 2. Từng câu hỏi (10 điểm)

Judge dùng thang 0–2. Bảng phản ánh lần benchmark cuối.

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Đối chiếu định nghĩa tiền chất trong một nguồn luật. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Đối chiếu tên người bị tử hình trong cùng bài. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Cần nối mức án Thành với Điều 251 và khoản cơ bản. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Cần alias Hoàng Nato và toàn Điều để lấy khung tối đa. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Cần nối vận chuyển, lượng MDMA với Điều 250 khoản 4. |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Danh sách toàn graph ghi rõ chất và nguồn cho mỗi vụ; còn nguy cơ node vụ trùng. |

## 3. Phân tích lỗi (20 điểm)

Bằng chứng của graph cuối lưu ở GRAPH_EVIDENCE.json; graph trước sửa ở GRAPH_EVIDENCE_before_q6.json. Hai lần dựng dùng LLM nên số node và lỗi trích có thể khác. Không quy toàn bộ khác biệt giữa lần chạy cho sửa retrieval.

### Lỗi E3: tên thực thể không ổn định

- **Hiện tượng:** ontology khóa theo tên; lần trước có Ketamine/ketamine và Methamphetamine/methamphetamine là node khác nhau. Kiểm tra graph cuối và cả tên vụ để đánh giá lỗi còn lại.
- **Bằng chứng:**

```cypher
MATCH (s:Substance) WHERE toLower(s.name) IN ['ketamine', 'methamphetamine'] RETURN s.name AS name ORDER BY toLower(s.name), s.name;
```

```json
[
  {
    "name": "Ketamine"
  },
  {
    "name": "ketamine"
  },
  {
    "name": "Methamphetamine"
  },
  {
    "name": "methamphetamine"
  }
]
```

Danh sách Case cuối:

```cypher
MATCH (k:Case) RETURN k.name AS name,k.doc_id AS doc_id ORDER BY k.name;
```

```json
[
  {
    "name": "Chuyên án A3-626P",
    "doc_id": "news-100261002184934505"
  },
  {
    "name": "Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy",
    "doc_id": "news-100260920221957595"
  },
  {
    "name": "Vụ bắt giữ giang hồ mạng 'Đức Cộng'",
    "doc_id": "news-100260924101703641"
  },
  {
    "name": "Vụ góp tiền mua ma túy tại Hà Nội",
    "doc_id": "news-100260918080821054"
  },
  {
    "name": "Vụ mua bán hơn 36kg ma túy tại TP.HCM",
    "doc_id": "news-100260928173914514"
  },
  {
    "name": "Vụ phát hiện 20kg ma túy tại Phú Quốc",
    "doc_id": "news-100260927182621527"
  },
  {
    "name": "Vụ sử dụng ma túy etomidate của Hoàng Nato và Phan Kim Nhi",
    "doc_id": "news-100260924095400982"
  },
  {
    "name": "Vụ triệt phá 8 đường dây ma túy tại TP.HCM",
    "doc_id": "news-100260925144412498"
  },
  {
    "name": "Vụ tông cảnh sát giao thông ở An Giang",
    "doc_id": "news-100260926112415229"
  },
  {
    "name": "Vụ tổ chức sử dụng ma túy tại Sầm Sơn",
    "doc_id": "news-100260930085028036"
  },
  {
    "name": "Vụ tổ chức sử dụng trái phép chất ma túy liên quan đến TikToker Phannhibeauty",
    "doc_id": "news-100260922111804786"
  },
  {
    "name": "Vụ vận chuyển 840kg ma túy đá tại Campuchia",
    "doc_id": "news-100260924145818945"
  },
  {
    "name": "Vụ vận chuyển ma túy của Cái Quang Huy",
    "doc_id": "news-100260918080821054"
  },
  {
    "name": "Vụ vận chuyển ma túy từ Đức về Việt Nam",
    "doc_id": "news-100260917203001265"
  },
  {
    "name": "Vụ án tại Viện Pháp y tâm thần Trung ương",
    "doc_id": "news-100260924105118645"
  }
]
```

- **Nguyên nhân:** add_news_case MERGE theo tên nguyên văn từ LLM; constraint chỉ chặn cùng khóa, không gộp khác cách viết hay cùng vụ có tên khác. Luật MENTIONS tên chuẩn, tin có thể INVOLVES tên khác. Sửa Q6 lọc không phân biệt chữ hoa/thường không sửa node đã trùng.
- **Đề xuất sửa:** chuẩn hóa Substance bằng code trước MERGE, giữ chất mới thay vì ép vào danh sách; với Case cần case_id theo sự kiện, lưu nhiều doc_ids. Không thêm token cho chuẩn hóa hoa/thường; gộp theo sự kiện cần bằng chứng và có nguy cơ gộp nhầm. Dựng lại graph sau sửa định danh.

### Lỗi E1: Case không nối tới tội chuẩn

- **Hiện tượng:** trước sửa có vụ Huy từ đoạn giới thiệu cuối bài Thành thiếu CHARGED_WITH; bài chính mô tả cùng vụ có cầu nối. Kết quả graph cuối dưới đây, không coi mọi Case thiếu cạnh là lỗi.
- **Bằng chứng — graph cuối:**

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name AS name, k.doc_id AS doc_id;
```

```json
[
  {
    "name": "Vụ vận chuyển ma túy của Cái Quang Huy",
    "doc_id": "news-100260918080821054"
  },
  {
    "name": "Vụ tông cảnh sát giao thông ở An Giang",
    "doc_id": "news-100260926112415229"
  }
]
```

Bằng chứng trước sửa:

```json
[
  {
    "name": "Vụ vận chuyển ma túy của Cái Quang Huy",
    "doc_id": "news-100260918080821054"
  },
  {
    "name": "Vụ tông cảnh sát giao thông ở An Giang",
    "doc_id": "news-100260926112415229"
  }
]
```

Bài news-100260918080821054 kết thúc bằng đoạn giới thiệu Huy “bị cáo buộc hai lần vận chuyển ma túy về Việt Nam qua sân bay Nội Bài”; bài chính news-100260917203001265 nêu tội vận chuyển. Một Case trích từ teaser bị thiếu cầu là lỗi trích/link; vụ tông CSGT có thể thuộc tội ngoài corpus nên không tự kết luận sai.

- **Nguyên nhân:** crawl giữ đoạn giới thiệu bài liên quan; prompt tạo thêm Case nhưng không luôn trích đủ tội; khóa tên không nhận biết hai bản ghi cùng sự kiện. Lỗi nằm ở trích xuất/định danh; sửa retrieval Q6 không sửa cầu nối này.
- **Đề xuất sửa:** phân biệt bài chính với teaser trong crawl/prompt, đối chiếu nguồn rồi trích lại; gộp sự kiện bằng khóa ổn định, giữ mọi nguồn. Tốn thêm lượt trích nếu sửa prompt; lọc nội dung quá mạnh có thể mất ngữ cảnh.

### Lỗi E5 trước sửa và kiểm chứng Q6 sau sửa

- **Hiện tượng trước sửa:** Q6 GraphRAG bỏ tên Viện Pháp y tâm thần dù graph có Case nối MDMA; recall 0,67/judge 1.
- **Bằng chứng nguyên văn trước sửa:**

```text
Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:

1. Vụ góp tiền mua ma túy tại Hà Nội: Trong vụ này, có liên quan đến 5 viên ma túy MDMA mà Lê Minh Thành đã bán.
2. Vụ vận chuyển ma túy của Cái Quang Huy: Cái Quang Huy bị cáo buộc vận chuyển hơn 9,6kg MDMA từ Đức về Việt Nam.
3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Tại đây, công an thu giữ 0,686g ma túy MDMA trong buồng chữa bệnh của Lê Văn Đông.

Tất cả các vụ việc này đều có liên quan đến ma túy MDMA.
```

- **Nguyên nhân:** context trộn summaries với seed facts/khoản luật; summary vụ Viện nhấn mạnh hối lộ, không nêu chất. LLM rút gọn và bỏ một mục. Đây là diễn giải có căn cứ từ prompt/câu trả lời, không chứng minh duy nhất nguyên nhân.
- **Sửa đã thực hiện:** nhánh tổng hợp trong src/graph.py truy vấn toàn graph theo chất, ghi rõ tên vụ, chất/amount, người, nguồn và summary cho từng bản ghi; không thêm khoản luật không cần. GRAPH_PROMPT yêu cầu bảo toàn các mục và nguồn, chỉ gộp sự kiện khi có căn cứ; nếu giới hạn facts thì báo chưa đầy đủ. Không hard-code tên vụ hoặc gold vào code.
- **Bằng chứng sau sửa:**

```text
Các vụ việc liên quan đến ma túy MDMA trong tin tức bao gồm:

1. **Vụ góp tiền mua ma túy tại Hà Nội**
   - Người: Trịnh Vũ Kiên, Kim Xuân Tuấn, Lê Minh Thành, Nguyễn Quang Hưng
   - Nguồn: [news-100260918080821054]
   - Chất liên quan: MDMA (5 viên)

2. **Vụ tổ chức sử dụng ma túy tại Sầm Sơn**
   - Người: Ngô Việt Dũng, Trần Quốc An, Cao Thị Bích Hằng, Nguyễn Thị Mai Anh, Lê Văn Đông
   - Nguồn: [news-100260930085028036]
   - Chất liên quan: MDMA (0,686g)

3. **Vụ vận chuyển ma túy của Cái Quang Huy**
   - Người: Cái Quang Huy
   - Nguồn: [news-100260918080821054]
   - Chất liên quan: MDMA (9,6kg)

4. **Vụ vận chuyển ma túy từ Đức về Việt Nam**
   - Người: Nguyễn Tiến Đạt, Cái Quang Huy
   - Nguồn: [news-100260917203001265]
   - Chất liên quan: MDMA (9,6kg)

5. **Vụ án tại Viện Pháp y tâm thần Trung ương**
   - Người: Bùi Thị Thanh Thủy, Dương Văn Biết, Dương Văn Lương, Lê Văn Đông, Trần Văn Trường, Trần Quốc An, Nguyễn Thị Mai Anh, Nguyễn Thị Thu Hoài, Nguyễn Văn Quang
   - Nguồn: [news-100260924105118645]
   - Chất liên quan: MDMA (không rõ lượng)

Tất cả các vụ việc trên đều có liên quan đến ma túy MDMA.
```

Truy vấn MDMA trên graph cuối:

```cypher
MATCH (k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'}) RETURN DISTINCT k.name AS name,k.summary AS summary,k.doc_id AS doc_id;
```

```json
[
  {
    "name": "Vụ tổ chức sử dụng ma túy tại Sầm Sơn",
    "summary": "Lê Văn Đông, trong thời gian chữa bệnh bắt buộc, đã trốn viện và tổ chức sử dụng ma túy trên bãi biển Sầm Sơn cùng với một số người khác.",
    "doc_id": "news-100260930085028036"
  },
  {
    "name": "Vụ án tại Viện Pháp y tâm thần Trung ương",
    "summary": "Vụ án liên quan đến việc 39 bác sĩ, điều dưỡng và nhiều bị cáo thao túng để 'chạy' giám định tâm thần cho những người phạm tội ma túy. Nhiều người đã chi hàng trăm triệu đến hàng tỉ đồng để được chữa bệnh bắt buộc.",
    "doc_id": "news-100260924105118645"
  },
  {
    "name": "Vụ vận chuyển ma túy của Cái Quang Huy",
    "summary": "Cái Quang Huy bị cáo buộc hai lần vận chuyển ma túy về Việt Nam qua sân bay Nội Bài, tổng số ma túy gồm hơn 9,6kg MDMA và gần 406g Ketamine.",
    "doc_id": "news-100260918080821054"
  },
  {
    "name": "Vụ góp tiền mua ma túy tại Hà Nội",
    "summary": "Ba thanh niên góp tiền để Lê Minh Thành mua ma túy cho tiệc sinh nhật nhưng sau đó không tham gia. Tòa sơ thẩm tuyên phạt mỗi người 24 tháng tù về tội mua bán trái phép chất ma túy.",
    "doc_id": "news-100260918080821054"
  },
  {
    "name": "Vụ vận chuyển ma túy từ Đức về Việt Nam",
    "summary": "Cái Quang Huy bị cáo buộc hai lần vận chuyển ma túy về Việt Nam qua sân bay Nội Bài, tổng khối lượng hơn 9,6kg MDMA và gần 406g Ketamine. Nguyễn Tiến Đạt cũng bị liên quan đến vụ việc này.",
    "doc_id": "news-100260917203001265"
  }
]
```

Q6 sau sửa: recall 1.00, judge 2. Hai lần benchmark cũng trích lại graph bằng LLM nên đây là bằng chứng đầu ra cải thiện, không phải thử nghiệm kiểm soát riêng prompt. Chưa khẳng định mọi lần chạy đều giống nhau.

## 4. Kết luận (5 điểm)

Lần cuối recall trung bình Flat 0.43, Graph 1.00; judge Flat 1.00, Graph 2.00. Chi phí indexing Graph/Flat 8.42 lần; phí mỗi câu 6.46 lần, thời gian mỗi câu 2.02 lần.

KG phù hợp khi phải nối tin với Điều/khoản luật. Với câu trong một nguồn, nếu chất lượng hòa thì Flat tiết kiệm hơn. Tổng hợp danh sách cần bảo toàn kết quả truy vấn thay vì chỉ đưa summary vào prompt. Q6 đã sửa retrieval, nhưng chất lượng vẫn phụ thuộc trích xuất và định danh; chưa bảo đảm đầy đủ ngoài tập dữ liệu lab.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
48 passed in 0.08s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[OK] KG-2 build_graph: 148 node / 292 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 17 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00076.
```

Check chạy trước benchmark cuối để graph đầy đủ được giữ lại. Kiểm tra cục bộ thêm: từng Case giữ chất và nguồn ngay cả summary không nhắc chất; giới hạn facts phải báo danh sách chưa đầy đủ.

Ảnh Neo4j của graph cuối: report/img/kg_count.png, kg_cross_kb.png, kg_my_case.png. Người chọn: **Cái Quang Huy**. Ảnh toàn cửa sổ, chạy :clear trước mỗi query, không chỉnh sửa ảnh. Truy vấn đếm trả bảng; phiên bản Browser này hiển thị Results overview ở tab Graph của hai ảnh đường đi.

## Vấn đề gặp phải (không tính điểm)

Lỗi jiter bị Windows Application Control chặn đã được xử lý trước benchmark. Kết nối API trong sandbox bị từ chối nên dùng chạy ngoài sandbox với quyền đã duyệt. Không sửa tests hoặc bench_kg.py; kết quả trước và sau đều sinh từ code, không chỉnh file benchmark.
