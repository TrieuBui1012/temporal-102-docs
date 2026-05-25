# Trạng thái của Workflow Task và Activity Task

## Mục tiêu của phần này

* Phần này giúp hiểu pattern trong tên `Events`.

* Khi đọc `Event History`, chỉ cần nhìn tên `Event` là có thể đoán được:

  * Task đang ở trạng thái nào
  * Ai thực hiện hành động đó: `Temporal Cluster` hay `Worker`
  * Timeout đó đang đo khoảng thời gian nào

# Pattern trong tên Activity-related Events

## Bỏ phần `ActivityTask` để thấy trạng thái của Task

* Các `Events` liên quan đến `Activity Task` thường có tên theo pattern:

```text
ActivityTask<STATE>
```

* Nếu bỏ phần `ActivityTask`, phần còn lại thường thể hiện trạng thái của Task tại thời điểm `Event` xảy ra.

* Ví dụ:

```text
ActivityTaskScheduled
ActivityTaskStarted
ActivityTaskCompleted
```

* Có thể hiểu lần lượt là:

```text
Scheduled
Started
Completed
```

# Các trạng thái chính của Activity Task

## Scheduled

* Các `Events` kết thúc bằng `Scheduled` cho biết Task đã được thêm vào `Task Queue`.

* Đây là hành động do `Temporal Cluster` thực hiện.

* `Scheduled` luôn là `Event` đầu tiên trong chuỗi xử lý của Task.

* Ví dụ:

```text
ActivityTaskScheduled
```

* Ý nghĩa:

  * `Temporal Cluster` đã schedule một `Activity Task`.
  * Task đã được đưa vào `Task Queue`.
  * Task đang chờ `Worker` poll và xử lý.

## Started

* Các `Events` kết thúc bằng `Started` cho biết `Worker` đã dequeue Task từ `Task Queue`.

* Ví dụ:

```text
ActivityTaskStarted
```

* Ý nghĩa:

  * Một `Worker` đã nhận Task.
  * `Worker` bắt đầu execute code của `Activity`.

## Completed

* Các `Events` kết thúc bằng `Completed` cho biết `Worker` đã xử lý Task thành công.

* Ví dụ:

```text
ActivityTaskCompleted
```

* Ý nghĩa:

  * Code của `Activity` chạy xong.
  * Không có error.
  * `Worker` báo kết quả thành công về `Temporal Cluster`.

# Closed states của Activity Task

## Activity Task có thể kết thúc thành công hoặc thất bại

* Giống như `Workflow Execution`, `Activity Task` cũng có các trạng thái đóng lại.

* Task không chỉ có thể `Completed`, mà còn có thể:

  * `Failed`
  * `TimedOut`
  * Các trạng thái đóng khác tùy từng loại Event

## Worker quyết định Completed hay Failed

* `Worker` là bên execute code của `Activity Task`.

* Nếu code chạy xong mà không có error:

```text
ActivityTaskCompleted
```

* Nếu code chạy và trả về error:

```text
ActivityTaskFailed
```

* Vì vậy, `Worker` là bên xác định Task thành công hay thất bại dựa trên kết quả chạy code.

## Temporal Cluster quyết định Timed Out

* `Temporal Cluster` là bên xác định Task có bị timeout hay không.

* Cluster dựa vào việc:

  * `Worker` có báo kết quả về trước khi thời gian cho phép kết thúc hay không.

* Nếu hết thời gian mà Cluster chưa nhận được kết quả từ `Worker`, Task có thể bị timeout.

* Ví dụ:

```text
ActivityTaskTimedOut
```

## Vì sao Cluster là bên quyết định timeout?

* Một lý do phổ biến khiến Task timeout là `Worker` bị crash.

* Nếu `Worker` đã crash, nó không thể tự báo rằng Task bị timeout.

* Vì vậy, `Temporal Cluster` phải là bên theo dõi thời gian và quyết định timeout.

# Liên hệ với Timeout names

## Hiểu trạng thái giúp hiểu tên Timeout

* Khi hiểu các trạng thái như `Scheduled`, `Started`, `Completed`, `Failed`, `TimedOut`, ta sẽ dễ hiểu tên của các loại Timeout.

## Ví dụ: Start-to-Close Timeout

* `Start-to-Close Timeout` là thời gian tối đa được phép tính từ lúc:

  * `Worker` bắt đầu xử lý Task

* Cho đến lúc:

  * Task đi vào một closed state

* Nói cách khác, đây là khoảng thời gian tối đa giữa:

  * Step 2: `Worker` bắt đầu execution
  * Step 3: Execution kết thúc

* Nếu quá thời gian này mà Task chưa đóng lại, Task có thể bị timeout.

```text
Started -> Closed
```

* Closed state có thể là:

  * `Completed`
  * `Failed`
  * `TimedOut`
  * Hoặc trạng thái kết thúc khác

# Workflow Task cũng theo pattern tương tự

## Pattern không chỉ áp dụng cho Activity Task

* Pattern trong tên `Events` cũng áp dụng cho `Workflow Task`.

* Các `Workflow Task Events` cũng thường thể hiện trạng thái của Task thông qua suffix trong tên Event.

## Ví dụ Workflow Task Events

```text
WorkflowTaskScheduled
WorkflowTaskStarted
WorkflowTaskCompleted
```

* Có thể hiểu lần lượt là:

  * `WorkflowTaskScheduled`: `Temporal Cluster` đưa `Workflow Task` vào `Task Queue`.
  * `WorkflowTaskStarted`: `Worker` lấy Task ra khỏi `Task Queue`.
  * `WorkflowTaskCompleted`: `Worker` xử lý Task thành công.

## Ý nghĩa khi đọc Event History

* Khi nhìn vào tên `Workflow Task Events` hoặc `Activity Task Events`, có thể nhanh chóng hiểu được Task đang đi qua bước nào.

* Điều này giúp đọc `Event History` trong `Temporal Web UI` hiệu quả hơn.