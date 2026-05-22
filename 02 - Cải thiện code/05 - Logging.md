# Logging trong Workflows và Activities

## Logging trong Workflows và Activities

* Dùng logging package giúp dễ theo dõi execution path của code khi runtime.

* Temporal Go SDK cung cấp một logging interface an toàn để dùng trong code.

* SDK cũng có một implementation cơ bản cho interface này và dùng mặc định.

* Bạn có thể thay thế logger mặc định bằng:

  * Implementation riêng.
  * Logging package bên thứ ba được adapt theo interface của Temporal.

## Vì sao không dùng trực tiếp third-party logging package?

* Temporal cung cấp Durable Execution cho Workflow.

* Cơ chế này dựa trên quá trình gọi là `History Replay`.

* Khi replay, Temporal Worker khôi phục state của Workflow bằng cách re-execute Workflow Definition một cách an toàn.

* Nếu dùng logging API của Temporal:

  * Worker có thể suppress log messages trong lúc replay.
  * Tránh việc log từ execution gốc bị ghi lại lần nữa khi re-execution.

## Các log levels trong Logger interface

* SDK’s `Logger` interface định nghĩa method tương ứng với 4 log levels.

* Từ ít quan trọng đến quan trọng nhất:

  * `Debug`
  * `Info`
  * `Warn`
  * `Error`

# Sử dụng Logger Interface

## Logger trong Workflow Definition

* Trong Workflow Definition, dùng `workflow.GetLogger` để lấy Workflow Logger.

* Thường nên gọi ở đầu Workflow function để có thể dùng logger trong phần code phía sau.

```go
logger := workflow.GetLogger(ctx)

logger.Debug("Preparing to execute an Activity")
logger.Info("Calculated cost of order", "Tax", tax, "Total", total)
```

* Khi log, gọi method tương ứng với log level.

* Có thể truyền:

  * Một string message.
  * Kèm theo nhiều key-value pairs.

## Thêm key-value pairs mặc định cho mọi log line

* Nếu có key-value pairs muốn xuất hiện trong mọi log line, không cần lặp lại ở từng log statement.

* Có thể import package `go.temporal.io/sdk/log`.

* Dùng `log.With` để tạo logger mới từ logger hiện tại và các key-value pairs mặc định.

```go
logger := log.With(workflow.GetLogger(ctx), "version", 4, "user", "jsmith")

// The output from this statement includes the two key-value pairs above
logger.Debug("Preparing to execute an Activity")

// This includes the key-value pairs above, plus the two specified here
logger.Info("Calculated cost of order", "Tax", tax, "Total", total)
```

## Logger trong Activity Definition

* Go SDK cũng cung cấp cách lấy logger trong Activity Definition.

* Dùng `activity.GetLogger(ctx)`.

```go
logger := activity.GetLogger(ctx)

logger.Info("Looking up customer in the database", "Key", customerID)
logger.Error("Database connection failed")
```

* Có thể log trong Activity tương tự như trong Workflow.

# Giới hạn của Default Logger

## Default Logger khá đơn giản

* Go SDK có sẵn default logging implementation.

* Tuy nhiên, default logger có nhiều giới hạn.

## Log được ghi ra standard output

* Default logger emit message ra standard output stream của Client.

* Log từ Workflow và Activity sẽ xuất hiện trong terminal dùng để start Worker.

* Log không được tự động collect vào log file.

## Không cấu hình được threshold

* Default logger không cho phép đặt minimum threshold cho logging.

* Vì vậy, mọi message đều được hiển thị, bất kể log level.

## Không customize được output

* Default logger không hỗ trợ customize output.

* Output thường chứa nhiều key-value pairs như:

  * Task Queue name.
  * Activity Type.

* Các key-value pairs này xuất hiện trước message bạn log.

* Điều này có thể gây khó đọc khi đang tìm một giá trị cụ thể trong output.

## Activity logger và History Replay

* Activities đã hoàn thành thành công sẽ không được execute lại khi Worker dùng History Replay để recover Workflow state.

* Vì vậy, Activity logger không cần suppress duplicate messages trong replay.

* Lợi ích chính khi dùng Temporal logging API trong Activity:

  * Dùng cùng logging API với Workflow code.
  * Tự động include các key-value pairs của Temporal.

# Tích hợp Logging Implementation khác

## Thay thế default logger

* Thường nên thay default logger bằng logger mạnh hơn.

* Có thể chỉ định logger khác thông qua field `Logger` trong `Client.Options`.

```go
// Override the default implementation and use some other logger 
customLogger := NewAlternateLogger()
c, err := client.Dial(client.Options{
    Logger: customLogger,
})
```

## Cấu hình third-party logger

* Cấu hình của third-party logging package không phải là phần đặc thù của Temporal.

* Bạn cấu hình giống như khi dùng logging package đó trong application bình thường.

* Ví dụ:

  * Set minimum log level.
  * Chỉ định path để tạo log files.

## Yêu cầu với custom Logger

* Logger được chỉ định phải conform với `log.Logger` interface trong Temporal Go SDK.

* Nếu third-party logger không conform trực tiếp với interface này:

  * Có thể viết adapter.
  * Adapter map các function từ Temporal logger interface sang logging package đang dùng.

* Repository `samples-go` của Temporal có ví dụ trong subdirectory `zapadapter`.

* Ví dụ này dùng package `Zap`, một logging package phổ biến trong Go applications.

