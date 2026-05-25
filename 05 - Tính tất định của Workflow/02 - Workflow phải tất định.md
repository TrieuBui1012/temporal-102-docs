# Vì sao Temporal yêu cầu Workflow phải deterministic

## Ý chính của phần này

* `Workflow code` trong Temporal bắt buộc phải chạy theo cách deterministic.

* Lý do là `History Replay` chỉ hoạt động đúng nếu mỗi lần replay tạo ra cùng một chuỗi `Commands`.

* Nếu replay tạo ra `Commands` khác với những gì `Event History` đang ghi nhận, Worker không thể khôi phục state chính xác và sẽ gặp `non-deterministic error`. 

# Code ví dụ non-deterministic Workflow

## NonDeterministicWorkflow

* Ví dụ trong video dùng một Workflow có logic phụ thuộc vào random number.

* Đây là ví dụ vi phạm yêu cầu deterministic của Temporal.

```go
func NonDeterministicWorkflow(ctx workflow.Context) error {

	options := workflow.ActivityOptions{
		StartToCloseTimeout: time.Minute * 45,
	}
	ctx = workflow.WithActivityOptions(ctx, options)

	// this Activity is always executed
	err := workflow.ExecuteActivity(ctx, ImportSalesData).Get(ctx, nil)
	if err != nil {
		return err
	}

	if rand.Intn(100) >= 50 {
		workflow.Sleep(ctx, time.Hour * 4)
	}

	workflow.GetLogger(ctx).Info("Preparing to run daily report")
	err = workflow.ExecuteActivity(ctx, RunDailyReport).Get(ctx, nil)
	if err != nil {
		return err
	}

	return nil
}
```

# Determinism trong Temporal nghĩa là gì?

## Cách hiểu đơn giản

* Ở mức cơ bản, deterministic nghĩa là:

  * Cùng input
  * Chạy cùng `Workflow Definition`
  * Thì cho ra cùng output

## Cách hiểu chính xác hơn trong Temporal

* Với Temporal, một `Workflow` được xem là deterministic nếu:

  * Mỗi lần execute cùng một `Workflow Definition`
  * Với cùng input
  * Phải tạo ra cùng các `Commands`
  * Theo đúng cùng một thứ tự

* Điều quan trọng không chỉ là output cuối cùng, mà là **chuỗi Commands được tạo ra trong quá trình chạy Workflow**.

# Commands và Events trong replay

## Worker tạo Commands khi chạy Workflow code

* Khi Worker execute `Workflow Definition`, một số API sẽ tạo `Commands`.

* Ví dụ:

  * Gọi `workflow.ExecuteActivity` tạo command để schedule Activity.
  * Gọi `workflow.Sleep` tạo command để start Timer.

## Cluster ghi Events tương ứng

* Một số `Events` trong `Event History` là kết quả trực tiếp từ `Commands`.

* Ví dụ:

```text
ScheduleActivityTask -> ActivityTaskScheduled
StartTimer -> TimerStarted
```

## Replay dùng Events để kiểm tra Commands

* Khi `Workflow Replay`, Worker chạy lại code từ đầu.

* Worker tạo lại các `Commands` từ code.

* Sau đó Worker so sánh các `Commands` này với `Events` đã có trong `Event History`.

* Nếu Command tạo ra khi replay khớp với Event tương ứng trong history, Worker có thể tiếp tục replay.

* Nếu không khớp, Worker không thể chắc chắn state được khôi phục đúng.

# Ví dụ luồng chạy ban đầu

## Activity đầu tiên luôn được execute

* Trong code, đoạn này luôn chạy:

```go
err := workflow.ExecuteActivity(ctx, ImportSalesData).Get(ctx, nil)
```

* Vì không phụ thuộc điều kiện random, đoạn này deterministic.

* Khi chạy thành công, history sẽ có các Events liên quan đến Activity này, ví dụ:

```text
ActivityTaskScheduled
ActivityTaskStarted
ActivityTaskCompleted
```

## Điều kiện random

* Sau Activity đầu tiên, Workflow chạy đến đoạn:

```go
if rand.Intn(100) >= 50 {
	workflow.Sleep(ctx, time.Hour * 4)
}
```

* Đây là phần gây non-deterministic.

* Giả sử lần chạy ban đầu, `rand.Intn(100)` trả về:

```text
84
```

* Điều kiện `>= 50` là true.

* Workflow đi vào block `if` và gọi:

```go
workflow.Sleep(ctx, time.Hour * 4)
```

## Sleep tạo Timer Command

* `workflow.Sleep` tạo command yêu cầu Temporal start Timer.

* Cluster ghi các Events liên quan đến Timer:

```text
TimerStarted
TimerFired
```

# Khi Worker crash và replay

## Worker mới replay từ Event History

* Giả sử Worker crash sau đoạn Timer.

* Worker khác nhận `Workflow Task` mới và bắt đầu `History Replay`.

* Worker dùng `Event History` để xác định chuỗi Commands kỳ vọng.

* Vì lần chạy trước đã đi qua `workflow.Sleep`, history đang kỳ vọng có command liên quan đến Timer tại vị trí đó.

## Replay Activity đầu tiên

* Khi replay đến:

```go
err := workflow.ExecuteActivity(ctx, ImportSalesData).Get(ctx, nil)
```

* Worker tạo `ScheduleActivityTask Command`.

* Command này khớp với Events trong history.

* Worker không chạy lại Activity, mà reuse kết quả từ history.

# Non-deterministic error xảy ra như thế nào?

## Random cho kết quả khác trong replay

* Khi replay đến đoạn:

```go
if rand.Intn(100) >= 50 {
	workflow.Sleep(ctx, time.Hour * 4)
}
```

* Lần này, giả sử `rand.Intn(100)` trả về:

```text
14
```

* Điều kiện `>= 50` là false.

* Workflow bỏ qua `workflow.Sleep`.

## Chuỗi Commands bị lệch

* Trong lần chạy ban đầu, Workflow đã tạo command cho Timer:

```text
StartTimer
```

* Nhưng trong lần replay, vì điều kiện false nên Worker không tạo `StartTimer`.

* Worker tiếp tục chạy đến Activity tiếp theo:

```go
err = workflow.ExecuteActivity(ctx, RunDailyReport).Get(ctx, nil)
```

* Lúc này Worker tạo command:

```text
ScheduleActivityTask
```

* Nhưng tại vị trí đó trong history, Worker đang kỳ vọng command tương ứng với Timer.

* Command thực tế và command kỳ vọng không khớp.

## Kết quả

* Worker không thể khôi phục state chính xác.

* Temporal báo `non-deterministic error`.

* Lỗi này xảy ra vì cùng một Workflow code, cùng input, nhưng replay tạo ra chuỗi Commands khác với lần chạy trước.

# Vì sao random number nguy hiểm trong Workflow code?

## Random làm thay đổi control flow

* `rand.Intn(100)` có thể trả về giá trị khác nhau ở mỗi lần chạy.

* Nếu giá trị random quyết định có gọi Temporal API hay không, nó sẽ làm thay đổi chuỗi Commands.

* Trong ví dụ:

  * Lần đầu random là `84` → có gọi `workflow.Sleep` → có `StartTimer`.
  * Lần replay random là `14` → bỏ qua `workflow.Sleep` → không có `StartTimer`.

## Điều Temporal cần đảm bảo

* Temporal không yêu cầu mọi dòng code đều tạo ra kết quả giống nhau một cách tuyệt đối.

* Nhưng nếu dòng code đó ảnh hưởng đến việc tạo `Commands`, thì nó phải deterministic.

* Các API như `ExecuteActivity`, `workflow.Sleep`, start Timer, schedule Activity cần xuất hiện theo cùng thứ tự trong mọi lần replay.

# Non-deterministic errors không thể catch trong Workflow code

## Không xử lý bằng try/catch hoặc if err

* Developer không thể catch hoặc handle `non-deterministic error` bên trong `Workflow code`.

* Đây không phải lỗi nghiệp vụ mà Workflow có thể xử lý.

* Đây là lỗi cho thấy Worker không thể replay và restore state chính xác.

## Cách xử lý đúng

* Tránh viết Workflow code gây non-deterministic.

* Không dùng trực tiếp các nguồn non-deterministic trong Workflow nếu chúng ảnh hưởng đến control flow tạo Commands.

* Ví dụ cần cẩn thận với:

  * Random number
  * Current time từ system clock
  * External API call trực tiếp trong Workflow
  * Map iteration không ổn định thứ tự
  * Logic thay đổi code cũ mà không tương thích với history đang chạy