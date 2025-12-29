# Hướng dẫn triển khai Service Mesh cho spring-petclinic-microservices

## 1. Chuẩn bị môi trường
- Đảm bảo có K8S cluster (Minikube, Kind, GKE, EKS...)
- Cài đặt các công cụ: `kubectl`, `helm`, `istioctl`
- Cài đặt Istio lên cluster:
  ```sh
  istioctl install --set profile=demo -y
  kubectl label namespace default istio-injection=enabled
  ```
- Cài đặt Kiali để quan sát topology:
  ```sh
  kubectl apply -f https://projectcontour.io/quickstart/kiali.yaml
  # hoặc dùng manifest của Istio: istioctl dashboard kiali
  ```

## 2. Bật mTLS cho các service
- Tạo file `peer-authentication.yaml`:
  ```yaml
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: default
  spec:
    mtls:
      mode: STRICT
  ```
- Áp dụng:
  ```sh
  kubectl apply -f peer-authentication.yaml
  ```

## 3. Cấu hình Authorization Policy
- Ví dụ chỉ cho phép service A gọi service B:
  ```yaml
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: allow-a-to-b
    namespace: default
  spec:
    selector:
      matchLabels:
        app: service-b
    rules:
    - from:
      - source:
          principals: ["cluster.local/ns/default/sa/service-a"]
  ```
- Áp dụng:
  ```sh
  kubectl apply -f authorization-policy.yaml
  ```

## 4. Cấu hình Retry Policy
- Ví dụ cho service gateway gọi backend có retry:
  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    name: backend-vs
    namespace: default
  spec:
    hosts:
    - backend
    http:
    - route:
      - destination:
          host: backend
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: gateway-error,connect-failure,refused-stream,5xx
  ```
- Áp dụng:
  ```sh
  kubectl apply -f virtualservice-retry.yaml
  ```

## 5. Kiểm tra & Test
- Dùng lệnh sau để kiểm tra kết nối:
  ```sh
  kubectl exec -it <pod> -- curl -v http://<service>:<port>/
  ```
- Kiểm tra:
  - Kết nối plaintext bị chặn khi mTLS bật
  - Chỉ các luồng hợp lệ mới truy cập được (HTTP 403 nếu bị chặn)
  - Khi backend trả 500, kiểm tra log để xác nhận retry

## 6. Quan sát topology với Kiali
- Truy cập dashboard Kiali (`istioctl dashboard kiali` hoặc qua NodePort/Ingress)
- Chụp screenshot topology các service
- Ghi chú lại flow, giải thích topology

## 7. Tổng hợp deliverables
- Lưu các file YAML cấu hình (mTLS, AuthorizationPolicy, VirtualService)
- Screenshot topology từ Kiali
- Test plan, log kết quả test (curl, retry)
- README hướng dẫn từng bước triển khai (file này)
