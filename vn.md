
[Nguồn](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/ "Permalink to MySQL 5.7 Performance Tuning Immediately After Installation")

# Tùy chỉnh hiệu năng MySQL 5.7 ngay sau khi cài đặt

Blog này cập nhật từ blog của Stephane Combaudon về điều chỉnh hiệu năng của MySQL] [1] và tổng quát hóa về vấn đề tùy chỉnh hiệu năng của MySQL 5.7 ngay sau khi cài đặt.

Cách đây 1 năm, Stephane Combaudon đã viết một bài đăng trên blog về các cài đặt điều chỉnh hiệu suất của Ten MySQL sau khi cài đặt, bao gồm cả các phiên bản trước đây (hiện tại) của MySQL như: 5.1, 5.5 và 5.6.

Tin tốt là MySQL 5.7 có giá trị mặc định tốt hơn rất nhiều. Morgan Tocker đã tạo một trang với danh sách đầy đủ các tính năng trong MySQL 5.7, và là một nguồn tham khảo tuyệt vời. Ví dụ, các biến dưới đây sẽ được thiết lập các giá trị mặc định: 

* innodb_file_per_table=ON
* innodb_stats_on_metadata = OFF
* innodb_buffer_pool_instances = 8 (or 1 if innodb_buffer_pool_size < 1GB)
* query_cache_type = 0; query_cache_size = 0; (vô hiệu hóa mutex)

Trong MySQL 5.7, chỉ có bốn biến thực sự quan trọng cần được thay đổi. Tuy nhiên, có các biến khác của InnoDB và MySQL toàn cục có thể cần được điều chỉnh riêng cho một phần công việc và phần cứng cụ thể.

Để bắt đầu, hãy thêm các cài đặt sau vào file my.cnf trong phần [mysqld]. Bạn sẽ cần phải khởi động lại MySQL:

```
    [mysqld]
    # other variables here
    innodb_buffer_pool_size = 1G # (điều chỉnh giá trị tại đây, chiếm 50%-70% dung lượng RAM)
    innodb_log_file_size = 256M
    innodb_flush_log_at_trx_commit = 1 # may change to 2 or 0
    innodb_flush_method = O_DIRECT
```
Description:

| ----- |
| **Variable** |  **Value** |  
| innodb_buffer_pool_size |  Khởi động với 50% đến 70% dung lượng RAM. Không cần phải lớn hơn kích thước của cơ sở dữ liệu |  
| innodb_flush_log_at_trx_commit | 

* 1   (Mặc định)
* 0/2 (hiệu suất cao hơn, độ tin cậy thấp hơn)
 |  
| innodb_log_file_size |  128M – 2G (không cần phải lớn hơn bộ nhớ đệm) |  
| innodb_flush_method |  O_DIRECT (avoid double buffering) | 

_**Điều gì chờ đón tiếp theo?**_

Đó là điểm khởi đầu tốt cho bất kỳ việc cài đặt mới nào. Có một số biến khác có thể tăng hiệu năng MySQL trong một số công việc. Thông thường, tôi sẽ thiết lập một công cụ giám sát / vẽ đồ thị MySQL (ví dụ, [Percona Monitoring and Management platform]) và sau đó kiểm tra bảng điều khiển MySQL để thực hiện điều chỉnh thêm.

_**Những gì chúng ta có thể điều chỉnh thêm dựa trên các đồ thị?**_

_Kích thước vùng đệm của InnoDB_. Hãy xem đồ thị sau:

![MySQL 5.7 Performance Tuning][4]

![MySQL 5.7 Performance Tuning][5]

Như chúng ta có thể thấy, chúng ta có thể có lợi từ việc tăng kích thước vùng đệm InnoDB một chút lên ~ 10G, vì chúng ta có RAM và số trang miễn phí nhỏ so với tổng số vùng đệm.

_Kích thước file log InnoDB._. Hãy xem đồ thị sau:

![MySQL 5.7 Performance Tuning][6]

Như chúng ta có thể thấy ở đây, InnoDB thường ghi 2,26 GB dữ liệu mỗi giờ, vượt quá tổng kích thước của các tệp nhật ký (2G). Bây giờ chúng ta có thể tăng giá trị biến _innodb_log_file_size_ và khởi động lại MySQL. Ngoài ra, hãy sử dụng “công cụ hiện thị trạng thái InnoDB” để tính toán kích thước tệp nhật ký InnoDB cho phù hợp nhất.

_**Những biến khác**_

Có một số biến InnoDB khác có thể được điều chỉnh thêm:

_innodb_autoinc_lock_mode_

Thiết lập giá trị của _innodb_autoinc_lock_mode_ = 2 (chế độ xen kẽ) có thể loại bỏ sự cần thiết của khóa AUTO-INC ở mức bảng (và có thể tăng hiệu suất khi các câu lệnh chèn nhiều hàng được sử dụng để chèn các giá trị vào các bảng với khóa chính auto_increment). Điều này đòi hỏi phải gán  _binlog_format_ = _ROW_ hoặc _MIXED_ (và ROW là mặc định trong MySQL 5.7)

_innodb_io_capacity _và_ innodb_io_capacity_max_

Đây là một điều chỉnh nâng cao hơn, và chỉ có ý nghĩa khi bạn đang thực hiện một công việc write mà thôi (nó không áp dụng cho các lần đọc, ví dụ: SELECT). Ví dụ, nếu máy chủ có một ổ SSD, chúng ta có thể gán giá trị _innodb_io_capacity_max_ = _6000_ và _innodb_io_capacity_ = _3000_ (Bằng một nữa so với giá trị max). Đó là một ý tưởng tốt để chạy sysbench hoặc bất kỳ công cụ benchmark khác nào để chuẩn hóa thông lượng đĩa.

Nhưng liệu chúng ta có cần phải bận tâm về cài đặt này không? Hãy xem biểu đồ "các trang rác" của vùng đệm:

![screen-shot-2016-10-03-at-7-19-47-pm][10]

Trong trường hợp này, tổng số trang rác rất là cao, và có vẻ như InnoDB không thể loại bỏ chúng kịp thời. Nếu chúng ta có hệ thống ổ đĩa phụ mạnh (ví dụ:  SSD), chúng ta có thể hưởng lợi bằng cách tăng giá trị của _innodb_io_capacity_ và _innodb_io_capacity_max_.

_**Kết luận hay version TL;DR**_

Các mặc định mới trong MySQL 5.7 tốt hơn nhiều cho các công việc nhằm mục đích chung. Đồng thời, chúng ta vẫn cần cấu hình biến InnoDB để tận dụng dung lượng RAM. Sau khi cài đặt, hãy làm theo các bước sau:

1. Thêm biến InnoDB vào my.cnf (như mô tả ở trên) và khởi động lại MySQL.
2. Cài đặt hệ thống giám sát, (ví dụ: Percona Monitoring và Management platform)
3. Nhìn vào biểu đồ và xác định xem liệu MySQL có cần được điều chỉnh thêm hay không.

### More resources:

#### Posts

#### Webinars

#### Presentations

#### Free eBooks

#### Tools

### _Related_

![][11]

[Alexander Rubin][12]

Alexander joined Percona in 2013. Alexander worked with MySQL since 2000 as DBA and Application Developer. Before joining Percona he was doing MySQL consulting as a principal consultant for over 7 years (started with MySQL AB in 2006, then Sun Microsystems and then Oracle). He helped many customers design large, scalable and highly available MySQL systems and optimize MySQL performance. Alexander also helped customers design Big Data stores with Apache Hadoop and related technologies.

[1]: https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/
[2]: http://www.thecompletelistoffeatures.com/
[3]: http://pmmdemo.percona.com
[4]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png
[5]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png
[6]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png
[7]: https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/
[8]: http://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html
[9]: http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page
[10]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png
[11]: https://secure.gravatar.com/avatar/79877aeedbd68531a30468cd771d5d07?s=84&d=mm&r=g
[12]: https://www.percona.com/blog/author/alexanderrubin/