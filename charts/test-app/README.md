# Test App - OpenTelemetry Auto-Instrumentation 예시

이 chart는 OpenTelemetry Auto-Instrumentation과 Sidecar Collector를 사용하는 완전한 예시입니다.

## 구성 요소

### 1. Namespace
- Label: `tenant-id=1`
- DaemonSet Collector가 이 레이블을 읽어 `vm_account_id=1`로 메트릭 전송

### 2. Instrumentation CR
- Python/Java/NodeJS/Go Auto-Instrumentation 지원
- OTLP Exporter 자동 설정
- Tenant ID 자동 주입

### 3. Deployment (Python Flask App)
- **Annotations**:
  - `sidecar.opentelemetry.io/inject: "monitoring/otel-sidecar"` - Sidecar 주입
  - `instrumentation.opentelemetry.io/inject-python: "true"` - Python Auto-Instrumentation
- **Endpoints**:
  - `GET /` - 기본 정보
  - `GET /api/users` - 사용자 목록
  - `GET /api/orders` - 주문 목록 (내부적으로 /api/users 호출 → 분산 트레이싱)
  - `GET /api/error` - 에러 시뮬레이션
  - `GET /health` - Health check

### 4. Service
- ClusterIP 타입
- Port 8080

### 5. Traffic Generator (CronJob)
- 매 1분마다 자동으로 트래픽 생성
- 모든 엔드포인트 호출
- 텔레메트리 데이터 생성

## 설치

```bash
# Monitoring chart가 먼저 설치되어 있어야 함
helm upgrade --install monitoring ./charts/monitoring -n monitoring --create-namespace

# Test app 설치
helm upgrade --install test-app ./charts/test-app -n test-app --create-namespace
```

## 확인

### 1. Pod 상태 확인
```bash
kubectl get pods -n test-app

# 출력 예시:
# NAME                        READY   STATUS    RESTARTS   AGE
# test-app-xxxxxxxxx-xxxxx    3/3     Running   0          1m
#                             ^^^
#                             app + otc-container + inst-* (auto-instrumentation)
```

### 2. Sidecar 주입 확인
```bash
kubectl describe pod -n test-app -l app=test-app | grep -A5 "Containers:"

# 다음이 보여야 함:
# - app (Flask 애플리케이션)
# - otc-container (OpenTelemetry Collector Sidecar)
# - opentelemetry-auto-instrumentation-python (Python Auto-Instrumentation)
```

### 3. Auto-Instrumentation 로그 확인
```bash
# Python instrumentation init container
kubectl logs -n test-app -l app=test-app -c opentelemetry-auto-instrumentation-python

# Collector sidecar
kubectl logs -n test-app -l app=test-app -c otc-container

# App logs (OTLP로 전송됨)
kubectl logs -n test-app -l app=test-app -c app
```

### 4. 수동으로 트래픽 생성
```bash
# Port-forward
kubectl port-forward -n test-app svc/test-app 8080:8080

# 요청 보내기
curl http://localhost:8080/
curl http://localhost:8080/api/users
curl http://localhost:8080/api/orders
curl http://localhost:8080/api/error
```

## 텔레메트리 확인

### VictoriaMetrics (메트릭)
```bash
# Port-forward to vmselect
kubectl port-forward -n monitoring svc/vmselect-victoria-metrics 8481:8481

# 브라우저에서 http://localhost:8481/select/1/vmui 접속
# AccountID=1 (tenant-id=1)에서 메트릭 확인

# 예시 쿼리:
# - http_server_duration_milliseconds_count
# - http_server_requests_total
# - process_cpu_seconds_total
```

### VictoriaLogs (로그)
```bash
# Port-forward to vlselect
kubectl port-forward -n monitoring svc/vlselect-victoria-logs 9428:9428

# curl로 로그 쿼리
curl -H "AccountID: 1" "http://localhost:9428/select/logsql/query" \
  -d 'query=_stream:{service_name="test-app"}'
```

### VictoriaTraces (트레이스)
```bash
# Port-forward to vtsingle
kubectl port-forward -n monitoring svc/vtsingle-victoria-traces 10428:10428

# Jaeger UI (내장)
# http://localhost:10428/jaeger/search
```

## 멀티테넌시 테스트

### 다른 tenant로 추가 배포
```bash
# Tenant 2
helm upgrade --install test-app-2 ./charts/test-app \
  -n test-app-2 \
  --create-namespace \
  --set tenantId=2

# Tenant 3
helm upgrade --install test-app-3 ./charts/test-app \
  -n test-app-3 \
  --create-namespace \
  --set tenantId=3
```

### 데이터 격리 확인
```bash
# Tenant 1 메트릭
curl "http://localhost:8481/select/1/vmui"

# Tenant 2 메트릭
curl "http://localhost:8481/select/2/vmui"

# Tenant 1 로그
curl -H "AccountID: 1" "http://localhost:9428/select/logsql/query" -d 'query=*'

# Tenant 2 로그
curl -H "AccountID: 2" "http://localhost:9428/select/logsql/query" -d 'query=*'
```

## 트러블슈팅

### Auto-Instrumentation이 작동하지 않음
```bash
# Instrumentation CR 확인
kubectl get instrumentation -n test-app

# Operator 로그 확인
kubectl logs -n opentelemetry-operator-system deployment/opentelemetry-operator-controller-manager

# Pod annotation 확인
kubectl get pod -n test-app -l app=test-app -o yaml | grep instrumentation
```

### Sidecar가 주입되지 않음
```bash
# Sidecar CR 확인
kubectl get opentelemetrycollector -n monitoring

# Pod annotation 확인
kubectl get pod -n test-app -l app=test-app -o yaml | grep sidecar
```

### 데이터가 VictoriaMetrics/Logs/Traces에 안 보임
```bash
# Sidecar 로그 확인
kubectl logs -n test-app -l app=test-app -c otc-container

# 수동으로 OTLP 전송 테스트
kubectl exec -it -n test-app deployment/test-app -c app -- \
  python -c "
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span('test-span'):
    print('Hello from test span!')
"
```

## 삭제

```bash
helm uninstall test-app -n test-app
kubectl delete namespace test-app
```
