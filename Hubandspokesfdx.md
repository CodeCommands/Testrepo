# Salesforce Hub-and-Spoke: Passing AMR/ACR via External Client Apps (ECA)

When using a Salesforce Hub-and-Spoke model for SSO, custom attributes must be mapped under the **Policies** section of the External Client App (ECA) framework rather than the old Connected App developer settings.

---

## Method 1: Point-and-Click (UI Setup)
If you are passing a predictable or profile-based value to your Spoke org, you can manage this right from the Salesforce Setup interface.

1. From the Hub org **Setup**, enter `External Client Apps Manager` in the Quick Find box, then select it.
2. Find your Spoke org's External Client App from the list.
3. Click the actions dropdown arrow on the far right and select **Edit Policies**.
4. Scroll down to the **Custom Attributes** section and click the **+ (Add)** icon.
5. Click the edit icon beneath **Key** and input your identifier (e.g., `AMR` or `authnmethodsreferences`).
6. Click the edit icon beneath **Value** and insert your string or Salesforce user formula field (e.g., `"mfa"` or `$User.User_MFA_Type__c`).
7. Click **Save**.

---

## Method 2: Source-Driven Development (Metadata XML)
If your team manages deployments using source control and SFDX/Salesforce CLI, custom attributes belong in the **`extlClntAppOauthPolicies`** file.

Locate the `[Your_App_Name].extlClntAppOauthPolicies-meta.xml` file in your repository and inject the `<customAttributes>` block:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ExternalClientAppOauthPolicies xmlns="http://sforce.com">
    <customAttributes>
        <key>AMR</key>
        <value>"mfa"</value>
    </customAttributes>
    <!-- Remaining policy settings -->
</ExternalClientAppOauthPolicies>
```
*Note: String literals in metadata values must be explicitly wrapped in escaped or normal double quotes (e.g., `"mfa"`).*

---

## Method 3: Dynamic Injection via Apex (For Complex Scenarios)
If you need to dynamically check the current user session context to see if they logged in via phishing-resistant MFA before forwarding the signal, the old `ConnectedAppPlugin` class is replaced by the `Auth.ExternalClientAppOauthHandler` class.

1. Create an Apex class that extends the new handler:
```java
global class DynamicMfaSignalPlugin extends Auth.ExternalClientAppOauthHandler {
    global override Map<String, String> customAttributes(
        Id userId, 
        Id ecAppId, 
        Map<String, String> formulaDefinedAttributes, 
        Auth.InvocationContext context
    ) {
        // Check current session security level 
        Map<String, String> sessionAttributes = Auth.SessionManagement.getCurrentSession();
        String securityLevel = sessionAttributes.get('SessionSecurityLevel');
        
        // Map the existing UI formula attributes so they aren't erased
        Map<String, String> returnedAttributes = new Map<String, String>(formulaDefinedAttributes);
        
        // If the user's session is High Assurance (MFA passed), pass the signal
        if (securityLevel == 'HIGH_ASSURANCE') {
            returnedAttributes.put('AMR', 'mfa');
        } else {
            returnedAttributes.put('AMR', 'password');
        }
        
        return returnedAttributes;
    }
}
```
2. Link this Apex class to your External Client App under the **OAuth Policies** configuration.
