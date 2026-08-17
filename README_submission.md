# Báo cáo Nộp bài Lab 17 - Multi-Memory Agent với Zep

## 1. Ba câu hỏi thực hành & thiết kế hệ thống
* **Layer quan trọng nhất trong bộ test:** **Long-term Memory** là tầng cốt lõi nhất vì bao phủ nhiều test case nhất (E02, E03, E08, E09, E07). Tầng này chịu trách nhiệm lưu giữ hồ sơ, sở thích (*Python*), việc còn dở (*open loops/16:00*), cập nhật xung đột theo thời gian (*Recency* ở E08) và bảo đảm cô lập người dùng (*User Isolation* ở E09).
* **Trade-off giữa Zep Context Block vs Redis + Qdrant:**
  * *Zep (Managed Graph):* Tự động trích xuất thực thể, quan hệ, quản lý tính mới (*Temporal validity*) và cô lập namespace theo user; tuy nhiên phụ thuộc vào mạng, chi phí API và độ trễ cloud (~700ms).
  * *Redis + Qdrant (Self-hosted):* Tốc độ cực nhanh (sub-millisecond), toàn quyền kiểm soát dữ liệu tại chỗ; nhưng phải tự xây dựng toàn bộ pipeline NLP trích xuất facts, xử lý xung đột và đồng bộ đồ thị.
* **Guardrail chống Memory Poisoning:** Áp dụng cổng kiểm duyệt quyền riêng tư (*Privacy Guard / Consent Gate*), phân tách nghiêm ngặt vùng nhớ theo `user_id`, lưu trữ nguồn gốc (*provenance*) cho từng episode, và giới hạn tiến trình nền (*Heartbeat*) chỉ làm nhiệm vụ dọn dẹp/đánh dấu stale task chứ không được tự ý ghi đè quyền hoặc chỉ thị hệ thống.

---

## 2. Phân tích kết quả Benchmark
1. **Layer có hit rate thấp nhất ở baseline:** Ở bản *No-memory*, các tầng **Long-term, Episodic, Semantic** đều đạt **0% hit rate** vì hoàn toàn không có khả năng truy xuất qua nhiều phiên (chỉ có Short-term đạt 100% do dữ liệu còn trong thread).
2. **Query retrieve nhiều token nhất:** Case **E02** và **E08** (~1,330 tokens) do phải trích xuất toàn bộ Context Block kết hợp đồ thị quan hệ edges để phân giải thực thể.
3. **Case Mixed (E07):** Cần kết hợp **Long-term** (sở thích `Python` của Minh) và **Semantic** (quy tắc `Idempotency-Key` từ tài liệu thanh toán chung).
4. **Token reduction vs Hit rate:** Bản *No-memory* đạt mức giảm token cao (81.8%) đơn giản vì không lấy thêm bất kỳ ngữ cảnh nào. Việc tiết kiệm token chỉ có ý nghĩa khi đi kèm với **Hit rate cao** (bản Memory đạt 100% hit rate với mức tinh gọn token hợp lý).

---

## 3. Phân tích Recency (E08) & Compaction (E10)
* **E08 (Recency wins):** Đồ thị Zep sử dụng nhãn thời gian (*valid_at*) để ưu tiên stack `TypeScript/NestJS` cho `BLUEBIRD-42` mà không làm mất tính toàn vẹn của sở thích `Python` trong `ORCHID-27`.
* **E10 (Compaction):** Cơ chế sliding window trích xuất các mẫu bền vững (*Durable Notes*) giúp bảo toàn deadline `REVIEW-DEADLINE-1600 (Friday at 16:00)` dù các tin nhắn thô ban đầu đã bị evict.
