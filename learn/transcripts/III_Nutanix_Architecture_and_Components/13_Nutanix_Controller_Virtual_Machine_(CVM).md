Hello again.​‌Chào lại.

So now we're going to talk about the severe component, the components of the controller virtual machine​‌Vậy bây giờ chúng ta sẽ nói về thành phần quan trọng, các thành phần của máy ảo điều khiển.

or the service of the controller virtual machine.​‌hoặc dịch vụ của máy ảo điều khiển.

So right here you see that every virtual machine connected to the hypervisor, that's mean.​‌Như bạn thấy ở đây, mọi máy ảo đều được kết nối với hypervisor, điều đó có nghĩa là...

We have a virtual switch inside the hypervisor that will have connection between the virtual machine​‌Chúng tôi có một bộ chuyển mạch ảo bên trong hypervisor sẽ thiết lập kết nối giữa các máy ảo.

and the environment and the other network.​‌và môi trường cũng như các mạng lưới khác.

So right here you see that if we.​‌Như vậy, ngay tại đây bạn thấy rằng nếu chúng ta...

Sent an input output from this virtual machine to the storage, for example, or to the client that​‌Đã gửi dữ liệu đầu vào/đầu ra từ máy ảo này đến thiết bị lưu trữ, ví dụ, hoặc đến máy khách.

ask for a specific information that exists in the data from the storage, how the virtual machine communicates​‌Yêu cầu thông tin cụ thể có trong dữ liệu từ bộ nhớ lưu trữ, cách thức máy ảo giao tiếp.

with the storage.​‌với dung lượng lưu trữ.

So the virtual machine right here will request the data through the virtual network with the virtual​‌Vì vậy, máy ảo ở đây sẽ yêu cầu dữ liệu thông qua mạng ảo với máy ảo.

switch that we have in the hypervisor for, for example, in the Nutanix hypervisor will have the open​‌Công tắc mà chúng ta có trong hypervisor, ví dụ như trong hypervisor Nutanix, sẽ có trạng thái mở.

virtual switch in the A6 AI.​‌Công tắc ảo trên A6 AI.

We have the virtual standard switch or virtual distributed switch, etc. Then this virtual switch will​‌Chúng ta có bộ chuyển mạch tiêu chuẩn ảo hoặc bộ chuyển mạch phân tán ảo, v.v. Khi đó, bộ chuyển mạch ảo này sẽ

send the data or the request to the server instead of the local disk.​‌Gửi dữ liệu hoặc yêu cầu đến máy chủ thay vì lưu vào ổ đĩa cục bộ.

So any request of the data from the virtual machine will go to the server, the controller, virtual​‌Vì vậy, bất kỳ yêu cầu dữ liệu nào từ máy ảo sẽ được gửi đến máy chủ, bộ điều khiển, máy ảo.

machine that will handle the input and output of the environment.​‌Máy móc sẽ xử lý đầu vào và đầu ra của môi trường.

So then this VM will send the request to the HP, the hardware, and that is go directly to the hard​‌Vì vậy, máy ảo này sẽ gửi yêu cầu đến HP, đến phần cứng, và yêu cầu đó sẽ được gửi trực tiếp đến phần cứng.

disk.​‌đĩa.

For example, right here it will go from the virtual machine through the virtual switch to the civil​‌Ví dụ, ngay tại đây, nó sẽ đi từ máy ảo thông qua bộ chuyển mạch ảo đến hệ thống dân dụng.

and KVM will send it to the hard disk to get the specific data necessary requested by this virtual machine.​‌Và KVM sẽ gửi dữ liệu đó đến ổ cứng để lấy các dữ liệu cụ thể cần thiết mà máy ảo này yêu cầu.

So the same if, for example, this virtual machine wants to get a data from another storage that exist​‌Tương tự như vậy, ví dụ, nếu máy ảo này muốn lấy dữ liệu từ một kho lưu trữ khác đã tồn tại.

in another node, the same process will go to the virtual switch.​‌Tại một nút khác, quy trình tương tự sẽ được thực hiện trên bộ chuyển mạch ảo.

The switch will send it to the CVA.​‌Công tắc sẽ gửi nó đến CVA.

The server will try to contact the other event through the enter event to get the specific data necessary​‌Máy chủ sẽ cố gắng liên hệ với sự kiện khác thông qua sự kiện enter để lấy dữ liệu cụ thể cần thiết.

that exist in another node.​‌tồn tại ở một nút khác.

So right here the next VM will send it to the HP and get the data from this storage if it exists in​‌Vì vậy, ngay tại đây, máy ảo tiếp theo sẽ gửi nó đến HP và lấy dữ liệu từ bộ nhớ này nếu nó tồn tại.

a storage separated to the node of the VM. ​‌một bộ nhớ riêng biệt cho từng nút của máy ảo.
