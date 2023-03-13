---
layout: post
title:  "Ubuntu: Extend dung lượng bộ nhớ Logical Volume (Actual size) cho Ubuntu Server"
date:   2023-03-12 11:00:00 +0700
tags: ubuntu
categories: ubuntu
---
\
Trước khi thực hiện extend bộ nhớ, bạn sử dụng command Volume Group Display để check dung lượng bộ nhớ:

>sudo vgdisplay

![image1](https://static1.makeuseofimages.com/wordpress/wp-content/uploads/2022/09/logical_volume_vg_display_info.jpg?q=50&fit=crop&w=1500&dpr=1.5)

Có 3 thông số quan trọng bạn cần chú ý tới:
* VG Size: Đây là volume group size và nó thể hiện tổng dung lượng bộ nhớ trong disk. 23GB
* Alloc PE/Size: Thể hiện dung lượng bộ nhớ Ubuntu được cấp phát để sử dụng. 11.5GB
* Free PE/Size: Phần dung lượng free, chưa được sử dụng trong tổng số dung lượng 23GB

Ngoài ra, bạn có thể check thêm với command sau:

>df -h

![image2](https://static1.makeuseofimages.com/wordpress/wp-content/uploads/2022/09/ubuntu_df_command_display_info-1.jpg?q=50&fit=crop&w=1500&dpr=1.5)

Câu lệnh df chỉ thể hiện dung lượng bộ nhớ được cấp phát cho Ubuntu, bạn có thể thấy ở cột Size. Ở đây là 12GB, sấp xỉ với Alloc FE/Size là 11.5GB.

Muốn extend dung lượng bộ nhớ, ta sẽ sử dụng câu lệnh **lvextend** để cấp phát vùng bộ nhớ Free cho vùng Alloc. Các bước như sau:

* Sử dụng câu lệnh sau để kiểm tra đường dẫn tới logical volume:
>sudo lvdisplay

![image3](https://static1.makeuseofimages.com/wordpress/wp-content/uploads/2022/09/lv_display_command_on_linux.jpg?q=50&fit=crop&w=1500&dpr=1.5)

* Từ bước trên bạn thấy đường dẫn: _/dev/ubuntu-vg/ubuntu-lv_. Tiếp tục dùng câu lệnh sau:
>sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

* Cuối cùng ta cần resize để effect phần bộ nhớ mới được cấp phát:
>resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

OK! DONE. Bạn có thể run câu lệnh df -h để check lại nhé.

## **Tổng kết**
---
\
Ở bài post này, tôi đã hướng dẫn bạn extend dung lượng bộ nhớ logical volume cho Ubuntu server. Hi vọng nó sẽ giúp ích cho các bạn, đặc biệt trong trường hợp sử dụng máy ảo để chạy Ubuntu server trên VM Ware hay VM Virtualbox.

Chúc bạn làm việc hiệu quả :)

## **Thông tin tham khảo**
---
\
[How to extend Logical Volumes on Ubuntu Server](https://www.makeuseof.com/extend-logical-volumes-lvm-ubuntu-server/)