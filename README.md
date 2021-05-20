# Event Loop là gì và hoạt động thế nào?
## Một số khái niệm cơ bản
- Stack: là một vùng nhớ đặc biệt trên con chip máy tính phục vụ cho quá trình thực thi các dòng lệnh mà cụ thể là các hàm. Hàm chẳng qua là một nhóm các lệnh và chương trình thì gồm một nhóm các hàm phối hợp với nhau. Mỗi khi một hàm được triệu gọi thì nó sẽ được đẩy vào một hàng đợi đặc biệt có tên là stack. Stack là một hàng đợi kiểu LIFO (Last In First Out) nghĩa là vào đầu tiên thì ra sau cùng. Một hàm chỉ được lấy ra khỏi stack khi nó hoàn thành và return.

```js
function Bar() {
}

function Foo() {
    Bar();
}

Foo();
```

```
        Stack    
 -------------------- 
|                    |
 -------------------- 
|      Bar           | <--
 -------------------- 
|      Foo           |
 --------------------
 ```

 - Heap: Heap là vùng nhớ được dùng để chưa kết quả tạm phục vụ cho việc thực thi các hàm trong stack. Heap càng lớn thì khả năng tính toán càng cao.

 ## Event Loop là gì?
Event Loop là thứ cho phép node.js thực hiện các tác vụ I/O không đồng bộ, mặc dù trên thực tế Javascript là singlethread bằng cách giảm tải các hoạt động cho nhân hệ điều hành bất cứ khi nào có thể.

 ## Event Loop hoạt động thế nào?
Khi node.js khởi động, nó khởi tạo Event Loop, xử lý tập lệnh đầu vào được cung cấp (hoặc REPL) có thể bao gồm việc thực hiện các hàm không đồng bộ, schedule timers hoặc process.nextTick(), sau đó bắt đầu xử lý Event Loop.

![](https://i.imgur.com/eUpfvqm.png)

Mỗi pha có một hàng đợi FIFO chứa các hàm callbacks. Mỗi giai đoạn đều có một nhiệm vụ riêng, nhưng nói chung khi Event Loop bước vào một giai đoạn nhất định, nó sẽ xử lý bất kỳ dữ liệu nào cho giai đoạn đó, sau đó thực hiện các hàm callbacks trong hàng đợi của pha đó cho đến khi hết hoặc đạt đến giới hạn thực thi. Tiếp đến Event Loop sẽ chuyển sang các giai đoạn tiếp theo.

Vì mỗi pha có thể có một số lượng lớn các hàm callbacks chờ được xử lý thế nên một số callback của các hàm timers (bộ đếm thời gian) có thể sẽ có thời gian chờ thực hiện lâu hơn là so với ngưỡng ban đầu đặt ra, ngưỡng thời gian ban đầu chỉ đảm bảo thời gian chờ ngắn nhất chứ không phải là thời gian chờ chính xác.

Ví dụ

```js
setTimeout(() => console.log('hello world'), 1000);
```

Thì 1000ms là thời gian chờ ngắn nhất, chứ không phải là sau đúng 1000ms lệnh console.log sẽ được thực hiện.

Tổng quan về các pha (phase) của Event Loop
- Timers: thực thi các hàm callbacks đã được lên lịch với setTimeout() và setInterval().
- Pending callbacks: thực hiện các I/O callbacks được hoãn lại cho lần lặp tiếp theo.
- Idle, prepare: dùng cho việc xử lý nội bộ của node.js.
- Poll: truy xuất các sự kiện I/O mới, thực hiện các hàm callbacks liên quan đến I/O (hầu như tất cả ngoại trừ close callback, timers callback và setImmediate()).
- Check: xử lý hàm callback của setImmediate.
- Close callbacks: thực thi các hàm callbacks cho các sự kiện close. Ví dụ: socket.on("close").

Giữa mỗi lần lặp của Event Loop, node.js sẽ kiểm tra xem nó có đang đợi bất kỳ I/O không đồng bộ hoặc timers nào không và thoát nếu không còn gì.

## Chi tiết các pha (phase) của Event Loop
## Timers
Một timers (bộ đếm thời gian) chỉ định ngưỡng mà sau đó một hàm callback có thể được thực hiện. Hàm callback của timers sẽ chạy sớm nhất có thể sau khi lượng thời gian được chỉ định trôi qua. Tuy nhiên, chúng cũng có thể bị delay trong một khoảng thời gian nào đó.

Lưu ý: Về mặt kỹ thuật, poll kiểm soát khi timers được thực thi.

Ví dụ: Giả sử chúng ta thiết lập một hàm setTimeout() được thực thi sau 100ms, sau đó chạy một hàm someAsyncOperation thực hiện việc đọc một file không đồng bộ mất 95ms:

```js
const fs = require('fs');

function someAsyncOperation(callback) {
  // giả sử đọc file mất 95ms
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms`);
}, 100);

// hàm someAsyncOperation mất 95ms để hoàn thành
someAsyncOperation(() => {
  const startCallback = Date.now();

  // vòng lặp sẽ làm delay 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

Khi Event Loop bước vào giai đoạn poll, nó có một hàng đợi trống (fs.readFile() chưa hoàn thành), vì vậy nó sẽ đợi số ms còn lại cho đến khi đạt đến ngưỡng của bộ định thời sớm nhất. Trong khi chờ 95 ms vượt qua, fs.readFile() đọc xong và hàm callback của nó mất 10ms để hoàn thành sẽ được thêm vào hàng đợi của poll và được thực thi. Khi hàm callback thực thi xong, không còn callback nào trong hàng đợi, do đó Event Loop sẽ thấy rằng ngưỡng của bộ định thời sớm nhất đã đạt đến sau đó kết thúc lại giai đoạn bộ định thời để thực hiện lệnh gọi lại của bộ định thời. Trong ví dụ này, bạn sẽ thấy rằng tổng thời gian trễ giữa bộ đếm thời gian được lập lịch và cuộc gọi lại của nó được thực thi sẽ là 105ms.

## Pending callbacks
Giai đoạn này thực hiện các hàm callback đối với một số hoạt động của hệ thống, chẳng hạn như các loại lỗi TCP. Ví dụ: nếu socket TCP nhận được ECONNREFUSED khi cố gắng kết nối, một số hệ thống *nix muốn đợi để báo lỗi. Nó sẽ được đưa vào hàng đợi này để chờ được thực thi.

## Poll
Poll có hai chức năng chính:

- Tính toán thời gian nó sẽ chặn và thăm dò các sự kiện I/O, sau đó:
- Xử lý các sự kiện trong hàng đợi poll

Khi Event Loop bước vào giai đoạn poll và không có các callback của timers nào, một trong hai trường hợp sau sẽ xảy ra:

- Nếu hàng đợi poll không trống, Event Loop sẽ lặp lại qua các hàm callback của nó và thực hiện lần lượt chúng cho đến khi hàng đợi hết hoặc đạt đến giới hạn của hệ thống.
Nếu hàng đợi poll trống, một trong hai trường hợp nữa sẽ xảy ra:
- Nếu các tập lệnh đã được lên lịch trước bởi setImmediate(), Event Loop sẽ kết thúc giai đoạn poll và tiếp tục đến giai đoạn check để thực thi các tập lệnh đã được lên lịch đó.
    - Nếu các tập lệnh chưa được lên lịch trước bởi setImmediate(), Event Loop sẽ đợi các hàm callbacks được thêm vào hàng đợi, sau đó thực thi chúng ngay lập tức.
    - Khi hàng đợi poll trống, Event Loop sẽ kiểm tra xem có bộ đếm thời gian nào đạt đến ngưỡng được thực thi. Nếu một hoặc nhiều cái đã sẵn sàng, Event Loop sẽ quay trở lại giai đoạn timers để thực hiện các hàm callbacks đó.

## Check
Giai đoạn này cho phép chúng ta thực hiện các hàm callbacks ngay sau khi giai đoạn poll hoàn thành. Nếu giai đoạn poll đang có hàng đợi trống và có các tập lệnh đã được lên lịch trước bởi setImmediate(), Event Loop có thể tiếp tục đến giai đoạn này thay vì phải đợi.

setImmediate() là một bộ đếm thời gian đặc biệt chạy trong một giai đoạn riêng biệt của Event Loop. Nó sử dụng một API libuv để lập lịch các hàm callbacks thực thi sau khi giai đoạn poll hoàn thành.

Nói chung, khi các đoạn mã được thực thi, Event Loop cuối cùng sẽ đến giai đoạn poll - nơi nó sẽ đợi các kết nối đến, request, v.v… Tuy nhiên, nếu một hàm callback đã được lên lịch bởi setImmediate() và giai đoạn poll vào trạng thái nhàn rỗi, nó sẽ kết thúc và tiếp tục đến giai đoạn check hơn là chờ đợi các sự kiện của poll.

## Close callback
Nếu một socket hoặc handle bị đóng đột ngột (ví dụ socket.destroy()), sự kiện 'close' sẽ được phát ra trong giai đoạn này. Nếu không, nó sẽ được phát ra thông qua process.nextTick().


# Non-blocking I/O
![](https://i.imgur.com/kF77pbd.png)
- callback để xử lý kết quả của các tác vụ I/O không đồng bộ để không chặn luồng chính. Bất kì hoạt động nào gây ra độ trễ sẽ được đi cùng với một hàm callback để thực hiện khi nó hoàn thành tại thời điểm nào đó. Các hoạt động này được thực thi song song bằng cách sử dụng Thread Pool và các hàm callback bản chất vẫn được thực thi ở trên luồng chính.
- Node.js Standard Library được thực thi sẽ chạy ở background (không phải trong Call Stack) bằng cách sử dụng nhóm luồng (Thread Pool) trong thư viện libuv. Ví dụ: fs là một hàm sẽ được thực thi ở background và khi hoàn thành, nó sẽ thêm hàm callback vào Event Queue.
- Event Queue chứa các hàm callback đang được chờ đợi cho đến khi Event Loop đưa chúng trở lại Call Stack và thực thi.
- Event Loop sẽ di chuyển các hàm gọi lại từ Event Queue sang Call Stack để được thực thi bởi luồng chính. Khi Call Stack trống và Event Queue đang có các hàm chờ xử lý, Event Loop sẽ di chuyển chúng sang Call Stack và chúng sẽ được thực thi bởi luồng chính.

Do đó, bất kỳ hoạt động I/O nào tốn thời gian sẽ không chặn luồng chính mà được chuyển tới Standard Library để được thực thi ở background. Do đó luồng chính vẫn tiếp tục được thực thi tiếp, đó gọi là bản chất không chặn (non-blocking). Khi hoạt động I/O hoàn tất, hàm callback của nó sẽ được thực thi bởi luồng chính bằng cách sử dụng Event Loop và Event Queue như đã nói ở trên.

## I/O trong Standard Library
Các hoạt động I/O dưới đây được coi là tốn thời gian và sẽ được đưa vào thực hiện ở một nhóm luồng gọi là Thread Pool:

### File Systems
Các hoạt động được xử lý bởi module fs như đọc hoặc ghi vào tệp. Ví dụ fs.write, fs.readStream

### Network Calls
Các cuộc gọi yêu cầu mạng như dns.resolve, dns.lookup, http.get, https.post, socket.connect…

### Timers
Các hoạt động liên quan đến setTimeout, setImmediate hoặc setInterval. Ngay cả hàm setTimeout(cb, 0) với 0ms độ trễ vẫn sẽ được đẩy đến Event Queue và được thực thi theo thứ tự. Do đó, setTimeout không thể đảm bảo thời gian chính xác mà là xác định thời gian tối thiểu hàm callback có thể được thực hiện.

Cần lưu ý rằng không phải tất cả tác vụ tốn thời gian đều là tác vụ I/O. Ví dụ: tác vụ chuyên sâu của CPU trong một vòng lặp không phải là tác vụ I/O và nó sẽ chặn luồng chính. Do đó node.js được coi là không phù hợp với các tác vụ chuyên sâu về CPU mà là phù hợp hơn với các tác vụ chuyên sâu về I/O.

```js
for (var i = 0; i < 10000; i++) {
   crypto.createHash(); // CPU intensive task
   // or
   sleep(2); // CPU intensive task
}
console.log('After CPU intensive task');
```
Ở đây, tác vụ chuyên sâu CPU sẽ chặn luồng chính và không thực thi lệnh console ngay lập tức như khi với các hoạt động I/O không đồng bộ. Đơn giản là do tác vụ chuyên sâu CPU sẽ ràng buộc CPU phải được thực thi ngay lập tức, còn đối với tác vụ ràng buộc I/O, nó được chuyển tới libuv để được xử lý không đồng bộ.

Cần lưu ý rằng nếu Call Stack đang không trống do các tác vụ chuyên sâu CPU này chặn luồng chính, nó sẽ không thể thực thi bất kỳ thứ gì từ Event Loop của bất kỳ tác vụ I/O nào đang chờ xử lý và sẽ dẫn đến tình trạng thắt cổ chai.
