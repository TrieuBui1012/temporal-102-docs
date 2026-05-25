# Tổng quan về Event History

## Event History là gì?

* Mỗi `Workflow Execution` đều có một `Event History` đi kèm.

* `Event History` là **single source of truth** cho những gì đã xảy ra trong quá trình chạy của `Workflow Execution`.

* `Temporal Cluster` duy trì lịch sử này bằng cách:

  * Nhận request từ `Clients` và `Workers`.
  * Append các `Events` mới vào history.
  * Lưu lại góc nhìn của `Temporal Cluster` về quá trình execution.

## Event History được lưu ở đâu?

* `Event History` được persist vào database mà `Temporal Cluster` sử dụng.

* Vì được lưu bền vững nên history vẫn tồn tại ngay cả khi `Temporal Cluster` bị crash.

* Đây là nền tảng quan trọng giúp Temporal hỗ trợ `Durable Execution`.

## Vai trò chính của Event History

### Khôi phục Workflow sau khi Worker crash

* Khi `Worker` bị crash, Temporal có thể dùng các `Events` trong history để reconstruct lại state của `Workflow Execution`.

* Nhờ đó, Workflow có thể tiếp tục chạy từ trạng thái đã được ghi nhận, thay vì mất toàn bộ tiến trình.

### Hỗ trợ debug và điều tra execution

* Developer có thể xem lại các bước đã xảy ra trong một `Workflow Execution`.

* Không chỉ xem được execution hiện tại, mà còn có thể xem lại execution đã chạy trong quá khứ.

* Đây là một lợi ích lớn so với ứng dụng truyền thống, nơi thông tin lịch sử thường khó truy vết hơn.

## Cách truy cập Event History

* Dù `Event History` được lưu trong database của cluster, developer vẫn có thể truy cập thông qua:

  * Code sử dụng `Temporal SDK`
  * Temporal command-line tool
  * Temporal Web UI

# Nội dung của Event History

## Event History là append-only log

* `Event History` hoạt động như một ordered append-only log của các `Events`.

* Các `Events` được ghi theo thứ tự tuần tự.

* Mỗi `Event` mới sẽ được append sau `Event` trước đó.

## Event là immutable

* Khi một `Event` đã được ghi vào history:

  * Không thể thay đổi nội dung của nó.
  * Không thể thay đổi vị trí của nó trong history.

## Event đầu tiên và cuối cùng

* Nếu `Workflow Execution` được start thành công, `Event` đầu tiên luôn là:

```text
WorkflowExecutionStarted
```

* Các `Events` tiếp theo sẽ phụ thuộc vào:

  * Nội dung trong `Workflow Definition`
  * Những gì thực sự xảy ra trong quá trình execution

* `Event History` kết thúc khi `Workflow Execution` đóng lại.

* Nếu Workflow chạy thành công, `Event` cuối cùng thường là:

```text
WorkflowExecutionCompleted
```

# Bảo vệ dữ liệu nhạy cảm

## Temporal có lưu dữ liệu gì?

* `Temporal Cluster` và `Temporal Cloud` không có quyền truy cập vào code của ứng dụng.

* Tuy nhiên, để có thể reconstruct state của Workflow sau crash, Temporal cần lưu `Event History`.

* `Event History` có thể chứa:

  * Input truyền vào `Workflows`
  * Output trả về từ `Workflows`
  * Input truyền vào `Activities`
  * Output trả về từ `Activities`

* Các giá trị này đôi khi có thể chứa dữ liệu nhạy cảm, ví dụ:

  * Dữ liệu y tế
  * Dữ liệu tài chính

## TLS bảo vệ dữ liệu khi truyền

* Temporal hỗ trợ `TLS` để bảo vệ dữ liệu khi truyền:

  * Giữa `Client` và `Frontend`
  * Giữa các service bên trong `Temporal Cluster`

* Tuy nhiên, `TLS` chỉ bảo vệ dữ liệu khi truyền qua network, không trực tiếp giải quyết việc dữ liệu được lưu trong database của cluster.

## Dùng Data Encoder để mã hóa dữ liệu lưu trong Event History

* Vì dữ liệu trong `Event History` cần được lưu lại để Temporal hoạt động đúng, cách bảo vệ dữ liệu nhạy cảm là mã hóa payload trước khi gửi vào cluster.

* Ứng dụng có thể tạo custom `Data Encoder` để:

  * Encrypt dữ liệu trước khi gửi đến `Temporal Cluster` hoặc `Temporal Cloud`.
  * Lưu dữ liệu trong cluster dưới dạng đã mã hóa.
  * Decrypt dữ liệu khi `Worker` nhận lại từ cluster.

* Nhờ vậy, dữ liệu nhạy cảm chỉ có thể đọc được bên trong application, nơi có logic encrypt/decrypt tương ứng.