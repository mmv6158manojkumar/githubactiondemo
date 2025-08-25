# Guide: Creating and Managing AWS S3 Buckets Using GitHub Actions

This document provides a step-by-step guide to:
- Setting up a GitHub Actions workflow
- Configuring AWS credentials using GitHub Secrets
- Hardcoding and passing variables to workflows
- Real-life YAML examples for creating and deleting S3 buckets

---

## 1. Introduction to GitHub Actions

**GitHub Actions** enables you to automate tasks within your software development lifecycle directly from your GitHub repository. Workflows are defined in `.github/workflows/` as YAML files.

---

## 2. Creating a Workflow Directory

**Steps:**
1. Navigate to your repository on GitHub.
2. Click **Add file** > **Create new file**.
3. Enter `.github/workflows/your-workflow.yml` as the file name.
4. Add some content (even a comment).
5. Commit the file.

This creates both the `.github` and `workflows` directories if they don’t exist.

---

## 3. Configuring AWS Credentials as GitHub Secrets

To securely use AWS credentials in workflows:

1. Go to your repository, click **Settings**.
2. In the sidebar, select **Secrets and variables > Actions**.
3. Click **New repository secret**.
4. Add the following secrets:
   - `AWS_ACCESS_KEY_ID`: Your AWS access key.
   - `AWS_SECRET_ACCESS_KEY`: Your AWS secret key.
   - (Optionally) `AWS_SESSION_TOKEN`: If using temporary credentials.

**Never commit credentials to your repository. Always use GitHub Secrets.**

---

## 4. Setting and Using Variables in Workflows

Variables can be:
- **Hardcoded** in workflow files (e.g., region or bucket name)
- **Provided as workflow inputs** (manually entered when running the workflow)

Example (hardcoded region for N. Virginia):
```yaml
aws-region: us-east-1
```
Example (input for region):
```yaml
inputs:
  region:
    description: 'AWS Region (e.g., us-east-1)'
    required: true
    type: string
```

---

## 5. Real-Time Workflow Examples

### 5.1 Create an S3 Bucket (with Hardcoded Bucket Name and Region)

```yaml name=.github/workflows/create-s3-bucket.yml
name: Create S3 Bucket

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  create-bucket:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Create S3 bucket
      run: |
        aws s3api create-bucket \
          --bucket "your-bucket-name-here" \
          --region "us-east-1"
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: us-east-1
```
**Note:** For `us-east-1`, do NOT include `--create-bucket-configuration`.

---

### 5.2 Create an S3 Bucket (with Manual Inputs)

```yaml name=.github/workflows/create-s3-bucket.yml
name: Create S3 Bucket

on:
  workflow_dispatch:
    inputs:
      bucket_name:
        description: 'S3 Bucket Name'
        required: true
        type: string
      region:
        description: 'AWS Region (e.g., us-east-1)'
        required: true
        type: string

jobs:
  create-bucket:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ github.event.inputs.region }}

    - name: Create S3 bucket
      run: |
        if [ "${{ github.event.inputs.region }}" = "us-east-1" ]; then
          aws s3api create-bucket \
            --bucket "${{ github.event.inputs.bucket_name }}" \
            --region "us-east-1"
        else
          aws s3api create-bucket \
            --bucket "${{ github.event.inputs.bucket_name }}" \
            --region "${{ github.event.inputs.region }}" \
            --create-bucket-configuration LocationConstraint="${{ github.event.inputs.region }}"
        fi
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: ${{ github.event.inputs.region }}
```

---

### 5.3 Delete Multiple S3 Buckets (Manual Input)

```yaml name=.github/workflows/delete-s3-buckets.yml
name: Delete S3 Buckets

on:
  workflow_dispatch:
    inputs:
      bucket_names:
        description: 'Comma-separated list of S3 Bucket Names to delete'
        required: true
        type: string
      region:
        description: 'AWS Region (e.g., us-east-1)'
        required: true
        type: string

jobs:
  delete-buckets:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ github.event.inputs.region }}

    - name: Delete all listed S3 buckets
      run: |
        IFS=',' read -ra BUCKETS <<< "${{ github.event.inputs.bucket_names }}"
        for BUCKET in "${BUCKETS[@]}"; do
          BUCKET_TRIMMED=$(echo "$BUCKET" | xargs)
          echo "Emptying $BUCKET_TRIMMED ..."
          aws s3 rm "s3://$BUCKET_TRIMMED" --recursive --region "${{ github.event.inputs.region }}"
          echo "Deleting $BUCKET_TRIMMED ..."
          aws s3api delete-bucket --bucket "$BUCKET_TRIMMED" --region "${{ github.event.inputs.region }}"
        done
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: ${{ github.event.inputs.region }}
```

---

## 6. Running the Workflow

- **Push Trigger:** Workflow runs automatically on pushes to the specified branch.
- **Manual Trigger:** Go to the Actions tab, select the workflow, click **Run workflow**, and provide required inputs.

---

## 7. Best Practices

- Always use GitHub Secrets for sensitive data.
- Never commit AWS credentials to your codebase.
- Use region-specific logic for AWS S3 bucket operations (`us-east-1` is a special case).
- Test workflows with non-critical buckets first.

---

## 8. Troubleshooting

- **InvalidLocationConstraint:** Do not specify `--create-bucket-configuration` for `us-east-1`.
- **Bucket Name Already Exists:** S3 bucket names must be globally unique.
- **Permissions:** Ensure the IAM user has the correct permissions (`s3:CreateBucket`, `s3:DeleteBucket`, `s3:ListBucket`, `s3:DeleteObject`).

---

## 9. References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS CLI S3 Documentation](https://docs.aws.amazon.com/cli/latest/reference/s3/index.html)
- [Best Practices for Managing AWS Credentials](https://docs.aws.amazon.com/general/latest/gr/aws-access-keys-best-practices.html)

---
