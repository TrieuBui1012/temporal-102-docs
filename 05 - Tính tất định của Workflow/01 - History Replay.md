# History Replay: Cách Temporal cung cấp Durable Execution

## Durable Execution là gì?

* Temporal đảm bảo `Workflow` có thể chạy bền vững dù môi trường bên dưới gặp sự cố.

* Ví dụ:

  * Process bị crash
  * Server bị reboot trong lúc maintenance
  * Service bên ngoài tạm thời offline

* Với Temporal, developer có thể viết `Workflow` như thể những sự cố đó không làm mất trạng thái execution.

## Khác với function thông thường

* Nếu một function Go thông thường bị crash khi đang chạy:

  * State trong memory bị mất.
  * Function không thể resume.
  * Chỉ có thể restart từ đầu.

* Với Temporal `Workflow`, từ góc nhìn developer:

  * Workflow có thể tiếp tục từ gần điểm bị crash.
  * State trước đó được khôi phục.
  * Execution tiếp tục như thể crash chưa từng xảy ra.

## Cơ chế chính: History Replay

* Temporal không yêu cầu developer tự viết checkpoint hoặc tự quản lý state.

* Temporal dùng `History Replay`, còn gọi là `Workflow Replay`, để reconstruct lại state của `Workflow Execution`.

* Worker sẽ dùng `Event History` để chạy lại `Workflow code` từ đầu, nhưng không lặp lại các side effects đã được ghi nhận trước đó.

# Code được dùng trong ví dụ

## PizzaWorkflow

* Video dùng ví dụ `PizzaWorkflow`.

* Code trong video là code Go thật, không phải pseudocode, vì cách assign kết quả từ `Activity Execution` rất quan trọng để hiểu replay.

```go
func PizzaWorkflow(ctx workflow.Context, order PizzaOrder) (string, error) {
	logger := workflow.GetLogger(ctx)

	options := workflow.ActivityOptions{
		StartToCloseTimeout: time.Second * 5,
	}

	ctx = workflow.WithActivityOptions(ctx, options)

	var totalPrice int
	for _, pizza := range order.Items {
		totalPrice += pizza.Price
	}

	logger.Info("Calculated cost of order", "Total", totalPrice)

	var distance Distance
	future := workflow.ExecuteActivity(ctx, GetDistance, order.Address)
	_ = future.Get(ctx, &distance)

	if order.IsDelivery && distance.Kilometers > 25 {
		return "", errors.New("Customer lives too far away for delivery")
	}

	_ = workflow.Sleep(ctx, time.Minute * 30)

	// call a local function to create the input passed to next Activity
	bill := createBill(order, totalPrice)

	var confirmation OrderConfirmation
	future = workflow.ExecuteActivity(ctx, SendBill, bill)
	_ = future.Get(ctx, &confirmation)

	return confirmation, nil
}
```

# Workflow Execution bắt đầu

## Cluster ghi Event khởi đầu

* Khi có request start `Workflow Execution`, Temporal kết hợp:

  * `Workflow Definition`
  * Input data truyền vào Workflow

* Trong ví dụ, input chứa thông tin:

  * Customer
  * Danh sách pizza đã order

* Cluster ghi Event đầu tiên vào history:

```text
WorkflowExecutionStarted
```

* Event này chứa input data ban đầu của Workflow.

## Cluster schedule Workflow Task đầu tiên

* Sau đó, cluster thêm một `Workflow Task` vào `Task Queue`.

* Cluster ghi Event:

```text
WorkflowTaskScheduled
```

## Worker nhận Workflow Task

* Khi `Worker` poll `Task Queue` và nhận Task, cluster ghi:

```text
WorkflowTaskStarted
```

* Worker invoke `PizzaWorkflow` và bắt đầu chạy code từng statement một.

# Các statement chạy trong Worker

## Một số statement không tương tác với Cluster

* Các đoạn code như tạo logger, set `ActivityOptions`, tính `totalPrice` chỉ chạy trong Worker.

* Những statement này không trực tiếp tạo `Command` gửi đến cluster.

```go
logger := workflow.GetLogger(ctx)

options := workflow.ActivityOptions{
	StartToCloseTimeout: time.Second * 5,
}

ctx = workflow.WithActivityOptions(ctx, options)

var totalPrice int
for _, pizza := range order.Items {
	totalPrice += pizza.Price
}

logger.Info("Calculated cost of order", "Total", totalPrice)
```

# Khi Workflow gọi Activity

## ExecuteActivity tạo yêu cầu schedule Activity

* Khi Workflow chạy đến:

```go
future := workflow.ExecuteActivity(ctx, GetDistance, order.Address)
_ = future.Get(ctx, &distance)
```

* `workflow.ExecuteActivity` tạo ra một `Command` yêu cầu Temporal schedule Activity `GetDistance`.

* Tuy nhiên, cần hiểu chính xác:

  * `ActivityTask` **không được schedule ngay tại thời điểm gọi `ExecuteActivity` trong lúc Workflow Task vẫn đang chạy**.
  * Command được buffer lại trong Worker.
  * Chỉ khi `Workflow Task` hiện tại complete, Worker mới gửi Command này về cluster.
  * Sau đó cluster mới ghi Event và schedule `Activity Task`.

## Workflow không thể progress thêm nếu chưa có Activity result

* Sau `ExecuteActivity`, code gọi:

```go
_ = future.Get(ctx, &distance)
```

* `future.Get` cần result của Activity để tiếp tục.

* Vì Workflow chưa có kết quả `distance`, nó không thể progress thêm.

* Đây là lúc `Workflow Task` hiện tại complete.

* Worker gửi Command schedule Activity về cluster.

## Cluster schedule Activity Task

* Cluster nhận Command và ghi Event:

```text
ActivityTaskScheduled
```

* Event này thể hiện `Activity Task` đã được đưa vào `Task Queue`.

* Sau đó Worker có thể poll và nhận `Activity Task`.

## Activity được execute

* Khi một Worker nhận `Activity Task`, cluster ghi:

```text
ActivityTaskStarted
```

* Worker chạy Activity:

```text
GetDistance
```

* Giả sử Activity trả về:

```text
15
```

* Worker báo kết quả về cluster.

* Cluster ghi:

```text
ActivityTaskCompleted
```

* Event này chứa result của Activity.

# Cluster tiếp tục Workflow sau Activity

## Schedule Workflow Task mới

* Sau khi `ActivityTaskCompleted`, cluster cần tiếp tục `Workflow Execution`.

* Cluster schedule một `Workflow Task` mới.

* Worker poll, nhận Task và tiếp tục chạy Workflow code từ sau đoạn lấy result Activity.

## Kiểm tra điều kiện delivery

* Vì `distance` đã có giá trị, Workflow có thể chạy tiếp:

```go
if order.IsDelivery && distance.Kilometers > 25 {
	return "", errors.New("Customer lives too far away for delivery")
}
```

* Nếu điều kiện false, Workflow tiếp tục sang bước tiếp theo.

# Khi Workflow gọi Timer

## workflow.Sleep tạo Timer Command

* Workflow chạy đến:

```go
_ = workflow.Sleep(ctx, time.Minute * 30)
```

* Tương tự Activity, lệnh này tạo Command yêu cầu cluster tạo Timer.

* Workflow không thể progress thêm cho đến khi Timer fire.

* Worker complete `Workflow Task` hiện tại và gửi Command về cluster.

## Cluster ghi TimerStarted

* Cluster ghi:

```text
TimerStarted
```

* Trong lúc chờ Timer:

  * Cluster không schedule thêm `Workflow Task` cho Workflow này.
  * Worker không bị giữ tài nguyên để chờ.
  * Timer dù kéo dài nhiều năm cũng không làm Worker bị block.

## Timer fire

* Sau 30 phút, cluster ghi:

```text
TimerFired
```

* Cluster schedule `Workflow Task` mới để Workflow tiếp tục.

# Khi Worker crash

## Worker crash sau khi Timer fire

* Giả sử Worker nhận `Workflow Task` sau khi Timer fire.

* Worker tiếp tục chạy code nhưng bị crash ngay sau đó.

* Temporal cần khôi phục Workflow về đúng state trước crash.

## Cluster phát hiện crash bằng Timeout

* Worker là thành phần bên ngoài cluster.

* Worker chủ động poll `Task Queue`, nhận Task và phải complete Task trong thời gian cho phép.

* Vì crash xảy ra khi Worker đang giữ `Workflow Task`, timeout liên quan là:

```text
Workflow Task Timeout
```

* Giá trị mặc định:

```text
10 seconds
```

* Nếu Worker không complete `Workflow Task` trong thời gian này, cluster xem như Worker đã down.

## Reschedule Workflow Task

* Cluster schedule một `Workflow Task` mới.

* Task mới này không được đưa vào `Sticky Queue`.

* Thay vào đó, nó được đưa về `Task Queue` gốc.

* Nhờ vậy, bất kỳ Worker nào đang poll queue này cũng có thể nhận Task và tiếp tục Workflow.

# Worker mới thực hiện History Replay

## Worker lấy Event History

* Worker mới cần biết trạng thái hiện tại của Workflow.

* Nó request `Event History` từ cluster.

* Worker stream history về và lưu trong local cache.

## Worker chạy lại Workflow code từ đầu

* Worker re-execute `PizzaWorkflow` từ đầu.

* Input được lấy lại từ Event:

```text
WorkflowExecutionStarted
```

* Vì `Workflow code` phải deterministic, các biến được tính lại sẽ có cùng giá trị như trước crash.

# Replay xử lý Activity như thế nào?

## Không execute lại Activity đã hoàn tất

* Khi replay đến đoạn:

```go
future := workflow.ExecuteActivity(ctx, GetDistance, order.Address)
_ = future.Get(ctx, &distance)
```

* Worker có thể tạo Command tương ứng, nhưng không gửi Command mới lên cluster ngay.

* Thay vào đó, Worker kiểm tra `Event History`.

* History đã có các Events:

```text
ActivityTaskScheduled
ActivityTaskStarted
ActivityTaskCompleted
```

* Vì `ActivityTaskCompleted` đã chứa result trước đó, Worker reuse result này.

* Trong ví dụ, `distance` được assign lại từ kết quả cũ:

```text
15
```

## Vì sao Activity không cần deterministic?

* `Activity code` không bắt buộc phải deterministic.

* Lý do là trong replay, Activity đã hoàn tất sẽ không bị chạy lại.

* Worker chỉ lấy result đã được lưu trong `Event History`.

* Điều này tránh việc Activity trả về kết quả khác so với lần chạy ban đầu.

# Replay xử lý Timer như thế nào?

## Không tạo lại Timer đã fire

* Khi Worker replay đến:

```go
_ = workflow.Sleep(ctx, time.Minute * 30)
```

* Worker kiểm tra history.

* Nếu history đã có:

```text
TimerStarted
TimerFired
```

* Worker biết Timer đã được tạo và đã fire trước crash.

* Vì vậy, Worker không gửi Command tạo Timer mới.

# State được khôi phục đến điểm crash

## Replay khôi phục từng biến trong Workflow

* Khi Worker chạy lại từng statement, state của Workflow được tái tạo.

* Ví dụ:

  * `totalPrice` được tính lại giống trước.
  * `distance` được lấy lại từ `ActivityTaskCompleted`.
  * Điều kiện delivery evaluate giống lần chạy trước.

* Khi Worker replay đến đúng điểm crash, state đã tương đương với state trước crash.

## Sau điểm crash, Workflow chạy tiếp bình thường

* Khi Worker đi qua điểm crash, `Event History` chưa có Events tương ứng với các bước tiếp theo.

* Từ đây, Worker tiếp tục chạy như execution bình thường.

# Activity tiếp theo: SendBill

## Tạo bill

* Workflow gọi local function:

```go
bill := createBill(order, totalPrice)
```

* Đây là code local trong Workflow, không phải Activity.

* Nó chỉ tạo input để truyền vào Activity tiếp theo.

## Execute SendBill Activity

* Workflow chạy đến:

```go
future = workflow.ExecuteActivity(ctx, SendBill, bill)
_ = future.Get(ctx, &confirmation)
```

* Vì đây là phần sau điểm crash và history chưa có Events tương ứng, Worker sẽ gửi Command mới khi `Workflow Task` complete.

* Cluster schedule `Activity Task`.

* Worker execute Activity `SendBill`.

* Khi hoàn tất, cluster ghi:

```text
ActivityTaskCompleted
```

# Workflow hoàn tất

## Worker return kết quả

* Sau khi có `confirmation`, Workflow return:

```go
return confirmation, nil
```

## CompleteWorkflowExecution

* Worker complete `Workflow Task`.

* Vì Workflow function đã chạy xong thành công, Worker gửi Command:

```text
CompleteWorkflowExecution
```

* Command này chứa result mà Workflow return.

## Event cuối cùng

* Cluster ghi Event cuối cùng:

```text
WorkflowExecutionCompleted
```

* `Workflow Execution` lúc này closed.

# Ghi nhớ

* `History Replay` là cơ chế Temporal dùng để khôi phục state của `Workflow Execution`.

* Worker replay `Workflow code` từ đầu bằng input được lưu trong `WorkflowExecutionStarted`.

* `ActivityTask` không được schedule ngay tại dòng `ExecuteActivity` nếu `Workflow Task` vẫn đang chạy.

* `ExecuteActivity` tạo Command; Command chỉ được gửi về cluster khi `Workflow Task` complete.

* Workflow thường complete Task khi không thể progress thêm, ví dụ đang chờ `future.Get` hoặc `workflow.Sleep`.

* Trong replay, Worker không chạy lại Activity đã hoàn tất, mà lấy result từ `ActivityTaskCompleted`.

* Timer đã start và fire cũng không bị tạo lại trong replay.

* `Workflow Task Timeout` giúp cluster phát hiện Worker crash khi Worker không complete Task đúng hạn.

* Sau crash, Workflow Task mới được đưa về `Task Queue` gốc thay vì `Sticky Queue`.

* `Workflow code` cần deterministic để replay tạo lại cùng state.

* Hiểu `History Replay` là nền tảng để hiểu các khái niệm nâng cao như `non-deterministic errors` và `Workflow Reset`.
