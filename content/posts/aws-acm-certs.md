+++
title = "AWS ACM Certificates"
date = "2025-01-16T01:52:08+11:00"
author = "Yasha Prikhodko"
authorTwitter = "yashafromrussia" #do not include @
cover = ""
tags = ["aws", "acm", "ssl", "route53"]
keywords = ["aws", "acm", "ssl", "route53"]
description = "AWS ACM certificates stuck in 'Pending validation' state even after creating the CNAME records in Route53."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

> TLDR: If your AWS ACM certificates are stuck in "Pending validation" state and you've created the
CNAME records in Route53, create a new certificate for the domains that are stuck. This will kick off
the domain validation process.

---

Today has been the most stressfull day of the 2025. I'm currently working on migrating all of our
images from Cloudinary to ImgProxy. That already being an urgent task due to the looming costs of
Cloudinary, I was also tasked with making sure the SSL certificates for the existing domains are
renewed. They're expiring in ~18 hours!

No problem. Hold my beer. Usually, SSL certificates are renewed automatically by AWS ACM. However,
our certificates and our DNS (Route53) are managed by different AWS accounts. This means that the
certificates are not automatically renewed, even with the DNS validation method.

A single certificate is issued for multiple domains. ACM lists all the DNS records that need to be
created in order to validate the certificate. There are way too many records to manually create in
Route53. I need to automate this process. Given that I'm already busy working on the image migration,
I wanted to automate this ASAP. This is also a one-time task since we're reorganizing our AWS
accounts.

Our infrastructure is fairly old - 10 or so years. And it's managed by homegrown "infrastructure as
code" scripts. So I asked Google Gemini for help. After 10 prompts, it gave me the
script:

```python
import boto3
import csv
import argparse


def get_top_domain(domain_name):
    """Extracts the top-level domain from a given domain name,
    handling various TLDs including .com.au.
    """

    try:
        # Split the domain name by dots
        parts = domain_name.split(".")

        # Handle special cases like .com.au, .co.uk, etc.
        if len(parts) >= 3 and parts[-2] in ["com", "co", "org", "net"]:
            return ".".join(
                parts[-3:]
            )  # Get the last three parts (e.g., example.com.au)
        elif len(parts) >= 2:  # Other cases (e.g., example.com, example.net)
            return ".".join(parts[-2:])  # Get the last two parts
        else:
            return None  # Invalid domain format
    except Exception as e:
        print(f"Error extracting top domain from {domain_name}: {e}")
        return None


def get_hosted_zone_id(domain_name):
    """Retrieves the Hosted Zone ID for a given domain name,
    handling wildcards by looking for the parent domain's zone.
    """

    route53_client = boto3.client("route53")

    try:
        print(f"Looking up Hosted Zone ID for {domain_name}")
        response = route53_client.list_hosted_zones_by_name(
            DNSName=domain_name, MaxItems="1"
        )
        if response["HostedZones"]:
            return response["HostedZones"][0]["Id"]
        else:
            return None
    except Exception as e:
        print(f"Error getting Hosted Zone ID for {domain_name}: {e}")
        return None


def update_cname_records(hosted_zone_id, cname_name, cname_value):
    """Updates a CNAME record in Route 53."""

    route53_client = boto3.client("route53")

    try:
        response = route53_client.change_resource_record_sets(
            HostedZoneId=hosted_zone_id,
            ChangeBatch={
                "Changes": [
                    {
                        "Action": "UPSERT",  # Use UPSERT to create or update
                        "ResourceRecordSet": {
                            "Name": cname_name,
                            "Type": "CNAME",
                            "TTL": 300,
                            "ResourceRecords": [{"Value": cname_value}],
                        },
                    }
                ]
            },
        )
        print(f"CNAME record updated successfully: {cname_name} -> {cname_value}")
    except Exception as e:
        print(f"Error updating CNAME record: {e}")


def main():
    """Reads CNAME records from a CSV file and updates Route 53."""

    parser = argparse.ArgumentParser(
        description="Update Route 53 CNAME records from a CSV file."
    )
    parser.add_argument(
        "csv_file", help="Path to the CSV file containing CNAME records."
    )
    args = parser.parse_args()

    with open(args.csv_file, "r") as csvfile:
        reader = csv.DictReader(csvfile)
        for row in reader:
            domain_name = row["Domain name"]
            cname_name = row["CNAME name"]
            record_type = row["Type"]
            cname_value = row["CNAME value"]

            print("")

            if record_type == "CNAME":
                hosted_zone_id = get_hosted_zone_id(get_top_domain(domain_name))
                if hosted_zone_id:
                    print(
                        f"Updating CNAME record for {domain_name} in hosted zone {hosted_zone_id}: {cname_name} -> {cname_value}"
                    )
                    update_cname_records(hosted_zone_id, cname_name, cname_value)
                else:
                    print(
                        f"Error: Could not find Hosted Zone ID for {domain_name}. Skipping record."
                    )
            else:
                print(
                    f"Skipping non-CNAME record for {domain_name}: {cname_name} ({record_type})"
                )


if __name__ == "__main__":
    main()

```

This script reads a CSV file that you can export from AWS ACM. The CSV file contains the domain
names and the CNAME records that need to be created in Route53. The script then tries to find the
Hosted Zone ID for the domain and creates the CNAME records.

Great! All of the records were created successfully. I check if the certificates were renewed and
they were __not__! Two domain names were stuck in "Pending validation" state. I checked for CNAME
records in Route53 and they were all there. I checked if it resolves on my end with `dig`:

```bash
dig CNAME <cname_name>. +short
```

It resolved to the correct value. After numerous SO threads and AWS documentation, I decided to give
it a few hours 🤞

Fast-forward to a few bit nails and 6 hours later, the certificates are still not renewed. Oof. We
have about 10 hours until they're expired. I decided to contact AWS support. After an hour of
troubleshooting with the support engineer, they got another engineer to look into it. They suggested
to create a new certificate for the domains that were stuck in "Pending validation" state. I did
that and the new certificate was issued right away. Once I checked the old certificate, all domains
were validated but the certificate was still not renewed. Noice! Apparently this kicked off the
domain validation process for the domains that were stuck. I had to wait another 20 or so minutes
for the certificate to be renewed. And it was! 🎉
