# Salesforce Hub and Spoke SSO Setup Instructions

This document provides step-by-step instructions for configuring the Hub and Spoke SSO architecture, where a central UAT Org acts as the Hub (Identity Provider) and lower-level sandboxes act as Spokes (Service Providers).

---

## 1. UAT / Dedicated Hub Org Setup

These steps configure the UAT org to act as the Identity Provider (IdP) for your lower-level sandboxes, while acting as a Service Provider to your external OneID system.

### A. Configure External SSO (OneID)
1. Navigate to **Setup > Single Sign-On Settings**.
2. Check **SAML Enabled**.
3. Click **New** (or New from Metadata File) and configure the connection to OneID:
   * **Name:** `OneID PIV`
   * **Issuer / Entity ID:** (Provided by OneID metadata)
   * **Identity Provider Login URL:** (Provided by OneID metadata)
   * **Identity Provider Certificate:** Upload the certificate provided by the OneID team.
   * **SAML Identity Type:** Select `Assertion contains the Federation ID from the User object`.
   * **SAML Identity Location:** `Identity is in the NameIdentifier element of the Subject statement` (or per OneID specifications).
4. Save the configuration.
5. Navigate to **Setup > My Domain** -> **Authentication Configuration**, and check the box for `OneID PIV` to enable it on the Hub's login page.

### B. Enable Identity Provider
1. Navigate to **Setup > Identity Provider**.
2. Click **Enable Identity Provider**.
3. Select an existing Self-Signed Certificate or create a new one (e.g., `UAT_Hub_IdP_Cert`). *Note: You will need to download this certificate later to upload to your Spoke sandboxes.*

### C. Create Connected Apps for Spokes
For **each** Spoke sandbox, you must create a Connected App in the UAT Hub.
1. Navigate to **Setup > App Manager**.
2. Click **New Connected App**.
3. **Basic Information:**
   * **Name:** e.g., `SSO to Dev1 Sandbox`
   * **Contact Email:** Enter a valid admin email.
4. **Web App Settings:**
   * Check **Enable SAML**.
   * **Entity ID:** The Spoke sandbox's My Domain URL (e.g., `https://mycompany--dev1.sandbox.my.salesforce.com`).
   * **ACS URL:** The Spoke sandbox's SAML consumption URL. *Note: This usually looks like `https://mycompany--dev1.sandbox.my.salesforce.com?so=[15-Digit-Org-ID]`*.
   * **Subject Type:** Select `Federation ID`.
   * **Name ID Format:** Select `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`.
   * **Issuer:** The UAT Org's My Domain URL.
   * **Verify Request Signatures:** Check this box.
   * **IdP Certificate:** Upload the "Shared Spoke Request" certificate (the cert all sandboxes will use to sign their requests to UAT).
5. Save the Connected App.
6. **Assign Access:** Navigate to the Connected App's **Manage** page -> **Manage Profiles** (or Permission Sets) and assign the necessary developer/admin profiles so they are authorized to use this app.

---

## 2. New Sandbox (Spoke) Setup

These steps configure a newly spun-up Spoke sandbox to delegate authentication to the UAT Hub.

### A. Certificate Management
*Best Practice:* Generate your "Shared Spoke Request" Self-Signed Certificate in Production and leave it there (do not use it for Prod SSO). This ensures it automatically copies down to every newly created or refreshed sandbox.
1. If the certificate is not present, navigate to **Setup > Certificate and Key Management** and import the shared keystore file for your "Shared Spoke Request" cert.

### B. Configure Single Sign-On Settings
1. Navigate to **Setup > Single Sign-On Settings**.
2. Check **SAML Enabled**.
3. Click **New** to configure the connection to UAT:
   * **Name:** `UAT Hub SSO`
   * **Issuer:** The UAT Org's My Domain URL (must match exactly what is in UAT).
   * **Entity ID:** The current Spoke's My Domain URL (e.g., `https://mycompany--dev1.sandbox.my.salesforce.com`).
   * **Identity Provider Certificate:** Upload the `UAT_Hub_IdP_Cert` you downloaded from the UAT Identity Provider setup.
   * **Request Signing Certificate:** Select the "Shared Spoke Request" certificate from the dropdown.
   * **Identity Provider Login URL:** The IdP Endpoint URL from the UAT Org (found in UAT at Setup > Identity Provider).
   * **SAML Identity Type:** Select `Assertion contains the Federation ID from the User object`.
4. Save the configuration.

### C. Enable SSO on the Login Page
1. Navigate to **Setup > My Domain**.
2. Scroll to **Authentication Configuration** and click **Edit**.
3. Under **Authentication Service**, check the box next to `UAT Hub SSO`.
4. Click **Save**.

---

## 3. Existing Sandbox: Post-Refresh Steps

When a sandbox is refreshed, it receives a **new Org ID**, which breaks the ACS URL in the Hub. Additionally, developer data (like Federation IDs) may be missing if they are not in Production.

### A. UAT Hub Updates (Critical Step)
1. Log into the **UAT Org (Hub)**.
2. Navigate to **Setup > App Manager** and find the Connected App for the refreshed Spoke.
3. Edit the Web App Settings.
4. **Update the ACS URL:** Replace the old 15-digit Org ID in the ACS URL with the newly refreshed sandbox's Org ID (e.g., `https://mycompany--dev1.sandbox.my.salesforce.com?so=[NEW-15-DIGIT-ORG-ID]`).
5. Save the Connected App.

### B. Spoke Sandbox Updates
1. Log into the newly refreshed **Spoke Sandbox** (using standard username/password initially).
2. **Verify Shared Certificate:** Go to **Setup > Certificate and Key Management**. Ensure the "Shared Spoke Request" certificate copied over from Production successfully. 
3. **Re-establish SSO settings:** Navigate to **Setup > Single Sign-On Settings**. Because the sandbox was refreshed from Prod, the Spoke-to-UAT SSO setting will likely be missing (since Prod doesn't connect to UAT).
   * Execute your automated post-refresh script (Apex/CLI) to recreate the `UAT Hub SSO` setting, OR
   * Manually recreate it following the steps in **Section 2.B**.
4. **Enable Login Button:** Go to **Setup > My Domain > Authentication Configuration** and re-check the box for `UAT Hub SSO`.
5. **Update Developer Federation IDs:** * Sandbox refreshes typically only bring over active Production users. If developers use different accounts in sandboxes, or if their Prod accounts don't have Federation IDs, execute a data load or an Apex post-copy script to populate the `FederationIdentifier` field on their User records so they can successfully authenticate.