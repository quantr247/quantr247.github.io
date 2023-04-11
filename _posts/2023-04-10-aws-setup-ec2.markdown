---
layout: post
title:  "AWS: Set up EC2 Ubuntu instance bằng AWS Console"
date:   2023-04-10 11:00:00 +0700
tags: aws
categories: aws
---
\
EC2 là một dịch vụ computing cloud của AWS. Thay vì phải mua và set up thiết bị phần cứng cho server vật lý, EC2 giúp bạn có các computing với khả năng scalable. Bạn có thể sử dụng EC2 để tạo các virtual server tăng giảm một cách linh động, đáp ứng nhu cầu thay đổi nhanh chóng của requirement.

## **Set up an instance**
---
\
Để set up và launch được EC2 instance, bạn thực hiện theo các bước sau:

1. Truy cập vào Amazon EC2 console: [EC2 console](https://console.aws.amazon.com/ec2/)
2. Trong mục **Instances** ở cột bên trái, chọn **Instances** --> Xuất hiện trang **Instances** dashboard, chọn **Launch instances**
3. Trong phần **Name and tags**, nhập tên của EC2 instances
4. **Application and OS Images**
    * Chọn tab **Quick Start**
    * Chọn **Ubuntu**
    * **Amazon Machine Image (AMI)** chọn Ubuntu Server 22.04
    * **Architecture** chọn **64-bit (x86)**
5. Trong phần **Instance type** chọn **t2.medium**
6. Trong phần **Key pair**, chọn **Create new key pair** để tạo cặp key mới
    * Nhập tên cho key pair name
    * Chọn **Key pair type** là **RSA**
    * Chọn **Private key file format** là **.pem**
    * Chọn **Create key pair**, lưu file .pem lại nhé.
    * Chọn key pair name đã được tạo trong drop list
7. **Network settings**: tạo security groups với tên là **SG for EC2 Ubuntu** dành cho EC2 instance từ mục **Security Groups**, với **Inbound rules** cho phép port 22 để SSH
    * Enbale **Auto-asign public IP** 
    * Chọn **Select existing security group**
    * List **Security groups** chọn **SG for EC2 Ubuntu**
8. **Configure storage** nhập 30GiB
9. Chọn **Launch instance**

## **Access EC2 instance và tạo user/pass**
---
\
Sau khi EC2 instance đã được launch, chúng ta cần ssh access vào EC2 để tiến hành cài đặt cho Ubuntu server trên EC2. 
Trước tiên, ta cần có public IP của instance. Trong **Instances** dashboard của EC2, bạn chọn vào instance đã tạo và sẽ thấy **Public IPv4 addess**.
Tiếp theo, ta sẽ ssh vào Ubuntu instance này từ máy tính local.

1. Mở **Terminal** trên laptop/PC
2. Chạy câu lệnh để truy cập vào EC2 instance: 
    >ssh -i ubuntu.pem ubuntu@[Public IPv4 addess]
    * ubuntu.pem: chính là file pem có được từ bước Generate key pair ở trên
    * user: ubuntu là user mặc định của Ubuntu instance
3. Tạo user để access bằng user và password

    >cd /etc/ssh\
    sudo vim sshd_config\
    Thay đổi giá trị của **PasswordAuthentication** thành **Yes**\
    >sudo service ssh restart\
    sudo adduser ec2ubuntu\
    Nhập password cho user ec2ubuntu

4. Mở **Terminal** mới trên laptop/PC và truy cập ssh vào **Public IPv4 addess** bằng user/pass đã tạo ở bước 3

## **Tổng kết**
---
\
Từ EC2 Ubuntu instances đã tạo, bạn có thể cài đặt thêm các phần mềm/ứng dụng để đáp ứng nhu cầu công việc của mình nhé.

Chúc bạn làm việc hiệu quả :)

### **Bạn có thể tham khảo thêm tài liệu của AWS**
---
\
[Amazon EC2 Linux instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
