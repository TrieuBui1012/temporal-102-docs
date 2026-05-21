# Phát triển input parameters và return values theo hướng backward-compatible

* Một cách để cải thiện Temporal application code là dùng `struct`.

* `struct` nên được dùng để đại diện cho:

  * Input parameters truyền vào Workflow hoặc Activity.
  * Results được return từ Workflow hoặc Activity.

---

## Vì sao nên dùng struct?

* Với Go SDK, Workflow Definition bắt buộc nhận `context` object làm input parameter đầu tiên.

* Sau `context`, Workflow có thể nhận thêm nhiều parameters khác để cung cấp input data cho function.

* Activity Definition không bắt buộc nhận `context` object làm input parameter đầu tiên.

* Tuy nhiên, điều này được khuyến nghị mạnh mẽ.

* Lý do:

  * Activity có thể nhận thông báo cancelation.
  * Hỗ trợ nhiều tính năng nâng cao khác.

* Giống Workflows, Activities cũng có thể nhận nhiều input parameters.

* Nhưng trong cả Workflow và Activity, việc thay đổi các yếu tố sau có thể ảnh hưởng backward compatibility với existing executions:

  * Số lượng parameters.
  * Vị trí parameters.
  * Type của parameters.

* Vì vậy, best practice là đóng gói toàn bộ input parameters vào một `struct` duy nhất.

* Sau đó truyền `struct` đó làm input cho Workflow hoặc Activity.

* Không nên truyền nhiều field riêng lẻ vào function.

* Lợi ích:

  * Có thể thay đổi cấu trúc bên trong `struct`.
  * Không làm thay đổi function signature.
  * Giữ backward compatibility tốt hơn với existing executions.

* Cách này cũng được khuyến nghị cho values được return từ Workflow và Activity Definitions.

---

## Ví dụ: dùng struct làm Activity Input

* Để hiểu vì sao đây là best practice, xem scenario “Hello World” trong Temporal 101.

* Scenario này có một Activity gọi microservice để lấy greeting bằng tiếng Tây Ban Nha.

* Function nhận input là một `string` chứa tên người.

* Function trả về output là một `string` chứa greeting đã được customize bằng tiếng Tây Ban Nha.

```go
// This Activity returns a customized greeting in Spanish, using the provided name 
func GreetInSpanish(ctx context.Context, name string) (string, error) {
   // implementation omitted for brevity
```

* Cách này vẫn hoạt động.

* Nhưng đây không phải cách tốt nhất nếu code sẽ được deploy lên production và maintain lâu dài.

* Ví dụ sau này có requirement mới:

  * Hỗ trợ greeting bằng các ngôn ngữ khác.
  * Ví dụ: German hoặc Zulu.

* Nếu thêm một input parameter mới vào Activity:

  * Function signature sẽ thay đổi.
  * Compatibility với ongoing executions của Activity sẽ bị ảnh hưởng.

* Nếu ngay từ đầu dùng một `struct` duy nhất làm input:

  * Có thể thêm field language code vào `struct`.
  * Không ảnh hưởng function signature.

---

## Code minh họa dùng struct

* Code sau minh họa cách hỗ trợ requirement mới bằng `struct`.

```go
// Define a struct to encapsulate all data passed as input for this Activity
type GreetingInput struct {
        Name         string
        LanguageCode string
}

// Define a struct to encapsulate the data returned by this Activity
type GreetingOutput struct {
        Greeting string
}

// Specify these types for the input parameter and return value of the Activity
func GetTranslatedGreeting(ctx context.Context, input GreetingInput) (GreetingOutput, error) {

        // An example to show how to access input parameters and create the return value
        if input.LanguageCode == "fr" {
                bonjour := fmt.Sprintf("Bonjour, %s", input.Name)
                return GreetingOutput{ Greeting: bonjour, }, nil
        }
        // support for additional languages would follow...
```

* Code này minh họa thiết kế tốt hơn cho Activity.

* Đầu tiên, định nghĩa `struct` đại diện cho input của Activity.

* `GreetingInput` gồm:

  * `Name`
  * `LanguageCode`

* Sau đó định nghĩa `struct` đại diện cho output của Activity.

* `GreetingOutput` hiện chỉ chứa:

  * `Greeting`

* Sau này nếu requirements thay đổi:

  * Có thể update `GreetingOutput`.
  * Không nhất thiết phải đổi function signature.

* Cuối cùng, Activity function được update để dùng các `struct` này thay vì các `string` trước đó.

---

## Lưu ý khi chuyển từ string sang struct

* Việc chuyển ban đầu từ `string` sang `struct` không phải là backward-compatible change.

* Tuy nhiên, nếu làm thay đổi này càng sớm càng tốt:

  * Code sẽ dễ xử lý các thay đổi tương lai của input data hơn.
  * Code cũng dễ xử lý các thay đổi tương lai của output data hơn.

---

## Tùy chọn: dùng structs cho data

* Nếu muốn tìm hiểu sâu hơn về cách dùng structs cho input và output data:

  * Exercise environment có một working example để tự học.

* Ví dụ nằm trong directory:

```text
samples/using-structs
```

* Code này minh họa cách dùng structs cho cả:

  * Input parameters.
  * Return values.

* Áp dụng trong cả:

  * Workflow Definition.
  * Activity Definition.
