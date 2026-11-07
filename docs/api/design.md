# BlueMoon — Quy ước và nghiệp vụ API

Phiên bản `1.1.0-sprint1`, ngày 06/11/2026. Bao phủ 36 thực thể ERD 00–06,
bao gồm [ERD thanh toán QR/ví](../ERD_Preview/ERD_06.svg), cùng 4 model bảo trì bổ sung.
**94/146 thao tác đã triển khai.** Phạm vi và quyết định triển khai tại
[bàn giao Sprint 1](../sprint-1-hiep.md); OpenAPI đánh dấu `x-implemented` từng thao tác.

- [OpenAPI 3.0.3](openapi.json): schema request/response, header, filter, HTTP và quyền.
- [146 thao tác](endpoints.md): danh mục theo nhóm.
- [36 thực thể và DTO](data-dictionary.md): đối chiếu đầy đủ cột ERD, kiểu, nullable.
- [API hệ thống đã triển khai](../api-contract.md#api-hệ-thống-đã-triển-khai).

## 1. Nguồn và giả định

ERD cung cấp kiểu dữ liệu, PK/FK/UQ, nullable. Endpoint, enum, quyền, công thức
phí, chuyển trạng thái và giao thức gateway là **đề xuất thiết kế** cần thống
nhất khi triển khai. File Thiet_ke_va_doi_chieu.md ghi trong ERD không có trong
repo; contract không giả định đã đọc file đó. OpenAPI mô tả cấu trúc; các bất
biến liên bản ghi/ngày/số dư dưới đây phải được kiểm tra ở tầng nghiệp vụ.

Bảng con không có CRUD tùy ý: allocation, intent item, event, receipt,
recipient, ticket update, audit và bảng nối RBAC được quản lý qua nghiệp vụ cha.
Unique liên cột, khóa số dư, idempotency store, outbox và session/token version
là yêu cầu triển khai bổ sung, chưa được thể hiện đầy đủ trong ERD.

## 2. HTTP và dữ liệu

Base local `http://localhost:8080/api`; frontend dùng /api qua Vite proxy.
JSON UTF-8 camelCase, Content-Type application/json. Response trực tiếp resource,
không bọc data. POST tạo trả 201 + Location (bảng con có thể trỏ resource cha);
lệnh trả 200; không có body trả 204. Chỉ cho DELETE dòng invoice DRAFT.

| Dữ liệu | Quy ước |
| --- | --- |
| ID | UUID; server cấp ID, người thao tác, thời điểm hệ thống và số chứng từ |
| occupancyId | Tham chiếu household_apartment.id |
| Tiền | Chuỗi decimal 2 số như `"150000.00"`; tính decimal, không float |
| VND | Số tiền thực thu có đuôi .00; HALF_UP đến đồng trên từng dòng |
| Diện tích / số lượng | Chuỗi 2 / 3 số thập phân, > 0, đúng precision ERD |
| Ngày | YYYY-MM-DD, nghiệp vụ theo Asia/Ho_Chi_Minh |
| Thời điểm | RFC 3339 có timezone, response UTC |
| Hiệu lực | [validFrom,validTo) hoặc [effectiveFrom,effectiveTo); null chưa kết thúc |
| Kỳ / chiến dịch | startDate và endDate bao gồm hai đầu |
| Lọc timestamp | from bao gồm, to không bao gồm |
| Nullable | Response giữ field null; request chỉ null ở field nullable |
| PATCH | Field vắng giữ nguyên, null xóa nếu cho phép; body rỗng trả 400 |
| Chuỗi | Trim, field bắt buộc không toàn khoảng trắng; username không phân biệt hoa thường |
| Field thừa | Trả 400, kể cả field server quản lý do client gửi |

Danh sách: page=0, size=20 (1–100), sort=id,asc mặc định. Một tiêu chí sort là
field vô hướng của resource, ID làm tie-break cùng chiều. Filter kết hợp AND;
field/filter/sort không hỗ trợ trả 400. Boolean, UUID, enum và ngày phải đúng
kiểu. q tìm chứa tên cư dân không phân biệt hoa thường, tối đa 120 ký tự.

```json
{"content":[],"page":0,"size":20,"totalElements":0,"totalPages":0}
```

Server tạo requestId và trả X-Request-Id. Lỗi theo Error, ngoại lệ health giữ
response hiện có. Không lộ stack trace/password/token/secret/JDBC/callback raw.

```json
{
  "code":"INSUFFICIENT_OUTSTANDING_BALANCE",
  "message":"Số tiền phân bổ vượt công nợ còn lại.",
  "requestId":"req-20261105-001",
  "timestamp":"2026-11-05T13:30:00Z",
  "path":"/api/payments",
  "details":[{"field":"allocations[0].amount","message":"Còn nợ 150000.00 VND."}]
}
```

| HTTP | Code tiêu biểu | Ý nghĩa |
| --- | --- | --- |
| 400 | INVALID_REQUEST, UNKNOWN_FIELD, MISSING_IDEMPOTENCY_KEY | Sai JSON, field, header, query |
| 401 | AUTHENTICATION_FAILED, TOKEN_EXPIRED, INVALID_GATEWAY_SIGNATURE | Xác thực thất bại |
| 403 | PERMISSION_DENIED | Thiếu quyền thao tác |
| 404 | RESOURCE_NOT_FOUND | Không tồn tại hoặc ngoài phạm vi được xem |
| 409 | DUPLICATE_RESOURCE, INVALID_STATE, VERSION_CONFLICT, IDEMPOTENCY_CONFLICT, TARGET_RESERVED, INSUFFICIENT_OUTSTANDING_BALANCE | Trùng/xung đột trạng thái hoặc số dư |
| 422 | BUSINESS_RULE_VIOLATION, INVALID_ALLOCATION_TARGET, INVALID_DATE_RANGE, EXPORT_TOO_LARGE | Sai nghiệp vụ |
| 429 | RATE_LIMITED | Retry-After tính bằng giây |
| 500 | INTERNAL_ERROR | Lỗi bất ngờ, không lộ chi tiết nội bộ |
| 503 | DEPENDENCY_UNAVAILABLE | Database/gateway/kênh gửi không sẵn sàng |

## 3. Xác thực và phân quyền

Login cấp Bearer token 1800 giây, giới hạn 5 lần/phút theo IP và username. Sai
username/password hoặc user không ACTIVE cùng trả 401, không dò được tài khoản.
Không có public signup, refresh token hoặc email quên mật khẩu. Logout client
xóa token; đổi/reset password phải vô hiệu hóa token cũ bằng session/token
version bổ sung ngoài ERD. User không ACTIVE bị chặn ngay; role/quyền thay đổi
có hiệu lực từ request kế tiếp, không chỉ tin claim cũ.

Quyền từng endpoint ở x-permission. AUTHENTICATED không phải permission lưu DB;
GATEWAY_SIGNATURE yêu cầu chữ ký adapter. Catalog permission do migration/seed
quản lý, API chỉ đọc. Có thêm ticket.handle và resident.identity.read cho kiểm
tra nghiệp vụ. Vai trò khởi tạo đề xuất:

| Role | Nhóm quyền seed | Phạm vi |
| --- | --- | --- |
| ADMIN | Toàn bộ catalog | Hệ thống |
| MANAGER | Cư trú, phí, kỳ/bảng kê, đóng góp, thu hộ, thông báo, ticket và report; duyệt/phát hành/khóa/lập/hủy bảng kê | Chung cư, không tự có RBAC hoặc hủy thu tiền |
| ACCOUNTANT | Đọc cư trú; quản lý phí/kỳ/bảng kê/đóng góp/thu hộ/đối soát; payment.read/record/void, receipt.read, payment-intent.read, payment-event.read, notice.read/remind và report | Chung cư |
| AUTHORITY | building.read, apartment.read, household.read, occupancy.read, resident.read, resident.identity.read, residence-change.read | Tra cứu cư trú toàn chung cư; không ghi và không xem tài chính |
| RESIDENT | Đọc dữ liệu hộ/căn hộ/cư trú/xe/phí/kỳ/bảng kê/đóng góp/thu hộ/thanh toán; residence-change.read/write, contribution.read/write, payment-intent.read/create, notice.read, ticket.read/write/update | Hộ của tài khoản |

Bảng là hướng dẫn seed, không phải wildcard runtime; gán các code cụ thể trong
OpenAPI. Không cho cư dân thu tiền thủ công, duyệt yêu cầu, đọc callback event,
ghi trực tiếp cư trú/giá/bảng kê hay xem báo cáo chung cư.

Phạm vi hộ: app_user.resident_id → household_member còn hiệu lực theo ngày
hiện tại. User nhân viên residentId null vẫn đăng nhập được. Invoice/subscription
kiểm tra hộ qua occupancy; householdId từ client không chứng minh quyền.
Khi membership kết thúc, cư dân mất quyền hộ, quản lý hỗ trợ tra cứu lịch sử.
Lọc phạm vi trước đếm và phân trang. Che identityNo của người khác trừ chính
chủ hoặc resident.identity.read. Thông báo chỉ người nhận hoặc notice.write;
đánh dấu đọc chỉ user chủ recipient. Cư dân chỉ ticket của hộ. Không có vai trò STAFF. Ngoài phạm vi xem trả 404. Audit chỉ audit.read được xem sau khi che.

## 4. Căn hộ và cư trú

Building.code unique; apartment.code unique trong building; household.code
unique. IdentityNo unique khi khác null, resident gắn tối đa một user.
Một căn hộ tối đa một occupancy hiệu lực; hộ có thể có nhiều căn hộ. Cùng cặp
hộ/căn hộ không chồng khoảng. Resident tối đa một membership hiệu lực, hộ tối
đa một chủ hộ hiệu lực. Chuyển chủ hộ nguyên tử, giữ lịch sử cư trú.
Ngày kết thúc > bắt đầu, birthDate không ở tương lai. Kết thúc occupancy cần
đóng subscription còn hiệu lực. VACANT/OCCUPIED suy ra từ occupancy; INACTIVE
là trạng thái vận hành, chưa có lệnh công khai chuyển trong bản này. Đóng hộ
không còn membership/occupancy hiệu lực, nợ hoặc intent PENDING.

ResidenceChange.details có cấu trúc theo changeType:

| changeType | Field bắt buộc | Khi duyệt |
| --- | --- | --- |
| MOVE_IN | residentId, details.relationship/isHead; details.apartmentId nếu mở occupancy mới | Mở membership; chỉ mở occupancy mới khi có apartmentId và chưa có occupancy đó |
| MOVE_OUT | residentId, details.endOccupancyIds (array, có thể rỗng) | Kết thúc membership và occupancy chỉ định tại effectiveFrom |
| TRANSFER | residentId, targetHouseholdId, details.relationship/isHead | Đóng membership nguồn, mở hộ đích khác hộ nguồn; không chuyển nợ |
| TEMPORARY_ABSENCE | residentId, effectiveTo | Lưu tạm vắng, giữ membership/occupancy |
| TEMPORARY_RESIDENCE | residentId, effectiveTo, details.relationship/isHead | Mở membership có validTo ở householdId |

Field details ngoài danh sách tương ứng trả 422; decisionReason do server ghi.
Cư dân chỉ tạo yêu cầu cho hộ mình, người duyệt kiểm tra hộ đích. Tạm vắng vẫn
tính phí nhân khẩu. PENDING → APPROVED/REJECTED/CANCELLED, trạng thái cuối bất
biến; người tạo không tự duyệt. Khóa và cập nhật lịch sử/audit cùng transaction.

## 5. Thu phí và đóng góp

Biển số chuẩn hóa unique giữa đăng ký hiệu lực. FeeRate không chồng khoảng theo
feeTypeId/vehicleType. Giá >= 0, đã dùng không sửa, thêm bản mới khi đổi giá.
PER_VEHICLE cần vehicleType, basis khác null. FeeType.basis/mandatory bất biến
sau sử dụng. SnapshotDate thuộc kỳ; dueDate >= endDate. Invoice unique theo
occupancyId/periodId, utilityBillId unique trên dòng nếu khác null. Mã chứng từ
server cấp unique, không tái sử dụng.

Lập bảng kê lấy snapshot occupancy/diện tích/thành viên/xe/giá tại snapshotDate,
bill theo kỳ. Hộ nhiều căn: PER_PERSON/PER_VEHICLE chỉ vào occupancy có validFrom
sớm nhất, hòa thì ID tăng dần; phí căn/diện tích riêng từng occupancy. Đây là
giả định nghiệp vụ cần thống nhất. Tạm vắng vẫn tính vào membership ngày chốt.

| Basis | quantity | unitPrice |
| --- | --- | --- |
| PER_APARTMENT | 1.000 | Giá/căn |
| PER_M2 | Diện tích | Giá/m² |
| PER_PERSON | Số thành viên | Giá/người |
| PER_VEHICLE | 1.000 mỗi xe, một dòng/xe | Giá theo loại xe |
| ACTUAL | 1.000 | Amount hóa đơn thu hộ |

Amount server tính quantity × unitPrice, HALF_UP tới đồng. Rate/vehicle/bill
phải khớp hộ/kỳ/feeType, thiếu dữ liệu trả 422, không bỏ qua. Phát hành cần ít
nhất một dòng, không tiền âm. Invoice có lines, totalAmount, paidAmount và
outstandingAmount dẫn xuất. Dòng đã phát hành giữ giá snapshot.

Kỳ DRAFT → ISSUED → CLOSED; phát hành toàn kỳ nguyên tử. Invoice DRAFT → ISSUED
→ PARTIALLY_PAID → PAID. DRAFT/ISSUED chưa thu được CANCELLED; hủy payment có thể
đưa về ISSUED/PARTIALLY_PAID. Chỉ sửa/xóa dòng khi cả kỳ và invoice DRAFT.
Kỳ CLOSED vẫn thu nợ cũ. Thêm/sửa/xóa dòng gửi If-Match: "3" với version hiện
tại; thiếu trả 400, cũ trả 409. Đổi dòng/trạng thái/số dư tăng invoice.version;
hủy invoice gửi version trong body. GET lại Invoice để lấy version mới.

Contribution unique campaignId/householdId; suggested/pledgedAmount nullable,
khi có >= 0. Cam kết không là nợ bắt buộc, có thể đóng ít hoặc vượt cam kết.
Chỉ thu trong ngày chiến dịch; không thu hẹp ngày loại trừ thực thu cũ. Thu qua
PaymentAllocation, không cập nhật paidAmount từ client.

## 6. Thu tiền và biên lai

POST /payments chỉ CASH/BANK_TRANSFER, BANK_TRANSFER yêu cầu externalRef chưa
dùng. PaidAt không tương lai hoặc sau khi hộ đóng. Chỉ thu bảng kê phát hành
trong kỳ ISSUED/CLOSED. Allocation đúng một invoiceLineId/contributionId khác
null, amount > 0, không trùng đích, tất cả cùng hộ. Tổng allocation = payment
amount, không có tiền dư chưa phân bổ. Không thu dòng bảng kê vượt số dư;
contribution không dùng cam kết làm hạn mức.

Số dư = amount dòng - allocation của payment POSTED; VOIDED không tính.
Khóa các đích theo ID ổn định và kiểm tra số dư sau khóa. Intent PENDING giữ
chỗ đích tới hết hạn/hủy; lệnh thu thủ công hoặc intent khác đụng đích trả 409
TARGET_RESERVED. Giữ chỗ contribution giúp tránh trả hai lần ngoài ý muốn;
giữ chỗ không giảm nợ hiển thị. Ghi payment/allocations/receipt/số dư/audit nguyên tử.

Receipt.paymentId unique, tự tạo một biên lai/payment. Snapshot bất biến gồm
schemaVersion, paymentNo, receiptNo, householdCode, apartmentCodes,
payerDisplayName, channel, amount, currency, paidAt, issuedAt,
recordedByDisplayName nullable và allocations (đích/mô tả/số tiền). Đổi tên/giá/
địa chỉ sau này không đổi biên lai cũ. VOIDED giữ snapshot nhưng xem/PDF phải có
dấu HỦY từ payment status.

Hủy ghi nhận thủ công POSTED → VOIDED cần reason và payment.void; đảo ảnh hưởng
số dư, giữ lịch sử. Không hủy QR/ví hoặc khoản đã vào đối soát CONFIRMED. Đây
là sửa sổ nội bộ, không tự hoàn tiền ngân hàng.

### Idempotency

Header Idempotency-Key bắt buộc tại endpoint được đánh dấu, 1–80 ký tự
`[A-Za-z0-9._:-]`, unique toàn hệ thống, không dùng lại giữa các lệnh.
Payment/intent lưu key vào cột ERD. Fingerprint gồm actor/method/path/JSON
canonical sau chuẩn hóa. Cùng key/fingerprint trả status/resource/Location cũ,
không tạo chứng từ/gọi gateway/gửi thông báo lần hai. Khác payload/actor/route
trả 409 IDEMPOTENCY_CONFLICT. Lệnh cùng key đồng thời được tuần tự hóa; đang xử
lý có thể trả 409 REQUEST_IN_PROGRESS + Retry-After. Không cache lỗi validation/
xác thực. Key tài chính giữ theo chứng từ, lệnh khác tối thiểu 24 giờ.
Timeout sau commit hoặc gateway nhận order: retry cùng key, server tra
merchantOrderId trước tạo mới. Store/outbox bổ sung ngoài ERD cần bảo đảm điều này.

```http
POST /api/payments
Authorization: Bearer <accessToken>
Idempotency-Key: manual-20261105-001
Content-Type: application/json

{
  "householdId":"00000000-0000-4000-8000-000000000001",
  "channel":"CASH",
  "amount":"150000.00",
  "paidAt":"2026-11-05T13:30:00Z",
  "allocations":[{
    "invoiceLineId":"00000000-0000-4000-8000-000000000002",
    "amount":"150000.00"
  }]
}
```

Thay UUID bằng dữ liệu thật. Response 201 Payment có POSTED, intentId null,
allocations do server cấp ID; đọc biên lai ngay sau commit.

## 7. Thu hộ và đối soát

Provider là nhà cung cấp dịch vụ, không phải gateway catalog. Subscription gắn
occupancy/provider, customerCode không trùng cùng provider trong khoảng chồng
lấn. Đổi hộ đóng và tạo subscription mới. Bill yêu cầu subscription hiệu lực
tại ngày chốt. Server tính sourceHash SHA-256 UTF-8 của
`subscriptionId|periodId|providerBillNo|amount` đã chuẩn hóa; unique bộ ba định
danh, cùng hash duplicate, khác hash 409. SourceHash/importedAt do server ghi.
Nhập JSON tối đa 1000 dòng nguyên tử, lỗi ghi details theo index. UtilityBill
bất biến, mỗi bill tối đa một dòng bảng kê; không có API ghi đè, hóa đơn sai cần
quy trình sửa nghiệp vụ riêng trước phát hành.

Reconciliation unique providerId/periodId, DRAFT sinh item từ bill nhà cung cấp/
kỳ, unique reconciliationId/utilityBillId. Billed lấy bill, collected từ
allocation POSTED; kế toán nhập remitted trong [0,collected]. Confirm khóa
payment liên quan, tính lại và chuyển CONFIRMED, item bất biến; không void tiền
đã góp số liệu chốt. Thu muộn không sửa snapshot, báo cáo cho thấy chưa chuyển;
đối soát bổ sung cần quy trình ở phiên bản sau, không coi thu muộn là đã chuyển.

## 8. QR/ví và callback

Client tạo intent → hiển thị QR/checkout → gateway callback → server ghi tiền
→ GET intent lấy paymentId → biên lai. Redirect không chứng minh đã trả tiền.
Server tính expectedAmount = tổng items, sinh merchantOrderId, expiresAt sau
15 phút, version 0; currency VND, providerCode theo cấu hình server. Channel
QR/EWALLET giữ qua metadata lệnh/adapter, ERD intent không có cột channel.
Items/số tiền bất biến. PENDING → SUCCEEDED/FAILED/EXPIRED/CANCELLED; trạng thái
cuối bất biến. Hết hạn bằng lock/compare-and-set, hủy cần version. SUCCEEDED
chỉ sau commit payment, luôn có paymentId.

### Adapter chuẩn hóa đề xuất

POST /payment-gateways/{providerCode}/events là giao thức adapter nội bộ,
chưa phải callback trực tiếp MoMo/VNPay. Adapter xác thực native rồi ánh xạ
sự kiện. X-Gateway-Timestamp là Unix seconds, X-Gateway-Signature là lowercase
hex HMAC-SHA256(secret, timestamp + "." + rawRequestBody). Secret riêng provider,
raw UTF-8 bytes không serialize lại, constant-time compare, lệch tối đa 300 giây.
Retry timestamp/chữ ký mới, giữ eventId và nguyên bytes body. Không đưa secret
vào URL/frontend/log.

```json
{
  "eventId":"evt-20261105-001",
  "merchantOrderId":"BM-20261105-000001",
  "providerTransactionId":"txn-20261105-001",
  "status":"SUCCEEDED",
  "amount":"150000.00",
  "currency":"VND",
  "paidAt":"2026-11-05T13:35:00Z"
}
```

| Trường hợp | HTTP / result | Tác động |
| --- | --- | --- |
| Sai JSON/thiếu header | 400 Error | Không ghi tiền |
| Sai chữ ký/timestamp | 401 Error | Chỉ log bảo mật đã che |
| Provider chưa cấu hình | 404 Error | Không ghi tiền |
| Order không tồn tại | 422 UNKNOWN_MERCHANT_ORDER | Không tạo event do intentId bắt buộc, chỉ log bảo mật |
| Thành công PENDING chưa hết hạn | 200 PROCESSED | Payment/receipt, intent SUCCEEDED |
| FAILED cho PENDING | 200 PROCESSED | Intent FAILED, giải phóng giữ chỗ |
| Cùng provider/eventId và hash | 200 DUPLICATE | Không ghi tiền lần hai |
| EventId cũ, hash khác | 409 EVENT_PAYLOAD_CONFLICT | Không ghi tiền, cảnh báo audit |
| Lệch amount/currency/provider/transaction; thành công sau hết hạn/hủy/thất bại; trạng thái cuối mâu thuẫn | 200 REVIEW_REQUIRED | Lưu event, không tự ghi tiền/đổi trạng thái cuối |
| Dữ liệu nghiệp vụ sai khác, ví dụ paidAt tương lai | 200 REJECTED | Lưu event, không ghi tiền |
| DB lỗi trước commit | 503 Error | Rollback, retry cùng eventId |

Ack `{"eventId":"evt-20261105-001","result":"PROCESSED"}`. Deduplicate unique
(providerCode,eventId), payloadHash SHA-256 raw body không gồm timestamp header.
Xác thực trước kiểm tra duplicate. Một providerTransactionId thành công không
gắn hai intent. Khóa intent rồi đích theo ID, kiểm tra lại tiền/thời hạn/số dư;
event/intent/payment/allocations/receipt/audit cùng transaction. Payment.intentId
unique ngăn hai event tạo hai payment. Intent SUCCEEDED nhận event thành công
khác ID nhưng cùng transaction/amount/currency là DUPLICATE, khác tham chiếu
đưa review. Event API chỉ đọc metadata, không raw/chữ ký, không ép SUCCEEDED.

## 9. Thông báo và ticket

Notice/recipients nguyên tử, gửi outbox sau commit; unique(noticeId,userId,channel).
FEE_REMINDER bắt buộc periodId; reminderKey theo kỳ/hộ/ngày VN, tối đa một
notice/hộ/ngày còn nợ. PENDING → SENT/FAILED, tổng attempts tối đa 5, backoff
1/5/15/60 phút, retry thủ công cùng giới hạn. Kênh chưa cấu hình không báo SENT.
IN_APP SENT là đã trong hộp thư, không đồng nghĩa đọc; chủ user ghi readAt lần
đầu, gọi lại giữ nguyên. Không lộ nợ hộ khác trong thông báo.

Ticket OPEN → IN_PROGRESS → RESOLVED → CLOSED. Người có ticket.handle có thể mở
lại RESOLVED → IN_PROGRESS với comment. CLOSED là cuối; comment cùng trạng thái
vẫn tạo update, không sửa/xóa lịch sử. Cư dân chỉ bình luận hoặc đóng RESOLVED
của hộ. Người xử lý ACTIVE có ticket.handle. Update + status nguyên tử,
phân công ghi audit.

## 10. Quản trị, kiểm toán, báo cáo

Username/role code/permission code unique. PUT role/permission thay tập nguyên
tử, [] gỡ hết, bảng nối khóa ghép đúng ERD. Không khóa/vô hiệu hóa/gỡ quyền của
quản trị viên ACTIVE cuối cùng. Không xuất password/hash trong profile/user/
audit/lỗi. Audit append-only server ghi, actorId null cho worker/gateway; che
password/token/secret/chữ ký/CCCD trước lưu before/after, không chỉ lúc đọc.

| Báo cáo | Cách tính |
| --- | --- |
| revenue | Payment POSTED theo paidAt; loại VOIDED/PENDING, tổng và theo channel |
| debts | Invoice đã phát hành chưa hủy trừ allocation POSTED, không cộng cam kết |
| contributions | Pledged nullable và thực thu POSTED theo campaign/hộ |
| collections | Bill, thực thu POSTED, đã chuyển từ đối soát CONFIRMED; chưa chuyển = thu - chuyển |

Revenue yêu cầu from/to dạng date, gồm ngày to theo giờ VN, from <= to, tối đa
366 ngày. Filter timestamp của payments/audit vẫn [from,to). Số liệu hiện tại,
không phải lịch sử as-of; VOIDED bị loại kể cả paidAt kỳ cũ. Báo cáo khác phân
trang, filter theo ID công bố. Export cùng filter, thêm report.export và quyền
báo cáo gốc, CSV UTF-8 BOM tối đa 10000 dòng, vượt trả 422 để thu hẹp lọc. Escape
CSV và chống formula injection ở ô văn bản bắt đầu =/+/-/@. PDF theo snapshot.

## 11. Nghiệm thu khi triển khai

- HTTP/request/response theo OpenAPI, hello/health giữ tương thích.
- Cư dân không đọc/ghi/đếm dữ liệu ngoài hộ qua ID con, filter, receipt/export.
- Thu/callback đồng thời không vượt nợ; cùng key/event chỉ một payment/receipt.
- Sai chữ ký/lệch tiền/callback muộn hoặc đảo thứ tự không tự ghi tiền.
- Duyệt cư trú/phát hành kỳ/nhập lô/thu/chốt/thay quyền nguyên tử.
- Giữ snapshot chứng từ; VOIDED không tính số dư, PDF có dấu hủy.
- Thay nghiệp vụ cập nhật cả OpenAPI và tài liệu. CORS nghiệp vụ cần cho phép
  Authorization, Content-Type, Idempotency-Key, If-Match và expose Location,
  X-Request-Id, Retry-After, Content-Disposition.

## Sơ đồ vật lý và vai trò hiện hành

ERD hiện hành: [BlueMoon_ERD.drawio](../BlueMoon_ERD.drawio), dùng ký pháp Chen để đối chiếu 39 bảng từ migration; hai bảng nối RBAC được biểu diễn bằng liên kết m-n.
UML hiện hành: [BlueMoon_UML.asta](../BlueMoon_UML.asta), gồm 39 lớp dữ liệu theo migration V1-V3, 5 actor, use case, phụ thuộc Spring Boot và sequence của các luồng đã triển khai. Xem [mục lục và quy ước sơ đồ](../diagrams.md).
Các ảnh ERD/UML preview được xuất từ file nguồn hiện hành. Tên bảng vật lý
`app_role`, `app_permission` tương ứng API `Role`, `Permission`; JSON snapshot
lưu TEXT. Tổng tiền và collections trong response được tính từ các bảng liên quan.
Danh mục vai trò cố định; POST /roles được giữ cho tương thích URL nhưng không
thể tạo thêm vai trò. AUTHORITY không được cấp quyền ngoài nhóm đọc cư trú.
