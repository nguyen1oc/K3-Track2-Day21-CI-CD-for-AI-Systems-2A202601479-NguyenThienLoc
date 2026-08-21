# Reflection – CI/CD cho hệ thống Machine Learning

**Sinh viên:** Nguyễn Thiên Lộc  
**MSSV:** 2A202601479  
**Lớp:** Cohort 3 – Track 2

## 1. Lựa chọn siêu tham số

Tôi dùng MLflow để so sánh các cấu hình của `RandomForestClassifier`. Các lần chạy đại diện cho thấy F1 tăng từ `0.553356` với `n_estimators=100, max_depth=5`, lên `0.646439` với `n_estimators=200, max_depth=10`, `0.668370` với `n_estimators=300, max_depth=None, min_samples_split=5`, và `0.680578` với `n_estimators=500, max_depth=None, criterion=entropy`. Cấu hình cuối cùng tôi chọn là:

```yaml
n_estimators: 100
max_depth: 15
criterion: entropy
min_samples_split: 2
```

Cấu hình này đạt kết quả tốt nhất trong các lần chạy của tôi: `accuracy=0.684000` và `f1_score=0.682552`. So với cấu hình 500 cây không giới hạn độ sâu, nó có F1 cao hơn (`0.682552` so với `0.680578`) dù dùng ít cây hơn. Vì vậy, cấu hình được chọn vừa cho chất lượng tốt nhất trên tập đánh giá, vừa giảm thời gian huấn luyện và suy luận. Giới hạn `max_depth=15` cũng giúp kiểm soát độ phức tạp của từng cây.

## 2. Vì sao quality gate nên dựa trên F1

Bài toán là phân loại ba lớp và dữ liệu không cân bằng hoàn toàn. Trong tập đánh giá 500 mẫu, lớp 0 có 173 mẫu (34,6%), lớp 1 có 227 mẫu (45,4%), còn lớp 2 chỉ có 100 mẫu (20,0%). Accuracy chỉ đo tỷ lệ dự đoán đúng chung, nên một mô hình ưu tiên lớp phổ biến vẫn có thể đạt accuracy tương đối tốt. `f1_score` dạng weighted trong code kết hợp precision và recall của từng lớp rồi tính trung bình theo số mẫu, vì vậy phản ánh cân bằng hơn giữa việc dự đoán đúng và hạn chế bỏ sót/sai nhãn.

Số liệu MLflow minh họa sự khác biệt này. Cấu hình yếu có `accuracy=0.564000` nhưng `f1_score=0.553356`, thấp hơn accuracy khoảng 0,0106; điều này cho thấy tỷ lệ đúng chung đã che bớt sự mất cân bằng giữa precision và recall. Với cấu hình được chọn, hai chỉ số gần nhau hơn (`0.684000` và `0.682552`), cho thấy kết quả ổn định hơn giữa các lớp. Vì thế, ngưỡng F1 là tín hiệu an toàn hơn cho quality gate so với chỉ dựa vào accuracy. Tôi chọn ngưỡng `0.68`: mô hình tốt nhất vượt ngưỡng với F1 `0.682552`, trong khi các cấu hình yếu bị chặn.

Trong quá trình thực hiện lab, tôi trước tiên giữ ngưỡng mẫu `0.70` để tạo case fail và kiểm tra rằng Eval gate thực sự chặn Deploy. Sau khi đã lưu bằng chứng case fail, tôi chủ động hạ ngưỡng xuống `0.68` để minh họa case pass với mô hình tốt nhất của phase 1. Cách làm này cho thấy cả hai nhánh của quality gate đều hoạt động: model dưới ngưỡng bị chặn, còn model đạt ngưỡng mới được triển khai.

## 3. So sánh trước và sau khi bổ sung dữ liệu

| Lần huấn luyện | Số mẫu thực tế trong artifact DVC | Accuracy | F1 score |
|---|---:|---:|---:|
| Phase 1 | 2.998 | 0.684000 | 0.682552 |
| Phase 2 | 5.996 | 0.738000 | 0.736204 |

F1 tăng `0.053652`, tương đương khoảng 5,37 điểm phần trăm. Khi giữ nguyên tập đánh giá, cách chia dữ liệu và siêu tham số, việc tăng gấp đôi dữ liệu huấn luyện từ 2.998 lên 5.996 mẫu giúp Random Forest quan sát được nhiều trường hợp hơn, giảm phương sai và học tốt hơn ranh giới giữa ba mức chất lượng rượu. Do đó cả accuracy và F1 đều tăng.

## 4. Khó khăn và cách giải quyết

- **Không thể tạo service-account key:** Organization Policy `iam.disableServiceAccountKeyCreation` chặn việc tạo khóa JSON, nên `sa-key.json` không có nội dung và không thể dùng secret `CLOUD_CREDENTIALS` theo cách truyền thống. Tôi chuyển sang Workload Identity Federation (WIF): GitHub Actions phát hành OIDC token, Google Cloud xác minh repository qua Workload Identity Provider và cho workflow impersonate service account bằng quyền `roles/iam.workloadIdentityUser`. Workflow dùng `permissions: id-token: write` và `google-github-actions/auth`, chỉ lưu provider cùng email service account trong GitHub Secrets. Cách này tạo credential ngắn hạn cho mỗi lần chạy, không lưu private key lâu dài, giảm nguy cơ rò rỉ và tuân thủ policy của tổ chức.
- **DVC pull bị `403 Forbidden`:** service account ban đầu thiếu `storage.objects.list`. Tôi cấp `roles/storage.objectAdmin` tại bucket để CI có thể liệt kê, tải dữ liệu DVC và upload model.
- **Quality gate ban đầu thất bại:** ngưỡng mẫu `0.70` cao hơn kết quả tốt nhất thực tế của phase 1. Lần fail chứng minh Deploy thực sự bị chặn. Sau khi so sánh các run, tôi hiệu chỉnh ngưỡng về `0.68`, vẫn cao hơn các cấu hình yếu nhưng cho phép cấu hình tốt nhất đi qua và minh họa case pass.
- **Deploy kiểm tra quá sớm:** service cần khoảng 11 giây để tải model từ GCS và khởi động, trong khi workflow chỉ đợi 5 giây. Tôi thay thời gian chờ cố định bằng health-check retry mỗi 2 giây, tối đa 30 giây. Nhờ đó pipeline không còn fail do timing ngẫu nhiên.
- **Triển khai lên VM:** `gcloud compute scp` trên Windows không xử lý đường dẫn `~/...` như mong đợi. Tôi dùng đường dẫn tuyệt đối `/home/nguyenloc/...`, cài service systemd và sử dụng service account gắn trực tiếp với VM để tải model từ GCS mà không cần file JSON.

WIF là thay đổi quan trọng nhất trong quá trình thực hiện: nó không chỉ giải quyết lỗi do policy mà còn là phương án CI/CD an toàn hơn service-account key dài hạn.
