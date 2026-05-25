# Sticky Execution

## Sticky Execution là gì?

* `Workers` có thể cache state của các `Workflow functions` mà chúng đang execute.

* Để việc cache này hiệu quả hơn, Temporal dùng một cơ chế tối ưu hiệu năng gọi là `Sticky Execution`.

* `Sticky Execution` cố gắng điều hướng các `Workflow Tasks` về cùng một `Worker` đã xử lý chúng trước đó trong cùng một `Workflow Execution`.

## Vì sao cần Sticky Execution?

* Khi cùng một `Worker` tiếp tục xử lý các `Workflow Tasks` của cùng một `Workflow Execution`, `Worker` có thể tận dụng state đã cache.

* Điều này giúp giảm chi phí phải reconstruct lại state từ `Event History`.

* Kết quả là Workflow có thể chạy hiệu quả hơn.

# Cách Sticky Execution hoạt động

## Workflow Task đầu tiên được đưa vào Task Queue gốc

* Khi `Workflow Execution` bắt đầu, `Temporal Service` sẽ schedule một `Workflow Task`.

* Task này được đưa vào `Task Queue` với name mà bạn đã chỉ định.

* Bất kỳ `Worker` nào poll `Task Queue` đó đều có thể nhận Task và bắt đầu execute Workflow.

## Worker nhận Task sẽ bắt đầu poll thêm Sticky Queue

* `Worker` nhận `Workflow Task` đầu tiên vẫn tiếp tục poll `Task Queue` gốc.

* Đồng thời, nó sẽ bắt đầu poll thêm một `Task Queue` khác.

* Queue này được `Temporal Service` chia sẻ riêng với đúng `Worker` đó.

* Queue này có name được tự động generate và được gọi là `Sticky Queue`.

## Các Workflow Tasks tiếp theo được đưa vào Sticky Queue

* Khi `Workflow Execution` tiếp tục chạy, `Temporal Service` sẽ schedule thêm các `Workflow Tasks`.

* Với `Sticky Execution`, các `Workflow Tasks` tiếp theo sẽ được đưa vào `Sticky Queue` riêng của `Worker` trước đó.

* Điều này giúp các Task có xu hướng quay lại cùng một `Worker`, nơi đã có cache state của Workflow.

## Khi Worker không xử lý kịp Sticky Queue

* Nếu `Worker` không start một `Workflow Task` trong `Sticky Queue` ngay sau khi Task được schedule, Temporal sẽ fallback.

* Mặc định, nếu quá khoảng: `5 seconds`

* `Temporal Service` sẽ disable stickiness cho `Workflow Execution` đó.

* Sau đó, `Workflow Task` sẽ được reschedule vào `Task Queue` gốc.

* Khi Task quay lại `Task Queue` gốc, bất kỳ `Worker` nào poll queue đó cũng có thể nhận và tiếp tục `Workflow Execution`.

# Sticky Execution chỉ áp dụng cho Workflow Tasks

## Không áp dụng cho Activity Tasks

* `Sticky Execution` chỉ áp dụng cho `Workflow Tasks`.

* Không áp dụng cho `Activity Tasks`.

## Vì sao không liên quan đến Activity Tasks?

* `Event History` gắn với `Workflow`.

* `Sticky Execution` tối ưu việc execute `Workflow Task` bằng cách tận dụng cached Workflow state.

* `Activity Tasks` không cần cơ chế này theo cùng cách, vì chúng không replay `Event History` để reconstruct state của Workflow.