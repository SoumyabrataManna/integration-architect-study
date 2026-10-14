# Boomi Platform Settings: SSO, SAML, Roles, Groups and User Management (Q&A)



  

## Section 1: Single Sign-On (SSO) & SAML Configuration (10 Questions)

### Q1: What is Single Sign-On (SSO) in Boomi AtomSphere, and how does it work?

Single Sign-On in Boomi AtomSphere allows users to authenticate once through an external Identity Provider (IdP) and gain access to the Boomi platform without entering separate Boomi credentials. Boomi supports SAML 2.0 (Security Assertion Markup Language) as its SSO protocol.

The flow works as follows: A user navigates to the Boomi AtomSphere login URL. Instead of entering a Boomi username and password, they are redirected to their organization’s Identity Provider (such as Okta, Azure AD/Entra ID, Ping Identity, or OneLogin). The user authenticates at the IdP. The IdP generates a SAML assertion containing the user’s identity attributes (like email, name, and Federation ID) and sends it back to Boomi’s Assertion Consumer Service (ACS) endpoint. Boomi validates the SAML assertion against the configured certificate, extracts the Federation ID, matches it to a Boomi user account, and grants access.

SSO is configured in Boomi under Settings > Account Information and Setup > Security Options > SSO Options. The key benefit is centralized authentication management — when an employee leaves, disabling their IdP account immediately revokes their Boomi access. This is critical for enterprise clients who need compliance with SOX, HIPAA, or GDPR access control requirements.

### Q2: Walk through the step-by-step process of configuring SAML SSO in Boomi AtomSphere.

Configuring SAML SSO in Boomi involves both the Boomi platform side and the Identity Provider side. Here are the detailed steps:

On the Boomi Platform Side: Log in to Boomi AtomSphere as an account administrator. Navigate to Settings > Account Information and Setup > Security Options > SSO Options tab. Check “Enable SAML Single Sign-On”. Note the AtomSphere Sign In URL and the AtomSphere Metadata URL — you’ll need these for the IdP configuration. Configure the Federation ID Location — this tells Boomi where to find the user’s unique identifier in the SAML assertion. Options are: “Federation ID is in NameID element of the Subject” or “Federation ID is in FEDERATION_ID Attribute element.” Set the Name ID Policy (typically Unspecified or Email Address). Set the SAML Authentication Context (typically Password Protected Transport). Set the SAML Authentication Context Comparison Level (typically Exact). Configure Assertion Encryption setting (typically Unencrypted unless your security policy requires encrypted assertions). Import the Identity Provider’s signing certificate (in .cer or .der format — if the IdP provides .pem, convert it first). Enter the Identity Provider Sign In URL. Optionally configure the Sign Out Redirect URL for proper logout behavior. Click Save.

On the Identity Provider Side: Create a new SAML application for Boomi. Set the Assertion Consumer Service (ACS) URL to the Boomi AtomSphere Sign In URL. Set the Audience/Entity ID using the value from Boomi’s metadata. Configure attribute statements to include the user’s Federation ID, email, first name, and last name. Assign users or groups who should have access.

The final step is ensuring each Boomi user has a Federation ID configured in their user profile (under User Management) that matches what the IdP sends in the SAML assertion.

### Q3: What is the Federation ID in Boomi SSO, and where can it be located in the SAML assertion?

The Federation ID is the unique identifier that links a user’s Identity Provider account to their Boomi AtomSphere user account. It’s the “bridge” between the IdP and Boomi — when Boomi receives a SAML assertion, it extracts the Federation ID and looks up the corresponding Boomi user.

The Federation ID can be located in one of two places in the SAML assertion, configured via the “Federation ID Location” setting in Boomi’s SSO Options:

Option 1 — NameID element of the Subject: The Federation ID is extracted from the standard SAML NameID element. This is the simpler approach and works well when the IdP sends the user’s email or a unique identifier as the NameID. The SAML assertion looks like: <saml:Subject><saml:NameID>user@company.com</saml:NameID></saml:Subject>.

Option 2 — FEDERATION_ID Attribute element: The Federation ID is extracted from a custom SAML attribute named FEDERATION_ID. This approach is used when the NameID is already being used for something else, or when you need a different identifier as the Federation ID. The SAML assertion includes: <saml:Attribute Name="FEDERATION_ID"><saml:AttributeValue>user@company.com</saml:AttributeValue></saml:Attribute>.

Each Boomi user must have their Federation ID configured in their user profile under Settings > User Management. The Federation ID value in Boomi must exactly match what the IdP sends in the SAML assertion. A common troubleshooting issue is case sensitivity — if the IdP sends “User@Company.com” but Boomi has “user@company.com”, the match will fail.

### Q4: How do you configure Boomi SSO with Microsoft Entra ID (Azure AD)? What are the key configuration points?

Microsoft Entra ID (formerly Azure AD) is one of the most common IdPs for Boomi SSO in enterprise environments. Here’s the configuration:

In Microsoft Entra ID: Sign in to the Microsoft Entra admin center. Navigate to Entra ID > Enterprise Applications > New Application. Search for “Boomi” in the gallery and add it. Go to Single sign-on and select SAML. In Basic SAML Configuration, you can upload Boomi’s Service Provider metadata file (downloaded from Boomi’s AtomSphere Metadata URL) to auto-populate the Identifier and Reply URL. Alternatively, manually set: Identifier (Entity ID) to the Boomi Metadata URL, Reply URL (ACS) to the Boomi Sign In URL, and Sign-on URL to your Boomi account URL (e.g., https://platform.boomi.com/AtomSphere.html#build;accountId={your-accountId}). Download the Federation Metadata XML or the Certificate (Base64) for import into Boomi. Copy the Login URL for the Identity Provider Sign In URL in Boomi.

In Boomi AtomSphere: Navigate to SSO Options. Enable SAML Single Sign-On. Import the certificate downloaded from Entra ID (convert to .cer/.der if needed). Paste the Entra ID Login URL as the Identity Provider Sign In URL. Set Federation ID Location to “Federation ID is in FEDERATION_ID Attribute element.” Set SAML Authentication Context to “Password Protected Transport.” Save.

Back in Entra ID: Assign users and groups to the Boomi enterprise application. Configure user provisioning if desired (Entra ID supports SCIM provisioning to Boomi). Test the SSO connection using the “Test” button.

A critical consideration: Entra ID’s Conditional Access policies can add MFA requirements for Boomi access, providing an additional security layer without any Boomi-side configuration. This is a major enterprise security advantage.

### Q5: What are the SAML Authentication Context options in Boomi, and when would you choose each?

The SAML Authentication Context in Boomi specifies the type of authentication the Identity Provider must perform before issuing the SAML assertion. It’s essentially telling the IdP: “The user must authenticate using at least this method.”

Boomi provides several Authentication Context options:

Password Protected Transport: This is the most commonly used setting. It means the user authenticates with a username and password over a protected (TLS/SSL) channel. This maps to the SAML context class urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport. I recommend this for most enterprise deployments.

Password: Authentication via username and password without requiring transport-level protection. Less secure than Password Protected Transport. Rarely used in production.

Kerberos: For environments using Kerberos authentication (common in Windows domain environments). This enables integrated Windows authentication where users logged into their corporate domain are automatically authenticated.

Unspecified: No specific authentication method is required. The IdP decides the authentication method. Use this when the IdP manages its own authentication policies (like Conditional Access in Azure AD that may enforce MFA) and you don’t want Boomi to dictate the method.

The Authentication Context Comparison Level works alongside this setting. “Exact” means the IdP must use exactly the specified context. “Minimum” means the IdP must use at least the specified context or a stronger one. For most deployments, I use Password Protected Transport with Exact comparison.

In practice, for clients using Azure AD or Okta with MFA policies, I often recommend “Unspecified” context with the IdP managing the actual authentication strength. This way, the IdP’s MFA and Conditional Access policies are respected without conflict from Boomi’s context requirement.

### Q6: What happens to existing Boomi users when SSO is enabled? How do you handle the transition?

Enabling SSO on a Boomi account requires careful planning because it changes how all users authenticate. Here’s what happens and how to manage the transition:

Impact on Existing Users: When SSO is enabled, all non-administrator users are expected to authenticate through the IdP. Their existing Boomi username/password credentials are still technically present but the login flow redirects them to the IdP. Each existing user must have a Federation ID configured in their Boomi user profile that matches their identity at the IdP. If the Federation ID is missing or doesn’t match, the user will be locked out.

Critical Requirement — Keep at Least One Non-SSO Admin: Boomi requires that at least one administrator account remains accessible without SSO. This is your “break glass” account for managing SSO configuration itself. If SSO is misconfigured and all admins are SSO-only, you’d be locked out with no way to fix it. Always maintain a local admin account with a strong password stored securely.

Transition Steps: First, inventory all current Boomi users and their email addresses. Second, ensure each user exists in the IdP (Azure AD, Okta, etc.) with matching email/Federation ID. Third, configure the Federation ID on each Boomi user profile before enabling SSO. Fourth, enable SSO in a controlled manner — I recommend doing this during a maintenance window. Fifth, test with a pilot group of 2-3 users before communicating to the full team. Sixth, communicate the change to all users with clear instructions on the new login process.

Unlocking Locked-Out SSO Users: If a user gets locked out after SSO is enabled (common during initial rollout), an administrator can unlock them from Settings > User Management. Boomi provides an “Unlock SSO User” option specifically for this scenario.

### Q7: How do you troubleshoot SSO authentication failures in Boomi?

SSO troubleshooting is a critical skill because authentication failures can lock out entire teams. Here’s my systematic approach:

Common Failure Scenarios and Diagnosis:

Federation ID Mismatch: The most common issue. The Federation ID in the SAML assertion doesn’t match what’s configured in the Boomi user profile. Check the user’s Federation ID in Settings > User Management. Compare it with what the IdP is sending. Use a SAML debugging tool (like SAML-tracer browser extension or the IdP’s SAML response logs) to inspect the actual assertion being sent.

Certificate Issues: If the IdP’s signing certificate has been rotated or expired, Boomi will reject the SAML assertion because it can’t validate the signature. Solution: import the updated certificate in Boomi’s SSO Options. This is a common issue that catches teams off guard — I recommend setting calendar reminders for certificate expiration dates.

Clock Skew: SAML assertions have time-based validity windows (NotBefore and NotOnOrAfter). If there’s a significant time difference between the IdP server and Boomi’s servers, assertions may be rejected as expired. Ensure NTP synchronization on all servers.

Incorrect ACS URL: If the Assertion Consumer Service URL configured in the IdP doesn’t exactly match Boomi’s expected URL, the assertion will be rejected. Verify the URL matches character-for-character.

Wrong Federation ID Location: If Boomi is configured to look for the Federation ID in the NameID element but the IdP is sending it as an attribute (or vice versa), authentication fails silently.

Debugging Steps: Enable SAML debugging in the IdP to capture the full assertion XML. Use browser developer tools (Network tab) to capture the SAML response during the redirect. Decode the Base64-encoded SAML response and inspect the XML for the Federation ID value, certificate, timestamps, and audience restriction. Check Boomi’s audit logs for SSO-related error messages.

My troubleshooting checklist: (1) Verify Federation ID match, (2) Verify certificate validity, (3) Verify URL configuration, (4) Check clock synchronization, (5) Inspect raw SAML assertion.

### Q8: Can Boomi SSO be configured with multiple Identity Providers? How do you handle multi-IdP scenarios?

Boomi AtomSphere’s native SSO configuration supports a single SAML Identity Provider per account. This is a platform limitation that you need to design around in complex enterprise scenarios.

Multi-IdP Scenarios and Workarounds:

Scenario 1 — Merged Organizations: After an acquisition, users from two companies need Boomi access but authenticate against different IdPs (e.g., Company A uses Okta, Company B uses Azure AD). The solution is to use an Identity Federation Hub or IdP proxy. Tools like Azure AD B2B, Okta’s Identity Engine, or PingFederate can federate multiple upstream IdPs into a single SAML assertion endpoint. Boomi sees one IdP, but behind the scenes, users from either company can authenticate.

Scenario 2 — Internal Users vs. External Partners: Internal team members use corporate AD, but external consultants or partners use a different IdP. Again, the IdP federation hub approach works here. Alternatively, you can create separate Boomi accounts for internal and external teams (using Boomi’s Account Groups to share resources) with each account configured with its own IdP.

Scenario 3 — Boomi API Management Specific: As of December 2025, Boomi API Management (API Control Plane) now supports its own SAML configuration separate from the main platform SSO. This means you can configure a different SAML IdP for API Management than what’s used for the core Integration platform. This is useful when API developers and integration developers come from different organizations.

It’s also important to note that there is not currently single sign-off between the Boomi Platform, API Control Plane, and Cloud API Management. Logging out of one doesn’t automatically log you out of the others.

### Q9: What is the Boomi Platform API for SSO configuration, and how would you automate SSO setup?

Boomi provides the AccountSSOConfig Platform API object that allows you to programmatically manage SSO configuration. This is valuable for partners or consultants who need to configure SSO across multiple client accounts programmatically.

The AccountSSOConfig API supports these operations: GET — retrieve the current SSO configuration for an account, Bulk GET — retrieve SSO configurations for up to 100 accounts, UPDATE — modify the SSO configuration, and DELETE — remove the SSO configuration.

The API object includes fields for: enabled (boolean), fedIdFromNameId (boolean — whether Federation ID comes from NameID), idpUrl (the Identity Provider Sign In URL), certificate (the IdP signing certificate in Base64), certInfo (certificate metadata), and accountId.

This is particularly useful in large partner scenarios where the same IdP is being configured across dozens of managed accounts. Instead of manually navigating the UI for each account, the automation script handles it in minutes.

The API can be called via SOAP or REST (JSON), and authentication uses the standard Boomi platform API credentials (username + API token). For enterprise clients, I recommend using a dedicated service account for API-based administration rather than individual user credentials.

### Q10: What is Assertion Encryption in Boomi SSO, and when should you enable it?

Assertion Encryption adds an additional layer of security to the SSO process by encrypting the SAML assertion payload. By default, Boomi uses “Unencrypted” assertions where the SAML response is signed but not encrypted.

When Assertion Encryption is enabled, the IdP encrypts the SAML assertion using Boomi’s public key (from Boomi’s metadata). Only Boomi’s private key can decrypt it. This means even if someone intercepts the SAML response in transit (beyond TLS), they cannot read the assertion contents.

When to Enable Encryption: For compliance-sensitive environments where regulations explicitly require encrypted assertions (certain financial services, government, healthcare contexts). When transmitting sensitive attributes in the SAML assertion (beyond just the Federation ID — for example, group memberships, roles, or employee IDs). When the organization’s security policy mandates defense-in-depth with both TLS and assertion-level encryption.

When Unencrypted is Acceptable: For most enterprise deployments where the SAML exchange occurs over TLS/HTTPS (which is always the case with Boomi’s cloud platform), unencrypted assertions are considered secure. The SAML response is already signed by the IdP, so tampering is detected. TLS protects the transport layer.

Configuration Impact: Enabling encryption requires that the IdP supports SAML assertion encryption and is configured with Boomi’s public key from the SP metadata. This adds complexity to the IdP setup and can make troubleshooting harder because you can’t easily inspect encrypted assertions with browser debugging tools.

### Q11: How does SSO interact with Boomi’s API authentication? Can API calls use SSO tokens?

This is an important architectural distinction. SSO (SAML) and Boomi’s Platform API authentication are separate mechanisms and serve different purposes.

SSO/SAML is for interactive user authentication — when a human logs into the Boomi AtomSphere web interface through a browser. The SAML flow involves browser redirects, IdP login pages, and session cookies.

Boomi Platform API authentication uses either: Basic Authentication (username + password or username + API token), or a SAML assertion token obtained through a separate API token exchange. For automated integrations and CI/CD pipelines, API tokens are the recommended approach.

The API Token Approach: An administrator generates an API token for a user in Settings > User Management. This token acts as the password for API calls. The user’s username (typically their email) and the API token are sent as Basic Auth credentials. This is the standard approach for CI/CD tools, deployment automation, and external systems calling Boomi APIs.

SAML Token for API: Boomi does support obtaining a SAML assertion token through the API for programmatic authentication. However, this is rarely used because API tokens are simpler and more suitable for automation.

Best Practice for Enterprise Clients: Use SSO/SAML for all interactive (human) access to the Boomi platform. Use API tokens for all programmatic access (CI/CD, automation scripts, external system calls). Create dedicated service accounts for API access with minimal required privileges. Rotate API tokens on a regular schedule (I recommend every 90 days).

  

## Section 2: User Roles & Privileges (10 Questions)

### Q12: What are the built-in user roles in Boomi Enterprise Platform, and what access does each provide?

Boomi Enterprise Platform provides four built-in roles with progressively restricted access:

Administrator: The most powerful role with full access to all platform capabilities. Administrators can manage users, configure account settings, create and edit integration processes, deploy to all environments, manage runtime Atoms/Molecules, configure API Management, access all security settings, and manage account groups. By default, the user who registers the Boomi account is granted the Administrator role. Administrators can access Settings, User Management, SSO configuration, and all governance functions.

Standard User: The default role for non-admin users. Standard Users can create and edit integration processes, deploy components (subject to environment restrictions), view process logs and execution history, and manage connectors. However, Standard Users cannot manage other users, modify account settings, configure SSO, or access account administration features. This is the appropriate role for most integration developers.

Production Support: A restricted role designed for operations teams who need to monitor production integrations without modifying them. Production Support users can view process execution status, access logs and process reporting, restart failed executions, and view environment configurations. They cannot edit process designs, deploy new processes, or modify environment settings. This role is part of Advanced User Security, available in Enterprise and Enterprise Plus editions.

Support: The most restrictive built-in role. Support users have read-only access across the platform. They can view processes, logs, and configurations but cannot execute, deploy, or modify anything. This is ideal for auditors, compliance teams, or external support personnel who need visibility without any ability to change configurations.

The key privilege difference between Standard User and Administrator is user management and account administration. If a user is not an administrator, they cannot add or remove users, change roles, or modify account-level settings.

### Q13: What are Custom Roles in Boomi, and how do you create them? Give practical examples.

Custom Roles allow administrators to create granular access control beyond the four built-in roles. You can combine specific privileges to create roles tailored to your organization’s needs.

Creating a Custom Role: Navigate to Settings > Account Access > User Management > Custom Roles tab. Click the Add icon. Enter a role name and description. In the “Inherits Role” dropdown, select a base role to inherit from (or “Does not inherit a role” to start from scratch). Select additional privileges from the Available Privileges list. Click OK.

Practical Custom Role Examples:

Example 1 — “Developer (No Data View)”: For developers working with sensitive data (PII, financial). Start with no inherited role. Add Build - Read and Write access, Build - Read and Write access with Deployment, and Execute privileges. Do NOT add the View Process Reporting Data privilege. This allows the developer to build, test, and deploy processes but prevents them from viewing the actual data being processed — critical for HIPAA or SOX compliance.

Example 2 — “Read-Only Environment Access”: For operations support needing visibility without modification rights. Inherit from Support role. Add the Environment Management - Read Access privilege. Assign this role to specific environments via the Roles With Access configuration. Users can view schedules, properties, and execution logs but cannot modify anything.

Example 3 — “Deployment Manager”: For release managers who deploy but don’t develop. Add Build - Read Access (view only), Build - Read and Write access with Deployment, and Runtime Management. Users can deploy packaged components to environments but cannot edit process designs.

Note: Production Support and Support roles and the Custom Roles feature are part of Advanced User Security, available in Enterprise and Enterprise Plus editions, or as an add-on to Professional editions. Contact your Boomi sales representative to enable this feature.

### Q14: How does the Advanced Environment Access feature work, and why is it critical for enterprise governance?

Advanced Environment Access is an account-level feature that enables per-environment role-based access control. It’s one of the most important governance features for enterprise Boomi deployments.

How It Works: When enabled (in Settings > Account > Features), administrators can assign custom roles with either Environment Management Read Access or Environment Management Full Access to specific environments. This creates environment-level RBAC where different users have different permissions in different environments.

Configuration Steps: First, create custom roles with either Read Access or Full Access privileges in Settings > User Management > Custom Roles. Second, assign these roles to users. Third, navigate to Manage > Runtime Management, select an environment, and add the custom roles in the “Roles With Access” field.

Practical Example: Consider a team with developers and a production support team. Create a “Developer” custom role with Full Access privilege. Create a “Production Viewer” custom role with Read Access privilege. Assign the Developer role to the DEV and QA environments. Assign the Production Viewer role to the PROD environment. Result: Developers have full control in DEV and QA (deploy, execute, modify schedules, update extensions) but can only view PROD (review logs, view configurations, but cannot deploy or modify).

Important Rules: When Advanced Environment Access is toggled ON, it enforces the read/write roles configured per environment. The most restrictive setting wins — if a user has Runtime Management at Full Access but Environment Access at Read Only for a particular environment, Read Only applies. It is highly discouraged to assign both Environment Management and Environment Management Full Access privileges simultaneously because Environment Management overrides both.

When toggled OFF, the account defaults to the standard Environment Management privilege behavior without per-environment granularity.

### Q15: Explain the privilege hierarchy in Boomi. What are the key privileges, and how do they interact?

Boomi’s privilege system is hierarchical, and understanding how privileges interact is crucial for designing effective access control.

Core Build Privileges (from most to least permissive): Build - Read and Write Access: User can create, edit, and delete components (processes, connectors, maps). Build - Read and Write Access with Deployment: Everything above plus the ability to deploy components to environments. Build - Read Access: View-only access to all components. Users can open and inspect processes but cannot save changes. A read-only banner displays when they open components.

Runtime Privileges: Runtime Management: Full control over runtime environments — deploy, execute, manage schedules, update extensions, view logs. Execute - Execute or Retry: Can execute processes and retry failed documents but cannot deploy or modify configurations.

Environment-Specific Privileges: Environment Management - Full Access: Read and write to assigned environments. Environment Management - Read Access: View-only for assigned environments. These work only when Advanced Environment Access is enabled.

Data and Reporting Privileges: View Process Reporting Data: Allows viewing actual document data in process execution logs. Without this privilege, users can see that processes executed but cannot view the data payloads.

User Management Privileges: User Management - Full: Can add, edit, delete users and assign roles. User Management - Limited: Can add users and assign roles but with restricted management capabilities. Used by API Control Plane Administrator roles.

API Management Privileges: API Manager, API Infrastructure Manager, API Administrator, API Support — each with specific read/write/execute permissions for API Management features.

Key Interaction Rule: When multiple privileges or roles apply to the same area, the most restrictive setting wins. For example, a user with both full Runtime Management and Environment Read Access for a specific environment will have read-only access in that environment.

### Q16: How do you design a role-based access control strategy for a large Boomi implementation with multiple teams?

For a large enterprise Boomi deployment serving multiple teams, I design RBAC around the principle of least privilege with clear separation of duties. Here’s my framework:

Step 1 — Identify User Personas: Integration Developer: Builds and tests processes in DEV/QA. Senior Developer/Lead: Builds, tests, and promotes to UAT. Deployment Manager/Release Manager: Deploys packaged components to PROD. Operations/Production Support: Monitors PROD, reviews logs, restarts failed executions. Administrator: Manages users, accounts, SSO, and platform configuration. API Developer: Works specifically with API Management. Auditor/Compliance: Read-only access across all environments for audit purposes.

Step 2 — Map Personas to Custom Roles: Developer Role: Build Read/Write, Execute, Runtime Management. Restrict to DEV/QA environments via Advanced Environment Access. Senior Developer Role: Everything Developer has plus Deployment privilege. Restrict to DEV/QA/UAT environments. Release Manager Role: Build Read Access (view only), Deployment privilege, Runtime Management. Assign Full Access to PROD environment only. Production Support Role: Inherit from built-in Production Support. Add Environment Management Read Access for PROD. Operations Role: Execute and Retry privileges plus Runtime Management. PROD-only with Full Access. Auditor Role: Build Read Access, Environment Management Read Access across all environments.

Step 3 — Implement Environment Segregation: Enable Advanced Environment Access in Account Features. Create at least 4 environments: DEV, QA, UAT, PROD. Assign roles to environments as described above.

Step 4 — Governance Controls: Require all PROD deployments to go through the Release Manager role. Maintain at least 2 Administrators (one as break-glass). Review role assignments quarterly. Document the RBAC design in the client’s governance framework.

This design ensures developers cannot touch production, production support cannot modify development work, and auditors can see everything without changing anything.

### Q17: How does Boomi handle API Management-specific roles and privileges?

Boomi API Management has its own set of roles and privileges that are separate from (but integrated with) the core Integration platform roles. This is important because API Management users often have different access needs than integration developers.

Built-in API Management Roles:

API Manager: Read and write access to APIs and Applications. Can create, publish, and manage API proxies and developer applications. This is the role for API developers and product owners.

API Infrastructure Manager: Read and write access to Gateways, Plans, and Authentication. Can configure API gateways, define rate limiting plans, set up authentication sources, and publish the Developer Portal. This is the infrastructure/operations role.

API Administrator: Full read and write access to all API Management pages, including the Dashboard. This is the combined role for full API Management control.

API Support: Read-only access to all API Management pages plus the specific ability to download Gateway logs. Designed for troubleshooting teams who need to diagnose API issues without modifying configurations.

Additional API Control Plane Roles: API Control Plane Administrator: Administrative functions in the newer API Control Plane. Includes User privilege and User Management Limited privileges. Automatically granted to platform Administrators. API Control Plane User: Can access both the Administrator Portal and Developer Portal. Cloud API Management Administrator: Administrative access to Cloud API Management. Cloud API Management User: Access to Cloud API Management functions.

Key Consideration: These roles cannot be used in custom roles and cannot be extended. The standard platform Administrator role automatically receives the API Control Plane Administrator and Cloud API Management Administrator roles.

### Q18: What are Account Groups in Boomi, and how do they relate to user management?

How Account Groups Work: A primary account administrator creates an Account Group and adds managed accounts to it. The primary account can then share resources with managed accounts through the group. Shareable resources include: runtime clouds (for partners hosting client runtimes), private connectors (custom-built connectors), custom roles, published processes (reusable process templates), integration packs, and models.

Custom Role Sharing: This is particularly relevant to user management. When a primary account creates a custom role, it can be shared with all accounts in an Account Group via the Resources tab. Users in managed accounts can then be assigned the shared custom role. Key behavior: if the privileges in the custom role are changed in the primary account, those changes propagate to all shared accounts immediately (server-side, requires page refresh on client). If the primary account removes an account from the group, any users with shared custom roles lose that role. Custom roles can be removed by the primary account admin, but only after the role has been unassigned from all users in managed accounts.

Important Note: If Advanced User Security is disabled in either the primary or managed account, existing shared custom roles remain available but no additional roles can be shared.

### Q19: How do you handle user provisioning and deprovisioning in Boomi for enterprise clients?

User lifecycle management in Boomi spans three areas: provisioning (creating users), ongoing management (role changes), and deprovisioning (removing access). In an enterprise context, this must be automated and integrated with the organization’s HR/identity systems.

Manual Provisioning (Small Teams): Navigate to Settings > User Management. Click the Add icon. Enter the user’s email address and name. Select the role (Administrator, Standard User, Production Support, Support, or a custom role). If SSO is enabled, configure the Federation ID. Click Save. An email with login instructions is sent to the user.

Automated Provisioning with SCIM: For large enterprises, Boomi supports SCIM (System for Cross-domain Identity Management) provisioning through IdPs like Azure AD/Entra ID and Okta. When a user is added to the Boomi application group in the IdP, SCIM automatically creates the user account in Boomi. When a user is removed from the group, SCIM deactivates or deletes the user. Role assignments can be mapped through SCIM attributes.

Deprovisioning Best Practices: Immediate Access Revocation: For SSO-enabled accounts, disabling the user in the IdP immediately prevents login. This is the fastest way to revoke access. Boomi Account Cleanup: Navigate to User Management, locate the user, click the three-dot icon, and select Deactivate or Delete. Deactivate preserves the user record for audit purposes; Delete removes it entirely. API-Based Deprovisioning: Use the Boomi Platform API (User object) to programmatically deactivate users as part of an automated offboarding workflow.

Governance Recommendations: Conduct quarterly access reviews comparing Boomi user lists against active employee directories. Implement an automated sync between HR systems and the IdP to ensure timely deprovisioning. Use SSO exclusively (disable direct Boomi login) so that IdP deprovisioning immediately revokes access. Document all user additions and removals for audit compliance.

### Q20: What is the Agent Garden in Boomi, and what roles govern access to it?

Agent Garden is a newer capability in the Boomi platform related to Boomi’s AI and agentic integration direction. It provides a conversational interface where users can interact with AI agents that have been built and deployed using Boomi’s Agent Designer and Agent Control Tower.

Agent Garden Roles: There are three roles that govern access:

Agent Garden User: This is the default role assigned to all Standard platform users. It allows users to interact with installed/deployed agents in the conversational interface and see a list of all deployed agents. Think of this as the “consumer” role — users can use agents but not create or modify them.

Agent Garden Developer (Agent Developer): This role allows users to create, edit, test, and manage agents in addition to using them. Developers can access the Agent Designer to build agent workflows, configure agent behaviors, and test them before deployment. This is the “builder” role.

Agent Garden Administrator: Full administrative access to the Agent Garden, including managing agent deployments, configuring providers, and controlling which agents are available to users. All platform Administrators automatically receive this role.

Agent Control Tower Access: Users with Administrator or Developer roles can access the Agent Control Tower, which manages agent providers and connections. Each provider (like AI model services) has its own prerequisites that must be met.

For enterprise governance, I’d recommend restricting the Agent Developer role to a small team of approved developers and keeping Agent Garden User as the default for business users who need to interact with deployed agents.

### Q21: How do you audit and monitor user access and activities in Boomi for compliance purposes?

Audit and compliance monitoring in Boomi spans several capabilities that together provide a comprehensive picture of who did what and when.

Built-in Audit Capabilities:

Audit Log: Boomi maintains an audit log that records significant account events including user logins and logouts, component changes (who created/modified/deleted processes), deployment events (who deployed what to which environment), user management changes (role assignments, user additions/removals), and configuration changes. Administrators can access audit logs through the platform.

Process Reporting: Records every process execution with details on execution time, status (success/failure), documents processed, and the user or schedule that triggered the execution. Combined with the “View Process Reporting Data” privilege, this provides data-level audit trailing.

User Management Page: Shows all current users, their roles, last login dates, and Federation IDs. This is the primary tool for access reviews.

Compliance Monitoring Practices I Implement:

Quarterly Access Reviews: Export the user list from User Management and cross-reference with the active employee directory. Flag any users who have left the organization or changed roles. Review whether current role assignments still align with job functions (principle of least privilege). Document the review for audit evidence.

Privileged Access Monitoring: Track who has Administrator access and justify each instance. Monitor API token usage for service accounts. Review custom role definitions for privilege creep.

Integration with Enterprise SIEM: For clients with enterprise security requirements, I configure Boomi alerts to feed into their SIEM platform (Splunk, Azure Sentinel, etc.) for centralized security monitoring. Process failure alerts, unusual execution patterns, and configuration changes can all trigger SIEM events.

SOX/HIPAA/GDPR Specific Controls: For SOX compliance: document all PROD deployment approvals and maintain separation of duties between developers and deployers. For HIPAA: ensure the “View Process Reporting Data” privilege is restricted to authorized personnel only. For GDPR: implement data access logging and ensure deprovisioning aligns with data retention policies.

  

## Section 3: Account Settings, Security & Governance (10 Questions)

### Q22: Walk through the Boomi Settings page structure. What are the key tabs and what does each control?

The Boomi Settings page is the central administration hub, accessed via Settings under the Account ID in the top menu bar. The available tabs depend on your role and platform edition. Here are the key sections:

Account Information and Setup: Contains the Account ID, account name, time zone settings, and the Security Options section which houses the SSO Options tab for SAML configuration. This is where you manage the fundamental account identity.

Account Access — User Management: The primary tab for managing users, roles, and privileges. Contains subtabs for Users (add/edit/remove users, assign roles, configure Federation IDs), Custom Roles (create and manage custom roles with granular privileges), Hub User Entitlements (for Boomi DataHub-specific roles — visible only for MDM-entitled accounts), and Hub Role Entitlements.

Account Features: Controls advanced settings like Advanced Environment Access (per-environment RBAC), API Management features, and other account-level toggles. This is where administrators enable features that affect the entire account.

Runtime Management (under Manage): While not technically on the Settings page, this is closely related. It’s where you assign custom roles to specific environments when Advanced Environment Access is enabled.

API Management Settings: If the account has API Management entitlements, additional tabs for API gateway configuration, authentication sources, developer portal settings, and API-specific role assignments.

Key Governance Point: Not all tabs are visible to all users. Standard Users see a limited Settings page. Only Administrators see the full suite including User Management, SSO Options, and Account Features. This is by design — the platform enforces role-based visibility on the administration interface itself.

### Q23: How does Boomi handle password policies and account security for non-SSO users?

For accounts that don’t use SSO (or for the break-glass admin accounts that bypass SSO), Boomi enforces several security controls:

Password Requirements: Boomi enforces minimum password complexity including length, character types (uppercase, lowercase, numbers, special characters). Password expiration policies can be configured at the account level. Previous password reuse restrictions prevent users from cycling back to old passwords.

Multi-Factor Authentication (MFA): Boomi supports MFA for direct platform login. Administrators can mandate MFA for all users in the account. This adds a second verification step (typically TOTP via an authenticator app) beyond the password.

Account Lockout: After a configurable number of failed login attempts, the user account is temporarily locked. Administrators can unlock users from the User Management page.

API Tokens: For programmatic access, Boomi supports API tokens as an alternative to passwords. Tokens can be generated per user and should be rotated regularly. Best practice is to use API tokens exclusively for automation and never embed actual user passwords in scripts.

Session Management: Boomi manages session timeouts for inactive users. After the configured inactivity period, users are automatically logged out.

### Q24: How do you manage Boomi access across a multi-account enterprise structure?

Large enterprises often have multiple Boomi accounts — for different business units, regions, or environments (separate accounts for production vs. non-production). Managing access across these accounts requires a structured approach.

Boomi’s Multi-Account Structure: Account Groups allow a primary account to manage relationships with multiple sub-accounts. A single user email can have access to multiple Boomi accounts with different roles in each. Users switch between accounts using the account selector in the top navigation.

Access Design Patterns:

Pattern 1 — Centralized Administration: The primary account’s administrators manage all user access across all accounts. Custom roles are created in the primary account and shared via Account Groups. SSO is configured identically across all accounts pointing to the same IdP.

Pattern 2 — Federated Administration: Each account has its own administrators for day-to-day user management. The primary account provides shared resources (roles, connectors, published processes) but doesn’t directly manage users. SSO is still centralized through the corporate IdP.

Pattern 3 — Environment-Based Accounts: Non-production (DEV, QA) in one account, Production in another. This provides the strongest separation but requires users to have access to both accounts. Deployment promotion requires cross-account packaged component deployment.

Best Practices: Standardize role definitions across accounts using shared custom roles from the primary account. Use a consistent Federation ID (email address) across all accounts so SSO works seamlessly. Maintain an access matrix documenting which users have access to which accounts and with what roles. Automate user provisioning/deprovisioning across all accounts using the Platform API.

### Q25: How does Boomi DataHub (MDH) user management differ from the core platform?

Boomi DataHub (Master Data Hub) has its own layer of user entitlements that sits on top of the standard platform roles. This is because MDH involves data stewardship workflows where different users need different levels of access to master data domains and records.

MDH-Specific Entitlements: Hub User Entitlements and Hub Role Entitlements tabs appear in the User Management page only for accounts with MDM - Stewardship Management privilege enabled. These tabs allow administrators to define granular access to MDH functions.

MDH User Roles: Data Stewards: Can review, approve, or reject data changes within their assigned domains. They work within the MDH Stewardship Portal to manage data quality. Data Administrators: Full access to MDH configuration including domain setup, source configuration, match rules, and data model management. Data Viewers: Read-only access to master data records for reporting and validation purposes.

Key Differences from Core Platform: Domain-Level Access Control: MDH supports assigning users to specific data domains (e.g., Customer, Product, Vendor). A steward might have approval rights for Customer data but not Product data. Record-Level Workflows: MDH has approval workflows where data changes go through review cycles. User roles determine who can initiate, review, and approve changes. Source System Trust Rankings: Administrators configure which source systems are authoritative for which fields. This is a configuration privilege, not a user access issue, but it affects data stewardship workflows.

Integration with Platform Roles: A user must first have a platform role (at minimum Standard User) before MDH entitlements can be assigned. The platform role governs access to the Boomi UI; the MDH entitlements govern what the user can do within the DataHub module specifically.

### Q26: What is the Sign Out Redirect URL in Boomi SSO, and why is it important?

The Sign Out Redirect URL is an optional but important configuration in Boomi’s SSO Options. It specifies where the user is redirected after they log out of Boomi AtomSphere.

How It Works: When a user clicks “Log Out” in Boomi, the platform terminates the Boomi session. If a Sign Out Redirect URL is configured, the user’s browser is redirected to that URL. Typically, this URL points to the IdP’s logout endpoint, which terminates the IdP session as well.

Why It Matters:

Without Sign Out Redirect: The user logs out of Boomi, but their IdP session remains active. If someone else uses the same browser and navigates to Boomi, the existing IdP session may automatically authenticate them into the previous user’s Boomi account. This is a security risk, especially on shared workstations.

With Sign Out Redirect to IdP Logout: The user logs out of Boomi and is redirected to the IdP’s logout page, which terminates both sessions. This ensures a clean logout across both systems.

Important Limitation: There is not currently single sign-off between the Boomi Platform, API Control Plane, and Cloud API Management. Logging out of the main Boomi Platform does not automatically log you out of the API Control Plane or Cloud API Management. Users must log out of each separately. This is a known limitation that clients should be aware of.

Configuration: Set the Sign Out Redirect URL to your IdP’s single logout endpoint. For Azure AD/Entra ID, this is typically: https://login.microsoftonline.com/{tenant-id}/oauth2/logout. For Okta: https://{org}.okta.com/login/signout.

### Q27: How do you handle the scenario where a client wants different Boomi modules accessible to different teams?

This is a common enterprise scenario where the integration team, API team, data team, and business process team all use different Boomi modules but need appropriate isolation.

Boomi Module Landscape: Boomi Integration (process design and execution), Boomi API Management (API proxy creation and management), Boomi DataHub/MDH (master data management), Boomi Flow (workflow and low-code applications), and the newer Agent Garden/Agent Designer (AI agents).

Access Segregation Strategy:

Integration Team: Assign Standard User role with Build Read/Write and Deployment privileges. Grant API Management privileges only if they need to expose processes as APIs. No MDH entitlements unless they’re building MDH integrations.

API Team: Assign API Manager and/or API Infrastructure Manager roles. Grant Standard User with Build Read Access (view integration processes that power APIs, but can’t modify them). No MDH entitlements.

Data Team: Assign MDH-specific entitlements through Hub User Entitlements. Grant Standard User with restricted Build access (can view MDH-related processes). No API Management privileges.

Operations Team: Assign Production Support role (or custom role) with Environment Read Access across all modules. Can monitor but not modify anything.

Implementation Approach: Create custom roles that combine the appropriate privileges for each team persona. Use environment-level access control to further restrict which environments each team can access. Document the access matrix clearly so that role assignments are auditable.

The challenge is that Boomi’s role system is primarily designed around the Integration module, with API Management, MDH, and Flow adding their own privilege layers. This means a single user might need a combination of platform role + API Management role + MDH entitlements. I maintain a privilege matrix spreadsheet for each client that maps every team member to their exact set of privileges across all modules.

### Q28: What are Boomi’s IP restriction and network security settings for platform access?

Boomi provides several network-level security controls that complement user-level access management:

IP Allowlisting: Administrators can configure IP address restrictions that limit which network addresses can access the Boomi platform. This is configured at the account level and applies to all users. Only users connecting from approved IP ranges can log in. This is particularly useful for restricting access to corporate networks or VPN IP ranges.

Atom Security: For on-premises Atoms and Molecules, network security involves ensuring that the Atom can communicate outbound to Boomi’s cloud management console (typically over HTTPS port 443). Inbound traffic to Atoms should be restricted to only necessary ports (like HTTP listener ports for webhook processes). Atom-to-Atom communication in Molecules requires specific ports to be open between nodes.

API Gateway Network Security: Boomi API Gateway deployments can be configured with IP-based access controls, mutual TLS (mTLS) for client certificate authentication, and integration with Web Application Firewalls (WAF) for additional protection.

HTTPS/TLS: All communication between the browser and Boomi’s cloud platform is encrypted with TLS. Atom communication with AtomSphere uses TLS. API traffic through Boomi gateways uses TLS.

My Enterprise Security Recommendation: Layer 1 — Network: Configure IP allowlisting to restrict platform access to corporate networks. Layer 2 — Authentication: Enable SSO with MFA through the corporate IdP. Layer 3 — Authorization: Implement custom roles with least-privilege access and Advanced Environment Access. Layer 4 — Monitoring: Enable audit logging and integrate with enterprise SIEM. This defense-in-depth approach ensures that even if one layer is compromised, the others provide protection.

### Q29: How do you handle Boomi access for external consultants or temporary team members?

Access Restrictions for External Users: Use environment-specific roles — give consultants Full Access to DEV/QA but Read Access (or no access) to PROD. Do NOT give consultants Administrator access unless absolutely necessary. Create a dedicated “External Consultant” custom role with appropriate privileges. Restrict the “View Process Reporting Data” privilege if consultants shouldn’t see actual business data.

Deprovisioning Process: When the engagement ends, immediately deactivate all consultant user accounts. If SSO is used, remove the consultant’s guest access from the IdP. Remove API tokens associated with consultant accounts. Document the deprovisioning for audit purposes.

### Q30: Explain the Boomi account hierarchy: what are primary accounts, sub-accounts, and managed accounts?

Boomi’s account hierarchy supports complex organizational structures and partner/client relationships:

Primary Account: The top-level account, typically owned by the organization or partner. It has full administrative control and can create Account Groups. The primary account administrator can share resources (roles, connectors, processes, runtime clouds) with managed accounts.

Sub-Accounts: Created under the primary account for organizational separation. Sub-accounts can have their own users, processes, environments, and configurations. They operate independently for day-to-day integration work but can receive shared resources from the primary account.

Managed Accounts: Accounts that are added to an Account Group managed by the primary account. These can be entirely separate Boomi accounts (with their own subscription) that agree to be managed by the primary account. The primary account can share resources but does not have direct user management access to managed accounts unless explicitly configured.

Enterprise Scenario: A large corporation has a primary Boomi account for the integration CoE. Each business unit has a sub-account. The CoE shares governance standards (custom roles, naming conventions via published process templates), centralized connectors, and integration patterns. Each business unit manages its own day-to-day development and operations.

Key Points: Users can have access to multiple accounts with different roles in each. Account switching is done through the account selector in the Boomi UI. SSO can be configured independently per account, though most enterprises use the same IdP. Licensing is typically at the account level, though partner/enterprise agreements may have umbrella licensing structures.

### Q31: What security considerations should you present to an enterprise client when setting up their Boomi governance framework?

People — Identity and Access Management: Implement SSO with MFA through the corporate IdP (Azure AD, Okta, or Ping). Design a role matrix mapping job functions to Boomi custom roles with least privilege. Enable Advanced Environment Access to enforce per-environment RBAC. Restrict Administrator access to 2-3 designated platform administrators. Establish quarterly access review cadence with documented evidence. Define onboarding and offboarding procedures with SLA for access revocation.

Process — Change and Deployment Management: Enforce separation of duties — developers cannot deploy to production. Require deployment approvals through a formal change management process. Maintain version control of all Boomi processes with external Git backup. Document all production changes in a change log with approver, timestamp, and business justification. Conduct regular DR testing of Boomi configurations.

Platform — Technical Security Controls: Configure IP allowlisting to restrict platform access. Enable audit logging and integrate with enterprise SIEM. Encrypt all data in transit (TLS on all connections). Restrict the “View Process Reporting Data” privilege to minimize exposure of sensitive data. Use API tokens (not passwords) for all programmatic access and rotate them quarterly. Configure Sign Out Redirect URL for proper session termination.

Compliance Alignment: For SOX: document all privileged access, enforce separation of duties, maintain audit trail of deployments. For HIPAA: implement data access controls, restrict PII visibility, enable audit logging. For GDPR: ensure data residency compliance in Atom deployment locations, implement data minimization in integrations. For SOC 2: demonstrate access control effectiveness, maintain incident response procedures.

Page
