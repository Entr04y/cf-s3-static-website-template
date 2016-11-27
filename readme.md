# Cloudformation template for static website hosting

## Overview

The AWS configurations in this template are inspired by this [post](https://lustforge.com/2016/02/27/hosting-hugo-on-aws/) by [Joe Lust](https://lustforge.com/)

This template will create two s3 buckets (one for logging and one for serving the website), a cloudfront distribution, an AWS ACM SSL certificate in the region that it is running, and update DNS for the specified domain name.

The resources in this template will not incur AWS charges in and of themselves.  There could be associated storage, bandwidth, and request charges based on the amount of content and traffic you get to your site, so keep an eye on your billing charges (preferably set a cloudwatch alarm so you get alerted if you start to spend more than you are willing to pay)

The cloudfront distribution will redirect http traffic on port 80 to https on port 443.

The template outputs and exports the values of the created buckets and ACM certificates.  

You can then upload your static website files to the website directory.  The template puts a redirect in place so that if your site generator behaves like [Hugo](https://gohugo.io) and creates a directory for each document with an index.html, S3 will serve it up.  You may need to modify those redirects if your site behaves differently.

## Requirements

* The DNS for the primary hostname is in an existing Route53 Hosted zone
* The DNS entry for the primary hostname cannot already exist in Route53
* You have access to a domain contact email for every domain that you include in the websiteHostedNames parameter to approve the ACM certificate.

## Usage

The template only requires two parameters.  The first (hostedZoneName) is the name of the Route53 hosted zone that you wish the DNS entry to be placed in.  The second (websiteHostNames) is a comma separated list of one or more hostnames to include as alternate domains in the ACM certificate and as valid CNAMES for the CloudFront distribution.

One of the first things that will happen is the creation of the certificate request.  The stack will not create until all hostnames have been approved for the certificate, so keep an eye on your email and don't wander off until that is complete.  The cloudfront distribution will take a significant amount of time to create (30 - 60 minutes), so go do something else while you wait.

The template will use the first hostname in the list provided in websiteHostNames to generate the DNS entry and as the primary hostname in the ACM certificate.  The dns entry will be a Route53 alias record pointed at the CloudFront distribution, as such, the hostname must be in the domain entered in the hostedZoneName parameter.  

For example, if the HostedZone is "example.com." the value for hostedZoneName would be "example.com" and the value for websiteHostNames could be "example.com,www.example.com,www.example.org,some.other.hostname.com".  The DNS entry created would be an alias record for the zone apex example.com pointing to the cloudfront distribution and the ACM certificate would be valid for example.com, www.example.com, www.example.org, and some.other.hostname.com.  The first hostname in the list does not need to be the apex of the domain.

I did not put in logic to create DNS entries for any but the primary hostname.  There are too many variations of hostname/domain combinations to handle in cloudformation natively, and since there is no iterator for a list, it would likely require some external resources to do correctly.  The DNS for the alternate hostnames will need to be done manually, use either CNAMES or alias records to point to your cloudfront distribution.

## Some things to note:

* The compiled template is in the dist directory of this repo.  It should be generic enough that you can use it as-is for most deployments of a static website.
* When ACM creates a certificate, it will email the owner of the domain.  The certificate isn't issued until the owner approves it, and there is one email per domain/alternate domain.  Cloudformation waits for the certificate to be issued before moving on to creating the cloudfront distribution that uses it.
* The template is generated from partial files using my [workflow](https://github.com/entr04y/cf-builder).  If you want to edit the partial files take a look at the readme in the workflow to see how to compile the template, otherwise just grab the compiled template and edit that.
* This template will currently only work in us-east-1 because (at least according to the [docs](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cloudfront-distributionconfig-viewercertificate.html#cfn-cloudfront-distributionconfig-viewercertificate-acmcertificatearn)) Cloudfront can only retrieve certificates from us-east-1.  I've tested this in us-east-2 and for now it doesn't work.  The template is written so that it should still work if and when they fix this in the other regions. 