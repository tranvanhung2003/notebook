Hello and welcome to this new lecture about Nutanix Solution Overview.​‌Xin chào và chào mừng các bạn đến với bài giảng mới này về tổng quan giải pháp Nutanix.

So before being able to start practicing by using the new techniques, software and solution, you need​‌Vì vậy, trước khi có thể bắt đầu thực hành bằng cách sử dụng các kỹ thuật, phần mềm và giải pháp mới, bạn cần phải...

to have some theory information you can't understand.​‌Có một số thông tin lý thuyết mà bạn không thể hiểu.

Do not annex solution platform and use the software without having due to your knowledge.​‌Không nên sao chép nền tảng giải pháp và sử dụng phần mềm mà không có sự cho phép của bạn.

So you need to understand how Nutanix work correctly.​‌Vì vậy, bạn cần hiểu đúng cách thức hoạt động của Nutanix.

Then you can easily start practicing what you will learn in the real environment.​‌Sau đó, bạn có thể dễ dàng bắt đầu thực hành những gì mình đã học trong môi trường thực tế.

So for that, the first thing you need to know is about the storage, because as we know, the main​‌Vì vậy, điều đầu tiên bạn cần biết là về dung lượng lưu trữ, bởi vì như chúng ta đã biết, phần chính

feature of the Nutanix solution is the storage.​‌Điểm nổi bật của giải pháp Nutanix chính là khả năng lưu trữ.

The first software created from Nutanix is to give you a solution for the storage problem and the storage​‌Phần mềm đầu tiên do Nutanix tạo ra nhằm cung cấp giải pháp cho vấn đề lưu trữ.

issue and the different troubleshooting about the storage.​‌vấn đề và các phương pháp khắc phục sự cố khác nhau liên quan đến lưu trữ.

So Nutanix creates a software to help you easily manage your storage environment.​‌Vì vậy, Nutanix tạo ra phần mềm giúp bạn dễ dàng quản lý môi trường lưu trữ của mình.

And eliminates are those problems that come with the storage.​‌Và nó loại bỏ những vấn đề phát sinh từ việc lưu trữ.

So for that, we need to have an idea about the storage.​‌Vì vậy, chúng ta cần có ý tưởng về dung lượng lưu trữ.

So the first thing we need to know the different type of storage we have in our environment.​‌Vậy điều đầu tiên chúng ta cần biết là các loại lưu trữ khác nhau mà chúng ta có trong môi trường của mình.

So the first one that's come with the server in the classical environment is the does that mean the​‌Vậy cái đầu tiên đi kèm với máy chủ trong môi trường cổ điển là gì? Điều đó có nghĩa là...

direct attached storage?​‌Lưu trữ gắn trực tiếp?

Then we have the NAS network attached storage, we have the sun storage area network and we have object​‌Tiếp theo, chúng ta có thiết bị lưu trữ mạng NAS, mạng lưu trữ Sun và Object.

storage, the computer data storage, and we have the SD as the software defined storage.​‌Bộ nhớ lưu trữ, bộ nhớ lưu trữ dữ liệu máy tính, và chúng ta có SD là bộ nhớ được định nghĩa bằng phần mềm.

And our course will concentrate about the software defined storage because this is the solution that​‌Khóa học của chúng ta sẽ tập trung vào lưu trữ được định nghĩa bằng phần mềm vì đây là giải pháp

created by the Nutanix cloud environment.​‌Được tạo bởi môi trường điện toán đám mây Nutanix.

So for that we have the direct attached storage.​‌Vì vậy, chúng ta có bộ nhớ gắn trực tiếp.

As we know, the direct data storage is the storage that is connected directly to the server, to the​‌Như chúng ta đã biết, lưu trữ dữ liệu trực tiếp là loại lưu trữ được kết nối trực tiếp với máy chủ.

physical hosts like the HDD, the SSD, the NVME, etc. This is the storage that is connected directly​‌Các thiết bị lưu trữ vật lý như HDD, SSD, NVME, v.v. Đây là thiết bị lưu trữ được kết nối trực tiếp.

to the motherboard of the host.​‌đến bo mạch chủ của máy chủ.

That's mean we don't have any network connection to connect the storage with the server.​‌Điều đó có nghĩa là chúng ta không có kết nối mạng nào để kết nối thiết bị lưu trữ với máy chủ.

When we install the application in a server, it is connected directly to the storage without any necessary​‌Khi chúng ta cài đặt ứng dụng trên máy chủ, nó sẽ được kết nối trực tiếp với thiết bị lưu trữ mà không cần bất kỳ thao tác trung gian nào.

for a network or a cable network just by using the direct attached storage.​‌Đối với mạng nội bộ hoặc mạng cáp, chỉ cần sử dụng thiết bị lưu trữ gắn trực tiếp.

And this is very important because when we start using the Nutanix solution are going to back to the​‌Và điều này rất quan trọng bởi vì khi chúng ta bắt đầu sử dụng giải pháp Nutanix, chúng ta sẽ quay lại với...

direct attached storage.​‌Lưu trữ gắn trực tiếp.

And this is why I explain you about the direct attached storage, because this is the main future of​‌Và đó là lý do tại sao tôi giải thích cho bạn về bộ nhớ gắn trực tiếp, bởi vì đây là tương lai chính của nó.

the Nutanix solution, because it concentrates on the direct attached storage with a new technology.​‌Giải pháp của Nutanix tập trung vào lưu trữ gắn trực tiếp với công nghệ mới.

So then we have the NAS, the network attached storage, this one that we use in our network as an external​‌Vậy là chúng ta có NAS, thiết bị lưu trữ gắn mạng, thiết bị mà chúng ta sử dụng trong mạng của mình như một thiết bị lưu trữ ngoài.

storage or as a shared storage.​‌lưu trữ riêng tư hoặc sử dụng làm bộ nhớ dùng chung.

That's mean multiple server.​‌Điều đó có nghĩa là nhiều máy chủ.

We connect to this storage and they can see the same storage together.​‌Chúng tôi kết nối với bộ nhớ này và họ có thể cùng nhau truy cập vào cùng một bộ nhớ.

By using the network.​‌Bằng cách sử dụng mạng.

And of course, in the last technology we use the protocol, SNMP and NFS.​‌Và tất nhiên, trong công nghệ mới nhất, chúng ta sử dụng giao thức SNMP và NFS.

This protocol will see that we will need it when we start using the Nutanix solution because it will​‌Giao thức này sẽ cho thấy rằng chúng ta sẽ cần nó khi bắt đầu sử dụng giải pháp Nutanix vì nó sẽ

handle the different protocol available for the storage by giving the solution.​‌Giải quyết các vấn đề liên quan đến giao thức lưu trữ khác nhau bằng cách đưa ra giải pháp.

So this storage, we use it as a file storage.​‌Vậy nên, chúng ta sử dụng bộ nhớ này để lưu trữ tệp tin.

If you have some knowledge about Nas and Sun, you will know that the NAS is a file storage.​‌Nếu bạn có chút hiểu biết về NAS và Sun, bạn sẽ biết rằng NAS là thiết bị lưu trữ tập tin.

It's like a shared file or a shared storage between different server.​‌Nó giống như một tập tin được chia sẻ hoặc một bộ nhớ được chia sẻ giữa các máy chủ khác nhau.

The same in Netflix.​‌Điều tương tự cũng xảy ra trên Netflix.

We have a solution for this option right here that's called Nutanix files.​‌Chúng tôi có một giải pháp cho tùy chọn này ngay tại đây, được gọi là các tệp Nutanix.

That mean if we use the Nutanix file software, it will lead you to create storage like the same as​‌Điều đó có nghĩa là nếu chúng ta sử dụng phần mềm quản lý tệp Nutanix, nó sẽ giúp bạn tạo ra bộ nhớ lưu trữ tương tự như...

you get from the NAS technology.​‌Bạn nhận được điều đó từ công nghệ NAS.

Then we have the same storage, the storage area network.​‌Vậy là chúng ta có cùng một hệ thống lưu trữ, đó là mạng lưu trữ khu vực (SAN).

This is a picture of our sun storage.​‌Đây là hình ảnh hệ thống lưu trữ năng lượng mặt trời của chúng tôi.

This is the same idea as NAS, but with another technology that need the Fibre Channel switch.​‌Ý tưởng này tương tự như NAS, nhưng sử dụng một công nghệ khác cần đến bộ chuyển mạch Fibre Channel.

It need Fibre Channel cable.​‌Cần cáp Fibre Channel.

This is more advanced and more expensive than the NAS and more faster in the network communication.​‌Thiết bị này tiên tiến hơn, đắt hơn so với NAS và có tốc độ truyền tải mạng nhanh hơn.

The protocol use it is the iSCSI, the Fibre Channel and the Fibre Channel over Ethernet.​‌Các giao thức được sử dụng là iSCSI, Fibre Channel và Fibre Channel qua Ethernet.

That's mean you use the Ethernet cable, but you use the protocol.​‌Điều đó có nghĩa là bạn sử dụng cáp Ethernet, nhưng bạn sử dụng giao thức.

That's Khalid's Fiber channel over Ethernet to be able to manage your sun storage by using the Ethernet​‌Đó là giải pháp Fiber Channel qua Ethernet của Khalid, cho phép bạn quản lý hệ thống lưu trữ năng lượng mặt trời của mình thông qua Ethernet.

cable.​‌cáp.

So this is a block storage that's mean.​‌Đây là một hệ thống lưu trữ khối, nghĩa là vậy.

It gives you the storage as a form of block and not at the form of the file.​‌Nó cung cấp cho bạn dung lượng lưu trữ dưới dạng khối chứ không phải dưới dạng tập tin.

So the same in Nutanix.​‌Tương tự với Nutanix.

We have a solution called Nutanix block volume.​‌Chúng tôi có một giải pháp gọi là Nutanix block volume.

That's mean.​‌Thật là độc ác.

When we use the Nutanix solution, we can create block volume by using the Nutanix solution.​‌Khi sử dụng giải pháp Nutanix, chúng ta có thể tạo khối dung lượng lưu trữ (block volume) bằng cách sử dụng giải pháp Nutanix.

That's mean.​‌Thật là độc ác.

Nutanix can give you a SAN solution and a nice solution.​‌Nutanix có thể cung cấp cho bạn một giải pháp SAN tuyệt vời.

And of course, the direct attached storage.​‌Và dĩ nhiên, cả bộ nhớ gắn trực tiếp nữa.

This is the one that we're going to use it.​‌Đây là cái mà chúng ta sẽ sử dụng.

Then we have a quick overview to see how the difference between the different types of storage that​‌Tiếp theo, chúng ta sẽ có một cái nhìn tổng quan nhanh chóng để xem sự khác biệt giữa các loại lưu trữ khác nhau.

we see in the does do NAS and the sun.​‌Chúng ta thấy trong đó NAS và mặt trời.

So right here you see that in the dust.​‌Như bạn thấy đấy, ngay tại đây, trong lớp bụi.

You send a request from the network to the server.​‌Bạn gửi một yêu cầu từ mạng đến máy chủ.

Every server will save the data in the local disk of it that's in the server.​‌Mỗi máy chủ sẽ lưu trữ dữ liệu trên ổ đĩa cục bộ của chính máy chủ đó.

Have access to this disk.​‌Tôi có quyền truy cập vào đĩa này.

The server have access only to his own disk and the same for the third server.​‌Máy chủ chỉ có quyền truy cập vào ổ đĩa của riêng nó và máy chủ thứ ba cũng vậy.

Have access to its own disk directly without needing to have any hardware right here.​‌Có thể truy cập trực tiếp vào ổ đĩa của chính nó mà không cần bất kỳ phần cứng nào tại chỗ.

Only the direct attached storage for donuts.​‌Chỉ có ngăn chứa đồ gắn trực tiếp dành cho bánh rán.

If a client send a request to the server, all the different server, we need to send a request to the​‌Nếu một máy khách gửi yêu cầu đến máy chủ, thì tất cả các máy chủ khác nhau mà chúng ta cần gửi yêu cầu đến...

file server that will handle the storage from the nurse, and then the file server will connect to the​‌Máy chủ tập tin sẽ xử lý việc lưu trữ dữ liệu từ y tá, và sau đó máy chủ tập tin sẽ kết nối với...

different storage and give the information or the request response to the user or to the client.​‌lưu trữ thông tin khác nhau và cung cấp phản hồi yêu cầu cho người dùng hoặc khách hàng.

So this is how it's look.​‌Đây là hình dáng của nó.

Like the NAS, then the Sun.​‌Giống như NAS, rồi đến Mặt Trời.

If the client send a request to the server, the server will need to connect to a specific network,​‌Nếu máy khách gửi yêu cầu đến máy chủ, máy chủ sẽ cần kết nối với một mạng cụ thể.

the Fibre Channel or the fibre switch right here.​‌Kênh Fibre Channel hoặc bộ chuyển mạch quang ngay tại đây.

Then the fibre switch will send the data to the different disk available in the network.​‌Sau đó, bộ chuyển mạch quang sẽ gửi dữ liệu đến các ổ đĩa khác nhau có sẵn trong mạng.

So this is a quick idea.​‌Đây là một ý tưởng nhanh chóng.

And here you see that this is the more simple and don't need any other hardware.​‌Và như bạn thấy, đây là cách đơn giản hơn và không cần bất kỳ phần cứng nào khác.

This one need more hardware switch and this one need more expensive hardware.​‌Cái này cần nhiều công tắc phần cứng hơn và cái kia cần phần cứng đắt tiền hơn.

The Fibre Channel switch.​‌Bộ chuyển mạch Fibre Channel.

If we check it right here, you see this other example right here?​‌Nếu chúng ta xem xét ngay tại đây, bạn sẽ thấy một ví dụ khác ngay đây chứ?

You see that the does is directly connected between the server and the storage.​‌Bạn thấy đấy, nó được kết nối trực tiếp giữa máy chủ và thiết bị lưu trữ.

We don't need to have more hardware necessary to manage our storage from the server, but in the NAS,​‌Chúng ta không cần thêm phần cứng nào để quản lý dung lượng lưu trữ từ máy chủ, nhưng đối với NAS thì khác,

we need an ethernet switch and a cable right here.​‌Chúng ta cần một bộ chuyển mạch Ethernet và một sợi cáp ngay tại đây.

You see you have multiple cable, multiple Ethernet switch.​‌Như bạn thấy, bạn có nhiều dây cáp, nhiều bộ chuyển mạch Ethernet.

Of course, if you have a big environment, you will have multiple switch and then do storage as external​‌Tất nhiên, nếu bạn có một môi trường lớn, bạn sẽ cần nhiều thiết bị chuyển mạch và sau đó lưu trữ dữ liệu bên ngoài.

storage.​‌kho.

But in the sun you see that we need Fibre Channel switch.​‌Nhưng dưới ánh nắng mặt trời, bạn sẽ thấy rằng chúng ta cần bộ chuyển mạch Fibre Channel.

That's mean, you need a Fibre Channel switch to be able to create a sun environment and then the external​‌Điều đó có nghĩa là bạn cần một bộ chuyển mạch Fibre Channel để có thể tạo môi trường ánh sáng mặt trời và sau đó là kết nối bên ngoài.

storage, the cable right here you need Fibre Channel cable or if you use the Fibre Channel of a Ethernet​‌Về phần lưu trữ, dây cáp ở đây bạn cần là cáp Fibre Channel hoặc nếu bạn sử dụng Fibre Channel thì là cáp Ethernet.

cable.​‌cáp.

So you see it's very expensive to create environment by using the different technology. ​‌Như vậy, bạn thấy đấy, việc tạo ra môi trường bằng cách sử dụng các công nghệ khác nhau rất tốn kém.
