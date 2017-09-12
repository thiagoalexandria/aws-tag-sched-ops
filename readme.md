# Start, Reboot, Stop and Back Up AWS Resources Using Tags

## Benefits

* **Save money** by stopping resources during off-hours
* **Enhance reliability** by taking frequent backups
* **Use tags** to schedule operations
* Secure tagging rights, and backup deletion, with Identity and Access Management (IAM) policies
* Install easily, from a CloudFormation template

## Quick Start

1. Log in to the [AWS Web Console](https://signin.aws.amazon.com/console) as a privileged user.

   _Security Tip:_ To see which services and resource types you'll need access to, [search for `Type: ` in CloudFormation templates](https://github.com/sqlxpert/aws-tag-sched-ops/search?q=Type%20path%3A%2F%20extension%3Ayaml).

2. Make sure that the desired **region** is selected in the black bar, near the top right corner.

3. Navigate to the [S3 Console](https://console.aws.amazon.com/s3/home). Click the name of the bucket where you keep CloudFormation templates, or create the bucket, if necessary.

   _Security Tip:_ Remove public read and write access. Carefully limit write access.

4. Upload the compressed source code of the AWS Lambda function, [`aws_tag_sched_ops_perform.py.zip`](aws_tag_sched_ops_perform.py.zip)

   _Security Tip:_ The file's current `md5sum` is `22026de6ba32c87d8cc41dd69bf474fc`

5. Navigate to the [CloudFormation Console](https://console.aws.amazon.com/cloudformation/home). Click Create Stack. Under "Choose a template", click "Upload a template to Amazon S3". Click Choose File and navigate to your local copy of [`aws_tag_sched_ops.yaml`](aws_tag_sched_ops.yaml). On the next page, set these items (only):

   |Item|Value|
   |--|--|
   |Stack name|`TagSchedOps`|
   |LambdaCodeS3Bucket|_Name of your S3 bucket_|
   
6. Navigate to [Volumes](https://console.aws.amazon.com/ec2/v2/home#Volumes) in the EC2 Console. Right-click the name of a volume and select Add/Edit Tags. Add:

   |Key|Value|Note|
   |--|--|--|
   |`managed-snapshot`||Leave value blank|
   |`managed-snapshot-periodic`|`d=*,H:M=11:30`|Replace `11:30` with [current UTC time](https://www.timeanddate.com/worldclock/timezone/utc) + 15 minutes|

7. After 20 minutes, check [Snapshots](https://console.aws.amazon.com/ec2/v2/home#Snapshots) in the EC2 Console, for a new snapshot.

8. Delete the sample snapshot and untag the volume.

9. Navigate to [Users](https://console.aws.amazon.com/iam/home#/users) in the IAM Console. Click on your regular (uprivileged) username. Click Add Permissions, then click "Attach existing policies directly". Add:

      * `Ec2TagSchedOpsAdminister`
      * `Rds2TagSchedOpsAdminister`
      
   _Security Tip_: Carefully limit all EC2 and RDS tagging privileges.

10. Log out of the AWS Console. You can now manage schedule tags without logging in as a privileged user.

## Operation-Enabling Tags

* To enable an operation, tag the resource with a tag from the table. The value of this tag does not matter; leave it blank.

  |AWS Resource|Start|Create Image|Reboot then Create Image|Reboot|Create Snapshot|Create Snapshot then Stop|Stop|
  |--|--|--|--|--|--|--|--|
  |EC2 compute instance|`managed-start`|`managed-image`|`managed-reboot-image`|`managed-reboot`| | |`managed-stop`|
  |EC2 EBS disk volume| | | | |`managed-snapshot`| | |
  |RDS database instance|`managed-start`| | |`managed-reboot`|`managed-snapshot`|`managed-snapshot-stop`|`managed-stop`|

* Also tag the resource with valid [repetitive (`-periodic`)](#repetitive-schedules) and/or [one-time (`-once`)](#one-time-schedules) schedule tag(s). Prefix with the operation.
* If there are no corresponding schedule tags, an enabling tag will be ignored, and the operation will never occur.
* To temporarily suspend an operation, delete its enabling tag. You may leave its schedule tag(s) in place.
* Examples (for an EC2 or RDS instance):

  |Tag(s) and Value(s)|Success|Comment|
  |--|--|--|
  |`managed-start`, `managed-start-periodic`=`u=1,H=09,M=05`|Yes|Enabled and scheduled|
  |`managed-start`=`No`, `managed-start-periodic`=`u=1,H=09,M=05`|Yes|Value of enabling tag is ignored|
  |`managed-start`, `managed-start-once`=`2017-12-31T09:05`|Yes|Enabled and scheduled|
  |`managed-start`, `managed-start-periodic`=`u=1,H=09,M=05`,`managed-start-once`=`2017-12-31T09:05`|Yes|Repetitive and one-time schedules can be combined|
  |`managed-start`|No|No schedule tag|
  |`managed-start-once`=`2017-12-31`|No|No enabling tag (suspend)|
  |`managed-start`, `managed-start-once`|No|Blank schedule|
  |`managed-start`, `managed-start-periodic`=`Monday`|No|Invalid schedule|
  |`managed-start`, `managed-stop-periodic`=`u=1,H=09,M=05`|No|Different operations|

## Scheduling
 
 * All times are UTC, on a 24-hour clock.
 * The function runs once every 10 minutes. The last digit of the minute is always ignored. For example, an operation scheduled for `M=47` is expected to begin between 40 and 50 minutes after the hour, depending on startup overhead.
 * Month and minute values must have two digits. Use a leading zero (for example, `03`) if a month or minute value is less than or equal to 9. (Because there are only 7 days in a week, weekday numbers have only one digit.)
 * Use a comma (`,`) _without any spaces_ to separate components. The order of components within a tag value does not matter.
 * `T` separates day information from time; it is not a variable.
 * Each operation supports a pair of tags for [repetitive (`-periodic`)](#repetitive-schedules) and [one-time (`-once`)](#one-time-schedules) schedules. Prefix with the operation.
 * If the corresponding [enabling tag](#operation-enabling-tags) is missing, schedule tags will be ignored, and the operation will never occur.

### Repetitive Schedules

  * Tag suffix: `-periodic`

  * Values: one or more of the following components:

    |Name|Minimum|Maximum|Wildcard|Combines With|
    |--|--|--|--|--|
    |Day of month (`d`)|`d=01`|`d=31`|`d=*`|`H` and `M`, or `H:M`|
    |Weekday (`u`)|`u=1` (Monday)|`u=7` (Sunday)||`H` and `M`, or `H:M`|
    |Hour (`H`)|`H=00`|`H=23`|`H=*`|`d` or `u`, and `M`|
    |Minute (`M`)|`M=00`|`M=59`||`d` or `u`, and `H`|
    |Hour and minute (`H:M`)|`H:M=00:00`|`H:M=23:59`||`d` or `u`|
    |Day of month, hour and minute (`dTH:M`)|`dTH:M=01T00:00`|`dTH:M=31T23:59`|||
    |Weekday, hour and minute (`uTH:M`)|`uTH:M=1T00:00`|`uTH:M=7T23:59`|||
    
      * To be valid, a component or combination of components must specify a day, hour and minute.
      * Repeat a whole component to specify multiple values. For example, `d=01,d=11,d=21` means the 1st, 11th and 21st days of the month.
      * The `*` wildcard is allowed for day (every day of the month) and hour (every hour of the day).
      * For consistent one-day-a-month scheduling, avoid `d=29` through `d=31`.
      * Labels are from `strftime` and weekday numbering follows the [ISO 8601 standard](https://en.wikipedia.org/wiki/ISO_8601#Week_dates) (different from `cron`).

  * Examples:
  
    |Value of `-periodic` Schedule Tag|Demonstrates|Operation Expected to Begin|
    |--|--|--|
    |`d=28,H=14,M=25` _or_ `dTH:M=28T14:25`|Monthly event|Between 14:20 and 14:30 on the 28th day of every month.|
    |`d=1,d=8,d=15,d=22,H=03,H=19,M=01`|`cron`-style schedule|Between 03:00 and 03:10 and again between 19:00 and 19:10, on the 1st, 8th, 15th, and 22nd days of every month.|
    |`d=*,H=*,M=15,M=45,H:M=08:50`|Extra event in the day|Between 10 and 20 minutes after the hour and 40 to 50 minutes after the hour, every hour of every day, _and also_ every day between 08:50 and 09:00.|
    |`d=*,H=11,M=00,uTH:M=2T03:30,uTH:M=5T07:20`|Extra weekly events|Between 11:00 and 11:10 every day, _and also_ every Tuesday between 03:30 and 03:40 and every Friday between 07:20 and 7:30.|
    |`u=3,H=22,M=15,dTH:M=01T05:20`|Extra monthly event(s)|Between 22:10 and 22:20 every Wednesday, _and also_ on the first day of every month between 05:20 and 05:30.|
    
### One-Time Schedules
 
  * Tag suffix: `-once`

  * Values: one or more [ISO 8601 combined date and time strings](https://en.wikipedia.org/wiki/ISO_8601#Combined_date_and_time_representations), of the form `2017-03-21T22:40` (this example means March 21, 2017 at 22:40)
      * Remember, the code runs once every 10 minutes and the _last digit of the minute is ignored_
      * Omit seconds and fractions of seconds
      * Omit time zone

## Operation Combinations

* Multiple _non-simultaneous_ operations on the same resource are allowed.
* If two or more operations on the same resource fall on the same day, during the same 10-minute time interval, the function combines them if possible:

  |Resource|Simultaneous Operations|Effect|
  |--|--|--|
  |EC2 instance|Stop + Reboot|Stop|
  |EC2 instance|Create Image + Reboot|Reboot then Create Image|
  |RDS instance|Stop + Reboot|Stop|
  |RDS instance|Stop + Create Snapshot|Create Snapshot then Stop|

* The EC2 instance Create Image + Reboot combination is the most useful. For example, you could use it to take hourly backups but reboot only before the midnight backup. The midnight backup would be guaranteed to be coherent for all files, but you could safely retrieve static files as of any given hour, from the other backups. To set up this example:

  |Tag|Value|
  |--|--|
  |`managed-image`||
  |`managed-image-periodic`|`d=*,H=*,M=59`|
  |`managed-reboot`||
  |`managed-reboot-periodic`|`d=*,H=23,M=59`|
  
  (23:59, which for the purposes of this function represents the last 10-minute interval of the day, is the unambiguous way to express _almost the end of some designated day_, on any system. 00:00 and 24:00 could refer to the start or the end of the designated day, and not all systems accept 24:00, in any case. Remember that all times are UTC; adjust for night-time in your time zone!)

* Non-combinable operations result in no operation. Affected resources are logged only when the function is executed in [Debug Mode](#debug-mode).

  |Bad Combination|Reason|Example|
  |--|--|--|
  |Mutually exclusive operations|These conflict with each other.|Start + Stop|
  |Choice of operation depends on current state of instance|The state could change between the status query and the operation request.|Start + Reboot|
  |Sequential or dependent operations|The logical order cannot always be inferred. Also, operations proceed asynchronously; one might not complete in time for another to begin. Note that Reboot then Create Image (EC2 instance) and Create Snapshot then Stop (RDS instance) are _single_ AWS operations.|Start + Create Image|
  
## "Child" Resources

Some operations create a child resource (image or snapshot) from a parent resource (instance or volume).
 
### Naming Conventions

* This function names _all_ child resources.
* For simplicity, no uppercase letters are used.
* AWS imposes different character set restrictions for different resource types. This function replaces known forbidden characters with with `X`.
* The name consists of these parts, in order, and separated by hyphens (`-`):

  |#|Part|Example|Purpose|
  |--|--|--|--|
  |1|Prefix|`zm`|Identifies and groups children created by this function, for interfaces that do not expose tags. `z` will sort after most manually-created images and snapshots. `m` stands for "managed".|
  |2|Parent name or identifier|`webserver`|Conveniently indicates the parent. Derived from the `Name` tag (if not blank), the logical name (if supported), or the physical identifier (as a last resort). Multiple children of the same parent will sort together, by creation date (see next row).|
  |3|Date/time|`20171231T1400`|Indicates when the child was created. The last digit of the minute is normalized to 0. The `-` and `:` separators are removed for brevity, and because AWS does not allow `:` in names, for some resource types. (The `managed-date-time` tag stores the original string, with separators intact.)|
  |4|Random string|`g3a8a`|Guarantees unique names. Five characters are chosen from a small set of letters and numbers that are unambiguous.|

* If it is ever necessary to parse these names, keep in mind that the second part may contain internal hyphens.
* For some resource types, the description is also set to the name, in case interfaces only expose one or the other.

### Special Tags

* All tags other than operation-enabling tags, schedule tags, and the `Name` tag, are copied from parent to child.

* Special tags are added to the child resource:

  |Tag(s)|Purpose|
  |--|--|
  |`Name`|Supplements EC2 resource identifiers. Key is renamed `managed-parent-name` when its value is passed from parent to child, because the child has a `Name` tag of its own; see [Naming Conventions](#naming-conventions). This function handles `Name` specially for both EC2 and RDS, in case EC2-style tag semantics are eventually extended to RDS.|
  |`managed-parent-name`|`Name` tag from the parent. Not added if blank.|
  |`managed-parent-id`|The identifier of the parent EC2 instance, EC2 EBS volume, or RDS instance. AWS metadata captures this (for example, as `VolumeId`, for EC2 EBS volume snapshots), but field names and usage capabilities differ for every AWS service and resource type.|
  |`managed-origin`|The operation (for example, `snapshot`) that created the child. Identifies resources created by this function. Also distinguishes special cases, such as whether an EC2 instance was or was not rebooted before an image was created. If operation-enabling tags were copied from parent to child, they might conflict with future tags for automated operations on the child (such as scheduled deletion of old images and snapshots).|
  |`managed-date-time`|Groups resources created during the same 10-minute interval. AWS metadata captures the _exact_ time, and field names and usage capabilities differ for every AWS service and resource type.|

## Security Model

 * It is a good practice to tag images and snapshots for deletion, but to let a privileged user actually delete them. This is the purpose of the `managed-delete` tag.
 
 * Entities that can create backups should not be able to delete backups (or even to tag them for deletion). Mutually exclusive IAM policies are provided:
 
   |Policy Name|Create Images/Snapshots|Tag for Deletion|Delete|Any Other Effects?|
   |--|--|--|--|--|
   |\*TagSchedOpsPerform|Yes|No|No|Yes|
   |\*TagSchedOpsAdminister|No|Yes|No|Yes|
   |\*TagSchedOpsTagForDeletion|No|Yes|No|No|
   |\*TagSchedOpsDelete|No|No|Yes|No|
   
 * There is not yet an automatic mechanism to tag images and snapshots for deletion. A correct archival policy would not be strictly age-based. For example, it might preserve the last 30 days' worth of daily backups, and beyond 30 days, the first backup of every month. Work on a syntax for specifying archival policies remains to be done. Consider the snapshot retention property of RDS databases instances: the daily backups created when that property is set cannot be kept longer than 35 days.

 * Only a few trusted users should be allowed to tag EC2 and RDS resources, because tags determine which resources are started, backed up, rebooted, and stopped, and which backups are protected from deletion. IAM policies for users with no other tagging privileges:
 
 * In IAM policies, the `Deny` Effect always takes precedence over `Allow`. Extending broad privileges and then denying tagging privileges works for entities not meant to have any tagging rights, but not for entities meant to have some tagging rights. For the latter, you must create policies that explicitly allow all desired EC2 and RDS actions _other than tagging_.
 
 * Due to an oversight in IAM for EC2, an entity that can create an instance image can always force a reboot by omitting the `NoReboot` option. Denying reboot privileges does not help. Ths combination of a safe privilege, taking a backup, with a dangerous one, rebooting, is particularly unfortunate.
 
 * Due to a limitation in IAM for EC2, tags are ignored when deleting images and snapshots. Limit EC2 image and snapshot deletion privileges -- even `Ec2TagSchedOpsDelete` -- to highly-trusted entities.
 
 * Due to a limitation in IAM for RDS, an entity that can add specific tags can add _any_ other tags in the same request. Limit RDS tagging privileges -- even `Rds2TagSchedOpsAdminister` -- to highly-trusted users.
 
 * Privileges to change the AWS Lambda function, the CloudWatch Event rule that triggers it, or the IAM policies on which it depends, must be strictly controlled.

## Licensing

|Scope|License|
|---|---|
|Source code files|[GNU General Public License (GPL) 3.0](http://www.gnu.org/licenses/gpl-3.0.html)|
|Source code within documentation files|[GNU General Public License (GPL) 3.0](http://www.gnu.org/licenses/gpl-3.0.html)|
|Documentation files|[GNU Free Documentation License (FDL) 1.3](http://www.gnu.org/licenses/fdl-1.3.html)|

Paul Marcelin, 2017
