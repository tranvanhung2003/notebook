So this is the Hyperconverged platform that we have the preview one before the Newton installation you​‌Đây là nền tảng siêu hội tụ mà chúng ta có bản xem trước trước khi cài đặt Newton.

see right here that when we installed the application in the virtualization software that exists in​‌Bạn thấy đấy, khi chúng ta cài đặt ứng dụng vào phần mềm ảo hóa hiện có trong...

different server, you need to connect every server by two different switch to separated switches mean​‌Mỗi máy chủ khác nhau, bạn cần kết nối từng máy chủ bằng hai bộ chuyển mạch khác nhau.

for the failure of any switch you should have connection with the stick on the switch.​‌Nếu bất kỳ công tắc nào bị lỗi, bạn cần kết nối cần điều khiển với công tắc đó.

You need to have at least two network adapter for every physical server.​‌Bạn cần có ít nhất hai card mạng cho mỗi máy chủ vật lý.

You need to have two cable for every server.​‌Bạn cần hai dây cáp cho mỗi máy chủ.

So you see right here how much hardware you have.​‌Như bạn thấy đấy, ở đây bạn có bao nhiêu phần cứng.

Then you need to have multiple connection from the different to switch to do different storage.​‌Sau đó, bạn cần có nhiều kết nối từ các thiết bị khác nhau đến bộ chuyển mạch để thực hiện các thao tác lưu trữ khác nhau.

We have storage controller for every external storage you see right here.​‌Chúng tôi có bộ điều khiển lưu trữ cho mọi thiết bị lưu trữ ngoài mà bạn thấy ở đây.

This is a separate storage.​‌Đây là kho lưu trữ riêng biệt.

This is another storage, every storage hub, the storage controller, etc. So this is the preview datacenter​‌Đây là một hệ thống lưu trữ khác, bao gồm tất cả các trung tâm lưu trữ, bộ điều khiển lưu trữ, v.v. Vì vậy, đây là trung tâm dữ liệu xem trước.

environment and how it's look like very complicated and need more hardware.​‌Môi trường và hình ảnh của nó trông rất phức tạp và cần nhiều phần cứng hơn.

So when we use the Nutanix solution in this example, if we buy a Nutanix physical server, you'll see​‌Vì vậy, khi chúng ta sử dụng giải pháp Nutanix trong ví dụ này, nếu chúng ta mua một máy chủ vật lý Nutanix, bạn sẽ thấy

that all this environment will be minimized it inside one single server or one block.​‌Toàn bộ môi trường này sẽ được thu nhỏ lại bên trong một máy chủ duy nhất hoặc một khối duy nhất.

We're going to see an example of a real Nutanix block and what the component of it.​‌Chúng ta sẽ xem một ví dụ về một khối Nutanix thực tế và các thành phần của nó.

But just to get to idea.​‌Nhưng chỉ để nêu ý tưởng thôi.

So instead of having this infrastructure, the server, the switch, the cable, the storage, you will​‌Vì vậy, thay vì phải có cơ sở hạ tầng này - máy chủ, bộ chuyển mạch, cáp, thiết bị lưu trữ - bạn sẽ...

use them all together inside one block that have its own hard disk and have the network inside it as​‌Sử dụng tất cả chúng cùng nhau trong một khối duy nhất có ổ cứng riêng và mạng được tích hợp bên trong.

a virtual network and the controller inside it has our KVM, the virtual machine controller.​‌Một mạng ảo và bộ điều khiển bên trong nó là KVM của chúng ta, bộ điều khiển máy ảo.

So right here, if we scal out computing and storage, if you add, for example, a new physical host,​‌Vậy ngay tại đây, nếu chúng ta mở rộng khả năng tính toán và lưu trữ, nếu bạn thêm vào, ví dụ, một máy chủ vật lý mới,

you will have more computing, more CPU, more memory and more storage by just adding a new physical​‌Chỉ cần thêm một ổ cứng vật lý mới, bạn sẽ có khả năng tính toán mạnh hơn, CPU mạnh hơn, bộ nhớ mạnh hơn và dung lượng lưu trữ lớn hơn.

server or a new block.​‌máy chủ hoặc một khối mới.

But in this environment, if you add server, you don't get benefits in the storage or in the network.​‌Nhưng trong môi trường này, nếu bạn thêm máy chủ, bạn sẽ không nhận được lợi ích nào về dung lượng lưu trữ hoặc mạng.

If you add storage, you don't get benefits in the computing like the CPU and the memory.​‌Nếu bạn tăng dung lượng lưu trữ, bạn sẽ không nhận được những lợi ích về hiệu năng như tăng CPU và bộ nhớ.

If you add in your switch, you don't have an upgraded in the storage, you have lots of upgrade right​‌Nếu bạn thêm bộ chuyển mạch của mình, bạn sẽ không có bản nâng cấp nào trong bộ nhớ, bạn sẽ có rất nhiều bản nâng cấp phải không?

here and very difficult to scale out in this environment.​‌Ở đây và việc mở rộng quy mô trong môi trường này rất khó khăn.

But in this one it's easy to scale out your environment by just adding a new physical host.​‌Nhưng với giải pháp này, bạn có thể dễ dàng mở rộng môi trường của mình chỉ bằng cách thêm một máy chủ vật lý mới.

Then we have the virtualization inside this different hosts, you have the virtualization software and​‌Tiếp theo, chúng ta có ảo hóa bên trong các máy chủ khác nhau này, bạn có phần mềm ảo hóa và

the application.​‌Ứng dụng.

Of course, you can use multiple type of virtualization, but when you buy a Nutanix server from the​‌Tất nhiên, bạn có thể sử dụng nhiều loại ảo hóa khác nhau, nhưng khi bạn mua máy chủ Nutanix từ...

Nutanix company, you have the virtualization software inside it.​‌Công ty Nutanix, phần mềm ảo hóa được tích hợp sẵn trong đó.

That's come out with the hardware.​‌Điều đó đã được thể hiện rõ qua phần cứng.

You don't need to buy our virtualization software.​‌Bạn không cần phải mua phần mềm ảo hóa của chúng tôi.

It's come out with the Nutanix physical server.​‌Nó đã được ra mắt cùng với máy chủ vật lý Nutanix.

Of course, if you want to use any other virtualization application, you can use it without any problem.​‌Dĩ nhiên, nếu bạn muốn sử dụng bất kỳ ứng dụng ảo hóa nào khác, bạn hoàn toàn có thể sử dụng mà không gặp vấn đề gì.

But if you buy a Nutanix hardware, you already have the virtualization inside it.​‌Nhưng nếu bạn mua phần cứng Nutanix, bạn đã có sẵn công nghệ ảo hóa bên trong đó rồi.

So you see right here, virtualization and administration included that when you have the virtualization​‌Như bạn thấy ở đây, ảo hóa và quản trị được bao gồm khi bạn có ảo hóa.

software and the administration software included with the Nutanix hardware.​‌phần mềm và phần mềm quản trị đi kèm với phần cứng Nutanix.

But if you don't use the hardware, you can use your own virtualization or buy the virtualization from​‌Nhưng nếu bạn không sử dụng phần cứng, bạn có thể sử dụng phần mềm ảo hóa của riêng mình hoặc mua phần mềm ảo hóa từ [tên nhà cung cấp].

the Nutanix.​‌Nutanix.

Then the new benefit of using the Nutanix hardware is that you are no more need to manage your virtualization.​‌Lợi ích mới của việc sử dụng phần cứng Nutanix là bạn không còn cần phải quản lý ảo hóa nữa.

The virtualization come inside the server and already preconfigured and ready to use without needing​‌Công nghệ ảo hóa được tích hợp sẵn trong máy chủ, đã được cấu hình sẵn và sẵn sàng sử dụng mà không cần thao tác thêm.

to handle your virtualization.​‌để quản lý ảo hóa của bạn.

You just need to handle your environment and your application.​‌Bạn chỉ cần quản lý môi trường và ứng dụng của mình.

You don't need to spend time handling your environment, your virtual environment by using the hypervisor.​‌Bạn không cần tốn thời gian quản lý môi trường của mình, môi trường ảo của bạn bằng cách sử dụng hypervisor.

This is the benefits of using the Nutanix solution technology, and especially if you use the Nutanix​‌Đây là những lợi ích của việc sử dụng công nghệ giải pháp Nutanix, đặc biệt là nếu bạn sử dụng Nutanix.

hardware.​‌phần cứng.

So it's look like you use a cloud service.​‌Vậy là có vẻ như bạn đang sử dụng dịch vụ điện toán đám mây.

You don't need to handle the storage, the CPU, the memory.​‌Bạn không cần phải lo lắng về việc lưu trữ, CPU hay bộ nhớ.

You just buy the virtual machine from the cloud service and you just concentrate on your application​‌Bạn chỉ cần mua máy ảo từ dịch vụ đám mây và tập trung vào ứng dụng của mình.

and your virtual machine that's needed in your company.​‌và máy ảo mà công ty bạn cần.

And the other problem is handled by the cloud company.​‌Vấn đề còn lại do công ty cung cấp dịch vụ điện toán đám mây xử lý.

So the same idea right here, the hardware will be handled by the Nutanix platform solution.​‌Vậy nên, ý tưởng ở đây cũng tương tự, phần cứng sẽ được xử lý bởi giải pháp nền tảng Nutanix.

So this is the idea of the Hyperconverged platform.​‌Vậy đây chính là ý tưởng của nền tảng siêu hội tụ.

So the Hyperconverged platform, as you can see, the hyper converged system there is the following​‌Như bạn thấy, nền tảng siêu hội tụ (Hyperconverged platform) bao gồm các thành phần sau:

virtually and move the controller to the host.​‌thực hiện việc chuyển đổi ảo và di chuyển bộ điều khiển đến máy chủ.

Let's mean this controller right here.​‌Ý tôi là bộ điều khiển này đây.

It's move it to the host.​‌Nó sẽ chuyển nó sang máy chủ.

Inside the host, if you remember as a KVM, provide core service and logic software distributes and​‌Bên trong máy chủ, nếu bạn nhớ đến KVM, nó cung cấp dịch vụ cốt lõi và phân phối phần mềm logic.

shared data across all nodes in the system.​‌Dữ liệu được chia sẻ giữa tất cả các nút trong hệ thống.

That means all the nodes, all the physical server will have a connection between them and a data shared​‌Điều đó có nghĩa là tất cả các nút, tất cả các máy chủ vật lý sẽ có kết nối với nhau và được chia sẻ dữ liệu.

between them.​‌giữa họ.

Then move the storage locale to the compute that's main will handle the storage to local storage to​‌Sau đó, di chuyển vị trí lưu trữ đến máy tính mà máy đó sẽ xử lý việc lưu trữ vào bộ nhớ cục bộ.

the different compute you have in your environment.​‌Các phương pháp tính toán khác nhau mà bạn có trong môi trường của mình.

So this is the idea of the Hyperconverged platform and the benefit that you can get from using the Nutanix​‌Vậy đây là ý tưởng về nền tảng siêu hội tụ và lợi ích mà bạn có thể nhận được khi sử dụng Nutanix.

solution.​‌giải pháp.

So next will understand more about how Nutanix solution work in the next lecture. ​‌Vậy nên trong bài giảng tiếp theo, chúng ta sẽ tìm hiểu thêm về cách thức hoạt động của giải pháp Nutanix.
