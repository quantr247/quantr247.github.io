---
layout: post
title:  "Serverless: Tối ưu việc xử lý batch data với AWS Lambda + SQS batch message"
date:   2023-02-15 15:54:02 +0700
tags: serverless
categories: serverless
---

![image1](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/assets/images/architecture.png)

Khi phát triển hệ thống back-end cho ứng dụng, hầu hết chúng ta đều gặp các yêu cầu xử lý dữ liệu theo 1 thời gian biểu nào đó. AWS cung cấp cho chúng ta **EventBridge** và **Lambda** để hiện thực hóa cronjob theo thời gian biểu định sẵn (_schedule expression_). 

Tuy nhiên, khi **Lambda** function thực hiện fetch dữ liệu từ DB/file để xử lý, số lượng data cần xử lý có thể sẽ rất lớn, dẫn tới việc **Lambda** sẽ vượt ngưỡng timeout 15 phút.

## **Giải pháp**
---

Để giải quyết vấn đề này, chúng ta có thể kết hợp cronjob Lambda cùng với SQS để tăng tốc việc xử lý data được fetch từ cronjob Lambda. Khi SQS nhận được data message từ cronjob Lambda, SQS sẽ trigger Lambda để xử lý message. Với việc Lambda hỗ trợ cơ chế autoscaling, các message trong SQS sẽ được xử lý đồng thời bởi các Lambda function, giúp cho thời gian xử lý lượng lớn data được giảm xuống đáng kể.

![image2](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/assets/images/lambda-scale.png)

Nhưng khi triển khai SQS, chúng ta có thể thấy rằng việc truyền nhận dữ liệu giữa **Lambda** và **SQS** thông qua HTTP protocol. Vậy có nghĩa là sẽ có latency giữa các request và response, và còn phụ thuộc vào connection tại thời điểm gửi nhận dữ liệu. Nếu từ cronjob Lambda, chúng ta thực hiện gửi tuần tự từng data request vào SQS, vì có latency giữa các request và response, thời gian hoàn thành xử lý batch message sẽ lâu hơn.

Chúng ta cần có 1 giải pháp tốt hơn khi cần xử lý số lượng lớn data, đó chính là [SQS Message Batch](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html). Với số lượng tối đa 10 message trong 1 request gửi tới SQS, chúng ta có thể tiết kiệm được 1 lượng lớn thời gian latency giữa các request và response so với việc gửi từng request riêng lẻ. Từ đó, thời gian để hoàn tất xử lý số lượng lớn data sẽ giảm xuống đáng kể.

![image3](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/assets/images/concurrency.png)

Và ngoài việc áp dụng SQS Message Batch khi gửi nhận từ SQS, chúng ta có thể tăng tốc độ xử lý batch message trong 1 Lambda function bằng cách triển khai concurrency handler. Với 1 batch message nhận được từ SQS, Lambda function cũng sẽ xử lý đồng thời các message trong batch message. Lại 1 lần nữa, thời gian để hoàn tất xử lý data có thể giảm xuống nữa.

## **Áp dụng**
---

Để kiểm nghiệm giải pháp này, tôi sẽ build 1 ứng dụng example bằng Go, ứng dụng gồm 2 phần:
* Lambda cronjob: fetch data và gửi vào SQS
* Lambda SQS hanlder: nhận message từ SQS và tiến hành xử lý data

### **Lambda cronjob**
{% highlight go %}
func main() {
	lambda.Start(func() (string, error) {
		if err := run(nil); err != nil {
			return "ERROR", fmt.Errorf("ERROR: %+v", err)
		}
		return "OK", nil
	})
}
{% endhighlight %}

Với func _main()_, lambda cronjob sẽ được start khi có trigger từ EventBridge. Lambda sẽ xử lý bussiness logic được định nghĩa trong func _run()_.

func _run()_ sẽ thực hiện các bước sau:
* fetch data: func dùng để fetch data từ DB hoặc từ file, ở ví dụ này mình sẽ tạo 1 data sample trong func _fetchData()_:
{% highlight go %}
	listData, err := fetchData(ctx)
	if err != nil {
		return err
	}
{% endhighlight %}

* init SQS service: khởi tạo connection tới SQS
{% highlight go %}
	sqsHelper := helper.NewSQS(common.AWSRegion)
	urlRes, err := sqsHelper.GetQueueURL(common.SQSName)
	if err != nil {
		return err
	}
{% endhighlight %}

* group data theo batch và gửi tới SQS
{% highlight go %}
	// send batch message to SQS
	batchMessageData := make([]*sqs.SendMessageBatchRequestEntry, 0, common.MaximumSQSBatchMessage)
	for i, data := range listData {
		message := map[string]interface{}{
			"type":  "cronjob",
			"id":    data.ID,
			"value": data.Value,
		}

		jsonMsg, err := json.Marshal(message)
		if err != nil {
			return err
		}

		messageRequest := &sqs.SendMessageBatchRequestEntry{
			Id:          aws.String(strconv.FormatInt(int64(data.ID), 10)),
			MessageBody: aws.String(string(jsonMsg)),
		}

		batchMessageData = append(batchMessageData, messageRequest)

		// send batch message when batch size = maximum batch message or when user is the last of listData
		if len(batchMessageData) == common.MaximumSQSBatchMessage || i == (len(listData)-1) {
			sendMessageRequest := sqs.SendMessageBatchInput{
				QueueUrl: urlRes.QueueUrl,
				Entries:  batchMessageData,
			}

			_, err = sqsHelper.SendMessageBatch(&sendMessageRequest)
			if err != nil {
				return err
			}

			// remove all in batchMessageData to append next data
			batchMessageData = batchMessageData[:0]
		}
	}
{% endhighlight %}

### **Lambda SQS handler**
{% highlight go %}
func main() {
	lambda.Start(func(sqsEvent events.SQSEvent) (string, error) {
		err := run(sqsEvent)
		if err != nil {
			return "ERROR", fmt.Errorf("ERROR: %+v", err)
		}

		return "OK", nil
	})
}
{% endhighlight %}

Tương tự, với func _main()_, lambda handler sẽ được start khi có trigger từ SQS. Lambda sẽ xử lý bussiness logic được định nghĩa trong func _run()_.

func _run()_ sẽ thực hiện các bước sau:
* init SQS service:
{% highlight go %}
	sqsHelper := helper.NewSQS(common.AWSRegion)
	urlRes, err := sqsHelper.GetQueueURL(common.SQSName)
	if err != nil {
		return err
	}
{% endhighlight %}

* handle message get from SQS
{% highlight go %}
	processedReceiptHandles := make([]*sqs.DeleteMessageBatchRequestEntry, 0, len(sqsEvent.Records))
	for _, mess := range sqsEvent.Records {
		intf := make(map[string]interface{})
		if err := json.Unmarshal([]byte(mess.Body), &intf); err != nil {
			return err
		}

		valueType, ok := intf["type"]
		if ok {
			typeMessage := valueType.(string)
			switch {
			case strings.EqualFold(typeMessage, "cronjob"):
				fmt.Println("do process execute message")
			}
		}
		processedReceiptHandles = append(processedReceiptHandles, &sqs.DeleteMessageBatchRequestEntry{
			Id:            aws.String(mess.MessageId),
			ReceiptHandle: &mess.ReceiptHandle,
		})
	}
{% endhighlight %}

* delete message in SQS: sau khi message đã được xử lý, cần phải delete message trong SQS để tránh việc message bị xử lý duplicate 
{% highlight go %}
	if len(processedReceiptHandles) > 0 {
		deleteMessageRequest := sqs.DeleteMessageBatchInput{
			QueueUrl: urlRes.QueueUrl,
			Entries:  processedReceiptHandles,
		}

		// do not check DeleteMessageBatchOutput because SQS message max retry only 1
		_, err := sqsHelper.DeleteMessageBatch(&deleteMessageRequest)
		if err != nil {
			fmt.Println("failed to delete message batch with err: ", err)
			return err
		}
	}
{% endhighlight %}

### **Deploy**

Tiến hành cấu hình SQS và deploy code của 2 Lambda function lên môi trường AWS Serverless, ở bước deploy này bạn có thể tìm hiểu các hướng dẫn trên internet nhé.

Để thêm trigger cho Lambda cronjob, bạn có thể vào tab **Configuration** của Lambda cronjob function, chọn **Trigger** và chọn **Add trigger**. Ở đây tôi add trigger EventBridge với schedule expression mỗi 5 phút:

![image4](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/assets/images/evenbridge-schedule-expression.png)

Đừng quên cấu hình batch size của SQS cho Lambda SQS handler nữa nhé, bởi vì chúng ta đã send batch message vào SQS, Lambda handler cũng phải nhận batch message từ SQS. Sau khi cấu hình xong bạn sẽ thấy như sau:

![image5](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/assets/images/sqs_batch_size.png)

OK done. Sau 5 phút chờ trigger lambda cronjob, ta sẽ nhận được kết quả ở Lambda SQS handler như sau:

![image6](https://raw.githubusercontent.com/quantr247/go-lambda-sqs-example/master/assets/images/cloudwatch_log.png)

## **Tổng kết**
---

Với việc triển khai giải pháp như trên, chúng ta có thể đạt được lợi ích:
* Tăng tốc độ xử lý batch data trong hệ thống
* Tiết kiệm resources của hệ thống
* Tiết kiệm chi phí do giảm số lượng request gửi vào SQS cũng như giảm dung lượng của message

Tuy nhiên, đây chỉ là giải pháp chung cho các yêu cầu về xử lý batch data trong AWS Serverless. Tùy thuộc vào use case, cũng như yêu cầu cụ thể mà bạn cần điều chỉnh lại cho phù hợp với hệ thống của mình.

Chúc bạn làm việc hiệu quả :)

### **Bạn có thể tham khảo thêm tài liệu của AWS**
---

[Batch action](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-batch-api-actions.html)

[Send Batch Message API Document](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html)

[Amazon Simple Queue Service](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)