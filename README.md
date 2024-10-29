### Introduction
In today’s enterprise environments, ensuring seamless yet secure access to applications is a priority. Integrating Azure Active Directory (Azure AD) Single Sign-On (SSO) with Wazuh allows organizations to enhance user experience by enabling centralized authentication. By implementing SSO, users can access Wazuh using their Azure AD credentials, reducing the need to manage separate accounts and passwords. This integration enhances productivity, simplifies IT management, and strengthens security.

### Objectives
The primary objectives of implementing Azure AD SSO for Wazuh are:

1. **Streamline Access Management**: Provide a single, unified login experience for users across various platforms, reducing the number of credentials they need to remember and improving productivity.
   
2. **Strengthen Security Posture**: Leverage Azure AD’s advanced security features, such as Multi-Factor Authentication (MFA) and Conditional Access, to secure access to the Wazuh platform.
   
3. **Enhance Compliance and Auditability**: Centralize authentication through Azure AD, making it easier to audit access logs and enforce compliance policies.
   
4. **Simplify User and Group Management**: Assign and manage access permissions from Azure AD, enabling easy onboarding/offboarding and role-based access control.

### Security Requirements
To ensure a secure and reliable SSO integration between Azure AD and Wazuh, several security requirements should be met:

1. **Azure AD Premium License**: Access to Azure AD Premium (P1 or P2) is required to enable Single Sign-On and Conditional Access features.
   
2. **Secure SAML Configuration**:
   - Configure **SAML Signing Certificate** in Azure AD to ensure secure communication.
   - **Entity ID and Reply URL** must be correctly set in both Azure AD and Wazuh to prevent unauthorized access.
   
3. **Certificate Management**: The SAML certificate downloaded from Azure AD must be securely stored and referenced in Wazuh. Proper permissions and access controls should be in place to protect the certificate.
   
4. **Multi-Factor Authentication (MFA)**: Enable MFA in Azure AD to provide an additional layer of security during the login process.
   
5. **Conditional Access Policies**: Use Conditional Access policies to restrict access based on device compliance, location, and sign-in risk, minimizing the risk of unauthorized access.

6. **Periodic Review of Access**: Regularly review user assignments and group memberships in Azure AD to ensure that only authorized personnel have access to Wazuh.

Here’s an in-depth, step-by-step guide for setting up Azure AD SSO for Wazuh, with detailed configurations for each part of the process:

---

### Step 1: Register Wazuh in Azure AD

1. **Login to Azure Portal**: Go to [Azure Portal](https://portal.azure.com/) and log in with an account that has permissions to create applications in Azure AD.

2. **Navigate to Azure AD**:
   - In the left-hand navigation, select **Azure Active Directory**.
   - Under **Manage**, select **App registrations**.
   - Click on **New registration**.

3. **Create App Registration**:
   - **Name**: Enter a descriptive name for your application (e.g., “Wazuh SSO”).
   - **Supported account types**: Choose **Single tenant** if only users within your organization should access Wazuh, or **Multitenant** if external users should also be able to log in.
   - **Redirect URI**: Select **Web** and enter:
     ```
     https://<your_wazuh_domain>/auth/realms/<realm_name>/broker/azure/endpoint
     ```
     - Replace `<your_wazuh_domain>` with the domain or IP of your Wazuh instance.
     - Replace `<realm_name>` with the Wazuh realm name (use `wazuh` for the default realm).
   - Click **Register** to complete the app registration.

4. **Save App Details**:
   - After registration, you’ll see your **Application (client) ID** and **Directory (tenant) ID** on the **Overview** page. Note these down; they’ll be used later for the SAML configuration in Wazuh.

5. **Create a Client Secret** (optional but recommended for security):
   - Under **Certificates & secrets**, click **New client secret**.
   - Enter a **Description** (e.g., “Wazuh Secret”) and select an **Expiration**.
   - Click **Add**, and copy the secret value (you won’t be able to see it again after you leave this page).

---

### Step 2: Configure SAML SSO in Azure AD

1. **Open the SSO Settings for Your Application**:
   - Go to **Azure Active Directory** > **Enterprise Applications**.
   - Select your application (e.g., “Wazuh SSO”) from the list.
   - Under **Manage**, select **Single sign-on** and choose **SAML**.

2. **Basic SAML Configuration**:
   - Click **Edit** in the **Basic SAML Configuration** section.
   - Enter the following details:
     - **Identifier (Entity ID)**: 
       ```
       https://<your_wazuh_domain>/auth/realms/<realm_name>
       ```
     - **Reply URL (Assertion Consumer Service URL)**:
       ```
       https://<your_wazuh_domain>/auth/realms/<realm_name>/broker/azure/endpoint
       ```
     - **Logout URL** (optional): 
       ```
       https://<your_wazuh_domain>/auth/realms/<realm_name>/protocol/openid-connect/logout
       ```
     - Replace `<your_wazuh_domain>` and `<realm_name>` as specified.
   - Click **Save** to confirm.

3. **Download the SAML Certificate**:
   - Scroll to the **SAML Signing Certificate** section.
   - Click **Download** next to **Certificate (Base64)**. This file will be used to configure Wazuh later.

4. **Edit User Attributes and Claims** (optional, for custom user mappings):
   - In the **Attributes & Claims** section, click **Edit**.
   - By default, Azure AD sends the **userPrincipalName** as the **Name ID**. You can add more claims if Wazuh requires specific user attributes.

5. **Configure Group Mapping** (optional, if roles are managed by groups):
   - If Wazuh uses roles based on groups, add group claims here by clicking on **Add group claim** and selecting the appropriate group configuration.

---

### Step 3: Configure Wazuh for Azure AD SSO

1. **Access the Wazuh Configuration File**:
   - On the Wazuh server, open the **wazuh.yaml** file. The file path is usually:
     ```
     /etc/wazuh/wazuh.yaml
     ```

2. **Configure SAML Settings**:
   - Within the configuration file, locate or add an `auth` section with the following settings:
     ```yaml
     auth:
       saml:
         enabled: true
         entity_id: "https://<your_wazuh_domain>/auth/realms/<realm_name>"
         sso_url: "https://login.microsoftonline.com/<tenant_id>/saml2"
         certificate: "/etc/wazuh/azure_certificate.pem"
     ```
   - Replace placeholders:
     - `<your_wazuh_domain>`: Use your Wazuh server’s domain or IP address.
     - `<realm_name>`: Replace with your Wazuh realm’s name.
     - `<tenant_id>`: Use your Azure AD **Directory (tenant) ID** (from Step 1).

3. **Upload the SAML Certificate to Wazuh**:
   - Place the **Certificate (Base64)** file downloaded from Azure into `/etc/wazuh/` and rename it to `azure_certificate.pem`.
   - Ensure the file permissions are correctly set for security:
     ```bash
     sudo chown wazuh:wazuh /etc/wazuh/azure_certificate.pem
     sudo chmod 600 /etc/wazuh/azure_certificate.pem
     ```

4. **Restart Wazuh**:
   - Restart the Wazuh service to apply the new SAML configurations:
     ```bash
     sudo systemctl restart wazuh-manager
     ```

---

### Step 4: Assign Users in Azure AD and Test SSO

1. **Assign Users in Azure AD**:
   - Go back to **Azure Active Directory** > **Enterprise Applications** > **Wazuh SSO** (your application).
   - Under **Manage**, select **Users and groups**.
   - Click **Add user/group** and select the users or groups that should have access to Wazuh.
   - Click **Assign**.

2. **Test SSO in Wazuh**:
   - Open your Wazuh login page in a web browser.
   - Select **Sign in with SSO** or use the specific SAML login URL:
     ```
     https://<your_wazuh_domain>/auth/realms/<realm_name>/protocol/saml
     ```
     - Replace `<your_wazuh_domain>` and `<realm_name>` with your specific details.
   - You should be redirected to the Azure AD login page.
   - Log in with an Azure AD user who has been assigned access to Wazuh.
   - After successful authentication, you’ll be redirected back to Wazuh and logged in automatically.

---

### Optional Configurations

1. **Configure Timeout and Session Settings**:
   - If specific session handling is required, configure SAML session settings in Wazuh and Azure AD.

2. **Custom Roles and Permissions**:
   - For more complex role management, map Azure AD groups to Wazuh roles. In Azure AD, add group claims to your SAML application configuration. Then, configure Wazuh to interpret these claims as roles within its own authorization structure.

---

### Troubleshooting Tips

- **SSO Login Issues**: If login fails, check the Wazuh and Azure AD logs for any errors related to SAML or authentication.
- **Certificate Errors**: Ensure the SAML certificate from Azure is properly uploaded and referenced in Wazuh.
- **Session Management**: Verify session durations and timeouts in both Wazuh and Azure to avoid unexpected logouts.

This setup should provide seamless Azure AD SSO integration with Wazuh, giving users access with a single Azure AD login and maintaining secure session management.

### Conclusion
Integrating Azure AD SSO with Wazuh provides significant security and usability benefits for organizations. By centralizing authentication, users can access Wazuh seamlessly and securely using their Azure AD credentials, enhancing productivity and reducing the risk of password-related vulnerabilities. The integration also allows IT teams to leverage Azure AD’s advanced security features, such as MFA and Conditional Access, to further protect Wazuh access and improve compliance. Overall, implementing Azure AD SSO for Wazuh is a strategic step toward robust, scalable, and secure access management for the enterprise.
