So this is an example of one need to be able to see that you have a Nutanix cluster.​‌Đây là một ví dụ cho thấy bạn cần phải biết mình đang sở hữu một cụm máy chủ Nutanix.

You should have at least three node with the same architecture.​‌Bạn nên có ít nhất ba nút có cùng kiến ​​trúc.

So you see, this is the node that we talk about it.​‌Như bạn thấy đấy, đây chính là nút mà chúng ta đang nói đến.

We have a single node with the same architecture and the third node with the same architecture.​‌Chúng ta có một nút đơn với cùng kiến ​​trúc và nút thứ ba cũng có cùng kiến ​​trúc.

So the same we have the hardware, the local or direct attached storage, the flash storage for the​‌Vì vậy, chúng ta cũng có phần cứng, bộ nhớ cục bộ hoặc bộ nhớ gắn trực tiếp, bộ nhớ flash cho...

casing, the HDD for the storage, or for the data safe, the HBP to manage the storage, the KVM,​‌Vỏ máy, ổ cứng HDD để lưu trữ hoặc để bảo quản dữ liệu, HBP để quản lý bộ nhớ, KVM,

the virtual machine controller to control the storage, the hypervisor that you use, any kind of hypervisor,​‌Bộ điều khiển máy ảo để điều khiển bộ nhớ, trình ảo hóa mà bạn sử dụng, bất kỳ loại trình ảo hóa nào,

the VMware ESXi, Hyper-V or the native Nutanix hypervisor, then the Acropolis operation system that​‌VMware ESXi, Hyper-V hoặc hypervisor gốc của Nutanix, sau đó là hệ điều hành Acropolis.

will be handled storage of the local disk.​‌Việc lưu trữ trên ổ đĩa cục bộ sẽ được thực hiện.

And then we create our virtual machine inside every node.​‌Và sau đó chúng ta tạo máy ảo bên trong mỗi nút.

And when we have three nodes with the same architecture, no, we can see that we have our Nutanix cluster​‌Và khi chúng ta có ba nút với cùng một kiến ​​trúc, thì đúng vậy, chúng ta có thể thấy rằng chúng ta có cụm Nutanix của mình.

and this control virtual machine will have a connection between them to manage the cluster.​‌Và máy ảo điều khiển này sẽ có kết nối với chúng để quản lý cụm máy chủ.

So we have connection between them through the entire KVM network or entire controller virtual machine​‌Vì vậy, chúng ta có kết nối giữa chúng thông qua toàn bộ mạng KVM hoặc toàn bộ máy ảo điều khiển.

network.​‌mạng.

So this controller virtual machine network communicate to each other to manage the environment perfectly​‌Vì vậy, mạng máy ảo điều khiển này giao tiếp với nhau để quản lý môi trường một cách hoàn hảo.

and manage the storage.​‌và quản lý kho lưu trữ.

Of course, you should have the same network ID between them.​‌Dĩ nhiên, cả hai thiết bị phải có cùng ID mạng.

This VM will have an EPI.​‌Máy ảo này sẽ có EPI.

This kvm also will have an IP address so the different IP addresses should be in the same network to​‌KVM này cũng sẽ có địa chỉ IP, vì vậy các địa chỉ IP khác nhau phải nằm trong cùng một mạng.

be able to communicate to each other.​‌Có khả năng giao tiếp với nhau.

So now we have this connection between them when we have a KVM in every physical.​‌Vậy là giờ đây chúng ta đã có sự kết nối giữa chúng khi mỗi máy vật lý đều có một thiết bị KVM.

Host No, we'll have a new distributed storage fabric layer will be created when we have three physical​‌Không, chúng ta sẽ tạo một lớp kiến ​​trúc lưu trữ phân tán mới khi chúng ta có ba thiết bị vật lý.

hosts that have the same architecture of the using the KVM.​‌các máy chủ có cùng kiến ​​trúc sử dụng KVM.

So here it will convert the local storage.​‌Vì vậy, ở đây nó sẽ chuyển đổi bộ nhớ cục bộ.

All this local storage that we have in the different server through the civil or the Acropolis operation​‌Tất cả bộ nhớ cục bộ mà chúng ta có trên các máy chủ khác nhau thông qua hoạt động dân sự hoặc hoạt động Acropolis.

system will be convergent or be merged to be one distributed storage fabrics that we collect our storage​‌Hệ thống sẽ hội tụ hoặc được hợp nhất thành một hệ thống lưu trữ phân tán duy nhất mà chúng ta thu thập dữ liệu lưu trữ.

pool.​‌hồ bơi.

So no, it will merge all the storage from this node, A node B and not C as a one storage pole that​‌Vậy nên không, nó sẽ hợp nhất tất cả bộ nhớ từ nút này, nút A, nút B chứ không phải C thành một cực lưu trữ duy nhất.

we call it distributed storage fabric.​‌Chúng tôi gọi đó là kiến ​​trúc lưu trữ phân tán.

So this is the layer of the Nutanix solution.​‌Đây là lớp dung dịch Nutanix.

It's work in this layer of the hard disk or the storage as a one distributed storage.​‌Nó hoạt động ở lớp này của ổ cứng hoặc bộ nhớ như một hệ thống lưu trữ phân tán.

So the storage will be shared between Node A, node B and C, So this node right here can see the same​‌Vì vậy, bộ nhớ sẽ được chia sẻ giữa Node A, node B và C, nên node này cũng có thể thấy cùng một bộ nhớ.

storage, even it exists in another server.​‌lưu trữ, ngay cả khi nó tồn tại trên một máy chủ khác.

So for example, the virtual machine, this one from the node A it can save the data in the storage​‌Ví dụ, máy ảo này, từ nút A, có thể lưu dữ liệu vào bộ nhớ.

locally or even in this local storage from the A.P. and the local storage from Node C as a shared storage​‌cục bộ hoặc thậm chí trong bộ nhớ cục bộ này từ AP và bộ nhớ cục bộ từ Node C như một bộ nhớ dùng chung.

between them.​‌giữa họ.

And you see right here that we have latency right here.​‌Và bạn có thể thấy ngay ở đây, chúng ta có độ trễ.

You see that the virtual machine is closely to the storage inside it.​‌Bạn thấy đấy, máy ảo nằm rất gần với thiết bị lưu trữ bên trong nó.

And this is the benefit of using the Nutanix solution.​‌Và đây chính là lợi ích của việc sử dụng giải pháp Nutanix.

So right here, when we use at least three node with the server, you see that we have a distributed​‌Như vậy, khi chúng ta sử dụng ít nhất ba nút với máy chủ, bạn sẽ thấy rằng chúng ta có một hệ thống phân tán.

storage fabric that gives you the capability to use multiple future like the snapshot, the duplicate​‌Kiến trúc lưu trữ cho phép bạn sử dụng nhiều tính năng như ảnh chụp nhanh, bản sao.

era encoding, resilience, compression, clone tiering locality and so on and so on.​‌Mã hóa theo kỷ nguyên, khả năng phục hồi, nén dữ liệu, phân tầng bản sao, tính cục bộ, v.v...

So this is the benefit of using the Nutanix solution cluster and how it's work.​‌Đây là lợi ích của việc sử dụng giải pháp cụm máy chủ Nutanix và cách thức hoạt động của nó.

And the same right here, you see that we have a network between the different virtual machine by using​‌Và cũng tương tự như vậy ở đây, bạn thấy rằng chúng ta có một mạng lưới giữa các máy ảo khác nhau bằng cách sử dụng

the simple network, the Ethernet network, between the server to this one and the server to this one.​‌Mạng đơn giản, mạng Ethernet, kết nối giữa máy chủ này với máy chủ kia và giữa máy chủ này với máy chủ khác.

This virtual machine will communicate to each other by using the simple network we have in our environment.​‌Máy ảo này sẽ giao tiếp với nhau bằng cách sử dụng mạng đơn giản mà chúng ta có trong môi trường của mình.

Of course, we should have one gigabit Ethernet network adapter and if you have ten it will be better.​‌Dĩ nhiên, chúng ta nên có một bộ chuyển đổi mạng Ethernet gigabit, và nếu có mười cái thì càng tốt.

And of course you know that the server come with one gigabyte at least of speed and some of them come​‌Và tất nhiên bạn biết rằng máy chủ thường có tốc độ ít nhất một gigabit, và một số máy chủ còn có tốc độ cao hơn nữa.

with the ten gigabytes.​‌với mười gigabyte.

And of course the switch should support at least ten gigabyte speed of the network adapter.​‌Và tất nhiên, thiết bị chuyển mạch phải hỗ trợ tốc độ ít nhất mười gigabit của bộ chuyển đổi mạng.

So right here we create a distributed fabric and get some future of the distributed storage in any other​‌Vì vậy, ngay tại đây chúng ta tạo ra một kiến ​​trúc phân tán và khai thác tiềm năng của lưu trữ phân tán trong tương lai.

data center infrastructure. ​‌Cơ sở hạ tầng trung tâm dữ liệu.
