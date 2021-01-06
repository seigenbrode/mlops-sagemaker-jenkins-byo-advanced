pipeline {

    agent any

    environment {
        AWS_ECR_LOGIN = 'true'
        DOCKER_CONFIG= "${params.JENKINSHOME}"
    }

    stages {
        stage("Checkout") {
            steps {
               checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/seigenbrode/mlops-sagemaker-jenkins-byo-advanced']]])
            }
        }

        stage("BuildPushContainer") {
            steps {
              sh """
                echo "${params.ECRURI}"
                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${params.ECRURI}
 	              docker build -t scikit-byo:${env.BUILD_ID} .
                docker tag scikit-byo:${env.BUILD_ID} ${params.ECRURI}:${env.BUILD_ID} 
                docker push ${params.ECRURI}:${env.BUILD_ID}
                echo ${params.S3_PACKAGED_LAMBDA}
              """
            }
        }
        
        stage("TrainModel") {
            steps { 
              sh """
               aws sagemaker create-training-job --training-job-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID} --algorithm-specification TrainingImage="${params.ECRURI}:${env.BUILD_ID}",TrainingInputMode="File" --role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST} --input-data-config '{"ChannelName": "training", "DataSource": { "S3DataSource": { "S3DataType": "S3Prefix", "S3Uri": "${params.S3_TRAIN_DATA}"}}}' --resource-config InstanceType='ml.c4.2xlarge',InstanceCount=1,VolumeSizeInGB=5 --output-data-config S3OutputPath='${S3_MODEL_ARTIFACTS}' --stopping-condition MaxRuntimeInSeconds=3600
              """
             }
        }

      stage("TrainStatus") {
            steps {
              script {
                    def response = sh """ 
                    TrainingJobStatus=`aws sagemaker describe-training-job --training-job-name \"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" | grep -Po \'"\'"TrainingJobStatus"\'"\\s*:\\s*"\\K([^"]*)\'`
                    echo \$TrainingJobStatus
                    while [ \$TrainingJobStatus = "InProgress" ] ; do
                      TrainingJobStatus=`aws sagemaker describe-training-job --training-job-name \"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" | grep -Po \'"\'"TrainingJobStatus"\'"\\s*:\\s*"\\K([^"]*)\'`
                      echo \$TrainingJobStatus
                      sleep 1m
                    done
                    """
                    
                  }
              }
      }

      stage("DeployToTest") {
            steps { 
              sh """
               if ! aws cloudformation describe-stacks --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-test ; then
                  echo -e "\nStack does not exist, creating ..."
                  aws cloudformation create-stack --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-test --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Test' ParameterKey=DataCaptureS3Bucket,ParameterValue="${params.SAGEMAKER_MM_BUCKET}/data-capture-test"
                  echo "Waiting for stack to be created ..."
                  aws cloudformation wait stack-create-complete --region us-east-1 --stack-name "${params.SAGEMAKER_TRAINING_JOB}"-test
               else
                  echo -e '\nStack exists, attempting update ...'
                  set +e
                  update_output=`aws cloudformation update-stack --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-test --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Test' ParameterKey=DataCaptureS3Bucket,ParameterValue="${params.SAGEMAKER_MM_BUCKET}/data-capture-test"`
                  status=\$?
                  set -e
                  echo \$update_output
                  if [ \$status -ne 0 ] ; then
                  # Don't fail for no-op update
                    if [[ \$update_output == *"ValidationError"* && \$update_output == *"No updates"* ]] ; then
                      echo -e "\nFinished create/update - no updates to be performed"
                      exit 0
                    else
                      exit \$status
                    fi
                  fi

               echo "Waiting for stack update to complete ..."
               aws cloudformation wait stack-update-complete --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-test

               fi
               echo "Finished create/update successfully!"
              """
             }
        }

      stage("TestEvaluate") {
            steps { 
              script {
                 def response = sh """ 
                 aws lambda invoke --function-name ${params.LAMBDA_EVALUATE_MODEL} --cli-binary-format raw-in-base64-out --region us-east-1 --payload '{"EndpointName": "'${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}'-Test", "Body": {"Payload": {"S3TestData": "${params.S3_TEST_DATA}", "S3Key": "test/iris.csv"}}}' evalresponse.json
              """
              }
            }
        }

      stage("BaselineModel") {
            steps { 
              script {
                 def response = sh """ 
                 aws sagemaker create-processing-job --processing-inputs '{"InputName": "baseline_dataset_input","S3Input": {"S3Uri": "${params.S3_TRAIN_DATA}","LocalPath": "/opt/ml/processing/input/baseline_dataset_input","S3DataType":"S3Prefix","S3InputMode":"File","S3DataDistributionType":"FullyReplicated","S3CompressionType":"None"}}'  --processing-output-config '{"Outputs": [{"OutputName": "monitoring_output","S3Output": {"S3Uri": "${params.SAGEMAKER_MM_BUCKET}/baseline","LocalPath" : "/opt/ml/processing/output","S3UploadMode" : "EndOfJob"}}] }' --processing-job-name '${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}' --processing-resources '{"ClusterConfig": {"InstanceCount": 1,"InstanceType": "ml.m5.xlarge","VolumeSizeInGB":15}}' --stopping-condition MaxRuntimeInSeconds=3600 --app-specification ImageUri='${params.MMBASELINE_ECRURI}' --role-arn '${params.SAGEMAKER_EXECUTION_ROLE_TEST}' --environment \"{"dataset_format": "{\"csv\": {\"header\": "true", \"output_columns_position\": \"START\"}}","dataset_source": "/opt/ml/processing/input/baseline_dataset_input","output_path": "/opt/ml/processing/output","publish_cloudwatch_metrics": "Disabled"}\"
              """
              }
            }
        }

      stage("DeployToProd") {
            steps { 
              sh """
               if ! aws cloudformation describe-stacks --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-prod ; then
                  echo -e "\nStack does not exist, creating ..."
                  aws cloudformation create-stack --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-prod --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Prod'
                  ParameterKey=DataCaptureS3Bucket,ParameterValue="${params.SAGEMAKER_MM_BUCKET}/data-capture-prod"
                  echo "Waiting for stack to be created ..."
                  aws cloudformation wait stack-create-complete --region us-east-1 --stack-name "${params.SAGEMAKER_TRAINING_JOB}"-prod
               else
                  echo -e '\nStack exists, attempting update ...'
                  set +e
                  update_output=`aws cloudformation update-stack --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-prod --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Prod'`
                  ParameterKey=DataCaptureS3Bucket,ParameterValue="${params.SAGEMAKER_MM_BUCKET}/data-capture-prod"
                  status=\$?
                  set -e
                  echo \$update_output
                  if [ \$status -ne 0 ] ; then
                  # Don't fail for no-op update
                    if [[ \$update_output == *"ValidationError"* && \$update_output == *"No updates"* ]] ; then
                      echo -e "\nFinished create/update - no updates to be performed"
                      exit 0
                    else
                      exit \$status
                    fi
                  fi
               echo "Waiting for stack update to complete ..."
               aws cloudformation wait stack-update-complete --region us-east-1 --stack-name '${params.SAGEMAKER_TRAINING_JOB}'-prod
               fi
               echo "Finished create/update successfully!"
              """
             }
        }

  }
}   
