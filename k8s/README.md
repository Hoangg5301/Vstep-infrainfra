# Deploy VSTEP lên k3s bằng Argo CD

Các manifest dùng Kustomize, namespace `vstep-production` và Traefik của k3s. Web chạy cổng 3000; AI worker dùng image tools và chạy liên tục. MongoDB dùng dịch vụ có sẵn bên ngoài các manifest này; file upload lưu trong MongoDB GridFS.

## Cấu hình trước khi deploy

- Trong `kustomization.yaml`, đổi `nguyenhoang01` thành đúng GitHub Actions variable `DOCKERHUB_USERNAME`. CI push image `ci_cd_vstep:web-latest`, `web-<github.sha>`, `tools-latest`, `tools-<github.sha>` khi push lên `master`.
- Thay domain mẫu `vstep.lifestack.io.vn` trong `config-map.yaml` và `ingress.yaml`; trỏ DNS về ingress của cluster. `APP_URL` và `AUTH_URL` phải là URL người dùng truy cập. Nếu dùng Google OAuth, đăng ký callback `https://<domain>/api/auth/callback/google`.
- Cần Traefik với IngressClass `traefik`. Tạo TLS Secret `vstep-tls` trong namespace `vstep-production` bằng chứng chỉ của domain, hoặc cấu hình cert-manager cấp Secret đó theo issuer của cluster.
- Nếu repo private, cấu hình repository credential trong Argo CD. Nếu image private, tạo Docker registry Secret trong namespace `vstep-production` và thêm `imagePullSecrets` vào cả hai Deployment.

Tạo namespace và Secret trước khi sync. Chuẩn bị file env riêng ngoài Git, ví dụ `/secure/vstep.env`, chứa:

```dotenv
MONGODB_URI=mongodb+srv://...
AUTH_SECRET=<random-secret>
# Các cấu hình tùy chọn; bỏ dòng nếu chưa sử dụng:
AUTH_GOOGLE_ID=...
AUTH_GOOGLE_SECRET=...
SMTP_HOST=...
SMTP_USER=...
SMTP_PASSWORD=...
EMAIL_FROM=...
OPENAI_API_KEY=...
GEMINI_API_KEY=...
```

```sh
kubectl apply -f k8s/namespace.yaml
kubectl -n vstep-production create secret generic vstep-app-secret \
  --from-env-file=/secure/vstep.env --dry-run=client -o yaml | kubectl apply -f -
kubectl -n vstep-production create secret tls vstep-tls --cert=/secure/fullchain.pem --key=/secure/privkey.pem
kubectl kustomize k8s
kubectl apply -f k8s/argocd/application.yaml
kubectl -n vstep-production rollout status deployment/vstep-deployment
kubectl -n vstep-production rollout status deployment/vstep-ai-worker
```

Commit/push thư mục `k8s` lên `master` trước khi tạo Application. Application tự sync cấu hình từ Git; không đưa Application vào danh sách resources của Kustomize. Không commit file chứa Secret. Healthcheck `/api/health` kiểm tra tiến trình web, không kiểm tra kết nối MongoDB.

## Cập nhật image

CI hiện chỉ build/push image, chưa cập nhật Git manifest. Argo CD không tự rollout khi nội dung tag `latest` thay đổi; `imagePullPolicy: Always` chỉ có tác dụng khi pod được khởi tạo.

Để triển khai mỗi bản bằng GitOps, sau khi job `images` thành công, cập nhật cả hai `newTag` trong `kustomization.yaml` thành `web-<github.sha>` và `tools-<github.sha>` của cùng commit đã build, rồi commit/push. Argo CD sẽ tự sync và rollout. Có thể tự động hóa bước cập nhật tag bằng pipeline riêng hoặc Argo CD Image Updater sau này.

Thay đổi ConfigMap/Secret không tự restart pod dùng biến môi trường. Sau khi sync cấu hình, restart cả hai Deployment để nạp lại:

```sh
kubectl -n vstep-production rollout restart deployment/vstep-deployment deployment/vstep-ai-worker
```

Image tools còn có thể dùng cho tác vụ seed/admin thủ công; không tự chạy seed mỗi lần deploy để tránh thay đổi dữ liệu ngoài ý muốn.
