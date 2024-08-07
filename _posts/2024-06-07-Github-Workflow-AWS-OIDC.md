# Github Workflow with AWS OIDC

Are you storing long-lived credentials in your Github workflows, to access AWS
resources? Let it be through Terraform or AWS CLI?

If you answer is yes, then you should consider switching to using OpenID Connect.

With OpenID Connect (OIDC), your workflow is going to request a short-lived
access token directly from your cloud provider. Your cloud provider also has to
support OIDC and we will have to configure a trust relationship between our
workflows and the cloud provider. AWS is one of the providers which supports
OIDC.

The benefits of using OIDC:

 * No need to store long-lived credentials in Github secrets anymore
 * You have more granular control over how workflows can use credentials and which
   resources they can access
 * The access token is short-lived and automatically expires

If you want to learn more about OIDC, Github provides some good documentation:
[Github OIDC](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connec)

## Adding OIDC to your Github Workflow

1. Go to AWS Console -> IAM -> Identity Providers -> Create Provider
2. Select `OpenID Connect` and enter the following:
    - Provider URL: `token.actions.githubusercontent.com`
    - Audience: `sts.amazonaws.com`
3. Go to AWS Console -> IAM -> Roles -> Create Role and use the following as
   trust relationship:
   ```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<aws-account-number>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:sub": [
                        "repo:<GITHUB ORG>/<GITHUB REPO>:<SUBJECT>",
                    ]
                }
            }
        }
    ]
   }
   ```
   For information about the subject, see here: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims
4. If you use S3 as your Terraform backend, add a policy to your IAM role
   with `s3:GetObject, s3:PutObject, s3:ListBucket` permissions and the bucket
   as resource.
5. If you use DynamoDB as your lock table in Terraform, add a policy to your IAM
   role with `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:DeleteItem`
   permissions and the table as resource.
6. Now you will have to add the following step to your workflow file:
   ```yaml
   jobs:
     your_job_name:
        steps:
          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v4
            with:
              role-to-assume: ${{ secrets.AWS_ROLE }}
              role-session-name: OIDCSession
              aws-region: your-region
   ```
   And do not forget to add the `AWS_ROLE` secret to your repository.

And that should be it :) You are now using OIDC in your Github workflows.


### Extend role with permissions

In step 3, you created a role with the necessary trust relationship. You will
have to add additional policies with permissions to the role, depending on what
you want to do with the credentials. Let it be saving information in an S3 bucket,
or uploading a Docker image to ECR. All those permissions are added to that
particular role.
