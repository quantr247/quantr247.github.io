---
layout: post
title:  "Serverless: Amazon API Gateway Throttling"
date:   2023-09-20 11:00:00 +0700
tags: serverless
categories: serverless
---
\
**Denial-of-service (DDOS) attack** chắn hẳn ai cũng đã biết về hình thức tấn công mạng này. Hình thức tấn công này sẽ làm cạn kiệt tài nguyên của hệ thống, khiến cho hệ thống không còn tài nguyên để phục vụ các request từ khách hàng được nữa.

Đặc biệt khi hệ thống của bạn sử dụng AWS Serverless, mỗi tài khoản chỉ có một số lượng Lambda concurrency limit. Chỉ cần tấn công DDOS vào một api, sẽ khiến tài khoản của bạn chạm tới ngưỡng của Lambda concurrency limit, từ đó các không thể invoke thêm được Lambda function nào nữa.

Và để ngăn chặn việc bị tấn công DDOS, mình sẽ chia sẻ với các bạn **Throttling** trong Amazon API Gateway.

## **API Gateway Throttling**
---
\
API Gateway throttles bằng cách sử dụng thuật toán token bucket, token được dùng để count số lượng request tới API. Có 2 giá trị sẽ được count để quyết định throttle request, đó là **request rate** per second (số lượng request / giây) và **burst** request (số lượng concurrent request). Khi API bị throttled, clients sẽ nhận được response HTTP status code _429 Too Many Requests_.

Có 4 loại, level config throttles cơ bản như sau:
* _AWS throttling limits_ được applied cho toàn bộ tài khoản và client trong một region. Đây là giá trị được set bởi AWS và bạn không thể thay đổi.
* _Per-account_ được applied cho toàn bộ API của một tài khoản trong một region. Bạn có thể config giá trị nhưng không thể lớn hơn _AWS throttling limits_.
* _Per-API, per-stage_ được applied cho API method level cho mỗi stage (môi trường).
* _Per-client_ được applied cho client nào gọi API với API keys. Giá trị config không thể lớn hơn per-account limits.

Khi bạn deploy một api lên API Gateway, throttling mặc định sẽ được **enable**. Tuy nhiên, giá trị default là: 10000 requests/second với burst là 5000 cho tài khoản của bạn. Bạn cần monitor metrics của hệ thống để có được giá trị **rate** và **burst** phù hợp, và nhất là cần phải monitor và điều chỉnh kịp thời khi số lượng user/request của hệ thống có sự tăng trưởng trong tương lai.

## **Serverless Framework - Serverless API Gateway Throttling plugin**
---
\
Bạn có thể config giá trị **rate** và **burst** trên AWS console, hoặc có thể dùng các công cụ IasC để deploy lên AWS. Mình sẽ sử dụng **Serverless framework** để tiến hành deploy và demo throttling trong API Gateway.

* Để cấu hình throttling bằng Serverless framework, trước tiên cần cài đặt plugin **Serverless API Gateway Throttling**:
    ```
        npm i serverless-api-gateway-throttling
    ```

* Add plugin trong file serverless.yml
    ```
        plugins:
          - serverless-api-gateway-throttling
    ```

* Add giá trị config vào custom để cấu hình throttling cho toàn bộ api
    ```
        custom:	
          # Configures throttling settings for the API Gateway stage
          # They apply to all http endpoints, unless specifically overridden
          apiGatewayThrottling:
            maxRequestsPerSecond: 2
            maxConcurrentRequests: 1
    ```

* Để set throttling cho mỗi api method riêng, ta có thể thêm cấu hình ví dụ như sau:
    ```
        # Requests are throttled using this endpoint's throttling configuration
        list-all-items:
            handler: rest_api/items/get/handler.handle
            events:
            - http:
                path: /items
                method: get
                throttling:
                    maxRequestsPerSecond: 2
                    maxConcurrentRequests: 1
    ```

Sau khi deploy lên AWS, sẽ có kết quả như sau trong API Gateway Settings console

![image1](/assets/20230920/api-gateway-throttling-settings.png)

Và để test thử, mình sẽ run test performance bằng postman. Bạn sẽ thấy kết quả là tại những thời điểm có 3 request/giây, sẽ có Error code là 429 Too many requests.

![image2](/assets/20230920/api-gateway-throttling-test.png)

## **Tổng kết**
---
\
Với việc triển khai giải pháp như trên, chúng ta có thể đảm bảo security cho hệ thống, ngăn chặn việc bị tấn công dẫn tới cạn kiệt tài nguyên của hệ thống. Tùy thuộc vào use case, cũng như yêu cầu cụ thể mà bạn cần điều chỉnh lại cho phù hợp với hệ thống của mình.

Chúc bạn làm việc hiệu quả :)

### **Bạn có thể tham khảo thêm tài liệu của AWS**
---

[AWS API Gateway Throttling](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)

[Serverless Framework - Serverless API Gateway Throttling plugin](https://www.serverless.com/plugins/serverless-api-gateway-throttling)