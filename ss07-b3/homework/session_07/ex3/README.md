# Bài 3 – Thiết lập MySQL và Systemd cho Spring Boot

**Session 07 – Exercise 3**  
**Thư mục nộp:** `homework/session_07/ex3/`

## 1. Mục tiêu

- Tạo MySQL database `springboot_db` và tài khoản `spring-admin` chỉ có quyền trên database này.
- Chạy ứng dụng Spring Boot (build bằng Gradle) dưới user hệ thống không có shell đăng nhập `spring-runner`.
- Quản lý ứng dụng qua Systemd; tự khởi động khi boot và khởi động lại sau 10 giây nếu lỗi.
- Lắng nghe tại cổng `8082`.

## 2. Chuẩn bị

Ubuntu có hỗ trợ Systemd, MySQL Server và Java tương thích với bản JAR. Kiểm tra:

```bash
java -version
systemctl --version
sudo systemctl status mysql
```

Nếu dùng WSL, kiểm tra `ps -p 1 -o comm=` trả về `systemd`.

## 3. Tạo database và phân quyền MySQL

```bash
sudo mysql
```

```sql
CREATE DATABASE IF NOT EXISTS springboot_db;
CREATE USER IF NOT EXISTS 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';
ALTER USER 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';
SHOW GRANTS FOR 'spring-admin'@'localhost';
EXIT;
```

Kiểm tra đăng nhập (nhập mật khẩu khi được hỏi, không đặt mật khẩu trong dòng lệnh):

```bash
mysql -u spring-admin -p springboot_db
```

Trong MySQL:

```sql
SELECT DATABASE();
SELECT CURRENT_USER();
EXIT;
```

> Mật khẩu trên là mật khẩu bài tập theo đề. Với môi trường thật, hãy dùng mật khẩu riêng và không commit bí mật lên GitHub.

## 4. Tạo user Linux và chuẩn bị JAR

```bash
id spring-runner || sudo useradd --system --no-create-home --shell /usr/sbin/nologin spring-runner
sudo mkdir -p /opt/spring-app
```

Build project Spring Boot Gradle từ thư mục gốc dự án:

```bash
./gradlew clean bootJar
ls -lh build/libs/
```

Copy đúng tên file JAR đã build (thay `<ten-file>.jar` bằng tên thực tế):

```bash
sudo cp build/libs/<ten-file>.jar /opt/spring-app/app.jar
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 750 /opt/spring-app
sudo chmod 640 /opt/spring-app/app.jar
```

Cấu hình `application.yml` của Spring Boot (nếu cần):

```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:mysql://localhost:3306/springboot_db}
    username: ${SPRING_DATASOURCE_USERNAME:spring-admin}
    password: ${SPRING_DATASOURCE_PASSWORD}
server:
  port: ${SERVER_PORT:8082}
```

## 5. Tạo file môi trường bảo mật

```bash
sudo mkdir -p /etc/spring-app
sudo nano /etc/spring-app/spring-app.env
```

Nhập dòng sau trong file, không đưa file này vào Git:

```ini
SPRING_DATASOURCE_PASSWORD=SpringSecure@123
```

```bash
sudo chown root:root /etc/spring-app/spring-app.env
sudo chmod 600 /etc/spring-app/spring-app.env
```

## 6. Cài đặt và chạy dịch vụ Systemd

File `spring-app.service` trong cùng thư mục là file cấu hình tự viết gồm `[Unit]`, `[Service]`, `[Install]`.

```bash
sudo cp spring-app.service /etc/systemd/system/spring-app.service
sudo systemctl daemon-reload
sudo systemctl enable spring-app.service
sudo systemctl start spring-app.service
```

Giải thích: `User=spring-runner` hạn chế quyền; `ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar` chạy đúng JAR; `Restart=on-failure` và `RestartSec=10` tự khởi động lại sau 10 giây khi tiến trình lỗi.

## 7. Kiểm thử (nộp kết quả lệnh dạng văn bản, không cần ảnh)

Chạy trên máy triển khai:

```bash
sudo systemctl status spring-app.service --no-pager
sudo ss -tlnp | grep 8082
ps -eo user,pid,args | grep '[j]ava -jar /opt/spring-app/app.jar'
curl -i http://localhost:8082/health
```

**Kết quả cần xác nhận:**

| Hạng mục | Kết quả mong đợi | Kết quả thực tế |
|---|---|---|
| Systemd | `active (running)` | Chưa ghi nhận |
| Cổng TCP | Java lắng nghe `:8082` | Chưa ghi nhận |
| User tiến trình | `spring-runner` | Chưa ghi nhận |
| API `/health` (nếu có) | HTTP 200 | Chưa ghi nhận |

Sau khi chạy thực tế, thay `Chưa ghi nhận` bằng kết quả và có thể dán output terminal vào README. Không dùng kết quả ví dụ như bằng chứng đã chạy.

Nếu dịch vụ lỗi:

```bash
sudo journalctl -u spring-app.service -n 80 --no-pager
sudo systemctl restart spring-app.service
```

## 8. Kết luận

Bộ cấu hình đáp ứng các ràng buộc thiết kế của đề bài: MySQL riêng, user non-root, lệnh Java cố định, cổng 8082 và chính sách tự khởi động lại. Trạng thái chạy thực tế cần được xác nhận bằng các lệnh kiểm thử tại mục 7.
