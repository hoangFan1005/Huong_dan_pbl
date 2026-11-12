# Trí — Xử lý log với Logstash và Elasticsearch

[Lịch của nhóm và cách khởi động lab](README.md) · [Lý thuyết chung](Ly-thuyet-chung.md)

**Thời lượng** 10 buổi, 120 phút mỗi buổi, tổng 20 giờ. **Đầu ra** một pipeline học tập dựng từ đầu, một bộ kiểm tra mapping và GeoIP, một lượt truy vết log thật qua stack của nhóm và một biên bản phục hồi khi gián đoạn. Thực hành bằng PowerShell và Kibana Dev Tools. Website đã có sẵn để phát sinh request.

**Bàn giao** Hoàng giao `event.dataset`, mẫu dòng nguồn và `request.id` hoặc URI đánh dấu. Trí giao cho Bảo index, kiểu dữ liệu và truy vấn tái hiện để làm Discover, dashboard và rules. Trí dùng bản clone riêng của `Maybe_Final` tại `a653ae4245002148bcafffaa6040858848c2b484`. Các lệnh tương đối trong tài liệu chạy ở gốc bản clone, trừ lệnh Dev Tools.

## Tuần 1 — Tự dựng và hiểu hợp đồng dữ liệu

### Ngày 1 — Theo dõi đường đi của log và chụp mốc trước ingest (25 phút học, 75 phút làm, 20 phút kiểm)

1. Mở `infra/filebeat/filebeat.yml`, `infra/logstash/pipelines/00-input.conf`, `10-normalize.conf`, `90-output.conf` theo thứ tự. Vẽ từ file nguồn đến Filebeat, Logstash và Elasticsearch. Đánh dấu cổng Beats `5044` dùng mTLS và Elasticsearch `9200` dùng HTTPS bên trong mạng Compose. Phân biệt input Beats, filter chuẩn hóa và output gửi bulk bằng user `logstash_writer`.
2. Kiểm tra SHA và bootstrap offline. Ghi kết quả từng lệnh.

   ```powershell
   git rev-parse HEAD
   python -m unittest discover -s tests/pipeline -v
   python scripts/bootstrap/install_elastic.py --check-only
   ```

3. Cùng Hoàng làm theo phần [khởi động lab](README.md) tới khi Elasticsearch, tài khoản và cấu hình ingest sẵn sàng. Trên stack mới chưa ingest, chụp E5 trước khi chạy Filebeat. Nếu Hoàng đã chạy lệnh ở đúng mốc, dùng chính báo cáo của Hoàng.

   ```powershell
   python scripts/verify/pipeline_evidence.py --phase bootstrap --output .runtime/evidence/pipeline-bootstrap.json
   ```

4. Mở báo cáo, đọc `pre_ingest_empty`, `template`, `ingest_pipeline` và số document của ba index thật. Kiểm tra xong thì Hoàng tiếp tục khởi động các dịch vụ còn lại. Dùng `docker compose ps` xác nhận các dịch vụ dài hạn đã hoạt động.

**Đạt khi** 14 unit test qua và bootstrap offline qua. Với runtime, E5 `PASS` và `pre_ingest_empty: true` với ba index thật đã có mapping, mỗi index có `document_count: 0`. Nếu đã có document, hẹn lượt stack sạch và ghi trạng thái mốc E5 hiện tại. Bàn giao cho Hoàng tên file E5, thời điểm UTC và mốc có thể bật Filebeat. [Bootstrap Elasticsearch tại commit nguồn](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/scripts/bootstrap/install_elastic.py)

### Ngày 2 — Viết pipeline Logstash tối giản (25 phút học, 75 phút làm, 20 phút kiểm)

1. Học phân biệt JSON bao ngoài do shipper gửi với chuỗi JSON nằm trong `message`. Tạo file `.runtime/tri-lab/mini.conf`. Pipeline học tập nhận fixture qua stdin, chuẩn hóa vài trường và in JSON ra stdout trong container không nối mạng.

   ```powershell
   New-Item -ItemType Directory -Force .runtime/tri-lab | Out-Null
   @'
   input { stdin { codec => json_lines } }
   filter {
     json { source => "message" target => "[@metadata][raw]" }
     if "_jsonparsefailure" not in [tags] {
       mutate {
         rename => {
           "[@metadata][raw][status]" => "[http][response][status_code]"
           "[@metadata][raw][request_id]" => "[request][id]"
         }
       }
       date { match => ["[@metadata][raw][time]", "ISO8601"] }
     }
     mutate { remove_field => ["message", "[event][original]"] }
   }
   output { stdout { codec => json_lines } }
   '@ | Set-Content -Encoding ascii .runtime/tri-lab/mini.conf

   $triLab = (Resolve-Path .runtime/tri-lab).Path
   $triImage = 'docker.elastic.co/logstash/logstash:9.5.3'
   $triInput = @{ message = '{"time":"2026-10-03T10:00:00+07:00","status":404,"request_id":"tri-study-01"}'; event = @{ dataset = 'nginx.access' }; labels = @{ synthetic = 'true' } } | ConvertTo-Json -Depth 5 -Compress
   $triInput | docker run --rm -i --network none -e 'LS_JAVA_OPTS=-Xms512m -Xmx512m' --mount "type=bind,source=$triLab,target=/lab,readonly" $triImage logstash --log.level=error --pipeline.workers=1 -f /lab/mini.conf
   ```

2. Đọc dòng output có `request.id` và so sánh với fixture. Giải thích tại sao `@metadata` không xuất hiện trong output. Thay `status:404` bằng `status:"404"` ở JSON bên trong. Xem vì sao pipeline học tập vẫn chuyển trường kiểu chuỗi, rồi đọc `number!` trong `infra/logstash/normalize.rb` để biết cách bản dự án bảo vệ mapping.
3. Thay giờ đầu vào thành giá trị sai, chạy lại và tìm `_dateparsefailure`. Giải thích vì sao một output có `@timestamp` vẫn có thể là lỗi parse giờ, vì Logstash gán giờ mặc định cho event khi nhận.

**Đạt khi** mẫu đúng in `event.dataset=nginx.access`, `labels.synthetic="true"`, `request.id=tri-study-01`, mã 404 kiểu số và giờ `03:00:00Z`. Trí có thêm hai mẫu quan sát lỗi kiểu dữ liệu và lỗi giờ, cùng kết quả cụ thể cho từng mẫu. [Codec JSON Lines của Logstash](https://www.elastic.co/docs/reference/logstash/plugins/plugins-codecs-json_lines)

### Ngày 3 — Mở rộng ba nhánh parse, xác thực và cách ly (20 phút học, 80 phút làm, 20 phút kiểm)

1. Tạo `.runtime/tri-lab/v2.conf` từ `mini.conf`. Giữ `input` và `output` như ngày 2. Thay khối `filter` bằng mẫu dưới đây. JSON được parse vào metadata trước khi lấy các trường cần thiết, còn error text được cắt theo định dạng dòng Nginx. Ca error thiếu `client:` giữ IP trống. Bài tập dùng stdout để nhìn thấy đích dự kiến qua `study_target`.

   ```logstash
   filter {
     if [event][dataset] == "nginx.access" or [event][dataset] == "shop.auth" {
       json { source => "message" target => "[@metadata][raw]" }
       if "_jsonparsefailure" not in [tags] {
         if [event][dataset] == "nginx.access" {
           mutate { rename => {
             "[@metadata][raw][status]" => "[http][response][status_code]"
             "[@metadata][raw][request_id]" => "[request][id]"
             "[@metadata][raw][remote_addr]" => "[source][ip]"
           } }
           date { match => ["[@metadata][raw][time]", "ISO8601"] }
         } else {
           mutate { rename => {
             "[@metadata][raw][event][action]" => "[event][action]"
             "[@metadata][raw][request][id]" => "[request][id]"
             "[@metadata][raw][source][ip]" => "[source][ip]"
           } }
           date { match => ["[@metadata][raw][@timestamp]", "ISO8601"] }
         }
       }
     } else if [event][dataset] == "nginx.error" {
       grok { match => { "message" => "^%{YEAR}/%{MONTHNUM}/%{MONTHDAY} %{TIME} \[%{LOGLEVEL:[log][level]}\] %{GREEDYDATA:[@metadata][error_body]}$" } }
       if "_grokparsefailure" not in [tags] {
         grok { match => { "message" => "^%{DATA:[@metadata][error_time]} \[" } }
         date { match => ["[@metadata][error_time]", "yyyy/MM/dd HH:mm:ss"] timezone => "UTC" }
         grok { match => { "[@metadata][error_body]" => "client: %{IP:[source][ip]}" } tag_on_failure => [] }
       }
     } else {
       mutate { add_tag => ["_invalid_dataset"] }
     }
     if "_jsonparsefailure" in [tags] or "_grokparsefailure" in [tags] or "_dateparsefailure" in [tags] or "_invalid_dataset" in [tags] {
       mutate { add_field => { "[pbl4][parse_failure]" => "true" "[pbl4][failure_reason]" => "%{[tags][0]}" "[@metadata][target_index]" => "logs-failed-lab-%{+YYYY.MM.dd}" } }
       mutate { convert => { "[pbl4][parse_failure]" => "boolean" } }
     } else if [labels][synthetic] == "true" {
       mutate { add_field => { "[@metadata][target_index]" => "logs-%{[event][dataset]}-%{+YYYY.MM.dd}-synthetic" } }
     } else {
       mutate { add_field => { "[@metadata][target_index]" => "logs-%{[event][dataset]}-lab-%{+YYYY.MM.dd}" } }
     }
     mutate {
       add_field => { "study_target" => "%{[@metadata][target_index]}" }
       remove_field => ["message", "[event][original]"]
     }
   }
   ```

2. Giải thích thứ tự các nhánh. Lỗi được ưu tiên trước định tuyến synthetic và log thật. So sánh với filter dự án trong `infra/logstash/normalize.rb`: dự án xác thực IP, status, kiểu dữ liệu và nhãn, sau đó dựng lại `event.original` cho log hợp lệ từ danh sách trường được phép. Với fixture học tập, stdout cho biết loại lỗi và tên index đích để tự kiểm từng bước.
3. Dùng một fixture cho mỗi nhánh và một JSON hỏng. Năm ID dưới đây có mẫu envelope trong `tests/fixtures/pipeline.json`. Gom các `input` thành năm dòng JSON và chuyển vào container. `$triLab` và `$triImage` dùng cùng giá trị như ngày 2.

   ```powershell
   $triFixtures = Get-Content -Raw -Encoding utf8 tests/fixtures/pipeline.json | ConvertFrom-Json
   $triIds = @('access-0', 'auth', 'error', 'error-no-ip', 'invalid-json')
   $triFixtures | Where-Object { $_.id -in $triIds } |
     ForEach-Object { $_.input | ConvertTo-Json -Depth 20 -Compress } |
     docker run --rm -i --network none -e 'LS_JAVA_OPTS=-Xms512m -Xmx512m' --mount "type=bind,source=$triLab,target=/lab,readonly" $triImage logstash --log.level=error --pipeline.workers=1 -f /lab/v2.conf
   ```
4. Đọc `source_ip!`, `number!` và phần `rescue` trong `normalize.rb`. Chạy test với filter thực tế của dự án.

   ```powershell
   python tests/pipeline/run_logstash.py --output .runtime/evidence/tri-parser.json
   ```

**Đạt khi** bản tập cho thấy ba nhánh success và một nhánh failed theo stdout. Bản kiểm tra filter thật báo `PASS` và `40/40` trên Docker. Chỉ ghi kết quả parse ở bước này vì harness tắt GeoIP và in stdout. Bàn giao cho Hoàng và Bảo bảng gồm input nào, dataset nào, dấu hiệu thành công và tên index đích. [Harness của Logstash tại commit nguồn](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/tests/pipeline/run_logstash.py)

### Ngày 4 — Tự tạo ingest pipeline, mapping và index (25 phút học, 75 phút làm, 20 phút kiểm)

1. So sánh `@timestamp` với `event.ingested`. Trường đầu là lúc sự kiện xảy ra, dùng cho cửa sổ truy vấn và hậu tố ngày UTC của index log hợp lệ. Trường sau được Elasticsearch gán lúc xử lý ingest. Học sự khác biệt giữa index template (áp mapping/settings lúc tạo index), ingest pipeline (biến đổi tài liệu) và ILM (vòng đời index). Policy `pbl4-logs-ilm` đặt giai đoạn xóa từ `7d`.
2. Trong Kibana Dev Tools của lab cá nhân, tạo ingest pipeline, template và index học tập. Tên bắt đầu bằng `study-pbl4-tri-` để tách khỏi namespace rule và dashboard. Chạy lần lượt bốn yêu cầu dưới đây.

   ```http
   PUT _ingest/pipeline/study-pbl4-tri-ingested
   {
     "processors": [{"set":{"field":"event.ingested","value":"{{{_ingest.timestamp}}}"}}]
   }

   PUT _index_template/study-pbl4-tri
   {
     "index_patterns": ["study-pbl4-tri-*"],
     "template": {
       "settings": {"index.default_pipeline":"study-pbl4-tri-ingested","number_of_replicas":0},
       "mappings": {"properties": {
         "@timestamp":{"type":"date"},
         "event":{"properties":{"ingested":{"type":"date"}}},
         "request":{"properties":{"id":{"type":"keyword"}}},
         "http":{"properties":{"response":{"properties":{"status_code":{"type":"long"}}}}},
         "source":{"properties":{"ip":{"type":"ip"},"geo":{"properties":{"location":{"type":"geo_point"}}}}},
         "labels":{"properties":{"synthetic":{"type":"keyword"}}}
       }}
     }
   }

   PUT study-pbl4-tri-000001

   PUT study-pbl4-tri-000001/_doc/study-1?refresh=wait_for
   {
     "@timestamp":"2026-10-03T03:00:00Z",
     "request":{"id":"tri-study-01"},
     "http":{"response":{"status_code":404}},
     "source":{"ip":"127.0.0.1"},
     "labels":{"synthetic":"true"}
   }
   ```

3. Đọc kết quả bằng `GET study-pbl4-tri-000001/_doc/study-1` và `GET study-pbl4-tri-000001/_mapping`. Lặp lại với ID khác và `source.ip: "not-an-ip"`, kỳ vọng HTTP 400 vì mapping `ip`. Đọc lại `GET study-pbl4-tri-000001/_count`, kỳ vọng một document nếu đây là index tập mới.
4. Đối chiếu `GET _index_template/pbl4-logs` và `GET _ingest/pipeline/pbl4-ingested` với file trong `infra/elasticsearch/`. Giải thích vì sao bản dự án tắt `manage_template` và `ilm_enabled` ở Logstash output khi bootstrap đã cài đối tượng trong Elasticsearch.

**Đạt khi** nhìn thấy `event.ingested` trên document học tập và giờ nguồn còn đúng giá trị ban đầu. Mapping có `long`, `ip`, `geo_point`, `date`, `keyword`; IP sai bị từ chối. Bàn giao Bảo kiểu trường và index pattern dùng trong query. [Index templates của Elasticsearch](https://www.elastic.co/docs/manage-data/data-store/templates)

### Ngày 5 — Mốc tuần 1 và bàn giao schema (20 phút học, 75 phút làm, 25 phút kiểm)

1. Viết bảng kiểu và nguồn cho `@timestamp`, `event.ingested`, `event.dataset`, `event.action`, `request.id`, `source.ip`, `source.geo.location`, `http.response.status_code`, `url.path`, `labels.synthetic`. Dùng `_field_caps` để phát hiện cùng trường có nhiều kiểu qua nhiều index.

   ```http
   GET logs-nginx.access-lab-*,logs-shop.auth-lab-*,logs-nginx.error-lab-*/_field_caps?fields=@timestamp,event.ingested,event.dataset,event.action,request.id,source.ip,source.geo.location,http.response.status_code,url.path,labels.synthetic
   ```

2. Ghi tên index theo thực tế: `logs-nginx.access-lab-*`, `logs-shop.auth-lab-*`, `logs-nginx.error-lab-*` là ba luồng thật. `logs-failed-lab-*` giữ chẩn đoán parser. Fixture hợp lệ được gắn `"true"` ở envelope dùng `logs-<dataset>-YYYY.MM.dd-synthetic` khi đi qua output của bản dự án. Query đo baseline và rule dùng luồng thật, kèm điều kiện `labels.synthetic == "false"`.
3. Đối chiếu `CONTRACT.md` với `normalize.rb`. Bản hiện tại dựng lại `event.original` đã giảm dữ liệu, không lấy query, referrer, XFF và tên đăng nhập từ raw để index. Nhánh lỗi giữ lý do hữu hạn và loại `event.original`. Trường `event.original` có `index:false` và `doc_values:false`, có thể đọc từ `_source` khi điều tra. Kiểm tra trên mẫu tự sinh của nhóm.
4. Cùng Bảo chốt giờ bắt đầu/kết thúc UTC để đo baseline 30 phút. Trong khung đó quan sát index access và schema, tránh chạy parser/drill hoặc tải riêng vào stack đang đo. Đọc lại kết quả mapping với Bảo và thống nhất dataset, thời gian và index pattern cho R1–R4.

**Đạt khi** Bảo tự chạy lại `_field_caps`, xác nhận `request.id` kiểu `keyword`, status kiểu `long`, IP kiểu `ip`, và dùng đúng index thật cho baseline. Hoàng và Bảo có bảng schema cùng lý do một HTTP 401 chưa đủ để chứng minh có `login_failed`. [Parser đang chạy tại commit nguồn](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/infra/logstash/normalize.rb)

## Tuần 2 — Truy vết, vận hành và bàn giao

### Ngày 6 — Tích hợp ba nguồn và truy tới index (20 phút học, 75 phút làm, 25 phút kiểm)

1. Nhận ba mẫu thực từ Hoàng. Dùng ID HTTP cho GET 404 và POST login sai 401. Với GET 403 dạng `/.pbl4-hoang-denied-<GUID>`, dùng URI đánh dấu và thời gian UTC để tìm `nginx.error`. Error Nginx trong lab không có `request.id`; access của chính request 403 vẫn có ID.
2. Trong Dev Tools, thay `ID_LOGIN_401` bằng ID nhận được và tra hai index với `term` trên `keyword`. Kỳ vọng một access 401 và một auth `login_failed` cùng ID. Đổi thành ID 404 và lặp lại, kỳ vọng một access 404 và không có auth cùng ID.

   ```http
   GET logs-nginx.access-lab-*,logs-shop.auth-lab-*/_search
   {
     "size": 10,
     "_source": ["@timestamp","event.ingested","event.dataset","event.action","http.response.status_code","request.id","url.path","labels.synthetic"],
     "query": {"bool":{"filter":[{"term":{"request.id":"ID_LOGIN_401"}},{"term":{"labels.synthetic":"false"}}]}}
   }
   ```

3. Với error, thay hai mốc dưới đây bằng cửa sổ UTC hẹp quanh lượt thử thật. Đọc `_source.event.original` của các hit và đối chiếu URI marker. Mở rộng `size` có kiểm soát nếu số hit vượt quá số trả về. Nếu mở lại bằng time picker của Kibana, giờ hiển thị theo `Asia/Ho_Chi_Minh` (UTC+7). Ví dụ `03:00Z` ứng với `10:00` cùng ngày tại Việt Nam.

   ```http
   GET logs-nginx.error-lab-*/_search
   {
     "size": 25,
     "_source": ["@timestamp","event.ingested","event.dataset","event.original","log.level"],
     "query":{"range":{"@timestamp":{"gte":"2026-10-03T03:00:00Z","lte":"2026-10-03T03:01:00Z"}}}
   }
   ```

4. Chạy verifier nếu có E5 từ cùng stack, cùng SHA và cho các index ngày UTC của lượt probe. Verifier tự tạo request riêng nên ID có thể khác mẫu thủ công. Nếu đã sang ngày UTC khác so với E5, chạy cặp bootstrap/trace liên tiếp trên stack sạch ở ngày 9 để chứng minh thứ tự.

   ```powershell
   python scripts/verify/pipeline_evidence.py --phase trace --bootstrap .runtime/evidence/pipeline-bootstrap.json --output .runtime/evidence/tri-day6-trace.json
   ```

**Đạt khi** bảng truy vết có nguồn, ID hoặc marker, index thực tế, giờ nguồn, giờ ingest và kết quả của từng mẫu. Bàn giao cho Bảo cùng truy vấn Dev Tools để mở lại trong Discover. Khi verifier chạy, cần `PASS`, `matched_access: 1`, `matched_auth: 1` và có ít nhất một error marker. [Collector và kiểm tra truy vết](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/scripts/verify/pipeline_evidence.py)

### Ngày 7 — GeoIP và cách đọc thiếu dữ liệu địa lý (30 phút học, 65 phút làm, 25 phút kiểm)

1. Vẽ cây quyết định của parser. IP nội bộ cho `not_public`, error thiếu IP cho `no_source_ip`. IP public đủ điều kiện để Logstash thử GeoIP; thành công có mã quốc gia và `resolved`, miss nhận `lookup_failure`. Ingest pipeline Elasticsearch chỉ thử fallback Country khi có IP, chưa có mã quốc gia và parser đã gán `lookup_failure`. Thành công fallback là `resolved_country_only` và chỉ có thông tin quốc gia.
2. Chạy native verifier để đọc trạng thái database và `_simulate` trên ingest pipeline thực. Dữ liệu mô phỏng nằm trong yêu cầu `_simulate`.

   ```powershell
   python scripts/verify/geoip.py --output .runtime/evidence/tri-geoip.json
   ```

3. Trong Kibana Dev Tools, chạy lại một ca public miss và một ca private. Đọc kết quả `docs[].doc._source`. Giữ địa chỉ public trong phạm vi fixture.

   ```http
   POST _ingest/pipeline/pbl4-ingested/_simulate
   {
     "docs": [
       {"_source":{"@timestamp":"2026-10-03T03:00:00Z","source":{"ip":"8.8.8.8"},"pbl4":{"geoip_status":"lookup_failure"},"labels":{"synthetic":"true"}}},
       {"_source":{"@timestamp":"2026-10-03T03:00:00Z","source":{"ip":"10.20.30.40"},"pbl4":{"geoip_status":"not_public"},"labels":{"synthetic":"true"}}}
     ]
   }
   ```

4. So sánh một log thật local của ngày 6 với hai mẫu `_simulate`. Đối với mẫu public, ghi mã quốc gia quan sát được từ database đang dùng. Nếu database chưa tải được, kiểm tra kết nối ra ngoài của Docker và `GET _ingest/geoip/stats`. Để trống tọa độ khi thiếu. Chia sẻ với Bảo cách giải thích `Unknown` trong dashboard và trường hợp có quốc gia nhưng chưa có `geo_point`.

**Đạt khi** báo cáo GeoIP ghi `PASS` cho 9 ca mô phỏng và hai mẫu giữ nguyên `@timestamp`. Mẫu public fallback có mã quốc gia và thiếu tọa độ; mẫu private giữ `not_public`. Nếu database chưa sẵn sàng, ghi `BLOCKED` với lý do và giữ kết quả public là chưa xác minh. [Verifier GeoIP tại commit nguồn](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/scripts/verify/geoip.py)

### Ngày 8 — Chẩn đoán lỗi parser và gián đoạn vận chuyển (20 phút học, 75 phút làm, 25 phút kiểm)

1. Mở báo cáo native parser ngày 3, xem `invalid-json`, `invalid-status`, `error-no-ip`, `synthetic` và `spoof-envelope`. Đối chiếu `expected_failed_routing`, `expected_target_prefix`, `original_sanitized`. Giải thích vì sao malformed JSON và status kiểu chuỗi vào failed, error hợp lệ thiếu IP vẫn vào `logs-nginx.error-lab-*`, và payload không ghi đè nhãn routing ở envelope.
2. Đọc ca `shop.auth-missing-ip-absent` trong `tests/fixtures/pipeline.json` và tìm kết quả của ca đó trong báo cáo harness gốc ngày 3. `PASS` cho biết filter thật đã trả `pbl4.failure_reason=invalid_source_ip` và đích `logs-failed-lab-*` theo kỳ vọng của fixture. Đưa cùng fixture vào v2 học tập để thấy khoảng cách giữa parse được JSON và xác thực được dữ liệu. V2 sẽ thiếu IP mà vẫn định tuyến theo dataset, còn parser dự án chủ động cách ly.
3. Cùng Hoàng làm một lượt stop/resume Filebeat trên lab nhóm. Hoàng điều khiển Filebeat, tạo 404 và lưu ID. Trí tra ID bằng truy vấn ngày 6 lúc dừng và sau khi chạy lại. Từ Logstash đọc counters trước và sau, để ý lượng healthcheck và sự kiện khác trong nền.

   ```powershell
   docker compose exec -T logstash curl --silent --show-error --fail http://127.0.0.1:9600/_node/stats/pipelines/main
   docker compose exec -T filebeat filebeat test output --strict.perms=false
   ```

4. Nếu marker chưa tới sau khôi phục, đi từ file Nginx và trạng thái Filebeat tới input Logstash, lỗi output Logstash, index/cửa sổ thời gian. Dùng `docker compose ps`, `docker compose logs --since 10m --tail 60 filebeat logstash` để xem riêng trên máy lab, rồi chỉ trích chẩn đoán đã kiểm. Có thể chạy stack verifier ở ngày 9 để kiểm cả mTLS, log rotation, Logstash outage và upstream phục hồi.

**Đạt khi** nhóm ghi thời điểm marker có ở nguồn, 0 hit trong lúc Filebeat dừng, rồi đúng 1 hit sau khi shipper phục hồi trong lượt thử. Trí giải thích riêng hai tình huống parser đã nhận rồi cách ly và pipeline đang chờ shipper. Bàn giao bảng gồm triệu chứng, lớp xác nhận, lệnh, kết quả, bước phục hồi. [Runbook local tại commit nguồn](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/docs/runbooks/local.md)

### Ngày 9 — Chạy lại với volume sạch và chốt bằng chứng (15 phút học, 85 phút làm, 20 phút kiểm)

1. Cùng Hoàng chọn Compose project mới trong bản clone lab và giải phóng cổng `8080`/`5601` trước khi chạy. Với project riêng, đặt `COMPOSE_PROJECT_NAME` trong cùng phiên PowerShell trước mọi lệnh của bài. Khởi động theo [trình tự chung](README.md). Tại mốc sau `install_elastic.py` và trước Filebeat, chạy bootstrap dưới đây.

   ```powershell
   python scripts/verify/pipeline_evidence.py --phase bootstrap --output .runtime/evidence/tri-day9-bootstrap.json
   ```

   Sau khi Hoàng khởi động stack đầy đủ, chạy trace trong cùng ngày UTC rồi mới chạy các kiểm thử gián đoạn. Trace tự tạo mẫu access, auth và error của lượt kiểm.

   ```powershell
   python scripts/verify/pipeline_evidence.py --phase trace --bootstrap .runtime/evidence/tri-day9-bootstrap.json --output .runtime/evidence/tri-day9-trace.json
   python scripts/verify/stack.py --output .runtime/evidence/tri-day9-stack.json
   python tests/pipeline/run_logstash.py --output .runtime/evidence/tri-day9-parser.json
   ```

2. Đọc status và exit code sau mỗi lệnh. Nếu chạy lại collector, dùng tên output mới trong `.runtime/evidence/` vì collector giữ nguyên file kết quả cũ. `stack.py` chủ động tạo traffic, thử gián đoạn Logstash, rotation và upstream fault có phục hồi. Chạy riêng theo phiên lab đã thống nhất, ngoài giờ đo baseline.
3. Đối chiếu trace E3 (mapping), E4 (các dòng của probe do verifier sở hữu), E5 (mốc bootstrap trước ingest). Báo cáo stack tách từng check. Tóm tắt theo SHA, thời điểm UTC, phạm vi, `PASS`/`FAIL`/`BLOCKED` và bằng chứng. Rà soát nội dung trước khi đưa trích đoạn vào bài trình bày.

**Đạt khi** E5 và trace cùng stack và cùng SHA qua, parser qua `40/40`, nhóm có kết quả từng check live hoặc lý do cụ thể còn mở. Bảo tái hiện được một truy vấn theo request ID của lượt thử, Hoàng kiểm tra dịch vụ nguồn và Filebeat hoạt động lại. [CI chạy bộ kiểm tại commit nguồn](https://github.com/hoangFan1005/PBL4-demo/blob/a653ae4245002148bcafffaa6040858848c2b484/.github/workflows/demo-ci.yml)

### Ngày 10 — Demo và đổi vai (20 phút ôn, 80 phút làm, 20 phút kiểm)

1. Trong lượt demo nhóm, Hoàng tạo 404 và đưa ID. Trí tìm document bằng `term`, chỉ ra `@timestamp`, `event.ingested`, dataset, status, mapping và index. Dùng mẫu 401 từ lượt chạy để tìm cặp access/auth và giải thích R2 chỉ đếm auth thất bại. Bảo mở Discover và dẫn tới một alert đã có tập `request.id` để đối soát.
2. Giải thích theo một mẫu mỗi loại vì sao log hợp lệ IP nội bộ vẫn được index, payload sai được cách ly, và GeoIP Country fallback có thể chỉ trả về quốc gia. Mở các báo cáo đã chạy để minh họa, ghi rõ báo cáo offline, mô phỏng native hoặc request thật.
3. Đổi vai. Bảo đọc ID và tìm index, Hoàng nói rõ tầng Logstash gán nhãn và kiểu dữ liệu, Trí tạo request 404 và dùng Discover để kiểm tra. Giữ thời gian tìm 404 dưới 5 phút tính từ lúc tạo request trong bài demo, bao gồm thao tác của người trình bày.
4. Nộp bảng schema, hai truy vấn theo ID, sơ đồ luồng và biên bản lỗi/khôi phục đã rà soát. Mỗi người tự trả lời điều gì xảy ra khi Filebeat dừng, khi Elasticsearch đã có document nhưng Kibana chưa hiện, và dữ liệu nào đủ để chứng minh cặp access/auth.

**Đạt khi** cả ba tự tìm được một request từ đầu vào tới tài liệu đã index và biết chuyển sang cửa sổ UTC đúng khi cần. Trí giao được chứng cứ để Bảo đối soát alert và Hoàng dùng lại bài chẩn đoán.
