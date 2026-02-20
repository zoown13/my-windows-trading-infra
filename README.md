# my-windows-trading-infra

CloudFormation 템플릿(`template.yaml`)은 EC2 인스턴스를 생성하고, 운영 알람 및 CloudWatch Agent 로그 수집을 함께 설정합니다.

## 포함된 운영 알람

- `HighCpuAlarm`: `CPUUtilization >= 80%` 상태가 5분 간격 2회 연속 발생 시 ALARM
- `StatusCheckFailedAlarm`: `StatusCheckFailed >= 1` 상태가 1분 간격에서 감지되면 ALARM

두 알람 모두 `AlarmNotificationTopicArn` 파라미터로 전달한 SNS Topic ARN에 알림을 전송합니다.

## 알람 대상(SNS) 파라미터

템플릿 파라미터:

- `AlarmNotificationTopicArn`: 운영자가 관리하는 SNS Topic ARN

예시 배포:

```bash
aws cloudformation deploy \
  --stack-name trading-infra \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    KeyName=<ec2-keypair-name> \
    AlarmNotificationTopicArn=arn:aws:sns:ap-northeast-2:123456789012:trading-alerts
```

> SNS Topic 구독에 이메일을 추가하면 메일 알림을 받을 수 있고, AWS Chatbot을 이용해 Slack 채널과도 연동할 수 있습니다.

## EC2 CloudWatch Agent 설치 및 로그 전송

`TradingServer` 리소스의 `UserData`에서 다음을 자동 수행합니다.

1. `amazon-cloudwatch-agent` 패키지 설치
2. 에이전트 설정 파일(`/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`) 생성
3. `/var/log/messages`, `/var/log/cloud-init.log`를 CloudWatch Logs로 전송 시작

## 로그 전송 IAM 권한(정책)

템플릿은 EC2 IAM Role(`EC2Role`)에 AWS 관리형 정책 `CloudWatchAgentServerPolicy`를 연결합니다.
이 정책에는 CloudWatch Logs 전송 및 메트릭/상태 보고에 필요한 권한이 포함됩니다.

직접 최소 권한 정책을 구성하려면 아래 액션이 필요합니다(예시):

- `logs:CreateLogGroup`
- `logs:CreateLogStream`
- `logs:DescribeLogStreams`
- `logs:PutLogEvents`
- `cloudwatch:PutMetricData`
- `ec2:DescribeTags`
- `ssm:GetParameter`

CloudFormation 배포 전에 사용 가능한 가용 영역(AZ)을 먼저 확인하고 `AvailabilityZone` 파라미터에 입력하세요.

```bash
aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output table
```

예시:
# Windows Trading Infra

이 저장소는 Windows 기반 트레이딩 워크로드를 위한 CloudFormation 템플릿(`template.yaml`)을 제공합니다.

## 보안 접속 원칙

- 기본값은 **SSM(Session Manager) 기반 접속**입니다.
- 직접 RDP(3389) 인바운드는 운영상 꼭 필요한 예외 상황에서만 제한된 IP로 허용합니다.

## 템플릿 주요 변경 사항

- `UseDirectRdp` 파라미터(`true`/`false`, 기본값 `false`)를 추가했습니다.
- `UseDirectRdp=false`일 때는 `Conditions`를 통해 3389 인바운드 규칙을 생성하지 않습니다.
- EC2 인스턴스에 `IamInstanceProfile`을 연결하고, 최소 SSM 접속 권한으로 `AmazonSSMManagedInstanceCore`를 사용합니다.

## 운영 시나리오

### 1) 권장: SSM 접속

권장 운영 모델입니다.

1. `UseDirectRdp=false`로 스택을 배포합니다.
2. EC2 인스턴스에 연결된 `IamInstanceProfile`(역할)에 의해 SSM Agent가 Systems Manager에 등록됩니다.
3. 운영자는 AWS Systems Manager Session Manager를 통해 인스턴스에 접속합니다.

전제 조건:

- AMI에 SSM Agent가 포함되어 있거나, 부팅 후 SSM Agent가 설치/실행되어야 합니다.
- 인스턴스가 SSM 엔드포인트(인터넷/NAT 또는 VPC Endpoint)로 통신 가능해야 합니다.

### 2) 예외: 제한된 IP의 직접 RDP

예외적으로 GUI 기반 접근이 즉시 필요한 경우에만 사용합니다.

1. `UseDirectRdp=true`로 스택을 배포합니다.
2. `AllowedRdpCidr`를 운영자 고정 공인 IP 대역(예: `/32`)으로 제한합니다.
3. 작업이 끝나면 `UseDirectRdp=false`로 되돌려 3389 인바운드를 제거합니다.

## 배포 예시

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name windows-trading-infra \
  --parameter-overrides \
    VpcId=vpc-xxxxxxxx \
    InstanceSubnetId=subnet-xxxxxxxx \
    UseDirectRdp=false \
    AllowedRdpCidr=203.0.113.10/32
# my-windows-trading-infra

## 배포 예시

> ⚠️ `AllowedRdpIp=본인공인IP/32` 입력은 **필수**입니다.
> 
> `0.0.0.0/0`은 보안상 매우 위험하므로 **운영 환경에서는 사용 금지**이며, 디버깅을 위한 임시 용도로만 제한적으로 사용하세요.

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-windows-trading-infra \
  --parameter-overrides AvailabilityZone=ap-northeast-2a
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides AllowedRdpIp=본인공인IP/32
```
