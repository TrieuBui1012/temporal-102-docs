# Workflow Definition và Workflow Execution

## Workflow Definition là gì?

* Trong Temporal, phần code định nghĩa business logic chính được gọi là `Workflow Definition`.

* Giống như code thông thường, `Workflow Definition` sẽ không làm gì cho đến khi được execute.

* Để chạy Workflow, cần dùng `Client` để gửi execution request.

* Có thể start Workflow bằng:

  * Command-line tool.
  * Code từ application.

* Cả hai cách đều tạo ra cùng một kết quả: một Workflow đang chạy.

* Trong Temporal, Workflow đang chạy được gọi là `Workflow Execution`.

## Một Workflow Definition có thể chạy nhiều lần

* Một `Workflow Definition` có thể được execute nhiều lần.

* Mỗi lần execute sẽ tạo ra một `Workflow Execution` mới.

* Ví dụ:

  * Chạy cùng một Workflow Definition mỗi sáng để tạo daily report.

## Input của Workflow

* Type của input được định nghĩa trong `Workflow Definition`.

* Nhưng value cụ thể của input được truyền vào trong execution request.

* Rất phổ biến khi chạy cùng một `Workflow Definition` nhiều lần với input khác nhau.

* Ví dụ:

  * Start cùng một Workflow Definition 5,000 lần.
  * Mỗi lần truyền một `customer ID` khác nhau.
  * Mục đích là gửi monthly statement cho từng customer.

# Trạng thái của Workflow Execution

## Hai trạng thái chính

* Một `Workflow Execution` có 2 trạng thái:

  * `open`
  * `closed`

## Open Workflow Execution

* `open` nghĩa là Workflow Execution hiện đang chạy.

* Khi ở trạng thái open, Workflow thường ở một trong hai tình huống:

  * Đang actively making progress.
  * Đang chờ một điều kiện nào đó để có thể tiếp tục.

## Closed Workflow Execution

* `closed` nghĩa là Workflow Execution đã dừng chạy vì một lý do nào đó.

* Ví dụ:

  * Completed successfully.
  * Failed.
  * Terminated.

## Chuyển trạng thái một chiều

* Workflow Execution có thể chuyển từ `open` sang `closed`.

* Đây là chuyển trạng thái một chiều.

* Khi Workflow Execution đã vào trạng thái `closed`, nó sẽ ở đó vĩnh viễn.

## Chạy lại Workflow Definition không phải là mở lại Workflow cũ

* Có thể chạy lại cùng một `Workflow Definition`.

* Thậm chí có thể dùng cùng input.

* Nhưng kết quả vẫn là một `Workflow Execution` mới.

* Mỗi Workflow Execution được định danh duy nhất bằng `Run ID`.

* `Run ID` được Temporal tự động generate khi Workflow Execution được launch.

* Vì vậy, mỗi Workflow Execution mới sẽ có `Run ID` khác với các execution trước đó.

# Workflow đang chạy thì làm gì?

## Progress và waiting

* Khi ở trạng thái `open`, Workflow lặp lại giữa hai việc:

  * Thực hiện progress.
  * Chờ điều kiện để tiếp tục progress.

## Chờ Activity Execution

* Một ví dụ là chờ `Activity Execution`.

* Nếu Workflow code cần dùng value trả về từ Activity, Workflow phải chờ Activity Execution hoàn tất để value có sẵn.

## Chờ Timer

* Một ví dụ khác là chờ `Timer`.

* Nếu Workflow đang block chờ Timer, nó không thể tiếp tục cho đến khi:

  * Timer fire.
  * Hoặc Timer bị canceled.

## Chu kỳ trong Workflow Execution

* Chu kỳ `progress → waiting → progress` tiếp tục trong suốt Workflow Execution.

* Chu kỳ này chỉ dừng lại khi Workflow Execution chuyển sang trạng thái `closed`.

# Các lý do Workflow Execution chuyển sang closed

## Completed

* Lý do tốt nhất là Workflow function trả về result.

* Điều này nghĩa là Workflow completed successfully.

## Continued-as-New

* `Continued-as-New` là một biến thể đặc biệt.

* Nghĩa là code vẫn tiếp tục chạy.

* Nhưng các progress tiếp theo sẽ diễn ra trong một Workflow Execution mới và Event History mới.

### Vì sao cần Continue-As-New?

* Temporal giới hạn:

  * Size của Event History.
  * Số lượng events trong Event History.

* Mục đích là giữ performance tốt.

* Thường sẽ không chạm giới hạn này trừ khi một execution chạy hàng nghìn Activities.

* `Continue-As-New` là kỹ thuật để tránh đạt đến các giới hạn đó.

## Failed

* Workflow Execution có thể close vì execution failed.

* Điều này xảy ra khi Workflow function trả về error thay vì result.

## Timed Out

* Workflow Execution có thể bị timeout.

* Timeout xảy ra khi time limit của execution đã hết trước khi Workflow function trả về:

  * Result.
  * Hoặc error.

## Terminated

* Workflow Execution có thể bị terminated.

* Termination có thể được thực hiện từ:

  * Code.
  * Command line.
  * Web UI.

## Canceled

* Workflow Execution cũng có thể bị canceled.

* Cancellation có thể được khởi tạo từ:

  * Code.
  * Command line.
  * Web UI.

* Cancellation giống termination ở chỗ đều kết thúc execution sớm.

* Nhưng cancellation graceful hơn.

* Workflows và Activities có thể được notified về cancellation và thực hiện cleanup trước khi exit.

# Ý nghĩa khi đọc Event History

## Open và closed

* `Open Workflow Execution` là execution hiện đang chạy.

* Cuối cùng, mọi Workflow Execution đều sẽ chuyển sang trạng thái `closed`.

* Khi closed, Workflow Execution sẽ có final status tương ứng với lý do kết thúc.

## Vì sao cần hiểu các trạng thái closed?

* Hiểu sự khác nhau giữa các trạng thái kết thúc giúp đọc `Workflow Execution Event History` tốt hơn.

* Điều này giúp xác định nguồn gốc vấn đề.

* Ví dụ:

  * Nếu Workflow Execution kết thúc với status `failed`, ta biết Workflow function đã trả về error.
  * Event cuối cùng trong history của Workflow Execution sẽ chứa thông tin về error đó.
  * Thông tin này cũng được hiển thị trong Web UI.
