# AWS Organizations & Multi-Account Learning Structure

## TL;DR

When we start learning about AWS Organizations, it can be overwhelming to know where to begin. This document provides a structured learning path to help you understand the key concepts and features of AWS Organizations in a logical order as well as a structure example to get you started with your own learning journey.

## What is AWS Organizations?

AWS Organizations is a service that allows you to centrally manage and govern multiple AWS accounts. It provides features such as account management, what services to enable, and policy types across your AWS environment.

## From Management to Member Account Permissions

When an account is created in AWS Organizations, it is called a member account and if it's joined at a later stage it becomes a member account as well.

### AWSServiceRoleForOrganizations

When an account is joined or created into an organization, AWS Organizations creates a service-linked role called `AWSServiceRoleForOrganizations` in that account. This role allows AWS Organizations to perform actions on behalf of the organization in that account. It has the necessary permissions to manage the account and its resources as part of the organization.

The intent however is not a full-blown admin role, but rather a role that allows AWS Organizations to inter-service coordination.

### OrganizationAccountAccessRole

When a new account is created via AWS Organizations, the management account (the account that created the organization) has full access to the member accounts, and this is done through the IAM role called `OrganizationAccountAccessRole`. This role allows the management account to assume this role and has the IAM policy `AdministratorAccess` attached to it, which gives it full access to the member account.

If the member account was created outside of AWS Organizations and then invited to join, the `OrganizationAccountAccessRole` is not automatically created in that account. This means that the management account will not have full access to the member account until this role is manually created and configured in the member account.

Keep in mind this is classified as a cross-account role.

## Multi-Account Strategy Considerations

Often when a company starts using AWS, they may have a single account for all their workloads. However, as the company grows and the number of workloads increases, it becomes necessary to segregate workloads into different accounts for better management, security, and cost allocation.

- How do we split our workload traffic with multiple VPCs, one account or multiple accounts?
- Where and how do we enforce tagging policies and cost allocation? OU or account level?
- Do we send all CloudTrail logs to a single account or do we have a logging account per OU?
- Do we aggregate all logging into a single account or do we have a logging account per OU?
- Do we house security tooling in a single account or do we have a security tooling account per OU?

This is just the surface of the questions that arise when designing a multi-account strategy. The answers to these questions depend on the specific needs and requirements of your organization, and there is no one-size-fits-all solution. It's important to carefully consider the trade-offs and implications of different approaches when designing your multi-account strategy.

## OU Example Structure

There are different levels of OU structure recommendations for an AWS Multi-Account strategy. If we check the foundational example we see Security OU with the accounts Log Archive and Security Tooling.

https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/foundational-ous.html

For workloads it's recommended to have multiple workload accounts under OUs named after environments such as Development, Staging, and Production. This allows for better segregation of workloads and easier management of resources and policies.

https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/application-ous.html

For my own personal accounts I ended up having something closed to the procedural OU suggestiins.

https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/procedural-ous.html

```tree
Root
├── management
├── (ou) infrastructure
│   ├── networking
│   ├── shared-services
├── (ou) sandbox
│   ├── sandbox
├── (ou) security
│   ├── audit
│   ├── log-archive
│   ├── security-tooling
├── (ou) workloads
│   ├── (ou) acc
│       ├── workload-acceptance
│   ├── (ou) dev
│       ├── workload-development
│   ├── (ou) prd
│       ├── workload-production
```

In some ways it might be overkill but I found this makes it easier for me to manage my accounts and resources, and also allows me to apply different policies and controls at the OU level for learning purposes.

### Final Thoughts on OU

You can use whatever you like, from business unit, environmental, project even, it really depends on your needs and requirements but I'd say stick to what AWS recommends for their customers. 

https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html

## Feature Modes

By default All Features mode is enabled in AWS Organizations, which means that all features of AWS Organizations are available to use. This includes features such as consolidated billing, service control policies, and organizational units.

If an account is invited it needs to approve the enabling of All Features mode before it can be fully integrated into the organization. We can then prevent an account from leaving the organization by enabling the "Leave Organization" feature, which is a security measure to prevent unauthorized accounts from leaving the organization.

https://aws.amazon.com/blogs/mt/essential-security-controls-to-prevent-unauthorized-account-removal-in-aws-organizations/

Consolidated billing allows you to combine the billing for multiple accounts into a single bill, which can help you save money by taking advantage of volume discounts and other cost-saving features.

https://docs.aws.amazon.com/organizations/latest/userguide/orgs_getting-started_concepts.html#feature-set-cb-only

## Reserved Instances & Savings Plans

They paying account (management) can turn off the sharing of Reserved Instances and Savings Plans, which means that the benefits of these cost-saving features will not be shared across the accounts in the organization.

This can be useful if you want to keep the benefits of these features within a specific account or if you want to prevent other accounts from using them.

## Moving Member Account Between AWS Organizations

1. Send invite to the account you want to move from the old organization to the new organization.
2. Accept the invite from the account you want to move.

https://docs.aws.amazon.com/organizations/latest/userguide/orgs_account_migration.html#migrate-account-process
