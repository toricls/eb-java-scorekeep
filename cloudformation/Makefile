include ../aws.env

.PHONY: check-region stack clean

check-region:
	test -n "$(AWS_REGION)" || (echo "AWS_REGION must be defined in ../aws.env"; exit 1)
	test -n "$(IAM_USER_NAME)" || (echo "IAM_USER_NAME must be defined in ../aws.env"; exit 1)

stack: check-region
	aws --region $(AWS_REGION) cloudformation create-stack --stack-name $(IAM_USER_NAME)-scorekeep --parameters ParameterKey=IAMUserName,ParameterValue=$(IAM_USER_NAME) --capabilities "CAPABILITY_NAMED_IAM" --template-body file://cf-resources.yaml

clean: check-region
	aws --region $(AWS_REGION) cloudformation delete-stack --stack-name $(IAM_USER_NAME)-scorekeep
