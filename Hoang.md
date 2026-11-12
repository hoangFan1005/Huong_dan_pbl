# Hoàng — Từ request HTTP đến Filebeat

[Trang bắt đầu](README.md) · [Lý thuyết chung](Ly-thuyet-chung.md)

**20 giờ trong 2 tuần, 10 buổi × 2 giờ.** Hoàng dựng lab, dùng shop đã clone để tạo sự kiện, tái dựng cấu hình log nguồn và Filebeat, rồi giao dấu truy vết cho Trí và Bảo. Kết thúc, Hoàng phải tự chỉ được điểm cuối còn thấy log khi pipeline trục trặc.

Nguồn đối chiếu là `PBL4-demo`, nhánh `Maybe_Final` tại commit `a653ae4245002148bcafffaa6040858848c2b484`. Các lệnh thực hành dùng PowerShell trong thư mục chứa `compose.yaml` của **PBL4-demo**. Dùng lab local trên Docker Desktop/WSL 2 với Linux containers, Python 3.10+, Docker Compose v2 trở lên, Docker có ít nhất 4 CPU và 8 GiB RAM, ổ chứa dữ liệu Docker còn khoảng 20 GiB trước khi tải image. Các block lệnh giả định bước trước đã exit 0, ngoại trừ những HTTP status được chỉ định. [Runbook local của dự án](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/runbooks/local.md) · [Docker Desktop trên Windows](https://docs.docker.com/desktop/setup/install/windows-install/)

## Tuần 1 — Dựng từng đoạn và đọc log tại nguồn

### Ngày 1 — 25 phút học, 75 phút dựng, 20 phút đối chiếu

**Học.** Image là gói chạy, container là tiến trình cô lập, named volume giữ trạng thái qua lần tạo lại container, còn bind mount đưa file cấu hình trên host vào container. `depends_on` với `service_healthy` đợi healthcheck khi khởi động. Điều này giúp mở đúng thứ tự nhưng khi debug vẫn phải kiểm tra trạng thái hiện tại của từng tầng. [Docker Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/)

1. Mở Docker Desktop, chọn Linux containers, kiểm tra hai cổng loopback `8080` và `5601` đang rảnh. Lần tải image đầu có thể bắt đầu trước buổi học. Chuẩn bị checkout riêng từ repo private bằng tài khoản nhóm đã được cấp quyền.

   ```powershell
   git clone --branch Maybe_Final https://github.com/hoangFan1005/PBL4-demo.git pbl4-hoang-lab
   Set-Location pbl4-hoang-lab
   git switch --detach a653ae4245002148bcafffaa6040858848c2b484
   git rev-parse HEAD
   $env:COMPOSE_PROJECT_NAME = 'pbl4-hoc-hoang'
   python --version
   python scripts/verify/preflight.py
   Get-NetTCPConnection -State Listen -LocalPort 8080,5601 -ErrorAction SilentlyContinue |
     Select-Object LocalAddress,LocalPort,OwningProcess
   wsl -d docker-desktop -u root sysctl vm.max_map_count
   ```

   Ghi SHA, kết quả preflight và project name. Nếu cổng đang dùng, xác định tiến trình trước khi chọn máy hoặc lịch khác. Nếu kernel Docker dưới `1048576`, áp dụng hướng dẫn Windows của nguồn và kiểm lại. Đặt lại `COMPOSE_PROJECT_NAME` như trên trong từng cửa sổ PowerShell chạy Compose hoặc script Python của repo. Hai lab dùng chung cổng host thì chạy luân phiên. Khi hai người nối bước trong cùng lab, đối chiếu checkout và dùng đúng cùng project name trước khi gọi Compose hoặc script. [Runbook Windows của dự án](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/runbooks/windows.md)

2. Tạo biến môi trường và dựng từng nhóm dịch vụ. `prepare.py` tạo bí mật local từ riêng checkout, còn `certificates` và `accounts` là các container chạy một lần. Dùng `config --quiet` để kiểm cú pháp, tránh in cấu hình đã nội suy mật khẩu.

   ```powershell
   python scripts/bootstrap/prepare.py
   docker compose config --quiet
   docker compose pull --ignore-buildable
   docker compose build php certificates
   docker compose up -d --wait elasticsearch
   docker compose run --rm accounts
   python scripts/bootstrap/install_elastic.py
   python scripts/verify/pipeline_evidence.py --phase bootstrap --output .runtime/evidence/pipeline-bootstrap.json
   ```

   Cùng Trí đọc file E5 trước khi bật Filebeat. `status` phải là `PASS`, `pre_ingest_empty` là `true` và các chỉ mục được đo có `document_count: 0`. Dùng lần chạy này để giải thích vì sao template và ingest pipeline phải có trước document đầu tiên. Nếu lab đã có dữ liệu, dùng project name mới và tên bằng chứng mới cho một lần đo trống thực sự. Phép thử GeoIP trong runbook cần Internet từ phía Elasticsearch và là phép kiểm riêng.

3. Khởi động các dịch vụ còn lại và seed shop đã clone.

   ```powershell
   docker compose up -d mariadb php nginx logstash kibana
   docker compose exec -T php php /var/www/html/bin/seed.php
   docker compose up -d --wait --wait-timeout 600 filebeat kibana
   docker compose ps --all
   docker compose exec -T nginx nginx -t
   docker compose exec -T filebeat filebeat test output --strict.perms=false
   curl.exe -fsS http://127.0.0.1:8080/health
   curl.exe -fsS http://127.0.0.1:8080/health.php
   ```

   `/health` trả `ok` tại Nginx và được cấu hình tắt access log cho route này. `/health.php` trả `{"status":"ok"}` khi PHP đọc được bảng ứng dụng và có thể ghi auth log. `filebeat test output` phải nối được Logstash qua TLS. Ghi trạng thái của các dịch vụ chạy dài và exit code của hai container một lần. Bảo chuẩn bị cấu hình Kibana theo tài liệu của mình.

4. Cùng Trí có thể chạy collector trace ngay trong cùng ngày UTC với lần đo E5. Collector tự tạo request kiểm thử và đối chiếu tên index trace với snapshot E5 theo ngày UTC. Mốc ngày 6 dùng mẫu thủ công nếu index đã sang ngày mới.

   ```powershell
   python scripts/verify/pipeline_evidence.py --phase trace --bootstrap .runtime/evidence/pipeline-bootstrap.json --output .runtime/evidence/pipeline-trace-day1.json
   ```

**Đạt khi** từ checkout sạch, Hoàng dựng được lab theo đúng thứ tự, lấy được E5 trước khi ship log, hiểu khác nhau giữa hai endpoint health và giao trạng thái ban đầu cho Trí/Bảo.

### Ngày 2 — 30 phút đọc, 65 phút tái dựng, 25 phút đối chiếu

**Học.** Access log là một dòng JSON theo request. Auth log là JSON do PHP ghi sau lần xác thực thực sự. Error log Nginx là text chẩn đoán. `$request_id` được Nginx trả trong response `X-Request-ID` và truyền cho PHP qua FastCGI `REQUEST_ID`. Một lần POST login có thể có cặp access/auth cùng ID. Error log thường phải đối chiếu theo URI và thời gian. [Nginx hiện hành](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/infra/nginx/nginx.conf) · [Hàm ghi auth trong Shop.php](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/app/src/Shop.php)

1. Tạo cấu hình thực hành trong `.runtime/`, giữ file nguồn để đối chiếu.

   ```powershell
   New-Item -ItemType Directory -Path .runtime/hoang -Force | Out-Null
   Copy-Item infra/nginx/nginx.conf .runtime/hoang/nginx.conf
   @'
   services:
     nginx:
       volumes:
         - ./.runtime/hoang/nginx.conf:/etc/nginx/nginx.conf:ro
   '@ | Set-Content -Encoding utf8 .runtime/hoang/compose.nginx.yaml
   ```

2. Mở bản `.runtime/hoang/nginx.conf`. Giữ khai báo worker, server, vị trí app và FastCGI. Viết lại đoạn định dạng/đích log trong đúng context. Giữ chính xác các key và kiểu số để parser của Trí đọc được. So với source trước khi chạy.

   ```nginx
   # Trong context chính, cùng cấp với events và http
   error_log /var/log/nginx/error.log warn;

   # Trong http
   log_format access_json escape=json
       '{"time":"$time_iso8601","remote_addr":"$remote_addr",'
       '"x_forwarded_for":"","host":"$host",'
       '"method":"$request_method","uri":"$uri","protocol":"$server_protocol",'
       '"status":$status,"body_bytes_sent":$body_bytes_sent,"request_length":$request_length,'
       '"request_time":$request_time,"referer":"",'
       '"user_agent":"$http_user_agent","request_id":"$request_id"}';
   access_log /var/log/nginx/access.json access_json;

   # Trong server
   add_header X-Request-ID $request_id always;

   # Trong location PHP
   fastcgi_param REQUEST_ID $request_id;
   ```

   `$uri` ở cấu hình local giữ đường dẫn thay vì chuỗi query của request. `x_forwarded_for` và `referer` là chuỗi rỗng ở snapshot này. `start.sh` đổi `error.log` từ symlink của image thành file thường để shipper đọc được. So file học với `infra/nginx/nginx.conf` để xem mình có đổi ngoài phạm vi cần học hay không.

3. Kiểm overlay và chạy Nginx bằng file vừa viết. Hai khai báo mount dùng cùng đích `/etc/nginx/nginx.conf`, nên overlay chọn file lab. Đường dẫn được tính từ file Compose đầu tiên. [Quy tắc merge volume của Compose](https://docs.docker.com/reference/compose-file/merge/#unique-resources)

   ```powershell
   docker compose -f compose.yaml -f .runtime/hoang/compose.nginx.yaml config --quiet
   docker compose -f compose.yaml -f .runtime/hoang/compose.nginx.yaml run --rm --no-deps --entrypoint nginx nginx -t
   docker compose -f compose.yaml -f .runtime/hoang/compose.nginx.yaml up -d --wait --no-deps nginx
   ```

4. Dùng helper dưới đây trong mỗi buổi cần tạo marker. Nó chỉ in thời gian, đường dẫn, HTTP status và request ID do server trả về. Lưu lại đoạn helper để nạp lại khi đổi cửa sổ PowerShell.

   ```powershell
   function Send-PblProbe {
     param([string]$Path)
     $sent = [DateTimeOffset]::UtcNow.ToString('o')
     $lines = curl.exe -sS -D - -o NUL -w 'HTTP_STATUS=%{http_code}\n' "http://127.0.0.1:8080$Path"
     if ($LASTEXITCODE -ne 0) { throw "HTTP probe lỗi kết nối: $Path" }
     $rawHeaders = $lines -join "`n"
     $idMatch = [regex]::Match($rawHeaders, '(?im)^X-Request-ID:[ \t]*([a-f0-9]{32})[ \t]*\r?$')
     $codeMatch = [regex]::Match($rawHeaders, '(?m)^HTTP_STATUS=(\d{3})\r?$')
     if (!$idMatch.Success -or !$codeMatch.Success) { throw "Thiếu X-Request-ID hoặc HTTP status: $Path" }
     [pscustomobject]@{
       SentUtc = $sent
       Path = $Path
       Status = [int]$codeMatch.Groups[1].Value
       RequestId = $idMatch.Groups[1].Value
     }
   }

   $marker = [guid]::NewGuid().ToString('N')
   $ok = Send-PblProbe '/index.php'
   $miss = Send-PblProbe "/pbl4-hoang-404-$marker"
   $denied = Send-PblProbe "/.pbl4-hoang-denied-$marker"
   $ok, $miss, $denied | Format-Table
   docker compose exec -T nginx tail -n 10 /var/log/nginx/access.json
   docker compose exec -T nginx tail -n 6 /var/log/nginx/error.log
   ```

   Kết quả dự kiến lần lượt là 200, 404, 403. Dùng marker của 403 để tìm `access forbidden by rule` trong error log. Từ trình duyệt local, mở DevTools → Network → Preserve log. Mở `/login.php`, đăng nhập sai bằng email mẫu như `invalid-demo@example.test` và mật khẩu giả. Chọn **POST** `/login.php`, ghi 401 và header `X-Request-ID`. Nếu thử thành công với tài khoản seed, ghi 302 của POST trước khi browser theo redirect. Kiểm tra file nguồn:

   ```powershell
   docker compose exec -T -u 0 php tail -n 6 /var/log/shop/auth.json
   ```

   Chỉ lưu các trường phục vụ đối chiếu. Nhìn vào `event.action`, `request.id`, `@timestamp`. Trong code hiện hành, `user.id` là ID nội bộ khi thành công và null khi thất bại. Nếu POST trả 403 do CSRF, tải form mới trong cùng session và thử lại. Không chụp mật khẩu, cookie, token hoặc email thật.

**Đạt khi** có ba mẫu nguồn riêng, chứng minh được access/auth của lần login sai chung ID, và tìm được error 403 theo dấu đường dẫn.

### Ngày 3 — 35 phút học, 65 phút cấu hình, 20 phút bàn giao

**Học.** Filebeat `filestream` tìm file theo path/glob, nhận diện file bằng fingerprint và lưu tiến độ đọc trong registry. Cấu hình này giữ ba input ID riêng, đọc cả file hiện tại và file đổi tên, chuyển từng dòng dưới dạng `message` cùng nhãn dataset. Registry đi theo volume `filebeat_data`. Logstash parse payload ở phía nhận. Filebeat xác minh CA và hostname của Logstash, còn Logstash yêu cầu chứng chỉ client. [Filestream](https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-filestream) · [mTLS Filebeat–Logstash](https://www.elastic.co/docs/reference/beats/filebeat/configuring-ssl-logstash)

1. Từ bảng này, tự viết `.runtime/hoang/filebeat.yml` với ba mục `filebeat.inputs`. Ví dụ đầu tiên và phần output ở dưới. Nhân mẫu input và thay đúng `id`, `paths`, `event.dataset` cho hai dòng còn lại. Dùng cùng ID với config chuẩn để tiếp tục dùng registry hiện tại.

   | ID | File chính và glob xoay vòng | Dataset |
   | --- | --- | --- |
   | `pbl4-nginx-access` | `/var/log/nginx/access.json`, `/var/log/nginx/access.json.*` | `nginx.access` |
   | `pbl4-nginx-error` | `/var/log/nginx/error.log`, `/var/log/nginx/error.log.*` | `nginx.error` |
   | `pbl4-shop-auth` | `/var/log/shop/auth.json`, `/var/log/shop/auth.json.*` | `shop.auth` |

   ```yaml
   filebeat.inputs:
     - type: filestream
       id: pbl4-nginx-access
       paths: [/var/log/nginx/access.json, /var/log/nginx/access.json.*]
       fields_under_root: true
       fields:
         event: {dataset: nginx.access}
         labels: {synthetic: "false"}
     # Thêm hai mục filestream vào đây
   output.logstash:
     hosts: ["logstash:5044"]
     ssl.enabled: true
     ssl.verification_mode: full
     ssl.certificate_authorities: [/usr/share/filebeat/certs/ca.crt]
     ssl.certificate: /usr/share/filebeat/certs/filebeat.crt
     ssl.key: /usr/share/filebeat/certs/filebeat.key
   logging.level: info
   logging.to_files: false
   ```

2. Tạo overlay Filebeat, giữ log volumes dạng chỉ đọc và `filebeat_data` như Compose gốc. Kiểm cấu hình trước khi chạy, rồi kiểm lại kết nối Logstash khi đã khởi động.

   ```powershell
   @'
   services:
     filebeat:
       volumes:
         - ./.runtime/hoang/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
   '@ | Set-Content -Encoding utf8 .runtime/hoang/compose.filebeat.yaml

   docker compose -f compose.yaml -f .runtime/hoang/compose.filebeat.yaml config --quiet
   docker compose -f compose.yaml -f .runtime/hoang/compose.filebeat.yaml run --rm --no-deps --entrypoint filebeat filebeat test config --strict.perms=false
   docker compose -f compose.yaml -f .runtime/hoang/compose.filebeat.yaml up -d --wait --no-deps filebeat
   docker compose exec -T filebeat filebeat test output --strict.perms=false
   docker compose exec -T filebeat ls -l /var/log/nginx /var/log/shop /usr/share/filebeat/data
   ```

   Nếu lỗi x509, đối chiếu hostname `logstash`, public CA và mount client cert/key ở đúng volume của project. Nếu lỗi DNS hoặc TCP, kiểm `docker compose ps --all logstash filebeat` và network `shipping` trong YAML. Nếu test output đạt nhưng event chưa đến, kiểm file có dòng mới, mount đúng và chờ scanner. Filebeat 9.5.3 mặc định hỗ trợ growing fingerprint để theo dõi cả file dưới 1 KiB. [File identity của Filebeat](https://www.elastic.co/docs/reference/beats/filebeat/file-identity) · [Cấu hình Filebeat hiện hành](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/infra/filebeat/filebeat.yml) · [Beats input hiện hành](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/infra/logstash/pipelines/00-input.conf)

3. Tạo thêm 404 bằng helper và nhờ Trí tìm `request.id` trong index access. Trả cấu hình Nginx/Filebeat về bản chuẩn sau bài thực hành để cả nhóm dùng cùng mốc từ ngày 4.

   ```powershell
   docker compose up -d --wait --no-deps nginx
   docker compose up -d --wait --no-deps filebeat
   docker compose exec -T filebeat filebeat test output --strict.perms=false
   ```

**Đạt khi** Filebeat từ YAML tự viết nối được Logstash, Trí thấy marker và Hoàng giải thích được dữ liệu thuộc mount nào, TLS tin cậy hai chiều ra sao, và lý do giữ registry.

### Ngày 4 — 25 phút học, 65 phút thực hành, 30 phút phối hợp

**Học.** Dấu kiểm thử giúp khoanh một thao tác cụ thể. Các lệnh gửi HTTP tạo log thật trong lab và nhãn ingest `"false"`. Fixture do Trí chạy trên pipeline kiểm thử đi qua nguồn riêng. Lưu lượng nền để đo ngưỡng R4 phải có khoảng thời gian riêng và được thống nhất với Bảo.

1. Chạy thử workload HTTP nhẹ **trước** giờ đo baseline của Bảo. Kiểm `http_requests`, `statuses`, `started_at` và `ended_at`.

   ```powershell
   python scripts/traffic/run.py baseline --target http://127.0.0.1:8080 --count 12 --rate 1
   ```

2. Ghi và giải thích chuỗi trường bằng dữ liệu của nhóm. So `status` với `http.response.status_code`, `uri` với `url.path`, `request_id` với `request.id`, giờ nguồn với `@timestamp`. `event.ingested` là lúc ingest ở Elasticsearch. Lấy dữ liệu chuẩn hóa từ Trí hoặc qua Discover của Bảo.
3. Chốt lịch đo baseline thực với Bảo. Trong khoảng đo, cả nhóm giữ cùng workload đã chọn. Dừng bài 403, quét 404, burst và thao tác có thể làm lệch mẫu. Đối chiếu giờ bắt đầu/kết thúc theo UTC và test output trước khung đo.

**Đạt khi** có một lượt chạy nhẹ với 12 HTTP 200, bảng ánh xạ trường từ một marker và cửa sổ baseline đã chốt.

### Ngày 5 — 20 phút ôn, 70 phút kiểm mốc, 30 phút trao đổi

1. Mở sơ đồ nguồn của nhóm và chỉ đúng ranh giới Docker network `frontend` / `web` / `shipping` / `elk`. Nêu lý do `9200` và `5044` chỉ dùng trong mạng Compose local, còn Nginx/Kibana bind loopback ở host. Cùng cả nhóm nhìn sang bản AWS đã ghi nhận ngày 12/11/2026. Shop/Filebeat trên EC2 WEB, ELK trên EC2 private, Filebeat gửi mTLS 5044. [Bằng chứng AWS của dự án](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/final-aws-verification-2026-11-12.md)
2. Kiểm nhanh trạng thái **ngoài** cửa sổ baseline bằng `docker compose ps --all` và `docker compose exec -T filebeat filebeat test output --strict.perms=false`. Trong cửa sổ đo, hỗ trợ Bảo theo dõi thời gian và workload đã thống nhất. Hoàng giữ nhật ký theo UTC cho mọi gián đoạn.
3. Giao Trí một cặp access/auth theo ID, một marker error và trạng thái E5. Giao Bảo khung baseline, cấu hình network và sự cố nếu có. Đổi vai 5 phút để Trí hoặc Bảo tự giải thích lại một đoạn từ HTTP tới shipper.

**Đạt khi** cả nhóm đọc được baseline do Bảo đo hoặc ghi rõ điều kiện còn thiếu, đồng thời cả ba nguồn đã được đối chiếu với log thật trước khi mở các ca phát cảnh báo.

## Tuần 2 — Tích hợp, chẩn đoán và trình diễn

### Ngày 6 — 20 phút chuẩn bị, 70 phút truy vết, 30 phút đối soát

1. Sau khi Bảo kết thúc baseline, chạy các marker mới. Dùng hai mẫu `GET /pbl4-hoang-404-<GUID>` và `GET /.pbl4-hoang-denied-<GUID>` với helper ngày 2. Thực hiện thêm POST `/login.php` sai trong browser, lấy 401 và ID từ Network. Đây là bộ mẫu truy vết; phối hợp giờ riêng với Bảo cho tải vượt ngưỡng.
2. Hoàng xác nhận tại nguồn bằng `tail` hoặc `grep -F` trong container. Ví dụ lệnh lọc raw access theo ID của 404:

   ```powershell
   $marker = [guid]::NewGuid().ToString('N')
   $miss = Send-PblProbe "/pbl4-hoang-404-$marker"
   $miss | Format-List
   docker compose exec -T nginx grep -F -- $miss.RequestId /var/log/nginx/access.json
   ```

3. Điền một hàng cho mỗi thao tác. Hai hàng của 401 có cùng ID; hàng error có URI và thời gian làm khóa tra cứu. Gửi phần bảng bên trái cho Trí để tìm index, rồi đối chiếu với Discover của Bảo. Kibana hiện hiển thị giờ `Asia/Ho_Chi_Minh`, dữ liệu xuất và ghi chú bàn giao dùng UTC. [Cấu hình múi giờ Kibana](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/infra/kibana/kibana.yml)

   | Thời gian UTC | HTTP / URI | Dataset | ID hoặc dấu kiểm thử | Kết quả Trí / Bảo |
   | --- | --- | --- | --- | --- |
   | Điền từ lần chạy | 404 / đường dẫn giả | `nginx.access` | `X-Request-ID` | Index + Discover |
   | Điền từ lần chạy | 401 /login.php | `nginx.access` | ID từ POST | Index + Discover |
   | Điền từ dòng auth | 401 /login.php | `shop.auth` | ID từ cùng POST | Index + Discover |
   | Điền từ lần chạy | 403 / marker bị chặn | `nginx.error` | URI + khung giờ UTC | Index + Discover |

**Đạt khi** cùng một bộ marker được nhìn thấy ở log nguồn và Elasticsearch/Discover. Nếu error chưa tới, xác nhận Nginx đã tạo dòng `access forbidden by rule`, rồi kiểm path, thời gian và Logstash.

### Ngày 7 — 30 phút học, 60 phút diễn tập, 30 phút kiểm chéo

1. Với Bảo, học sự khác nhau giữa tốc độ gửi request, thời gian xuất hiện trong Elasticsearch và thời gian người xem thấy trong Discover. Dùng mốc gửi UTC từ helper và `event.ingested` để tính độ trễ cho từng mẫu. Bảo ghi thêm giờ quan sát Discover và nhịp refresh. Máy chạy thao tác và Docker cần đồng hồ đồng bộ.
2. Ngoài cửa sổ baseline, cùng Bảo thử ca 404 dưới ngưỡng, dùng 4 request trên 2 đường dẫn. Kiểm 404 tại nguồn và đưa Bảo các số đếm độc lập. Thực hiện sau khi Bảo xác nhận cửa sổ kiểm thử của rule đã sạch dữ liệu từ lượt trước.

   ```powershell
   python scripts/traffic/run.py scan404 --target http://127.0.0.1:8080 --count 4 --distinct 2 --rate 1
   ```

3. Khi Bảo chạy ca vượt ngưỡng riêng, ghi tập marker hoặc thời gian của đợt từ góc nhìn nguồn. Giải thích vì sao các request 404 vẫn làm tăng tổng access của R1/R4, còn R3 cần cả số 404 và số đường dẫn khác nhau. Đối chiếu một ID nguồn trong alert mà Bảo tìm thấy.

**Đạt khi** phân biệt được bài đối chứng âm với lỗi ship log, và giải thích được một alert bằng bằng chứng nguồn.

### Ngày 8 — 25 phút lập giả thuyết, 70 phút gây lỗi/khôi phục, 25 phút tổng kết

**Học.** Nếu ứng dụng vẫn phục vụ và file raw có marker mới trong khi Filebeat dừng, đoạn HTTP → Nginx đang hoạt động. Nếu ES chưa có ID cho đến khi Filebeat trở lại, vị trí gián đoạn đã được thu hẹp. Giữ volume registry, nhờ Trí đếm ID trước/sau, nhờ Bảo xác nhận kết quả trên Discover. Chọn giờ riêng vì đây là bài làm gián đoạn quan sát.

1. Đảm bảo đã có event đầu tiên đi qua Filebeat và hai bạn còn lại đã mở sẵn phép truy vấn. Dùng một terminal dừng Filebeat, tạo marker và kiểm tại nguồn. Trong lúc `Read-Host` đang chờ, Trí/Bảo tra ID ở một cửa sổ khác. Bấm Enter để phục hồi kể cả nếu phép đếm trước khôi phục không như dự kiến.

   ```powershell
   $marker = [guid]::NewGuid().ToString('N')
   try {
     docker compose stop filebeat
     if ($LASTEXITCODE -ne 0) { throw 'Không dừng được Filebeat' }
     docker compose ps --all filebeat
     $held = Send-PblProbe "/pbl4-hoang-shipper-gap-$marker"
     $held | Format-List
     docker compose exec -T nginx grep -F -- $held.RequestId /var/log/nginx/access.json
     if ($LASTEXITCODE -ne 0) { throw 'Chưa thấy marker trong raw log' }
     Read-Host 'Enter sau khi Trí/Bảo ghi số document hiện tại' | Out-Null
   } finally {
     docker compose up -d --wait --wait-timeout 120 --no-recreate --no-deps filebeat
   }
   docker compose exec -T filebeat filebeat test output --strict.perms=false
   ```

2. Poll trong tối đa 120 giây. Mục tiêu thực hành là 0 hit trước phục hồi và đúng 1 hit sau phục hồi. Kiểm lại sau thêm một chu kỳ scanner khoảng 12 giây. Nếu chưa thấy, kiểm Filebeat state, mount, kích thước file, Logstash input và index failed theo thứ tự. Ghi rõ đoạn cuối còn bằng chứng và thời điểm lần đầu thấy hit.
3. Khi có giờ chạy tự động trên lab riêng, dùng verifier gốc để xem thêm rotation, mất kết nối Logstash, mTLS âm/dương và lỗi listener PHP-FPM 502 có phục hồi.

   ```powershell
   python scripts/verify/stack.py --output .runtime/evidence/stack-day8.json
   ```

   Verifier có các phép thử dừng service và gián đoạn PHP ngắn. Ghi phạm vi kiểm thử gồm marker của lượt chạy và các báo cáo PASS/FAIL thực tế. [Verifier gốc](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/scripts/verify/stack.py)

**Đạt khi** chứng minh được việc vận chuyển bù đúng một marker trong ca dừng/khôi phục hữu hạn và ghi thời gian thực tế. Kết quả đo áp dụng cho các marker và khoảng gián đoạn đã kiểm.

### Ngày 9 — 20 phút chuẩn bị, 75 phút chạy lại, 25 phút kiểm độc lập

1. Đưa cấu hình về bản chuẩn và hạ **project lab của Hoàng** bằng `docker compose down`. Lệnh giữ volumes để có thể mở lại. Dùng project name mới trong PowerShell cho lượt tái lập sau khi cổng host đã rảnh.

   ```powershell
   $env:COMPOSE_PROJECT_NAME = 'pbl4-hoc-hoang'
   docker compose ps --all
   docker compose down
   $env:COMPOSE_PROJECT_NAME = 'pbl4-hoc-hoang-lap-lai'
   docker compose config --quiet
   docker compose up -d --wait elasticsearch
   docker compose run --rm accounts
   python scripts/bootstrap/install_elastic.py
   python scripts/verify/pipeline_evidence.py --phase bootstrap --output .runtime/evidence/pipeline-bootstrap-day9.json
   ```

2. Lặp phần shop, seed, Filebeat của ngày 1. Bảo cài lại dashboard từ source theo runbook cá nhân. Cùng Trí lấy trace mới trong cùng ngày UTC với E5 vừa tạo, dùng tên output khác. E5 của ngày 1 giữ nguyên để so hai lần.

   ```powershell
   python scripts/verify/pipeline_evidence.py --phase trace --bootstrap .runtime/evidence/pipeline-bootstrap-day9.json --output .runtime/evidence/pipeline-trace-day9.json
   ```

3. Mời Trí thao tác theo hướng dẫn, Hoàng chỉ quan sát. Ghi bước nào phải hỏi, bước nào cần chờ service, và bằng chứng để kết luận. Nếu dùng project mới cho phần phát cảnh báo, Bảo cần baseline mới cho môi trường đó. Có thể mở lại project đã đo khi trình diễn phần luật.

**Đạt khi** bạn khác tự dựng được một project mới, có log thật đến ELK và có hai lượt E5/trace để so trong phạm vi đã chạy.

### Ngày 10 — 30 phút chuẩn bị, 60 phút demo và đổi vai, 30 phút bàn giao

1. Chuẩn bị một bộ 404, 401 và 403 mới với thời gian UTC. Kiểm `/health.php` và `filebeat test output` trên project đang trình diễn. Bảo xác nhận đang ở đúng Kibana và khoảng thời gian.
2. Dành khoảng 3 phút trong bài demo chung cho đoạn nguồn. Gửi 404, chỉ `X-Request-ID` và một dòng access. Gửi login sai, chỉ cùng ID ở dòng auth. Mở mẫu 403 và chỉ phần URI/thời gian trong error. Nói một câu Filebeat đọc file nào, ở volume nào, đẩy tới đâu và TLS xác minh những bên nào. Bàn giao màn hình cho Trí nối trace tới ES và Bảo nối sang Discover/alert.
3. Đổi vai. Hoàng tự tìm lại một marker trong Discover và giải thích tại sao thiếu GeoIP với IP riêng là kết quả có thể gặp. Trí hoặc Bảo tự tìm raw log trong container và trả lời cách khoanh lỗi nếu `filebeat test output` đạt nhưng truy vấn chưa có hit.
4. Gom bằng chứng gồm SHA, project name, trạng thái bootstrap, bảng marker, đối soát nhận bù và ảnh/query đã rà soát. Mẫu bằng chứng chỉ cần thời gian, path lab, status, dataset, request ID, index và trạng thái kiểm thử. Giữ `.env`, `secrets/`, raw log và thư mục runtime trong máy lab. Khi kết thúc dùng `docker compose down` trên project mình vừa chạy để giữ dữ liệu.

**Đạt khi** cả ba người giải thích cùng một trace theo thứ tự HTTP → file nguồn → Filebeat → Logstash → Elasticsearch → Kibana và Hoàng tự tái hiện được ca log thiếu rồi xuất hiện lại sau phục hồi.
