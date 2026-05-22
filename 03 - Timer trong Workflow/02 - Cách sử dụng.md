# Tạm dừng Workflow Execution trong một khoảng thời gian cụ thể

## Dùng `workflow.Sleep`

* Nếu cần tạm dừng việc execute tiếp Workflow trong một khoảng thời gian cụ thể, dùng function `workflow.Sleep`.

```go
// This will pause Workflow Execution for 10 seconds
// The first parameter (ctx) is the context passed to the Workflow
err := workflow.Sleep(ctx, time.Second*10)
```

## `workflow.Sleep` là gì?

* `workflow.Sleep` là phiên bản an toàn trong Workflow để thay thế cho `time.Sleep` của Go.

* Function này nhận 2 input parameters:

  * `ctx`: context được truyền vào Workflow function.
  * Timer duration: khoảng thời gian Timer sẽ chờ.

## Cách hoạt động

* Khi gọi `workflow.Sleep`, Workflow code sẽ block.

* Code chỉ tiếp tục chạy khi:

  * Timer fire.
  * Hoặc Timer bị canceled.

# Chạy code tại một thời điểm cụ thể trong tương lai

## Dùng `workflow.NewTimer`

* Nếu cần chạy code tại một thời điểm cụ thể trong tương lai, dùng function `workflow.NewTimer`.

```go
// workflow.NewTimer is a Workflow-safe counterpart to time.NewTimer
timerFuture := workflow.NewTimer(ctx, time.Second * 5)
logger.Info("The timer was set")

// Unlike workflow.Sleep, waiting for the timer is a separate operation
timerFuture.Get(ctx, nil)
logger.Info("The timer has fired")
```

## `workflow.NewTimer` là gì?

* `workflow.NewTimer` là phiên bản an toàn trong Workflow để thay thế cho `time.NewTimer` của Go.

* Function này nhận cùng 2 input parameters như `workflow.Sleep`:

  * `ctx`: context được truyền vào Workflow function.
  * Timer duration: khoảng thời gian Timer sẽ chờ.

## Khác biệt với `workflow.Sleep`

* `workflow.Sleep` sẽ block ngay sau khi được gọi.

* `workflow.NewTimer` thì không block sau khi set Timer.

* Thay vào đó, `workflow.NewTimer` trả về một `Future`.

## Cách hoạt động

* `Future` được trả về bởi `workflow.NewTimer` sẽ trở nên ready khi:

  * Timer fire.
  * Hoặc Timer bị canceled.

* Việc chờ Timer là một thao tác riêng.

* Khi gọi `timerFuture.Get(ctx, nil)`, code sẽ block cho đến khi `Future` ready.