# Workflow Pizza Order: Workflow code, Commands và Events

## Bối cảnh ví dụ

* Ví dụ này mô phỏng một Workflow xử lý pizza order đơn giản.

* Code trong bài là **pseudocode**, không phải Go code chuẩn để dùng trong production.

* Pseudocode được đơn giản hóa để tập trung vào concept chính:

  * Workflow thực hiện business logic như thế nào.
  * Khi nào Workflow chỉ chạy logic nội bộ.
  * Khi nào Worker gửi `Command` tới Temporal Cluster.
  * Temporal Cluster ghi `Event` nào vào `Event History`.

* Một số chi tiết đã được lược bỏ hoặc viết đơn giản hơn thực tế:

  * Error handling.
  * Cú pháp Go SDK đầy đủ.
  * Cách lấy result từ Activity bằng `Future.Get`.
  * Return value dạng `struct`.

## Pseudocode trong bài

```go
func PizzaWorkflow(ctx workflow.Context, order Order) (string, error) {

    options := workflow.ActivityOptions{
        StartToCloseTimeout: time.Second * 5,
    }
    ctx = workflow.WithActivityOptions(ctx, options)

    // Iterate over the items and calculate the cost of the order
    var totalPrice int
    for pizza := order.Items {
        totalPrice += pizza.Price
    }

    distance := workflow.ExecuteActivity(ctx, GetDistance, order.Address)

    if order.IsDelivery && distance > 25 {
        return "", errors.New("Customer too far away for delivery")
    }

    // Wait 30 minutes before billing the customer
    workflow.Sleep(ctx, time.Minute * 30)

    bill := Bill{
        CustomerId:  order.Customer.CustomerId,
        Amount:      totalPrice,
        Description: order.OrderNumber,
    }

    confirmation = workflow.ExecuteActivity(ctx, SendBill, bill)

    return confirmation, nil
}
```

# Luồng xử lý tổng quát

## Workflow xử lý những gì?

* Workflow nhận một `Order`.

* Workflow tính tổng giá tiền của các pizza trong order.

* Workflow gọi Activity `GetDistance` để lấy khoảng cách đến địa chỉ customer.

* Nếu customer chọn delivery nhưng ở xa hơn 25 km, Workflow trả về error.

* Nếu hợp lệ, Workflow chờ 30 phút để giả lập thời gian nấu và giao hàng.

* Sau đó Workflow tạo `Bill`.

* Workflow gọi Activity `SendBill` để bill customer.

* Cuối cùng Workflow return `confirmation`.

# Pseudocode khác gì Go SDK thực tế?

## `ExecuteActivity` không trả result trực tiếp

* Trong pseudocode, đoạn này được viết như thể `ExecuteActivity` trả về trực tiếp `distance`:

```go
distance := workflow.ExecuteActivity(ctx, GetDistance, order.Address)
```

* Nhưng trong Go SDK thực tế, `workflow.ExecuteActivity(...)` trả về một `Future`.

* Muốn lấy result từ Activity, cần gọi `.Get(...)`.

```go
var distance int
err := workflow.ExecuteActivity(ctx, GetDistance, order.Address).Get(ctx, &distance)
if err != nil {
    return "", err
}
```

* Tương tự, đoạn này trong pseudocode:

```go
confirmation = workflow.ExecuteActivity(ctx, SendBill, bill)
```

* Trong Go SDK thực tế nên hiểu gần giống như:

```go
var confirmation string
err = workflow.ExecuteActivity(ctx, SendBill, bill).Get(ctx, &confirmation)
if err != nil {
    return "", err
}
```

## Code gần với Go SDK thực tế hơn

```go
func PizzaWorkflow(ctx workflow.Context, order Order) (string, error) {
    options := workflow.ActivityOptions{
        StartToCloseTimeout: time.Second * 5,
    }
    ctx = workflow.WithActivityOptions(ctx, options)

    var totalPrice int
    for _, pizza := range order.Items {
        totalPrice += pizza.Price
    }

    var distance int
    err := workflow.ExecuteActivity(ctx, GetDistance, order.Address).Get(ctx, &distance)
    if err != nil {
        return "", err
    }

    if order.IsDelivery && distance > 25 {
        return "", errors.New("Customer too far away for delivery")
    }

    err = workflow.Sleep(ctx, time.Minute*30)
    if err != nil {
        return "", err
    }

    bill := Bill{
        CustomerId:  order.Customer.CustomerId,
        Amount:      totalPrice,
        Description: order.OrderNumber,
    }

    var confirmation string
    err = workflow.ExecuteActivity(ctx, SendBill, bill).Get(ctx, &confirmation)
    if err != nil {
        return "", err
    }

    return confirmation, nil
}
```

# Hai loại statement trong Workflow

## Logic nội bộ trong Workflow

* Một số statement chỉ chạy bên trong Workflow Worker.

* Những statement này không tạo interaction với Temporal Cluster.

* Ví dụ:

```go
options := workflow.ActivityOptions{
    StartToCloseTimeout: time.Second * 5,
}
ctx = workflow.WithActivityOptions(ctx, options)
```

```go
var totalPrice int
for _, pizza := range order.Items {
    totalPrice += pizza.Price
}
```

```go
bill := Bill{
    CustomerId:  order.Customer.CustomerId,
    Amount:      totalPrice,
    Description: order.OrderNumber,
}
```

* Đây là các bước như code bình thường:

  * Set options.
  * Tính toán.
  * Kiểm tra biến.
  * Tạo data structure.

## Statement tương tác với Temporal Cluster

* Một số Temporal SDK APIs khiến Worker gửi `Command` tới Temporal Cluster.

* Ví dụ:

```go
workflow.ExecuteActivity(ctx, GetDistance, order.Address)
```

```go
workflow.Sleep(ctx, time.Minute * 30)
```

```go
workflow.ExecuteActivity(ctx, SendBill, bill)
```

* Ngoài ra, việc Workflow return success hoặc return error cũng khiến Worker gửi Command tương ứng.

# Command là gì?

* `Command` là yêu cầu do Worker gửi tới Temporal Cluster.

* Command nói rằng Workflow muốn Temporal thực hiện một hành động nào đó.

* Ví dụ trong Workflow này:

```text
ScheduleActivityTask
StartTimer
CompleteWorkflowExecution
FailWorkflowExecution
```

* Command không phải là Event.

* Command là yêu cầu được gửi đi.

* Event là bản ghi được Temporal Cluster lưu lại sau khi điều gì đó xảy ra.

# Event là gì?

* `Event` là bản ghi được Temporal Cluster append vào `Event History`.

* Event History là lịch sử durable của Workflow Execution.

* Event History được lưu trong database của Temporal Cluster.

* Vì vậy, Event History vẫn tồn tại kể cả khi Worker hoặc Temporal Cluster gặp sự cố.

* Worker cũng có thể cache Events để tăng performance, nhưng nguồn sự thật vẫn là Event History trong Temporal Cluster.

# Flow khi gọi Activity

## 1. Workflow gọi `ExecuteActivity`

```go
workflow.ExecuteActivity(ctx, GetDistance, order.Address)
```

* Worker đang execute Workflow code gặp lời gọi Activity.

* Worker gửi Command tới Temporal Cluster:

```text
ScheduleActivityTask
```

## 2. Temporal Cluster tạo Activity Task

* Temporal Cluster nhận `ScheduleActivityTask`.

* Cluster tạo một `Activity Task`.

* Activity Task được đưa vào Task Queue đã cấu hình.

* Lưu ý: Cluster tạo **Activity Task**, không phải tạo Task Queue mới mỗi lần gọi Activity.

* Cluster ghi Event vào Event History:

```text
ActivityTaskScheduled
```

## 3. Activity Worker nhận task

* Một Worker đang poll Task Queue sẽ nhận Activity Task.

* Worker này có thể là:

  * Cùng Worker đang chạy Workflow.
  * Hoặc một Worker khác đang poll cùng Task Queue.

* Khi Activity Task được giao cho Worker, Temporal Cluster ghi Event:

```text
ActivityTaskStarted
```

## 4. Activity hoàn thành

* Activity Worker execute code trong Activity Definition.

* Khi Activity function return result, Worker notify Temporal Cluster rằng task đã hoàn thành.

* Đây là notification, không phải Command trong ngữ cảnh bài học này.

* Temporal Cluster ghi Event:

```text
ActivityTaskCompleted
```

## 5. Workflow tiếp tục

* Nếu Workflow đang chờ result của Activity, Workflow sẽ tiếp tục sau khi Activity hoàn thành.

* Trong Go SDK thực tế, Workflow chờ bằng `.Get(...)`.

```go
var distance int
err := workflow.ExecuteActivity(ctx, GetDistance, order.Address).Get(ctx, &distance)
```

# Flow khi dùng Timer

## 1. Workflow gọi `workflow.Sleep`

```go
workflow.Sleep(ctx, time.Minute * 30)
```

* Worker gửi Command tới Temporal Cluster:

```text
StartTimer
```

## 2. Temporal Cluster start Timer

* Cluster tạo Timer với duration 30 phút.

* Cluster ghi Event:

```text
TimerStarted
```

## 3. Timer fire

* Sau khi 30 phút trôi qua, Timer fire.

* Temporal Cluster ghi Event:

```text
TimerFired
```

## 4. Workflow resume

* Workflow tiếp tục chạy statement tiếp theo sau khi Timer fire.

* Timer do Temporal Cluster quản lý.

* Worker không cần giữ thread hoặc tài nguyên trong lúc chờ Timer.

# Flow khi Workflow kết thúc

## Workflow return error

```go
return "", errors.New("Customer too far away for delivery")
```

* Nếu Workflow return error, Worker gửi Command:

```text
FailWorkflowExecution
```

* Temporal Cluster ghi Event:

```text
WorkflowExecutionFailed
```

* Workflow Execution chuyển sang trạng thái closed với status failed.

## Workflow return success

```go
return confirmation, nil
```

* Nếu Workflow return result và error là `nil`, Worker gửi Command:

```text
CompleteWorkflowExecution
```

* Temporal Cluster ghi Event:

```text
WorkflowExecutionCompleted
```

* Workflow Execution chuyển sang trạng thái closed với status completed.

# Mapping giữa Workflow code, Command và Event

| Workflow action            | Command Worker gửi          | Events thường thấy                                                      |
| -------------------------- | --------------------------- | ----------------------------------------------------------------------- |
| Gọi `GetDistance` Activity | `ScheduleActivityTask`      | `ActivityTaskScheduled`, `ActivityTaskStarted`, `ActivityTaskCompleted` |
| Gọi `workflow.Sleep`       | `StartTimer`                | `TimerStarted`, `TimerFired`                                            |
| Gọi `SendBill` Activity    | `ScheduleActivityTask`      | `ActivityTaskScheduled`, `ActivityTaskStarted`, `ActivityTaskCompleted` |
| Workflow return error      | `FailWorkflowExecution`     | `WorkflowExecutionFailed`                                               |
| Workflow return success    | `CompleteWorkflowExecution` | `WorkflowExecutionCompleted`                                            |

# Ghi chú về Task Queue

* Task Queue là nơi Temporal Cluster đặt các task cần Worker xử lý.

* Khi gọi Activity, Temporal Cluster tạo `Activity Task` và đưa vào Task Queue.

* Worker nào poll đúng Task Queue và có capacity thì có thể nhận task.

* Temporal đảm bảo một task chỉ được giao cho một Worker tại một thời điểm.

* Nếu Worker nhận task nhưng không hoàn thành trong timeout đã cấu hình, Temporal có thể reschedule task.

# Ghi chú về `StartToCloseTimeout`

* Trong ví dụ, Activity Options có:

```go
StartToCloseTimeout: time.Second * 5
```

* Điều này nghĩa là sau khi Activity Worker bắt đầu xử lý Activity Task, Activity phải hoàn thành trong vòng 5 giây.

* Nếu không hoàn thành trong thời gian này, Activity có thể bị timeout.

* Timeout và retry sẽ được Temporal xử lý theo cấu hình tương ứng.
