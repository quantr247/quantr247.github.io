---
layout: post
title:  "Tối ưu hệ thống với AWS Lambda + SQS batch message"
date:   2023-02-15 16:54:02 +0700
categories: serverless
---

![image1](https://blog.knoldus.com/wp-content/uploads/2022/05/image-3-1.png)
_Photo by [Naincy Kumari!](https://blog.knoldus.com/author/naincykumariknoldus/) at [knoldus!](https://blog.knoldus.com/how-to-schedule-a-lambda-function-using-amazon-eventbridge/)_

Khi phát triển hệ thống back-end cho ứng dụng, hầu hết chúng ta đều gặp các yêu cầu xử lý dữ liệu theo 1 thời gian biểu nào đó. AWS cung cấp cho chúng ta **EventBridge** và **Lambda** để hiện thực hóa cronjob theo thời gian biểu định sẵn (_schedule expression_). 

Tuy nhiên, khi **Lambda** function thực hiện query dữ liệu từ DB/file để xử lý, số lượng data cần xử lý có thể sẽ rất lớn --> dẫn tới việc **Lambda** sẽ vượt ngưỡng timeout 15 phút. 

![image2](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2021/11/09/sqs1.png)

Để giải quyết vấn đề này, chúng ta có thể kết hợp cronjob Lambda cùng với SQS để tăng tốc việc xử lý data được query từ cronjob Lambda. Khi SQS nhận được data message từ cronjob Lambda, SQS sẽ trigger Lambda để xử lý message. Với việc Lambda hỗ trợ cơ chế autoscaling, các message trong SQS sẽ được xử lý đồng thời bởi các Lambda function, giúp cho thời gian xử lý lượng lớn data được giảm xuống đáng kể. 

Khi tìm hiểu thêm về SQS, tôi nhận thấy rằng việc truyền nhận dữ liệu giữa **Lambda** và **SQS** thông qua HTTP protocol. Vậy có nghĩa là sẽ có latency giữa các request và response, và còn phụ thuộc vào connection tại thời điểm gửi nhận dữ liệu. Nếu từ cronjob Lambda, chúng ta thực hiện gửi tuần tự từng data request vào SQS, do có latency giữa các request và response, thời gian hoàn thành xử lý batch message sẽ lâu hơn.

Và AWS cũng cung cấp cho chúng ta 1 giải pháp tốt hơn khi cần xử lý lượng lớn data, đó chính là [SQS Message Batch!](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html). Với số lượng tối đa 10 message trong 1 request gửi tới SQS, chúng ta có thể giảm thời gian xử lý data xuống thêm 10 lần (maybe :)) )

Các bạn có thể xem thêm tài liệu của AWS [tại đây!](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-throughput-horizontal-scaling-and-batching.html)

Và ngoài việc áp dụng SQS Message Batch khi gửi nhận từ SQS, chúng ta có thể tăng tốc độ xử lý batch message trong 1 Lambda function bằng cách triển khai concurrency hanlder. Như vậy với 1 batch message nhận được từ SQS, Lambda function cũng sẽ xử lý đồng thời các message trong batch message.

will update code soon.