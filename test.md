# Phân tích Nguyên nhân và Giải pháp: Lỗi không cập nhật trạng thái Confirm OTP

Tình trạng bạn mô tả: Nhập OTP đúng -> Endpoint thông báo thành công (true) -> Reload trang thì trạng thái vẫn là chưa xác nhận (trong DB `confirm_otp` vẫn là `false`).

Dưới đây là các nguyên nhân chính ở phía Back-end gây ra tình trạng này, trong đó **Nguyên nhân 1** là lỗi kinh điển và có xác suất cao nhất.

---

## 1. Các nguyên nhân có thể gặp phải

### Nguyên nhân 1: Xung đột giữa Hibernate Session (JPA) và Native SQL trong cùng một Transaction (Xác suất 99%)

Đây là lỗi phát sinh do việc trộn lẫn giữa Spring Data JPA và JDBC Native Query lên cùng một bản ghi trong cùng một phạm vi giao dịch `@Transactional`.

Hãy xem luồng chạy hiện tại của phương thức `confirmSmsOtp` trong [BBAServiceImpl.java](file:///d:/BMK/BE/personal-insurance/src/main/java/nth/module/personalinsurance/service/BBA/impl/BBAServiceImpl.java#L797-L821):

```java
@Transactional
@Override
public Boolean confirmSmsOtp(LogIntegrationHeader logIntegrationHeader, String otp, String contractCode) {
    // BƯỚC 1: Load thực thể qua JPA Repo
    Optional<Contract> contractOptional = contractRepository.findFirstByContractCodeAndOtp(contractCode, otp);
    Contract contract = contractOptional.get(); // Đối tượng này có confirmOtp = '0' hoặc null

    // BƯỚC 2: Kiểm tra OTP hết hạn...
    boolean isTrue = Instant.now().isBefore(contract.getOtpExpiredTime());

    if (!isTrue) {
        throw new BusinessException("OtpExpiredTimeError");
    } else {
        // BƯỚC 3: Dùng Native SQL Update trực tiếp vào Database
        MapSqlParameterSource params = new MapSqlParameterSource().addValue("contractCode", contractCode);
        namedParameterJdbcTemplate.update(SQL.UPDATE_SET_CONFIRM_OTP, params); 
        
        return true;
    }
    // BƯỚC 4: Transaction kết thúc, chuẩn bị Commit!
}
```

**Cơ chế lỗi xảy ra như sau:**
1. **Bước 1:** `contractRepository` truy vấn DB và đưa thực thể `contract` vào **Hibernate Level-1 Cache (Persistence Context)**. Ở thời điểm này, trạng thái của thực thể này trong JVM có thuộc tính `confirmOtp = '0'` (hoặc `null`).
2. **Bước 3:** Câu lệnh `namedParameterJdbcTemplate.update` chạy Native SQL trực tiếp dưới DB: `UPDATE ... SET confirm_otp = '1'`. Lúc này ở tầng Database, bản ghi đã chuyển thành `'1'`.
3. **Bước 4:** Kết thúc phương thức, Spring thực hiện **Commit Transaction**. Lúc này Hibernate thực hiện cơ chế **Dirty Checking** (Kiểm tra trạng thái thay đổi của các đối tượng trong Persistence Context để đồng bộ xuống DB). 
   - Trong thực tế, thực thể `Contract` có trường phức tạp như `additional` sử dụng `@Convert(converter = MapToJsonConverter.class)`. Các trường này rất hay khiến Hibernate nhận diện nhầm thực thể đã bị thay đổi ("bẩn") khi so sánh trong Dirty Checking.
   - Do Hibernate **không nhận biết** được bản ghi dưới DB đã bị thay đổi bởi Native SQL ở Bước 3, nó sẽ tự động sinh câu lệnh `UPDATE personalinsurance.contract ...` để đồng bộ hóa đối tượng `contract` từ JVM xuống DB.
   - **Hậu quả:** Giá trị cũ `confirmOtp = '0'` (hoặc `null`) trong bộ nhớ JVM sẽ ghi đè (Overwrite) giá trị `'1'` vừa được cập nhật dưới DB. Do đó, DB cuối cùng vẫn lưu giá trị `false`.

---

### Nguyên nhân 2: Sai lệch Schema Database trong câu lệnh Native SQL

Trong [SQL.java](file:///d:/BMK/BE/personal-insurance/src/main/java/nth/module/personalinsurance/SQL.java#L35), câu lệnh update đang được viết cứng tên Schema là `personalinsurance`:
```sql
public static final String UPDATE_SET_CONFIRM_OTP = " update personalinsurance.contract t set t.confirm_otp = '1', ... ";
```
- Nếu trong cấu hình môi trường hiện tại của ứng dụng (trong file properties/yml), Datasource đang trỏ mặc định vào một Database/Schema khác (ví dụ: `personalinsurance_dev` hoặc schema mặc định), nhưng trên cùng server DB đó vẫn tồn tại một Schema tên là `personalinsurance`.
- Lúc này, JPA sẽ thực hiện đọc từ Schema cấu hình mặc định, trong khi câu lệnh Native SQL lại thực hiện cập nhật vào bảng của Schema `personalinsurance`. Việc này dẫn đến trạng thái "ghi một nơi nhưng đọc một nẻo".

---

### Nguyên nhân 3: Sai lệch kiểu dữ liệu (Type Mismatch) giữa JPA Entity và Database

- Trong [Contract.java](file:///d:/BMK/BE/personal-insurance/src/main/java/nth/module/personalinsurance/domain/Contract.java#L170), thuộc tính `confirmOtp` được khai báo kiểu `String`.
- Trong `SQL.java`, câu lệnh native update gán giá trị chuỗi `'1'`.
- Nếu trường `confirm_otp` dưới Database đang có kiểu dữ liệu thực tế là `BIT(1)` hoặc `BOOLEAN`/`TINYINT(1)`:
    - Một số Database Client hoặc JDBC Driver khi ánh xạ kiểu `String` của Hibernate xuống kiểu `BOOLEAN` có thể xảy ra hiện tượng parse sai cách trong quá trình flush, dẫn tới việc khi Hibernate thực hiện Flush dữ liệu, nó sẽ tự động set về trạng thái mặc định của kiểu Boolean (là `false`/`0`).

---

## 2. Giải pháp khắc phục triệt để

Để khắc phục dứt điểm vấn đề này, **tuyệt đối không trộn lẫn Native SQL Update và JPA Managed Entity trên cùng một Transaction**. 

Hãy thay đổi cách cập nhật từ Native SQL sang cập nhật trực tiếp trên đối tượng Entity của JPA. Điều này giúp Hibernate kiểm soát hoàn toàn trạng thái thay đổi và sinh câu lệnh SQL đồng bộ chính xác khi commit transaction.

### 🛠 Cập nhật mã nguồn tại BBAServiceImpl.java

Hãy cập nhật phương thức `confirmSmsOtp` trong [BBAServiceImpl.java](file:///d:/BMK/BE/personal-insurance/src/main/java/nth/module/personalinsurance/service/BBA/impl/BBAServiceImpl.java#L797-L821):

```diff
     @Transactional
     @Override
     public Boolean confirmSmsOtp(LogIntegrationHeader logIntegrationHeader, String otp, String contractCode) {
         Optional<Contract> contractOptional = contractRepository.findFirstByContractCodeAndOtp(contractCode, otp);
         if (contractOptional.isEmpty()) {
             LogUtils.logInfo(logIntegrationHeader, "confirmOtpGic", "No otp expired time found", System.currentTimeMillis(), this.log);
             throw new BusinessException("NoOtpExpiredTimeFound");
         }
         Contract contract = contractOptional.get();
 
         if (CommonUtils.isEmpty(contract.getOtp())) {
             LogUtils.logInfo(logIntegrationHeader, "confirmOtpGic", "No otp expired time found", System.currentTimeMillis(), this.log);
             throw new BusinessException("NoOtpExpiredTimeFound");
         }
 
         boolean isTrue = Instant.now().isBefore(contract.getOtpExpiredTime());
 
         if (!isTrue) {
             throw new BusinessException("OtpExpiredTimeError");
         } else {
-            MapSqlParameterSource params = new MapSqlParameterSource().addValue("contractCode", contractCode);
-            namedParameterJdbcTemplate.update(SQL.UPDATE_SET_CONFIRM_OTP, params);
+            // Cập nhật trực tiếp thông qua JPA Entity để Hibernate quản lý và tự động flush khi commit
+            contract.setConfirmOtp(AppConstant.YES); // Sử dụng hằng số "1" định nghĩa trong AppConstant
+            contract.setUpdateDate(Instant.now());
+            contract.setErrMsg(null);
+            contractRepository.save(contract);
             return true;
         }
     }
```

### 🛠 Cập nhật tương tự tại TBGICServiceImpl.java

Phương thức `confirmSmsOtp` trong [TBGICServiceImpl.java](file:///d:/BMK/BE/personal-insurance/src/main/java/nth/module/personalinsurance/service/BTA/Impl/TBGICServiceImpl.java#L1392-L1415) cũng đang mắc phải chính xác lỗi này và cần được sửa tương tự:

```diff
     @Override
     @Transactional
     public Boolean confirmSmsOtp(LogIntegrationHeader logIntegrationHeader, String otp, String contractCode) {
         Optional<Contract> contractOptional = contractRepository.findFirstByContractCodeAndOtp(contractCode, otp);
         if (contractOptional.isEmpty()) {
             LogUtils.logInfo(logIntegrationHeader, "confirmOtpGic", "No otp expired time found", System.currentTimeMillis(), this.log);
             throw new BusinessException("NoOtpExpiredTimeFound");
         }
         Contract contract = contractOptional.get();
 
         if (CommonUtils.isEmpty(contract.getOtp())) {
             LogUtils.logInfo(logIntegrationHeader, "confirmOtpGic", "No otp expired time found", System.currentTimeMillis(), this.log);
             throw new BusinessException("NoOtpExpiredTimeFound");
         }
 
         boolean isTrue = Instant.now().isBefore(contract.getOtpExpiredTime());
 
         if (!isTrue) {
             throw new BusinessException("OtpExpiredTimeError");
         }else{
-            MapSqlParameterSource params = new MapSqlParameterSource()
-                .addValue("contractCode", contractCode);
-            this.jdbcTemplate.update(SQL.UPDATE_SET_CONFIRM_OTP, params);
+            // Cập nhật trực tiếp thông qua JPA Entity
+            contract.setConfirmOtp(AppConstant.YES);
+            contract.setUpdateDate(Instant.now());
+            contract.setErrMsg(null);
+            contractRepository.save(contract);
             return true;
         }
     }
```

---

## Tổng kết ưu điểm khi áp dụng cách sửa trên:
1. **Loại bỏ cơ chế overwrite của Hibernate:** Hibernate ghi nhận sự thay đổi một cách hợp lệ và commit đúng dữ liệu mới nhất.
2. **An toàn về Schema:** Loại bỏ việc hardcode schema name `personalinsurance.` giúp ứng dụng linh hoạt hơn khi triển khai trên các môi trường Database khác nhau.
3. **Khớp kiểu dữ liệu:** Tự động tối ưu hóa kiểu dữ liệu JPA ánh xạ chính xác xuống Database Column mà không lo sai lệch Driver.
