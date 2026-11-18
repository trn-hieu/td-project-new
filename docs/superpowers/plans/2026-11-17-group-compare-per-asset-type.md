# Tách tài sản so sánh theo loại tài sản (2 giai đoạn)

## Context

`appraisal_group_compares` (entity `AppraisalGroupCompare`, ~85 cột) lưu TSSS của mọi loại tài sản (đất=1, căn hộ=2, ô tô=3, tàu=4); `compare_information` (kho, `CompareInformation`) là bản sao gần y hệt, được đồng bộ bằng `CompareInformation.copyFrom` chép từng cột. Mỗi loại tài sản mới phải: thêm cột vào **cả hai bảng** (lịch sử: V62–V100 căn hộ, V88/V94 ô tô, V20261015/V20261103 tàu), và sửa các switch rải rác: `AppraisalGroupCompare.applyForm`/`reset`, `CompareInformation.copyText/copyNumbers/copyApartmentAndCar/copyShip`, `CompareFactorSync.sourceOf`, `GroupCompareService`.

Ràng buộc: legacy (`td-project-legacy`) **vẫn chạy production**, sở hữu schema, đọc/ghi mọi cột của hai bảng và có native SQL báo cáo đọc thẳng cột. ⇒ Người dùng chọn **2 giai đoạn**:
- **GĐ1 (làm ngay, không đổi schema, không đổi JSON API):** gom cột theo loại thành `@Embeddable` dùng chung cho cả hai entity + một handler cho mỗi loại tài sản. Thêm loại mới = 1 embeddable + 1 handler + 1 migration phía legacy.
- **GĐ2 (sau khi tắt legacy):** chuyển mỗi embeddable sang bảng con 1-1, chủ yếu đổi mapping.

Mục tiêu GĐ1: không thay đổi giá trị ghi xuống DB và JSON trả FE; các test hiện có là lưới an toàn.

## GĐ1 — Thiết kế

### Embeddable (`model/entity/*Columns`, tránh trùng tên DTO `ShipCompareFields`)
| Class | Cột | Ghi chú |
|---|---|---|
| `ShipColumns` | 8 cột `ship_*` | `ship_for_sale_price_usd` ở lại `AppraisalGroupCompare` (kho không có) |
| `CarColumns` | 14 cột `car_*` | xác nhận kiểu giống nhau ở hai bảng |
| `ApartmentColumns` | unit_kind, 3 hướng, interior, floor_position(_other), apartment_structure, 3 `*_area_hundredths` (`Double`, mặc định 0.0), ownership_area_basis | `apartment_ownership_right_price` ở lại entity (cột giá, kho không có) |
| `LandColumns` | land_use_term, land_dimension, land_use_purpose, road_type, number_of_frontages, main_frontage_width, 4 diện tích, price_in_planning, unit_price_planning, convert_tax, unit_price_conversion_tax, construction_price, plant_price, structure, 4 cột JSON kèm `@Convert` | |

Giữ trên entity: cột chung (source, contact, các giá kể cả `land_use_right_price`, address, cột tab Bảng điều chỉnh, 2 id liên kết kho), nhóm "listing" đất+căn hộ (information, legal, transaction_*, road_width), tọa độ (kiểu khác nhau giữa hai bảng: BigDecimal vs Double).

- **Embedded null:** Hibernate gán null khi mọi cột null. Dùng getter tự khởi tạo trên entity (`getShip()` trả `new ShipColumns()` nếu null) + field khởi tạo sẵn; không dùng `create_empty_composites` (deprecated ở Hibernate 6/7). Test task 0 xác nhận load rồi flush **không** sinh UPDATE thừa.
- Mỗi embeddable có `copy()` (không chia sẻ instance giữa hai entity), `clear()` (đúng giá trị `reset` hiện tại: chuỗi null, giá 0, diện tích căn hộ 0.0, basis 1), và logic ghi riêng chuyển từ entity sang: `ShipColumns.apply(ShipCompareFields)/syncFrom(ship, quality)`, `CarColumns.apply(in)/syncFrom(vehicle)`, `ApartmentColumns.apply(in)`, `LandColumns.apply(in, inside, outside, plants)` — giữ nguyên trim/scale2/"chỉ ghi khi khác null".
- `AppraisalGroupCompare.reset` = cột chung + 4 `clear()`. `CompareInformation.copyFrom` = cột chung + `this.ship = row.getShip().copy()`… thay cho 4 hàm `copy*`.
- Caller đổi `row.getShipBuildYear()` → `row.getShip().getBuildYear()` (compiler dẫn đường): `*CompareTable`, `ShipCompareInfoBlock`, `GroupSheetWriter`, `GroupCompareRow.from`, `CompareInformationDetail.from`, `ShipCompareInformationItem`, `CompareFactorSync`, test. JPQL đổi path (vd. `c.carName` → `c.car.name` trong `CompareInformationRepository`).

### Handler (`service/compare/`)
```java
interface CompareAssetTypeHandler {
    int assetType();
    void applyForm(AppraisalGroupCompare row, GroupCompareRowInput in);   // phần riêng của loại
    Function<AppraisalGroupCompare, String> factorSource(String label);  // null = không đồng bộ
    default void validate(GroupCompareRowInput in) {}                    // tàu: chất lượng 0..100
}
```
- `CompareAssetTypeHandlers` (`@Component`, nhận `List<handler>` → `Map<Integer, handler>`, `forType(int)`; loại không biết → Land như nhánh `default` hiện tại).
- 4 handler: Land (gọi `AreaDetailsMerge`, gỡ nhánh LAND trong `GroupCompareService.writeRow`), Apartment, Car, Ship (giữ quy tắc USD không gửi thì giữ, `market_price=0`).
- Entity giữ hàm nhỏ: `applyCommon(in)` (source, contact, 3 giá, market_price), `applyListing(in)`, các hàm gán giá đuôi theo loại.
- `GroupCompareService.writeRow` → `applyCommon` + `handlers.forType(t).applyForm`; `requireValid` → `handler.validate`.
- `CompareFactorSync` thành `@Component`: **giữ 5 nhãn không phụ thuộc loại** (ROAD_WIDTH, ROAD_TYPE, SIZE, FRONTAGES, LAND_USE_PURPOSE — hiện trả giá trị cho mọi loại, kể cả tàu/ô tô) ở map chung; nhãn còn lại hỏi `handler.factorSource(label)`.
- Không đưa vào interface (YAGNI): reset, copy kho, sync từ tab mô tả (mỗi service mô tả gọi thẳng embeddable của mình), block walking-route LAND||APARTMENT.

### DTO
`GroupCompareRow`, `GroupCompareRowInput`, `CompareInformationDetail`, `ShipCompareInformationItem` giữ nguyên hình dạng và tên field (FE không đổi); chỉ đổi cách đọc getter. Test snapshot JSON bảo vệ hợp đồng.

## GĐ1 — Thứ tự task (mỗi task 1 commit, compile + test xanh)
0. **Lưới an toàn:** test snapshot JSON `GroupCompareRow`/`CompareInformationDetail` đủ field; test parity `copyFrom` đủ cột (nếu `GroupCompareEntitiesRepositoryTest` chưa có); repository test load dòng toàn null + flush không UPDATE.
1. `ShipColumns` (nhỏ, mới nhất) — test: `AppraisalGroupCompareShipFormTest`, `ShipColumnsRepositoryTest`, `ShipDescriptionServiceTest`, thêm `ShipColumnsTest`.
2. `CarColumns` — `VehicleDescriptionServiceTest`, `AppraisalVehicleDescriptionRepositoryTest`.
3. `ApartmentColumns` — `ApartmentDescriptionServiceTest`.
4. `LandColumns` — sửa JPQL/search path; `LandDescriptionServiceTest`, `CompareInformationSearchRepositoryTest`, thêm `LandColumnsTest`.
5. Gọn `CompareInformation.copyFrom`, xóa 4 hàm `copy*` — `CompareInformationSyncTest` giữ nguyên assertion.
6. Handler + registry + 4 handler; chuyển switch `applyForm` ra khỏi entity; chuyển `AppraisalGroupCompareFormTest`/`ShipFormTest` thành `*HandlerTest`; test registry: mọi hằng `AppraisalAssetType` có handler.
7. `CompareFactorSync` dùng registry, giữ 5 nhãn chung; thêm case tàu hỏi ROAD_WIDTH vào `CompareFactorSyncTest`.
8. Docs: thêm dòng vào bảng `backend/CLAUDE.md` ("Thêm loại tài sản vào TSSS → `*Columns` + handler + migration legacy"); xóa bảng Javadoc cũ ở `applyForm`, chuyển mô tả vào từng handler.

Thực thi: subagent model **sonnet**, agent chính review từng task; mỗi task xong commit và đánh dấu ✅ ở đây.

## GĐ2 — Phác thảo (điều kiện: legacy đã tắt)
- Mapping: spike 1 loại trước với `@SecondaryTable(name="agc_ship_details", pkJoinColumns=@PrimaryKeyJoinColumn(name="appraisal_group_compare_id"))` + `@AttributeOverride(column=@Column(table=...))` + `@org.hibernate.annotations.Table(optional=true)` (LEFT JOIN, không chèn dòng con rỗng). Lưu ý: mỗi load join mọi bảng phụ. Phương án dự phòng: `@OneToOne(LAZY) @MapsId` child entity.
- Flyway (tách file): tạo `agc_<type>_details` + `ci_<type>_details` → `INSERT … SELECT` chép dữ liệu → đổi mapping → sau ổn định mới `DROP COLUMN` (bước duy nhất không đảo ngược).
- Native SQL cần LEFT JOIN bảng con: `TsssReportSql`, `VcbReportSql`, `CoordinateSearchRepository` (tọa độ ở bảng chung → có thể không đổi), `CompareInformationRepository.findIdsWithinRadius` (không đổi). Mỗi query đổi có repository test trên MySQL.

## Rủi ro
- Embedded null ↔ rỗng gây UPDATE thừa → getter tự khởi tạo + test task 0.
- Cột NOT NULL double (`*_area_hundredths`) nhận null → mặc định 0.0.
- Mất 5 nhãn yếu tố chung khi tách theo loại → map chung + test.
- JPQL path sai chỉ lộ lúc chạy (`ddl-auto=none`) → repository test mỗi task.
- Trôi JSON → snapshot test.

## Verification
- Sau mỗi task: `cd backend && ./mvnw test` (repository test cần MySQL schema `thanhdo`, xem `.env.example`).
- Chạy nhanh: `./mvnw -q test -Dtest=GroupCompareServiceTest,CompareInformationSyncTest,CompareFactorSyncTest,AppraisalGroupCompare*Test,GroupCompareSaveRepositoryTest,GroupCompareEntitiesRepositoryTest,ShipColumnsRepositoryTest`.
- Cuối GĐ1: chạy FE (`frontend` test `ShipCompareTab.spec.ts`, `LandCompareTab.spec.ts`…) để chắc JSON không đổi; thử tay màn "Thông tin so sánh" cho 4 loại: lưu, đẩy kho, chọn kho, đổi loại tài sản, xuất Word/Excel.
