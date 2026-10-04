# container-with-ecs-module-3.4
How I created Deploy-ecr.yml
To push a container image to a private AWS ECR repository using GitHub Actions,
using AWS OpenID Connect (OIDC) is recommended over long-lived IAM access keys because it avoids storing static AWS credentials in GitHub.
1.  1.Configure IAM OIDC Identity Provider and Role in AWS:In the AWS IAM Console,
add an Identity Provider with URL [https://token.actions.githubusercontent.com]
(https://token.actions.githubusercontent.com) and Audience sts.amazonaws.com.
     2.Create an IAM role for GitHub Actions with a trust policy scoped to your repository
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<ORG_OR_USER>/<REPO>:ref:refs/heads/main"
        }
      }
    }
  ]
}

    3.Attach the managed policy AmazonEC2ContainerRegistryPowerUser (or an equivalent least-privilege policy allowing
    ecr:GetAuthorizationToken, ecr:BatchCheckLayerAvailability, ecr:PutImage, etc.) to the role.
2. Add Secrets and Variables in GitHub:
In your GitHub repository,navigate to Settings > Secrets and variables > Actions:
Secrets:
AWS_ROLE_ARN: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
Variables (or Secrets):
AWS_REGION: e.g., us-east-1
ECR_REPOSITORY: e.g., my-private-app

3. Create the GitHub Actions Workflow File:
   Create .github/workflows/deploy-ecr.yml in your repository:

name: Build and Push Docker Image to Amazon ECR

on:
  push:
    branches: [ "main" ]

permissions:
  id-token: write   # Required for requesting the JWT from AWS OIDC
  contents: read    # Required for actions/checkout

jobs:
  deploy:
    name: Build & Push
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to Amazon ECR
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          REPOSITORY: ${{ vars.ECR_REPOSITORY }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG -t $REGISTRY/$REPOSITORY:latest .
          docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
          docker push $REGISTRY/$REPOSITORY:latest

Above code in ecr.yml file


4.  Verify the Push in AWS ECR:
Commit and push the workflow file to your main branch.
Check the execution under the Actions tab in GitHub.
Open the Amazon ECR console, navigate to Private repositories > <ECR_REPOSITORY>, and verify both the commit SHA tag and latest tag are listed.


