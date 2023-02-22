---
layout: post
title:  "Serverless: (Golang) Giảm duration time của AWS Lambda bằng caching với Global variables"
date:   2023-02-22 11:00:00 +0700
tags: serverless
categories: serverless
---
\
Khi phát triển các backend API cho ứng dụng, tôi luôn hướng tới mục đích làm sao để giảm tối thiểu duration time của backend service, giúp cho latency của ở phía client đạt được mức thấp nhất.

Thông thường có nhiều giải pháp để tối ưu duration time ở phía backend: 
* Set index cho table trong database, giúp tăng tốc khi query dữ liệu
* Tối ưu code
* Sử dụng cơ chế cache

Nếu bạn đã tìm hiểu hoặc làm qua về cache, chắc hẳn cũng đã nghe tới **Redis**, hoặc là **in-memory cache** (locally cache). Tuy nhiên trong phạm vi bài post này, tôi sẽ nói về cách triển khai in-memory cache trong AWS Lambda như thế nào.

Nào chúng ta cùng bắt đầu thôi :)))

## **Lambda execution environment**
---
\
Theo tài liệu của AWS định nghĩa

>Lambda invokes your function in an execution environment, which provides a secure and isolated runtime environment. The execution environment manages the resources required to run your function. The execution environment also provides lifecycle support for the function's runtime and any external extensions associated with your function.

![image1](https://raw.githubusercontent.com/quantr247/golang-lambda-inmemcache-example/master/assets/lambda_execute_env_life_cycle.png)

Có thể hiểu đơn giản rằng mỗi khi có request gọi vào Lambda function, Lambda service sẽ khởi tạo môi trường, cũng như khởi tạo các static code của Lambda trước. Sau đó sẽ thực thi code được định nghĩa bên trong function lambda.Start() của lambda sdk (tương ứng bước INVOKE như hình vẽ trên).

**Lambda execution environment** chính là vùng nhớ được dùng chung cho các INVOKE function riêng biệt. Nếu trong lần INVOKE trước, ta lưu giá trị vào 1 biến nằm trong Lambda execution environment. Thì ở lần INVOKE sau, ta có thể lấy được giá trị từ biến đã được lưu ở lần INVOKE trước. Và biến này chính là **Global variable**.

Nhưng câu chuyện không đơn giản như vậy, vì AWS Lambda được thiết kế với khả năng scale up nhanh chóng nhằm đáp ứng được các request đồng thời. Ta sẽ cùng tìm hiểu tiếp về cơ chế scaling của Lambda nhé.

## **Lambda function scaling**
---
\
Theo AWS định nghĩa về concurrency trong Lambda

>Concurrency is the number of in-flight requests your AWS Lambda function is handling at the same time. For each concurrent request, Lambda provisions a separate instance of your execution environment. As your functions receive more requests, Lambda automatically handles scaling the number of execution environments until you reach your account's concurrency limit.

![image2](https://raw.githubusercontent.com/quantr247/golang-lambda-inmemcache-example/master/assets/concurrency-3-ten-requests.png)

Với định nghĩa và hình vẽ giải thích ở trên, ta có thể hiểu được rằng khi có nhiều request đồng thời gửi tới Lambda, Lambda service sẽ khởi tạo đồng thời nhiều Lambda execution environment và dùng nó để INVOKE function. Sau khi lần INVOKE của request trước kết thúc, Lambda execution environment sẽ được tái sử dụng để INVOKE function cho request tiếp theo. 

Nhưng nếu sau 1 khoảng thời gian (tôi không tìm thấy con số chính xác trong tài liệu của AWS, nhưng có thể tùy vào resource của AWS Lambda tại từng thời điểm), Lambda execution environment sẽ bị terminate nếu không có bất kỳ hoạt động nào. Và nếu có request mới sau đó, Lambda execution environment sẽ phải khởi tạo lại từ đầu (bước INIT như hình). Đây chính là nguồn cơn của 2 thuật ngữ rất phổ biến khi bạn làm việc với AWS Lambda: **cold start** và **warm start**.

## **Global variables**
---
\
Quay lại với mục đích chính là in-memory cache trong Lambda, chúng ta sẽ sử dụng Globla variables được lưu tại execution environment của Lambda để lưu giữ và truy xuất dữ liệu khi có các request INVOKE function. 

Tuy nhiên, cơ chế của Lambda là tạo từng vùng environment độc lập khi có nhiều request đồng thời. Nên các Global variables cũng sẽ độc lập với nhau, không có cơ chế nào để đồng bộ được dữ liệu của Global variables giữa các environment. Chúng ta sẽ không làm được như Redis là dùng 1 key để cache, và cache data này có thể cho toàn bộ các distribute service trong hệ thống truy xuất được.

Vậy thì in-memory cache trong Lambda có thể được dùng cho mục đích gì? Nó sẽ tùy thuộc vào use case mà bạn gặp phải, với tôi thì có thể sử dụng trong 1 số trường hợp như:
* Static data từ database, đây là những dữ liệu gần như không có sự thay đổi hoặc ít khi cần thay đổi. Ví dụ: mã quốc gia, mã bưu điện, danh sách tỉnh thành, quận huyện,... hoặc dữ liệu đặc thù theo requirement của ứng dụng.
* Threshold value, các giá trị config dùng cho business logic của API hoặc hệ thống,...
* Vân vân và mây mây, tùy theo use case cụ thể :)))

OK! Lý thuyết tới đây là ổn, giờ chúng ta sẽ thực hiện code thử nhé

## **Example in Go**
---

