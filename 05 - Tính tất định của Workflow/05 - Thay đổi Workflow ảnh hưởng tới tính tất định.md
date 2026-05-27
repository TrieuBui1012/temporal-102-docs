# Workflow Changes có thể gây non-deterministic errors như thế nào

## Ý chính

* Không chỉ code dùng random number, system time, external state mới gây `non-deterministic errors`.

* Việc deploy thay đổi cho `Workflow Definition` cũng có thể gây lỗi non-deterministic.

* Lỗi xảy ra khi một `Workflow Execution` đang chạy dở không thể replay bằng version code mới.

# Nhắc lại mối quan hệ giữa Workflow code, Commands và Events

## Worker tạo Commands

* Khi Worker chạy `Workflow code`, một số API sẽ tạo `Commands`.

* Ví dụ:

```text
ExecuteActivity -> ScheduleActivityTask Command
workflow.Sleep -> StartTimer Command
```

## Cluster ghi Events

* `Temporal Cluster` nhận `Commands` và ghi các `Events` tương ứng vào `Event History`.

* Ví dụ:

```text
ScheduleActivityTask -> ActivityTaskScheduled
StartTimer -> TimerStarted
```

## Replay cần Commands khớp với Event History

* Khi `History Replay`, Worker chạy lại `Workflow code`.

* Worker dùng `Event History` để kiểm tra xem các `Commands` được tạo ra có khớp với history hay không.

* Nếu code mới tạo ra chuỗi `Commands` khác với code cũ, Worker không thể reconstruct state chính xác.

# Vì sao thay đổi Workflow code có thể gây lỗi?

## Code mới có thể không tương thích với executions cũ

* Một `Workflow Execution` có thể đã bắt đầu bằng version code cũ.

* Trong lúc execution vẫn đang chạy, bạn deploy version code mới.

* Khi Worker restart và replay execution cũ bằng code mới, code mới phải tạo ra cùng chuỗi `Commands` như code cũ.

* Nếu không, sẽ xảy ra `non-deterministic error`.

## Không cần dùng random vẫn có thể lỗi

* Ngay cả khi code mới không dùng:

  * Random number
  * System time
  * External state

* Nó vẫn có thể gây non-determinism nếu thay đổi chuỗi `Commands`.

# Đánh giá một code change có ảnh hưởng determinism hay không

## Câu hỏi cần tự hỏi

* Với cùng input, version code mới có tạo ra chính xác cùng các `Commands`, theo cùng thứ tự, như version code cũ không?

* Nếu câu trả lời là “có thể không”, thì change đó **có thể** gây `non-deterministic error`.

## “May lead to” không có nghĩa là chắc chắn lỗi

* Một thay đổi có thể gây lỗi, nhưng không phải lúc nào cũng gây lỗi ngay.

* Nó chỉ thực sự thành vấn đề nếu khi deploy có `Workflow Executions` thuộc `Workflow Type` đó đang chạy dở.

* Nếu không có execution cũ nào cần replay bằng code mới, lỗi có thể không xuất hiện.

# Vai trò của Worker restart

## Worker cache code đang chạy

* `Workers` cache code mà chúng đang chạy.

* Sau khi deploy code mới, cần restart Workers để chúng dùng version mới.

## Sau restart, Workers replay các executions đang chạy

* Khi Workers đã restart, các `Workflow Executions` đang in progress sẽ được replay bằng code mới.

* Nếu execution cũ replay bằng code mới tạo ra chuỗi `Commands` khác với version cũ, sẽ có `non-deterministic error`.

## Long-running Workflows dễ gặp lỗi hơn

* Lỗi này thường dễ gặp với `long-running Workflows`.

* Lý do là tại thời điểm deploy, có khả năng cao vẫn còn một hoặc nhiều executions đang chạy dở.

# Compatibility của Workflow changes

## Không thể liệt kê mọi trường hợp

* Không thực tế để liệt kê đầy đủ mọi thay đổi có thể gây `non-deterministic error`.

* Cách hiểu quan trọng là nhìn vào việc thay đổi đó có ảnh hưởng đến chuỗi `Commands` hay không.

# Những thay đổi có thể gây non-deterministic errors

## Thêm hoặc xóa Activity

* Nếu thêm hoặc xóa một lời gọi `Activity`, chuỗi `Commands` sẽ thay đổi.

* Ví dụ:

```text
Code cũ: Activity A -> Activity B
Code mới: Activity A -> Activity C -> Activity B
```

* Code mới có thêm một `ScheduleActivityTask Command`, nên replay execution cũ có thể bị lệch history.

## Đổi Activity Type trong ExecuteActivity

* Nếu vẫn gọi `ExecuteActivity`, nhưng đổi `Activity Type`, Command có thể không còn khớp với Event cũ.

* Ví dụ:

```text
Code cũ: ExecuteActivity(ctx, SendEmail, ...)
Code mới: ExecuteActivity(ctx, SendInvoiceEmail, ...)
```

* Dù cùng là schedule Activity, chi tiết Command đã khác.

## Thêm hoặc xóa Timer

* Thêm hoặc xóa `workflow.Sleep` hoặc Timer sẽ thay đổi chuỗi `Commands`.

* Ví dụ:

```text
Code cũ: Activity A -> Activity B
Code mới: Activity A -> Timer -> Activity B
```

* Code mới có thêm `StartTimer Command`.

## Đổi thứ tự Activities hoặc Timers

* Nếu thay đổi thứ tự execute giữa Activities và Timers, replay cũng có thể lỗi.

* Ví dụ:

```text
Code cũ: Activity A -> Timer -> Activity B
Code mới: Timer -> Activity A -> Activity B
```

* Các Commands vẫn có thể giống về số lượng, nhưng thứ tự khác.

* Temporal yêu cầu Commands phải giống **cả nội dung lẫn thứ tự**.

# Những thay đổi không gây non-deterministic errors

## Thay đổi statement không tạo Commands

* Có thể sửa các statement trong `Workflow Definition` nếu chúng không ảnh hưởng đến Commands.

* Ví dụ:

  * Logging statements
  * Tính toán local không làm thay đổi việc gọi Activity/Timer
  * Refactor code nhưng vẫn tạo cùng Commands theo cùng thứ tự

## Thay đổi ActivityOptions hoặc RetryPolicy

* Thay đổi attributes trong:

```text
ActivityOptions
RetryPolicy
```

* Theo nội dung bài học, các thay đổi này không dẫn đến non-deterministic errors.

* Tuy nhiên, chúng có thể ảnh hưởng behavior của executions mới hoặc các command mới được tạo sau điểm replay.

## Sửa code bên trong Activity Definition

* Có thể sửa logic bên trong `Activity Definition`.

* Lý do:

  * `Activity` không bị replay giống Workflow.
  * Nếu Activity đã hoàn tất, replay sẽ dùng result từ `ActivityTaskCompleted`.
  * Nếu Activity chưa chạy, execution mới sẽ dùng code Activity mới.

# Thay đổi Timer duration

## Có thể đổi duration của Timer trong nhiều trường hợp

* Có thể thay đổi duration của một Timer mà không ảnh hưởng compatibility.

* Điều kiện: không đổi duration **từ zero sang non-zero** hoặc **từ non-zero sang zero**.

## Lưu ý theo SDK

* Hành vi này có thể khác nhau giữa các SDK.

* Ví dụ:

  * Trong `TypeScript SDK`, đổi duration sang hoặc từ zero không ảnh hưởng compatibility.

# Cách suy nghĩ khi deploy thay đổi Workflow

## Luôn kiểm tra từ góc nhìn replay

* Đừng chỉ hỏi: “Code này có chạy đúng không?”

* Hãy hỏi thêm:

  * “Nếu một execution cũ đang chạy dở replay bằng code mới, nó có tạo cùng Commands như trước không?”
  * “Có thêm, bớt, đổi loại, hoặc đổi thứ tự Activity/Timer không?”
  * “Có nhánh điều kiện nào khiến command sequence khác version cũ không?”

## workflowcheck không bắt được lỗi kiểu này

* `workflowcheck` dùng static analysis.

* Tool này có thể phát hiện một số nguồn non-determinism trong code.

* Nhưng nó không phân tích runtime behavior giữa version cũ và version mới.

* Vì vậy, nó không thể phát hiện các lỗi do breaking backwards compatibility với executions đang chạy.