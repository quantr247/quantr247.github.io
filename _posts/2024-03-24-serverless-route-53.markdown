---
layout: post
title:  "Serverless: Amazon Route 53 là gì? Theo dõi kỹ bill của bạn khi sử dụng Route 53 Resolver"
date:   2024-03-24 11:00:00 +0700
tags: serverless
categories: serverless
---

Amazon Route 53 cung cấp dịch vụ DNS có tính linh hoạt và đáp ứng cao, dịch vụ đăng ký tên miền và là 1 dịch vụ health-checking.

Amazon Route 53 sẽ tính phí cho những gì bạn sử dụng, có 3 dịch vụ chính:
* Managing hosted zones: trả phí hàng tháng cho mỗi hosted zone được quản lý bởi Route 53
* Serving DNS queries: trả phí cho mỗi DNS query được trả lời bởi dịch vụ Amazon Route 53 ngoại trừ những queries tới Alias của record mapped với Elastic Load Balancing, CloudFront, AWS Elastic Beanstalk, API Gateways, VPC endpoints, Amazon S3 (những dịch vụ không bị charge phí).
* Managing domain names: trả phí định kỳ cho mỗi domain name được đăng ký hoặc transferred into Route 53.

Trong quá trình phát triển và tích hợp api cho back-end service trên AWS Lambda, bill của tôi đã tăng lên chóng mặt từ khi tích hợp api gọi sang third-party. Thủ phạm chính khiến bill tăng là do Route 53 Resolver, nên chúng ta sẽ cùng tìm hiểu về Route 53 Resolver và những lưu ý để kiểm soát bill của bạn tốt hơn nha.

## **Route 53 Resolver**

Route 53 Resolver là một dịch vụ DNS theo khu vực, nó cung cấp dịch vụ tìm kiếm recursive DNS cho names hosted in EC2 cũng như public names trên internet. Chức năng này là mặc định trong mỗi VPC.

Amazon Route 53 có 2 service là Authoritative DNS và Recursive DNS (còn gọi là DNS resolvers). Route 53 Resolver chính là recursive DNS service. [Docs](https://aws.amazon.com/vi/route53/faqs/)


## **Tổng kết**
---
\
Với việc triển khai giải pháp như trên, chúng ta có thể đạt được lợi ích:
* Tăng tốc độ xử lý batch data trong hệ thống
* Tiết kiệm resources của hệ thống
* Tiết kiệm chi phí do giảm số lượng request gửi vào SQS cũng như giảm dung lượng của message

Tuy nhiên, đây chỉ là giải pháp chung cho các yêu cầu về xử lý batch data trong AWS Serverless. Tùy thuộc vào use case, cũng như yêu cầu cụ thể mà bạn cần điều chỉnh lại cho phù hợp với hệ thống của mình.

Chúc bạn làm việc hiệu quả :)

### **Bạn có thể tham khảo thêm tài liệu của AWS**
---
\
[Batch action](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-batch-api-actions.html)

[Send Batch Message API Document](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html)

[Amazon Simple Queue Service](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)