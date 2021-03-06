# aws-rolling-snaps
aws-rolling-snaps is a simple yet powerful tool to maintain ZFS-like rolling snapshots for EBS volumes

Features
========
- Written in Python 2.7/boto3
- No external module dependencies (except boto3)
- Simple tag based selection, only snapshots volumes that carry [preconfigured] EC2 tag
- Run anywhere, even outside of AWS (from cron)
- Or, run it serverless, in AWS Lambda
- Configurable retention policy with reasonable defaults
- SNS notifications

Quick start (cron)
=========
- (Optional) Create an IAM user to execute the script with the [following policy](makesnapshot-policy.json). If you're impatient or like to live dangerously, you can always resort to running it with AWS root or IAM admin privileges.  If you will only snapshot from volume tags, not instances, you can omit the "ec2:DescribeInstances" permission.

- Install boto3

```sh

    $ pip install boto3
```
Next, set up credentials (in e.g. ``~/.aws/credentials``). You can use the default profile or use another one. ([More info at AWS CLI documentation](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-multiple-profiles)) :

```ini

    [default]
    aws_access_key_id = YOUR_KEY
    aws_secret_access_key = YOUR_SECRET
```
Then, set up a default region (in e.g. ``~/.aws/config``):

```ini

    [default]
    region=us-east-1
```

- Mark EBS volumes that you wish to snapshot with a tag ('MakeSnapshot': 'true' by default)

- Configure cron to run the script

```sh

    $ chmod +x makesnap3.py
    $ crontab -e
    30 1 * * 1-6  /path-to/makesnap3.py day
    30 2 * * 7    /path-to/makesnap3.py week
    30 3 1 * *    /path-to/makesnap3.py month
    # optional hourly/yearly runs
    #15 */8 * * * /path-to/makesnap3.py hour
    #30 4 31 12 * /path-to/makesnap3.py year
```

- Profit

~~Quick~~ start (AWS Lambda)
=========

Setting up Lambda function in AWS is a somewhat cumbersome process, including defining policies, roles, CloudWatch rules, permissions and so on. Please feel free to inspect/modify (schedule times, optional hourly run etc) and then run [lambda-setup.sh](lambda-setup.sh) script for it. In case you feel like doing this multiple times (usually this is the case) there's a cleanup script [lambda-cleanup.sh](lambda-setup.sh).

Don't forget to mark EBS volumes that you wish to snapshot with a tag ('MakeSnapshot': 'true' by default)

Configuration
=========
- Configurable parameters:
```ini
        'arn': 'arn:aws:sns:eu-west-1:1234xxxxx:yyyyyyy',
        'tag_name': 'MakeSnapshot',
        'tag_value': 'true',
        'tag_type': 'volume',
        'running_only': false,
        'keep_day': 3,
        'keep_week': 4,
        'keep_month': 3,
        'keep_hour': 4,
        'keep_year': 10,
        'log_file': 'makesnapshots.log',
        'aws_profile_name': 'default',
        'ec2_region_name': 'us-west-2',
        'skip_create': false,
        'skip_delete': false
```
- Configuration is read from 'config.json' file (cp config.json.sample config.json, and edit)
- Config parameters are also read from the environment. Environment variables `MAKESNAP_<parameter>` (f.e. `MAKESNAP_KEEP_HOUR` etc) are read and applied after config file, overriding the values. Lambda now supports environment variables (https://aws.amazon.com/blogs/aws/new-for-aws-lambda-environment-variables-and-serverless-application-model/), it is a nice way of configuring lambda without bundling a config file.
- 'tag_type' can be 'volume' or 'instance'. 'instance' type snapshots all of the tagged instance's volumes, no matter tagged or not
- 'running_only' - when 'tag_type' is set to 'instance' - snapshot only currently running instances' volumes
- 'skip_create'/'skip_delete' - skip creating/deleting steps. Useful if snapshots are created/deleted by some other means

Notes
=========
- Snapshots of busy volumes may take long time. If you have a lot of (or) busy volumes - don't use Lambda. Maximum timeout for Lambda is 300s and there's currently no way to disable or confgure retry on error (if you know - let me know, please).

TODO
=========
- Per volume retention policy override with tags (like 'MakeSnapRetention': 'daily:7,weekly:8,monthly:6')
- CloudFormation template for easy Lambda setup/teardown

Credits
=========
This script started as a boto3 rewrite of the excellent makesnapshot tool (https://github.com/evannuil/aws-snapshot-tool)
