---
layout: post
title:  "Ubuntu: Set up Docker on Ubuntu Server"
date:   2023-04-11 11:00:00 +0700
tags: ubuntu
categories: ubuntu
---
\
Docker là 1 nền tảng cho phép chúng ta vận hành, triển khai ứng dụng theo cách container hóa. Với việc mỗi ứng dụng sẽ được chạy trong 1 môi trường container độc lập, không phụ thuộc vào nhau.

## **Install Docker on Ubuntu server**

1. Đầu tiên, cần update các packages hiện tại của Ubuntu server
```
    sudo apt update
    sudo apt upgrade
```

2. Install các packages cần thiết cho HTTPS
```
    sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

3. Tiếp theo, ta sẽ add GPG key cho repository chính thức của Docker
```
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

4. Và ta cần add Docker repository vào APT sources
```
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```

5. Cuối cùng, ta sẽ install Docker
```
    sudo apt install docker-ce
```

6. Kiểm tra lại xem Docker đã được cài đặt chưa
```
    sudo systemctl status docker
```

7. Để tránh việc phải nhập **sudo** khi chạy các câu lệnh **docker**, ta sẽ add username vào docker group
```
    sudo usermod -aG docker ${USER}
```

8. Để apply cần chạy câu lệnh sau
```
    su - ${USER}
```

9. Giờ bạn có thể log out và log in lại Ubuntu server, sau đó check group bằng câu lệnh sau
```
    groups
```

## **Tổng kết**
---

Trên đây tôi đã hướng dẫn các bạn cài đặt Docker trên Ubuntu server, hi vọng sẽ giúp ích cho các bạn trong quá trình làm việc của mình.

Chúc bạn làm việc hiệu quả :)

### **Bạn có thể tham khảo thêm tài liệu của AWS**
---
\
[Install Docker on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)
