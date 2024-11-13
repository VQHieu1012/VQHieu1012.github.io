---
title: "Database Internal #1: Pages và Files trong Database"
description: "Cách pages và files trong database hoạt động"
author: qhieu
date: 2024-11-13 00:00:00 +0800
categories: [Note]
tags: [database-internal]
pin: true
math: true
mermaid: true
image:
  path: ../images/database_internal_1/background.avif
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  # alt: None
---

# Overview

Các database được thiết kế theo Disk-based Architecture. Nghĩa là chúng ta muốn data được lưu trữ vào non-volatile disk (rút điện và data vẫn được lưu trữ, ví dụ như HDD, SSD...). Còn các thành phần của DBMS (Database Management System khác Database) sẽ quản lý việc di chuyển data qua lại giữa các non-volatile và volatile storage (kiểu lưu trữ mà mất điện là tạch hết data, ví dụ như lưu trên memory ở Ram...). Làm sao để đưa data từ disk lưu trên memory hiệu quả, CPU tính toán, trả về kết quả, hay làm sao để ghi dữ liệu vào memory và chắc chắn rằng data được sync lại đầy đủ vào disk... Đó là việc mà các DBMS cần giải quyết.

### Phân cấp trong lưu trữ

![image.png](https://images.viblo.asia/7b4a1794-ab3e-47e5-b8da-ea60cf0587e9.png)

Trong phân cấp lưu trữ này, các CPU registers có tốc độ truy xuất nhanh nhất, tiếp theo là CPU cache, DRAM và các Disk. Nhìn từ dưới đi lên, nói chung là đắt thì nhanh, nhanh thì lại nhỏ.

* Random Access (khả năng truy xuất): truy cập trực tiếp vào bất kỳ ô dữ liệu nào mà không cần duyệt qua các dữ liệu ở vị trí trước. Chúng ta có thể đi tới bất kỳ địa chỉ nào trong bộ nhớ mà không cần đọc tuần tự hoặc theo bất kỳ một trật tự xác định nào. Random access có lợi thế khi cần truy xuất dữ liệu nhanh chóng từ nhiều vị trí khác nhau. Ví dụ anh em có 1 cuốn từ điển và 10 trang sách bất kỳ, mỗi lần mở từ điển là tới ngay trang cần tìm -> nhanh.
* Sequential Access (khả năng truy xuất): dữ liệu được đọc theo tuần tự. Ví dụ chúng ta cần tìm đến trang 25, anh em sẽ lật từng trang một cho tới khi đến trang 25.
* Byte-Addressable (khả năng lưu trữ): dữ liệu có thể được lấy theo đơn vị bytes. Ví dụ chúng ta có 1MB data, đưa nó vào memory và chúng ta có thể lấy 64 bits. Hoặc trong cuốn từ điển có rất nhiều từ, nếu mỗi từ lưu ở 1 ô nhớ, chúng ta có thể lấy được từ đó (khác với Block addressable khi chúng ta chỉ có thể lấy cả trang chứa từ đó).
* Block-Addressable (khả năng lưu trữ): dữ liệu được lưu trữ theo từng khối (block) với cỡ cố định. Mỗi khối có một địa chỉ riêng và chứa được nhiều byte. Ví dụ chúng ta có thể chia dữ liệu thành các khối 4KB (hoặc 8KB hoặc tùy), mỗi khi truy xuất dữ liệu, chúng ta sẽ lấy cả khối 4KB đó. Ví dụ với cuốn từ điển nhiều từ được chia thành nhiều trang, để lấy được 1 từ, chung ta không thể lấy luôn từ đó, mà cần lấy cả trang chứa từ đó, đưa nó vào memory và lấy từ cần lấy.

### Tốc độ truy cập

![image.png](https://images.viblo.asia/347de74e-96bb-4d1e-b436-9cdc30db8748.png)

Có một sự khác biệt lớn giữa tốc độ truy cập của từng loại phân cấp trong lưu trữ. Chính vì thế mà chúng ta sẽ tối đa hóa việc sử dụng DRAM vì chúng nhanh hơn rất nhiều so với Disk (tuy nhiên thì không phải lúc nào cũng có thể làm thế).

### Sequential và Random Access

Một tính chất nữa khi thiết kế DBMS là Sequential Access và Random Access. Trên lý thuyết khi chúng ta giả sử data luôn nằm trong memory, điều này khiến chúng ta quên đi cách mà data thực sự được truy xuất và sử dụng trong việc tính toán. Tuy nhiên trong các thiết bị lưu trữ, cách data được truy xuất như thế nào và theo phương thức nào cực kỳ quan trọng.

Việc sử dụng Random Access trong các non-volatile storage sẽ chậm hơn đáng kể so với Sequential Access. Và DBMS muốn data được lưu ở non-volatile storage nên mục tiêu của chúng ta là tối đa hóa Sequential Access ghi đọc hoặc ghi data từ Disk. Các thuật toán được sử dụng cần tối thiểu số lần ghi data vào các random pages &rarr; data được lưu ở các block dữ liệu gần kề nhau. Ví dụ MySQL sẽ ghi các dirty pages (là gì thì nói sau) từ memory vào side buffer (bộ đệm phụ) trước theo kiểu Sequential Write (để tối ưu hóa việc ghi vào Disk) sau đó đẩy data vào Disk. Cuối cùng, các physical pages trong Disk được update (có thể là ở các địa chỉ ngẫu nhiên). Như vậy, MySQL ghi dữ liệu 2 lần, lần 1 là Sequential Write (nhanh) vào side buffer, lần 2 là ghi vào Disk (có thể là random write) .Ngoài ra, chúng ta có thể phân bổ nhiều pages cùng lúc để các block dữ liệu có thể nằm gần kề nhau (tuyệt nhất là khi chúng liên tục, đọc và ghi data đều nhanh). Quá trình trên được gọi là "extent".

### Mục tiêu thiết kế hệ thống DBMS
* Cho phép DBMS có thể quản lý database kể cả khi lượng memory của database vượt quá dung lượng memory khả dụng. (Virtual memory có phải phương án tốt?)
* Random Access trên Disk chậm hơn nhiều so với Sequential Access, chính vì thế mà DBMS cần tối ưu hóa các sequential access.
* Việc đọc/ghi trên đĩa rất tốn kém tài nguyên, nhưng chúng ta cũng có một số trick trong thiết kế hệ thống để tránh gián đoạn (thường do lượng IO cao) khi làm việc với Disk và tránh suy giảm hiệu năng.

### Disk-oriented DBMS

* Hiểu đơn giản thì database chỉ là các files được lưu trữ trong disk, cũng không có gì quá đặc biệt. Các data file được lưu trữ trong Disk, mỗi data file lại được chia thành nhiều data pages với độ lớn bằng nhau. Bằng việc phân chia các data file thành các pages bằng nhau, chúng ta có thể nhanh chóng tiếp cận tới bất kỳ offset nào 
* Buffer Pool (hoặc Buffer Pool Manager, Cache Manager... mỗi kiểu DBMS lại thích đặt 1 kiểu): đây là vùng nhớ trên memory được quản lý bởi DBMS dùng để cache các data pages.

(dài quá sau viết tiếp)
