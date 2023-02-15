---
layout: post
title:  "Serverless: Tối ưu việc xử lý batch data với AWS Lambda + SQS batch message"
date:   2023-02-13 15:54:02 +0700
tags: serverless
categories: serverless
---

{% if post.tags.size > 0 %}
  Tag{% if post.tags.size > 1 %}s{% endif %}:
  {{ post.tags | sort | join: ", " }}
{% endif %}

![image1](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/resources/images/architecture.png)

Khi phát triển hệ thống back-end cho ứng dụng, hầu hết chúng ta đều gặp các yêu cầu xử lý dữ liệu theo 1 thời gian biểu nào đó. AWS cung cấp cho chúng ta **EventBridge** và **Lambda** để hiện thực hóa cronjob theo thời gian biểu định sẵn (_schedule expression_). 

Tuy nhiên, khi **Lambda** function thực hiện query dữ liệu từ DB/file để xử lý, số lượng data cần xử lý có thể sẽ rất lớn --> dẫn tới việc **Lambda** sẽ vượt ngưỡng timeout 15 phút. 

Để giải quyết vấn đề này, chúng ta có thể kết hợp cronjob Lambda cùng với SQS để tăng tốc việc xử lý data được query từ cronjob Lambda. Khi SQS nhận được data message từ cronjob Lambda, SQS sẽ trigger Lambda để xử lý message. Với việc Lambda hỗ trợ cơ chế autoscaling, các message trong SQS sẽ được xử lý đồng thời bởi các Lambda function, giúp cho thời gian xử lý lượng lớn data được giảm xuống đáng kể.

![image2](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/resources/images/lambda-scale.png)

Nhưng khi triển khai SQS, tôi nhận thấy rằng việc truyền nhận dữ liệu giữa **Lambda** và **SQS** thông qua HTTP protocol. Vậy có nghĩa là sẽ có latency giữa các request và response, và còn phụ thuộc vào connection tại thời điểm gửi nhận dữ liệu. Nếu từ cronjob Lambda, chúng ta thực hiện gửi tuần tự từng data request vào SQS, do có latency giữa các request và response, thời gian hoàn thành xử lý batch message sẽ lâu hơn.

Và AWS cũng cung cấp cho chúng ta 1 giải pháp tốt hơn khi cần xử lý lượng lớn data, đó chính là [SQS Message Batch](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html). Với số lượng tối đa 10 message trong 1 request gửi tới SQS, chúng ta có thể giảm thời gian xử lý data.

Và ngoài việc áp dụng SQS Message Batch khi gửi nhận từ SQS, chúng ta có thể tăng tốc độ xử lý batch message trong 1 Lambda function bằng cách triển khai concurrency handler. Như vậy với 1 batch message nhận được từ SQS, Lambda function cũng sẽ xử lý đồng thời các message trong batch message.

Với việc triển khai giải pháp như trên, chúng ta có thể đạt được lợi ích:
* Tăng tốc độ xử lý batch trong hệ thống
* Tiết kiệm resources của hệ thống
* Tiết kiệm chi phí do giảm số lượng request gửi vào SQS cũng như giảm dung lượng của message

Các bạn có thể tham khảo tài liệu của AWS:

[Batch action](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-batch-api-actions.html)

[Send Batch Message API Document](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html)

[Amazon Simple Queue Service](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)