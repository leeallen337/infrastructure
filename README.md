# Infrastructure Guide

#### Table of Contents
  - [Deployment](#deployment)
    - [Front End](#front-end)
      - [FAQ](#front-end-faq)
    - [Back End](#back-end)
      - [FAQ](#back-end-faq)
    - [Full Rails](#full-rails)
      - [FAQ](#full-rails-faq)
  - [Custom Domains](#custom-domains)
    - [FAQ](#custom-domains-faq)
  - [SSL](#ssl)
    - [Viewer to CloudFront](#viewer-to-cloudfront)
    - [CloudFront to Origin](#cloudfront-to-origin)
    - [Custom Domain SSL](#custom-domain-ssl)
    - [FAQ](#ssl-faq)
## Deployment

This section is dedicated to setting up and administration around deployment.

### Front End

We deploy our static assest on Amazon Web Services (AWS) utilizing Simple Storage Service (S3). These files are served up using Amazon CloudFront (an Amazon CDN). This section will walk you through deploying your HTML, JavaScript, and CSS files to S3 and setting up CloudFront to serve the files.

The steps below will walk you through the entire process. The first few steps will need to be done only when initially setting up your computer.

**1. Amazon Web Services Command Line Interface (AWS CLI)**

Install AWS CLI. There are several ways to do this. You can go to the [official site](https://aws.amazon.com/cli/) and follow the instructions, but we recommend to install it through `brew` if you have `brew` installed. This is an [official distribution](https://github.com/Homebrew/homebrew-core/blob/master/Formula/awscli.rb) which should be in sync with the official release.
```
$ brew install awscli
```
Now check to make sure you have it installed. Your output may be slightly different.
```
$ aws --version
// aws-cli/1.14.10 Python/3.6.4 Darwin/16.7.0 botocore/1.8.14
```

**2. Security Credentials**

  1. Log into Smashing Boxes' AWS account. The credentials are in 1Password.
  2. Navigate to the Identity and Access Management (IAM) page. [See here](/deployment/frontend/security-credentials/menu-iam.png).
  3. Navigate to `Add user`. [See here](/deployment/frontend/security-credentials/add-user.png).
  4. On the `Details` page: [See here](/deployment/frontend/security-credentials/add-user-details.png).
      1. Enter your `User name` as `<firstname>.<lastname>`.
          - ex. `lee.allen`
          - This standard will make it easy to manage and see who has access and whose access needs to removed.
      2. Check `Programmatic access` for `Access type`.
      3. Click `Next`.
  5. On the `Permissions` page: [See here](/deployment/frontend/security-credentials/add-user-permissions.png).
      1. Check the `Deployment` group.
      2. Click `Next`.
  6. On the `Review` page: [See here](/deployment/frontend/security-credentials/add-user-review.png).
      1. Verify your choices.
      2. Click `Create User`.
  7. On the `Complete` page: [See here](/deployment/frontend/security-credentials/add-user-complete.png).
      - You can now see your `Access key ID` and `Secret access key`
      - **SAVE THESE**. You will need these to configure AWS CLI to be able to deploy to S3 from your command line.

**3. Configure AWS CLI**

Now we configure AWS CLI using your security crendentials generated in the previous step.
```
$ aws configure
AWS Access Key ID [None]: *ENTER YOUR ACCESS KEY ID*
AWS Secret Access Key [None]: *ENTER YOUR SECRET ACCESS KEY*
Default region name [None]: *you can leave this blank*
Default output format [None]: *you can leave this blank*
```
You can just press enter for the region name and output format and you should be done setting up your configuration.

Your configuration and credentials are stored in
```
# ~/.aws/config
# ~/.aws/credentials
```
and you can have multiple credentials. See the [FAQ](#front-end-faq) for more information.

**4. Create an S3 Bucket**

Now we'll create an S3 bucket for your project and your project's environments on AWS which will be used to store static files (e.g. HTML, JS, CSS).

  1. Navigate to the S3 page. [See here](/deployment/frontend/create-bucket/menu-s3.png).
  2. Click on `+ Create bucket`. [See here](/deployment/frontend/create-bucket/s3-bucket-create.png).
  3. On the `Name and region` section. [See here](/deployment/frontend/create-bucket/create-bucket-name-region.png).
      1. Enter in the project name in the format `app.<company-name>.<project-name>.<environment>.sb`.
          - ex. `app.diamler.plant-tour.qa.sb`
          - ex. `app.duke.reach.staging.sb`
          - ex. `app.diamler.quick-quote.qa.sb`
          - This standard will namespace all the projects and group them together so it will be easy to navigate between buckets and everyone knows what to expect.
          - S3 requires all bucket names to be unique across all of AWS.
      2. Click `Next`.
  4. On the `Set properties` section. [See here](/deployment/frontend/create-bucket/create-bucket-set-properties.png)
      1. You can set some custom settings here but it should be fine to leave these disabled.
      2. Click `Next`.
  5. On the `Set permissions` section. [See here](/deployment/frontend/create-bucket/create-bucket-set-permissions.png)
      1. It should be okay to leave these as the defaults.
      2. Click `Next`.
  5. On the `Review` section. [See here](/deployment/frontend/create-bucket/create-bucket-review.png)
      1. Review your choices.
      2. Click `Create bucket`.
  6. Verify that the bucket has been created. [See here](/deployment/frontend/create-bucket/bucket-list.png).
  7. Click into the bucket. You should be in the `Overview` section. [See here](/deployment/frontend/create-bucket/bucket-overview.png).
  8. Click on `Properties`. You should see a `Static web hosting` option. [See here](/deployment/frontend/create-bucket/bucket-properties.png).
      1. Click on `Disable`. This should open up options for you to select. [See here](/deployment/frontend/create-bucket/bucket-properties-static-web-hosting.png).
          1. Choose `Use this bucket to host a website`.
          2. Type `index.html` for the `Index document` option.
          3. Type `index.html` for the `Error document` option.
          4. Click `Save`.
      2. Verify that `Static web hosting` is no longer `Disabled`. [See here](/deployment/frontend/create-bucket/bucket-properties-after.png).
  9. As of right now, in Permissions > Bucket Policy. It's empty. [See here](/deployment/frontend/create-bucket/bucket-permissions-bucket-policy.png). This will change after setting up CloudFront.

**5. Deploy to S3**

In this section we'll also demonstrate how to deploy files from your command line to your S3 bucket. If you look into your `package.json` you should see some scripts similar to below. These will be explained through this section and the CloudFront section. The scripts will need to be updated with the relevant bucket and distrubtion ID information.
```
"deploy:qa": "npm run build:production && aws s3 sync dist/ s3://<S3 QA BUCKET NAME> --delete",
"deploy:qa:invalidate": "npm run deploy:qa && npm run deploy:invalidate",
"deploy:staging": "npm run build:production && aws s3 sync dist/ s3://<S3 STAGING BUCKET NAME> --delete",
"deploy:staging:invalidate": "npm run deploy:staging && npm run deploy:invalidate",
"invalidate:html": "aws cloudfront create-invalidation --distribution-id <CLOUDFRONT DISTRUBTION ID> --paths /index.html",
```
Following with our example from the previous screenshots, we'll replace
```
"deploy:qa": "npm run build:production && aws s3 sync dist/ s3://<S3 QA BUCKET NAME> --delete",
"deploy:staging": "npm run build:production && aws s3 sync dist/ s3://<S3 STAGING BUCKET NAME> --delete",
```
with (assuming the buckets are correctly set up in S3)
```
"deploy:staging": "npm run build:production && aws s3 sync dist/ s3://app.company-name.project-name.qa.sb --delete",
"deploy:staging": "npm run build:production && aws s3 sync dist/ s3://app.company-name.project-name.staging.sb --delete",
```
Now we run
```
npm run deploy:staging
```
Now you should be able to see the files in your bucket. [See here](/deployment/frontend/deploy-to-bucket/s3-deployed-files.png). This built whatever branch you were on, in this tutorial's case it was `master`, and synced it with S3.

**6. Set up CloudFront**

In this section we'll set up CloudFront.

  1. Navigate to the CloudFront page. [See here](/deployment/frontend/configure-cloudfront/menu-cloudfront.png).
  2. Click on `Create Distrubtion`. [See here](/deployment/frontend/configure-cloudfront/cloudfront-getting-started.png).
  3. Click on `Get Started` under `Web`. [See here](/deployment/frontend/configure-cloudfront/cloudfront-delivery-method.png).
  4. Under `Step 2: Create distrubtion`.
      1. When clicking into `Origin Domain Name` select your bucket name. [See here](/deployment/frontend/configure-cloudfront/cloudfront-create-distribution.png)
      2. Leave `Origin Path` blank.
      3. `Origin ID` will be filled in by your selection in `Origin Domain Name`.
      4. Select `Yes` on `Restrict Bucket Access`.
      5. For `Origin Access Identity` select `Create a New Identity`.
      6. For `Grant Read Permissions on Bucket` select `Yes, Update Bucket Policy`.
          - This configures the bucket to only be read from CloudFront. In setting up S3 and within the Web hosting option it shows an endpoint. If you navigate to that endpoint now you'll get a 403 Forbidden page. [See here](/deployment/frontend/configure-cloudfront/cloudfront-options-restriction.png).
      7. Your settings should look like [this](/deployment/frontend/configure-cloudfront/cloudfront-create-distribution-options.png).
      8. Set the `Default Root Object` to `index.html`. [See here](/deployment/frontend/configure-cloudfront/cloudfront-create-distribution-root.png).
      9. Click `Create Distribution`.
  5. Now under `Distributions` you should see your new distribution. [See here](/deployment/frontend/configure-cloudfront/cloudfront-distributions.png).
      - It will probably show as `In progress` for a while (e.g 10 - 20 mins).
  6. Now click on the distribution and navigate to the `Error Pages` tab.
      1. Click on `Create Custom Error Responses`. [See here](/deployment/frontend/configure-cloudfront/cloudfront-create-custom-error-responses.png).
      2. Fill out the information for 404 and 403 errors. [See here](/deployment/frontend/configure-cloudfront/cloudfront-custom-error-response-details.png).
      3. See the [final](/deployment/frontend/configure-cloudfront/cloudfront-error-response-finished.png) result.

Once CloudFront has finished deploying. [See here](/deployment/frontend/configure-cloudfront/cloudfront-deployed.png). You should be able to open up the domain in your browser (e.g. in this case `d2yy4dksv8ny1g.cloudfront.net`) and be able to see your application!

Now if you go back to S3 and look at the Bucket Policy you'll see it's been updated. [See here](/deployment/frontend/configure-cloudfront/s3-bucket-policy.png).

**7. Invalidate Cache**

One of the reasons for using CloudFront is to provide enhanced caching. However, this provides one problem with our current React based single-page applications. CloudFront will cache the `index.html` which could be problematic when deploying the latest and greatest since a cached version of `index.html` might have stale links to our static files. To combat this we can invalidate the CloudFront cache of `index.html` file with a script. In `package.json` there should be a script similar to:
```
"invalidate:html": "aws cloudfront create-invalidation --distribution-id <CLOUDFRONT DISTRUBTION ID> --paths /index.html",
```
We'll update this to be 
```
"invalidate:html": "aws cloudfront create-invalidation --distribution-id E3HT5LU0DX7BXS --paths /index.html",
```
The distribution ID we can find on distribution page. [See here](/deployment/frontend/configure-cloudfront/cloudfront-distribution-id.png). Now if we run:
```
npm run deploy:staging:invalidate
```
We will deploy to the staging bucket in addition to invalidating the cache of `index.html`. After the initial build and deploy to the bucket. This is probably the way to go for every subsequent deployment after the initial one.

### Front End FAQ

#### I didn't save my security credentials when I made my user in IAM. What do I do?
You can delete your user and create a new one.

#### I deployed a new build to my bucket and my web application doesn't appear to have updated. What's wrong?
Did you remember to invalidate the cache of `index.html` in CloudFront?

#### What's with that ID that starts with an E?
The Access Identity number (it looks similar to the CloudFront distribution ID number) is what allows CloudFront to access the private S3. See the S3 Bucket Policy [here](/deployment/frontend/faq/s3-access-identity.png) and the CloudFront reconciliation [here](/deployment/frontend/faq/cloudfront-access-identity/png).

#### Why do I need to join the `Deployment` group in AWS IAM?
Joining the group will make it easier to see who has deployment access and credentials and easier to manage users. I have also set the group to have a Policy under the Permissions tab to give those within the group access to S3 which will allow you to deploy to an S3 bucket through your command line using your security credentials. [See here](/deployment/frontend/faq/aws-group-permissions.png).

#### How do I delete an Origin Access Identity after it's no longer needed?
From the CloudFront main page, you can navigate to the `Origin Access Identity` section from the left-hand navigation, select the identity you wish to delete, and then delete it. [See here](/deployment/frontend/faq/origin-access-identity-delete.png).

#### Where are some additional resources that I can read?
- [AWS CloudFront - Working with Web Distributions](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web.html)
- [AWS CloudFront - Invalidating Objects (Web Distributions Only)](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html)
- [AWS CloudFront - Serving Private Content Through CloudFront](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html)
- [AWS CloudFront - How CloudFront Delivers Content](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html)
- [AWS S3 - Working with Amazon S3 Buckets](http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html)
- [AWS S3 - Hosting a Static Website on Amazong S3](http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html)
- [AWS CLI](https://aws.amazon.com/cli/)
- [AWS CLI CloudFront](http://docs.aws.amazon.com/cli/latest/reference/cloudfront/index.html#cli-aws-cloudfront)
- [AWS CLI - How to Use a Single IAM User to Easily Access All Your Accounts by Using the AWS CLI](https://aws.amazon.com/blogs/security/how-to-use-a-single-iam-user-to-easily-access-all-your-accounts-by-using-the-aws-cli/)


### Back End

This section will walk you through setting up deployment for back-end repos.

TODO: Update this section

### Full Rails

This section will walk you through setting up deployment for a full-stack rails repos.

TODO: Update this section

## Custom Domains

This section is dedicated to setting up a custom domain to route to CloudFront. When setting up a distribution in CloudFront you're given a domain (i.e. `ab3df938cd96kemw.cloudfront.net`) to access the distribution. We, and our clients, would prefer the web application to have a better domain name that takes into account the company and/or project name (i.e. `stilio.com`).

TODO: Update this section

## SSL

This section is dedicated to setting up SSL and enforcing HTTPS.

Currently our CloudFront domain accepts HTTP and HTTPS. We should set up SSL and enforce HTTPS access from the view to CloudFront and from CloudFront to our origin (e.g. our S3 bucket).

### Viewer to CloudFront

We should require our viewers to use HTTPS when navigating to our CloudFront domain (e.g. foobar123.domain).

1. Click into the distribution you want to edit.
2. Click on the `Behaviors` tab, select the behavior, and then click on `Edit`. [See here](/ssl/cloudfront-behaviors.png).
3. Under `Viewer Protocol Policy` select `Redirect HTTP to HTTPS`. [See here](/ssl/ssl-redirect.png).
4. Now after it has finished deploying, navigate to the CloudFront domain and verify that even when navigating to the HTTP URL you get redirected to HTTPS.

### CloudFront to Origin

From the documentation on AWS's website:
```
When your origin is an Amazon S3 bucket, CloudFront always forwards requests to S3 by using the protocol that viewers used to submit the requests. The default setting for the Origin Protocol Policy (Amazon EC2, Elastic Load Balancing, and Other Custom Origins Only) setting is Match Viewer and can't be changed.

If you want to require HTTPS for communication between CloudFront and Amazon S3, you must change the value of Viewer Protocol Policy to Redirect HTTP to HTTPS or HTTPS Only. The procedure later in this section explains how to use the CloudFront console to change Viewer Protocol Policy.
```

Therefore, the setting change we made in setting up the HTTP redirect to HTTPS in the `View to CloudFront` section will also apply to setting up HTTPS requests for `CloudFront to Origin`.

After these changes have been updated and deployed you can verify these changes by attempting to navigate to `http://<CloudFront Domain>`

[Before Redirect](/ssl/ssl-browser-before-redirect.png)

[After Redirect](/ssl/ssl-browser-after-redirect.png)

### Custom Domain SSL

TODO: UPDATE THIS SECTION

### SSL FAQ

#### Where are some additional resources that I can read?
- [Using HTTPS with CloudFront](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https.html)
- [Requiring HTTPS for Communication Between Viewers and CloudFront](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-viewers-to-cloudfront.html)
- [Requiring HTTPS for Communication Between CloudFront and Your Amazon S3 Origin](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-cloudfront-to-s3-origin.html)