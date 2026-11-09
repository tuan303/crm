# Bản nháp triển khai Twenty TEST: Vercel + Firebase PostgreSQL

**Trạng thái ngày 09/11/2026: cấu hình chuẩn bị, chưa sẵn sàng vận hành.** `vercel.json` đã được kiểm tra với schema Vercel; chưa build image, kết nối dữ liệu hoặc kiểm thử ứng dụng. Worker đang để `instances.hnd1 = 0`. Chỉ bật sau khi hoàn tất các điều kiện dưới đây.

## Đích triển khai

| Thành phần | Đích đã xác định |
| --- | --- |
| Repository / nhánh | `tuan303/crm` / [`test/vercel-firebase`](https://github.com/tuan303/crm/tree/test/vercel-firebase) — nhánh đã tạo |
| Vercel team / project | `tuan303-5380s-projects` / `crm-test` — đã tạo |
| Vercel Project ID | `prj_TwHerVhacSPE2NP2txSYXZGNZLMI` |
| Firebase project | `crm-test`, ID `crm-test-8b859` |
| SQL Connect service | `crm-test-8b859-service` |
| Cloud SQL instance / database | `crm-test-8b859-instance` / `crm-test-8b859-database` |
| Connection name | `crm-test-8b859:asia-northeast1:crm-test-8b859-instance` |
| PostgreSQL / vị trí | `POSTGRES_18`, `db-f1-micro`, Tokyo `asia-northeast1`; worker Vercel `hnd1` |

Lần kiểm tra gần nhất bằng `gcloud describe`: instance đã `RUNNABLE`, phiên bản `POSTGRES_18`; các cờ `cloudsql.iam_authentication` và `cloudsql.enable_google_ml_integration` đều là `on`. Public IPv4 bật nhưng `authorizedNetworks` trống. Chưa truy cập SQL hoặc xác minh kết nối từ Vercel. UI Firebase ghi thời hạn trial đến **07/02/2027**; cần kiểm tra lại quyền và thời hạn trước khi sử dụng.

Trong **project test riêng** `crm-test`, đặt Production Branch là `test/vercel-firebase`. Daemons hiện chỉ chạy ở target **Production** của Vercel; Preview không chạy worker. Target này vẫn thuộc hệ thống TEST. Xác nhận đúng team/project/branch trước mọi thao tác deploy.

## Điều kiện bắt buộc

1. **PostgreSQL sẵn sàng và truy cập được từ container Vercel.** Twenty dùng kết nối PostgreSQL trực tiếp, không dùng Firestore hoặc Firebase Web SDK thay thế DB. Với cấu hình mạng hiện tại, phải hoàn thiện phương thức Cloud SQL Auth Proxy/connector hoặc đường kết nối được cho phép; bản nháp này chưa đóng gói proxy/connector. Chưa thay đổi quyền DB hay cấu hình TLS.
2. **DB test đã khởi tạo.** Xác nhận tài khoản có quyền tạo schema và các extension `uuid-ossp`, `unaccent`, `citext`, cùng quyền cần cho migrations của phiên bản này. Chạy setup, migrations và đăng ký cron riêng trước khi web/worker sử dụng DB.
3. **Redis ngoài container** đã kiểm tra bằng giao thức Redis/TCP, phù hợp BullMQ. Dùng cùng Redis test cho web và worker; endpoint REST không thay thế trực tiếp `REDIS_URL`.
4. **Bucket S3 hoặc tương thích S3 riêng cho TEST** đã kiểm tra ghi/đọc/xóa. Web và worker dùng cùng bucket. Không dùng local storage của container làm nơi lưu tệp bền vững.
5. **Khóa ổn định:** tạo và lưu riêng `APP_SECRET`, `ENCRYPTION_KEY`; dùng cùng giá trị cho web, worker và tiến trình migration. Giữ nguyên qua các lần deploy; thay khóa phải theo quy trình rotation của Twenty.

## Biến môi trường

Nhập qua phần Environment Variables của đúng project Vercel và target Production. Không đưa credential vào Git, tài liệu hoặc Docker build context.

| Biến | Giá trị / yêu cầu |
| --- | --- |
| `NODE_ENV` | `production` — chế độ runtime, dù đây là hệ thống test |
| `PORT`, `NODE_PORT` | Cùng đặt `3000`: Vercel định tuyến theo `PORT`, Twenty lắng nghe `NODE_PORT` |
| `SERVER_URL` | URL HTTPS ổn định thực tế của project test; chưa dùng URL minh họa |
| `PG_DATABASE_URL` | URL PostgreSQL đã kiểm tra từ môi trường tương đương Vercel |
| `PG_SSL_ALLOW_SELF_SIGNED` | Giữ mặc định `false`/không đặt; xác minh TLS theo cách kết nối đã chọn |
| `REDIS_URL` | URL Redis test thực tế, dùng TLS khi endpoint yêu cầu |
| `APP_SECRET`, `ENCRYPTION_KEY` | Secret riêng, đủ entropy, ổn định và thống nhất giữa các tiến trình |
| `STORAGE_TYPE` | `S_3` |
| `STORAGE_S3_REGION`, `STORAGE_S3_NAME` | Region và tên bucket test thực tế |
| `STORAGE_S3_ENDPOINT` | Chỉ đặt khi nhà cung cấp tương thích S3 yêu cầu endpoint riêng |
| `STORAGE_S3_ACCESS_KEY_ID`, `STORAGE_S3_SECRET_ACCESS_KEY` | Credential giới hạn vào bucket test, hoặc cơ chế credential được SDK hỗ trợ và đã kiểm tra |
| `STORAGE_S3_PRESIGNED_URL_ENABLED` | Đặt `true` sau khi cấu hình bucket CORS cho đúng `SERVER_URL` và kiểm tra upload trực tiếp |
| `DISABLE_DB_MIGRATIONS`, `DISABLE_CRON_JOBS_REGISTRATION` | `true` cho runtime web/worker, sau khi setup riêng thành công |

Vercel Functions giới hạn request/response body 4,5 MB; upload trực tiếp bằng presigned URL cần được kiểm thử. Chưa coi mọi luồng tải tệp của Twenty là tương thích chỉ vì trang đăng nhập mở được.

## Thứ tự triển khai

1. Xác nhận toàn bộ điều kiện và biến môi trường. Chọn region HTTP Functions Tokyo (`hnd1`) trong project để gần DB; kiểm tra lại region thực tế sau build.
2. Với **DB test mới**, chạy riêng các lệnh sau trong image target `twenty-server`, tại `/app/packages/twenty-server`, bằng đúng các biến môi trường test:

   ```sh
   yarn database:init:prod
   yarn command:prod upgrade
   yarn command:prod cache:flush
   yarn command:prod cron:register:all
   ```

   Dừng nếu một lệnh lỗi; xác nhận migrations hoàn tất. Không suy ra thành công chỉ từ container đang chạy: entrypoint upstream tiếp tục khi một số bước upgrade/cron lỗi. Với DB đã có dữ liệu, dùng quy trình nâng cấp phù hợp thay vì tự chạy lại initialization.
3. Build và kiểm tra cả hai target. Cấu hình dùng `Dockerfile#twenty` cho web/UI và `Dockerfile#twenty-server` cho worker; không dùng target mặc định `twenty-app-dev` vốn chứa DB/Redis cục bộ.
4. Khi DB/Redis/S3 và build đã đạt, đổi `daemons.worker.instances.hnd1` từ `0` thành `1`, rồi triển khai target Production của **project test**. Giữ các biến disable migration/cron đã nêu. Worker gọi `node dist/queue-worker/queue-worker` trực tiếp nên không chạy entrypoint khởi tạo DB.
5. Xác nhận `/healthz`, đăng nhập, tạo/sửa một bản ghi test, xử lý một tác vụ nền và upload/download tệp. Kiểm tra dữ liệu và tệp còn tồn tại sau một lần deploy lại; xem log web/worker để loại trừ lỗi DB, Redis, quyền hoặc mã hóa.
6. Khi dừng đợt test, đưa worker về `0` và kiểm tra instance đã dừng; tạm dừng project nếu không cần web. Việc dừng worker không tự dừng DB/Redis/storage hoặc kết thúc trial của dịch vụ khác.

Không dùng daemon kết thúc ngay làm job setup một lần: Vercel khởi động lại daemon đã thoát và không có `depends_on` để web chờ migration. `terminationGracePeriodSeconds = 360` cho phép thời gian drain mặc định 300 giây của worker hoàn tất trước khi bị kết thúc.

## Những điểm cần xác minh khi build

- Docker build context là gốc repo. Vercel build từng service/daemon; checkout phải chứa đầy đủ các package và Yarn release/patch mà Dockerfile `COPY`. Node 26 được giữ trong image upstream.
- Không đặt top-level `framework`, `buildCommand`, `installCommand`, `outputDirectory` hoặc `functions` khi dùng `services`. Không chọn thư mục `public`/gốc repo làm output tĩnh: UI nằm trong image web, routing đi vào service `web`.
- Root của daemon là `.`; cấu hình này chỉ có container services, không xuất repository thành static assets. `.dockerignore` hiện không loại mọi biến thể `.env.*`; giữ file secret ngoài checkout/build context.
- Schema hợp lệ không bảo đảm build đủ RAM, image nằm trong giới hạn hoặc ứng dụng kết nối thành công. Chưa đo runtime memory/pool size với instance DB `db-f1-micro`.
- Services/Daemons đang Beta. Hobby có 420 GB-hours RAM/tháng cho daemon; một instance 2 GiB chạy liên tục tiêu hết khoảng 9 ngày. Bản nháp không cam kết vận hành liên tục miễn phí.

## Nguồn đối chiếu

- [Schema Vercel hiện hành — cú pháp `#stage`](https://openapi.vercel.sh/vercel.json)
- [Services và phạm vi cấu hình](https://vercel.com/docs/services)
- [Daemon configuration](https://vercel.com/docs/daemons/config-reference) · [Giới hạn Daemons](https://vercel.com/docs/daemons/limits)
- [Container Runtime](https://vercel.com/docs/functions/runtimes/container) · [Giới hạn Functions](https://vercel.com/docs/functions/limitations)
- Source của nhánh này: `packages/twenty-docker/twenty/Dockerfile`, `packages/twenty-docker/twenty/entrypoint.sh`, `packages/twenty-server/package.json`, `packages/twenty-server/src/engine/core-modules/twenty-config/config-variables.ts`, `packages/twenty-server/src/database/scripts/setup-db.ts`.
