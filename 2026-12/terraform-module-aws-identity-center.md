# Terraform Module – AWS Identity Center

There are a few public modules available for AWS Identity Center, but many of them separate concerns like users, groups, permission sets, and account assignments into different layers. In practice, I found account assignments to be the most difficult part to manage consistently.

This module takes a slightly different approach by aligning more closely with how access is typically reasoned about:

- Users are members of groups  
- Groups are responsible for account assignments  
- Permission sets are defined independently and attached only when needed  

This allows permission sets to remain reusable and decoupled, while groups become the central place for managing access. It also enables direct user-to-group binding, which simplifies day-to-day access management.

The module supports creating groups without users and without permission set assignments, making it suitable for bootstrapping new environments or incrementally building out access models.

```mermaid
graph TD
    Users["👥 Users"]
    Groups["👤 Groups"]
    AccountAssignment["🔗 Account Assignment"]
    PermissionSets["🔐 Permission Sets"]
    Accounts["☁️ AWS Accounts"]
    
    Users -->|Member of| Groups
    Groups -->|Responsible for| AccountAssignment
    PermissionSets -->|Attached to| AccountAssignment
    AccountAssignment -->|Grants access to| Accounts
    
    style Users fill:#e1f5ff
    style Groups fill:#f3e5f5
    style AccountAssignment fill:#fff3e0
    style PermissionSets fill:#e8f5e9
    style Accounts fill:#fce4ec
```

The module can be found here: [terraform-aws-iam-identity-center](https://registry.terraform.io/modules/davaba86/iam-identity-center/aws/latest)
