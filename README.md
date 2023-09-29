# AWS Core SSO Configuration

Utility to manage AWS SSO Permission Sets, SSO Groups, and their assignments to AWS
accounts from declarative YAML configuration files. Control Tower is not required.

This utility allows you to define groups like this:
```yaml
sso_group_name: 
  acme-inc-cost-management
  
description:
  Members of this group can manage AWS billing, payment, budgeting, and cost reporting 
  in the AWS Organisation account.
```

... and permission sets like so:
```yaml
permission_set_name: 
  DeveloperAccess
  
description: 
  A permission set for builders/developers. It includes read-only 
  permissions for everything a developer needs, plus mutation rights for 
  things that AWS Copilot might deploy or access.

session_duration: PT8H

permissions_boundary: developer-permission-boundary-policy

aws_managed_policies:
    - ViewOnlyAccess

inline_policy:
    Version: '2012-10-17'
    Statement:
        - 
            Sid: AllowGlobalServices
            Effect: Allow
            Resource: "*"
            Action:
                - access-analyzer:*
                - budgets:ViewBudget
                <etc>
                <etc>
                <etc>
```
... and to declare account assignments in the following way:
```yaml
account_id: "333333333333"

sso_groups:
  acme-inc-cloud-administration:
    - AWSAdministratorAccess
    - AWSReadOnlyAccess
  acme-inc-security-administration:
    - AWSReadOnlyAccess

  acme-inc-account-administration:
    - AWSServiceCatalogAdminFullAccess
  acme-inc-cost-management:
    - BillingAccess

  AWSControlTowerAdmins:
    - AWSAdministratorAccess
  AWSSecurityAuditPowerUsers:
    - AWSPowerUserAccess
  AWSSecurityAuditors:
    - AWSReadOnlyAccess
  AWSServiceCatalogAdmins:
    - AWSServiceCatalogAdminFullAccess
  AWSAccountFactory:
    - AWSServiceCatalogEndUserAccess

```
You can then deploy your groups, permission sets, and account assignments like so:
```console
$ ./sync-groups acme-inc

====================================================================================================
    conf/acme-inc/sso_groups/acme-inc-account-administration.yaml:
----------------------------------------------------------------------------------------------------

The SSO group acme-inc-account-administration already exists.

$ ./sync-permission-sets acme-inc

Retrieving permission sets...
Retrieving AWS managed policies (this may take a while)...

====================================================================================================
    conf/acme-inc/sso_permission_sets/DeveloperAccess.yaml:
----------------------------------------------------------------------------------------------------

DeveloperAccess already exists.
Updating description and session duration...
Putting inline policy...
Putting permissions boundary...
Aligning AWS managed policies...
Provisioning...

$ ./sync-accounts acme-inc

====================================================================================================
    conf/acme-inc/accounts/Org.yaml:
----------------------------------------------------------------------------------------------------

Account 333333333333
Keeping AWSAccountFactory with AWSServiceCatalogEndUserAccess
Keeping AWSControlTowerAdmins with AWSAdministratorAccess
Keeping AWSSecurityAuditPowerUsers with AWSPowerUserAccess
Keeping AWSSecurityAuditors with AWSReadOnlyAccess
Keeping AWSServiceCatalogAdmins with AWSServiceCatalogAdminFullAccess
Keeping acme-inc-cloud-administration with AWSAdministratorAccess
Keeping acme-inc-account-administration with AWSServiceCatalogAdminFullAccess
Keeping acme-inc-cost-management with BillingAccess
```


This utility will keep all settings in sync with what's specified in the YAML configuration files,
but by design this utility will not delete SSO Permission Sets once created, as deleting an SSO
Permission Set requires it first to be disassociated from any accounts to which it might
have been deployed. This is best done in the console, or even better, using AFT.

It should be noted that `./sync-accounts` can be used to sync group and permission set 
assignments to any account. There are example files (`Org.yaml`, `Audit.yaml`, and `LogArchive.yaml`) 
to cover configuration of Control Tower accounts not covered by AFT. However, neither AFT nor Control 
Tower is required.

You may also want to take a look at https://github.com/PeterBengtson/AFT-SSO-account-configuration,
which is an AFT addon customisation to allow SSO Groups, SSO Users and SSO Permission Sets to be assigned
declaratively for all Control Tower accounts. It can be used in conjunction with this utility.


## Prerequisites

* AWS CLI (v2.7.21 or late enough to support `put-permissions-boundary-to-permission-set` et al.)
* `zsh`
* `jq` (Should be pre-installed on MacOS)
* `python-yq` (On MacOS: `brew install python-yq`)


## Operation

Clone or fork this repository. Copy the `conf/acme-inc` directory to `conf/your-company-name` 
and tailor its files to your needs. Let's say you create a directory named `looney-tunes`. You can then 
deploy it by running one or all of the following, the first time probably in the order given:

```console
./sync-groups looney-tunes
./sync-permission-sets looney-tunes
./sync-accounts looney-tunes
```

This allows you to manage multiple installations in the same repository. You might also want to
modify `.gitignore` to allow your new directory to be committed to version control, as all
subdirectories under `conf` except `acme-inc`are excluded by default.

You can authenticate by any means you like, including temporary credentials pasted from the SSO
login screen, or through something like `aws sso login --profile xxxxx`. The named profile, or your
default profile, must give you admin access to the AWS Organizations administrative account.
