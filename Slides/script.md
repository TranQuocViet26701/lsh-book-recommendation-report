# Kịch bản thuyết trình — LSH Book Recommendation

**Thời lượng tổng:** ~15 phút (3 thành viên × ~5 phút) + ~3-5 phút Q&A
**Slide deck:** `Slides/main.pdf` (23 trang)

Mỗi block dưới đây tương ứng với một slide. Định dạng:

> **(Slide N — Thành viên X — ~T giây)** *Tên slide*
>
> - Nội dung cần nói
> - Transition cue → slide kế tiếp

---

## Mở đầu — Thành viên 1

### (Slide 1 — Thành viên 1 — 30s) Title

- *Chào quý thầy cô và các bạn. Nhóm chúng em xin trình bày Mini Project Dữ liệu lớn học kỳ 2 năm 2025-2026, đề tài: **Hệ thống gợi ý sách tương tự dựa trên Locality-Sensitive Hashing.***
- Giới thiệu nhanh 3 thành viên: ai phụ trách phần nào.
- → "Phần đầu sẽ do em [TV1] trình bày. Trước hết là mục lục."

### (Slide 2 — Thành viên 1 — 30s) Mục lục

- Báo cáo gồm 3 phần chính: **bài toán & tech stack**, **thuật toán & metric framework**, và **kết quả thí nghiệm + bài học**.
- → "Bắt đầu với bài toán."

---

## Phần 1 — Thành viên 1 (~4 phút)

### (Slide 3 — Thành viên 1 — 15s) Phần 1 divider

- *Phần 1 — em [TV1] trình bày.*
- → bài toán

### (Slide 4 — Thành viên 1 — 60s) Bài toán Similarity Search

- Trong kỷ nguyên số, lượng tài liệu văn bản tăng nhanh; bài toán **tìm cặp tài liệu giống nhau** xuất hiện ở nhiều ngữ cảnh: thư viện số, anti-plagiarism, recommendation.
- Brute-force pairwise có độ phức tạp $O(N^2)$, không khả thi với hàng triệu tài liệu.
- LSH là phương pháp **xấp xỉ**: hash sao cho tài liệu giống nhau có xác suất cao trùng bucket. Giảm chi phí xuống gần tuyến tính.
- → "Vậy đề tài này nhắm vào những mục tiêu cụ thể nào?"

### (Slide 5 — Thành viên 1 — 60s) Mục tiêu đề tài

- 4 mục tiêu: pipeline end-to-end, triển khai phân tán trên Spark, đánh giá toàn diện, phân tích thực tế.
- Phạm vi: Project Gutenberg (English), Jaccard similarity, content-based — *không* xử lý collaborative filtering.
- → "Bây giờ kiến trúc tổng thể."

### (Slide 6 — Thành viên 1 — 60s) Pipeline 5 giai đoạn

- 5 giai đoạn: Raw text → Preprocessing → Shingling+MinHash → LSH Banding → Query.
- 3 quyết định công nghệ then chốt: storage (HDFS thiết kế / Databricks Volumes thực thi), format trung gian Parquet, engine Apache Spark.
- → "Phần thú vị nhất ở đây là câu chuyện pivot công nghệ."

### (Slide 7 — Thành viên 1 — 60s) Tech Stack & câu chuyện pivot

- **Kế hoạch ban đầu:** cluster Hadoop 3 nút trên VPS, dataset 3K-10K sách. Cản trở: cloud credits không được duyệt đúng tiến độ.
- **Pivot:** chuyển sang **Databricks Free Edition Serverless**. Lợi: không cần dựng cluster, auto-mount Git repo. Đánh đổi: mất `spark.sparkContext`, driver memory hẹp khiến TN3 phải scale-down (sẽ thấy ở phần 3).
- *Đây là bài học hạ tầng quan trọng nhất của đề tài.*
- → "Phần 1 hết. Mời em [TV2] tiếp tục với thuật toán và metric framework."

---

## Phần 2 — Thành viên 2 (~5 phút)

### (Slide 8 — Thành viên 2 — 15s) Phần 2 divider

- *Cảm ơn anh/em. Em là [TV2], em sẽ trình bày thuật toán LSH chi tiết và bộ metric đánh giá.*

### (Slide 9 — Thành viên 2 — 45s) Bước 1 — Shingling

- Shingling là bước tách tài liệu thành các **k-gram**, ở đây là trigram cấp từ ($k=3$).
- Đưa ví dụ "the quick brown fox jumps" → 3 trigram tương ứng.
- Output: Parquet với schema `[book_id, shingles: array<string>]`.
- → "Sau khi có tập shingle, làm sao so sánh hiệu quả? MinHash."

### (Slide 10 — Thành viên 2 — 60s) Bước 2 — MinHash

- MinHash dùng $n$ hàm hash, lấy **giá trị nhỏ nhất** sau khi hash từng phần tử trong tập shingle. Kết quả là vector $n$ số nguyên — gọi là *signature*.
- **Tính chất quan trọng:** xác suất hai signature trùng tại một vị trí *bằng đúng* Jaccard similarity của hai tập gốc. Đây là cơ sở lý thuyết của toàn bộ pipeline.
- Project dùng $n = 100$ (xem TN2 vì sao chọn 100).
- → "Có signature rồi, làm sao tìm cặp giống nhau hiệu quả? LSH banding."

### (Slide 11 — Thành viên 2 — 75s) Bước 3 — LSH Banding + S-curve

- Chia signature $n$ giá trị thành $b$ band $\times$ $r$ row, $n = b \cdot r$.
- Quy tắc candidate: trùng band ở **ít nhất 1 vị trí**.
- Threshold S-curve: $t \approx (1/b)^{1/r}$ — chọn $(b, r)$ là chọn vị trí "lốc" của đường cong.
- Ví dụ trong project: $n=100$, $b=20$, $r=5$ → $t \approx 0.55$ — gần với GT threshold $0.5$.
- *(Chỉ vào đường cong S đang vẽ trên slide.)*
- → "Cuối cùng là bước truy vấn."

### (Slide 12 — Thành viên 2 — 45s) Bước 4 — Query Top-K

- Pipeline truy vấn: book_id → lookup signature → tìm bucket candidates → tính exact Jaccard → rank → Top-K.
- Bucket lookup chỉ tốn $O(b)$ thay vì $O(N)$ của brute-force.
- LSH là xấp xỉ → có thể có False Negative (bỏ sót) và False Positive (nhiễu) — chính là động cơ cho metric framework ở slide tiếp theo.
- → "Đo cái gì để đánh giá hệ thống?"

### (Slide 13 — Thành viên 2 — 45s) 4 nhóm metric

- **Chất lượng:** Recall, Precision, F1 — so với ground truth brute-force ở threshold 0.5.
- **Hiệu năng:** Execution Time, Speedup, Scalability.
- **LSH-specific:** **Selectivity** (lượng candidate / tổng), False Positive Rate, Bucket Imbalance.
- **Dataset rep.:** GT density — để chẩn đoán dataset có đủ phân biệt hay không.
- → "Setup cụ thể cho 4 thí nghiệm."

### (Slide 14 — Thành viên 2 — 45s) Setup TN1--TN4

- 90 sách Gutenberg, 4005 cặp khả dĩ, **4 cặp ground-truth** ở $J \geq 0.5$, brute-force GT compute mất 15.73s.
- Runtime: Databricks Free Edition Serverless. Code reuse từ `src/shingling.py`, `src/minhash.py`, `src/lsh.py`.
- 4 thí nghiệm: TN1 sweep $(b,r)$, TN2 sweep $n$, TN3 scalability, TN4 LSH vs Brute-force.
- → "Tổng quan kết quả nhanh."

### (Slide 15 — Thành viên 2 — 60s) Bảng tổng hợp

- *(Lướt nhanh qua bảng — chi tiết từng thí nghiệm em [TV3] sẽ trình bày.)*
- Headline: TN1 và TN2 đều cho F1 = 1.0 (metric bão hòa); TN3 speedup 0.006 → 0.018 (đang dốc lên); TN4 speedup 0.011×.
- *Hai quan sát:* (1) chất lượng bão hòa do GT chỉ có 4 cặp; (2) speedup < 1 vì đang ở vùng overhead-dominated của LSH ở scale này.
- → "Em chuyển sang em [TV3] để trình bày chi tiết kết quả thí nghiệm và phần kết luận."

---

## Phần 3 — Thành viên 3 (~5 phút)

### (Slide 16 — Thành viên 3 — 15s) Phần 3 divider

- *Cảm ơn anh/em. Em là [TV3], em sẽ trình bày chi tiết kết quả 4 thí nghiệm và rút ra bài học.*

### (Slide 17 — Thành viên 3 — 60s) TN1 — Sweep $(b, r)$

- 4 cấu hình $(b,r)$ với threshold lý thuyết trải từ 0.14 tới 0.79.
- *(Chỉ vào biểu đồ S-curve bên trái — 4 đường cong khác nhau theo $(b, r)$.)*
- Tất cả đều F1 = 1.0, thời gian xấp xỉ $1260$s — chênh nhau dưới 0.5% giữa các cấu hình. Điều này xác nhận **overhead Spark/JVM chi phối**, chứ không phải bản thân thuật toán LSH.
- → "Tiếp theo là TN2 — số hàm hash."

### (Slide 18 — Thành viên 3 — 60s) TN2 — Sweep $n$

- Sweep $n \in \{50, 100, 150, 200\}$, giữ $r = 5$.
- F1 vẫn = 1.0 cho mọi $n$. Thời gian tăng tuyến tính — fit ra hệ số khoảng **12.3 giây / hash unit**.
- *Bài học:* không có lý do thực nghiệm để chọn $n > 100$ trên dataset này.
- → "Bài toán scalability — TN3."

### (Slide 19 — Thành viên 3 — 75s) TN3 — Scalability

- Sinh dataset synthetic bằng nhân bản DataFrame, kích thước 200, 500, 1000 sách. (Kế hoạch ban đầu là 1K-10K, nhưng driver memory không cho phép.)
- Speedup tăng từ 0.006 → 0.006 → 0.018 — *gấp 3 lần khi N tăng 5 lần.*
- *Đây là điểm quan trọng:* speedup *đang dốc lên*, gợi ý điểm hòa vốn LSH-vs-BF nằm ở $N \gg 1000$. Không có dấu hiệu LSH sẽ tiếp tục thua brute-force ở scale $\geq 5{,}000$ sách.
- → "TN4 — so sánh end-to-end."

### (Slide 20 — Thành viên 3 — 75s) TN4 — LSH vs Brute-force

- 90 sách thật, $n=100$, $b=20$, $r=5$. Hai metric chính của LSH cho hai câu trả lời khác nhau:
  - **Selectivity:** LSH chỉ trả 4 candidate trên 4005 cặp — giảm $1000\times$ — *thắng tuyệt đối.*
  - **Wall time:** LSH 1257s vs brute-force 13.3s — *thua $\sim$95×*. Speedup chỉ 0.011×.
- Mâu thuẫn này không phải lỗi: ở $N = 90$, overhead Spark/JVM chi phối; theo TN3, ngưỡng wall-time hòa vốn nằm ở $N \gg 1000$.
- → "Vậy hạn chế và hướng phát triển là gì?"

### (Slide 21 — Thành viên 3 — 75s) Hạn chế

- 3 hạn chế dataset: (H1) chỉ 4 cặp GT trong 4005 → metric bão hòa; (H2) overhead Spark chi phối ở scale nhỏ; (H3) threshold $J \geq 0.5$ quá khắc nghiệt cho corpus đa thể loại.
- 3 ràng buộc hạ tầng từ pivot Databricks Free Edition: driver memory hẹp, mất `spark.sparkContext`, compute dùng chung.
- *Đây là context cần thiết để diễn giải kết quả.*
- → "Hướng phát triển."

### (Slide 22 — Thành viên 3 — 60s) Hướng phát triển

- **Hạ tầng:** dựng cluster ≥ 3 worker khi xin được credits → chạy lại TN3 ở 5K-10K sách để cắt qua điểm hòa vốn.
- **Phương pháp:** mở rộng dataset thật, sinh positives với gradient $J$ thay vì duplicate, tách $T_{\text{setup}}$ vs $T_{\text{algo}}$ để báo speedup công bằng.
- **Sản phẩm:** triển khai tầng query real-time tách khỏi pipeline batch (đây là use-case mà overhead Spark không còn chi phối, sẽ thể hiện đúng giá trị của LSH).
- → kết luận

### (Slide 23 — Thành viên 3 — 75s) Kết luận + Q&A

- 4 takeaways:
  1. Pipeline LSH end-to-end đã được triển khai và verify đúng chức năng.
  2. Pivot HDFS+Spark → Databricks Free Edition là một quyết định pragmatic; có cost trade-off rõ ràng.
  3. TN1, TN2 chứng minh hệ thống đúng; TN3, TN4 cho thấy đang ở vùng overhead-dominated.
  4. Đường cong scalability dốc lên → ở $N \gg 1000$, LSH sẽ thắng brute-force về wall time.
- *Cảm ơn quý thầy cô. Nhóm em sẵn sàng nhận câu hỏi.*

---

## Bảng thời gian rehearsal

| Phần         | Slide  | Thành viên | Thời gian |
|--------------|--------|------------|-----------|
| Mở đầu       | 1--2   | TV1        | ~1 phút   |
| Phần 1       | 3--7   | TV1        | ~4 phút   |
| Phần 2       | 8--15  | TV2        | ~5 phút   |
| Phần 3       | 16--23 | TV3        | ~5 phút   |
| **Tổng**     |        |            | **~15 phút** |

## Lưu ý khi thuyết trình

- **Không đọc slide.** Mỗi gạch đầu dòng trên slide chỉ là cue cho người nói; nội dung đầy đủ ở script này.
- **Số liệu cần thuộc:** 90 sách / 4005 cặp / 4 cặp GT; speedup TN3 0.006→0.018; speedup TN4 0.011×; selectivity 0.001 vs 1.000.
- **Câu hỏi có thể bị hỏi:**
  - *"Tại sao không dùng cluster Spark thật?"* → Cloud credits không kịp + chi phí thiết lập VPS đơn lẻ vượt ngân sách. Pivot Databricks là quyết định ưu tiên giá trị học thuật của thí nghiệm hơn là cấu hình hạ tầng.
  - *"Tại sao F1 = 1.0 vẫn là kết quả tốt?"* → Không phải tốt — là *bão hòa*. Chính là lý do nhóm đề xuất Hướng cải thiện 1: mở rộng dataset để metric có sức phân biệt.
  - *"Tại sao LSH chậm hơn brute-force?"* → Vì $N = 90$ quá nhỏ để khấu hao overhead Spark/JVM (~1240s init/shuffle so với 4ms thuật toán). TN3 cho thấy speedup tăng tuyến tính với $N$.
- **Có demo sống không?** → Không; phạm vi đã thu hẹp (xem slide 21 + báo cáo § 6.2.2). Người đọc xem kết quả qua bảng và biểu đồ.
