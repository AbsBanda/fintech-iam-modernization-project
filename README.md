# FinTech Labs IAM Modernization Project

## Project Overview

This project demonstrates how FinTech Labs can transition from a traditional Castle-and-Moat security model toward a Zero Trust, identity-centric security architecture using AWS Identity and Access Management (IAM).

The project covers identity classification, least-privilege access design, Separation of Duties, AAA security analysis, and a hands-on AWS implementation using IAM and Amazon S3.

## Technologies and Security Concepts

- Amazon Web Services (AWS)
- AWS Identity and Access Management (IAM)
- Amazon S3
- Zero Trust Security
- Identity as the Perimeter
- Principle of Least Privilege
- Separation of Duties (SoD)
- Role-Based Access Control concepts
- AAA: Authentication, Authorization, and Accounting
- IAM Users and User Groups
- Custom IAM Policies
- JSON

---

## Part 1 - Identity Taxonomy and Risk Analysis

The first stage was to identify the different types of identities operating within the FinTech Labs environment.

| Identity | Identity Type | Example Security Risk |
|---|---|---|
| Sarah - Software Engineer | Workforce Identity | Excessive access to source code or production resources |
| Payment-Gateway-API-Key | Non-Human / Workload Identity | Stolen API credentials could be used to impersonate the application |
| Alex - Customer Service Representative | Workforce Identity | Unauthorized access to customer or support data |
| Lambda-Log-Processor | Non-Human / Workload Identity | An overly permissive execution role could provide unnecessary access to cloud resources |

This demonstrated that cloud security must protect both human and non-human identities.

---

## Part 2 - Least-Privilege Access Matrix

A least-privilege access model was designed for three job functions.

| Identity | Development Code | Production Database | IAM Console |
|---|---|---|---|
| Sarah - Software Engineer | Read/Write | None | None |
| Bob - Database Administrator | Read | Read/Write | None |
| Dave - DevOps Engineer | Read | Read | Read/Write |

The matrix applies Separation of Duties by preventing developers from modifying production database resources and preventing database administrators from pushing development code.

In a real production environment, IAM administrative permissions would be further restricted to only the actions required by the DevOps role.

---

## Part 3 - AAA Forensic Audit Analysis

A security log recorded activity performed using the `DevOps-Deployment-Role`.

### Identification and Authentication

The recorded principal was an IAM role. This shows that role credentials were used to perform the action.

However, the event alone does not prove whether the role was assumed by a human user or by a workload. Additional CloudTrail role-assumption events would be required to determine the original identity.

### Authorization

The recorded action completed successfully, demonstrating that the effective AWS permissions allowed the operation.

A successful event does not mean that AWS permits the action by default. AWS follows an implicit-deny model. An applicable permission must allow the requested action unless another control explicitly denies it.

The provided log alone does not identify the exact IAM policy statement responsible for granting the permission.

### Accounting and Forensics

The activity occurred at approximately `02:14 UTC` and originated from an external source IP address.

These details should be compared against:

- Approved deployment windows
- Normal developer working hours
- Corporate network ranges
- VPN addresses
- Approved CI/CD infrastructure
- Known cloud automation sources

If the timestamp or source IP does not match normal organizational activity, the event could represent anomalous or potentially unauthorized use of the deployment role.

The timestamp and IP address alone are not sufficient to prove that an account was compromised.

---

## Part 4 - Zero Trust Transition Strategy

Traditional Castle-and-Moat security assumes that users and systems inside the corporate network can be trusted after crossing the network perimeter. This model is no longer sufficient for cloud environments because applications, data, employees, APIs, and workloads operate across multiple networks and locations. If an attacker obtains valid credentials, a firewall alone may not prevent access to cloud resources.

FinTech Labs should therefore move toward a Zero Trust model where identity becomes a key security perimeter. Every request should be authenticated and authorized based on the identity making the request and the permissions required for the task.

AWS IAM supports this approach through least-privilege policies, role-based access patterns, MFA, Separation of Duties, and temporary credentials. Access to sensitive resources such as production databases should be restricted to explicitly authorized identities.

Continuous logging and monitoring should also be used to identify unusual access patterns and investigate suspicious activity.

By combining strong identity verification, least privilege, and continuous monitoring, FinTech Labs can reduce unnecessary access and limit the potential impact of compromised credentials.

---

# Hands-On AWS Implementation

The theoretical access-control design was then implemented and tested in AWS.

## Step 1 - Create S3 Resources

Two S3 buckets were created to represent separate development and production environments:

- `fintech-dev-code-an1982`
- `fintech-prod-data-an1982`

Public access remained blocked to prevent unintended public exposure.
![FinTech S3 Buckets](01-fintech-s3-buckets.png)
## Step 2 - Create the Software Engineer Policy

A custom IAM policy named:

`FinTech-SoftwareEngineer-Policy`

was created.

The policy allows the software engineering identity to list AWS S3 buckets while granting object access only to the development code bucket.

It does not grant access to the production data bucket.
![Software Engineer IAM Policy](02-software-engineer-policy.png)
## Step 3 - Create the DBA Policy

A second custom IAM policy named:

`FinTech-DBA-Policy`

was created.

This policy grants the database administrator access to the production data bucket while excluding access to the development code bucket.
![DBA IAM Policy](03-dba-policy.png)
## Step 4 - Configure IAM Groups

Two IAM groups were created:

- `SoftwareEngineers`
- `DatabaseAdmins`

The appropriate custom IAM policy was attached to each group.

This allows permissions to be managed through group membership rather than assigning permissions directly to individual users.
![Software Engineers Group](04-software-engineers-group.png)

![Database Admins Group](05-database-admins-group.png)
## Step 5 - Create IAM Users

Two IAM users were created:

- `sarah-dev`
- `bob-dba`

`sarah-dev` was added to the `SoftwareEngineers` group.

`bob-dba` was added to the `DatabaseAdmins` group.

## Step 6 - Test Sarah's Authorized Access

Sarah signed in using her own IAM identity and accessed the development S3 bucket.

A test file was successfully uploaded to:

`fintech-dev-code-an1982`

This confirmed that the Software Engineer policy permitted the required development activity.
![Sarah Development Upload Success](06-sarah-dev-upload-success.png)
## Step 7 - Test Sarah's Production Restriction

Sarah then attempted to access:

`fintech-prod-data-an1982`

AWS returned an insufficient permissions message for `s3:ListBucket`.

This confirmed that Sarah could see the bucket name but could not list or access its production objects.
![Sarah Production Access Denied](07-sarah-prod-access-denied.png)
## Step 8 - Test Bob's Authorized Access

Bob signed in using the `bob-dba` identity.

He successfully accessed the production data bucket and uploaded a test object.

This confirmed that the DBA policy provided the required production access.
![Bob Production Upload Success](08-bob-prod-upload-success.png)
## Step 9 - Test Bob's Development Restriction

Bob attempted to access:

`fintech-dev-code-an1982`

AWS denied the `s3:ListBucket` operation.

This demonstrated that the database administrator could not access the development code environment.
![Bob Development Access Denied](09-bob-dev-access-denied.png)
---

## Test Results

| IAM User | Development Code Bucket | Production Data Bucket |
|---|---|---|
| `sarah-dev` | ✅ Read/Write tested successfully | ❌ Access denied |
| `bob-dba` | ❌ Access denied | ✅ Read/Write tested successfully |

The results demonstrate that authentication alone does not provide access to AWS resources. IAM authorization determines which actions an authenticated identity can perform.

---

## Security Outcomes

The implementation successfully demonstrated:

- Principle of Least Privilege
- Separation of Duties
- Identity-based access control
- Group-based permission management
- Authentication vs Authorization
- AWS implicit deny
- Segregation of development and production resources
- Testing of both authorized and unauthorized actions

## Key Lessons Learned

This project demonstrated how identity can function as a security perimeter in a cloud environment.

The most important lesson was that successfully authenticating to AWS does not automatically authorize access to resources. Permissions must be explicitly granted through IAM policies.

It also demonstrated why both successful and unsuccessful access attempts should be tested when validating a least-privilege architecture.

## Conclusion

The FinTech Labs IAM modernization project demonstrates how Zero Trust principles can be translated into practical AWS security controls.

By combining identity-based authorization, least privilege, Separation of Duties, and access testing, organizations can reduce excessive permissions and limit the potential impact of compromised identities.
