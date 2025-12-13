# AWS S3 & CloudFront End-to-End Setup Guide

This guide details the steps to configure an AWS S3 bucket for storage and serve content securely via CloudFront, including setting up developer access.

## 1. Create S3 Bucket
*   **Bucket Name**: `example-bucket` (Choose a unique name)
*   **Region**: Select your target region (e.g., `ap-south-1`)
*   **Block Public Access**: Enable "Block all public access" (Recommended for security; CloudFront OAC handles access)

## 2. Create IAM Policy for Developer Access
Create a policy to allow developers to manage files in the bucket.

**Policy Name**: `s3-bucket-access`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::example-bucket/*"
        }
    ]
}
```

## 3. Create IAM User
1.  Create a new IAM User.
2.  Attach the `s3-bucket-access` policy created in Step 2.
3.  Generate **Security Credentials** (Access Key and Secret Key) for this user.

## 4. CloudFront Setup

### Origin Configuration
*   **Origin Domain**: Select your S3 Bucket (`example-bucket.s3.amazonaws.com`)
*   **Origin Access**: Select **Origin access control settings (recommended)**.
*   **Origin Access Control**: Click **Create Control Setting** (default settings are usually fine).
    *   **Action**: CloudFront will provide a policy statement. **Copy** this policy.
    *   Go to your **S3 Bucket Permissions** -> **Bucket Policy**.
    *   **Paste** the policy (Example below):

```json
{
    "Version": "2008-10-17",
    "Id": "PolicyForCloudFrontPrivateContent",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::example-bucket/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::<Your-Account-ID>:distribution/<Your-Distribution-ID>"
                }
            }
        }
    ]
}
```

### Behavior Configuration
*   **Path Pattern**: `/images/*` (This limits CloudFront to only serve files in the /images/ folder. Use `*` to serve everything).
*   **Origin**: Select the S3 Origin created above.
*   **Viewer Protocol Policy**: Redirect HTTP to HTTPS (Recommended).
*   **Allowed HTTP Methods**: `GET, HEAD` (or more if needed).

Click **Create Behavior**.

## 5. Cache Invalidation Setup (Optional)
If developers need to clear the cache (e.g., after updating an image), they need permission to create invalidations.

**Policy Name**: `CDN-CreateInvalidation`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "cloudfront:CreateInvalidation",
            "Resource": "arn:aws:cloudfront::<Your-Account-ID>:distribution/<Your-Distribution-ID>"
        }
    ]
}
```

Attach this policy to the Developer IAM User.

## 6. Developer Handoff
Provide the following details to the developer:

*   **Access Key ID**: `<Access Key>`
*   **Secret Access Key**: `<Secret Access Key>`
*   **Region**: `ap-south-1` (or your specific bucket region)
*   **Bucket Name**: `example-bucket`
*   **CloudFront Distribution ID**: `<Distribution ID>` (for invalidations)

---

## Technical Considerations & Best Practices

1.  **CORS Configuration**: If you plan to load images via JavaScript (e.g., for a canvas or WebGL), you must configure **CORS (Cross-Origin Resource Sharing)** on the S3 bucket to allow requests from your website's domain.
2.  **Path Patterns**: The guide sets the path pattern to `/images/*`. Ensure your application actually stores images in a folder named `images/` inside the bucket. If files are at the root, change the path pattern to `*` or Default.
3.  **Public Access**: Ensure "Block Public Access" is enabled on the S3 bucket. The CloudFront Bucket Policy explicitly allows CloudFront to read the files, so the bucket itself does not need to be public.
4.  **SSL/TLS**: This setup uses the default CloudFront domain (`*.cloudfront.net`). If you want to use a custom domain (e.g., `assets.yourdomain.com`), you will need to add a CNAME in CloudFront and attach an SSL certificate via AWS ACM.
