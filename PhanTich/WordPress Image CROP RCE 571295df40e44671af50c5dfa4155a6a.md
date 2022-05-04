# WordPress Image CROP RCE

# I, **TỔNG QUAN**

Bài viết này bao gồm 2 CVE là CVE-2019-8942 và CVE-2019-8943

| CVE | Mô tả |
| --- | --- |
| CVE-2019-8942 | Lỗ hổng cho phép thực thi mã từ xa bằng cách thực thi PHP chứa mã độc thông qua giá trị bảng wp_postmeta. |
| CVE-2019-8943 | Lỗ hổng trong đó các tệp có thể được lưu trong một đường dẫn tùy ý bằng cách sử dụng tham số meta_input khi xảy ra thao tác thay đổi thông tin kích thước của hình ảnh đã tải lên. |

Bằng cách sử dụng hai lỗ hổng trên cùng nhau, có thể thực thi mã từ xa sau khi tải hình ảnh có mã PHP lên một đường dẫn tùy ý.

Các phiên bản bị ảnh hưởng:

WordPress <= 4.9.8.

WordPress từ 5.0.0 đến 5.0.1.

Trong hàm eidt_post(), dữ liệu POST được lưu trữ trong biến $post_data mà không cần kiểm tra và giá trị POST được chuyển qua wp_update_post() → wp_insert_post() cập nhật dữ liệu được lưu trữ trong DB bởi hàm update_post_meta. Tại thời điểm này, có thể thay đổi đường dẫn dữ liệu mà hình ảnh sẽ được nhập bằng cách cập nhật giá trị _wp_attached_file.

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled.png)

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%201.png)

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%202.png)

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%203.png)

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%204.png)

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%205.png)

Hàm wp_crop_image() được gọi khi thay đổi kích thước tệp sẽ lấy đường dẫn để lưu tệp thông qua hàm get_attached_file() và hàm get_attached_file() lấy đường dẫn tệp từ _wp_attached_file được lưu trữ trong DB thông qua hàm get_post_meta().

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%206.png)

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%207.png)

# II. KHAI THÁC

Sử dụng tool exiftool để truyền đoạn mã PHP.

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%208.png)

Vào Media chọn Add New để upload ảnh.

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%209.png)

Sau đó chọn vào hình ảnh đã tải lên, chọn Edit more details và cuối cùng chọn Updates.

Dùng burp để bắt req như hình bên dưới:

![2.png](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/2.png)

Chọn Edit image để cắt ảnh.

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%2010.png)

Burp để bắt req sẽ như hình bên dưới:

![3.png](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/3.png)

Hàm wp_insert_post() lấy hầu hết các biến $post_data mà không kiểm tra trong đó có meta_input. Để thực hiện khai thác, ta sẽ chèn thêm tham số meta_input[_wp_attached_file], đây là tham số nhằm thay đổi một tên tệp bất kỳ. Tên tệp này sẽ được gọi khi một yêu cầu chỉnh sửa hình ảnh được gửi đến server (ta sẽ thực hiện ở bước sau). Thực hiện thêm như sau &meta_input[_wp_attached_file]=2019/03/finally.jpg#/../../../../themes/twentyseventeen/Twings.jpg vào /wp-admin/post.php ta được:

![5.png](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/5.png)

Sau khi thực hiện cắt ảnh, máy chủ sẽ thực hiện lưu ảnh. Tuy nhiên sau khi cắt máy chủ mặc định sử dụng tên tệp cục bộ như giá trị gốc truyền vào meta_input[_wp_attached_file]=2019/03/finally.jpg#/../../../../themes/twentyseventeen/Twings.jpg  Điều này dẫn đến lỗ hổng LFI(local file include) có thể được khai thác.

Để ý res trả về có gen một tên mới cho ảnh sau khi cắt là: cropped-Twings.jpg. Hãy note lại tên này.

![6.png](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/6.png)

Tạo bài đăng có kèm theo ảnh chứa đoạn mã PHP bằng cách tạo thêm bài đăng

![Untitled](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/Untitled%2011.png)

Burp bắt req và thực hiện thêm &meta_input[_wp_page_template]= [tên bản sao] (tên bản sao là tên được tự động gen lúc cắt ảnh)

![7.png](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/7.png)

Lúc này ta chỉ cần vào xem bài đăng mới là sẽ thấy kết quả của mã PHP được thực thi.

![8.png](WordPress%20Image%20CROP%20RCE%20571295df40e44671af50c5dfa4155a6a/8.png)

# III, KẾT LUẬN

Những lỗ hổng trên nằm tại những nơi mà nếu chỉ khai thác riêng rẽ thì gần như không có nhiều tác động. Tuy nhiên CVE trên đã xâu chuỗi các lỗ hổng lại và tạo nên 1 lỗ hổng vô cùng nguy hiểm.

Tại những phiên bản sau wordpress đã thực hiện loại bỏ Loại bỏ các tham số đầu vào như meta_input, file,... Đồng thời có đoạn mã kiểm tra nếu đầu vào có tồn tại ../ hoặc .. hay không. => Điều này đã vá thành công