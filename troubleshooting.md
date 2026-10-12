# BÀI 5 - SESSION 12

## 1. Chẩn đoán và khắc phục 3 sự cố đứt gãy Pipeline thực tế

---

### Sự cố 1: Gradle Wrapper thiếu quyền thực thi (Permission Denied)

#### a. Triệu chứng & Log lỗi thực tế:
```text
Run ./gradlew test
/home/runner/work/_temp/...sh: line 1: ./gradlew: Permission denied
Error: Process completed with exit code 126.
```

#### b. Nguyên nhân:
- Môi trường phát triển ban đầu được thao tác trên hệ điều hành Windows (hệ thống tệp NTFS không lưu thuộc tính cờ thực thi `POSIX executable bit (+x)`).
- Khi đẩy mã nguồn lên Git và được checkout sang môi trường GitHub Actions Runner chạy hệ điều hành Linux (`ubuntu-latest`), tệp `gradlew` chỉ có quyền đọc/ghi (`-rw-r--r--`) mà thiếu quyền thực thi (`+x`).
- Do đó, khi runner gọi `./gradlew test`, hệ thống trả về mã lỗi `126 (Permission Denied)`.

#### c. Giải pháp khắc phục:
- Thêm bước gán quyền thực thi cho `gradlew` bằng lệnh `chmod +x gradlew` ngay trước khi thực thi lệnh `./gradlew test`:
  ```yaml
  # Khắc phục Lỗi 1: Cấp quyền thực thi cho gradlew
  - name: Grant execute permission for gradlew
    run: chmod +x gradlew

  - name: Run Tests with Gradle
    run: ./gradlew test
  ```

---

### Sự cố 2: Lỗi xác thực Docker Registry do sai tên biến Secret (Authentication Failed)

#### a. Triệu chứng & Log lỗi thực tế:
```text
Run docker/login-action@v3
Error: Username and password required
# Hoặc:
Error: Cannot perform an interactive login from a non TTY device
Error: Process completed with exit code 1.
```

#### b. Nguyên nhân:
- Tệp kịch bản truyền sai tên biến Secret lưu trữ token/mật khẩu xác thực (ví dụ: gõ nhầm thành `${{ secrets.REGISTY_TOKEN }}` hoặc `${{ secrets.REGISTRY_PASS }}`).
- Khi tên Secret bị sai, GitHub Actions sẽ trả về chuỗi rỗng (`""`), dẫn đến việc action `docker/login-action@v3` không nhận được thông tin đăng nhập hợp lệ và ném ra ngoại lệ xác thực thất bại.

#### c. Giải pháp khắc phục:
- Kiểm tra lại chính tả và chuẩn hóa tên biến Secret trong tệp workflow:
  ```yaml
  # Khắc phục Lỗi 2: Sử dụng đúng tên biến secret REGISTRY_TOKEN
  - name: Log in to Docker Registry
    uses: docker/login-action@v3
    with:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.REGISTRY_TOKEN }}
  ```
- Đảm bảo trong mục **Settings > Secrets and variables > Actions** của Repository đã được cấu hình 2 Secrets:
  - `DOCKER_USERNAME`: Tên tài khoản Docker Hub (hoặc actor tương ứng).
  - `REGISTRY_TOKEN`: Personal Access Token của Docker Registry.

---

### Sự cố 3: Không tìm thấy tệp Dockerfile do chỉ định sai thư mục làm việc (Build Context Error)

#### a. Triệu chứng & Log lỗi thực tế:
```text
Run docker/build-push-action@v5
ERROR: failed to solve: failed to read dockerfile: open /var/lib/docker/tmp/.../Dockerfile: no such file or directory
Error: buildx failed with: ERROR: failed to solve: failed to read dockerfile
Error: Process completed with exit code 1.
```

#### b. Nguyên nhân:
- Trong bước `docker/build-push-action@v5`, tham số `context` và `file` bị chỉ định sai đường dẫn (ví dụ: trỏ vào thư mục con không tồn tại `context: ./src` hoặc `file: ./app/Dockerfile` trong khi tệp `Dockerfile` nằm ngay tại thư mục gốc của repository).
- Docker daemon không tìm thấy `Dockerfile` trong ngữ cảnh build (Build Context) được cung cấp, dẫn đến tiến trình build bị dừng lại.

#### c. Giải pháp khắc phục:
- Cấu hình lại `context` trỏ về thư mục gốc `.` và `file` trỏ chính xác vào `./Dockerfile`:
  ```yaml
  # Khắc phục Lỗi 3: Đặt context ở thư mục gốc (.) và trỏ đúng Dockerfile
  - name: Build and Push Docker Image
    uses: docker/build-push-action@v5
    with:
      context: .
      file: ./Dockerfile
      push: false
      tags: ${{ secrets.DOCKER_USERNAME }}/my-app:latest
  ```

---

## 2. Nội dung tệp kịch bản hoàn chỉnh (.github/workflows/broken-pipeline.yml)

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # Lỗi 1: Cấp quyền thực thi cho gradlew
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run Tests with Gradle
        run: ./gradlew test

      # Lỗi 2: Sử dụng đúng secret REGISTRY_TOKEN
      - name: Log in to Docker Registry
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      # Lỗi 3: Đặt context ở thư mục gốc (.) và trỏ đúng Dockerfile
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: false
          tags: ${{ secrets.DOCKER_USERNAME }}/my-app:latest
```