# Deploying-a-Static-Website-to-AWS-S3-Using-CloudFormation-and-the-AWS-CLI

# Deploying a Static Website to AWS S3 Using CloudFormation and the AWS CLI

This repository details the configuration files and deployment steps required to provision a production-ready, highly secure public static website on AWS S3 using native **AWS CloudFormation** blueprints.

By leveraging the **AWS CLI**, the infrastructure stack creation and file synchronization cycles are managed entirely from a local developer environment inside **Visual Studio Code**, establishing a verified architectural baseline before introducing automated CI/CD pipeline triggers.

> 💡 **Substitution Guide:** Throughout this README, anything written in `<angle-brackets>` is something **you must replace** with your own values before running the commands.

---

## ⚠️ Important Architecture Note

The `BucketPolicy` (public read access) is intentionally **excluded** from `template.yaml`. AWS enforces an account-level `AWS::EarlyValidation::PropertyValidation` hook that blocks CloudFormation from deploying any bucket policy with `Principal: '*'`. As a result, the public read policy is applied separately via the AWS CLI after the stack is created (Step 3 below).

---

## 📂 Project Directory Structure
Create a dedicated workspace folder on your machine and arrange your project assets to match this structure precisely:

> 🚨 SUBSTITUTE: Replace `<your-project-folder>` with your own project folder name

```text
<your-project-folder>/
├── template.yaml              # Infrastructure-as-Code Blueprint
├── index.html                 # Main Landing Page Document
├── error.html                 # Custom 404 Routing Fallback Page
└── styles.css                 # Core Layout Design Sheet
```

---

## 📝 Code Components

### 1. The CloudFormation Blueprint (`template.yaml`)
This configuration layout file declares the underlying cloud hardware to be provisioned. It specifies public accessibility flags, disables legacy Access Control Lists (ACLs), and configures the bucket as a static website hosting engine. The public read bucket policy is applied separately via CLI due to AWS account-level validation restrictions.

### 2. The Main Page Asset (`index.html`)

### 3. The Fallback Page Asset (`error.html`)

### 4. The Style Sheet Asset (`styles.css`)

---

## 🚀 Execution Steps from VS Code Terminal

Open the built-in terminal window in Visual Studio Code (**Ctrl + `** or **Cmd + `**) and run the following deployment routines:

### Step 1: Verify AWS Terminal Access
Ensure your command-line interface is authenticated securely with your personal IAM configuration profile credentials:
```bash
aws sts get-caller-identity
```
> ✅ You should see your `Account`, `UserId` and `Arn` printed in the terminal. If you get an error, run `aws configure` to set up your credentials first.

---

### Step 2: Deploy the Infrastructure Stack
Provisions the S3 bucket, disables public access blocks, and configures static website hosting.

> 🚨 SUBSTITUTE: Replace `<your-stack-name>` with your preferred stack name e.g. `MyWebsiteStack`
> 🚨 SUBSTITUTE: Replace `<your-bucket-name>` with your globally unique S3 bucket name e.g. `my-portfolio-site-123456789012` (must be lowercase, no spaces)

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name <your-stack-name> \
  --parameter-overrides BucketName=<your-bucket-name>
```
> ✅ Wait until the terminal outputs `Successfully created/updated stack - <your-stack-name>`

---

### Step 3: Apply the Public Read Bucket Policy via CLI
Since AWS account-level validation blocks CloudFormation from deploying public bucket policies, apply it directly using the CLI.

> 🚨 SUBSTITUTE: Replace `<your-bucket-name>` with your S3 bucket name in **both** places inside the command below

```bash
aws s3api put-bucket-policy \
  --bucket <your-bucket-name> \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "PublicReadGetObject",
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::<your-bucket-name>/*"
      }
    ]
  }'
```
> ✅ No output means the policy was applied successfully.

---

### Step 4: Synchronize Code Files to S3
Pushes your front-end assets (`index.html`, `error.html`, `styles.css`, images) into the active cloud bucket while ignoring local infrastructure scripts.

> 🚨 SUBSTITUTE: Replace `<your-bucket-name>` with your S3 bucket name

```bash
aws s3 sync . s3://<your-bucket-name> \
  --delete \
  --exclude ".git*" \
  --exclude "template.yaml" \
  --exclude "*.md"
```
> ✅ You should see each file being uploaded listed in the terminal output.

---

### Step 5: Extract and Inspect the Public Website Endpoint
Queries the stack output parameters to retrieve your live website URL.

> 🚨 SUBSTITUTE: Replace `<your-stack-name>` with your stack name

```bash
aws cloudformation describe-stacks \
  --stack-name <your-stack-name> \
  --query "Stacks[0].Outputs[?OutputKey=='WebsiteURL'].OutputValue" \
  --output text
```
> ✅ Copy the generated URL, paste it into your browser, and your static website should be live!

---

## 🛑 Infrastructure Clean-Up Sequence
When you want to stop serving the website publicly and tear down all allocated AWS resources to prevent extra billing, run these commands:

> 🚨 SUBSTITUTE: Replace `<your-bucket-name>`, `<your-region>` and `<your-stack-name>` with your actual values
> - `<your-region>` example: `us-east-1`, `eu-west-1`, `sa-east-1`

```bash
# 1. Erase all stored file objects inside the S3 bucket container
aws s3 rm s3://<your-bucket-name> --recursive

# 2. Delete the S3 bucket
aws s3api delete-bucket \
  --bucket <your-bucket-name> \
  --region <your-region>

# 3. Dissolve the CloudFormation infrastructure stack
aws cloudformation delete-stack --stack-name <your-stack-name>
```
