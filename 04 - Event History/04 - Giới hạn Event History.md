# Giới hạn của Event History

## Vì sao Event History có giới hạn?

* Để tiết kiệm dung lượng lưu trữ và giữ hiệu năng tốt, Temporal giới hạn:

  * Số lượng `Events` trong `Event History`
  * Tổng dung lượng của `Event History`
  * Kích thước của từng payload được lưu trong history

# Giới hạn số lượng Events

## Warning khi Event History lớn dần

* `Temporal Cluster` bắt đầu log warning khi `Event History` đạt hơn `10K` events.

* Cụ thể, ngưỡng warning là:

```text
10,240 events
```

* Sau mốc này, cluster sẽ tiếp tục log thêm warning khi các `Events` mới được append vào history.

## Ngưỡng có thể bị terminate

* Nếu `Event History` vượt quá:

```text
51,200 events
```

* `Workflow Execution` có thể bị terminate.

## Best practice

* Không nên để một `Workflow Execution` có quá nhiều `Events`.

* Temporal khuyến nghị nên giữ history ở mức chỉ vài nghìn `Events`.

* Mục tiêu là có đủ thời gian phát hiện và xử lý trước khi Workflow chạm giới hạn và bị terminate.

## Cách xử lý khi history sắp quá lớn

* Một kỹ thuật phổ biến là dùng `Continue-As-New`.

* `Continue-As-New` cho phép Workflow tiếp tục logic xử lý nhưng dưới một `Workflow Execution` mới.

* `Workflow Execution` mới sẽ có `Event History` mới.

* Có thể lặp lại cơ chế này mỗi khi Workflow lại tiến gần đến giới hạn.

# Giới hạn kích thước Event History

## Vì sao kích thước history có thể tăng lớn?

* Input parameters và output values của cả `Workflows` và `Activities` đều được lưu trong `Event History`.

* Nếu truyền hoặc trả về dữ liệu lớn, history sẽ phình to và gây vấn đề hiệu năng.

## Các giới hạn kích thước quan trọng

* `Workflow Execution` có thể bị terminate nếu:

  * Bất kỳ payload nào vượt quá:

```text
2 MB
```

* Toàn bộ `Event History` vượt quá:

```text
50 MB
```

## Payload là gì trong ngữ cảnh này?

* Payload có thể là:

  * Input truyền vào `Workflow`
  * Result trả về từ `Workflow`
  * Input truyền vào `Activity`
  * Result trả về từ `Activity`

# Best practice khi truyền dữ liệu

## Tránh truyền dữ liệu lớn qua Workflow và Activity

* Không nên truyền lượng dữ liệu lớn vào hoặc ra khỏi `Workflows` và `Activities`.

* Việc này giúp tránh:

  * Chạm giới hạn payload
  * Làm `Event History` quá lớn
  * Gây giảm hiệu năng

## Dùng Claim Check pattern

* Một cách phổ biến để xử lý dữ liệu lớn là dùng `Claim Check pattern`.

* Pattern này thường được dùng trong messaging systems như `Apache Kafka`.

## Ý tưởng của Claim Check pattern

* Thay vì truyền trực tiếp dữ liệu lớn vào function:

  * Lưu dữ liệu ở bên ngoài Temporal.
  * Ví dụ: database hoặc file system.

* Sau đó chỉ truyền identifier của dữ liệu vào function.

* Identifier có thể là:

  * Primary key
  * File path
  * Một ID dùng để truy xuất dữ liệu

## Áp dụng với Activity

* Workflow có thể truyền identifier vào `Activity`.

* `Activity` sẽ dùng identifier đó để retrieve dữ liệu khi cần.

* Nếu `Activity` tạo ra lượng dữ liệu lớn:

  * Ghi dữ liệu đó ra external system.
  * Return về identifier thay vì return toàn bộ dữ liệu.