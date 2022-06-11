# Using Amazon Web Services from Rust-based Kubernetes applications using Rusoto and IAM Roles for Service Accounts

In order to use Amazon Web Services, such as the Simple Queue Service (SQS), the API client must supply IAM credentials that would authenticate it and help determine whether it is allowed to perform the requested operation on the target resource (e.g., send a message to a queue).

[Rusoto's](https://github.com/rusoto/rusoto) default credentials provider supports the most common use-cases, such as supplying AWS credentials via environment variables and user's credential files. More advanced use-cases, such as authentication using STS tokens, require additional configuration.

This project delivers a rudimentary Amazon SQS consumer application running in Kubernetes (on EKS), which polls for, receives, and logs messages from the configured SQS queue. But rather than using static AWS credentials, it runs under a Kubernetes service account linked to an IAM role through a time-constrained STS token.

The best part -- with a little of bit configuration, the management of this time-constrained, periodically refreshed security token is done automatically!

## Pre-requisites

1. Rust nightly
2. Push access to a Docker registry (you can set one up on GitHub)
3. Amazon Web Services account with admin-level credentials
4. Access to an Amazon SQS queue in that account (the examples refer to a queue named `rusoto-sqs-k8s-demo` in the `us-east-1` region)

> Note: In the snippets below you may see references to AWS account #1234567890. This is a placeholder; please replace it with your own account number, which you can obtain by executing `aws sts get-caller-identity --query Account --output text`.

## Building for deployment

To build your own Docker image for deployment into Kubernetes, make sure you have push access to a Docker registry. If you set one up on GitHub:

```bash
docker build -t 652839185683.dkr.ecr.us-east-1.amazonaws.com/demo:rusoto-sqs-k8s-demo .
aws ecr get-login-password --region us-east-1 --no-cli-auto-prompt | docker login --username AWS --password-stdin 652839185683.dkr.ecr.us-east-1.amazonaws.com && docker push 652839185683.dkr.ecr.us-east-1.amazonaws.com/demo:rusoto-sqs-k8s-demo
```

## Deploying into EKS

### IAM policy

You need a policy that grants access to your SQS queue. To create one, e.g.:

```bash
aws iam create-policy --policy-name RusotoSQSK8sDemoConsumer --policy-document '{"Version": "2012-10-17", "Statement": [{"Effect": "Allow", "Action": ["sqs:DeleteMessage", "sqs:GetQueueUrl", "sqs:ChangeMessageVisibility", "sqs:DeleteMessageBatch", "sqs:ReceiveMessage", "sqs:GetQueueAttributes", "sqs:ChangeMessageVisibilityBatch"], "Resource": ["arn:aws:sqs:*:1234567890:*"]}]}'
```

### Service account with attached IAM role and policy

To create a Kubernetes service account for your pods linked to a dedicated IAM role that has your IAM policy attached to it, in one step:

```bash
eksctl create iamserviceaccount --name rusoto-sqs-consumer --namespace demo --cluster rusoto-sqs-demo --attach-policy-arn  arn:aws:iam::123456789:policy/RusotoSQSK8sDemoConsumer --approve
```

### Container secrets

For configuration values that you don't want to store in plain text:

```bash
kubectl -n demo create secret generic rusoto-sqs-k8s-demo-secrets --from-literal=queue_url=https://sqs.us-east-1.amazonaws.com/1234567890/rusoto-sqs-k8s-demo
```

### Create deployment

If you're using your own Docker registry, then you must update the [deployment.yaml](./deployment.yaml) file and replace the reference to the `docker.pkg.github.com/ecliptical/rusoto-sqs-k8s-demo/rusoto-sqs-k8s-demo:v1` image with your own.

Finally, to deploy the application:

```bash
kubectl apply -f manifests
```

### Tailing the logs

To see if your application works (i.e., no errors, and received/processed messages are logged):

```bash
kubectl -n demo logs -f -l app.kubernetes.io/name=rusoto-sqs-k8s-demo

{"timestamp":"Aug 23 23:57:51.508","level":"INFO","target":"rusoto_sqs_k8s_demo","fields":{"message":"rusoto-sqs-k8s-demo 0.1.0 (4b04653, release build, linux [x86_64], Sun, 23 Aug 2020 19:12:11 +0000)","log.target":"rusoto_sqs_k8s_demo","log.module_path":"rusoto_sqs_k8s_demo","log.file":"src/main.rs","log.line":189}}
{"timestamp":"Aug 23 23:57:51.488","level":"INFO","target":"rusoto_sqs_k8s_demo","fields":{"message":"rusoto-sqs-k8s-demo 0.1.0 (4b04653, release build, linux [x86_64], Sun, 23 Aug 2020 19:12:11 +0000)","log.target":"rusoto_sqs_k8s_demo","log.module_path":"rusoto_sqs_k8s_demo","log.file":"src/main.rs","log.line":189}}
{"timestamp":"Aug 23 23:57:51.690","level":"INFO","target":"rusoto_sqs_k8s_demo","fields":{"message":"rusoto-sqs-k8s-demo 0.1.0 (4b04653, release build, linux [x86_64], Sun, 23 Aug 2020 19:12:11 +0000)","log.target":"rusoto_sqs_k8s_demo","log.module_path":"rusoto_sqs_k8s_demo","log.file":"src/main.rs","log.line":189}}
```

## Testing it out

To send a test message to your SQS queue:

```bash
aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/1234567890/rusoto-sqs-k8s-demo --message-body 'Hello world!'
```

In your application logs you should see entries similar to the following:

```json
{"timestamp":"Aug 23 23:58:09.780","level":"INFO","target":"rusoto_sqs_k8s_demo","fields":{"message":"Message { attributes: None, body: Some(\"Hello world!\"), md5_of_body: Some(\"86fb269d190d2c85f6e0468ceca42a20\"), md5_of_message_attributes: None, message_attributes: None, message_id: Some(\"d1ec1019-6398-4c75-b320-4a1e653e63ef\"), receipt_handle: Some(\"AQEBDrxJ...fnjddGjP8J6zvFKtw==\") }","log.target":"rusoto_sqs_k8s_demo","log.module_path":"rusoto_sqs_k8s_demo","log.file":"src/main.rs","log.line":129}}
```
