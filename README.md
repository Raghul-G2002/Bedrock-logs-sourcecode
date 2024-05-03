# Send logs of Bedrock to Amazon CloudWatch
This repository will give you an overview of adding logs of bedrock using bedrock client to CloudWatch with some sample calls to the Model.

# About CloudWatch
Amazon CloudWatch is a Management and Governance AWS Service that allows you to collect, access, and correlate metrics, logs, and events data on a single platform from across all your AWS resources, applications, and services running on AWS and on-premises. It breaks down data silos (Silos are structures that store bulk materials, typically in a cylindrical shape) to better understand the health and performance of our resources. Amazon CloudWatch helps you detect anomalous behavior in your environments, set alarms (CloudWatch Alarms) in your environments, visualize logs (CloudWatch Logs), and metrics (CloudWatch Metrics) side by side, take automated actions, troubleshoot issues, and discover insights to keep your application running smoothly. 

>How did I use CloudWatch to send events from Bedrock?
To send the logs of bedrock to CloudWatch, I’ve used two important things namely
1.	Helping Class called CloudWatch Helper to support creating log groups from SDK, and printing recent log streams by using time stamps. 
2.	Created a logging config for the bedrock client to put the model logging configuration to the Cloud Watch.

Here is the Logging Configuration template, I’ve used.
 ```python
loggingConfig = {
    'cloudWatchConfig': {
        'logGroupName':log_group_name,
        'roleArn':'arn:aws:iam::228947353622:role/MyIAMRole',
        'largeDataDeliveryS3Config': {
            'bucketName':"bedrock-logs-sample", 
            'keyPrefix':"amazon_bedrock_large_data_delivery",
        }
    },
    's3Config': {
        'bucketName':"bedrock-logs-sample",
        'keyPrefix':"amazon_bedrock_logs",
    },
    'textDataDeliveryEnabled': True
}
```
The wholesome architecture is given below
![Cloud Watch Logs - Amazon Bedrock](https://github.com/Raghul-G2002/Bedrock-logs-sourcecode/assets/83855692/c7d08911-7860-445d-8018-341e2c4ccd25)

# Create an IAM Role using IaC Templates
>This is the IaC template I’ve used to create the IAM Role called MyIAMRole and attach the policy namely MyRolePolicy so that the role can be attached to Amazon Bedrock.

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Description": "CloudFormation Template to Create an IAM Role with a Trust Relationship and Permissions Policy",
    "Resources": {
        "MyIAMRole": {
            "Type": "AWS::IAM::Role",
            "Properties": {
                "RoleName": "MyIAMRole",
                "AssumeRolePolicyDocument": {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Principal": {
                                "Service": "bedrock.amazonaws.com"
                            },
                            "Action": "sts: AssumeRole",
                        }
                    ]
                }
            }
        }
    },
    "MyRolePolicy": {
            "Type": "AWS::IAM::Policy",
            "Properties": {
                "PolicyName": "MyIAMRolePolicy",
                "Roles": [
                    "MyIAMRole"
                ],
                "PolicyDocument": {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Action": [
                                "logs: CreateLogStream",
                                "logs: PutLogEvents"
                            ],
                            "Resource": "arn:aws:logs:us-east-1:228947353622:log-group:bedrock-logs:*"
                        }
                    ]
                }
            }
        },
    "Outputs": {
        "RoleArn": {
            "Description": "The ARN of the created IAM Role",
            "Value": "MyIAMRole.Arn"
        }
    }
}
```

1. Create a Stack in CloudFormation using the command below

```
aws cloudformation create-stack --stack-name IAMRoleBedrock --template-body file://cloudformation.json --capabilities CAPABILITY_NAMED_IAM
```
2. Describe the Stack to check whether the stack has been created successfully with the underlying resources.

```
 aws cloudformation describe-stacks --stack-name IAMRoleStack
```
![image](https://github.com/Raghul-G2002/Bedrock-logs-sourcecode/assets/83855692/9bf00f99-4634-47bc-9e86-ca0ce56d1304)
![image](https://github.com/Raghul-G2002/Bedrock-logs-sourcecode/assets/83855692/741dff22-21bc-4558-b35c-abf8ef95a0f0)

Now, you are good to go by adding the role in the logging config shown above. 

# Check with Bedrock Runtime Client

Run a sample prompt with the model to check whether the logs have been captured in the CloudWatch Logs
```python
#Create a Model
prompt = "Write an article about Mahendra Singh Dhoni, formerly known as MSD."

kwargs = {
  "modelId": "amazon.titan-text-express-v1",
  "contentType": "application/json",
  "accept": "application/json",
  "body": json.dumps(
      {
          "inputText":prompt,
          "textGenerationConfig":
          {
              "maxTokenCount":4096,
              "stopSequences":["User:"],
              "temperature":0.8,
              "topP":0.9
              }
              }
  )
}

response = bedrock_runtime.invoke_model(**kwargs)
response_body = json.loads(response['body'].read())

generated_text = response_body['results'][0]['outputText']
```
Here are the CloudWatch Logs from the associated log groups
```json
{
    "schemaType": "ModelInvocationLog",
    "schemaVersion": "1.0",
    "timestamp": "2024-05-02T15:07:29Z",
    "accountId": "228947353622",
    "identity": {
        "arn": "arn:aws:iam::228947353622:user/RaghulG"
    },
    "region": "us-east-1",
    "requestId": "8fe3c68a-1fe5-4051-99b5-ebaf3738f76b",
    "operation": "InvokeModel",
    "modelId": "amazon.titan-text-express-v1",
    "input": {
        "inputContentType": "application/json",
        "inputBodyJson": {
            "inputText": "Write an article about Mahendra Singh Dhoni, formerly known as MSD.",
            "textGenerationConfig": {
                "maxTokenCount": 4096,
                "stopSequences": [
                    "User:"
                ],
                "temperature": 0.8,
                "topP": 0.9
            }
        },
        "inputTokenCount": 17
    },
    "output": {
        "outputContentType": "application/json",
        "outputBodyJson": {
            "inputTextTokenCount": 17,
            "results": [
                {
                    "tokenCount": 1161,
                    "outputText": "\nMahendra Singh Dhoni, a former Indian captain and wicket-keeper batsman, is widely regarded as one of the greatest cricket captains of all time. He led India to unprecedented success in international cricket, including two ICC Cricket World Cups (2007 and 2011), as well as numerous other tournaments. Dhoni's calm and composed demeanor on the field, coupled with his exceptional skills as a wicket-keeper and batsman, made him a fan favorite worldwide. In this article, we will take a closer look at the life and career of Mahendra Singh Dhoni.\n\nEarly Life and Background:\n\nMahendra Singh Dhoni was born on July 7, 1981, in Ranchi, Jharkhand, India. He grew up in a middle-class family and showed an early interest in cricket. Dhoni's father, Pan Singh, was a police officer, while his mother, Devaki Devi, was a homemaker. Dhoni has an elder sister, Deepa, and a younger brother, Jayant.\n\nCricket Career:\n\nMahendra Singh Dhoni's cricketing journey began at a young age. He started playing cricket in his hometown of Ranchi and quickly caught the attention of local coaches. Dhoni played for his school and local club teams and soon gained recognition for his exceptional skills as a wicket-keeper and batsman.\n\nIn 2000, Dhoni made his debut for the Indian Under-19 cricket team and impressed with his performances. He was selected for the Indian squad for the 2002 ICC Under-19 Cricket World Cup, where he played a pivotal role in India's victory. Dhoni's performances in the tournament earned him a place in the Indian Senior Cricket Team, and he made his debut for the team in 2004.\n\nMahendra Singh Dhoni's Career as a Captain:\n\nMahendra Singh Dhoni's career as a captain began in 2007 when he was appointed as the captain of the Indian ODI team. He quickly gained popularity among fans and critics alike for his calm and composed demeanor on the field, as well as his ability to lead the team from the front. Dhoni's captaincy style was characterized by his decision-making skills, his ability to read the game, and his ability to inspire his players.\n\nUnder Dhoni's captaincy, India became one of the most successful cricket teams in the world. They won numerous international tournaments, including two ICC Cricket World Cups (2007 and 2011), the ICC Champions Trophy (2008), and the ICC World Twenty20 (2007 and 2009). Dhoni's leadership was particularly evident in the 2011 Cricket World Cup, where India emerged as the champions after defeating Sri Lanka in the final.\n\nMahendra Singh Dhoni's Retirement:\n\nMahendra Singh Dhoni's retirement from international cricket came in 2016. He announced his retirement after the conclusion of the 2016 Cricket World Cup, which India had failed to reach the finals. Dhoni's retirement was met with mixed reactions from fans and critics, but it was widely accepted that he had left the game on a high note.\n\nAfter Retirement:\n\nAfter retirement, Mahendra Singh Dhoni remained active in the cricket world. He was appointed as the mentor of the Indian Premier League (IPL) team Chennai Super Kings, which he led to victory in the 2018 IPL. Dhoni also played a few innings for his former franchise, the Delhi Daredevils, in the IPL.\n\nMahendra Singh Dhoni's Legacy:\n\nMahendra Singh Dhoni's legacy in cricket is immense. He is widely regarded as one of the greatest captains of all time, and his achievements as the captain of the Indian cricket team are nothing short of remarkable. Dhoni's calm and composed demeanor on the field, coupled with his exceptional skills as a wicket-keeper and batsman, made him a fan favorite worldwide.\n\nDhoni's success as a captain was largely due to his decision-making skills, his ability to read the game, and his ability to inspire his players. He was known for his ability to keep calm under pressure, and his calmness often translated into success for the team. Dhoni's leadership was also characterized by his ability to build strong relationships with his players, which helped to create a sense of unity and camaraderie within the team.\n\nIn addition to his success as a captain, Mahendra Singh Dhoni was also an exceptional batsman. He was known for his ability to score quickly and build partnerships with his teammates. Dhoni's batting style was characterized by his calm demeanor and his ability to stay focused even in the most challenging situations.\n\nConclusion:\n\nMahendra Singh Dhoni is a legendary figure in the world of cricket. He is widely regarded as one of the greatest captains of all time, and his achievements as the captain of the Indian cricket team are nothing short of remarkable. Dhoni's calm and composed demeanor on the field, coupled with his exceptional skills as a wicket-keeper and batsman, made him a fan favorite worldwide retirement, Mahendra Singh Dhoni remained active in the cricket world, and he continued to inspire young cricketers with his calm and composed demeanor. His legacy in cricket will continue to be felt for years to come, and he will always be remembered as one of the greatest players of his generation.",
                    "completionReason": "FINISH"
                }
            ]
        },
        "outputTokenCount": 1161
    }
}
```
>An alternate way, is to use Amazon Bedrock Console to invoke the logs to the CloudWatch.
![Bedrock Logs](https://github.com/Raghul-G2002/Bedrock-logs-sourcecode/assets/83855692/4dd821dd-70e0-48e0-aef9-5d3937752932)

# Repo Access
Clone the repo using the following Command
```
git clone https://github.com/Raghul-G2002/Bedrock-logs-sourcecode.git
```
> Stay Connected with me <br>
> 🔗 Raghul Gopal Linkedin: https://www.linkedin.com/in/raghulgopaltech/ <br>
> 🔗Raghul Gopal YouTube: https://www.youtube.com/@rahulg2980 <br>
> 📒Subscribe to my Newsletter: Subscribe on LinkedIn https://www.linkedin.com/build-relation/newsletter-follow?entityUrn=7183725729254158336







