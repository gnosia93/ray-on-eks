
### 1. vpc 생성 ###
```
cd ~/ray-on-aws/lesson
pwd

aws cloudformation create-stack \
  --stack-name ray-vpc \
  --template-body file://$(pwd)/template/ray-vpc.yaml
  --parameters ParameterKey=KeyPairName,ParameterValue=ap-northeast-1 \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=ray-on-aws
```

### 2. Role 생성 ###
Ray Head 노드가 Worker 노드들을 생성/삭제할 수 있도록 ray-instance-profile을 생성한다.
```
cat <<EOF > ray-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ray-autoscaling-role \
  --assume-role-policy-document file://ray-trust-policy.json
cat <<EOF > ray-autoscaling-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeSubnets",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ray-autoscaling-role \
  --policy-name ray-autoscaling-policy \
  --policy-document file://ray-autoscaling-policy.json

# 인스턴스 프로파일 생성
aws iam create-instance-profile --instance-profile-name ray-instance-profile

# 프로파일에 역할 추가
aws iam add-role-to-instance-profile \
  --instance-profile-name ray-instance-profile \
  --role-name ray-autoscaling-role
```
