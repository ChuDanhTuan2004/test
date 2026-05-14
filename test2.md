2. Phát hiện bổ sung: Một lỗi chí mạng khác ở Back-End liên quan tới việc "Reset OTP"
Trong lúc kiểm tra luồng lưu trữ, tôi phát hiện phương thức lưu tổng hợp hợp đồng ở Back-End đang cưỡng bức ghi đè lại trạng thái OTP về false mỗi khi có cập nhật!

Hãy xem tệp 

ContractServiceImpl.java
 tại dòng 3817:

java
@Override
    public ContractDTO saveAllContractDetail(LogIntegrationHeader logIntegrationHeader, ContractDTO contractDTO)
        throws IllegalAccessException, InstantiationException {
        
        // ... các đoạn xử lý cập nhật thông tin xe, người mua, người thụ hưởng ...
        ObjectUtil.mergeObjects(contractDTO, oldContract);
        // ❌ LỖI TẠI ĐÂY: Ép buộc set confirmOtp thành "0" bất kể trạng thái cũ!
        contractDTO.setConfirmOtp(AppConstant.NO); 
        contractDTO = this.save(contractDTO);
        return contractDTO;
    }
Hệ quả của đoạn code BE này: Cho dù trước đó người dùng đã nhập OTP thành công và lưu vào DB là 1, thì sau đó:

Người dùng thực hiện sửa đổi thông tin hợp đồng và nhấn nút Cập nhật (Edit) ở phía FE.
Hoặc đối với bảo hiểm Xe (Car Insurance), người dùng nhấn Tính phí. Lúc này API saveAllContractDetail được gọi và Back-End sẽ tự động đưa trạng thái OTP về lại "0" (Chưa xác nhận).
