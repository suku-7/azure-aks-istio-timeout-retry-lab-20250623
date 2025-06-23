# Model
## azure-aks-istio-timeout-retry-lab-20250623
https://labs.msaez.io/#/courses/cna-full/2c7ffd60-3a9c-11f0-833f-b38345d437ae/deploy-my-app-2024

## Azure aks환경 Istio 서비스 메시의 고급 트래픽 관리 기능 중 타임아웃과 재시도 실습
- Istio 환경에서 Order 서비스에 인위적인 지연을 유발하고, Istio VirtualService를 통해 타임아웃 정책을 적용하여 이를 제어합니다.
- 워크로드 생성기(Siege Pod)를 사용하여 서비스에 부하를 발생시키고, 타임아웃이 발생하는 시나리오를 확인합니다.
- 실패한 요청에 대해 재시도 정책을 추가하여 서비스의 견고성을 높이는 방법을 실습합니다.

## 사전 준비
Azure 계정 및 구독, Gitpod 워크스페이스, Spring Boot 애플리케이션 코드

https://github.com/suku-7/azure-aks-istio-servicemesh-monitoring-lab-20250620
- 위의 실습에서 이어지는 작업입니다.

![스크린샷 2025-06-23 120738](https://github.com/user-attachments/assets/955a195c-3db3-44a7-9cec-425711f84990)
![스크린샷 2025-06-23 120745](https://github.com/user-attachments/assets/07d44ad2-16fb-4206-95b7-f0fbd5856b48)
![스크린샷 2025-06-23 121332](https://github.com/user-attachments/assets/3464533d-6835-4c5b-8b08-e46618ccb60d)
![스크린샷 2025-06-23 121734](https://github.com/user-attachments/assets/d62d541c-b5dd-4ca2-81d3-6239f449f78d)
![스크린샷 2025-06-23 121852](https://github.com/user-attachments/assets/259654f3-6c75-4bed-bee6-2022d31e893e)
![스크린샷 2025-06-23 122011](https://github.com/user-attachments/assets/9594e292-403e-4dd6-9021-9a2391422035)
![스크린샷 2025-06-23 122833](https://github.com/user-attachments/assets/d3321c2b-98f3-4433-8e5e-61b2cecec038)
![스크린샷 2025-06-23 123121](https://github.com/user-attachments/assets/e714bb3f-4cbd-4478-af4a-cb16c1638dce)

---

## 실습 단계별 상세 설명


1. 이전 환경 정리 및 Order 서비스 배포
--- 터미널 1 (메인 작업) ---
```
# 이전 랩에서 사용된 애플리케이션 및 Istio 리소스 삭제 (tutorial 네임스페이스 기준)
kubectl delete deployment,service --all -n tutorial
kubectl get dr -n tutorial # DestinationRule이 있다면 확인
kubectl get vs -n tutorial # VirtualService가 있다면 확인
kubectl delete dr --all -n tutorial # 모든 DestinationRule 삭제
kubectl delete vs --all -n tutorial # 모든 VirtualService 삭제
kubectl get all -n tutorial # tutorial 네임스페이스 리소스 삭제 확인
```
```
# 주문 서비스(Order Service) 배포
# 이 Order 서비스는 특정 조건에서 응답 지연을 유발하도록 설계된 이미지입니다.
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
  namespace: tutorial # 또는 현재 작업 중인 네임스페이스 (예: shop)
  labels:
    app: order
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: jinyoung/order:timeout # 타임아웃 테스트를 위한 특별한 이미지
          ports:
            - containerPort: 8080
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
EOF
```
```
# Order 서비스 Pod가 정상적으로 배포되었는지 확인 (2/2 Ready 상태)
kubectl get all -n tutorial

# Order 서비스를 ClusterIP 타입으로 노출
kubectl expose deploy order --port=8080 -n tutorial
```
2. 타임아웃 정책 적용 및 Siege Pod 배포
--- 터미널 1 (메인 작업) ---
```
# Order 서비스에 3초 타임아웃 정책을 포함하는 VirtualService 배포
# 3초를 초과하는 응답은 Istio에 의해 강제로 종료됩니다.
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-order-network-rule
  namespace: tutorial # 또는 현재 작업 중인 네임스페이스
spec:
  hosts:
  - order
  http:
  - route:
    - destination:
        host: order
      timeout: 3s # 3초 타임아웃 설정
EOF
```
```
# 워크로드 생성기(Siege Pod) 배포
# 이 Pod를 통해 Order 서비스로 트래픽을 발생시킬 것입니다.
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: siege
  namespace: tutorial # 또는 현재 작업 중인 네임스페이스
spec:
  containers:
  - name: siege
    image: apexacme/siege-nginx
EOF
```
```
# Siege Pod가 정상적으로 배포되었는지 확인 (0/2 PodInitializing -> 2/2 Running 상태로 바뀔 때까지 대기)
kubectl get all -n tutorial
```
3. Siege를 통한 워크로드 생성 및 타임아웃 확인
--- 터미널 1 (메인 작업) ---
```
# Siege Pod 내부 셸로 접속
# (접속 후 셸 프롬프트가 바뀝니다. 이후 명령은 이 셸에서 실행합니다.)
kubectl exec -it siege -c siege -n tutorial -- /bin/bash

--- Siege Pod 내부 셸 (접속 후 이어서 실행) ---

# 단일 요청으로 타임아웃 확인 (예시: 4초 동안 대기하다 타임아웃 발생)
siege -c1 -t4S -v --content-type "application/json" 'http://order:8080/orders POST {"productId": "1001", "qty":5}'

# 다수의 동시 요청으로 타임아웃 트랜잭션 확인 (예: 30명의 사용자, 20초 동안 테스트)
# 이 명령을 실행하면 "HTTP: unable to determine chunk size" 경고와 함께 실패한 트랜잭션(Failed transactions)이 발생할 것입니다.
siege -c30 -t20S -v --content-type "application/json" 'http://order:8080/orders POST {"productId": "1001", "qty":5}'

# (선택 사항) 더 짧은 시간으로 테스트
# siege -c30 -t5S -v --content-type "application/json" 'http://order:8080/orders POST {"productId": "1001", "qty":5}'
```
4. Order 서비스에 Retry Rule 추가
--- 터미널 1 (메인 작업) ---
```
(Siege Pod 내부 셸에서 exit 명령으로 빠져나온 후, 다시 메인 터미널에서 실행합니다.)

# Order 서비스의 VirtualService에 재시도(Retry) 정책 추가
# 3초 타임아웃 후, 최대 3번 재시도하며, 각 재시도마다 2초의 타임아웃을 적용합니다.
# 5xx 에러, 4xx 재시도 가능 에러, 게이트웨이 에러, 연결 실패, 스트림 거부 시 재시도합니다.
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-order-network-rule
  namespace: tutorial # 또는 현재 작업 중인 네임스페이스
spec:
  hosts:
  - order
  http:
  - route:
    - destination:
        host: order
      timeout: 3s # 총 타임아웃
      retries:
        attempts: 3 # 최대 3회 재시도
        perTryTimeout: 2s # 각 재시도당 타임아웃
        retryOn: 5xx,retriable-4xx,gateway-error,connect-failure,refused-stream # 재시도 조건
EOF
```
5. Retry 동작 확인 시나리오
--- 터미널 1 (메인 작업) ---
```
# Siege Pod 내부 셸로 다시 접속
# (접속 후 셸 프롬프트가 바뀝니다. 이후 명령은 이 셸에서 실행합니다.)
kubectl exec -it siege -c siege -n tutorial -- /bin/bash

--- Siege Pod 내부 셸 (접속 후 이어서 실행) ---

# 새로운 주문 생성 요청 (재시도 정책 적용 후 동작 확인)
# 응답에서 "href" 필드에 생성된 Order ID가 포함됩니다.
http POST http://order:8080/orders qty=5

# 생성된 주문 Key를 가지는 주문 정보 삭제
# [생성된 Order ID] 부분에 위에서 확인한 실제 Order ID를 입력합니다.
# 예: http DELETE http://order:8080/orders/673
http DELETE http://order:8080/orders/[생성된 Order ID]
```
