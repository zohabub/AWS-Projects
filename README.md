**Static Website Hosting on AWS with Amazon S3 and CloudFront**

**Overview**

This project hosts a static website using a public Amazon S3 bucket and an Amazon CloudFront distribution. S3 stores the website files, while CloudFront delivers them through a content delivery network (CDN).

The implementation uses a public bucket rather than a private bucket secured with Origin Access Control (OAC). Only content intended for public access should be uploaded.

**Architecture**

```
Visitor
   |
   v
Amazon CloudFront
   |
   v
Public Amazon S3 bucket
   |
   +-- HTML files
   +-- CSS and JavaScript
   +-- Images and other static assets
```

When a visitor requests a page, CloudFront can serve a cached copy. When it needs to retrieve content from the origin, it requests the file from S3 and caches it according to the distribution's cache settings and the object's caching headers.

**AWS Services**

| Service           | Purpose                                                                              |
| ----------------- | ------------------------------------------------------------------------------------ |
| Amazon S3         | Stores the website's static files and makes the intended objects publicly readable.  |
| Amazon CloudFront | Delivers content through distributed edge locations and supports HTTPS for visitors. |

This setup serves static content; it does not execute server-side application code.

**Project Details**

Replace the placeholders below with your actual deployment details. Do not include AWS credentials.

| Setting                    | Value                                               |
| -------------------------- | --------------------------------------------------- |
| S3 bucket name             | `<YOUR_BUCKET_NAME>`                                |
| AWS Region                 | `<YOUR_AWS_REGION>`                                 |
| CloudFront distribution ID | `<YOUR_DISTRIBUTION_ID>`                            |
| Website URL                | `https://<YOUR_DISTRIBUTION_DOMAIN>.cloudfront.net` |
| CloudFront origin type     | `<S3_REST_ENDPOINT_OR_S3_WEBSITE_ENDPOINT>`         |

**Deployment Guide**

The following steps describe how to reproduce a public-S3-and-CloudFront deployment. Match the origin configuration to the endpoint actually used in your distribution.

**1\. Create an S3 Bucket**

1. Open Amazon S3 in the AWS Management Console.
2. Create a bucket with a globally unique name in your chosen AWS Region.
3. Use this bucket only for files intended to be publicly accessible.

**2\. Upload the Website Files**

Upload your website files to the bucket, preserving the directory structure used by your HTML, CSS, and JavaScript.

Example structure:

```
website/
├── index.html
├── error.html       # Optional error page
├── css/
│   └── style.css
├── js/
│   └── script.js
└── images/
```

Upload the contents of the website directory so that index.html is at the bucket root. If the site requires a build step, upload the generated build output rather than the source project.

**3\. Allow Public Read Access**

For this public-bucket implementation, configure the applicable bucket- and account-level S3 Block Public Access settings to permit the intended public bucket policy. Change only the settings required for this deployment; do not disable account-wide protections without reviewing the impact on other buckets.

The following example grants anonymous read access to all objects in the bucket. Replace &lt;YOUR_BUCKET_NAME&gt; before applying it:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadWebsiteObjects",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
    }
  ]
}
```

This policy permits object reads. It does not grant public upload, deletion, or bucket-listing permissions. Avoid public write access and do not upload credentials, private documents, or other sensitive data.

**4\. Choose the Correct Origin Endpoint**

S3 provides two different endpoint types. A public bucket can be used with either, but their behavior differs.

| Setting                     | S3 website endpoint                      | S3 REST endpoint                                     |
| --------------------------- | ---------------------------------------- | ---------------------------------------------------- |
| S3 static website hosting   | Enable it                                | Not required                                         |
| Origin configuration        | Custom origin using the website endpoint | S3 origin using the bucket endpoint                  |
| CloudFront-to-S3 connection | HTTP only                                | Supports HTTPS                                       |
| Index and error documents   | Uses S3 website behavior                 | Does not use S3 website index/error settings         |
| OAC support                 | Not supported                            | Supported, but not used by this public-bucket design |

If using the website endpoint:

1. Open the bucket's Properties tab.
2. Enable Static website hosting.
3. Set the index document to index.html.
4. Optionally set an error document, such as error.html, after uploading it.
5. Copy the website endpoint shown by S3 and use its hostname as the CloudFront origin domain. Enter it explicitly rather than selecting the regular S3 bucket endpoint.

If using the REST endpoint, select the bucket as the S3 origin and configure public origin access. Set CloudFront's default root object to index.html. S3 website hosting does not need to be enabled for this route.

**5\. Create the CloudFront Distribution**

1. Open Amazon CloudFront and create a distribution.
2. Configure the selected S3 endpoint as the origin.
3. For a public S3 REST origin, use public origin access rather than OAC.
4. Set the viewer protocol policy to Redirect HTTP to HTTPS.
5. Set the default root object to index.html, without a leading slash.
6. Choose cache settings appropriate for your website.
7. Use the default CloudFront certificate for the generated cloudfront.net hostname.
8. Save the distribution and wait for its configuration to deploy.

For a website endpoint origin, CloudFront must connect to S3 over HTTP. Visitors can still connect to CloudFront over HTTPS, but that does not make the CloudFront-to-S3 connection HTTPS.

**6\. Test the Website**

Open:

```
https://<YOUR_DISTRIBUTION_DOMAIN>.cloudfront.net
```

Verify that:

- The home page loads.
- CSS, JavaScript, images, and internal links work.
- HTTP requests redirect to HTTPS if that policy was configured.
- Missing pages behave as intended.
- Nested page URLs work when opened directly, not only when reached through links.

A CloudFront default root object handles the distribution root; it does not automatically turn every subdirectory request into a request for that directory's index.html. Plan routing explicitly, especially when using the S3 REST endpoint.

**Updating the Website**

Upload the updated files to the same bucket. CloudFront may continue serving cached versions until they expire or are invalidated.

If you use the AWS CLI, a deployment can look like this:

```
aws s3 sync ./website/ s3://<YOUR_BUCKET_NAME>/
```

Replace ./website/ with your actual static-site or build-output directory. This example does not delete older objects that are no longer present locally.

To invalidate cached paths after an update:

```
aws cloudfront create-invalidation   --distribution-id <YOUR_DISTRIBUTION_ID>   --paths "/*"
```

For routine deployments, versioned asset filenames and deliberate cache lifetimes can reduce the need for broad invalidations.

**Troubleshooting**

| Issue                           | What to check                                                                                                                       |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Access denied                   | Verify the bucket policy, effective Block Public Access settings, exact object key, and origin configuration.                       |
| Home page does not load         | Confirm that index.html exists at the expected location and the default root object is correct.                                     |
| Styles or images are missing    | Check file paths, filename capitalization, uploaded assets, and content types.                                                      |
| Old content is displayed        | Review CloudFront and browser caching; wait for expiration or invalidate the affected paths.                                        |
| Nested routes fail              | Check whether the application needs explicit rewrites or fallback handling. Do not assume a default root object handles all routes. |
| Website origin fails with HTTPS | S3 website endpoints support HTTP only; use the correct origin protocol.                                                            |

**Security Considerations**

The bucket is public, so readable objects can be accessed directly through S3 without passing through CloudFront. CloudFront does not make a public bucket private, and CloudFront-only restrictions do not prevent direct access to public S3 objects.

For a stronger production architecture, use a private S3 bucket with a regular S3 REST origin and CloudFront Origin Access Control. Restrict the bucket policy to the intended distribution and retain S3 Block Public Access. This is a different design from the public-bucket implementation described here; OAC cannot be attached to an S3 website endpoint.

**Costs and Cleanup**

S3 storage and requests, CloudFront delivery, and some invalidation usage can incur charges. Review current AWS pricing and monitor usage rather than assuming the deployment is free.

When the project is no longer needed, disable and delete the CloudFront distribution, then remove the website objects and delete the S3 bucket. If bucket versioning is enabled, remove retained versions and delete markers as needed.

**AWS Documentation**

- [Tutorial: Configuring a static website on Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/HostingWebsiteOnS3Setup.html)
- [S3 website endpoints and their differences from REST endpoints](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteEndpoints.html)
- [Using S3 origins with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html)
- [CloudFront origin settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesOrigin.html)
- [Restricting access to an S3 origin with OAC](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)