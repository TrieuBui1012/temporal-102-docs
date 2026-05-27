# Deploy code gây Non-Deterministic Error

## Ý chính

* `non-deterministic error` không chỉ xảy ra khi `Workflow code` dùng random number, system time, hoặc external state.

* Việc deploy một thay đổi không tương thích vào `Workflow Definition` cũng có thể gây lỗi với các `Workflow Executions` đang mở.

* Lỗi xảy ra khi các executions cũ được start bằng code cũ, nhưng sau deploy lại phải replay bằng code mới.

# Bối cảnh demo

## Workflow đặt pizza

* Demo dùng lại ví dụ `PizzaWorkflow`.

* Scenario giả định có một developer mới làm việc tại cửa hàng pizza.

* Do giá nhiên liệu cao và thiếu tài xế, cửa hàng quyết định bỏ dịch vụ delivery.

* Vì tất cả khách hàng giờ phải tự đến lấy pizza, không cần kiểm tra địa chỉ khách hàng có nằm trong vùng phục vụ hay không.

## Thay đổi được thực hiện

* Developer xóa phần kiểm tra khoảng cách giao hàng.

* Cụ thể, developer xóa:

  * Lời gọi `GetDistance Activity`
  * Conditional statement kiểm tra địa chỉ có nằm ngoài vùng giao hàng hay không

* Vì xóa đoạn này làm mất một biến được dùng sau đó, code cũng phải chỉnh lại phần liên quan.

* Ngoài ra, developer cũng xóa import chỉ dùng cho đoạn code vừa bị xóa.

# Vì sao thay đổi này nguy hiểm?

## Code cũ có schedule GetDistance Activity

* Trước khi thay đổi, `PizzaWorkflow` có gọi Activity:

```text
GetDistance
```

* Khi Workflow chạy đến đoạn này, Worker tạo `ScheduleActivityTask Command`.

* Cluster ghi Event tương ứng:

```text
ActivityTaskScheduled
```

* Event này nằm trong `Event History`.

## Code mới không còn GetDistance Activity

* Sau khi deploy, version code mới không còn gọi:

```text
GetDistance
```

* Điều này nghĩa là khi replay execution cũ, Worker sẽ không tạo Command tương ứng với `GetDistance`.

* Nhưng `Event History` của execution cũ vẫn đang có Event:

```text
ActivityTaskScheduled
```

* Do đó, Commands tạo ra từ code mới không khớp với Events đã có trong history.

# Quyết định deploy trong giờ hoạt động

## Có executions đang chạy dở

* Demo giả định việc deploy diễn ra trong giờ kinh doanh bình thường của cửa hàng pizza.

* Developer biết có thể đang có các executions của `Workflow Type` này chạy dở.

* Nhưng developer vẫn deploy vì nghĩ rằng Temporal sẽ tự động reconstruct state của `Workflow Executions`.

## Vấn đề nằm ở compatibility

* Đúng là Temporal có thể reconstruct state bằng `History Replay`.

* Nhưng replay chỉ hoạt động nếu code mới vẫn tạo ra cùng chuỗi `Commands` như code cũ đối với các executions đang chạy.

* Nếu code mới đã xóa một Activity từng được schedule trong history, replay không thể khôi phục chính xác.

# Điều gì xảy ra sau khi restart Worker?

## Restart Worker là cần thiết

* Sau khi deploy code mới, cần restart `Worker` để thay đổi có hiệu lực.

* Khi Worker restart, nó sẽ dùng code mới.

## Worker replay các open Workflow Executions

* Worker tự động dùng `History Replay` để reconstruct state của các `Workflow Executions` đang mở tại thời điểm restart.

* Với những execution chưa đi qua phần `GetDistance`, có thể chưa gặp vấn đề ngay.

* Nhưng với những execution đã tới điểm `GetDistance Activity` được schedule, history đã chứa Event liên quan đến Activity này.

## Replay bị mismatch

* Code mới không tạo Command schedule `GetDistance`.

* Nhưng history của execution cũ có Event `ActivityTaskScheduled` cho `GetDistance`.

* Worker không thể map Event trong history về Command tương ứng từ code mới.

* Kết quả là xảy ra `non-deterministic error`.

# Lỗi hiển thị trong Web UI

## Non-deterministic error xuất hiện

* Ngay sau khi restart Worker, trong Web UI xuất hiện `non-deterministic error` ở detail page của một open Workflow.

* Message có nội dung:

```text
lookup failed
```

* Message cũng nhắc đến:

```text
Event ID 5
```

## Event ID 5 là gì?

* Khi scroll trong `Event History`, có thể thấy `Event ID 5` tương ứng với Event:

```text
ActivityTaskScheduled
```

* Event này là của Activity:

```text
GetDistance
```

* Đây chính là Activity đã bị xóa khỏi version code mới.

# Phân tích nguyên nhân

## Event History vẫn ghi nhận GetDistance

* Execution cũ đã từng chạy bằng code cũ.

* Code cũ đã schedule `GetDistance Activity`.

* Vì vậy, history có Event:

```text
ActivityTaskScheduled
```

* Event này là bằng chứng rằng trước đây Worker đã tạo Command schedule Activity đó.

## Code mới không còn tạo Command tương ứng

* Trong replay, Worker chạy code mới.

* Vì code mới đã xóa `GetDistance`, Worker không tạo `ScheduleActivityTask Command` cho Activity này nữa.

* Chuỗi Commands tạo ra trong replay khác với chuỗi Commands từ lần chạy trước.

## Worker không thể reconstruct state

* Vì Command sequence không khớp với `Event History`, Worker không thể xác định cách khôi phục state chính xác.

* Temporal báo `non-deterministic error`.

# Vì sao lỗi này không phụ thuộc số lượng Workers?

## Demo dùng một Worker

* Demo dùng một `Worker` để đơn giản hóa.

## Production thường có nhiều Workers

* Trong production thường có nhiều Workers cùng poll `Task Queue`.

* Tuy nhiên, số lượng Workers không quan trọng trong ví dụ này.

## Lỗi nằm ở code compatibility

* Dù có một hay nhiều Workers, nếu Worker replay execution cũ bằng code mới không tương thích, lỗi vẫn xảy ra.

# Bài học khi deploy Workflow changes

## Không xóa hoặc thay đổi Command-producing code tùy tiện

* Các thay đổi như xóa `Activity`, thêm `Activity`, đổi `Activity Type`, thêm/xóa `Timer`, đổi thứ tự `Activity` và `Timer` đều có thể phá compatibility.

* Nếu còn open executions, những thay đổi này có thể gây `non-deterministic error`.

## Temporal tự replay, nhưng không tự sửa incompatibility

* Temporal có thể tự động replay để reconstruct state.

* Nhưng Temporal cần code mới vẫn tương thích với history cũ.

* Nếu code mới không tạo ra Commands tương ứng với Events cũ, replay sẽ fail.

# Dùng Workflow Reset để khôi phục sau bad deployment

## Workflow Reset là gì?

* `Workflow Reset` là một tính năng của Temporal dùng để xử lý một số trường hợp `Workflow Execution` bị lỗi.

* Trong phần này, `Workflow Reset` được dùng để khắc phục `non-deterministic error` do deploy một thay đổi không tương thích vào `Workflow Definition`.

* Cách hoạt động chính:

  * Terminate `Workflow Execution` hiện tại.
  * Tạo một `Workflow Execution` mới.
  * Execution mới tiếp tục từ một điểm được chỉ định trong `Event History`.

## Khi nào dùng Workflow Reset?

* Dùng khi một `Workflow Execution` đang mở không thể replay bằng code mới do deploy sai.

* Ví dụ trước đó:

  * Code cũ có `GetDistance Activity`.
  * Code mới đã xóa `GetDistance Activity`.
  * Open Workflow Executions cũ có `ActivityTaskScheduled` cho `GetDistance`.
  * Worker dùng code mới replay history cũ và gặp `non-deterministic error`.

# Workflow Reset không phải cách duy nhất

## Có thể cancel hoặc terminate rồi start lại

* Một cách khác là:

  * Cancel hoặc terminate `Workflow Execution` hiện tại.
  * Start một Workflow mới với cùng input.

## Nhược điểm

* Nếu start lại từ đầu, progress đã thực hiện trước đó có thể bị mất.

* Vì vậy, cách này không phải lúc nào cũng phù hợp.

## Lợi ích của Workflow Reset

* `Workflow Reset` cho phép giữ lại một phần history trước điểm reset.

* Những gì xảy ra trước reset point được carry over sang execution mới.

* Workflow có thể chạy lại từ một điểm hợp lý, thay vì luôn bắt đầu hoàn toàn từ đầu.

# Chọn reset point

## Ba lựa chọn reset point

* Khi reset Workflow, có 3 cách chọn điểm reset:

  * `First Workflow Task`
  * `Last Workflow Task`
  * Một `Event ID` cụ thể

## Không phải Event nào cũng reset được

* Nếu chọn reset theo `Event ID`, không phải mọi `Event Type` đều được hỗ trợ.

* Cần hiểu nguyên nhân lỗi để chọn đúng reset point.

# Ví dụ với PizzaWorkflow

## Nguyên nhân lỗi

* Lỗi non-deterministic xảy ra vì code mới đã xóa:

```text
GetDistance
```

* Trong khi execution cũ đã có Event schedule Activity này:

```text
ActivityTaskScheduled
```

## Reset về trước khi GetDistance được schedule

* Để code mới có thể replay thành công, cần reset Workflow về một điểm trước khi `GetDistance Activity` được schedule.

* Trong ví dụ này, `GetDistance Activity` là thứ đầu tiên trong `Workflow Execution` tạo ra `Command`.

* Vì vậy, lựa chọn phù hợp nhất là reset về:

```text
First Workflow Task
```

# Điều gì được giữ lại sau reset?

## Chỉ Events trước reset point được carry over

* Khi reset Workflow, chỉ các `Events` xảy ra trước reset point được đưa sang execution mới.

* Các `Events` sau reset point sẽ không được giữ nguyên như progress đã hoàn tất.

## Sau reset, các bước sau reset point chạy lại

* Mọi thứ sau reset point sẽ được chạy lại.

* Điều này áp dụng cả khi các bước đó đã từng hoàn tất thành công trong execution cũ.

## Ví dụ Timer

* Trong ví dụ, Timer trước đó đã từng fire.

* Nhưng nếu `TimerStarted` xảy ra sau reset point, Timer đó sẽ không được carry over.

* Vì vậy, trong execution mới, Timer sẽ được start lại và chờ fire lại.

# Ghi reason khi reset

## Nên luôn ghi lý do reset

* Khi reset Workflow, có thể và nên specify reason.

* Reason này được ghi vào `Event History` của:

  * Execution gốc
  * Execution mới

## Vì sao reason quan trọng?

* Reason giúp người xem history sau này hiểu tại sao Workflow bị reset.

* Đây là thông tin hữu ích khi debug hoặc audit.

* Ví dụ reason:

```text
Deployed an incompatible change (deleted Activity)
```

# Sau khi reset Workflow

## Execution hiện tại bị terminate

* Khi thực hiện reset, `Workflow Execution` hiện tại sẽ bị terminate.

* Reason được hiển thị trong Event:

```text
WorkflowExecutionTerminated
```

## Web UI tạo link sang execution mới

* Temporal Web UI cung cấp link đến execution mới được tạo bởi reset.

* Execution mới tiếp tục từ reset point đã chọn.

## Event trong execution mới

* Execution mới chứa Event:

```text
WorkflowTaskFailed
```

* Event này tương ứng với reset point.

* Event này cũng chứa reason đã specify khi reset.

## Workflow tiếp tục chạy

* Sau reset point:

  * `Workflow Task` tiếp theo completed.
  * Workflow chạy tiếp qua các bước còn lại.
  * Khi tới Timer, Timer được start lại nếu Timer nằm sau reset point.
  * Sau khi Timer fire, Activity cuối cùng chạy.
  * Workflow hoàn tất thành công.

# Reset Workflow bằng command line

## Lệnh temporal workflow reset

* Có thể reset Workflow từ Web UI hoặc command line.

* Lệnh tương đương với demo trong Web UI:

```bash
temporal workflow reset \
        --workflow-id pizza-workflow-order-Z1238 \
        --event-id 4 \
        --reason "Deployed an incompatible change (deleted Activity)"
```

## Ý nghĩa các option

* `--workflow-id`:

  * Chỉ định `Workflow ID` cần reset.

* `--event-id`:

  * Chỉ định reset point trong `Event History`.

* `--reason`:

  * Ghi lý do reset vào history.
