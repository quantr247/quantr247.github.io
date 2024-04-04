---
layout: post
title:  "Go: Phân biệt giữa data race và race conditions trong race problem"
date:   2025-03-10 11:00:00 +0700
tags: golang
categories: golang
---

## **Data race**
---


## **Race condition**
---

## **Tổng kết**
---
\
Data race: 
* 2 goroutine cùng truy cập vào 1 biến dữ liệu và tăng giá trị của biến đó lên. Có thể giải quyết data race bằng mutex. 
* 2 goroutine cùng truy cập vào 1 column data trong database, và cùng tăng giá trị của column đó lên. Bài toán này có thể giải quyết bằng SetNX trong Redis. Tức là data race có thể giải quyết bằng việc 1 trong 2 goroutine, bên nào tới trước xử lý trước, bên nào tới sau xử lý tiếp từ của bên tới trước, không ảnh hưởng tới kết quả.
\
Race condition: 
* 2 goroutine cùng truy cập vào 1 biến dữ liệu và set giá trị có biến đó, tức là cho dù có dùng mutex cũng không giải quyết đc vấn đề. Ví dụ có 1 biến i, goroutine 1 set i = 1, goroutine 2 set i = 2. Ở đây nếu có mutex chỉ giải quyết tranh chấp giữa 2 goroutine, 1 trong 2 sẽ xử lý trước, chứ mutex không giải quyết được vẫn đề là có lúc i = 1, và có lúc i = 2.
* Ví dụ user thao tác trên FE, chọn step 1 --> lưu (1), chọn step 2 --> lưu (2). Do network chập chờn, vô tình 2 request (1) và (2) được gửi tới server đồng thời. Và server xử lý song song 2 request này để lưu vào cùng 1 table trong DB, dẫn tới request (2) có thể lưu vào trong table trước request (1). Và bài toán này cũng không thể sử dụng SetNX để giải quyết như data race được.