## Tạo s3 bucket
### Bước 1:
- Bật Static website hosting (trong console S3 → Properties → Static website hosting):

- Hosting type: “Host a static website”

- Index document: index.html
### Bước 2:
Permissions → Bucket policy:
```bash
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicReadGetObject",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::khoi-web/*"
            }
        ]
    }
```

## Tạo codecommit
- name: masterwork
- Lấy HTTPS clone URL:
 https://git-codecommit.ap-southeast-2.amazonaws.com/v1/repos/masterwork
## Tạo codepipeline
Source: 
- AWS CodeCommit
- Repository name: masterwork
- Branch name: main
- Output artifact format: CodePipeline default
  
- Deploy provider: Amazon S3

    - Bucket: khoi-web

    - “Extract file before deploy”: Enabled (giải nén artifact)

- skip build và test

## Cấu hình tự động đồng bộ từ GitHub sang CodeCommit mỗi khi push
###   Bước 1: Tạo IAM User cho GitHub Actions
    ```bash
    khoi-github-actions-codecommit

    có policy/AWSCodeCommitPowerUser

    Lưu lại AccessKeyId và SecretAccessKey.
    ```
###   Bước 2: Thêm Secrets vào GitHub Repository
```bash
        AWS_ACCESS_KEY_ID: Access Key từ bước 1
        AWS_SECRET_ACCESS_KEY: Secret Key từ bước 1
```
###   Bước 3: Tạo GitHub Actions Workflow
