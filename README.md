# Hai tuần tự học và dựng hệ thống log PBL4

Từ một thao tác trên shop, cả nhóm sẽ lần được sự kiện từ file log đến Elasticsearch, xem nó trong Kibana và giải thích khi nào nó góp phần sinh cảnh báo. Dùng nguyên web PHP/MariaDB của [`PBL4-demo` tại commit tham chiếu](https://github.com/hoangFan1005/PBL4-demo/tree/a653ae4245002148bcafffaa6040858848c2b484) trên nhánh `Maybe_Final` để tạo request thật trong lab.

| Đọc | Trách nhiệm chính |
| --- | --- |
| [Lý thuyết chung](Ly-thuyet-chung.md) | Cả ba học hợp đồng log, lần theo dữ liệu và hiểu cách kiểm chứng |
| [Hoàng](Hoang.md) | Compose, Nginx, log nguồn, Filebeat và mTLS |
| [Trí](Tri.md) | Logstash, chuẩn hóa, GeoIP, Elasticsearch và bằng chứng ingest |
| [Bảo](Bao.md) | Discover, dashboard, baseline và bốn rule ES\|QL |

## Lịch học

Học 10 buổi trong hai tuần, mỗi người khoảng **20 giờ**. Khung mặc định mỗi buổi là 20 phút đọc lý thuyết và đối chiếu source, 75 phút thực hành, 25 phút kiểm tra và bàn giao. Từng tài liệu cá nhân điều chỉnh theo bài tập. Ngày 5 và 6 có thể dùng phần thời gian chờ baseline hoặc index để đối chiếu truy vấn và cập nhật bằng chứng. Nhóm tự chọn ngày bắt đầu và có thể dùng cuối tuần làm thời gian bù.

| Buổi | Hoàng | Trí | Bảo | Kết quả chung |
| --- | --- | --- | --- | --- |
| 1 | Chuẩn bị máy, clone và khởi động theo lớp | Cài và kiểm template, ingest pipeline trước Filebeat | Xác định luồng, mở Kibana, đọc một document | Cùng SHA, có báo cáo bootstrap trên lab trống và một log tìm lại được |
| 2 | Sinh request và đọc ba file log | Dựng parser nhỏ và map access | Tạo data view, lọc và lưu phiên Discover | Có mẫu 404 và cặp access/auth dùng chung ID |
| 3 | Tái dựng cách khai báo input và nhãn Filebeat | Xử lý auth/error, lỗi parse và dữ liệu mô phỏng | Tái dựng biểu đồ và bảng bằng Lens/ES\|QL | Cùng giải thích dataset, index và bộ lọc |
| 4 | Kiểm thử mTLS và chuẩn bị tải shop | Dựng template, mapping và ingest pipeline nhỏ | Viết truy vấn R1–R4, chuẩn bị đo baseline | Chuẩn bị lab yên tĩnh cho phiên đo |
| 5 | Xác nhận log nguồn và phối hợp tải thường | Đối chiếu số đếm theo phút trên Elasticsearch | Đo baseline 30 phút và chuẩn bị bật rule theo kết quả | Chốt baseline và chứng minh đường đi 404 trước tuần 2 |
| 6 | Gửi bộ 404/401/403 có dấu kiểm | Truy vết từng loại log và kiểm tập ID | Mở Discover, chạy đối chứng và kiểm cảnh báo | Ghép bằng chứng từ file nguồn đến Kibana |
| 7 | Đối chiếu mốc phát request, ingest và Discover | Kiểm GeoIP và trạng thái khi thiếu dữ liệu địa lý | Kiểm ngưỡng và vòng đời cảnh báo trên lab riêng | Có kết quả độ trễ, GeoIP và cảnh báo theo đúng phạm vi từng ca |
| 8 | Khoanh lỗi ở nguồn/Filebeat | Khoanh lỗi ở parser/mapping | Khoanh lỗi time range/rule/action | Diễn tập đổi người xử lý sự cố và khôi phục |
| 9 | Dựng lại lab riêng từ commit tham chiếu | Thu bootstrap + trace trong cùng ngày UTC | Mở lại dashboard và ghi nhận kết quả lặp lại | Người khác chạy được từ hướng dẫn, có bảng PASS/FAIL |
| 10 | Demo từ request tới Filebeat | Demo parser, mapping và Elasticsearch | Demo dashboard, rule và tập ID nguồn | Demo 10–12 phút và đổi vai giải thích |

### Bàn giao giữa ba người

1. **Hoàng → Trí.** Gửi giờ UTC, URL path/dấu kiểm, HTTP status, dataset và `request.id` nếu có. Ngày 6 dùng 404 access, 401 access + auth, và 403 access + error do Nginx chặn đường dẫn ẩn.
2. **Trí → Bảo.** Gửi tên index, field truy vấn, kiểu mapping, thời gian ingest và cách nhận diện dữ liệu thật. Access/auth truy theo ID. Error có thể tra theo dataset, UTC và dấu kiểm trong nội dung đã làm sạch.
3. **Bảo → cả nhóm.** Gửi ảnh Discover hoặc dashboard có time range, cùng truy vấn và số đếm. Với cảnh báo active, ghi rule, cửa sổ, ngưỡng, số đếm và cách đối chiếu `pbl4.source_request_ids`.

Mỗi mẫu bằng chứng có người chạy, commit, múi giờ, câu lệnh hoặc truy vấn, kết quả mong đợi, kết quả quan sát và trạng thái `PASS`, `FAIL` hoặc `NOT RUN`. Lưu JSON/ảnh lab dưới `.runtime/evidence/` của checkout nguồn. Khi chia sẻ, chỉ lấy các trường cần thiết và ảnh đã kiểm tra dữ liệu nhạy cảm.

## Buổi 1 và buổi 9 · Dựng bản tham chiếu

Chạy từng lệnh theo thứ tự trong PowerShell. Khi có lệnh trả lỗi, kiểm tra bước đó trước khi tiếp tục. Mỗi người cần quyền đọc repo nguồn và dùng một checkout lab riêng. Trên cùng máy, chỉ chạy một lab dùng cổng 8080 và 5601 tại một thời điểm.

### 1. Chuẩn bị checkout và Docker

Yêu cầu Python 3.10+, Git, Docker Desktop dùng Linux containers và Compose v2. Runbook dự án yêu cầu tối thiểu 4 CPU, 8 GiB RAM cấp cho Docker (khuyến nghị 12 GiB), 20 GiB đĩa trống trước khi tải ảnh, cùng `vm.max_map_count` tối thiểu `1048576` trên Linux host của Docker. Xem cách xử lý WSL2 và yêu cầu dung lượng trong [runbook Windows của bản tham chiếu](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/runbooks/windows.md).

```powershell
git clone --branch Maybe_Final https://github.com/hoangFan1005/PBL4-demo.git PBL4-lab
Set-Location PBL4-lab
git switch --detach a653ae4245002148bcafffaa6040858848c2b484
git rev-parse HEAD
docker version
docker compose version
python --version
$env:COMPOSE_PROJECT_NAME = 'pbl4-hoc-hoang'
```

Trí và Bảo dùng lần lượt `pbl4-hoc-tri` và `pbl4-hoc-bao`. Dùng lại đúng tên project khi tiếp tục với volume cũ. Buổi 9 đặt tên project mới, chẳng hạn `pbl4-hoc-hoang-lap-lai`, và dùng một checkout sạch. Chạy một project có tên riêng vẫn dùng cùng hai cổng loopback.

```powershell
python scripts/bootstrap/prepare.py
python scripts/verify/preflight.py
docker compose config --quiet
docker compose pull --ignore-buildable
docker compose build php certificates
```

`prepare.py` tạo `.env` riêng và cấu hình bootstrap trong `secrets/` của checkout nguồn. Dùng `config --quiet` để kiểm tra mà không in cấu hình đã nội suy. Nếu đã có image đúng tag trong Docker, có thể dùng cache khi registry tạm thời không truy cập được.

### 2. Dựng Elasticsearch trước khi bật shipper

```powershell
docker compose up -d --wait --wait-timeout 300 elasticsearch
docker compose run --rm accounts
python scripts/bootstrap/install_elastic.py
python scripts/verify/pipeline_evidence.py --phase bootstrap --output .runtime/evidence/pipeline-bootstrap.json
```

Điều kiện đạt trên project mới là script báo `Native pipeline evidence bootstrap: PASS`, với `pre_ingest_empty=true`. Nó chứng minh template, mapping, ingest pipeline và các index trống đã sẵn sàng. Bước này cần thực hiện trước Filebeat. Khi tiếp tục lab đã có dữ liệu, chỉ chạy ba lệnh đầu và giữ báo cáo bootstrap của lượt sạch. Bài kiểm GeoIP native ở phần Trí cần Elasticsearch tải được cơ sở dữ liệu GeoIP.

### 3. Chạy shop có sẵn và bắt đầu thu log

```powershell
docker compose up -d mariadb php nginx logstash kibana
docker compose exec -T php php /var/www/html/bin/seed.php
docker compose up -d --wait --wait-timeout 360 filebeat kibana
docker compose ps
docker compose exec -T filebeat filebeat test output --strict.perms=false
python scripts/bootstrap/install_kibana.py
python scripts/traffic/run.py baseline --target http://127.0.0.1:8080 --count 8 --rate 2
```

Lần cài Kibana đầu tạo các dashboard, data view và bốn luật quản lý ở trạng thái tắt. Dùng [hướng dẫn của Bảo](Bao.md) để đo baseline trước khi bật. Khi chỉ cập nhật giao diện trên lab đã có luật đang bật, dùng `python scripts/bootstrap/install_kibana.py --dashboards-only`.

- Shop chạy ở [http://127.0.0.1:8080](http://127.0.0.1:8080). Mở `/health` để xác nhận Nginx đang trả lời. Dùng `/health.php` để kiểm qua PHP và cơ sở dữ liệu.
- Kibana chạy ở [https://localhost:5601](https://localhost:5601). Có thể xuất **CA công khai** bằng `docker compose cp elasticsearch:/usr/share/elasticsearch/config/certs/ca.crt secrets/ca.crt` và tin cậy CA trên trình duyệt dành cho lab. Đăng nhập bằng `elastic` và mật khẩu `ELASTIC_PASSWORD` đọc cục bộ từ `.env`. Xóa trust CA đã thêm khi kết thúc lab.
- Điều kiện đạt là shop trả 200, Filebeat báo kết nối thành công đến `logstash:5044`, và có thể mở Discover để truy một request vừa tạo. Tám request trên tạo dữ liệu để khám phá và thuộc lượt kiểm tra khởi động. Lượt đo baseline riêng ở buổi 5 kéo dài 30 phút. Chi tiết từng lớp nằm trong [runbook local](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/runbooks/local.md).

### 4. Kiểm tra đường đi và kết thúc phiên

Trên **cùng checkout, cùng Compose project và cùng ngày UTC** với snapshot bootstrap, chạy bài trace native trước khi qua 00:00 UTC. Nó tự tạo request và thu các mẫu của chính nó. Dùng tên output mới khi lặp lại.

```powershell
python scripts/verify/pipeline_evidence.py --phase trace --bootstrap .runtime/evidence/pipeline-bootstrap.json --output .runtime/evidence/pipeline-trace.json
```

Ngày 6 thực hành truy vết thủ công theo bộ dấu kiểm riêng. Ngày 9 chạy lại bootstrap và trace trong một phiên trên lab sạch. `docker compose down` tắt dịch vụ và giữ dữ liệu. Ghi lại tên project để mở lại bằng cùng thứ tự.

## Demo cuối buổi 10

| Thời lượng | Người trình bày | Bằng chứng trên màn hình |
| --- | --- | --- |
| 3 phút | Hoàng | Tạo một request 404 mới và đọc một dòng nguồn cùng ID, giải thích Filebeat gửi đến cổng 5044 |
| 3 phút | Trí | Tìm document cùng ID, chỉ ra timestamp, mapping, dataset và index. Mở cặp access/auth 401 đã thu |
| 4 phút | Bảo | Mở dashboard và cảnh báo đã tạo trong lượt thử, chỉ ra cửa sổ/ngưỡng và lần ngược một ID trong snapshot |
| 1–2 phút | Cả nhóm | Đổi vai. Mỗi người giải thích một lớp do người khác phụ trách và cách phân biệt lỗi ở hai lớp liền kề |

Chấm đạt khi người khác dựng được lab, nhóm truy được cả ba nguồn, đối chiếu được ít nhất một alert active với tập ID nguồn và giải thích được vì sao lượt đối chứng không tạo alert. Đánh dấu rõ các bài chưa chạy, đặc biệt là vòng đời đầy đủ R1–R4, latency trên trình duyệt và kiểm chứng AWS. [Biên bản AWS ngày 12/11/2026](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/final-aws-verification-2026-11-12.md) là tài liệu tham chiếu cho kết quả của bản đã triển khai.
