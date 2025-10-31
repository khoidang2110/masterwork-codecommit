# Triển khai web tĩnh GitHub → CodeCommit → CodePipeline → S3

## 0) Yêu cầu
- **Region** thống nhất: `ap-southeast-2` (Sydney)
- Repo GitHub sẵn sàng
- S3 bucket tên **unique** (vd: `khoi-web`)

---

## 1) Tạo S3 bucket (Static Website Hosting)

### Bước 1 — Bật website hosting
Vào **S3 → Bucket `khoi-web` → Properties → Static website hosting**
- Hosting type: `Host a static website`
- Index document: `index.html`
- (tuỳ) Error document: `404.html`

### Bước 2 — Cho phép đọc public
Vào **S3 → Bucket `khoi-web` → Permissions**
- *Block public access*: bỏ tick “Block all public access”
- *Bucket policy*:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::khoi-web/*"
  }]
}
```

> Truy cập qua website endpoint:  
> `http://khoi-web.s3-website-ap-southeast-2.amazonaws.com`

---

## 2) Tạo CodeCommit
- **Repository name:** `masterwork`
- **HTTPS clone URL:**  
  `https://git-codecommit.ap-southeast-2.amazonaws.com/v1/repos/masterwork`

---

## 3) Tạo CodePipeline (Source → Deploy)

### Source
- **Source provider:** `AWS CodeCommit`
- **Repository:** `masterwork`
- **Branch name:** `main` 
- **Output artifact format:** `CodePipeline default`
- ✅ *Start the pipeline on source code changes:* Enabled

### Build
- Skip (không cần cho web tĩnh)

### Deploy
- **Deploy provider:** `Amazon S3`
- **Bucket:** `khoi-web`
- **Extract file before deploy:** Enabled

> Sau khi tạo, bấm **Release change** để chạy thử.

---

## 4) Đồng bộ GitHub → CodeCommit (GitHub Actions)

### Bước 1 — Tạo IAM User
```bash
Tên user: khoi-github-actions-codecommit
Policy: AWSCodeCommitPowerUser
```
Lưu lại `AccessKeyId` và `SecretAccessKey`.

### Bước 2 — Thêm Secrets vào GitHub Repository
```bash
AWS_ACCESS_KEY_ID: <AccessKeyId>
AWS_SECRET_ACCESS_KEY: <SecretAccessKey>
```

### Bước 3 — Tạo workflow `.github/workflows/mirror-to-codecommit.yml`

```bash
name: Mirror to CodeCommit

on:
  push:
    branches: [ "main" ]  # đổi theo branch bạn dùng

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout GitHub
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # mirror cần full history/tags

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION || 'ap-southeast-2' }}

      - name: Configure CodeCommit credential helper
        run: |
          git config --global credential.helper '!aws codecommit credential-helper $@'
          git config --global credential.UseHttpPath true

      - name: Set CodeCommit remote URL (HTTPS)
        run: |
          CODECOMMIT_URL="https://git-codecommit.ap-southeast-2.amazonaws.com/v1/repos/masterwork"
          git remote add codecommit "$CODECOMMIT_URL" || git remote set-url codecommit "$CODECOMMIT_URL"
          git remote -v

      - name: Set Git identity
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Push to CodeCommit (mirror)
        run: |
          # Đẩy tất cả refs (branches, tags) – hoặc chỉ main: git push codecommit main:main
          git push --force  codecommit main:main
```

### Kick pipeline
Push commit mới lên GitHub → workflow sẽ mirror sang CodeCommit → CodePipeline chạy tự động.  
Hoặc bấm **Release change** trong CodePipeline.

### URL website:
`http://khoi-web.s3-website-ap-southeast-2.amazonaws.com`

