Hello again.​‌Chào lại.

So, no, let's talk a little about the evolution of the data center.​‌Vậy thì, hãy cùng bàn một chút về sự phát triển của trung tâm dữ liệu.

If you have some experience with the VMware or you come from my other course about VMware, you know​‌Nếu bạn đã có kinh nghiệm sử dụng VMware hoặc đã tham gia khóa học VMware khác của tôi, bạn sẽ biết điều này.

that we talk about environment as a data center and cluster and some other expression we have in the​‌Chúng ta nói về môi trường như một trung tâm dữ liệu, một cụm máy chủ và một số thuật ngữ khác mà chúng ta sử dụng.

virtualization environment.​‌Môi trường ảo hóa.

So here we see that, we talk about the different storage.​‌Như vậy, ở đây chúng ta thấy rằng, chúng ta đang nói về các loại lưu trữ khác nhau.

We have the dust donuts, the sun, the object storage, and is this the software defined storage.​‌Chúng ta có những vòng bụi, mặt trời, bộ nhớ đối tượng, và đây có phải là bộ nhớ được định nghĩa bằng phần mềm không?

So here when we talk about the datacenter, you see that the data center go from the dust to the is​‌Vậy nên khi chúng ta nói về trung tâm dữ liệu, bạn sẽ thấy rằng trung tâm dữ liệu được xây dựng từ con số không cho đến khi trở thành một hệ thống hoàn chỉnh.

the s and what is the benefits we get when we pass from one type storage to other type storage and the​‌và những lợi ích chúng ta nhận được khi chuyển từ loại lưu trữ này sang loại lưu trữ khác là gì và

benefit that you can get in your environment and why Nutanix is very, very helpful in your environment.​‌Lợi ích mà bạn có thể nhận được trong môi trường của mình và lý do tại sao Nutanix lại rất, rất hữu ích trong môi trường của bạn.

So right here we see that this is the classical data center we have in the beginning when we don't have​‌Như vậy, ngay tại đây chúng ta thấy đây là trung tâm dữ liệu truyền thống mà chúng ta có từ thuở ban đầu khi chưa có...

any new technology about the hypervisor and the SDS technology.​‌Bất kỳ công nghệ mới nào liên quan đến hypervisor và công nghệ SDS.

So right here you see that we have server, the classical server when we get our physical server.​‌Như bạn thấy ở đây, chúng ta có máy chủ, máy chủ vật lý truyền thống.

So this is a physical server.​‌Đây là một máy chủ vật lý.

Every server in a datacenter will be colored as a node.​‌Mỗi máy chủ trong trung tâm dữ liệu sẽ được tô màu như một nút.

We have no one, No.​‌Chúng tôi không có ai cả, không.

Two or three or not A, not B, nor the C, etc..​‌Hai hoặc ba, hoặc không phải A, không phải B, cũng không phải C, v.v...

So right here you see that in the beginning of the data center we get a node, a physical server or​‌Như bạn thấy ở đây, ngay từ đầu trung tâm dữ liệu, chúng ta có một nút, một máy chủ vật lý hoặc...

a physical host, a metal bar that's half the x86 architecture.​‌một máy chủ vật lý, một thanh kim loại chiếm một nửa kiến ​​trúc x86.

That's mean from Intel, the technology we installed the application inside it and we have the hard​‌Điều đó có nghĩa là từ Intel, công nghệ mà chúng tôi đã cài đặt ứng dụng bên trong nó và chúng tôi có phần cứng

disk inside it.​‌Đĩa bên trong nó.

But the mean, the application and the data will be switched into local artist.​‌Nhưng phương thức, ứng dụng và dữ liệu sẽ được chuyển sang cho nghệ sĩ địa phương.

So this is the DAS technology.​‌Đây chính là công nghệ DAS.

So right here every server will have its own hard disk and will communicate to other server throughout​‌Như vậy, tại đây mỗi máy chủ sẽ có ổ cứng riêng và sẽ giao tiếp với các máy chủ khác thông qua hệ thống.

the network and every one server would have only one application.​‌Mạng lưới và mỗi máy chủ sẽ chỉ có một ứng dụng duy nhất.

Of course, some application that works together in the same server.​‌Dĩ nhiên, một số ứng dụng hoạt động cùng nhau trên cùng một máy chủ.

You can have it, but lots of other application can't be running in the same server.​‌Bạn có thể cài đặt nó, nhưng nhiều ứng dụng khác không thể chạy đồng thời trên cùng một máy chủ.

So right here we have for example, these three server, every server have one application or its own​‌Ví dụ, ở đây chúng ta có ba máy chủ, mỗi máy chủ đều có một ứng dụng riêng.

application and have its own local hard disk.​‌ứng dụng này có ổ cứng cục bộ riêng.

So right here.​‌Ngay tại đây.

The new technology when we created the sun and doughnut storage, that's mean we use external storage​‌Công nghệ mới khi chúng tôi tạo ra hệ thống lưu trữ hình mặt trời và hình bánh rán, có nghĩa là chúng tôi sử dụng bộ nhớ ngoài.

or the expression shared storage.​‌hoặc cụm từ "bộ nhớ dùng chung".

That's mean a storage that is shared between multiple servers.​‌Điều đó có nghĩa là bộ nhớ được chia sẻ giữa nhiều máy chủ.

We use the shared storage and the controller of the storage or the controller of the hard disk.​‌Chúng tôi sử dụng bộ nhớ dùng chung và bộ điều khiển bộ nhớ hoặc bộ điều khiển ổ cứng.

That's mean the control of the input output per second.​‌Điều đó có nghĩa là kiểm soát tốc độ đầu vào/đầu ra mỗi giây.

So right here we add some external storage that have at least two controller to manage the storage and​‌Vậy nên ngay tại đây chúng ta sẽ thêm một số thiết bị lưu trữ ngoài có ít nhất hai bộ điều khiển để quản lý thiết bị lưu trữ đó.

we have a network connection with the different server.​‌Chúng tôi có kết nối mạng với máy chủ khác.

That's mean.​‌Thật là độc ác.

Every server we have a storage outside of the physical server by using the shared storage, by using​‌Mỗi máy chủ của chúng tôi đều có bộ nhớ lưu trữ bên ngoài máy chủ vật lý bằng cách sử dụng bộ nhớ dùng chung, bằng cách sử dụng

the different technology.​‌Công nghệ khác nhau.

So right here you see that we have a high performance.​‌Như vậy, ngay tại đây bạn có thể thấy chúng ta có hiệu suất cao.

So no, we have a high performance, but we have only three physical servers.​‌Vậy nên, không, chúng tôi có hiệu năng cao, nhưng chỉ có ba máy chủ vật lý.

Of course, if you are for server, for physical server, you will have only four server.​‌Dĩ nhiên, nếu bạn dùng máy chủ vật lý, bạn sẽ chỉ có bốn máy chủ.

As a server, you have only three server because we have three node or three physical hosts.​‌Về mặt máy chủ, bạn chỉ có ba máy chủ vì chúng ta có ba nút hoặc ba máy chủ vật lý.

And the performance is high because we have server that use its own CPU and memory and we have external​‌Và hiệu năng cao là vì chúng tôi có máy chủ sử dụng CPU và bộ nhớ riêng, cùng với các thiết bị ngoại vi.

hard disk that give you a high input output.​‌Ổ cứng cung cấp tốc độ đầu vào/đầu ra cao.

So you can manage your environment by using this technology.​‌Vì vậy, bạn có thể quản lý môi trường của mình bằng cách sử dụng công nghệ này.

Then you see right here, if we start right and read data from the hard disk, you see that the input​‌Sau đó, bạn có thể thấy ngay ở đây, nếu chúng ta bắt đầu từ bên phải và đọc dữ liệu từ ổ cứng, bạn sẽ thấy rằng đầu vào

outputs of the.​‌đầu ra của.

Controller will be separately.​‌Bộ điều khiển sẽ được bán riêng.

As the same for the different servers.​‌Điều này cũng áp dụng tương tự cho các máy chủ khác nhau.

Let's imagine that this controller have a speed of 1100 IOPS.​‌Giả sử bộ điều khiển này có tốc độ 1100 IOPS.

As a speed of the input output per second, every server will take at least 33,000 inputs outputs per​‌Với tốc độ nhập/xuất dữ liệu mỗi giây, mỗi máy chủ sẽ xử lý ít nhất 33.000 lượt nhập/xuất dữ liệu mỗi giây.

second to get access to the data with this height speed.​‌thứ hai là có thể truy cập dữ liệu với tốc độ cao như vậy.

So right here you'll see that the performance is good.​‌Như bạn thấy ở đây, hiệu năng rất tốt.

Every cell will have a good value of the IOPS and the data will be go between the server and the storage.​‌Mỗi ô sẽ có giá trị IOPS tốt và dữ liệu sẽ được truyền giữa máy chủ và thiết bị lưu trữ.

So next, when we have the new technology from Intel that leads you to virtually is the CPU, the intel.​‌Vậy tiếp theo, khi chúng ta có công nghệ mới từ Intel, thứ sẽ dẫn bạn đến với CPU, tức là bộ xử lý Intel.

Give us a new solution of the CPU virtualization that mean you can use one CPU to create multiple virtual​‌Hãy cung cấp cho chúng tôi một giải pháp mới về ảo hóa CPU, cho phép sử dụng một CPU để tạo ra nhiều máy ảo.

CPA.​‌Kế toán viên được chứng nhận (CPA).

This technology leads us to use the new hypervisor technology.​‌Công nghệ này dẫn chúng ta đến việc sử dụng công nghệ ảo hóa mới.

This hypervisor will let you to use the CPU as a multiple virtual CPU.​‌Trình ảo hóa này cho phép bạn sử dụng CPU như một CPU ảo đa luồng.

This technology will help us a lot to upgrade our environment by using.​‌Công nghệ này sẽ giúp chúng ta rất nhiều trong việc nâng cấp môi trường bằng cách sử dụng nó.

For example, in this example, just three physical hosts.​‌Ví dụ, trong ví dụ này, chỉ có ba máy chủ vật lý.

Before the hypervisor, we have only three applications inside this, but by using the hypervisor now,​‌Trước khi sử dụng hypervisor, chúng ta chỉ có ba ứng dụng bên trong, nhưng giờ đây, nhờ sử dụng hypervisor,

we can create multiple applications inside a multiple virtual machine in the same server.​‌Chúng ta có thể tạo nhiều ứng dụng bên trong nhiều máy ảo trên cùng một máy chủ.

So you see right here in the old technology, we have only three physical hosts.​‌Như bạn thấy đấy, với công nghệ cũ này, chúng ta chỉ có ba máy chủ vật lý.

We have three server.​‌Chúng tôi có ba máy chủ.

But now by using the hypervisor, we have a big number of the server or the application, but using​‌Nhưng giờ đây, bằng cách sử dụng hypervisor, chúng ta có một số lượng lớn máy chủ hoặc ứng dụng, nhưng việc sử dụng

the same hardware.​‌Cùng một phần cứng.

So right here, one, two, three physical hosts.​‌Vậy là ngay đây, một, hai, ba máy chủ vật lý.

But we have, for example, four, eight, 12 virtual machine, that's man 12 server, that's mean 12​‌Nhưng chúng ta có, ví dụ, bốn, tám, 12 máy ảo, tức là 12 máy chủ, nghĩa là 12

application in our environment.​‌ứng dụng trong môi trường của chúng ta.

So instead of buying another nine physical server to be able to create this environment, we only use​‌Vì vậy, thay vì mua thêm chín máy chủ vật lý để tạo ra môi trường này, chúng ta chỉ cần sử dụng...

the hypervisor technology to create multiple server or multiple virtual machine by the same hardware.​‌Công nghệ hypervisor cho phép tạo ra nhiều máy chủ hoặc nhiều máy ảo bằng cùng một phần cứng.

So this is the benefit of using the hypervisor.​‌Vậy đây chính là lợi ích của việc sử dụng hypervisor.

But if you check right here, you see that we have a big number of server by using the same hardware,​‌Nhưng nếu bạn kiểm tra ngay tại đây, bạn sẽ thấy chúng tôi có rất nhiều máy chủ sử dụng cùng một phần cứng.

but the performance becomes low, the performance is degraded.​‌nhưng hiệu năng trở nên thấp, bị suy giảm.

In this situation right here, you see that we have less performance because the same IOPS will be separated​‌Trong trường hợp này, bạn thấy rằng hiệu năng thấp hơn vì cùng một IOPS sẽ bị phân tách.

on the different virtual machine.​‌trên các máy ảo khác nhau.

That's mean.​‌Thật là độc ác.

We have for example, 12 server right here will consume the same speed separately between them.​‌Ví dụ, chúng ta có 12 máy chủ ở đây, mỗi máy chủ sẽ tiêu thụ cùng một tốc độ mạng riêng biệt.

So this is the problem that we start getting when we start using this technology of the NAS and Sun.​‌Đây chính là vấn đề mà chúng ta bắt đầu gặp phải khi sử dụng công nghệ NAS và Sun.

So right here you see that if we add another physical server on your power metal server as a new node​‌Như bạn thấy ở đây, nếu chúng ta thêm một máy chủ vật lý khác vào máy chủ Power Metal của bạn dưới dạng một node mới, thì đó là một nút mới.

and use the hypervisor, we have new virtual machine that's mean new server in our environment.​‌Và bằng cách sử dụng hypervisor, chúng ta có máy ảo mới, điều đó có nghĩa là máy chủ mới trong môi trường của chúng ta.

So right here the number of the server become more, but the performance is become less because we consume​‌Vì vậy, ở đây số lượng máy chủ tăng lên, nhưng hiệu suất lại giảm xuống vì chúng ta tiêu thụ nhiều hơn.

the same IOPS between the different physical servers.​‌Chỉ số IOPS giống nhau giữa các máy chủ vật lý khác nhau.

So instead of 33,000 IOPS by a server, no, we have 25,000 IOPS by server and every server will separate​‌Vậy thay vì 33.000 IOPS trên mỗi máy chủ, chúng ta chỉ có 25.000 IOPS trên mỗi máy chủ và mỗi máy chủ sẽ hoạt động riêng biệt.

this 25,000 IOPS for the different virtual machine by users of every virtual machine.​‌Mức này là 25.000 IOPS cho mỗi máy ảo khác nhau, tùy thuộc vào người dùng của từng máy ảo.

So right here, you see, this is the problem that we start getting.​‌Như bạn thấy đấy, đây chính là vấn đề mà chúng ta bắt đầu gặp phải.

Of course, the new technology give you the SSD more faster, you need to buy more powerful controller,​‌Dĩ nhiên, công nghệ mới giúp SSD hoạt động nhanh hơn, nhưng bạn cũng cần mua bộ điều khiển mạnh hơn.

etc. But every time when you try to scale up your environment, try to upgrade your environment or scale​‌vân vân. Nhưng mỗi khi bạn cố gắng mở rộng quy mô môi trường của mình, hãy thử nâng cấp môi trường hoặc mở rộng quy mô.

up server or your virtual machine, you will have a degraded infrastructure or a degraded performance.​‌Khi khởi động máy chủ hoặc máy ảo, bạn sẽ gặp phải tình trạng suy giảm cơ sở hạ tầng hoặc hiệu năng.

And the same if you have a hard disk filer, if you have, for example, this hard disk, file it in​‌Tương tự nếu bạn có một ổ cứng lưu trữ, ví dụ như ổ cứng này, hãy lưu trữ nó trong...

your external storage, the control will be consume it to.​‌Bộ nhớ ngoài của bạn sẽ tiêu thụ dung lượng đó.

Back up your hard disk in other hard disk available.​‌Sao lưu dữ liệu trên ổ cứng của bạn vào một ổ cứng khác.

So you see right here the overloading of your IOPS controller in your environment.​‌Như bạn thấy ở đây, bộ điều khiển IOPS đang bị quá tải trong môi trường của bạn.

So this is a problem.​‌Vậy đây là một vấn đề.

If you already pass this problem in your environment and you have a file or hard disk, you will see​‌Nếu bạn đã gặp phải vấn đề này trong môi trường của mình và bạn có tệp tin hoặc ổ cứng, bạn sẽ thấy

that will have a very, very low storage network in your environment.​‌Điều đó sẽ dẫn đến dung lượng lưu trữ mạng rất, rất thấp trong môi trường của bạn.

And this is a very big problem.​‌Và đây là một vấn đề rất lớn.

So this is another problem that we get from using the external storage, the Sun and NAS technology.​‌Đây là một vấn đề khác mà chúng ta gặp phải khi sử dụng thiết bị lưu trữ ngoài, công nghệ Sun và NAS.

So next we'll see more about the data center and explain more the benefit that we can get from the Nutanix​‌Tiếp theo, chúng ta sẽ tìm hiểu thêm về trung tâm dữ liệu và giải thích rõ hơn về những lợi ích mà chúng ta có thể nhận được từ Nutanix.

solution. ​‌giải pháp.
