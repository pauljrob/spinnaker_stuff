# Install Spinnaker Easily from OSX

### Prepare OSX Workstation Environment

To simplify EKS cluster creation, one can use [eksctl](https://www.eksctl.io) and have a cluster created in a few steps.

***Install JSON Parser for OSX***

`brew install jq`

***Install eksctl for OSX***

`brew install weaveworks/tap/eksctl`

***Install kubectl***

```
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/darwin/amd64/kubectl

chmod +x ./kubectl

cp kubectl /usr/local/bin/

```

If necessary, add "/usr/local/bin" to your $PATH.


***Install aws-iam-authenticator***

`https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/darwin/amd64/aws-iam-authenticator

chmod +x ./aws-iam-authenticator

echo 'export PATH=$HOME/bin:$PATH' >> ~/.bash_profile`

***Create a Cluster with EKSCTL***

`eksctl create cluster --name=spinnaker-fu --nodes=3 --node-ami=auto --region=us-west-2`

This will take 15-20 minutes and you can monitor progress in the AWS EKS Console.

***Install Halyard***

```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/macos/InstallHalyard.sh
sudo bash InstallHalyard.sh
```

***Install kubectl***

`brew install kubernetes-cli`

---

# Install spinnaker

For this initial install we will be using the K8S V2 (Manifest) provider. Start by enabling the K8S provider.

```
hal config provider kubernetes enable

```

Next, create a K8S [service account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) for the newly created Spinnaker cluster.

```

CONTEXT=$(kubectl config current-context)

# This service account uses the ClusterAdmin role -- this is not necessary,
# more restrictive roles can by applied.
kubectl apply --context $CONTEXT \
    -f https://spinnaker.io/downloads/kubernetes/service-account.yml

TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user

```

Replace $GENERIC_ACCOUNT_NAME with a name of your choice. This is not tied to anything AWS specific but rather a logical name of your choosing. The account name will be referenced throughout the rest of the configuration.

```

hal config features edit --artifacts true

hal config deploy edit --type distributed --account-name $GENERIC_ACCOUNT_NAME

```

In the AWS IAM Console, create a user with privileges to create an AWS bucket and provide the region your bucket is in. Capture the Access Key ID and write down your Secret Access Key.

```

hal config storage s3 edit \
    --access-key-id $YOUR_ACCESS_KEY \
    --secret-access-key \
    --region us-west-2

```

In the AWS IAM Console, create a user with privileges to an AWS bucket. Capture the Access Key ID and write down your Secret Access Key.

```
hal config storage edit --type s3

```

Choose the version of Spinnaker you'd like to install from [https://www.spinnaker.io/community/releases/versions/](https://www.spinnaker.io/community/releases/versions/)

```
hal config version edit --version 1.12.1

hal deploy apply
```

Once you have run this command you can view your Spinnaker K8S pods running in the Spinnaker namespace.

`kubectl get pods -n spinnaker`

You can view specific POD's logs by running the following

`kubectl logs $POD_NAME -n spinnaker -f`

Once you see that your pods all have a status of "running" you can connect to your cluster.

`hal deploy connect`

This starts a proxy on your machine and then you can access Spinnaker at [http://localhost:9002](http://localhost:9002).

## Configuring the Lambda clouddriver

***Overriding the Default Clouddriver Halyard Config***

To override the default clouddriver configuration in Halyard you must create a file on your local workstation (or wherever you are running Halyard) and place it in ~/.hal/default/profiles/clouddriver-local.yml (create a new file if you have to).


`vi /Users/rbpa/.hal/default/profiles/clouddriver-local.yml`

Add the following into your profiles file. Replace "test-lambda" with a generic name of your choice different from what you chose above. Replace the account ID with the AWS account ID that your K8S cluster is running in.

```
aws:
  lambda:
    enabled: true
  accounts:
    - name: test-lambda
      lambdaEnabled: true
      accountId: '123456789101112'
      assumeRole: role/spinnakerManaged

```

Run the following Cloudformation template. Make sure you capture the EC2 instance profile ARN from your EKS cluster worker nodes. Use the instance profile role ARN for your $AUTH_ARN parameter. Change the $MANAGING_ACCOUNT_ID to the AWS Account ID you used above.

```
curl -O https://d3079gxvs8ayeg.cloudfront.net/templates/managed.yaml  

aws cloudformation deploy --stack-name spinnaker-managed-infrastructure-setup --template-file managed.yaml \
--parameter-overrides AuthArn=$AUTH_ARN ManagingAccountId=$MANAGING_ACCOUNT_ID --capabilities CAPABILITY_NAMED_IAM

```

Once the CloudFormation template has run successfully, it is time to re-deploy Spinnaker. Please note that your prior configurations will have been saved.

`hal deploy apply`

Once you have run this command you can view your Spinnaker K8S pods running in the Spinnaker namespace.

`kubectl get pods -n spinnaker`

You can view specific POD's logs by running the following command. It is recommended to watch your clouddriver POD start up to validate it is successful.

`kubectl logs $POD_NAME -n spinnaker -f`

Once you see that your pods all have a status of "running" you can connect to your cluster.

`hal deploy connect --service-names clouddriver,deck`

Now try running some REST API calls to see your existing Lambda functions

`curl -X GET --header 'Accept: application/json' 'http://localhost:7002/functions'`

The response should look something like the following.

```

{
  "account" : "test-lambda",
  "codeSha256" : "WaP97e/0DPjkeNFnBuR5hmjlJgK9gZL0mtNuB9g+NRw=",
  "codeSize" : 1240,
  "description" : "",
  "eventSourceMappings" : [ ],
  "functionArn" : "arn:aws:lambda:us-west-2:1234568791011:function:my_lambda_function",
  "functionName" : "my_lambda_function",
  "handler" : "lambda_function.lambda_handler",
  "lastModified" : "2017-05-10T19:48:36.818+0000",
  "layers" : [ ],
  "memorySize" : 128,
  "region" : "us-west-2",
  "revisionId" : "b62055cb-7435-405a-ba8c-f12b5834d2fa",
  "revisions" : {
    "b62055cb-7435-405a-ba8c-f12b5834d2fa" : "$LATEST"
  },
  "role" : "arn:aws:iam::1234567891011:role/my_lambda_role",
  "runtime" : "python2.7",
  "timeout" : 120,
  "tracingConfig" : {
    "mode" : "PassThrough"
  },
  "version" : "$LATEST",
  "vpcConfig" : {
    "securityGroupIds" : [ ],
    "subnetIds" : [ ],
    "vpcId" : ""
  }
}

```

## Building Custom Containers

See more details [https://medium.com/@edwin.a.avalos/updating-spinnaker-halyard-releases-with-custom-containers-373494a532b9](https://medium.com/@edwin.a.avalos/updating-spinnaker-halyard-releases-with-custom-containers-373494a532b9)

## Troubleshooting

Common issues when starting a Clouddriver POD are often due to permissions.

```
Factory method 'synchronizeAwsProvider' threw exception; nested exception is com.amazonaws.services.securitytoken.model.AWSSecurityTokenServiceException: Access denied (Service: AWSSecurityTokenService; Status Code: 403; Error Code: AccessDenied; Request ID: 6b6bb2d2-4430-11e9-9a57-abc1d6c09386)

```

If you the "AccessDenied" error above, enable CloudTrail and look in CloudWatch Logs and tweak your IAM credentials as needed.
