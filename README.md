# Static website on S3 and CloudFront

**Full YouTube tutorial:** https://youtu.be/mIiGXZNuuPc?si=0zx5H6sEBqWLaZRh

A one-page static site stored in a private Amazon S3 bucket and served over HTTPS by Amazon CloudFront. The bucket stays private. Visitors reach the site only through CloudFront.

## Files

```text
index.html
style.css
script.js
assets/
├── logo.svg
├── hero-image.webp
└── favicon.ico
```

Upload these files with the `assets/` folder intact. `index.html` is the default root object.

## 1. Create a private S3 bucket

1. In the AWS console, open **S3** and choose **Create bucket**.
2. Use a globally unique name, all lowercase.
3. Choose a Region close to you. CloudFront serves the site worldwide, so the Region is not what visitors connect to.
4. Leave **Block all public access** turned on.
5. Leave the other settings on their defaults and create the bucket.
6. Do not turn on **Static website hosting**. CloudFront serves the files.

## 2. Upload the site

1. Open the bucket and choose **Upload**.
2. Add `index.html`, `style.css`, `script.js`, and the `assets/` folder.
3. Upload.

The object keys must stay like this:

- `index.html`
- `style.css`
- `script.js`
- `assets/logo.svg`
- `assets/hero-image.webp`
- `assets/favicon.ico`

## 3. Create the CloudFront distribution

1. Open **CloudFront** and choose **Create distribution**.
2. If a pricing-plan screen appears, the Free plan is not available on accounts that are using the AWS Free Tier. Use **pay-as-you-go** in that case. A site this small stays close to free.
3. Set the origin type to **Amazon S3** and select the bucket.
4. Under origin access, choose **Origin access control settings (recommended)** and create a new control with the defaults.
5. Set **Viewer protocol policy** to **Redirect HTTP to HTTPS**.
6. Leave the recommended web application firewall setting on.
7. Set **Default root object** to `index.html`.
8. Create the distribution.

CloudFront then shows a banner that the bucket policy must be updated.

1. Choose **Copy policy**.
2. Open the bucket, go to **Permissions**, edit the **Bucket policy**, paste the policy, and save.

The policy allows `s3:GetObject` only when the request comes from this CloudFront distribution.

Wait until the distribution status is **Enabled** and **Deployed** before testing. A distribution that is still deploying can reset the connection instead of returning a page.

## 4. Test

1. Copy the distribution domain name (`d1234abcd.cloudfront.net`) and open it. The site should load over HTTPS.
2. Open the same `index.html` object by its direct S3 URL. It should return **Access Denied**.

If the CloudFront URL says the connection was reset, check that the distribution is **Deployed**. If it is deployed and the distribution is on the Free plan, cancel that plan and switch the distribution to pay-as-you-go, then wait until it is deployed again.

## 5. Request a certificate

CloudFront only uses a public certificate from **US East (N. Virginia)**, `us-east-1`.

1. Switch the console Region to **US East (N. Virginia)**.
2. Open **Certificate Manager** and request a public certificate.
3. Add your domain and the `www` name.
4. Choose **DNS validation** and request the certificate.
5. Add the CNAME record Certificate Manager shows.
   - In Route 53, choose **Create records in Route 53**.
   - At another registrar, create a CNAME with the name and value from the certificate.
6. Wait until the certificate status is **Issued**.

Some DNS panels append your domain to the record name. If validation does not finish, use only the part of the name that comes before your domain.

## 6. Attach your domain

1. Open the distribution, go to **Settings**, and choose **Edit**.
2. Under **Alternate domain name (CNAME)**, add your domain and the `www` name.
3. Under **Custom SSL certificate**, select the issued certificate.
4. Save. If the certificate is missing from the list, it is in the wrong Region or it is not issued yet.

Point DNS at CloudFront:

- **Route 53:** create an **A** record, turn on **Alias**, and choose the CloudFront distribution. Repeat for `www`.
- **Another registrar:** create a CNAME for `www` that points at the distribution domain name. For the root domain, use an ALIAS, ANAME, or flattened CNAME if the registrar supports it.

DNS can take a few minutes. Then open `https://your-domain`.

## 7. Fix common errors

**Access Denied (403).** The bucket policy must match this distribution, and the default root object must be `index.html`.

**A client-side app errors on refresh.** Under **Error pages**, create a custom error response for 403 and for 404. Set the response page path to `/index.html` and the HTTP response code to **200**. This one-page site does not need that.

**An update does not show.** Under **Invalidations**, create an invalidation for `/*`.

**The certificate is missing, or CloudFront says the account must be verified.** Request the certificate in `us-east-1` and wait until it is issued. If CloudFront says the account must be verified, open a case with AWS Support.

## Cost and cleanup

A few small files in S3 cost a fraction of a cent, and CloudFront’s free usage covers a small site. The domain is the usual yearly cost. Create a budget alert so a surprise charge does not go unnoticed.

To remove a practice setup:

1. If the distribution uses a CloudFront pricing plan, cancel that plan first. A Free plan cancels immediately. AWS will not delete the distribution until the plan is canceled.
2. Disable and delete the distribution.
3. Empty and delete the bucket.
4. Delete the certificate.
5. If you created a Route 53 hosted zone, delete it. A hosted zone has a monthly fee.
