# The EKS IAM Service Account testing pod.

This pod can be run to try and diagnose permission issues with your pod.

Sometimes SDK output makes it hard to determine what is causing your issues with ServiceAccount 
IAM permissions.  This pod will provide some output to try and determine why you may be experiencing
issues.

This will not provide a definitive answer to permission issues but it should give you some insight.

## Requirements
kubectl 1.14 or above

## Running

Create a new folder.

In the folder put a kustomization.yaml file with the following contents, updating the namespace:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
# Update the namespace to the namespace you want to run in
namespace: test-iam
bases:
- github.com/jasonrichardsmith/eks-pod-identity-tester//kustomize
patches:
- patch.yaml

```

In the same folder put a patch.yaml file with your custom settings as described in the
comments:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
# Set serviceAccount if you are not using the default
  serviceAccountName: default
  containers:
  - name: test-iam
    env:
# AWS_DEFAULT_REGION is recommended
    - name: AWS_DEFAULT_REGION
      value: eu-west-1
# AWS_TEST_COMMAND is optional, set to check specific privileges, default is:
# aws sts assume-role --role-arn ${AWS_ROLE_ARN} --role-session-name test-session
    - name: AWS_TEST_COMMAND
      value: "aws s3api list-buckets"
# If using the web identity webhook, **DO NOT** include any of the below,
# everything will be set by the webhook.
# If you are not using the webhook everything below must be included.
# Set the role to your role arn.
    - name: AWS_ROLE_ARN
      value: arn:aws:iam::123456789012:role/mycustomrole
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    volumeMounts:
    - mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
      name: aws-iam-token
      readOnly: true
  volumes:
  - name: aws-iam-token
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400
          path: token
```

From within the folder run the pod:
```bash
kubectl apply -k .
```
Wait for the pod to start (with correct namespace):
```bash
kubectl get pods -n test-iam test-pod -w
```
Follow the logs (with correct namespace):
```bash
kubectl logs -n test-iam test-pod -f
```
After getting your output you can delete the pod:
```bash
kubectl delete -k .
```
