# Cloud Challenge Write Up
## Background
There are 3 challenges that you need to solve in order: Q1 Behind, Q2 Hidden, and Q3 Desire. Q2 will be built on top of Q1, and so on. The cloud service used is of AWS and you need to have an AWS account (doesn't have to be paid!) to interact with the challenge. Also, you might want to turn on some of the extra regions in the setting, particularly Hong Kong because they might host their challenges there (for some reasons the HK region is not on by default if you're not from there).

Also sorry I did not document my process (I think I shouldn't be writing about this at all???? The CTF is kind of secretive too so uh my bad). But you should get the idea! I tried narrating the WU so it's easier to follow through for someone with little to no experience! A cloud-based CTF is a rare one for sure, so I want people to be able to learn.

## Q1: Behind
I was given a static web page that was hosted on AWS's CDN service, or Cloudfront. At first I didn't know what to do since (with the 2 weeks prep time I had) I only learnt how to exploit S3 buckets and IAM policy (foreshadowing). I did a DNS scan with dns-lookup, I tried curl and see the headers, to see if there are parts of the Cloudfront I can hijack or change using SSRF etc. And then my teammate bruteforced for valid paths of the website, which resulted in a login page being found (this step isn't necessary because you can click on the login button and it'll work. It's just how we found it). The login page wasn't implemented yet, it's just an image that is _hosted on an S3 bucket._ 

![images](https://github.com/user-attachments/assets/e212fac5-9d80-4091-a91e-b6108338e5ff)

Found my potential weak point. First thing I did was trying to **list all of the contents** of the S3 bucket, and it worked. It seemed like the cloud engineer didn't secure the bucket enough because **everyone** can look up what the bucket has. This is how I found the flag. For this challenge, you don't _need_ the AWS console to list the files. I believe you can just visit the S3 bucket URL and it will show all the files it has. You can search for more open buckets [here](https://buckets.grayhatwarfare.com/).

**Vulnerability:** S3 Policy Misconfiguration/Open Bucket. 

## Q2: Hidden
For this challenge, I was given a premise that an older version of the flag existed and I needed to find it. This is kind of new to me so I Googled the command needed to show multiple versions of the bucket. If I'm not mistaken the command was this: `aws s3api list-object-versions --bucket bucket-name`. More info [click here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/list-obj-version-enabled-bucket.html). This will show you the versions this bucket has.

After it is confirmed that a previous/older version of the bucket exists, you can make a GET request with the version you want to inspect as the parameter as shown in this [documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/RetrievingObjectVersions.html). I was able to not only get the flag but also a **suspicious looking CSV file** called accessKey. This will come handy for the next challenge. 

**Vulnerability:** S3 Policy Misconfiguration.

## Q3: Desire
I didn't even read what the challenge's premise was because I straight up use the accessKey to see if it's still valid (and it was). For those who don't know how AWS users/roles work, basically you have a user: which is the account you use to login/sign up/manage stuff, and a role: a more temporary/universal thing (it can be attached not just to an _actual user_ but to machines, programs, etc) that will **grant temporary access** (GROSS OVERSIMPLIFICATION, PLEASE TAKE A CLOUD COMPUTING CLASS INSTEAD). The way you'd authorize a role is to use an access key that looks something like this:

![1_qHEytQdhSTVQb1MpA-NMAw](https://github.com/user-attachments/assets/8c6ec950-7e8f-48bf-a534-e46e8f06fd5e)

You'd want to import these credentials to your .aws file (this is another potential vulnerabilty, if you don't secure your .aws file you will get screwed). You can make a new one and copy the credentials you found or I believe you can use the CLI to do it for you. I did that and checked the IAM policy to see what role I just got and at first it was _pretty innocent._ It's a **test role** with no special privileges, no attached bucket, absolutely nothing. However, it _does_ have something interesting. 

This is what a typical IAM policy looks like. It basically tells you what role have what permission and under what conditions (if any):
```
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::vulnerable-cloud",
      "Condition": {}
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::what-the-f-is-a-kilometer/*",
      "Condition": {}
    }
  ]
}
```

I do not have the original policy, but I remember one policy that a role can be assumed, with the condition that it's assumed by an IAM user with the **vulnerable** wildcard rule: `hackaday_*`. Now what this mean is basically, if you have an IAM user with the name `hackaday_a` or `hackaday_b` you **can** assume this role. Assuming role in AWS term (which is an oversimplification) is basically a way to authenticate yourself as the role, kind of like _stealing_ a role? The tricky part here, since the test role that was given via the leaked accessKey file didn't have permission to create a new IAM user (nor role), you need to go to your AWS account and **make that user yourself.** As long as it follows the wildcard rule, you will be able to assume the role. An IAM policy that allows role assumption looks something like this:

![tDrSMUyf](https://github.com/user-attachments/assets/abfee37a-7ce9-4f8d-8860-40f07adbd871)

Note: Before assuming any role, make sure the new IAM user you made has the **required permission** to do an AssumeRole task. When you first create an IAM user from the AWS console, it's going to be _really_ stripped down, you need to specify what it can do afterwards. After you setup the user, you should export/take note of the Access Key and Access Secret (just like what you got from the leaked accessKey file). Then, authenticate yourself as that user. 

To assume a role simply do this: `aws sts assume-role <role arn>`. The role arn is the line that starts with `arn:aws:iam:.../user:...`. If you have successfully assumed the role you will be given an output similar to this:

![image](https://github.com/user-attachments/assets/ea458553-e170-49aa-ad1a-c8d385d23162)

Okay now what do you do with an assumed role? Well, you import the credentials you just got _and_ I tried what my instinct told me: check for S3 buckets. You can list an attached resource from an IAM user/role using the [command](https://docs.aws.amazon.com/AmazonS3/latest/userguide/list-buckets.html): `aws s3 ls`. It is possible that other resources than S3 buckets are attached to this user (EC2, Docker Image, etc). If that's the case, you can just browse the AWS documentation to see the appropriate commands. Then, I use the same trick from the first challenge to list the content of the bucket that I owned. I got the flag this way! I think you need to use the CLI for this since you need to be the _authenticated role_ to be able to access the contents. Use this command: `aws s3 list-objects --bucket <name>`. [More info.](https://docs.aws.amazon.com/cli/latest/reference/s3api/list-objects.html)

**Vulnerability:** Not revoking leaked credentials, weak/ambiguous IAM policy. 

[Read more about this challenge's specific vulnerability.](https://rhinosecuritylabs.com/aws/assume-worst-aws-assume-role-enumeration/)
