# 0 - Concepts & Fundamentals

#### Root account vs IAM user

* The **root** account is the main account created when you open an AWS account. Some actions can only be performed with it. It must **never** be used for day-to-day work or in production, since it has full, unrestricted access to every resource in the account. It must obviously be protected with a strong password + MFA.
* If the root account gets compromised, it's a disaster: recovering it means proving and justifying many things, which can be very hard if nobody knows who originally created the account (for example, a former employee who has since left). Once the AWS account is set up, IAM users take over.

#### IAM (Identity and Access Management)

In an AWS account, IAM is the service that controls who can do what on the account's resources. It handles authentication and authorization through **users**, **groups**, **roles** and **policies**, to precisely control access to the various AWS services (S3, EC2, VPC, etc.).

**For example:**

* Giving a developer read-only access to an S3 bucket.
* Allowing an EC2 instance to write data to CloudWatch.
* Letting an administrator manage all resources without using the root account.

**Note:** everything you can do from the AWS console can also be done through the API. The console is just one way among others to interact with the same underlying features. For this introduction to AWS, both approaches are covered.

#### Support plans

AWS offers several support plans:

* **Basic**, included with every account (along with the **Free Tier**, to try services with limited free usage).
* **Developer**, aimed at developers.
* **Business**, with technical support from AWS staff.

Paid support remains useful in production, for billing issues or technical problems that require direct help from AWS.

#### Account ID

A 12-digit identifier shared by the root account and by every IAM user and role created within that account.

#### Multi-account organization (AWS Organizations & Identity Center)

AWS Organizations lets you create several accounts for different use cases: one for consolidated billing, one for development, one to host a specific workload, etc. IAM Identity Center then centralizes user access across these accounts.

#### Permissions and tags

AWS provides **managed policies**: ready-to-use sets of permissions. Every resource created in AWS is a resource in its own right, to which rules and permissions can be applied.

**Note: the `AdministratorAccess` policy grants full administrative rights. For an IAM user to access billing, IAM access to the Billing console must also be enabled from the root account.**

**Tags**, on the other hand, are labels attached to a resource. They are useful to tell environments apart (development, staging, production) and to break down costs.

### Access key best practices

* Never store one in plain text, whether in a code repository or in source code. If a key is ever pushed to a GitHub/GitLab repository, even for one minute, assume it is compromised and revoke it.
* Deactivate or delete it once it is no longer needed.
* Apply the principle of least privilege when granting permissions.
* Rotate access keys regularly, to limit the impact of a leak.