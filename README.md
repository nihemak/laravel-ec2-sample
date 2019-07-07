# Setup Environment

## Create CodeCommit

```bash
git clone --mirror https://github.com/nihemak/laravel-ec2-sample.git
cd laravel-ec2-sample.git
aws cloudformation validate-template \
    --template-body file://infra/CodeStore.cfn.yml
aws cloudformation create-stack \
    --stack-name laravel-ec2-sample-CodeStore \
    --template-body file://infra/CodeStore.cfn.yml
git push ssh://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/laravel-ec2-sample --all
```

## Create CodeBuild ecr

```bash
aws cloudformation validate-template \
    --template-body file://infra/BuildEnv.cfn.yml
aws cloudformation create-stack \
    --stack-name laravel-ec2-sample-BuildEnv \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
      ParameterKey=CodeCommitStackName,ParameterValue=laravel-ec2-sample-CodeStore \
    --template-body file://infra/BuildEnv.cfn.yml
CODEBUILD_ID=$(aws codebuild start-build --project-name laravel-ec2-sample-ecr --source-version master | tr -d "\n" | jq -r '.build.id')
echo "started.. id is ${CODEBUILD_ID}"
while true
do
  sleep 10s
  STATUS=$(aws codebuild batch-get-builds --ids "${CODEBUILD_ID}" | tr -d "\n" | jq -r '.builds[].buildStatus')
  echo "..status is ${STATUS}."
  if [ "${STATUS}" != "IN_PROGRESS" ]; then
    if [ "${STATUS}" != "SUCCEEDED" ]; then
      echo "faild."
    fi
    echo "done."
    break
  fi
done
```

## Create VPC and EC2 instance

```bash
key_pair=$(aws ec2 create-key-pair --key-name test-laravel)
echo $key_pair | jq -r ".KeyMaterial" > test-laravel.pem
chmod 400 test-laravel.pem
```

```bash
aws cloudformation validate-template \
    --template-body file://infra/Network.cfn.yml
aws cloudformation create-stack \
    --stack-name laravel-ec2-sample-Network \
    --capabilities CAPABILITY_NAMED_IAM \
    --template-body file://infra/Network.cfn.yml
aws cloudformation validate-template \
    --template-body file://infra/EC2.cfn.yml
aws cloudformation create-stack \
    --stack-name laravel-ec2-sample-EC2 \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
      ParameterKey=NetworkStackName,ParameterValue=laravel-ec2-sample-Network \
    --template-body file://infra/EC2.cfn.yml
```

## Create Deploy CodePipeline

```bash
aws cloudformation validate-template \
    --template-body file://infra/DeployPipeline.cfn.yml
aws cloudformation create-stack \
    --stack-name laravel-ec2-sample-DeployPipeline \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
      ParameterKey=CodeCommitStackName,ParameterValue=laravel-ec2-sample-CodeStore \
    --template-body file://infra/DeployPipeline.cfn.yml
```

# Usage

```bash
ec2_ip=$(\
  aws cloudformation describe-stacks --stack-name laravel-ec2-sample-EC2 \
   | jq -r '.Stacks[].Outputs[] | select(.OutputKey == "EC2IP").OutputValue')
echo ${ec2_ip}
```

## Access to ssh

```bash
ssh -i test-laravel.pem ec2-user@${ec2_ip}
```

## Access to https

```
https://${ec2_ip}
```


