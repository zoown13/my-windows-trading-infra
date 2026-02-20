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

