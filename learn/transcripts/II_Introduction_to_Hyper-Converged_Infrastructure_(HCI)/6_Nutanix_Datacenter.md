Hello again.​‌Chào lại.

So let's continue our center evolution.​‌Vậy chúng ta hãy tiếp tục quá trình phát triển trung tâm của mình.

So now we have this is the famous datacentre evolution we have in our environment, the hypervisor,​‌Vậy là giờ đây chúng ta có sự phát triển nổi tiếng của trung tâm dữ liệu trong môi trường của mình, đó chính là hypervisor.

the virtual machine and x86 architecture from different server technology.​‌Máy ảo và kiến ​​trúc x86 từ các công nghệ máy chủ khác nhau.

And here you see the speed of the IOPS separated by the different physical server and the controller​‌Và ở đây bạn có thể thấy tốc độ IOPS được phân tách theo từng máy chủ vật lý và bộ điều khiển khác nhau.

and the external storage.​‌và bộ nhớ ngoài.

You see right here we have high number of server, but less performance there using this technology.​‌Như bạn thấy đấy, ở đây chúng ta có số lượng máy chủ rất lớn, nhưng hiệu suất lại thấp khi sử dụng công nghệ này.

So for that, the Nutanix solution or Nutanix technology will lead us to eliminate all this problem​‌Vì vậy, giải pháp hoặc công nghệ của Nutanix sẽ giúp chúng ta loại bỏ tất cả những vấn đề này.

and to give you a more powerful solution and more powerful datacenter environment.​‌và để cung cấp cho bạn một giải pháp mạnh mẽ hơn và môi trường trung tâm dữ liệu hiệu quả hơn.

By giving a solution of the controllers, Nutanix concentrates on the controller of the storage instead​‌Bằng cách cung cấp giải pháp cho bộ điều khiển, Nutanix tập trung vào bộ điều khiển của thiết bị lưu trữ.

of using an external controller in external storage.​‌về việc sử dụng bộ điều khiển bên ngoài trong bộ nhớ ngoài.

Will take this technology and change it inside every host by creating a virtual machine controller that​‌Sẽ áp dụng công nghệ này và thay đổi nó bên trong mọi máy chủ bằng cách tạo ra bộ điều khiển máy ảo.

will control the input outputs per second from the different virtual machine.​‌sẽ điều khiển số lượng dữ liệu đầu vào/đầu ra mỗi giây từ các máy ảo khác nhau.

And the same will only use the local storage instead of using the external storage, the Sun and NAS,​‌Và điều tương tự cũng sẽ chỉ sử dụng bộ nhớ trong thay vì bộ nhớ ngoài, như Sun và NAS.

etc. But here you will only use the local storage.​‌vân vân. Nhưng ở đây bạn chỉ sử dụng bộ nhớ cục bộ.

So you see right here we eliminate all this environment right here, the external storage, the controller,​‌Như bạn thấy đấy, ngay tại đây chúng ta đã loại bỏ toàn bộ môi trường này, bộ nhớ ngoài, bộ điều khiển,

the cable to go inside the bare metal server, the physical host, and.​‌cáp để đi vào bên trong máy chủ vật lý (bare metal server), máy chủ ảo (physical host), và...

Here's the look of storage instead of the external storage that is more expensive than using the local​‌Đây là hình ảnh về bộ nhớ trong thay vì bộ nhớ ngoài, vốn đắt hơn so với việc sử dụng bộ nhớ trong cục bộ.

storage.​‌kho.

So right here you see that the distance between the virtual machine and the storage is very, very small.​‌Như bạn thấy ở đây, khoảng cách giữa máy ảo và thiết bị lưu trữ là rất, rất nhỏ.

Then using this technology, so you see the virtual machine right here, you need to go through the​‌Sau đó, sử dụng công nghệ này, như bạn thấy máy ảo ở đây, bạn cần phải thực hiện các bước sau:

network.​‌mạng.

If you have a speed limit network, you will have speed limit IOPS.​‌Nếu mạng của bạn có giới hạn tốc độ, thì IOPS (số thao tác đọc/ghi mỗi giây) cũng sẽ có giới hạn tốc độ.

Then the controller, then the storage.​‌Tiếp theo là bộ điều khiển, rồi đến bộ nhớ lưu trữ.

Then if you have a problem in the network, your virtual machine will not be able to communicate to​‌Nếu xảy ra sự cố mạng, máy ảo của bạn sẽ không thể giao tiếp được với hệ thống.

the storage.​‌kho lưu trữ.

If you have a problem in this storage, you will lost all your environment and not lots of problems.​‌Nếu bạn gặp sự cố với bộ nhớ này, bạn sẽ mất toàn bộ dữ liệu và không chỉ vậy, mà còn nhiều vấn đề khác nữa.

For example, the controller one data or this filer, you will have an overload on the controller.​‌Ví dụ, với bộ điều khiển dữ liệu hoặc bộ lọc này, bạn sẽ gặp tình trạng quá tải trên bộ điều khiển.

Number two, if for example the network of a problem, etc., you have lots of troubleshooting by using​‌Thứ hai, nếu chẳng hạn mạng gặp sự cố, v.v., bạn có thể khắc phục sự cố bằng cách sử dụng nhiều phương pháp khác nhau.

this environment.​‌môi trường này.

But when we use the Nutanix solution will eliminate a lot of problems and a lot of hardware to manage​‌Nhưng khi sử dụng giải pháp Nutanix, chúng ta sẽ loại bỏ được rất nhiều vấn đề và giảm bớt khối lượng phần cứng cần quản lý.

our environment by making the controller as a virtual machine.​‌môi trường của chúng ta được cải thiện bằng cách biến bộ điều khiển thành một máy ảo.

So this is a virtual machine created by the Nutanix software will be a controller in your physical host​‌Vậy đây là một máy ảo được tạo bởi phần mềm Nutanix, sẽ đóng vai trò là bộ điều khiển trong máy chủ vật lý của bạn.

and will use the local storage from every server.​‌và sẽ sử dụng bộ nhớ cục bộ từ mọi máy chủ.

It is a hybrid storage By using the Flash and the HDD, that means SSD and HDD.​‌Đây là một loại lưu trữ lai bằng cách sử dụng bộ nhớ Flash và ổ cứng HDD, tức là sự kết hợp giữa SSD và HDD.

You can use them together without any problem.​‌Bạn có thể sử dụng chúng cùng nhau mà không gặp vấn đề gì.

If we use the Nutanix solution right here, what do not an acceleration give you, It will take all​‌Nếu chúng ta sử dụng giải pháp Nutanix ngay tại đây, thì việc tăng tốc sẽ mang lại lợi ích gì? Nó sẽ giải quyết tất cả.

this local storage, direct attached storage from every physical host and will create our one big storage.​‌Bộ nhớ cục bộ này, bộ nhớ gắn trực tiếp từ mỗi máy chủ vật lý sẽ tạo thành một kho lưu trữ lớn duy nhất của chúng ta.

That give you the size of the total of the different storage in your environment and will be like a​‌Điều đó cho bạn biết tổng dung lượng của các loại kho lưu trữ khác nhau trong môi trường của bạn và sẽ giống như một

one big storage that Scarlett's single storage pool.​‌một kho chứa đồ lớn mà Scarlett dùng để chứa đồ.

It will create a pool storage in your environment from the different physical hosts.​‌Nó sẽ tạo ra một vùng lưu trữ chung trong môi trường của bạn từ các máy chủ vật lý khác nhau.

Of course, every host should have a KVM, a controller virtual machine by using the software of Nutanix.​‌Dĩ nhiên, mỗi máy chủ đều cần có một KVM, một máy ảo điều khiển được thực hiện bằng phần mềm của Nutanix.

Every physical host that have this kvm inside, it will use the storage of it to add it to the storage​‌Mỗi máy chủ vật lý có chứa KVM này sẽ sử dụng bộ nhớ của nó để thêm vào kho lưu trữ.

pool.​‌hồ bơi.

So right here we get all the storage as a one big storage when we create the single storage pool.​‌Như vậy, ngay tại đây, chúng ta có toàn bộ dung lượng lưu trữ dưới dạng một kho lưu trữ lớn duy nhất khi tạo ra một nhóm lưu trữ duy nhất.

This solution will give you a native support for the snapshots, the client, the compression, the​‌Giải pháp này sẽ cung cấp cho bạn sự hỗ trợ nguyên bản cho ảnh chụp nhanh, máy khách, nén, v.v.

deduplication, etc. Instead of using this technology, that means every host have its own virtual machine​‌khử trùng lặp, v.v. Thay vì sử dụng công nghệ này, điều đó có nghĩa là mỗi máy chủ sẽ có máy ảo riêng.

and our own local storage.​‌và kho lưu trữ cục bộ của riêng chúng tôi.

You don't have the capability to have this option right here, for example, the motion, the deduplication,​‌Bạn không có khả năng tích hợp tùy chọn này ngay tại đây, ví dụ như chuyển động, loại bỏ dữ liệu trùng lặp,

etc. because it is not available in the local storage.​‌vân vân, vì nó không có sẵn trong bộ nhớ cục bộ.

But by using the Nutanix solution that we are going to create our single pool storage from the local​‌Nhưng bằng cách sử dụng giải pháp Nutanix, chúng ta sẽ tạo ra vùng lưu trữ đơn từ máy chủ cục bộ.

storage.​‌kho.

It's like you convert this local storage as an external storage, so then you will have all the capability​‌Nó giống như việc bạn chuyển đổi bộ nhớ cục bộ này thành bộ nhớ ngoài, nhờ đó bạn sẽ có đầy đủ khả năng.

to get the different future of the external storage, like snapshots, clone compression, deduplication,​‌Để khai thác các tính năng khác nhau của bộ nhớ ngoài, như ảnh chụp nhanh, sao chép, nén, khử trùng lặp,

locality and so on.​‌địa phương, v.v.

We're going to discover them later.​‌Chúng ta sẽ tìm ra chúng sau.

So right here you see that we have one big storage.​‌Như các bạn thấy ở đây, chúng ta có một kho chứa đồ lớn.

So the benefits again that we get from using this technology, the Nutanix solution is when we try to​‌Vì vậy, một lần nữa, lợi ích mà chúng ta nhận được từ việc sử dụng công nghệ này, giải pháp Nutanix, là khi chúng ta cố gắng

upgrade or to scale up the environments by adding a new physical server.​‌Nâng cấp hoặc mở rộng quy mô môi trường bằng cách thêm máy chủ vật lý mới.

You see right here we have more number of servers and the same for the performance.​‌Như bạn thấy đấy, ở đây chúng ta có nhiều máy chủ hơn và hiệu năng cũng vậy.

We have more upgraded performance instead of the preview technology.​‌Chúng tôi đã nâng cấp hiệu năng lên một tầm cao mới thay vì chỉ sử dụng công nghệ xem trước.

The performance be degraded every time when we add a new physical host.​‌Hiệu năng sẽ bị suy giảm mỗi khi chúng ta thêm một máy chủ vật lý mới.

But in this environment, in this new technology, when we add a physical host, the performance become​‌Nhưng trong môi trường này, với công nghệ mới này, khi chúng ta thêm một máy chủ vật lý, hiệu năng sẽ trở nên...

more because we add to our environment and your CPU and your memory and a new size hard disk without​‌nhiều hơn vì chúng tôi bổ sung vào môi trường của bạn, CPU, bộ nhớ và ổ cứng có dung lượng mới mà không cần

needing to consume the IOPS of the other physical host.​‌cần sử dụng IOPS của máy chủ vật lý khác.

So for example, these hosts have 100 IOPS.​‌Ví dụ, các máy chủ này có 100 IOPS.

This host also, for example, have 100 IOPS.​‌Ví dụ, máy chủ này cũng có 100 IOPS.

Just an example because if we take this is the same brand physical hosts.​‌Đây chỉ là một ví dụ vì nếu chúng ta lấy cùng một thương hiệu máy chủ vật lý.

We have three physical hosts from the same brand, the same speed IOPS.​‌Chúng tôi có ba máy chủ vật lý cùng nhãn hiệu, cùng tốc độ IOPS.

So this one have 100, this one have 100, this one have 100.​‌Vậy cái này có 100, cái này có 100, cái này cũng có 100.

If we add the same physical hosts and the same brand and the same speed, we have a new 100 IOPS in​‌Nếu chúng ta thêm cùng các máy chủ vật lý, cùng thương hiệu và cùng tốc độ, chúng ta sẽ có thêm 100 IOPS.

our environment, so the total will be up instead of degraded in the previous example.​‌Môi trường của chúng ta sẽ như vậy, vì vậy tổng số sẽ tăng lên thay vì giảm xuống như trong ví dụ trước.

So right here we have more size, we have more hard disk, and we have one single storage pool that​‌Vậy là ở đây chúng ta có dung lượng lớn hơn, nhiều ổ cứng hơn, và chúng ta có một nhóm lưu trữ duy nhất.

will manage all them together.​‌sẽ quản lý tất cả chúng cùng nhau.

So this is the datacenter evolution from the classical one to the external storage.​‌Như vậy, đây là sự tiến hóa của trung tâm dữ liệu từ mô hình truyền thống sang lưu trữ ngoài.

No, we have the as there is by using the Nutanix solution.​‌Không, chúng tôi có giải pháp tương tự như khi sử dụng giải pháp của Nutanix.

And next we're going to talk about how this solution work and how we can create this solution in our​‌Tiếp theo, chúng ta sẽ thảo luận về cách giải pháp này hoạt động và cách chúng ta có thể tạo ra giải pháp này trong dự án của mình.

environment.​‌môi trường.

So see in the next lecture. ​‌Hãy cùng chờ đón bài giảng tiếp theo.
