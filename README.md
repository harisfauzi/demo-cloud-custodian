# README

This is a demonstration on how to implement [Cloud Custodian](https://cloudcustodian.io/).

## Requirement

- python >= 3.7.
- AWS account.
- AWS profile that allows you to deploy resources to the AWS account.
- AWS CloudTrail enabled.
- AWS Config enabled.
- AWS Security Hub enabled.
- AWS Security Hub integration with Cloud Custodion has been configured to accept findings from Cloud Custodian.
- Integration with Security Hub in the AWS account.

## Deployment

Some IAM roles are need for the Lambda functions provided by Cloud Custodian to work. To create the IAM roles, do the following:

1. Logon to your AWS account using CLI (e.g. using [saml2aws](https://github.com/Versent/saml2aws)) and set the AWS authentication environment variables:

   ```shell
   saml2aws
   AWS_PROFILE=<my profile>
   AWS_DEFAULT_REGION=us-west-2
   export AWS_PROFILE AWS_DEFAULT_REGION
   ```

2. Install [sceptre](https://github.com/Sceptre/sceptre) in a virtual environment:

   ```shell
   virtualenv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

3. Deploy the CloudFormation template for the IAM Roles:

   ```shell
   cd sceptre
   sceptre --var aws_region="${AWS_DEFAULT_REGION}" launch -y common.yaml
   ```

4. Deploy the Cloud Custodian demo policy:

   ```shell
   cd ../custodian
   custodian run \
     --region "${AWS_DEFAULT_REGION}" \
     --output-dir logs \
     --log-group "/c7n/demo-cloud-custodian" \
     security.yaml
   ```

## Testing

To test if the Cloud Custodian policy defined in [security.yaml](custodian/security.yaml) works, you can deploy the CloudFormation in [demo-securitygroup.yaml](sceptre/config/test-sg/demo-securitygroup.yaml). Run the following:

1. Authenticate to the AWS account.
2. Get the VPC ID from an existing VPC, e.g. `vpc-0123456789abcdef`.
3. Deploy the demo CloudFormation (please inspect the content). It will have two security groups, one with one inbound rule that allows inbound traffic from all IPv4 address on all protocols, another with three inbound rules from all IPv4 address on port 80 (approved), 443 (approved), and 8080 (not approved). After the deployment, the first security group named Demo01 will have no inbound rule, the second security group will only have two inbound rules that has approved ports.

   ```shell
   cd ../sceptre
   VPC_ID=vpc-0123456789abcdef
   sceptre --var vpc_id="${VPC_ID}" --var aws_region="${AWS_DEFAULT_REGION}" launch -y test-sg
   ```

4. Inspect the produced security groups to see if Cloud Custodian removed the offending rules.
5. To cleanup the demo, run the following:

   ```shell
   cd ../sceptre
   VPC_ID=vpc-0123456789abcdef
   sceptre --var vpc_id="${VPC_ID}" --var aws_region="${AWS_DEFAULT_REGION}" delete -y test-sg
   ```

## Cleanup

1. Delete all Lambda Functions prefixed with `demo-cloud-custodian-` prefix.

   ```shell
   CC_LAMBDA_LIST=$(aws lambda list-functions \
     --query "Functions[?starts_with(FunctionName,'demo-cloud-custodian-')].[FunctionName]" \
     --output text)
   for CC_LAMBDA_NAME in ${CC_LAMBDA_LIST}; do
     aws lambda delete-function --function-name "${CC_LAMBDA_NAME}"
   done
   ```

2. Delete all CloudWatch Event (EventBridge) rules with `demo-cloud-custodian-` prefix.

   ```shell
   CC_RULES_LIST=$(aws events list-rules \
     --query "Rules[?starts_with(Name,'demo-cloud-custodian')].[Name]" \
     --output text)
   for CC_RULE_NAME in ${CC_RULES_LIST}; do
      RULE_TARGETS=$(aws events list-targets-by-rule \
        --rule "${CC_RULE_NAME}" \
        --query "Targets[].Id" --output text)
      for RULE_TARGET_ID in ${RULE_TARGETS}; do
        aws events remove-targets --rule "${CC_RULE_NAME}" --ids "${RULE_TARGET_ID}"
      done
      aws events delete-rule --name "${CC_RULE_NAME}"
   done
   ```

3. Delete the CloudFormation for the IAM Roles.

   ```shell
   cd ../sceptre
   sceptre --var aws_region="${AWS_DEFAULT_REGION}" delete -y common.yaml
   ```
