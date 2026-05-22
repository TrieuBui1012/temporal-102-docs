# Timer là gì?

## Durable Timer

* `Durable Timer` là một tính năng của Temporal.

* Timer được dùng để tạo độ trễ trong Workflow Execution.

* Khi một Timer được set:

  * Phần code đang chờ Timer sẽ bị block.
  * Code chỉ tiếp tục chạy khi Timer fire.

## Timer được quản lý bởi Temporal Cluster

* Timer được duy trì bởi Temporal Cluster.

* Worker không tiêu tốn tài nguyên trong lúc chờ Timer.

* Điều này giúp Workflow có thể chờ trong thời gian dài mà không cần giữ Worker bận.

## Thời lượng của Timer

* Duration của Timer là cố định.

* Timer có thể kéo dài từ:

  * 1 giây.
  * Đến vài năm.

## Tránh dùng Timer dưới 1 giây

* Có thể chỉ định duration nhỏ hơn 1 giây.

* Tuy nhiên, nên tránh làm vậy.

* Lý do:

  * Timer phụ thuộc vào roundtrip giữa Worker và Temporal Cluster.
  * Latency của roundtrip có thể ảnh hưởng đến độ chính xác.
  * Timer có duration dưới 1 giây sẽ không đảm bảo precision tốt.

# Use Cases của Timer

## Khi nào cần dùng Timer?

* Có nhiều lý do để thêm delay vào Workflow Execution.

* `Timer` giúp Workflow tạm dừng một khoảng thời gian trước khi tiếp tục bước tiếp theo.

## Chạy Activity theo fixed intervals

* Business logic có thể yêu cầu execute Activity theo các mốc thời gian cố định.

* Ví dụ: Workflow dùng cho customer onboarding có thể gửi email nhắc nhở sau khi user đăng ký:

  * Sau 1 ngày.
  * Sau 1 tuần.
  * Sau 1 tháng.

## Chạy Activity nhiều lần theo interval được tính động

* Có trường hợp cần execute một Activity nhiều lần, nhưng khoảng cách giữa các lần chạy không cố định.

* Interval có thể được tính toán động trong runtime.

* Ví dụ:

  * Một Activity trước đó trả về một giá trị.
  * Workflow dùng giá trị đó để quyết định delay bao lâu trước khi gọi Activity tiếp theo.

## Chờ các bước offline hoàn tất

* Timer cũng hữu ích khi Workflow cần chờ các bước ngoài hệ thống hoàn thành trước khi tiếp tục.

* Ví dụ: Workflow xử lý thanh toán có thể cần chờ:

  * Paper check được clear.
  * Paperwork được chuyển đến bởi courier.
  * Paperwork được chuyển đến qua mail carrier.
