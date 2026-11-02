**Hosting a Static Website on AWS with S3 and CloudFront**

A practical guide to deploying a static website using Amazon S3 for file storage and Amazon CloudFront for HTTPS delivery and caching.

This guide uses a private S3 bucket and CloudFront Origin Access Control (OAC). Visitors can view the website through CloudFront, while direct public access to the bucket remains blocked.

**Architecture**

```
Visitor's browser
       |
       | HTTPS
       v
Amazon CloudFront
       |
       | HTTPS + signed origin requests (OAC)
       v
Private Amazon S3 bucket
       |
       +-- index.html
       +-- styles.css
       +-- script.js
       +-- assets/
```

- Amazon S3 stores HTML, CSS, JavaScript, images, and other static files.
- CloudFront is a content delivery network (CDN). It caches content at edge locations and retrieves files from S3 when needed.
- OAC lets CloudFront authenticate requests to S3. A bucket policy limits read access to your distribution.
- CloudFront provides an HTTPS URL for visitors without requiring a custom domain.

This setup serves static files; it does not run server-side application code or databases. Applications that need server-side functionality require a separate backend.

**Important: S3 Origin vs. S3 Website Endpoint**

Use the regular S3 bucket origin, not its static website hosting endpoint.

For this architecture:

- Keep S3 Block Public Access enabled.
- Leave S3 static website hosting disabled.
- Use CloudFront to handle the public website URL and HTTPS.
- Enable OAC and select the option to always sign origin requests.

S3 website endpoints are a different architecture: they do not support OAC or HTTPS between CloudFront and the website endpoint. Do not mix those instructions with this guide.

**Prerequisites**

- An AWS account with permission to manage S3 and CloudFront.
- A static website or a generated production build.
- An index.html file at the root of the files you deploy.
- Optional: AWS CLI installed and authenticated for command-line uploads.

Do not commit AWS credentials, secrets, or private environment files to GitHub or upload them with your website. Browser-delivered JavaScript and configuration are visible to visitors.

**Example Website Structure**

```
website/
├── index.html
├── styles.css
├── script.js
└── assets/
    └── logo.png
```

For a framework-based project, deploy the generated output directory, such as dist/ or build/, rather than the source repository. The correct directory depends on your framework and build configuration.

**1\. Create an S3 Bucket**

1. Open Amazon S3 in the AWS Management Console.
2. Create a general-purpose bucket with a globally unique name, such as my-static-site-unique-name.
3. Select an appropriate AWS Region.
4. Keep all Block Public Access settings enabled.
5. Keep Object Ownership set to Bucket owner enforced, with ACLs disabled.
6. Leave static website hosting disabled.

You do not need to make the bucket public or give it the same name as your domain.

**2\. Upload the Website Files**

Upload the contents of your website's output directory to the bucket root. Upload index.html itself, not a parent directory containing it.

Using the AWS CLI:

```
aws s3 sync ./website/ s3://YOUR_BUCKET_NAME/
```

For a generated build:

```
aws s3 sync ./dist/ s3://YOUR_BUCKET_NAME/
```

Replace placeholder values before running commands. Check that files have appropriate content types, especially text/html for HTML, text/css for CSS, and a JavaScript MIME type for JavaScript files.

**3\. Create a CloudFront Distribution**

Open CloudFront and create a distribution with these settings. Console labels may vary.

| Setting                | Value                                           |
| ---------------------- | ----------------------------------------------- |
| Origin                 | Select your S3 bucket's regular bucket endpoint |
| Origin path            | Leave empty if files are at the bucket root     |
| Origin access          | Origin Access Control (OAC)                     |
| OAC signing behavior   | Sign requests / always sign                     |
| Viewer protocol policy | Redirect HTTP to HTTPS                          |
| Allowed HTTP methods   | GET and HEAD for a basic static site            |
| Automatic compression  | Enable                                          |
| Default root object    | `index.html`                                    |

Create an OAC if you do not already have one, and associate it with the S3 origin.

Use index.html without a leading slash for the default root object. This makes a request to the distribution root return your homepage.

For a basic public static site, a managed caching policy such as CachingOptimized can be a starting point. Review its TTLs against your update strategy: a nonzero minimum TTL can keep content cached even when origin headers request otherwise.

**4\. Allow CloudFront to Read the Bucket**

After creating the distribution, update the S3 bucket policy. CloudFront may provide a policy you can copy from the console.

The following example grants read access to objects only for the specified CloudFront distribution:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

Replace:

- YOUR_BUCKET_NAME: your bucket name.
- YOUR_ACCOUNT_ID: your AWS account ID.
- YOUR_DISTRIBUTION_ID: your CloudFront distribution ID.

Apply the policy under the bucket's Permissions settings. Keep Block Public Access enabled; this policy authorizes a specific AWS service access path rather than anonymous public access.

This example assumes standard S3-managed encryption. If your objects use a customer-managed AWS KMS key, you must also configure the key policy to allow the required CloudFront access, as described in the OAC documentation.

**5\. Test the Website**

Wait for the distribution configuration to finish deploying, then open its assigned domain:

```
https://YOUR_DISTRIBUTION_DOMAIN.cloudfront.net
```

Verify that:

- The homepage loads at /.
- CSS, JavaScript, and images load correctly.
- HTTP requests redirect to HTTPS.
- Direct anonymous access to the S3 objects is denied.

The bucket is private, but the website is public through CloudFront. OAC does not authenticate website visitors. Do not deploy confidential files expecting OAC to hide them from visitors.

**6\. Update the Website**

Upload the updated files:

```
aws s3 sync ./website/ s3://YOUR_BUCKET_NAME/
```

If you want S3 to mirror your local directory exactly, you can add --delete. Review the destination carefully: that flag deletes destination objects that are absent from the local directory. Only use it with a bucket or prefix dedicated to this deployment.

CloudFront may continue serving cached files after an upload. To refresh cached paths:

```
aws cloudfront create-invalidation   --distribution-id YOUR_DISTRIBUTION_ID   --paths "/*"
```

For larger sites, invalidate only changed paths when practical. A common strategy is to use versioned or hashed filenames for assets and an appropriately short cache lifetime for HTML. Revisit the CloudFront cache policy as well as the files' cache headers when implementing this strategy.

Invalidation requests and other AWS usage may incur charges; check current pricing rather than assuming the deployment is free.

**Optional: Use a Custom Domain**

To use a domain such as <www.example.com>:

1. Request or import a certificate in AWS Certificate Manager in us-east-1 (US East, N. Virginia), the certificate Region required by CloudFront.
2. Complete domain validation for an ACM-issued certificate.
3. Add the domain to your CloudFront distribution's alternate domain names.
4. Attach a certificate that covers that domain.
5. Point your DNS record to CloudFront. With Route 53, use an appropriate alias record; other providers may use CNAME records for subdomains or an apex-compatible alias feature.
6. Wait for DNS and CloudFront changes to propagate, then test HTTPS.

A custom domain is optional. You can use the CloudFront-provided domain without configuring your own certificate.

**Routing and Single-Page Applications**

CloudFront's default root object handles /; it does not automatically map every directory path, such as /docs/, to /docs/index.html.

For a multi-page static site, use explicit file URLs or implement deliberate URL rewriting for directory-style routes.

For a single-page application with client-side routing, refreshing a route such as /dashboard may produce an S3 error because no object exists at that path.

Possible approaches:

- Use a CloudFront Function on viewer requests to rewrite application routes to /index.html, while excluding asset paths.
- Configure custom error responses for relevant S3 403 and 404 errors to return /index.html with HTTP 200.

Use error-response fallback carefully. It can hide genuine permission problems or return HTML for a missing CSS or JavaScript file. Test deep links and missing asset requests separately, and account for error-cache TTLs.

**Troubleshooting**

| Problem                                | Checks                                                                                                                                                                                                                      |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 403 Access Denied through CloudFront   | Confirm the bucket policy has the correct distribution ARN, OAC is attached, requests are signed, and the requested object exists. Missing objects can also return 403 when the caller lacks permission to list the bucket. |
| Homepage fails at /                    | Check that index.html exists at the origin root and the default root object is index.html.                                                                                                                                  |
| HTML loads but styling or scripts fail | Check asset paths, filename case, uploaded files, content types, and any routing fallback.                                                                                                                                  |
| Old content still appears              | Check CloudFront and browser caching; invalidate affected paths or use versioned asset filenames.                                                                                                                           |
| Direct S3 URL returns Access Denied    | This is expected for anonymous requests in this private-bucket architecture.                                                                                                                                                |
| Custom domain has a certificate error  | Check certificate domain coverage, validation, attachment to CloudFront, and the ACM Region.                                                                                                                                |

**Security and Cost Notes**

- Keep the bucket private and grant CloudFront only the access it needs.
- Give deployment identities least-privilege permissions; avoid using root-account credentials.
- Consider a CloudFront response headers policy for suitable browser security headers. Tailor any Content Security Policy to your application.
- Monitor usage and configure AWS Budgets alerts. Alerts are notifications, not a spending cap.
- Costs depend on S3 storage and requests, CloudFront delivery and requests, invalidations, optional services, and the pricing plan you select. Consult current AWS pricing.

**Clean Up**

If this is a temporary project:

1. Remove custom-domain DNS records that point to this distribution, if applicable.
2. Disable the CloudFront distribution and wait for the change to deploy.
3. Delete the distribution.
4. Empty and delete the S3 bucket, including object versions and delete markers if versioning was enabled.
5. Delete unused OAC resources and any other dedicated resources you created.

Deleting the local project or GitHub repository does not remove AWS resources.

**AWS Documentation**

- [Restrict access to an S3 origin with OAC](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Secure static website architecture](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/getting-started-secure-static-website-cloudformation-template.html)
- [CloudFront default root objects](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html)
- [CloudFront origin settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesOrigin.html)