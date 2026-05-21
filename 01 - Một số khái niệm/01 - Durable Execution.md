# Hệ thống Durable Execution

---

## Durable Execution System là gì?

* Một **durable execution system** đảm bảo code trong application chạy:

  * Đáng tin cậy.
  * Chính xác.
  * Ngay cả khi có sự cố xảy ra.

* Hệ thống này duy trì state của application.

* Nhờ đó, code có thể tự động recover sau failure.

* Failure có thể là lỗi nhỏ, ví dụ:

  * Network timeout.

* Hoặc lỗi nghiêm trọng hơn, ví dụ:

  * Kernel panic trên production application server.

---

## Nâng cao năng suất cho developer

* Durable execution system cũng giúp cải thiện productivity cho developer.

* Khi xây dựng application, developer thường phải xử lý nhiều vấn đề ảnh hưởng đến reliability, ví dụ:

  * Failures.
  * Timeouts.

* Việc viết code để xử lý các vấn đề này thường tốn nhiều thời gian.

* Temporal cung cấp các abstraction cấp cao hơn cho application development.

* Temporal cũng có built-in scalability.

* Nhờ đó, bạn có thể tập trung nhiều hơn vào:

  * Business logic của application.

* Temporal còn cung cấp các tools giúp tăng productivity.

* Ví dụ:

  * Web UI để xem execution history của application.

* Web UI cho phép xem execution history của cả:

  * Các execution trong quá khứ.
  * Các execution hiện tại.

* Bạn có thể xem thêm:

  * Input parameters.
  * Return values.

* Trong course này, bạn sẽ dùng Web UI để:

  * Debug Workflow Execution.
  * Kiểm tra rằng fix của bạn đã giải quyết được vấn đề.
