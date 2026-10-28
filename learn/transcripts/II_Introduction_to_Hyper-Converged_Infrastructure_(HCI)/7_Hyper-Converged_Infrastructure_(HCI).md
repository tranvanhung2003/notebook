Hello again.​‌Chào lại.

So in the preview lecture, we see the data center evolution from the classical one to the external​‌Vì vậy, trong bài giảng giới thiệu, chúng ta sẽ thấy sự phát triển của trung tâm dữ liệu từ mô hình truyền thống sang mô hình bên ngoài.

storage to the Nutanix solution by using the software defined storage.​‌Lưu trữ dữ liệu vào giải pháp Nutanix bằng cách sử dụng lưu trữ được định nghĩa bằng phần mềm.

So in this technology, it's called the hyper converged infrastructure.​‌Trong công nghệ này, nó được gọi là cơ sở hạ tầng siêu hội tụ.

So what is the hyper converged infrastructure or the hyper converged platform?​‌Vậy cơ sở hạ tầng siêu hội tụ hay nền tảng siêu hội tụ là gì?

So the hyper converged infrastructure is a software defined IT infrastructure that virtually is out​‌Vì vậy, cơ sở hạ tầng siêu hội tụ là một cơ sở hạ tầng CNTT được định nghĩa bằng phần mềm, về cơ bản là...

of the element of conventional hardware, define IT system.​‌Các yếu tố của phần cứng thông thường định nghĩa hệ thống CNTT.

The HCI include at a minimum, virtual is computing software defined storage and virtualized IT network.​‌HCI tối thiểu bao gồm điện toán ảo, phần mềm lưu trữ định nghĩa và mạng CNTT ảo hóa.

The HCI.​‌HCI.

That's mean to hyper converged typically run on commercial of the self of the shell server.​‌Điều đó có nghĩa là các hệ thống siêu hội tụ thường chạy trên nền tảng thương mại của chính máy chủ shell.

So what is the Hyperconverged platform?​‌Vậy nền tảng siêu hội tụ là gì?

To make it simple, the hyper converged infrastructure, the symbol of it is called HCI is a combination​‌Nói một cách đơn giản, cơ sở hạ tầng siêu hội tụ, ký hiệu của nó là HCI, là sự kết hợp của...

of server and storage into a distributed infrastructure platform with intelligent software to create​‌Tích hợp máy chủ và thiết bị lưu trữ vào một nền tảng cơ sở hạ tầng phân tán với phần mềm thông minh để tạo ra...

flexible building blocks that run plus legacy infrastructure consisting of separate server storage,​‌Các khối xây dựng linh hoạt hoạt động song song với cơ sở hạ tầng kế thừa bao gồm hệ thống lưu trữ máy chủ riêng biệt,

network and storage array.​‌Mạng và mảng lưu trữ.

So to be able to see that you have a hyper converged platform, you must converge and collapse the computing​‌Vì vậy, để có thể thấy rằng bạn đang sở hữu một nền tảng siêu hội tụ, bạn phải hội tụ và thu gọn hệ thống tính toán.

stack, for example, the compute and the storage.​‌Ví dụ, đó là sự kết hợp giữa khả năng tính toán và khả năng lưu trữ.

You must share or distribute the data and service across node in the system.​‌Bạn phải chia sẻ hoặc phân phối dữ liệu và dịch vụ giữa các nút trong hệ thống.

You must appear and provide the same capability as centralized storage, for example, the height of​‌Bạn phải xuất hiện và cung cấp khả năng tương tự như hệ thống lưu trữ tập trung, ví dụ như chiều cao của...

ability to live migration, etc..​‌khả năng sinh sống, di cư, v.v...

If you remember, I you do that when we have a single storage pool.​‌Nếu bạn còn nhớ, tôi thường làm vậy khi chúng ta chỉ có một bể chứa duy nhất.

This single storage pool give you the height of ability to live migration, the cloning, the snapshots,​‌Nhóm lưu trữ duy nhất này cung cấp cho bạn khả năng tối ưu hóa việc di chuyển trực tiếp, sao chép và tạo ảnh chụp nhanh.

etc..​‌vân vân..

So to be able to see that you have a hyper converged platform, your environment should support the​‌Để có thể nhận biết rằng bạn đang sử dụng nền tảng siêu hội tụ, môi trường của bạn cần phải hỗ trợ các yếu tố sau:

height, availability, the live migration and etc. that mean the same capability as centralized storage.​‌Chiều cao, tính khả dụng, khả năng di chuyển trực tiếp, v.v., có nghĩa là chúng có khả năng tương tự như lưu trữ tập trung.

The exact expression that we see it for the shared storage or external storage.​‌Cách diễn đạt chính xác mà chúng ta thường thấy đối với bộ nhớ dùng chung hoặc bộ nhớ ngoài.

That's mean when you make your platform hyper converged by using the local storage, you should have​‌Điều đó có nghĩa là khi bạn xây dựng nền tảng siêu hội tụ bằng cách sử dụng bộ nhớ cục bộ, bạn sẽ có

this capability as a centralized storage.​‌Khả năng này đóng vai trò như một kho lưu trữ tập trung.

Then you must keep data as close to the execution, the compute as possible.​‌Do đó, bạn phải giữ dữ liệu càng gần với quá trình thực thi và tính toán càng tốt.

This is important of the latency.​‌Điều này rất quan trọng đối với độ trễ.

If you remember, I tell you that we start using the local storage.​‌Nếu bạn còn nhớ, tôi đã nói với bạn rằng chúng ta bắt đầu sử dụng bộ nhớ cục bộ.

That means the application will be very close to the hard disk.​‌Điều đó có nghĩa là ứng dụng sẽ nằm rất gần ổ cứng.

So we have more speed in the latency of the storage.​‌Vì vậy, chúng ta có tốc độ nhanh hơn về độ trễ của bộ nhớ.

Then should be hypervisor agnostic.​‌Vậy thì nó phải không phụ thuộc vào hypervisor.

That's mean you can use multiple type of hypervisor and Nutanix gives you a solution for that by using​‌Điều đó có nghĩa là bạn có thể sử dụng nhiều loại hypervisor khác nhau và Nutanix cung cấp cho bạn giải pháp cho việc đó bằng cách sử dụng...

the different hypervisor have in the market.​‌Các loại hypervisor khác nhau hiện có trên thị trường.

Then should be hardware agnostic that mean you can use multiple type of hardware and not connected or​‌Vậy thì nó phải tương thích với mọi loại phần cứng, nghĩa là bạn có thể sử dụng nhiều loại phần cứng khác nhau mà không cần kết nối hoặc tương thích với nhau.

specified by one specific hardware.​‌Được xác định bởi một phần cứng cụ thể.

Of course we're going to see that the Nutanix also have its own physical server in the market that you​‌Dĩ nhiên chúng ta sẽ thấy rằng Nutanix cũng có máy chủ vật lý riêng của họ trên thị trường mà bạn

can use it, but if you don't want to use the Nutanix hardware, you can use multiple different types​‌Bạn có thể sử dụng nó, nhưng nếu bạn không muốn sử dụng phần cứng Nutanix, bạn có thể sử dụng nhiều loại khác nhau.

of hardware in the market without any problem.​‌về phần cứng trên thị trường mà không gặp bất kỳ vấn đề gì.

And this is the benefit of using the Nutanix solution. ​‌Và đây chính là lợi ích của việc sử dụng giải pháp Nutanix.
