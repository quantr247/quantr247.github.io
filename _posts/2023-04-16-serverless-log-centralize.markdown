---
layout: post
title:  "Serverless: (Go) Triển khai log tập trung cho AWS Lambda với giải pháp ship log từ Cloudwatch về Loki"
date:   2023-04-16 11:00:00 +0700
tags: serverless
categories: serverless
---
\
Khi triển khai các ứng dụng chạy trên AWS Lambda, chúng ta có thể sử dụng Cloudwatch để xem log cũng như trace các request invoke vào Lambda. Tuy nhiên, khi sử dụng Cloudwatch bạn sẽ gặp một số vấn đề như:
* Log được chia thành nhiều log stream, khó khăn trong việc xem log
* Logs Insights không hỗ trợ search với text bất kì, câu query khó sử dụng
* Khó khăn trong việc phân quyền user được phép xem log

Để giải quyết nhưng vấn đề trên, chúng ta có thể sử dụng giải pháp log tập trung như ELK hay Loki, Grafana. Trong bài post này, tôi sẽ chia sẻ cách triển khai log tập trung cho AWS Lambda với Loki và xem log trên Grafana. Khái quát mô hình mà tôi sẽ triển khai như sau

![image1](/assets/20230416/lambda-loki.png)

Tôi sẽ tạo ra 1 EC2 instance, mục đích để run Grafana và Loki trên EC2 instance này. Grafana sẽ lấy datasource từ chính Loki, và log sẽ được ship từ Cloudwatch tới Loki bằng 1 Lambda function đóng vai trò là log shipper.

Ở các bài post trước, tôi đã chia sẻ về cách cài đặt EC2 instance cũng như cài đặt Docker:\
[Cài đặt EC2 Ubuntu]({% post_url 2023-04-10-aws-setup-ec2 %})\
[Cài đặt Docker trên Ubuntu]({% post_url 2023-04-11-ubuntu-setup-docker %})

Tiếp theo, tôi sẽ chia sẻ cách cài đặt Grafana, Loki cũng như tích hợp với AWS Cloudwatch SDK để ship log từ Cloudwatch tới Loki.

## **Cài đặt Grafana trong Docker**

Tạo persistent storage
```
    docker volume create grafana-storage
```

Start grafana từ image open source office
```
    docker run -d -p 3000:3000 --name=grafana -v grafana-storage:/var/lib/grafana grafana/grafana-oss
```

## **Cài đặt Loki trong Docker**
Tạo persistent storage data
```
    docker volume create loki-data
```

Start grafana từ image open source office
```
    docker run -d --name=loki --mount source=loki-data,target=/loki -p 3100:3100 grafana/loki
```

## **Security** 

Do Docker chạy trong EC2 instance, nên cần phải thêm port 3000 và 3100 vào **Inbound rules** của **Security** để có thể truy cập được Grafana và Loki.

![image2](/assets/20230416/ec2_port.png)

Đến đây bạn đã có thể truy cập vào Grafana và Loki qua Public IPv4 DNS của EC2 và port đã cấu hình:

Grafana: http://ec2-xx-2xx-3xx-4xx.region.compute.amazonaws.com:3000

Loki: http://ec2-xx-2xx-3xx-4xx.region.compute.amazonaws.com:3100

## **Log Shipper Lambda Function**

Để ship log từ Cloudwatch về Loki, tôi sẽ sử dụng **Subscription filter** của Cloudwatch để invoke Lambda function, khi đó các CloudWatch LogEvents sẽ được gửi tới Lambda function, và event message sẽ được push tới Loki trong Lambda function này.

OK giờ ta sẽ code cho Log Shipper Lambda function.

Đầu tiên, ta cần code để start lambda function trong hàm main với event là CloudwatchLogsEvent.

{% highlight go %}
package main

import (
	"context"
	"fmt"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func main() {
	lambda.Start(func(ctx context.Context, event events.CloudwatchLogsEvent) (string, error) {
		err := Run(ctx, event)
		if err != nil {
			return "ERROR", fmt.Errorf("ERROR: %+v", err)
		}

		return "OK", nil
	})
}
{% endhighlight %}

Tiếp theo, func Run để decode message từ LogsEvent Data.

{% highlight go %}
func Run(ctx context.Context, event events.CloudwatchLogsEvent) error {
	fmt.Println("Loki shipper start.....")
	// Decode base 64 of Logs data
	decodedData, _ := event.AWSLogs.Parse()
	err := ProcessLog(ctx, decodedData)
	if err != nil {
		return err
	}

	return nil
}
{% endhighlight %}

Tại func ProcessLog, tôi sẽ parse log group name để lấy label cho Loki, cũng như filter các event message nào là json để push sang Loki.

{% highlight go %}
func ProcessLog(ctx context.Context, event events.CloudwatchLogsData) error {
	if len(event.LogEvents) == 0 {
		return fmt.Errorf("LogEvents is empty")
	}

	var (
		label, labelValue string
		lokiMessages      []*LokiMessage
	)

	for _, le := range event.LogEvents {
		label, labelValue = getLabelFromLogGroupName(event.LogGroup)
		if json.Valid([]byte(le.Message)) || strings.Contains(le.Message, "panic") {
			lokiMessages = append(lokiMessages, &LokiMessage{
				Timestamp: le.Timestamp * 1000000,
				Message:   le.Message,
			})
		}
	}

	if len(lokiMessages) > 0 {
		if err := pushLoki(ctx, lokiMessages, label, labelValue); err != nil {
			fmt.Println(fmt.Sprintf("failed to push log to Loki: %v", err))
		}
	}
	return nil
}

func getLabelFromLogGroupName(lg string) (svcName, funcName string) {
	if strings.Contains(lg, "/") {
		splittedLg := strings.Split(lg, "/")
		funcName = splittedLg[len(splittedLg)-1]
		if strings.Contains(funcName, "-") {
			splitedName := strings.Split(funcName, "-")
			svcName = splitedName[0]
		}
	}
	return
}
{% endhighlight %}

Và cuối cùng, ta cần 1 func để push log event message tới Loki.

{% highlight go %}
func pushLoki(ctx context.Context, messages []*LokiMessage, label, labelValue string) error {
	lokiURL := LokiURL + "api/v1/push"

	// append message to array values to push Loki 
	var values []*[]string
	for _, msg := range messages {
		v := []string{
			fmt.Sprintf("%v", msg.Timestamp), msg.Message,
		}
		values = append(values, &v)
	}

	postBody, _ := json.Marshal(map[string]interface{}{
		"streams": []map[string]interface{}{
			{
				"stream": map[string]interface{}{
					label: labelValue,
				},
				"values": values,
			},
		},
	})

	requestBody := bytes.NewBuffer(postBody)
	resp, err := http.Post(lokiURL, "application/json", requestBody)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	return nil
}
{% endhighlight %}

## **Add permission cho Log Shipper Lambda**

Để có thể invoke được Log Shipper Lambda function, cần phải add permission cho Lambda function. Bạn có thể sử dụng script sau trên AWS CLI để add permission

```
aws lambda add-permission \
    --function-name "logshipper" \
    --statement-id "helloworld" \
    --principal "logs.amazonaws.com" \
    --action "lambda:InvokeFunction" \
    --source-arn "arn:aws:logs:region:123456789123:log-group:TestLambda:*" \
    --source-account "123456789012"
```

Ngoài ra, trong project của mình thì tôi sử dụng **[Serverless framework](https://www.serverless.com/framework/docs/providers/aws/guide/resources)** để deploy, nên tôi sẽ sử dụng script sau để add resources trong file serverless.yaml

```
resources:
  Resources:
    PermissionLog:
      Type: AWS::Lambda::Permission
      Properties:
        Action: "lambda:InvokeFunction"
        FunctionName: logshipper
        Principal: "logs.amazonaws.com"
        SourceAccount: "123456789012"
```

Sau khi add permissions thành công, ta có thể thấy ở tab **Permissions** trong **Configuration** của Lambda function như sau

![image3](/assets/20230416/lambda-permissions.png)

## **Add subscription filter cho log group**

Cuối cùng, ta cần add **Subscription filter** vào log group để invoke Log shipper Lambda function và gửi log event mỗi khi có event message được thêm vào trong log group.

Bạn có thể sử dụng script sau trên AWS CLI để tạo subscription filter cho log group

```
aws logs put-subscription-filter \
    --log-group-name myLogGroup \
    --filter-name demo \
    --filter-pattern "" \
    --destination-arn arn:aws:lambda:region:123456789123:function:logshipper
```

Tương tự, tôi cũng sẽ add resources cho serverless.yaml cho những log group nào cần ship tới Loki

```
resources:
  Resources:
    SubscriptionFilter:
      Type: AWS::Logs::SubscriptionFilter
      Properties:
        LogGroupName: "logGroupName"
        FilterPattern: ""
        DestinationArn: "arn:aws:lambda:region:123456789123:function:logshipper"
```

![image4](/assets/20230416/subscription-filter.png)

[**Full source code**](https://github.com/quantr247/go-lambda-logshipper)

DONE! Tới đây các bạn có thể vào Grafana để trace log của Lambda thoải mái rồi nhé.

## **Tổng kết**
---

Trong bài post này, tôi đã chia sẻ với các bạn cách triển khai log tập trung trên Loki cho Cloudwatch với giải pháp dùng Log Shipper Lambda function. Ngoài ra còn có các giải pháp khác với các công nghệ khác, các bạn có thể thử nghiệm và so sánh với giải pháp này để tìm ra cho mình 1 giải pháp phù hợp nhất với hệ thống.

Chúc bạn làm việc hiệu quả :)

### **Tài liệu tham khảo**
---
\
[Grafana documentation](https://grafana.com/docs/grafana/latest/)

[Grafana Loki documentation](https://grafana.com/docs/loki/latest/)

[AWS Cloudwatch SDK for Go](https://docs.aws.amazon.com/sdk-for-go/api/service/cloudwatchlogs/)

[Subscription Filter Lambda Function](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html#LambdaFunctionExample)
