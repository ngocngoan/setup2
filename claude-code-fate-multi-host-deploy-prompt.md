# Deploy FATE Multi-Host with Monitoring

Bạn là một Senior DevOps Engineer kiêm Machine Learning Infrastructure Engineer, am hiểu Federated Learning, FATE Framework, KubeFATE, FATE-Serving, FATE-Community, eggroll, FATE-Board, OSX, Docker Compose, Linux, SSH automation và vận hành production.

Hãy dựa vào toàn bộ mã nguồn hiện có của dự án và các nguồn tham khảo chính thức sau đây để triển khai FATE multi-host:

- https://github.com/FederatedAI/FATE
- https://github.com/FederatedAI/FATE-Serving
- https://github.com/FederatedAI/FedLCM
- https://github.com/FederatedAI/KubeFATE
- https://github.com/FederatedAI/FATE-Flow
- https://github.com/FederatedAI/FATE-LLM
- https://github.com/FederatedAI/FATE-Cloud
- https://github.com/FederatedAI/FATE-Community
- https://github.com/FederatedAI/eggroll
- https://github.com/FederatedAI/FATE-Board
- https://github.com/FederatedAI/FATE-Client
- https://github.com/FederatedAI/FATE-Builder
- https://fate.readthedocs.io/en/latest/architecture
- https://fate.readthedocs.io/en/latest/2.0/fate/quick_start
- https://fate.readthedocs.io/en/latest/2.0/osx/osx

## 1. Bối cảnh triển khai

Tôi muốn deploy ứng dụng này trên 3 host với thông tin như sau:

| IP host | Vai trò |
|---|---|
| `172.16.21.190` | `Party 10000` |
| `172.16.21.138` | `Party 9999` |
| `172.16.21.169` | `Party 9998` |

Thông tin SSH của các host đã được cấu hình trong file:

```bash
/home/ngoandn/.ssh/config
```

Yêu cầu bắt buộc: phải truy cập từng host thông qua SSH để thực hiện quá trình deploy FATE, ngoại trừ môi trường monitoring local được mô tả bên dưới.

## 2. Nguyên tắc thực hiện

1. Hãy làm việc từ thư mục root của repository hiện tại.
2. Trước khi chỉnh sửa hoặc chạy lệnh, hãy inspect cấu trúc dự án, `Makefile`, thư mục `infras`, thư mục generator, file `.env`, `compose.yaml`, `monitoring.compose.yaml`, các script deploy và tài liệu monitoring liên quan.
3. Ưu tiên dùng các target đã có trong `Makefile`, đặc biệt: `clean-workspace`, `init-multi-machine-config`, `generate-multi-machine-config`, `deploy-ssh`, `deploy-monitoring-local`, `deploy-monitoring-ssh` nếu phù hợp.
4. Không tự ý giả định tên thư mục generated nếu trong repository có naming convention khác. Hãy dùng `find`, `ls`, `grep`, `make help` hoặc đọc trực tiếp `Makefile` để xác minh.
5. Không thay đổi các biến môi trường không được yêu cầu thay đổi.
6. Không xóa dữ liệu ngoài các thư mục deploy được chỉ định.
7. Với mọi thao tác xóa thư mục/file, phải kiểm tra biến đường dẫn khác rỗng và nằm đúng trong phạm vi deploy.
8. Mỗi lần deploy/redeploy, phải làm sạch các file cấu hình, compose, script và artifact cũ trong thư mục deploy trước khi copy bản mới; tuy nhiên **không được xóa dữ liệu Docker volume/bind-mount do container tạo ra**.
9. Dữ liệu persistent sau khi chuyển volume sang bind mount phải được bảo vệ tối thiểu trong thư mục `workspace/` dưới deploy directory. Nếu phát hiện compose dùng thêm bind mount source khác ngoài `workspace/`, phải đưa các path đó vào danh sách bảo vệ trước khi cleanup.
10. Nếu một bước bị lỗi, hãy đọc log, xác định nguyên nhân, sửa lỗi hợp lý rồi chạy lại. Không dừng ở thông báo lỗi chung chung.
11. Sau mỗi bước lớn, hãy in ra kết quả kiểm tra ngắn gọn để chứng minh bước đó thành công.
12. Không commit code. Sau khi hoàn thành, hãy hiển thị `git diff --stat` và tóm tắt các file đã thay đổi.

## 3. Các bước cần thực hiện

### Bước 1: Làm sạch workspace cũ và khởi tạo tập tin cấu hình multi-machine

Yêu cầu tiên quyết bắt buộc: trước khi tạo mới cấu hình deploy, phải chạy lệnh sau trong root repository để xóa cấu hình cũ và workspace sinh tự động của lần chạy trước:

```bash
make clean-workspace
```

Sau khi `make clean-workspace` chạy thành công, tiếp tục khởi tạo lại cấu hình multi-machine bằng lệnh:

```bash
make init-multi-machine-config
```

Sau đó xác định chính xác file cấu hình multi-machine được tạo ra. Theo mã nguồn hiện tại, file thường là:

```bash
infras/generator/configs/parties.multi-machine.conf
```

Nếu repository có thay đổi khác, hãy dùng giá trị thực tế trong `Makefile`.

### Bước 2: Cập nhật cấu hình host và deploy directory

Sau khi khởi tạo tập tin cấu hình, hãy điền topology sau vào file multi-machine config:

```bash
PARTY_IDS=(10000 9999 9998)

PARTY_IPS=(172.16.21.190 172.16.21.138 172.16.21.169)
SERVING_IPS=(172.16.21.190 172.16.21.138 172.16.21.169)
```

Quy tắc cấu hình user:

- User deploy cho từng host là user đã được cấu hình trong `/home/ngoandn/.ssh/config`.
- Hãy parse hoặc kiểm tra SSH config để xác định user tương ứng với từng IP/host alias.
- Không hardcode user nếu SSH config đã có user phù hợp.
- Cập nhật `DEPLOY_USERS=(...)` theo đúng user đã xác minh.

Quy tắc cấu hình deploy directory:

- Thư mục deploy logic là:

```bash
~/opt/dockers/fate-ops
```

- Trên từng remote host, hãy chuyển `~` thành absolute path tương ứng với home directory của user deploy trên host đó.
- Cập nhật `DEPLOY_DIRS=(...)` theo absolute path thực tế.
- Kiểm tra thư mục đã tồn tại hay chưa:
  - Nếu chưa tồn tại: tạo thư mục.
  - Nếu đã tồn tại: **không xóa toàn bộ thư mục deploy**. Chỉ xóa các file cấu hình/compose/script/artifact cũ bên trong thư mục deploy, đồng thời bảo vệ dữ liệu persistent do container tạo ra.

Quy tắc cleanup bắt buộc khi redeploy trên remote host:

- Không xóa chính thư mục `~/opt/dockers/fate-ops`.
- Không xóa `~/opt/dockers/fate-ops/workspace` và mọi file/thư mục con bên trong nó.
- Nếu `compose.yaml` hoặc `monitoring.compose.yaml` hiện tại có bind mount source khác ngoài `./workspace/...`, phải bảo vệ thêm các path đó.
- Chỉ xóa các file/thư mục deploy artifact như `compose.yaml`, `monitoring.compose.yaml`, `.env`, config, cert, script, log tạm, package copy từ generated output.

Trên mỗi host, tạo thêm/thêm lại thư mục nếu chưa có:

```bash
~/opt/dockers/fate-ops/workspace
```

### Bước 3: Cập nhật biến môi trường

Cập nhật file môi trường phù hợp của dự án, ưu tiên file được generator sử dụng. Nếu dự án hiện tại dùng `infras/common/.env.example` làm nguồn generate config, hãy cập nhật file đó tạm thời để generate.

Các biến cần thay đổi:

```env
PROJECT_NAME=federated-learning-platform-fate
COMPUTE_CORE=16

MYSQL_ROOT_PASSWORD=ViettelAI@2026
MYSQL_PASSWORD=ViettelAI@2026
MYSQL_EXPORTER_PASSWORD=ViettelAI@2026

GRAFANA_ADMIN_PASSWORD=ViettelAI@2026

MONITORING_HOST=172.16.21.143
MONITORING_DEPLOY_USER=monitoring
MONITORING_DEPLOY_DIR=/home/ngoandn/projects/federated-learning/deployments
MONITORING_BIND_ADDRESS=172.16.21.143

FATEBOARD_PASSWORD=ViettelAI@2026

DOCKER_REGISTRY=hub.vtcc.vn:8989
```

Các biến môi trường khác phải giữ nguyên.

Lưu ý:

- `MONITORING_HOST=172.16.21.143` là địa chỉ IP của máy tính nội bộ/local machine.
- Không cần đăng nhập SSH vào `MONITORING_HOST` để deploy monitoring server.
- Phần monitoring server sẽ được deploy trực tiếp tại `MONITORING_DEPLOY_DIR` trên máy hiện tại.

### Bước 4: Generate cấu hình multi-machine

Chạy validate nếu Makefile có target tương ứng:

```bash
make validate-multi-machine-config
```

Tạo các thư mục cấu hình cho các Party bên trong thư mục generated của dự án, ví dụ:

```bash
infras/generator/generated
```

Chạy lệnh:

```bash
make generate-multi-machine-config
```

Sau khi generate xong:

- Xác minh các thư mục Party đã được tạo ra tương ứng với `10000`, `9999`, `9998`.
- Xác minh thư mục monitoring generated đã được tạo ra nếu generator có sinh monitoring config.
- Nếu đã phải chỉnh trực tiếp `infras/common/.env.example`, hãy revert thay đổi của file này sau khi generate để không lưu secret và cấu hình host-specific trong file example. Chỉ revert sau khi chắc chắn generated output đã được tạo xong.
- Lưu ý: quy tắc revert ở trên **chỉ áp dụng cho file `.env.example` tạm thời**. Không được revert thay đổi bind mount trong các file compose đã generate ở Bước 5.

### Bước 5: Chuyển Docker Compose storage từ named volume sang bind mount và migrate dữ liệu cũ nếu có

Trong các thư mục generated của từng party, tìm các file:

```bash
compose.yaml
monitoring.compose.yaml
```

Các file này thường nằm trong các thư mục dạng `conf<party_number>`, `confs<party_number>`, `config<party_number>` hoặc tên tương tự. Hãy xác định chính xác bằng lệnh `find`.

#### 5.1. Chuyển compose generated sang bind mount

Yêu cầu:

- Tại cả `compose.yaml` và `monitoring.compose.yaml`, thay đổi storage type từ Docker named volume sang bind mount.
- Dùng thư mục bind mount nằm dưới:

```bash
./workspace
```

vì sau này toàn bộ nội dung config sẽ được copy vào:

```bash
~/opt/dockers/fate-ops
```

Ví dụ nguyên tắc chuyển đổi:

```yaml
# Trước đây có thể là named volume
volumes:
  - mysql-data:/var/lib/mysql

# Sau khi chuyển sang bind mount
volumes:
  - ./workspace/mysql-data:/var/lib/mysql
```

Quy tắc bắt buộc:

1. Không làm hỏng target path trong container.
2. Giữ nguyên option read-only nếu volume cũ đang dùng `:ro`.
3. Nếu compose dùng top-level `volumes:`, hãy xóa hoặc bỏ các named volume không còn được dùng.
4. Tạo sẵn các thư mục bind mount cần thiết bên trong `workspace` nếu Docker Compose yêu cầu host path tồn tại.
5. Sau khi chuyển named volume sang bind mount, **không revert các file compose về named volume**. Bind mount là trạng thái cuối cùng dùng để deploy.
6. Khi deploy/redeploy, `workspace/` là vùng dữ liệu persistent và phải được bảo toàn.
7. Khi copy generated config lên remote host ở Bước 6, không được làm mất dữ liệu đã migrate trong `~/opt/dockers/fate-ops/workspace`.

Chạy kiểm tra cú pháp:

```bash
docker compose -f compose.yaml config
```

và tương tự cho:

```bash
docker compose -f monitoring.compose.yaml config
```

nếu môi trường local có Docker Compose. Nếu không thể chạy local vì thiếu Docker, hãy kiểm tra YAML bằng Python/YAML parser và giải thích rõ.

#### 5.2. Kiểm tra hệ thống đang chạy và named volume hiện có trên từng host

Trước khi copy toàn bộ generated config mới sang remote host, phải kiểm tra thư mục deploy hiện tại trên từng host:

```bash
~/opt/dockers/fate-ops
```

Với từng host `172.16.21.190`, `172.16.21.138`, `172.16.21.169`, hãy SSH vào host và thực hiện:

1. Nếu thư mục deploy chưa tồn tại hoặc chưa có `compose.yaml` / `monitoring.compose.yaml`, ghi nhận là host chưa có dữ liệu deploy cũ và bỏ qua bước migrate dữ liệu cũ cho host đó.
2. Nếu thư mục deploy đã tồn tại, kiểm tra hệ thống hiện tại có đang chạy không:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f compose.yaml ps
```

3. Nếu hệ thống FATE chính chưa chạy nhưng `compose.yaml` tồn tại, hãy chạy hệ thống hiện tại lên trước để Docker Compose khởi tạo đầy đủ volume và trạng thái container hiện có:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f compose.yaml up -d

docker compose -f compose.yaml ps
```

4. Nếu `monitoring.compose.yaml` tồn tại, kiểm tra monitoring agent hiện tại:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f monitoring.compose.yaml ps
```

5. Nếu monitoring agent chưa chạy nhưng `monitoring.compose.yaml` tồn tại, hãy chạy monitoring agent hiện tại lên trước:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f monitoring.compose.yaml up -d

docker compose -f monitoring.compose.yaml ps
```

6. Sau khi hệ thống hiện tại đã chạy hoặc đã xác nhận trạng thái, kiểm tra compose hiện tại có đang dùng Docker named volume hay không. Không đoán bằng mắt; hãy dùng `docker compose config`, inspect YAML hoặc JSON output nếu Docker Compose hỗ trợ:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f compose.yaml config > /tmp/fate-compose.resolved.yaml

docker compose -f monitoring.compose.yaml config > /tmp/fate-monitoring-compose.resolved.yaml 2>/dev/null || true
```

Các dấu hiệu named volume cần phát hiện:

- Service volume có source không bắt đầu bằng `./`, `/`, `~/` hoặc biến path host.
- Có top-level `volumes:` định nghĩa volume do Compose quản lý.
- `docker volume ls` có volume thuộc project deploy hiện tại, ví dụ có label `com.docker.compose.project` tương ứng với project trong compose.

Ví dụ lệnh inspect volume theo Compose project:

```bash
COMPOSE_PROJECT_NAME=$(docker compose -f compose.yaml config --format json 2>/dev/null | python3 -c 'import json,sys; print(json.load(sys.stdin).get("name", ""))' 2>/dev/null || true)

if [ -n "$COMPOSE_PROJECT_NAME" ]; then
  docker volume ls --filter "label=com.docker.compose.project=$COMPOSE_PROJECT_NAME"
fi
```

Nếu không có `--format json`, hãy dùng `docker compose -f compose.yaml config` và parse YAML bằng Python/YAML parser hoặc kiểm tra thủ công có kiểm soát.

#### 5.3. Đồng bộ dữ liệu từ named volume sang bind mount trước khi xoá named volume

Nếu phát hiện host đang dùng named volume, phải migrate dữ liệu từ named volume sang bind mount dưới `./workspace` sao cho dữ liệu nhất quán trước và sau khi đồng bộ.

Quy trình bắt buộc trên từng host có named volume:

1. Lập danh sách mapping đầy đủ từ named volume sang bind mount mới. Mapping phải dựa trên target path trong container và compose sau khi chuyển đổi, ví dụ:

```text
mysql-data -> ./workspace/mysql-data -> /var/lib/mysql
fateflow-data -> ./workspace/fateflow-data -> <container-target-path>
```

2. Dừng/quiesce container để tránh ghi dữ liệu trong lúc copy. Không dùng `down -v` vì sẽ xoá volume. Chỉ dùng `stop` hoặc `down` không kèm `-v`:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f monitoring.compose.yaml stop 2>/dev/null || true

docker compose -f compose.yaml stop
```

3. Tạo thư mục bind mount tương ứng dưới `workspace`:

```bash
mkdir -p ./workspace/<volume-name>
```

4. Copy dữ liệu từ named volume sang bind mount bằng helper container. Ưu tiên dùng `tar` để giữ permission, owner, symlink và file mode:

```bash
docker run --rm \
  -v <named-volume>:/from:ro \
  -v "$(pwd)/workspace/<volume-name>:/to" \
  alpine:3.20 \
  sh -c 'cd /from && tar cf - . | tar xpf - -C /to'
```

5. Kiểm tra tính nhất quán dữ liệu trước và sau đồng bộ. Tối thiểu phải so sánh số file, tổng dung lượng và checksum/tar hash nếu khả thi:

```bash
SOURCE_SUMMARY=$(docker run --rm -v <named-volume>:/from:ro alpine:3.20 sh -c 'cd /from && find . -xdev -type f | sort | wc -l && du -sb /from 2>/dev/null || du -sk /from')
TARGET_SUMMARY=$(docker run --rm -v "$(pwd)/workspace/<volume-name>:/to:ro" alpine:3.20 sh -c 'cd /to && find . -xdev -type f | sort | wc -l && du -sb /to 2>/dev/null || du -sk /to')

echo "SOURCE_SUMMARY=$SOURCE_SUMMARY"
echo "TARGET_SUMMARY=$TARGET_SUMMARY"
```

Nếu có thể, dùng tar stream hash để so sánh nội dung:

```bash
SOURCE_HASH=$(docker run --rm -v <named-volume>:/from:ro alpine:3.20 sh -c 'cd /from && tar cf - .' | sha256sum | awk '{print $1}')
TARGET_HASH=$(docker run --rm -v "$(pwd)/workspace/<volume-name>:/to:ro" alpine:3.20 sh -c 'cd /to && tar cf - .' | sha256sum | awk '{print $1}')

test "$SOURCE_HASH" = "$TARGET_HASH"
```

Nếu hash khác do metadata động hoặc thứ tự tar không ổn định, phải giải thích rõ và dùng thêm kiểm tra file count, size, quyền sở hữu thư mục quan trọng và chạy thử container bằng bind mount để xác nhận dữ liệu dùng được.

6. Sau khi copy dữ liệu xong, phải đảm bảo remote host đã dùng compose bind mount trước khi khởi động lại. Có thể thực hiện một trong hai cách:

- Copy trước các file compose đã được chuyển sang bind mount từ generated output sang host, nhưng **không xóa `workspace/`**.
- Hoặc tạm thời patch compose hiện tại trên host sang bind mount theo đúng mapping đã lập.

Sau đó khởi động lại bằng compose bind mount và kiểm tra container chạy được với dữ liệu bind mount:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f compose.yaml up -d

docker compose -f compose.yaml ps

docker compose -f monitoring.compose.yaml up -d 2>/dev/null || true

docker compose -f monitoring.compose.yaml ps 2>/dev/null || true
```

7. Chỉ sau khi đã xác nhận container chạy thành công bằng bind mount và dữ liệu nhất quán, mới xoá named volume cũ. Không dùng `docker volume prune`. Chỉ xoá đúng các named volume đã migrate và đã xác minh không còn container nào dùng:

```bash
docker ps -a --filter volume=<named-volume>

docker volume rm <named-volume>
```

Nếu `docker volume rm` báo volume còn đang được dùng, không force remove. Hãy inspect container đang giữ volume, xác định nguyên nhân và chỉ xoá sau khi chắc chắn compose mới không còn tham chiếu named volume.

8. Ghi lại trong báo cáo:

- Host nào có named volume cũ.
- Danh sách named volume đã migrate.
- Bind mount path mới tương ứng.
- Kết quả so sánh trước/sau migrate.
- Danh sách named volume đã xoá.
- Named volume nào không xoá được và lý do.

### Bước 6: Copy generated config lên từng host

Copy nội dung bên trong thư mục generated config của từng Party vào thư mục deploy tương ứng trên remote host:

```bash
~/opt/dockers/fate-ops
```

Mapping bắt buộc:

```text
Party 10000 -> 172.16.21.190 -> ~/opt/dockers/fate-ops
Party 9999  -> 172.16.21.138 -> ~/opt/dockers/fate-ops
Party 9998  -> 172.16.21.169 -> ~/opt/dockers/fate-ops
```

Ưu tiên dùng `rsync -av --delete` qua SSH nếu khả dụng. Tuy nhiên, khi dùng `--delete`, phải exclude mọi thư mục dữ liệu persistent đã được bảo vệ, tối thiểu là `workspace/`. Ví dụ nguyên tắc:

```bash
rsync -av --delete \
  --exclude='/workspace/***' \
  <generated-party-dir>/ <remote-host>:~/opt/dockers/fate-ops/
```

Nếu không có `rsync`, dùng `scp -r`, nhưng trước đó phải cleanup thủ công các artifact cũ trong deploy directory theo đúng quy tắc bảo vệ `workspace/`.

Sau khi copy, SSH vào từng host và xác minh:

```bash
cd ~/opt/dockers/fate-ops
pwd
ls -la
test -f compose.yaml
test -f monitoring.compose.yaml
test -d workspace
```

### Bước 7: Deploy hệ thống FATE trên 3 host

Trên từng host, thực hiện lần lượt:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f compose.yaml pull

docker compose -f compose.yaml up -d

docker compose -f compose.yaml ps
```

Sau khi chạy:

1. Kiểm tra các container có ở trạng thái healthy/running không.
2. Nếu có container failed/restarting/exited, đọc log:

```bash
docker compose -f compose.yaml logs --tail=200
```

3. Điều tra nguyên nhân và sửa lỗi phát sinh nếu có.
4. Sau khi deploy thành công, kiểm tra kết nối giữa các OSX ở các Party khác nhau.

Đối với kiểm tra OSX:

- Không tự đoán cổng nếu compose có định nghĩa cụ thể. Hãy inspect `compose.yaml`, container name, exposed ports, environment variables và network config.
- Kiểm tra network reachability giữa 3 host bằng `ping`, `nc`, `curl` hoặc command tương ứng với service OSX nếu có sẵn.
- Nếu project có script health check hoặc script verify federation/OSX, hãy ưu tiên sử dụng script đó.
- Ghi lại kết quả kiểm tra kết nối Party-to-Party.

### Bước 8: Deploy monitoring agent trên 3 host

Trên từng host, thực hiện lần lượt:

```bash
cd ~/opt/dockers/fate-ops

docker compose -f monitoring.compose.yaml pull

docker compose -f monitoring.compose.yaml build

docker compose -f monitoring.compose.yaml up -d

docker compose -f monitoring.compose.yaml ps
```

Sau khi chạy:

1. Kiểm tra container monitoring agent đã running/healthy chưa.
2. Nếu lỗi, đọc log:

```bash
docker compose -f monitoring.compose.yaml logs --tail=200
```

3. Điều tra nguyên nhân và sửa lỗi phát sinh nếu có.

### Bước 9: Deploy môi trường monitoring server trên local machine

Monitoring server được deploy tại máy local, không SSH vào `MONITORING_HOST`.

Trước khi deploy monitoring server, hãy tham khảo file tài liệu sau trong repository để đối chiếu các biến cấu hình monitoring đang được dùng cho mô hình triển khai 3 máy:

```bash
docs/monitoring/deployment-three-machines.md
```

Yêu cầu xử lý biến cấu hình monitoring:

- Nếu file tham khảo tồn tại trong repository, hãy đọc file đó và đối chiếu các biến `MONITORING_*`, `GRAFANA_*`, `PROMETHEUS_*`, `LOKI_*`, `ALERTMANAGER_*`, exporter/password/bind-address/deploy-dir liên quan.
- Các biến trong file tham khảo được dùng để xác nhận hoặc bổ sung cấu hình monitoring, nhưng không được tự ý thay đổi các biến FATE/Party không liên quan.
- Nếu file tham khảo không tồn tại trong repository hoặc không khớp với mã nguồn hiện tại, hãy tiếp tục dùng các giá trị monitoring đã khai báo trong prompt này và ghi rõ vào báo cáo.

Dùng thông tin mặc định dưới đây, trừ khi file tham khảo nêu trên chỉ ra giá trị monitoring cụ thể cần đồng bộ:

```bash
MONITORING_HOST=172.16.21.143
MONITORING_DEPLOY_USER=monitoring
MONITORING_DEPLOY_DIR=/home/ngoandn/projects/federated-learning/deployments
MONITORING_BIND_ADDRESS=172.16.21.143
```

Thực hiện:

1. Chuyển đến hoặc tạo thư mục deploy monitoring:

```bash
mkdir -p /home/ngoandn/projects/federated-learning/deployments
cd /home/ngoandn/projects/federated-learning/deployments
```

2. Xóa các file/thư mục deploy artifact cũ bên trong `MONITORING_DEPLOY_DIR`, nhưng không xóa chính thư mục deploy và không xóa dữ liệu persistent do container tạo ra. Dùng quyền root nếu cần thiết, nhưng phải có guard tránh xóa nhầm. Bảo vệ tối thiểu thư mục `workspace/`; nếu compose hiện tại có bind mount source khác, hãy bảo vệ thêm path đó trước khi chạy cleanup.

Ví dụ cleanup an toàn cho monitoring local:

```bash
MONITORING_DEPLOY_DIR=/home/ngoandn/projects/federated-learning/deployments
EXPECTED_MONITORING_DEPLOY_DIR=/home/ngoandn/projects/federated-learning/deployments

if [ -n "$MONITORING_DEPLOY_DIR" ] && [ "$MONITORING_DEPLOY_DIR" = "$EXPECTED_MONITORING_DEPLOY_DIR" ] && [ -d "$MONITORING_DEPLOY_DIR" ]; then
  sudo find "$MONITORING_DEPLOY_DIR" -mindepth 1 -maxdepth 1 \
    ! -name 'workspace' \
    ! -name '.workspace' \
    -exec rm -rf {} +
else
  echo "Refuse to cleanup unsafe MONITORING_DEPLOY_DIR=$MONITORING_DEPLOY_DIR" >&2
  exit 1
fi
```

3. Copy nội dung từ thư mục monitoring generated của dự án vào `MONITORING_DEPLOY_DIR`. Hãy xác định chính xác thư mục generated monitoring bằng `find`, ví dụ có thể là:

```bash
infras/generator/generated/monitoring
```

Khi copy monitoring generated output, không được overwrite/xóa dữ liệu trong `MONITORING_DEPLOY_DIR/workspace`. Nếu dùng `rsync --delete`, phải dùng `--exclude='/workspace/***'`.

4. Chạy Docker Compose cho monitoring server:

```bash
cd /home/ngoandn/projects/federated-learning/deployments

docker compose -f compose.yaml pull

docker compose -f compose.yaml up -d

docker compose -f compose.yaml ps
```

Nếu compose có build context hoặc Dockerfile local, chạy thêm:

```bash
docker compose -f compose.yaml build
```

trước khi `up -d`.

5. Kiểm tra monitoring server:

- Prometheus container running/healthy.
- Grafana container running/healthy.
- Loki/Alertmanager running/healthy nếu compose có.
- Exporter/agent targets visible nếu compose có cấu hình target.
- Grafana bind address dùng `172.16.21.143` nếu compose hỗ trợ bind address.
- Nếu có endpoint web, in ra URL truy cập Grafana/Prometheus tương ứng.

### Bước 10: Kiểm tra sau triển khai và báo cáo kết quả

Sau khi hoàn thành, hãy tạo hoặc cập nhật file báo cáo trong repository:

```bash
docs/deployments/fate-multi-host-deploy-report.md
```

Nội dung báo cáo gồm:

1. Topology triển khai: Party ID, IP, SSH host/user, deploy directory.
2. Danh sách file config đã generate.
3. Danh sách thay đổi chính trong compose, đặc biệt phần bind mount.
4. Danh sách thư mục persistent/bind-mount đã được bảo vệ khi redeploy, tối thiểu gồm `workspace/` trên từng host và monitoring local.
5. Kết quả migrate dữ liệu từ named volume sang bind mount:
   - Host nào có named volume cũ.
   - Named volume nào đã được copy.
   - Bind mount path mới.
   - Kết quả kiểm tra consistency trước/sau.
   - Named volume nào đã xoá.
   - Named volume nào chưa xoá được và lý do.
6. Kết quả `docker compose ps` của từng host.
7. Kết quả kiểm tra kết nối OSX giữa các Party.
8. Kết quả deploy monitoring agent trên 3 host.
9. Kết quả deploy monitoring server local.
10. URL truy cập FATE-Board, Fate-Flow, Grafana, Prometheus nếu xác định được từ compose.
11. Các lỗi đã gặp và cách khắc phục.
12. Các bước vận hành lại nhanh: restart, stop, view logs, redeploy.

Cuối cùng, trong câu trả lời của Claude Code, hãy tóm tắt:

- Đã triển khai thành công hay chưa.
- Host/Party nào thành công, host/Party nào còn lỗi.
- Command quan trọng đã chạy.
- Endpoint truy cập được.
- File báo cáo đã tạo.
- `git diff --stat`.
- Những việc tôi cần kiểm tra thủ công nếu còn tồn tại hạn chế môi trường.
