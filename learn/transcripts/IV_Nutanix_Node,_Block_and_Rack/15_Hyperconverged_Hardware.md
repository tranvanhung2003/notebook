Hello and welcome again.​‌Xin chào và chào mừng trở lại.

So, no, in this video, we're going to talk about the Nutanix nerd BLOCK and rack.​‌Vậy nên, trong video này, chúng ta sẽ nói về Nutanix Nerd BLOCK và Rack.

So this is the different type of hardware we have in any environment, any environment with it should​‌Vậy đây là các loại phần cứng khác nhau mà chúng ta có trong bất kỳ môi trường nào, bất kỳ môi trường nào có nó đều nên như vậy.

have nerd block and drag and of course, the network hardware to switch and to cable.​‌Cần có các công cụ phức tạp như khối lệnh và kéo thả, và tất nhiên, cả phần cứng mạng để chuyển mạch và kết nối cáp.

But we're going to talk about the nerd block and drag, because it is very important to know this information​‌Nhưng chúng ta sẽ nói về chứng "bí ý tưởng" và sự trì trệ của dân mọt sách, bởi vì việc nắm rõ thông tin này rất quan trọng.

before being able to make the configuration of the Nutanix hight of ability or the Nutanix resiliency,​‌trước khi có thể thực hiện cấu hình khả năng cao hoặc khả năng phục hồi của Nutanix,

because through the course we try to config some Nutanix resiliency, you will find that we have resiliency​‌Bởi vì trong suốt khóa học, chúng tôi cố gắng cấu hình một số tính năng phục hồi của Nutanix, bạn sẽ thấy rằng chúng tôi có tính năng phục hồi.

at the level of the node or at the block or at the level of the rack.​‌ở cấp độ nút, cấp độ khối hoặc cấp độ giá đỡ.

So before that, if we back to this picture, I want to add an information, a quick information right​‌Vậy trước đó, nếu chúng ta quay lại bức ảnh này, tôi muốn bổ sung thêm một vài thông tin ngắn gọn.

here is, if you remember in the preview lecture, we talk about the Hyperconverged platform from the​‌Nếu các bạn còn nhớ trong bài giảng giới thiệu, chúng ta đã nói về nền tảng siêu hội tụ (Hyperconverged platform) từ...

classical one to do a Hyperconverged platform right here.​‌Đây là một cách cổ điển để xây dựng nền tảng siêu hội tụ.

And you always see that I use the Nutanix, a hardware block right here.​‌Và các bạn luôn thấy tôi sử dụng Nutanix, một khối phần cứng ngay tại đây.

I want you to know that we use the Nutanix hardware as an example because this course is about Nutanix​‌Tôi muốn các bạn hiểu rằng chúng tôi sử dụng phần cứng Nutanix làm ví dụ vì khóa học này tập trung vào Nutanix.

solution, but it is not necessary to have a Nutanix hardware in your environment to be able to follow​‌giải pháp, nhưng không nhất thiết phải có phần cứng Nutanix trong môi trường của bạn để có thể làm theo.

this course and to apply the Nutanix solution in your environment.​‌Khóa học này sẽ giúp bạn áp dụng giải pháp Nutanix vào môi trường của mình.

You can use any other hardware that support the hyper converged platform.​‌Bạn có thể sử dụng bất kỳ phần cứng nào khác hỗ trợ nền tảng siêu hội tụ.

So the same that you see in this example, it can also be as the same as in this example right here​‌Vì vậy, điều bạn thấy trong ví dụ này cũng có thể tương tự như trong ví dụ ngay đây.

that mean any brand of block or hardware you have.​‌Điều đó có nghĩa là bất kỳ thương hiệu khối hoặc phần cứng nào bạn đang sử dụng.

You can use the Hyperconverged platform from the classical one to the new Hyperconverged platform,​‌Bạn có thể sử dụng nền tảng siêu hội tụ, từ nền tảng truyền thống đến nền tảng siêu hội tụ mới.

and you get the same benefits scale, cloud computing and storage and virtualization and administration​‌và bạn nhận được những lợi ích tương tự về quy mô, điện toán đám mây, lưu trữ, ảo hóa và quản trị.

included.​‌Bao gồm.

Of course, if you use another hardware rather than the Nutanix hardware, you don't get the virtualization​‌Dĩ nhiên, nếu bạn sử dụng phần cứng khác thay vì phần cứng của Nutanix, bạn sẽ không có được khả năng ảo hóa.

included.​‌Bao gồm.

The virtualization and administration is included with the hardware of Nutanix.​‌Công nghệ ảo hóa và quản trị được tích hợp sẵn trong phần cứng của Nutanix.

If you get your own hardware from the different company, you need to buy the license from the Nutanix​‌Nếu bạn tự mua phần cứng từ một công ty khác, bạn cần mua giấy phép từ Nutanix.

solution to get the Nutanix software and use the Hyperconverged platform.​‌Giải pháp để có được phần mềm Nutanix và sử dụng nền tảng siêu hội tụ.

You only need that you hardware support the Hyperconverged platform.​‌Bạn chỉ cần đảm bảo phần cứng của mình hỗ trợ nền tảng Hyperconverged.

So right here, some example if you want to use the hyper converged hardware, both the Nutanix and​‌Vậy nên, ngay đây là một vài ví dụ nếu bạn muốn sử dụng phần cứng siêu hội tụ, cả Nutanix và...

third party option are available.​‌Có sẵn các tùy chọn của bên thứ ba.

That's mean you can use the Nutanix hardware or you can use the IBM, the Dell, Lenovo or any brand​‌Điều đó có nghĩa là bạn có thể sử dụng phần cứng của Nutanix hoặc của IBM, Dell, Lenovo hay bất kỳ thương hiệu nào khác.

you have in your hardware.​‌Bạn có trong phần cứng của mình.

It only should support the Hyperconverged platform.​‌Nó chỉ nên hỗ trợ nền tảng siêu hội tụ.

So this is a quick idea before start talking about the block node and rack. ​‌Đây là một ý tưởng nhanh trước khi bắt đầu nói về khối nút và giá đỡ.
