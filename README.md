# My Windows Trading Infra

AWS CLI로 Windows 기반 트레이딩 인프라를 생성/운영할 때 필요한 기본 절차를 정리한 문서입니다.

## 사전 준비

- AWS CLI v2 설치 및 `aws configure` 완료
- PEM 키 페어(`.pem`) 보관 경로 확인
- 원격 접속용 RDP 클라이언트 준비(Windows 원격 데스크톱, Microsoft Remote Desktop 등)

예시 환경 변수:

```bash
export AWS_REGION=ap-northeast-2
export STACK_NAME=my-windows-trading-infra
export TEMPLATE_FILE=infra/cloudformation/windows-trading.yaml
export KEY_PAIR_NAME=my-keypair
export KEY_FILE=~/.ssh/my-keypair.pem
```

## CLI 예시: 스택 생성부터 접속 확인까지

### 1) `create-stack`

```bash
aws cloudformation create-stack \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --template-body "file://$TEMPLATE_FILE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue="$KEY_PAIR_NAME" \
    ParameterKey=InstanceType,ParameterValue=t3.large
```

### 2) 생성 완료 대기

```bash
aws cloudformation wait stack-create-complete \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"
```

### 3) `describe-stacks`로 출력값 확인

```bash
aws cloudformation describe-stacks \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --query "Stacks[0].Outputs"
```

확인 포인트:
- `InstanceId`
- `PublicIp` 또는 `PublicDns`
- (템플릿에서 제공 시) `RDPEndpoint`

### 4) EC2 인스턴스 ID/공인 IP 조회

```bash
INSTANCE_ID=$(aws cloudformation describe-stack-resources \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --logical-resource-id WindowsInstance \
  --query "StackResources[0].PhysicalResourceId" \
  --output text)

PUBLIC_IP=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)
```

### 5) RDP 관리자 비밀번호 복호화

Windows 인스턴스 초기화가 끝난 뒤(보통 부팅 후 수 분), 아래로 암호를 조회합니다.

```bash
aws ec2 get-password-data \
  --region "$AWS_REGION" \
  --instance-id "$INSTANCE_ID" \
  --priv-launch-key "$KEY_FILE" \
  --query PasswordData \
  --output text
```

> 출력값이 비어 있으면 아직 준비되지 않은 상태입니다. 30~60초 간격으로 재시도하세요.

### 6) RDP 접속 확인 절차

1. RDP 클라이언트에서 대상 주소에 `PUBLIC_IP:3389` 입력
2. 사용자명 `Administrator` 입력
3. 위 단계에서 복호화한 비밀번호 입력
4. 인증서 경고(자체 서명) 확인 후 접속 진행
5. 로그인 후 아래를 확인
   - CPU/메모리 사용량 정상
   - 네트워크 아웃바운드 가능
   - 매직스플릿 앱 프로세스/로그 경로 접근 가능

---

## 스택 운영 명령

### 스택 업데이트 (`update-stack`)

```bash
aws cloudformation update-stack \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --template-body "file://$TEMPLATE_FILE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,UsePreviousValue=true \
    ParameterKey=InstanceType,ParameterValue=t3.xlarge

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"
```

변경 사항이 없는 경우 `No updates are to be performed.` 메시지가 발생할 수 있습니다.

### 스택 삭제 (`delete-stack`)

```bash
aws cloudformation delete-stack \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"

aws cloudformation wait stack-delete-complete \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"
```

삭제 전 백업 체크:
- 전략 설정 파일
- 체결/거래 로그
- 운영 스크립트 및 인증서

### 비용 주의사항 (EC2 / 공인 IP / EBS)

- **EC2 인스턴스**: 실행 시간 기준 과금. 장시간 상시 실행 시 월 비용 증가.
- **공인 IPv4**: AWS 정책에 따라 할당/사용 중 공인 IP도 과금될 수 있음.
- **EBS 볼륨**: 인스턴스 중지/삭제 후에도 볼륨이 남으면 계속 과금.

권장 사항:
- 미사용 시 스택 삭제 또는 인스턴스 중지
- EBS 스냅샷 보관 정책 수립
- AWS Budgets/Cost Anomaly Detection 알림 설정

---

## 운영 가이드

### 매직스플릿 앱 배치 위치

권장 디렉터리 구조:

```text
C:\Trading\MagicSplit\
  ├─ app\            # 실행 파일/패키지
  ├─ config\         # 전략/환경설정
  ├─ logs\           # 런타임 로그
  └─ scripts\        # 배포/재기동 스크립트
```

운영 권장:
- 실행 계정(예: `svc_trading`) 권한 최소화
- `config` 폴더는 백업/버전관리 분리
- 로그 롤링 정책(용량/기간) 적용

### 자동 시작 설정

#### A. Windows Task Scheduler

- 트리거: `At startup`
- 실행 계정: 서비스용 계정(로그온 여부 무관)
- 동작: `C:\Trading\MagicSplit\app\MagicSplit.exe`
- 옵션:
  - `Run with highest privileges`
  - 실패 시 재시도(예: 1분 간격, 5회)

#### B. Windows 서비스 등록(대안)

앱이 서비스 모드를 지원하거나 `nssm` 같은 래퍼를 사용할 수 있습니다.

체크 항목:
- 시작 유형: `Automatic (Delayed Start)`
- 복구 설정: 1/2/이후 실패 시 `Restart the Service`
- 서비스 계정의 파일/네트워크 접근 권한

### 장애 시 재기동 체크리스트

1. **프로세스 상태 확인**: 앱 프로세스 존재 여부 / 좀비 상태 확인
2. **로그 확인**: `C:\Trading\MagicSplit\logs` 최근 오류/예외 확인
3. **네트워크 점검**: 브로커 API/DNS/NTP 연결 정상 여부
4. **자격 증명 점검**: 만료된 토큰/인증서/비밀번호 확인
5. **자동 시작 설정 확인**: Task Scheduler 이력, 서비스 복구 정책 확인
6. **수동 재기동**: 정상 종료 후 재실행, 포트 충돌 여부 확인
7. **재발 방지**: 원인 기록(RCA), 알림 임계치/헬스체크 보강

---

## 참고

- 프로덕션 전 스테이징 환경에서 템플릿/시작 스크립트 검증
- 점검 창(maintenance window) 동안 업데이트/재배포 수행
