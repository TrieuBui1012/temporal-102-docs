# Điều gì xảy ra với Timer nếu Worker crash?

## Timer là durable

* Timers trong Temporal là durable.

* Timer được duy trì bởi Temporal Cluster, không phải bởi Worker.

* Khi Worker gặp một trong các lời gọi sau trong Workflow code:

  * `workflow.Sleep`
  * `workflow.NewTimer`

* Worker sẽ gửi request đến Temporal Cluster để start Timer với duration đã chỉ định.

## Khi Timer fire

* Khi Timer fire, Temporal Cluster sẽ thêm một Workflow Task mới vào Task Queue.

* Sau đó Worker sẽ resume execution của Workflow.

* Cluster sẽ fire Timer sau đúng duration đã chỉ định, bất kể lúc đó có Worker nào đang chạy hay không.

## Ví dụ khi Worker crash

* Giả sử Workflow dùng `workflow.Sleep` để set Timer 10 giây.

* Hệ thống chỉ có một Worker process.

* Worker crash sau 3 giây.

### Trường hợp restart Worker sau 2 giây

* Worker crash ở giây thứ 3.

* Restart sau 2 giây, tức là Timer mới trôi qua tổng cộng 5 giây.

* Workflow Execution sẽ tiếp tục pause thêm 5 giây còn lại.

* Hành vi giống như Worker chưa từng crash.

### Trường hợp restart Worker sau 20 phút

* Timer 10 giây đã fire từ lâu trong lúc Worker không chạy.

* Khi Worker được start lại, nó sẽ nhận Workflow Task từ Task Queue.

* Workflow sẽ resume phần code còn lại ngay, không cần delay thêm.

## Ý nghĩa quan trọng

* Timer được Temporal Cluster quản lý nên vẫn fire dù không có Worker nào đang chạy.

* Tuy nhiên, Worker có thể bị downtime lâu vì:

  * Crash.
  * System maintenance.

* Vì vậy, Workflow code nên đủ robust để xử lý trường hợp Timer thực tế mất lâu hơn duration đã chỉ định.

* Không nên giả định rằng Workflow sẽ luôn resume ngay tại đúng thời điểm Timer fire.
