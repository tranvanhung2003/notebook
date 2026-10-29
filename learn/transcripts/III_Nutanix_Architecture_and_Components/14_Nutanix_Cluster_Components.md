So now we want to understand the different components we have in the survey.​‌Vậy bây giờ chúng ta cần hiểu rõ các thành phần khác nhau có trong bản khảo sát này.

And here a list of the components we have in the survey.​‌Và đây là danh sách các thành phần trong khảo sát của chúng tôi.

Every survey have a specific component that we use it to manage our environment.​‌Mỗi cuộc khảo sát đều có một thành phần cụ thể mà chúng ta sử dụng để quản lý môi trường của mình.

So right here, this is some name.​‌Đây quả là một cái tên hay.

It will help you to get to the big idea of how cluster components work, but it's not necessary to memorize​‌Nó sẽ giúp bạn nắm được ý tưởng tổng quan về cách thức hoạt động của các thành phần cụm, nhưng không nhất thiết phải học thuộc lòng.

them together.​‌Họ ở cùng nhau.

It's only it will help you if you have some trouble shooting in a specific tank.​‌Nó chỉ hữu ích nếu bạn gặp khó khăn khi bắn vào một loại xe tăng cụ thể.

You can easily know which components handle this problem that you have in your environment.​‌Bạn có thể dễ dàng biết được các thành phần nào xử lý vấn đề mà bạn đang gặp phải trong môi trường của mình.

So let's start with this simple architecture.​‌Vậy chúng ta hãy bắt đầu với kiến ​​trúc đơn giản này.

Of course, there are some other complicated architecture.​‌Dĩ nhiên, còn có một số kiểu kiến ​​trúc phức tạp khác.

I use this simple one that will help you to understand the environment.​‌Tôi sử dụng một phương pháp đơn giản giúp bạn hiểu rõ hơn về môi trường.

So let's start with the browser right here.​‌Vậy chúng ta hãy bắt đầu với trình duyệt ngay tại đây.

That's mean the browser or the user that will use the browser or the orchestration to connect to the​‌Điều đó có nghĩa là trình duyệt hoặc người dùng sẽ sử dụng trình duyệt hoặc hệ thống điều phối để kết nối với...

cluster environment, the Nutanix cluster environment through the browser to send a specific request​‌Môi trường cụm, môi trường cụm Nutanix thông qua trình duyệt để gửi yêu cầu cụ thể.

or a specific task.​‌hoặc một nhiệm vụ cụ thể.

So, for example, he will go from the prison, the zookeeper, the curator, the Cassandra, the target​‌Ví dụ, anh ta sẽ lần lượt xuất hiện trong nhà tù, rồi người trông coi sở thú, người phụ trách bảo tàng, Cassandra, và cuối cùng là mục tiêu.

to get you the hypervisor of the server.​‌Để lấy cho bạn trình ảo hóa của máy chủ.

So let's try to explain them one by one and understand every component and its function so everyone​‌Vậy chúng ta hãy cùng giải thích từng phần một và hiểu rõ từng thành phần cũng như chức năng của nó để mọi người đều hiểu.

does it.​‌Đúng vậy.

What are some of the main service components that make up a Nutanix?​‌Nutanix bao gồm những thành phần dịch vụ chính nào?

So the following is a simplified view of the main Nutanix cluster components.​‌Dưới đây là cái nhìn tổng quan đơn giản về các thành phần chính của cụm máy chủ Nutanix.

All components that run on multiple nodes in the cluster and depend on connectivity between the pair​‌Tất cả các thành phần chạy trên nhiều nút trong cụm và phụ thuộc vào khả năng kết nối giữa các cặp nút đó.

that also run the same components.​‌những thứ cũng sử dụng các thành phần tương tự.

So this is the different components we have in every VM.​‌Đây là các thành phần khác nhau mà chúng ta có trong mỗi máy ảo.

So the first component we have is the Stargate.​‌Vậy thành phần đầu tiên chúng ta có là Cổng Sao.

So the Stargate role is the data input output manager.​‌Vậy vai trò của Stargate là quản lý đầu vào và đầu ra dữ liệu.

So what is the Stargate is a distributed system that present storage to other system, such as a hypervisor,​‌Vậy Stargate là gì? Nó là một hệ thống phân tán cung cấp không gian lưu trữ cho các hệ thống khác, chẳng hạn như hypervisor.

needs a unified component for receiving and processing data that is received.​‌Cần một thành phần thống nhất để tiếp nhận và xử lý dữ liệu nhận được.

The Nutanix cluster has a large software component called Stargate that manage this responsibility.​‌Cụm máy chủ Nutanix có một thành phần phần mềm lớn gọi là Stargate đảm nhiệm vai trò này.

So from the perspective of the hypervisor, Stargate is the main point of contact for the Nutanix cluster.​‌Vì vậy, từ góc nhìn của hypervisor, Stargate là điểm liên lạc chính cho cụm máy chủ Nutanix.

Our read and write requests are sent across the switch.​‌Các yêu cầu đọc và ghi của chúng ta được gửi qua thiết bị chuyển mạch.

Nutanix Lets mean the open virtual switch to the Stargate process running on that node.​‌Nutanix, ý tôi là mở một switch ảo cho tiến trình Stargate đang chạy trên node đó.

So the Stargate depend on Medusa to gather metadata and zeros to gather cluster configuration.​‌Vì vậy, Cổng Sao phụ thuộc vào Medusa để thu thập siêu dữ liệu và các số 0 để thu thập cấu hình cụm.

That means the Medusa and Zeus is an access interface to communicate with the other components.​‌Điều đó có nghĩa là Medusa và Zeus là giao diện truy cập để giao tiếp với các thành phần khác.

So Target is responsible for all data management and inputs, outputs, operation and is the main interface​‌Như vậy, Target chịu trách nhiệm quản lý toàn bộ dữ liệu, đầu vào, đầu ra, hoạt động và là giao diện chính.

from the hypervisor via the NFS iSCSI or SMB protocol.​‌Từ máy chủ ảo hóa thông qua giao thức NFS iSCSI hoặc SMB.

This service ran on every node in the cluster in order to serve localized inputs output.​‌Dịch vụ này chạy trên mọi nút trong cụm máy chủ để cung cấp đầu vào và đầu ra cục bộ.

Then we have the Cassandra.​‌Tiếp theo là Cassandra.

The role of Cassandra is the distributed metadata store, so the Cassandra stores and manage all of​‌Vai trò của Cassandra là kho lưu trữ siêu dữ liệu phân tán, do đó Cassandra lưu trữ và quản lý tất cả các siêu dữ liệu đó.

the cluster metadata in a distributed ring like manner based upon a heavy modified Apache Cassandra.​‌Siêu dữ liệu của cụm được phân tán theo dạng vòng dựa trên Apache Cassandra đã được sửa đổi nhiều.

The Paxos algorithm is designed to enforce strict consistency.​‌Thuật toán Paxos được thiết kế để đảm bảo tính nhất quán nghiêm ngặt.

This service run on every node in the cluster.​‌Dịch vụ này chạy trên mọi nút trong cụm máy chủ.

The Cassandra is accessed via an interface.​‌Cassandra được truy cập thông qua một giao diện.

Scarlett Medusa and Cassandra depend on Zeus to gather information about the cluster information.​‌Scarlett, Medusa và Cassandra đều phụ thuộc vào Zeus để thu thập thông tin về cụm thiên hà.

So between the Cassandra and the Stargate, we have access interface that let them communicate to each​‌Như vậy, giữa Cassandra và Stargate, chúng ta có một giao diện truy cập cho phép chúng liên lạc với nhau.

other that Scarlett the Medusa.​‌ngoài Scarlett, nàng Medusa.

So the Medusa is the access interface for Cassandra.​‌Như vậy, Medusa là giao diện truy cập cho Cassandra.

Medusa is a distributed system that store data for other systems.​‌Medusa là một hệ thống phân tán lưu trữ dữ liệu cho các hệ thống khác.

For example, a hypervisor that hosts a virtual machine must have a way to keep track of where that​‌Ví dụ, một hypervisor lưu trữ máy ảo cần có cách để theo dõi vị trí của máy ảo đó.

data is, in that case, of a Nutanix cluster.​‌Trong trường hợp đó, dữ liệu là của một cụm Nutanix.

It is also important to track where the replicates of the data are storage.​‌Việc theo dõi vị trí lưu trữ các bản sao dữ liệu cũng rất quan trọng.

So Medusa is a Nutanix abstraction layer that sits in front of the database that hold this meta data.​‌Như vậy, Medusa là một lớp trừu tượng của Nutanix nằm phía trước cơ sở dữ liệu lưu trữ siêu dữ liệu này.

The database is distributed across all node in the cluster using a modified from Apache Cassandra.​‌Cơ sở dữ liệu được phân tán trên tất cả các nút trong cụm bằng cách sử dụng một phiên bản được sửa đổi từ Apache Cassandra.

Then we have the curator.​‌Tiếp theo là người phụ trách bảo tàng.

The role of the curator is the MapReduce Cluster Management and cleanup.​‌Vai trò của người quản lý là quản lý và dọn dẹp cụm MapReduce.

So it is responsible for managing and distributing tasks throughout the cluster, including disk balancing,​‌Vì vậy, nó chịu trách nhiệm quản lý và phân phối các tác vụ trong toàn bộ cụm máy chủ, bao gồm cả việc cân bằng tải đĩa.

proactive scrubbing and many more items.​‌Vệ sinh chủ động và nhiều hạng mục khác nữa.

So the curator run on every node and is controlled by an elected curator later who is responsible for​‌Vì vậy, trình quản lý được vận hành trên mọi nút và sau đó được điều khiển bởi một người quản lý được bầu chọn, người chịu trách nhiệm về...

the task and job delegation.​‌Phân công nhiệm vụ và công việc.

There are two scale type for curator a false scan which occurs around every 6 hours, and a partial​‌Có hai loại thang đo dành cho người quản lý: một loại quét sai xảy ra khoảng 6 giờ một lần, và một loại quét một phần.

scan which occurs every how.​‌Quá trình quét diễn ra thường xuyên.

So the curators depend on the Zeus to check nodes, availability and Medusa to gather data.​‌Vì vậy, những người quản lý phụ thuộc vào Zeus để kiểm tra các nút, tính khả dụng và Medusa để thu thập dữ liệu.

It's then send command to Stargate based on the information collected.​‌Sau đó, hệ thống sẽ gửi lệnh đến Stargate dựa trên thông tin thu thập được.

Then we have the zookeeper component is the cluster Configuration Manager, the Zookeeper store, all​‌Tiếp theo, chúng ta có thành phần Zookeeper, đó là Trình quản lý cấu hình cụm, kho lưu trữ Zookeeper, tất cả đều như vậy.

of the cluster configuration, including hosts, APIs, states, etc. and is based upon Apache Zookeeper.​‌Cấu hình cụm, bao gồm máy chủ, API, trạng thái, v.v., và dựa trên Apache Zookeeper.

This service run on three or five nodes, depending on the residency factor in the cluster.​‌Dịch vụ này chạy trên ba hoặc năm nút, tùy thuộc vào yếu tố cư trú trong cụm.

We're going to talk about the redundancy factor later.​‌Chúng ta sẽ bàn về hệ số dư thừa sau.

But to be able to see that you have a radius, you need to have at least three physical hosts or three​‌Nhưng để có thể thấy rằng bạn có một bán kính, bạn cần có ít nhất ba máy chủ vật lý hoặc ba

node.​‌nút.

If you have more than five, then you can have other factor.​‌Nếu bạn có nhiều hơn năm yếu tố, thì bạn có thể có thêm các yếu tố khác.

That's correct.​‌Đúng vậy.

RF three.​‌RF ba.

So one of which is elected as a leader.​‌Vì vậy, một trong số đó được bầu làm lãnh đạo.

That's mean.​‌Thật là độc ác.

It's depend on the redundancy factor in the cluster.​‌Điều đó phụ thuộc vào hệ số dự phòng trong cụm máy chủ.

In one off the note that is elected as a leader.​‌Trong một diễn biến bất ngờ, người đó được bầu làm lãnh đạo.

So you should know that in our environment, one of the seven will be elected as a leader to handle​‌Vì vậy, bạn nên biết rằng trong môi trường của chúng ta, một trong bảy người sẽ được bầu làm lãnh đạo để điều hành.

or to manage The cluster is the same when we use the EXI hosts TRUCK, the VMware vSphere environment,​‌hoặc để quản lý cụm máy chủ, quy trình cũng tương tự khi chúng ta sử dụng máy chủ EXI TRUCK, môi trường VMware vSphere.

we have one E6 II host leader and the other E6 II as a slave.​‌Chúng tôi có một E6 II làm máy chủ chính và một E6 II khác làm máy phụ.

So the same idea right here.​‌Vậy nên ý tưởng ở đây cũng tương tự.

The environment will elected once.​‌Môi trường sẽ được bầu chọn một lần.

If we aim to be a leader to handle and manage the cluster and the other one be a slave until something​‌Nếu một người muốn trở thành người lãnh đạo để điều hành và quản lý nhóm, còn người kia sẽ là nô lệ cho đến khi có điều gì đó xảy ra.

wrong happen to the VM, the leader, then the environment will elected a new civil MGB.​‌Nếu có điều gì đó không hay xảy ra với VM, người lãnh đạo, thì môi trường sẽ bầu ra một MGB dân sự mới.

Ah, leader in the environment.​‌À, người dẫn đầu trong lĩnh vực bảo vệ môi trường.

Of course we're going to talk about it.​‌Tất nhiên chúng ta sẽ bàn về chuyện đó.

The leader will have a virtual app that will use this virtual API to connect to the leader directly​‌Người lãnh đạo sẽ có một ứng dụng ảo sử dụng API ảo này để kết nối trực tiếp với người lãnh đạo.

instead of connection to the different KVM.​‌thay vì kết nối với các KVM khác nhau.

So if the leader file to respond, a new leader is automatically elected.​‌Vì vậy, nếu người lãnh đạo cũ phản hồi, một người lãnh đạo mới sẽ tự động được bầu chọn.

Zuckerberg is accessing via an interface.​‌Zuckerberg đang truy cập thông qua một giao diện.

Scarlett Zeus.​‌Nữ thần Zeus đỏ thắm.

So this is enterprise access to the zookeeper.​‌Đây là quyền truy cập doanh nghiệp vào Zookeeper.

So the zookeeper has no dependency, meaning that it can start without any other cluster components​‌Như vậy, Zookeeper không có bất kỳ phụ thuộc nào, nghĩa là nó có thể khởi động mà không cần bất kỳ thành phần nào khác trong cụm máy chủ.

running mean if the other components didn't start in the KVM, the zookeeper can work separately without​‌Nếu các thành phần khác không khởi động trong KVM, Zookeeper có thể hoạt động riêng biệt mà không cần...

needing to have the other service available.​‌cần phải có dịch vụ khác sẵn có.

Then do zoos.​‌Tiếp theo là các sở thú.

The access interface for Zookeeper is a key element of a distributed system.​‌Giao diện truy cập của Zookeeper là một yếu tố quan trọng của hệ thống phân tán.

This is a method for our node to store and update the cluster configuration.​‌Đây là phương pháp để nút của chúng ta lưu trữ và cập nhật cấu hình cụm.

This Zeus configuration includes the tie about the physical components in the cluster, such as a host​‌Cấu hình Zeus này bao gồm liên kết giữa các thành phần vật lý trong cụm, chẳng hạn như máy chủ.

and disk and logical component like storage container.​‌và đĩa cũng như các thành phần logic như thùng chứa lưu trữ.

The state of these components, including their IP address capacities and data replication rule, are​‌Trạng thái của các thành phần này, bao gồm dung lượng địa chỉ IP và quy tắc sao chép dữ liệu, là:

also stored in the cluster configuration.​‌Thông tin này cũng được lưu trữ trong cấu hình cụm.

So this is the Nutanix library.​‌Đây là thư viện Nutanix.

That's all other components used to access the cluster configuration, which is currently implemented​‌Đó là tất cả các thành phần khác được sử dụng để truy cập cấu hình cụm, hiện đang được triển khai.

using Apache Zookeeper.​‌Sử dụng Apache Zookeeper.

Then last and not least, the prison.​‌Và cuối cùng nhưng không kém phần quan trọng, đó là nhà tù.

So the prison is the management interface for Nutanix struck the graphical user interface, Nutanix​‌Vì vậy, nhà tù là giao diện quản lý cho Nutanix, được thiết kế dựa trên giao diện người dùng đồ họa của Nutanix.

command line interface and the RPA interface.​‌giao diện dòng lệnh và giao diện RPA.

So the prison is the management gets UI for component and administrator to configure and monitor the​‌Vì vậy, nhà tù là nơi ban quản lý có giao diện người dùng (UI) cho các thành phần và quản trị viên để cấu hình và giám sát.

Nutanix cluster.​‌Cụm máy chủ Nutanix.

These include Nutanix command line interface, the HTML5 user interface and the rest API interface.​‌Chúng bao gồm giao diện dòng lệnh Nutanix, giao diện người dùng HTML5 và giao diện API REST.

So PRISM run on every node in the cluster and used an elected leader like all components in the cluster.​‌Vì vậy, PRISM chạy trên mọi nút trong cụm và sử dụng một người lãnh đạo được bầu chọn giống như tất cả các thành phần khác trong cụm.

All requests are for what to do later using Linux IP tables.​‌Tất cả các yêu cầu đều liên quan đến những việc cần làm sau này bằng cách sử dụng bảng IP của Linux.

This allow across the prison using any KVM IP address that mean when we create our environment and create​‌Điều này cho phép truy cập từ khắp nhà tù bằng bất kỳ địa chỉ IP KVM nào, có nghĩa là khi chúng ta tạo môi trường và thiết lập.

the different KVM, we can connect to any KVM to manage our environment.​‌Với nhiều thiết bị KVM khác nhau, chúng ta có thể kết nối với bất kỳ thiết bị KVM nào để quản lý môi trường của mình.

And one of those kvm will be due later If we connect to the slave VM IP address.​‌Và một trong những KVM đó sẽ cần thiết sau này nếu chúng ta kết nối với địa chỉ IP của máy ảo phụ.

The slave VM will forward the connection to the later automatically to the one that is elected to be​‌Máy ảo phụ sẽ tự động chuyển tiếp kết nối đến máy ảo được chọn.

a later.​‌sau đó.

So the same we can always manage our environment from the different KVM IP address they always will​‌Vì vậy, chúng ta luôn có thể quản lý môi trường của mình từ các địa chỉ IP KVM khác nhau, chúng sẽ luôn như vậy.

redirect us to do later kvm or if we use the virtual IP.​‌Hãy chuyển hướng chúng tôi đến việc thực hiện KVM sau này hoặc nếu chúng tôi sử dụng địa chỉ IP ảo.

Address of the cluster, it will automatically redirect us to the laser beam.​‌Địa chỉ của cụm máy chủ sẽ tự động dẫn hướng chúng ta đến chùm tia laser.

So the PRISM communicates with Zeus for cluster configuration data and Cassandra for statistics to present​‌Vì vậy, PRISM giao tiếp với Zeus để lấy dữ liệu cấu hình cụm và với Cassandra để lấy số liệu thống kê để trình bày.

to the user.​‌cho người dùng.

It also communicates with the A-6 I host for VMS states and related information.​‌Nó cũng liên lạc với máy chủ A-6 I để lấy trạng thái VMS và các thông tin liên quan.

So this is the different components we have in our VM that will help us to understand.​‌Đây là các thành phần khác nhau mà chúng ta có trong máy ảo, giúp chúng ta hiểu rõ hơn.

How do see them work?​‌Bạn thấy chúng hoạt động như thế nào?

And also will help us if we have some problem in a specific data or in a specific situation, we can​‌Và điều này cũng sẽ giúp ích cho chúng ta nếu gặp vấn đề với dữ liệu cụ thể hoặc trong một tình huống cụ thể nào đó, chúng ta có thể...

easily know which service or which components responsible for this troubleshooting.​‌Dễ dàng xác định dịch vụ hoặc thành phần nào chịu trách nhiệm cho việc khắc phục sự cố này.

Then we can check the components easily.​‌Như vậy chúng ta có thể dễ dàng kiểm tra các thành phần.

For example, check if the service is star or not.​‌Ví dụ, hãy kiểm tra xem dịch vụ đó có đạt chuẩn sao hay không.

Is the service running perfectly or not?​‌Dịch vụ có đang hoạt động hoàn hảo hay không?

So we know the problem.​‌Vậy là chúng ta đã biết vấn đề rồi.

Where is it?​‌Nó đâu rồi?

So this is the idea of the component of the VM and you can easily know in the style what happened in​‌Vậy đây là ý tưởng về thành phần của máy ảo và bạn có thể dễ dàng biết được điều gì đã xảy ra theo phong cách này.

the background when you connect to the cluster environment through the browser right here.​‌Đây là hình ảnh nền khi bạn kết nối với môi trường cụm máy chủ thông qua trình duyệt.

So the browser will connect to the present HTTP or rest API.​‌Vì vậy, trình duyệt sẽ kết nối với API HTTP hoặc REST hiện tại.

The PRISM will handle the environment through any task you do is from the prison.​‌Hệ thống PRISM sẽ xử lý môi trường xung quanh bất kể bạn thực hiện nhiệm vụ nào từ trong nhà tù.

For example, something about the Zuercher.​‌Ví dụ, một vài thông tin về giống chó Zuercher.

It will send it to the Zucker, it will send it to Cassandra, to Cassandra, to the street and then​‌Nó sẽ gửi cho Zucker, nó sẽ gửi cho Cassandra, cho Cassandra, ra đường và sau đó...

the hypervisor.​‌bộ ảo hóa.

So I hope that you get the idea about how our Nutanix environments work and the components of the Nutanix​‌Tôi hy vọng bạn đã hiểu được cách thức hoạt động của môi trường Nutanix và các thành phần của Nutanix.

solution environment.​‌Môi trường giải pháp.

So see in the next lecture. ​‌Hãy cùng chờ đón bài giảng tiếp theo.
