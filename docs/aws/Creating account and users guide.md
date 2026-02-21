# Creating account and users

- AWS Accounts - The basics
    
    ## 🧩 **AWS Accounts — Core Concepts Study Notes**
    
    ![image.png](Creating%20account%20and%20users/image.png)
    
    ---
    
    ### 🟦 1. **What an AWS Account *Is***
    
    - An **AWS account = a container** 🧱
        
        It holds:
        
        - **Identities** (users, roles, etc.)
        - **AWS resources** (EC2, S3, VPCs, etc.)
    - **Crucial distinction**:
        
        **AWS Account ≠ Users inside the account**
        
        → Students often confuse the container (account) with the identities created *inside* it.
        
    - Small projects may use **1 account**
    - Enterprises may use **tens to hundreds of accounts**
        
        → For isolation, billing separation, security boundaries, and team segregation.
        
    
    ---
    
    ### 🟩 2. **Creating an AWS Account**
    
    When creating an account, you must provide:
    
    1. **Account name** (e.g., `prod`, `dev`)
    2. **A *unique* email address** 📧
        - ❗ Email **must be unique per account**
        - One email → one AWS account
    3. **A payment method** (usually a credit card)
    
    ✔ The **same credit card can be used for multiple accounts**
    
    ✖ But **email cannot**.
    
    ---
    
    ### 🔑 3. **The Root User**
    
    When the account is created:
    
    - The provided email becomes the **root user** 👑
    - The root user:
        - Has **full, unrestricted access**
        - Cannot be limited or permission-restricted
        - Exists *only for that one account*
    - Example:
        - Production account → Prod root user
        - Developer account → Dev root user
        - They are independent and cannot access each other.
    
    ⚠ **Security Warning**
    
    Root user access = total control → can delete everything in the account.
    
    If compromised → catastrophic results.
    
    ---
    
    ### 💳 4. **Billing Model**
    
    AWS uses a **pay-as-you-go** model:
    
    - Use a resource for 2 minutes → pay for 2 minutes ⏱
    - Billing goes to the **payment method of the AWS account**
    - Many services include **Free Tier** allowances
        
        → used heavily in the course to reduce costs
        
    
    ---
    
    ### 🛡️ 5. **IAM and Security Inside the Account**
    
    AWS provides IAM (**Identity and Access Management**) for creating identities:
    
    | IAM Identity | Purpose |
    | --- | --- |
    | **IAM Users** 👤 | Humans or apps needing long-term credentials |
    | **IAM Groups** 👥 | Collection of users sharing permissions |
    | **IAM Roles** 🎭 | Identity with temporary credentials (services, cross-account, etc.) |
    
    Key facts:
    
    - All IAM identities start with **zero permissions**
        
        → You must explicitly grant access using IAM policies.
        
    - IAM identities belong to **one account** unless you configure cross-account access.
    
    ---
    
    ### 🧱 6. **The Account Boundary**
    
    AWS accounts have a strong **security boundary**:
    
    ### 🔒 What the boundary does:
    
    - Keeps **bad things *inside***
        
        → E.g., if a dev user deletes resources, damage is limited to that account.
        
    - Keeps **unauthorized access *outside***
        
        → By default, everything external is **denied**.
        
    
    ### Key principle:
    
    > By default: Deny All
    > 
    > 
    > Only the root user is exempt.
    > 
    
    You can allow external access (e.g., contractor Julie) but **only by explicit configuration**.
    
    ---
    
    ### 🧨 7. **Why Multiple Accounts Are Important**
    
    Using multiple AWS accounts allows:
    
    - Isolation of **prod vs dev vs test**
    - Limiting blast radius from:
        - Human mistakes
        - Credential leaks
        - Misconfigurations
    - Different billing per team or application
    - Clear governance and compliance boundaries
    
    Common multi-account patterns:
    
    - `prod`
    - `dev`
    - `test`
    - `sandbox`
    - Per-team or per-application accounts
    
    ---
    
    ### 🚀 8. **Why You Must Understand Accounts Deeply**
    
    Whether you're a **Solutions Architect, Developer, SysAdmin, or DevOps engineer**, AWS accounts are a foundational concept.
    
    You will use multiple accounts in the course, mirroring **real-world environments** used by businesses.
    
    ---
    
    ### ⭐ **Core Takeaways**
    
    | Concept | Summary |
    | --- | --- |
    | AWS Account | Container for identities + resources |
    | Root User | Full control, cannot be restricted |
    | IAM Users/Roles | Start with 0 permissions, must be granted access |
    | Security Boundary | Accounts isolate damage + block external access |
    | Billing | Pay-as-you-go, tied to the account's payment method |
    | Email Uniqueness | One email per AWS account, required |
    | Multi-account Strategy | Key to safety, scaling, compliance |
    
    ---
    
- [demo] Creating AWS Account
    
    ## 🧩 **Creating an AWS Account (2025 Free Plan vs Paid Plan)**
    
    ---
    
    ### 🟦 1. **Key Change in 2025**
    
    In mid-2025 AWS introduced **two types of accounts**:
    
    | Plan Type | Features |
    | --- | --- |
    | **Free Plan** 🆓 | • Access to an AWS account• Up to **$200 in AWS credits** (valid 6 months or until consumed)• Monthly free usage for select services• Some AWS services **not available**• **No billing** until you upgrade |
    | **Paid Plan** 💳 | • Full service access• Usage billed normally (pay-as-you-go) |
    
    → Free Plan is ideal for training and demos.
    
    → Upgrading to Paid Plan is possible anytime.
    
    ---
    
    ### 🟩 2. **High-Level Workflow for Creating a Free Plan Account**
    
    1. Start at the **AWS homepage**
        
        → Click **Create Account**
        
    2. **Enter an unused email address**
        - Must be **new** (cannot have been used with AWS before)
        - Enter **account name**
    3. **Verify the email**
        - AWS sends a code → enter it to continue
    4. **Set root user password** 🔐
        - Must meet complexity rules
        - This creates the **root user** for the account
    
    ---
    
    ### 🟧 3. **Choose Account Plan**
    
    You’ll be prompted to choose:
    
    - **Free Plan**
        - Up to $200 credits
        - Some free monthly service allowances
        - No billing occurs unless you upgrade
    - **Paid Plan**
        - Required if you want full AWS service access
    
    → The demo selects **Free Plan** for cost-free course usage.
    
    ---
    
    ### 🟨 4. **Enter Personal Information**
    
    You must provide:
    
    - Name
    - Address
    - Country
    - Agreement checkbox confirming AWS Customer Agreement
    
    Then → **Agree and Continue**
    
    ---
    
    ### 🟥 5. **Payment Verification**
    
    Even free plan accounts require a payment method:
    
    - **Card is used only for identity verification**
    - **No charges occur** unless you upgrade to a paid plan
    - This is a benefit of the free plan
    
    → Enter card details → **Verify and Continue**
    
    ---
    
    ### 🟪 6. **Identity Verification (SMS Recommended)**
    
    Steps:
    
    1. Choose **Text Message**
    2. Enter country code + phone number
    3. Complete security check
    4. Enter SMS code → Continue
    
    ---
    
    ### ⚠️ 7. **Important Restriction: Only One Free Plan Account per Person**
    
    If AWS detects you've used a free plan before, you’ll see a message like:
    
    - **“You cannot sign up for multiple free plan accounts.”**
    
    Options:
    
    - **Proceed with a paid plan** ✔
    - **Or stop if you only want the free plan**
    
    ---
    
    ### 🟦 8. **Choose Support Plan (If Shown)**
    
    Most users see a prompt:
    
    - Select **Basic Support** (free)
    
    Some accounts skip this step automatically.
    
    ---
    
    ### 🟩 9. **AWS Account Creation**
    
    AWS begins provisioning the new account.
    
    You may see a short tutorial → click through it.
    
    After this step:
    
    - Your AWS account is created 🎉
    - Additional **security hardening** is needed (covered in the next lesson)
    
    ---
    
    ### ⭐ **Core Takeaways**
    
    | Concept | Summary |
    | --- | --- |
    | 2025 Change | AWS now offers **Free Plan** accounts with credits |
    | Free Plan Limits | Some services unavailable, no billing until upgrade |
    | Verification | Requires unique email + payment card + phone |
    | One Free Plan Rule | You cannot create multiple free plan accounts |
    | Next Steps | After creation, secure the account (separate lesson) |
    
    ---
    
- Multi-factor Auth (MFA)
    
    ## 🔐 Multi-Factor Authentication (MFA) — Core Concepts & AWS Implementation
    
    ![image.png](Creating%20account%20and%20users/image%201.png)
    
    ---
    
    ## 🌟 Why MFA Is Needed
    
    - Most web logins use **username + password** → both are **knowledge factors** (something you know).
    - If these leak → attackers can impersonate you.
    - MFA adds **more evidence of identity**, making impersonation dramatically harder.
    
    ---
    
    ## 🧩 Authentication Factors
    
    | Factor Type | Meaning | Examples |
    | --- | --- | --- |
    | 🔑 **Knowledge** | Something you *know* | Username, password, PIN |
    | 📱 **Possession** | Something you *have* | Bank card, security token, MFA app |
    | 🧬 **Inherence** | Something you *are* | Fingerprint, face scan, iris, voice |
    | 📍 **Location** | Somewhere you *are* | Physical coordinates, corporate network |
    
    ### Key Idea
    
    - **Single-factor authentication** → only one factor type (usually knowledge).
    - **Multi-factor authentication (MFA)** → two or more factor types.
    - More factors → more security, but less convenience.
    
    ---
    
    ## 🏦 Everyday MFA Example (ATM)
    
    - **Something you have:** bank card
    - **Something you know:** PIN
        
        → Two-factor authentication in the real world.
        
    
    ---
    
    ## 🟦 MFA in AWS: How It Works
    
    ### 1️⃣ Default State
    
    - Logging into AWS uses **username + password** → both are *knowledge* → **single-factor**.
    
    ### 2️⃣ The Risk
    
    - If both credentials leak (phishing, malware, password reuse), attackers can log in as you.
    
    ### 3️⃣ Enabling MFA
    
    AWS supports two main MFA options:
    
    | MFA Type | Description |
    | --- | --- |
    | 🔒 **Physical MFA device** | Hardware key fob generating rotating codes |
    | 📲 **Virtual MFA device** | App like Google Authenticator, Authy, 1Password |
    
    ---
    
    ## 🧪 Behind the Scenes: How AWS Sets Up MFA
    
    ### Step-by-step
    
    1. You activate MFA for an identity (e.g., root user or IAM user).
    2. AWS generates a **secret key** + metadata (username, service name).
    3. AWS encodes this into a **QR code**.
    4. Your MFA app scans the QR code → the secret key is stored inside the app.
    5. The app begins generating **time-based one-time passwords (TOTPs)**:
        - New code every ~30 seconds
        - Based on the shared secret key
        - Never repeats
        - Works offline
    
    ### Virtual MFA App
    
    - Stores multiple virtual MFA entries → one per service or identity.
    - Allows MFA for AWS, Google, Azure, GitHub, banking apps, etc.
    
    ---
    
    ## 🔁 Logging in With MFA
    
    To sign in after MFA is enabled, AWS requires:
    
    1. **Username** (knowledge)
    2. **Password** (knowledge)
    3. **Current MFA code** from your virtual or physical device (possession)
    
    This creates **multi-factor authentication**:
    
    - Even if passwords leak → attacker needs your device.
    - Even if someone steals your phone → attacker needs your password.
    
    ### Extra Security Layer
    
    Modern phones protect the MFA app itself with:
    
    - PIN
    - Fingerprint
    - Facial recognition
    
    So MFA can stack into **multi-level security**.
    
    ---
    
    ## 🛡️ Why MFA Is Effective
    
    - It combines **different factor types**, making identity impersonation extremely difficult.
    - Requires an attacker to compromise multiple independent systems:
        - Credential store
        - Physical device
        - Phone biometric/PIN
    
    ---
    
    ## 📘 Summary Table
    
    | AWS Login Stage | Factor Types Used | Security Level |
    | --- | --- | --- |
    | Username + password | Knowledge | Low |
    | Username + password + MFA code | Knowledge + Possession | High |
    
    ---
    
    ## ✔️ Key Takeaways
    
    - MFA dramatically increases security by combining *multiple* identity proofs.
    - AWS MFA uses **time-based one-time codes (TOTPs)** via physical or virtual devices.
    - QR code = simple way to transfer secret keys to your MFA app.
    - MFA protects your AWS identity even if your password leaks.
    
    ---
    
- [demo] Adding MFA - General Account Root User
    
    ## 🔐 Applying MFA to the AWS Root User
    
    *(Demo lesson summary)*
    
    ---
    
    ## 🎯 Goal
    
    Enable **multi-factor authentication (MFA)** for the **AWS root user**, ensuring that login requires:
    
    - 📧 Root user email
    - 🔑 Password
    - ⏱️ One-time authentication code (TOTP)
    
    This significantly increases account security.
    
    ---
    
    ## 🛠️ Steps to Enable MFA on the Root User
    
    ### 1️⃣ Navigate to Security Credentials
    
    - Click the **account ID dropdown** (top-right).
    - Select **Security credentials**.
    - Verify you're logged in as **root user**.
    - Notice the **best practice warning**: “No MFA assigned.”
    
    ---
    
    ## 2️⃣ Begin MFA Setup
    
    - Click **Assign MFA**.
    - AWS offers three MFA types:
    
    | MFA Type | Description |
    | --- | --- |
    | 🔐 **Passkey / Hardware security key** | FIDO2-based (e.g., YubiKey) |
    | 📱 **Authenticator app (TOTP)** | Google Authenticator, 1Password, ProtonPass |
    | 🧩 **Hardware TOTP token** | Physical device generating rotating codes |
    
    **Recommended for most users:** Authenticator app on a mobile phone.
    
    ---
    
    ## 3️⃣ Configure the MFA Device
    
    1. **Name the MFA device**
        - Use something descriptive:
            - *Example:* `Juan - ProtonPass`
    2. Select **Authenticator app** → **Next**
    3. Open the list of **compatible apps** to confirm yours is supported.
    
    ---
    
    ## 4️⃣ Scan the QR Code
    
    - Click **Show QR code**.
    - Scan it using your authenticator app.
    - The app will start generating 6-digit rotating codes (TOTP), typically every **30 seconds**.
    
    ---
    
    ## 5️⃣ Enter Two Consecutive MFA Codes
    
    - Enter **Code 1** → first box
    - Wait for the next refresh
    - Enter **Code 2** → second box
    - Click **Add MFA**
    
    When successful, the MFA device appears under **Multi-factor authentication** for the root user.
    
    ---
    
    ## 🔐 Testing MFA Login
    
    1. **Logout**
    2. Click **Sign in to console**
    3. Select **Root user**
    4. Enter root user **email**
    5. Enter **password**
    6. When prompted, enter the **current MFA code**
    
    ### Why this is secure
    
    - Even if username & password leak → cannot log in without the rotating MFA code
    - Codes expire every **30 seconds**
    - Attacker must also possess your MFA device/app
    
    ---
    
    ## ⚠️ Important AWS Security Practice
    
    - **Do NOT use the root user for everyday tasks.**
    - The root user cannot be restricted—it's too powerful and too risky for daily use.
    
    ---
    
    ## Next Step (mentioned in lesson)
    
    - Create an **IAM admin user** (e.g., `iam-admin`)
    - Use that IAM user instead of the root user for all future work
    
    *(Covered in a separate video.)*
    
    ---
    
- [demo] Accounts - Creating Budget
    
    AWS Free Tier : [https://aws.amazon.com/free/](https://aws.amazon.com/free/)
    
    ## 💰 AWS Cost Management Essentials
    
    *(Demo lesson summary)*
    
    ---
    
    ## 🎯 Objectives
    
    - Understand the **AWS Free Tier**
    - Explore core **billing tools**
    - Set up a **cost budget** to track spending
    - Learn best practices for monitoring usage during the course
    
    ---
    
    ## 🆓 AWS Free Tier Overview
    
    AWS offers multiple categories of free usage:
    
    | Free Tier Type | Description | Examples |
    | --- | --- | --- |
    | 🎁 **Free Trial** | Short-term promotional access | Certain AI/ML services, new launches |
    | 📅 **12-Month Free Tier** | Free for 12 months from account creation | 750 hrs/month of t2.micro or t3.micro in EC2; 5 GB S3 Standard; 750 hrs of RDS db.t2.micro |
    | ♾️ **Always Free** | No time restriction; limited allocation | Lambda: 1M free requests/month; DynamoDB: 25 GB storage |
    
    🔎 The **AWS Free Tier Overview page** provides the full list of allocations. Students should reference it regularly.
    
    ---
    
    ## 📊 AWS Billing Tools
    
    AWS gives granular visibility into usage and costs:
    
    ### Accessing Billing
    
    1. Log in as **root user**
    2. Click account dropdown → **Billing Dashboard**
    
    ### Key Billing Features
    
    - **Bills**:
        - View current unbilled usage
        - Access previous monthly bills
        - See payments, credits
    - **Cost Explorer**:
        - Visualize usage and spend
        - Month-to-date + forecast
        - Service-level breakdown (S3, EC2, data transfer, etc.)
    
    ---
    
    ## 📨 Billing Preferences Setup
    
    Navigate to: **Billing Dashboard → Billing preferences**
    
    Recommended settings:
    
    | Setting | Purpose |
    | --- | --- |
    | 📄 **Receive PDF invoice by email** | Get the actual invoice delivered—no need to log in |
    | 🎛️ **Free tier usage alerts** | Warns you when you approach free tier limits |
    | 🔔 **Billing alerts** | General alerts about charges or thresholds |
    | ✉️ **Email address** | Required to receive alerts |
    
    ---
    
    ## 📉 Creating a Cost Budget
    
    1. In Billing Dashboard → **Budgets**
    2. Click **Create budget**
    3. Choose **Use a template**
    
    ### Budget Templates
    
    Two main options:
    
    | Template | Best For | Description |
    | --- | --- | --- |
    | 🚫 **Zero Spend Budget** | Staying strictly within the free tier | Alerts you if *any* cost is incurred |
    | 💵 **Monthly Cost Budget** | Allowing limited spend | You specify a maximum monthly amount |
    
    ### Setup Steps
    
    - Give your budget a **name**
    - If using a monthly cost budget → enter a dollar amount (e.g., **$10**)
    - Enter **email recipients** for alerts
    - Click **Create budget**
    
    ⏱️ Spend data may take up to **24 hours** to populate.
    
    ---
    
    ## 🧠 Best Practices
    
    - Review the **Billing Dashboard** regularly
    - Always set at least **one budget**
    - Enable **all billing-related email alerts**
    - For real-world production accounts → budget creation is mandatory for cost control
    
    ---
    
    ## ✔️ Summary
    
    You now know how to:
    
    - Understand AWS Free Tier categories
    - Navigate and interpret billing and usage tools
    - Configure alerts and budgets to prevent unexpected charges
    
    Continue monitoring usage as you progress through the course.
    
- [demo] Creating the Production Account
    
    ## 🏗️ Creating the **Production AWS Account** (Multi-Account Setup)
    
    ![image.png](Creating%20account%20and%20users/image%202.png)
    
    ### 🌐 1. Multi-Account Context
    
    - You already created a **general AWS account**.
    - This lesson adds a second account: a **production AWS account**.
    - Purpose: simulate a **realistic multi-account environment**, which is standard practice in AWS organizations.
    - Some demos (e.g., **cross-account S3 access**) require multiple accounts.
    
    ---
    
    ## 🆕 2. Create a **New Production AWS Account**
    
    ### 📧 Choose a Unique Email
    
    - Each AWS account requires a **unique email address**.
    - ✔️ Use the **Gmail "+" trick**:
        
        ```
        yourname+production@gmail.com
        
        ```
        
    - All variations still deliver to the same inbox.
    
    ### 📝 Sign-Up Process
    
    Use **the same steps and values** as with the general account:
    
    | Field | Recommendation |
    | --- | --- |
    | **Account email** | Unique (using + trick) |
    | **Account name** | `yourname-production` (instead of `yourname-general`) |
    | **Address info** | Same as general account |
    | **Credit card** | Same card allowed |
    | **Support plan** | Same as general account |
    
    After completion, you’ll be logged in to a **fresh AWS production account console**.
    
    ---
    
    ## 🔐 3. Secure the Production Account
    
    Follow the **same security steps** used in the general account.
    
    ### 🛡️ Add MFA to the *root user*
    
    - Use an MFA app: Google Authenticator, Authy, 1Password, etc.
    - **Create a *separate* MFA profile** for this account.
        - Do *not* reuse the MFA token from the general account.
    - Log out + log back in to **test MFA**.
    
    ---
    
    ## 💵 4. Configure Billing Settings
    
    Navigate to the **billing console**.
    
    ### Billing Preferences
    
    Check the same **three checkboxes** as before:
    
    | Setting | Purpose |
    | --- | --- |
    | Receive billing alerts | Stay aware of charges |
    | Receive free tier usage alerts | Avoid accidentally exceeding free tier |
    | Receive reservation & savings plan recommendations | Cost optimization |
    
    Add your email address for alerts.
    
    ### 📊 Budgets
    
    Create a budget **just like in the general account** (usually a cost budget with notifications).
    
    ---
    
    ## 👤 5. IAM Access to Billing
    
    In **Account Settings**:
    
    - Enable **IAM user/role access to billing** (same as general account).
    - Optional: fill in the **3 account contact types** (billing, operations, security).
    
    ---
    
    ## 🔁 6. If Needed, Rewatch Earlier Lessons
    
    - All configuration steps mirror the general AWS account steps.
    - Rewatch prior lessons if a detail is unclear.
    
    ---
    
    ## 🎯 Summary
    
    You're creating a production AWS account **with the same configuration** as your general account, but with:
    
    - A **different email**
    - A **different account name**
    - A **separate MFA profile**
    
    This prepares you for working in a **professional-grade multi-account AWS environment**.
    
- Identity and Access Management (IAM) Basics
    
    ## 🔐 **Identity and Access Management (IAM) — Core Concepts**
    
    ### ⭐ Why IAM Exists
    
    - Every AWS account starts with **one identity**: the **root user**, which has **full, unrestricted access**.
    - ❗ **Problem:**
        - Root credentials can’t be restricted.
        - If compromised, **entire account → all services → all regions** are exposed.
        - Real workflows require granting different permissions to people, teams, and applications.
    
    👉 AWS needs a way to create **more identities** and control **least-privilege access** → **IAM**.
    
    ---
    
    ## 🌍 **What IAM Is**
    
    - IAM = **Identity and Access Management**.
    - Runs as a **dedicated, isolated instance per AWS account**.
    - IAM inside *your account* is trusted by your account and can perform almost anything root can (except a few actions like closing an account or modifying billing).
    - **Global service**:
        - One global IAM database per account
        - **Globally resilient** (important for the exam!)
    
    ---
    
    ## 🧩 **IAM Identities**
    
    ![image.png](Creating%20account%20and%20users/image%203.png)
    
    IAM lets you create **three types of identity objects**:
    
    ### 👤 1. **IAM Users**
    
    - Represent **humans or applications** with specific credentials.
    - Examples:
        - Bob (billing access)
        - Jane (EC2 admin)
        - Backup software (programmatic access)
    
    ### 👥 2. **IAM Groups**
    
    - Collections of users.
    - Used to apply permissions **to multiple users at once**.
    - Examples:
        - `DevelopmentTeam`
        - `FinanceDept`
        - `HRGroup`
    
    ### 🎭 3. **IAM Roles**
    
    - The hardest concept initially.
    - Do **not** have long-term credentials.
    - Assume-based access: entities “take” the role temporarily.
    - Used when the number of entities needing access is **unknown** or **dynamic**.
    
    **Common IAM Role Use Cases:**
    
    - EC2 instances needing permission to access S3
    - Allowing AWS services to act on your behalf
    - Allowing **external accounts** to access your resources
    - Granting access to many or unpredictable entities (e.g., all EC2 instances, all Lambda functions)
    
    | Use Case | Use **User**? | Use **Group**? | Use **Role**? |
    | --- | --- | --- | --- |
    | Login for a specific human | ✅ | Optional | ❌ |
    | Permissions for a team | ❌ | ✅ | ❌ |
    | EC2 instance reading S3 | ❌ | ❌ | ✅ |
    | Temporary access for external account | ❌ | ❌ | ✅ |
    | Application with API keys | ✅ | ❌ | ❌ |
    
    ---
    
    ## 📜 **IAM Policies**
    
    - Policies = JSON documents that **allow or deny** access.
    - Policies do nothing **until attached** to a user, group, or role.
    
    💡 *Think of policies as “permission sets.”*
    
    You attach them to identities to give them abilities.
    
    ---
    
    ## 🔐 What IAM Does (3 Core Jobs)
    
    ### 1️⃣ **Identity Provider (IdP)**
    
    - Create/manage identities:
        - Users
        - Groups
        - Roles
    
    ### 2️⃣ **Authentication**
    
    - Proves **who you are**.
    - Methods: password, access keys, MFA, federated login.
    
    ### 3️⃣ **Authorization**
    
    - Decides **what you're allowed to do**.
    - Based on the **policies** attached to your identity.
    
    ---
    
    ## 🧠 Key Terms to Remember
    
    | Term | Meaning |
    | --- | --- |
    | **Identity Provider (IdP)** | Creates identities |
    | **Authentication** | Prove you are who you claim to be |
    | **Authorization** | Allowed or denied access based on policies |
    | **Security Principal** | The identity making a request |
    
    ---
    
    ## 💰 IAM Cost and Limits
    
    - **IAM is free**
    - Some limits (users, groups, roles, policies), which may matter in real deployments and on the exam.
    
    ---
    
    ## 🌐 IAM Scope
    
    - IAM is **global** (not region-specific).
    - IAM only controls **identities within your account**.
        - ❗ You cannot use IAM to manage identities in *other* AWS accounts directly.
    
    ---
    
    ## 🌉 Identity Federation (Intro Only)
    
    - Lets you use **external identities** (Active Directory, SAML, Google, Facebook, etc.) to access AWS.
    - Covered deeply later — just know the term exists.
    
    ---
    
    ## 🔑 Multi-Factor Authentication (MFA)
    
    - You’ve used app-based MFA already (Google Authenticator, Authy, 1Password).
    - AWS supports **hardware MFA tokens** as well (e.g., physical device generating one-time codes).
    
    ---
    
    ## 🛠️ What’s Next: Create an IAM Admin User
    
    - Root user should only be used for **initial setup**.
    - Next steps:
        - Create an **IAM admin user** with full permissions.
        - Use this admin user to create **all future identities** with **least privilege**.
    
    This ensures a secure, production-grade environment for the rest of the course.
    
- [demo] Creating IAM Admin user and adding MFA
    
    # 👤 **Creating Your First IAM User (Admin) — Study Notes**
    
    ---
    
    ## 🔐 Why Create an IAM User Instead of Using the Root Account?
    
    - **Root user = unlimited, unrestricted access**
        - Cannot be restricted by policies.
        - Best practice: **avoid** using it for daily tasks.
    - **IAM user = controlled permissions**
        - You can limit access according to job/task needs.
        - Safer and more auditable.
    
    👉 For this course, we create an IAM user with **AdministratorAccess**, not the root account.
    
    ---
    
    ## 🛠️ Step 1 — Navigate to the IAM Service
    
    - Go to AWS Console → search for **IAM**.
    - Open IAM in a new tab.
    
    ---
    
    ## 🌐 Step 2 — Create an Easy-to-Use IAM Sign-In URL
    
    IAM users sign in using a special URL.
    
    ### Default Format:
    
    ```
    https://<account-id>.signin.aws.amazon.com/console
    
    ```
    
    ### Customize It (Easier to Remember)
    
    1. Under **Account alias**, click **Create**.
    2. Enter a globally unique alias
        - Example: `candle-aws-training-2026-1`.
    3. New URL becomes:
        
        ```
        https://<alias>.signin.aws.amazon.com/console
        
        ```
        
    4. **Bookmark it** for future use.
    
    ---
    
    ## 👤 Step 3 — Create the IAM User (`iam-admin`)
    
    1. Go to **Users** → **Create user**.
    2. Username: `iam-admin` (unique within your AWS account).
    3. Check: **Provide user access to the AWS Management Console.**
    
    ### 🔑 Password Options
    
    - **Auto-generate password** (quickest)
    - **Custom password** (preferred if using a password manager)
    
    Option: *Require password reset at next sign-in*
    
    - Enable → when creating user for someone else
    - Disable → when creating for yourself
    
    Click **Next**.
    
    ---
    
    ## 🛡️ Step 4 — Assign Permissions (AdministratorAccess)
    
    You can:
    
    - Add user to a group
    - Copy permissions
    - Attach policies directly
    
    For simplicity:
    
    1. Choose **Attach policies directly**
    2. Check **AdministratorAccess**
    3. Click **Next**
    4. Review → **Create user**
    
    ⚠️ **If AWS generated a password, save it now—AWS won’t show it again.**
    
    ---
    
    ## 📱 Step 5 — Enable MFA (Multi-Factor Authentication)
    
    MFA improves account security by requiring a one-time code.
    
    1. Go to **IAM → Users → iam-admin → Security credentials**
    2. Under **MFA**, click **Assign MFA device**
    3. Choose **Authenticator app**
    4. Name the device uniquely (e.g., `andrium-proton-2`)
    5. Scan the QR code using your authenticator app
    6. Enter **two consecutive MFA codes**
    7. Click **Add MFA**
    
    ---
    
    ## 🔐 Step 6 — Test the Login
    
    1. Sign out of AWS.
    2. Open your customized sign-in URL.
    3. Enter:
        - Username: `iam-admin`
        - Password: the one you set
        - MFA code from your app
    4. You should now be logged in as the **IAM admin user**, not root.
    
    ---
    
    ## 🧩 Big Picture (Security Best Practices)
    
    - Create separate IAM identities for different roles/tasks.
    - Grant **least privilege** (minimum permissions required).
    - Admin users should be rare; regular users should have limited scopes.
    - Identity federation (e.g., Azure AD, Okta, Google Workspace) is another login method—covered later.
    
    ---
    
    ## 🎉 Completion Recap
    
    You have successfully:
    
    | Step | Completed |
    | --- | --- |
    | Created AWS account | ✅ |
    | Enabled MFA for root user | ✅ |
    | Created IAM alias login URL | ✅ |
    | Created `iam-admin` user | ✅ |
    | Assigned admin permissions | ✅ |
    | Configured MFA for IAM user | ✅ |
    | Logged in using IAM credentials | ✅ |
    
    This IAM admin user will be your main login throughout the course.
    
- IAM Access Keys
    
    # 🔑 **AWS IAM Access Keys — Study Notes**
    
    ---
    
    ## 🚀 Overview
    
    Up to this point, AWS access has been through the **web console** using:
    
    - Username
    - Password
    - MFA
    
    This lesson introduces a different authentication method used for:
    
    - **Command Line Interface (CLI)**
    - **Applications using AWS APIs**
    
    That method = **IAM Access Keys**.
    
    ---
    
    ## 🧩 What Are Access Keys?
    
    Access keys are **long-term credentials** for IAM users.
    
    ### Used For:
    
    - CLI access
    - Programmatic access (SDKs, APIs)
    
    ### Not Used For:
    
    - AWS Console login (uses username + password instead)
    
    ---
    
    ## 🏷️ Long-Term Credentials vs Console Credentials
    
    | Credential Type | Used For | Parts | Max Per IAM User | Rotation |
    | --- | --- | --- | --- | --- |
    | **Username + Password** | AWS Console | Public: usernamePrivate: password | 1 | Manual password change |
    | **Access Keys** | CLI / API | Public: **Access Key ID**Private: **Secret Access Key** | 2 | Create new pair + delete old |
    
    📌 Both serve as **authentication credentials**, but access keys are specifically for programmatic use.
    
    ---
    
    ## 🔒 Public vs Private Credential Parts
    
    Every AWS credential has:
    
    - **Public part** (safe to reveal):
        - Username
        - Access Key ID
    - **Private part** (must remain secret):
        - Password
        - Secret Access Key
        - MFA code
    
    If someone gets both the public + private part ⇒ **Credential leak**.
    
    ---
    
    ## 🗝️ Access Key Structure
    
    Each access key has **two components**:
    
    1. **Access Key ID** (e.g., `AKIAxxxxxxxxxxxxxx`)
        - Public
        - Comparable to a username
    2. **Secret Access Key**
        - Private
        - Comparable to a password
        - Shown **only once** at creation time
    
    📥 **AWS will never show the secret access key again.**
    
    You must save it securely when it's created (password manager, secrets manager, etc.).
    
    ---
    
    ## 🔁 Why Can IAM Users Have Two Sets of Access Keys?
    
    Because **rotation** requires overlap.
    
    ### 🔄 Access Key Rotation Process
    
    1. Create a **second** access key pair.
    2. Update all apps/CLI tools to use the new pair.
    3. Delete the **old** access key.
    
    This minimizes downtime.
    
    ---
    
    ## 🛑 If You Lose the Secret Access Key
    
    You **cannot** recover it.
    
    You must:
    
    1. Delete the access key (ID + secret pair)
    2. Create a new one
    
    ---
    
    ## 🚫 IAM Users vs Roles (Important Distinction)
    
    - **IAM users**
        - Can have access keys
        - Use long-term credentials
    - **IAM roles**
        - **Do NOT** use access keys
        - Use short-term credentials generated through STS
        - Explained in depth later
    
    🔹 Root user *can* have access keys, but AWS strongly recommends **never** using root access keys.
    
    ---
    
    ## 🧪 What Happens If You Disable or Delete Access Keys?
    
    - Any CLI tools or applications using them will stop working
    - They must be updated with:
        - reactivated keys **or**
        - newly generated keys
    
    ---
    
    ## 🛠️ What’s Coming Next
    
    You’ll:
    
    - Create access keys for the **general** and **production** AWS accounts
    - Install AWS CLI tools
    - Configure multiple profiles using access keys
    - Use CLI to authenticate into both AWS accounts
    
    These access keys are required for that setup.
    
    ---
    
- [demo] Creating Access Keys and settings up AWS CLI v2 tools
    
    ## 🔐 Creating & Configuring AWS Access Keys + CLI (Step-by-Step Notes)
    
    ---
    
    ## 🧩 **1. Goal of the Lesson**
    
    - Create **access keys** for:
        - IAM admin user in the **General AWS account**
        - IAM admin user in the **Production AWS account**
    - Install the **AWS CLI v2**
    - Configure **named profiles** so the CLI can authenticate to both accounts.
    
    ---
    
    ## 🔑 **2. Creating Access Keys (General AWS Account)**
    
    ### 📌 Steps to create access keys
    
    1. Log in as **IAM admin** in the General account.
    2. Click the **account dropdown → Security credentials**.
    3. Scroll to **Access keys**, select **Create access key**.
    4. Choose **Command Line Interface (CLI)** as the use case.
    5. Check the confirmation box → **Next**.
    6. Provide a description (e.g., `local CLI IAM admin - general`).
    7. Click **Create access key**.
    
    ### 📄 What you get
    
    You’re shown:
    
    - **Access Key ID** (public, like a username)
    - **Secret Access Key** (private, like a password)
    
    ⚠️ **You only see the Secret Access Key this one time.
    If you lose it, you must delete & recreate the key.**
    
    ### 📥 Save keys
    
    - Click **Download CSV**.
    - Rename file to:
        
        **`iam_admin_access_keys_general.csv`**
        
    
    ---
    
    ## 🔁 **3. Access Key Management**
    
    | Action | Notes |
    | --- | --- |
    | **Deactivate** | Suspends key; can be reactivated. |
    | **Activate** | Reactivates a suspended key. |
    | **Delete** | Requires deactivation first; completely removes key. |
    | **Max keys** | IAM users can have **2 access keys** (active/inactive). |
    
    Used for **key rotation**:
    
    1. Create 2nd key
    2. Update apps/CLI to new key
    3. Delete old key
    
    ---
    
    ## 🔑 **4. Repeat Process for Production AWS Account**
    
    - Log in as the Production IAM admin (via another browser profile or log out/in).
    - Create access keys the same way.
    - Download and rename file to:
        
        **`iam_admin_access_keys_production.csv`**
        
    
    ---
    
    ## 🖥️ **5. Install AWS CLI v2**
    
    Use the provided AWS installation page.
    
    ### Supported OS:
    
    - Linux
    - macOS
    - Windows
    
    ### Install:
    
    - Use the installer package for your OS
    - Accept all defaults
    - After install, verify:
    
    ```bash
    aws --version
    ```
    
    Should return something like:
    
    ```
    aws-cli/2.x.x
    ```
    
    ---
    
    ## ⚙️ **6. Configure AWS CLI with Named Profiles**
    
    The CLI uses credential profiles stored in:
    
    - `~/.aws/credentials`
    - `~/.aws/config`
    
    ### ✔️ Named profiles allow switching between accounts.
    
    ---
    
    ## 🔧 **7. Configuring General Account CLI Profile**
    
    ### Command:
    
    ```bash
    aws configure --profile iam_admin-general
    
    ```
    
    CLI will prompt for:
    
    1. **Access Key ID** → from CSV (before comma)
    2. **Secret Access Key** → from CSV (after comma)
    3. **Default region** →
        
        ```
        us-east-1
        
        ```
        
    4. **Default output format** → Press **Enter** (none)
    
    ### Test the profile:
    
    ```bash
    aws s3 ls --profile iam_admin-general
    
    ```
    
    Expected output:
    
    - **Empty list** (✓ OK)
    - **Error** (✗ credentials likely mistyped)
    
    To fix:
    
    ```bash
    aws configure --profile iam_admin-general
    
    ```
    
    ---
    
    ## 🔧 **8. Configure Production Account CLI Profile**
    
    Same process:
    
    ```bash
    aws configure --profile iam_admin-production
    
    ```
    
    Then test:
    
    ```bash
    aws s3 ls --profile iam_admin-production
    
    ```
    
    ---
    
    ## ☣️ **9. Critical Security Warning**
    
    - Anyone with **Access Key ID + Secret Access Key** can act as that IAM user.
    - IAM admin = **full account control** → severe risk if leaked.
    - Never show keys publicly (in this lesson, the instructor invalidates them immediately).
    
    ### 🔥 Best practice:
    
    - Delete the downloaded CSV files after configuration.
    - Keep keys only inside the CLI credential store.
    
    ---
    
    ## 🎯 **10. End State**
    
    You should now have:
    
    ### ✔️ AWS CLI v2 installed
    
    ### ✔️ Two named profiles configured:
    
    - `iam_admin-general`
    - `iam_admin-production`
    
    These will be used throughout the course for all CLI tasks.