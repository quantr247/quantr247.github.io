---
layout: post
title:  "Serverless: (Go) Hiểu về cách sử dụng Lambda với AWS SQS batch message. Report batch item failure cho Lambda function"
date:   2023-06-30 11:00:00 +0700
tags: serverless golang
categories: serverless
---
\
Ở bài post trước [Tối ưu việc xử lý batch data với AWS Lambda + SQS batch message]({% post_url 2023-02-15-go-lambda-sqs %}), tôi đã chia sẻ với bạn việc sử dụng message batch trong SQS để tối ưu hệ thống cũng như giảm cost. Tuy nhiên khi test và triển khai chạy thực tế, tôi phát hiện ra có 1 số vấn đề về cách mà tôi triển khai cũng như những hiểu biết về SQS mà tôi chưa hiểu hết. Hãy theo dõi bài post này để cùng tôi khám phá về những kiến thức mới nào.

## **Lambda Timeout và SQS Visibility Timeout**
---
\
**SQS visibility timeout** là khoảng thời gian mà SQS sẽ không return message khi có request gọi tới SQS để pull message. Tức là message được gửi tới SQS bởi producer, nó sẽ ở trạng thái _available_. Khi consumer nhận được message từ queue, trạng thái của message sẽ là _in flight_. Khi message ở trạng thái _in flight_, nó sẽ không bị delete trong queue, nhưng các consumer khác sẽ không nhận được message. Và thời gian để message tồn tại ở trạng thái _in flight_ này chính là thời gian _Visibility Timeout (in seconds)_. Giá trị mặc định của visibility timeout là 30 giây, tối thiểu là 0 giây và tối đa là 12 giờ. [AWS Docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)

![image1](https://docs.aws.amazon.com/images/AWSSimpleQueueService/latest/SQSDeveloperGuide/images/sqs-visibility-timeout-diagram.png)

**Lambda function timeout** là khoảng thời gian tối đa mà Lambda function có thể chạy trong AWS Lambda. Giá trị mặc định là 3 giây, tối thiểu là 1 giây và tối đa là 15 phút. [AWS Docs](https://docs.aws.amazon.com/lambda/latest/dg/configuration-function-common.html)

## **Vấn đề**
---
```
    Lambda function timeout: 900
    Lambda ReservedConcurrency: 3
    SQS visibility timeout: 900
    Max Retry Count: 1
```

Trên đây là cấu hình cho Lambda function consumer message từ SQS và cấu hình thời gian _visibility timeout_ của SQS ở bài post trước. Và khi test hệ thống với cấu hình như trên, tôi nhận ra có vấn đề:
* Giả sử có 130 message gửi vào SQS. 130 message được [AWS Lambda Event Source Mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html) pull về để xử lý.
* Tuy nhiên sau khi Consumber Lambda function xử lý xong message (khoảng 100 giây). Có khoảng 30 message vẫn ở trạng thái _in flight_ và giữ nguyên trạng thái này đến khi hết 900 giây của _visibility timeout_.
* Sau 900 giây, 30 message này được chuyển về Dead Letter Queue.

Đến đây tôi cũng thắc mắc, tại sao sau khi 100 message được xử lý thành công mà 30 message còn lại không được **AWS Lambda Event Source Mapping** invoke Lambda function để xử lý???

## **Giải pháp**
---
\
Lý do đầu tiên tôi nghĩ đến cho vấn đề trên chính là _Lambda Reserved Concurrency_. Có thể là do khi event source mapping invoke Lambda function để xử lý message, nhưng Lambda function đã tới ngưỡng limit của reserved concurrency là 3 nên 30 message này không thể được xử lý. Đây được gọi là _throttled errors_.

Để chứng minh giả thuyết này, tôi đã thử bỏ cấu hình _Lambda Reserved Concurrency_ và test lại. Kết quả là 130 message đã được xử lý hết, không còn 30 message ở trạng thái _in flight_ như trên.

Nhưng nếu bỏ đi cấu hình _Lambda Reserved Concurrency_ thì không đúng mục đích thiết kế của tôi. Lần này tôi quyết định vẫn set cấu hình _Lambda Reserved Concurrency_ là 3, _Lambda function timeout_ giảm còn 150 giây và _visibility timeout_ vẫn là 900 giây (gấp 6 lần của Lambda timeout, theo tài liệu tôi tham khảo từ [AWS Docs](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html#events-sqs-queueconfig)). Với _visibility timeout_ gấp 6 lần Lambda timeout sẽ cho phép Lambda có thể retry lại những batch nào bị _throttled errors_ ở trên.

OK tiếp tục test lại 1 lần nữa nào. Và lần này 130 message cũng đã được xử lý hết, không còn 30 message ở trạng thái _in flight_ như trên. Vậy là vấn đề của tôi đã được giải quyết với thông số cấu hình như sau:

```
    Lambda function timeout: 150
    Lambda ReservedConcurrency: 3
    SQS visibility timeout: 900
    Max Retry Count: 1
```

## **Reporting batch item failures**
---
Khi bạn đã implement Lambda để xử lý batch message, bạn cũng đã biết rằng nếu có lỗi xảy ra trong quá trình xử lý 1 message nào đó của batch và return error, thì tất cả các message trong batch sẽ xuất hiện lại trong queue, bao gồm cả những message đã được xử lý thành công. Cho nên trong quá trình xử lý message của batch, ta cần delete các message đã được xử lý thành công ở trong queue. Ở bài post trước [Tối ưu việc xử lý batch data với AWS Lambda + SQS batch message]({% post_url 2023-02-15-go-lambda-sqs %}), tôi đang dùng giải pháp gọi api _DeleteMesssageBatch_, với data input là những message cần được delete.

Để đơn giản hơn, tôi sẽ sử dụng giải pháp _Report batch item failures_, với config này _event source mapping_ sẽ tự động xóa những message không nằm trong batch item failure, còn những message nào được add vào batch item failures sẽ được visible lại trong queue. Đây được gọi là **partial batch response**.

### **Implement**
---
* AWS Console\
    a. Chọn AWS Lambda service\
    b. Chọn function handle batch message from SQS\
    c. Chọn tab **Configuration** --> ****Trigger**\
    d. Chọn SQS --> click _Edit_\
    e. Tại màn hình **Edit trigger**, enable config _Report batch item failures - optional_\
    ![image2](/assets/20230630/lambda-edit-trigger.png)
    f. Chọn _Save_

    Và bạn sẽ được kết quả như sau:
    ![image3](/assets/20230630/lambda-trigger-report-yes.png)

* AWS CLI
    ```
        aws lambda update-event-source-mapping \
        --uuid "a1b2c3d4-5678-90ab-cdef-11111EXAMPLE" \
        --function-response-types "ReportBatchItemFailures"
    ```
    Với uuid là giá trị như hình kết quả ở trên.

### **Golang integrate**
---
* Trước hết, ta có 1 Lambda function nhận input batch message là events.SQSEvent, và để report nhũng item nào bị failure trong batch, ta sẽ cần output là events.SQSEventResponse.

{% highlight go %}
func main() {
	lambda.Start(func(ctx context.Context, sqsEvent events.SQSEvent) (events.SQSEventResponse, error) {
		sqsResponse := events.SQSEventResponse{}
		listItemFailure, err := Run(ctx, sqsEvent)
		if err != nil {
			return sqsResponse, fmt.Errorf("ERROR: %+v", err)
		}

		sqsResponse.BatchItemFailures = listItemFailure
		return sqsResponse, nil
	})
}
{% endhighlight %}

* Và ta cần add những message bị xử lý fail vào list item failure để response cho _event source mapping_

{% highlight go %}
/ Run executes the downstream process
func Run(ctx context.Context, sqsEvent events.SQSEvent) ([]events.SQSBatchItemFailure, error) {
    listItemFailure := []events.SQSBatchItemFailure{}
    for _, mess := range sqsEvent.Records {
        if err := s.proceedMessage(ctx, mess.Body); err != nil {
            listItemFailure = append(listItemFailure, events.SQSBatchItemFailure{
                ItemIdentifier: mess.MessageId,
            })
            continue
        }
    }
    return listItemFailure, nil
}
{% endhighlight %}

## **Tổng kết**
---

Qua bài post này, tôi đã chia sẻ với các bạn những hiểu biết về AWS SQS cũng như cách triển khai Lambda với AWS SQS batch message, hi vọng sẽ giúp ích cho các bạn trong quá trình làm việc của mình.

Chúc bạn làm việc hiệu quả :)

### **Tài liệu tham khảo**
---
\
[SQS Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)

[Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)

[Understanding Amazon SQS and AWS Lambda Event Source Mapping for Efficient Message Processing](https://aws.amazon.com/blogs/apn/understanding-amazon-sqs-and-aws-lambda-event-source-mapping-for-efficient-message-processing/)

[Implementing partial batch responses](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html#services-sqs-batchfailurereporting)