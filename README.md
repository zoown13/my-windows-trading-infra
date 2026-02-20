# Windows Trading Infra (MagicSplit on AWS)

매직스플릿(MagicSplit) 운영을 위한 Windows EC2 인프라를 CloudFormation으로 배포하는 가이드입니다.

## 포함 리소스

- VPC / 퍼블릭 서브넷 / IGW / 라우팅
- Windows Server 2022(Korean Full Base) EC2
- SSM 접속용 IAM Role/Instance Profile
- 선택적 Direct RDP(3389) 인바운드
- CloudWatch 알람 2종(CPU, StatusCheckFailed)
- 암호화된 루트 EBS(gp3)

## 사전 준비

- AWS CLI v2 설치 및 `aws configure` 완료
- EC2 Key Pair 생성 완료
- 알람 수신용 SNS Topic ARN 준비
- 사용 가능한 AZ 확인

```bash
aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[].ZoneName' \
  --output table
```

## 보안 원칙

- 기본값은 `UseDirectRdp=false` (권장)
- Direct RDP가 꼭 필요할 때만 `UseDirectRdp=true`
- 이 경우 `AllowedRdpIp`를 반드시 본인 공인 IP `/32`로 제한
- `0.0.0.0/0`는 운영 환경에서 사용 금지

## 배포 예시

```bash
aws cloudformation deploy \
  --stack-name my-windows-trading-infra \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    KeyName=my-keypair \
    InstanceType=t3.medium \
    AvailabilityZone=ap-northeast-2a \
    UseDirectRdp=false \
    AllowedRdpIp=203.0.113.10/32 \
    RootVolumeSize=100 \
    DisableApiTermination=true \
    AlarmNotificationTopicArn=arn:aws:sns:ap-northeast-2:123456789012:trading-alerts \
    ServiceName=magic-split \
    Environment=prod \
    Owner=trading-team
```

## 배포 후 확인

```bash
aws cloudformation describe-stacks \
  --stack-name my-windows-trading-infra \
  --query 'Stacks[0].Outputs' \
  --output table
```

출력값:
- `InstanceId`
- `InstancePublicIp`
- `DirectRdpEnabled`

## RDP 비밀번호 복호화

```bash
INSTANCE_ID=$(aws cloudformation describe-stack-resources \
  --stack-name my-windows-trading-infra \
  --logical-resource-id TradingWindowsInstance \
  --query 'StackResources[0].PhysicalResourceId' \
  --output text)

aws ec2 get-password-data \
  --instance-id "$INSTANCE_ID" \
  --priv-launch-key ~/.ssh/my-keypair.pem \
  --query PasswordData \
  --output text
```

> 비밀번호가 비어 있으면 인스턴스 초기화 중입니다. 잠시 후 재시도하세요.

## 업데이트 / 삭제

```bash
aws cloudformation deploy \
  --stack-name my-windows-trading-infra \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ...

aws cloudformation delete-stack --stack-name my-windows-trading-infra
```

## 비용 주의사항

- EC2 실행 시간 과금
- Public IPv4 과금 가능
- EBS 볼륨/스냅샷 지속 과금

운영 종료 시 스택 정리 또는 인스턴스 중지 정책을 반드시 운영하세요.
