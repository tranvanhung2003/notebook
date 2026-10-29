Hello again.​‌Chào lại.

So, no, we're going to talk about the Nutanix architecture.​‌Vậy nên, không, chúng ta sẽ nói về kiến ​​trúc của Nutanix.

That's mean.​‌Thật là độc ác.

If you want to see that you have a Nutanix cluster in your environment, you should have at least three​‌Nếu bạn muốn kiểm tra xem mình có cụm Nutanix trong môi trường của mình hay không, bạn cần có ít nhất ba cụm.

physical hosts that mean three nerd in the physical server.​‌Các máy chủ vật lý có nghĩa là ba chuyên gia kỹ thuật trong máy chủ vật lý.

We have block and block have, for example, four nerd or two nerd or one nerd.​‌Chúng ta có các khối, ví dụ như bốn mọt sách, hai mọt sách hoặc một mọt sách.

When we see a nerd, that means a one physical server.​‌Khi chúng ta thấy một người mê công nghệ, điều đó có nghĩa là người đó chỉ sở hữu một máy chủ vật lý duy nhất.

So we have block that have four nerd.​‌Vậy là chúng ta có một khối gồm bốn người mọt sách.

You should have at least three nerd in your environment.​‌Trong môi trường của bạn nên có ít nhất ba người đam mê công nghệ.

That's using the Nutanix solution to be able to see that you have a Nutanix cluster.​‌Điều đó có nghĩa là sử dụng giải pháp Nutanix để có thể biết rằng bạn đang có một cụm Nutanix.

If you don't have more than three nerd or three physical server, you can't see that you have a Nutanix​‌Nếu bạn không có nhiều hơn ba chuyên gia công nghệ hoặc ba máy chủ vật lý, bạn sẽ không thể biết mình đang sở hữu một thiết bị Nutanix.

solution and you can't get the full benefit of the Nutanix software and Nutanix solution.​‌Nếu không có giải pháp này, bạn sẽ không thể tận dụng tối đa lợi ích của phần mềm Nutanix và giải pháp Nutanix.

So this is the basic architecture for every node.​‌Đây là kiến ​​trúc cơ bản cho mỗi nút.

When you have at least three physical hosts, every host of the three or even more physical hosts,​‌Khi bạn có ít nhất ba máy chủ vật lý, mỗi máy chủ trong số ba hoặc thậm chí nhiều hơn các máy chủ vật lý đó,

every physical host or every node composite from this infrastructure or this architecture.​‌Mỗi máy chủ vật lý hoặc mỗi nút cấu thành từ cơ sở hạ tầng hoặc kiến ​​trúc này.

So right here, you see this is our physical server.​‌Như các bạn thấy ở đây, đây là máy chủ vật lý của chúng tôi.

This is the bare metal server or the node that half the hardware, the CPU, the memory and the HP controller.​‌Đây là máy chủ vật lý hoặc nút mạng chỉ bao gồm một nửa phần cứng, gồm CPU, bộ nhớ và bộ điều khiển HP.

That's mean the hardware that control the storage in your environment.​‌Điều đó có nghĩa là phần cứng điều khiển việc lưu trữ trong môi trường của bạn.

Then we have do storage, do hybrid storage.​‌Tiếp theo, chúng ta có lưu trữ, lưu trữ lai.

This example right here have a hybrid storage that's composited from SSD and HDD.​‌Ví dụ này minh họa một hệ thống lưu trữ lai kết hợp giữa SSD và HDD.

So in this situation, we use this hybrid format to store the data inside it.​‌Vì vậy, trong trường hợp này, chúng ta sử dụng định dạng lai này để lưu trữ dữ liệu bên trong.

So the Nutanix solution will use the SSD for caching and do HDD for the storage or the data saved.​‌Vì vậy, giải pháp của Nutanix sẽ sử dụng SSD để làm bộ nhớ đệm và HDD để lưu trữ dữ liệu.

Also, if you have a flash infrastructure, that's when you use all the storage from the form of SSD.​‌Ngoài ra, nếu bạn có cơ sở hạ tầng flash, đó là lúc bạn sử dụng toàn bộ dung lượng lưu trữ từ ổ SSD.

It will be good and faster.​‌Nó sẽ tốt hơn và nhanh hơn.

Then the Nutanix will use them for the flushing and the storage together.​‌Sau đó, Nutanix sẽ sử dụng chúng cho cả việc xả và lưu trữ.

But in this example right here, the most famous one is the hybrid version, because it is very expensive​‌Nhưng trong ví dụ này, phiên bản nổi tiếng nhất là phiên bản lai, bởi vì nó rất đắt tiền.

to have all flash a hard disk in your environment.​‌để có tất cả ổ cứng flash trong môi trường của bạn.

So the most fun, most we find that we have the HDD, we use it for the storage and the SSD, we use​‌Vậy nên điều thú vị nhất, điều mà chúng ta thấy là chúng ta có ổ cứng HDD, chúng ta dùng nó để lưu trữ và ổ SSD, chúng ta sử dụng

it for the flash.​‌Nó dành cho đèn flash.

Of course the Nutanix VM will use them separately, one for the kitchen and one for the storage.​‌Dĩ nhiên, máy ảo Nutanix sẽ sử dụng chúng riêng biệt, một cho nhà bếp và một cho kho lưu trữ.

So this is the hardware layer.​‌Đây là lớp phần cứng.

You have the parameters that have the CPU, the memory, the HP, the information about the hardware​‌Bạn có các thông số bao gồm CPU, bộ nhớ, HP và thông tin về phần cứng.

right here, the views, etc. graphical interface, etc. Are we going to see a real example of our Nutanix​‌Ngay đây, các khung nhìn, giao diện đồ họa, v.v. Chúng ta sẽ được xem một ví dụ thực tế về Nutanix chứ?

hardware and we'll see this information.​‌phần cứng và chúng ta sẽ xem thông tin này.

Then we have the storage connected directly to the server, the DAS Technology, direct attached storage.​‌Tiếp theo là hệ thống lưu trữ được kết nối trực tiếp với máy chủ, gọi là công nghệ DAS (Direct Attached Storage).

Then the next layer we have app of the hardware is the hypervisor.​‌Lớp tiếp theo của phần cứng chính là hypervisor.

Right here we use the hypervisor, the one that's come from Nutanix, Scarlett, ESV, the Native Hypervisor​‌Ở đây chúng ta sử dụng hypervisor, hypervisor đến từ Nutanix, Scarlett, ESV, Native Hypervisor.

from Nutanix, or you can use the other hypervisor available, for example, the VMware E6 II or the​‌Từ Nutanix, hoặc bạn có thể sử dụng trình ảo hóa khác có sẵn, ví dụ như VMware E6 II hoặc...

Hyper-V or the VM, etc..​‌Hyper-V hoặc máy ảo, v.v...

So you can use any kind of hypervisor, but if you prefer to use the Nutanix native hypervisor, you​‌Vì vậy, bạn có thể sử dụng bất kỳ loại hypervisor nào, nhưng nếu bạn muốn sử dụng hypervisor gốc của Nutanix, bạn

can use it as the same as the other hypervisor.​‌Có thể sử dụng nó giống như các hypervisor khác.

Then in there, this layer of the hypervisor, we have the software of Nutanix that collects Acropolis​‌Sau đó, trong lớp hypervisor này, chúng ta có phần mềm của Nutanix thu thập Acropolis.

operating system.​‌hệ điều hành.

We added to the hypervisor that we're going to handle the storage of the physical server.​‌Chúng tôi đã bổ sung vào hypervisor chức năng quản lý lưu trữ của máy chủ vật lý.

The storage will be handled and managed by the Acropolis operating system that we're going to use it​‌Việc lưu trữ sẽ được xử lý và quản lý bởi hệ điều hành Acropolis mà chúng ta sẽ sử dụng.

from the Nutanix solution.​‌Từ giải pháp của Nutanix.

Then on top of this layer, we have the KVM, a virtual machine as a controller.​‌Trên lớp này, chúng ta có KVM, một máy ảo đóng vai trò là bộ điều khiển.

So in the preview architecture of the datacenter, we have a controller as a hardware.​‌Như vậy, trong kiến ​​trúc xem trước của trung tâm dữ liệu, chúng ta có bộ điều khiển dưới dạng phần cứng.

Now we have the controller of the storage as our software right here will handle the storage through​‌Giờ đây chúng ta đã có bộ điều khiển lưu trữ vì phần mềm của chúng ta sẽ xử lý việc lưu trữ thông qua nó.

this software instead of the hardware that we use it in the stand and do nothing storage.​‌phần mềm này thay thế phần cứng mà chúng ta sử dụng trong giá đỡ và không làm gì để lưu trữ.

Then we have the application inside this hypervisor, the virtual machine that we need in our environment​‌Tiếp theo, chúng ta có ứng dụng bên trong hypervisor này, tức là máy ảo mà chúng ta cần trong môi trường của mình.

to create application necessary of the company.​‌Tạo ra các ứng dụng cần thiết cho công ty.

So this is the architecture, the basic architecture that we should have in every node to be able to​‌Đây là kiến ​​trúc, kiến ​​trúc cơ bản mà chúng ta cần có trong mỗi nút để có thể...

see that this is not an environment or an architecture.​‌Hãy hiểu rằng đây không phải là một môi trường hay một kiến ​​trúc.

So right here, this is the hardware layer, the storage layer and the hypervisor layer up of the hardware​‌Như vậy, ở đây, đây là lớp phần cứng, lớp lưu trữ và lớp ảo hóa nằm trên phần cứng.

inside the hypervisor or between the hypervisor and the hardware.​‌Bên trong hypervisor hoặc giữa hypervisor và phần cứng.

We have a software solution from the Nutanix that's called the Acropolis that will lead you to convert​‌Chúng tôi có một giải pháp phần mềm từ Nutanix có tên là Acropolis, sẽ giúp bạn chuyển đổi.

your physical server to a hyper converged infrastructure.​‌Chuyển đổi máy chủ vật lý của bạn thành cơ sở hạ tầng siêu hội tụ.

That's mean will lead you to convert your storage as a distributed storage fabric.​‌Điều đó có nghĩa là bạn sẽ phải chuyển đổi hệ thống lưu trữ của mình thành một kiến ​​trúc lưu trữ phân tán.

And then we have the KVM.​‌Và sau đó chúng ta có KVM.

We should have only one VM in every note to be able to see that this node or this physical server handle​‌Chúng ta chỉ nên có một máy ảo (VM) trong mỗi nút để có thể thấy được nút này hoặc máy chủ vật lý này đang xử lý điều gì.

it.​‌Nó.

Rock the Nutanix Acropolis system operation. ​‌Hãy vận hành hệ thống Nutanix Acropolis một cách hiệu quả.
