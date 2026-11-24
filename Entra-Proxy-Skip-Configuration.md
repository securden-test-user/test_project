# Configure Proxy Support for Mobile Applications

Securden PAM mobile applications can reach an on-premises PAM server through Microsoft Entra Application Proxy. After this is configured, the mobile application uses a token from PAM and does not ask the user to sign in to the proxy on every connection.

Two Microsoft Entra applications are required. They are not the same application, and the two client IDs must not be swapped.

| Create this in Microsoft Entra | Enter it in Securden PAM |
| --- | --- |
| App registration that PAM uses to request a token | **Admin >> Cloud Services**: Tenant ID, Client ID, Client Secret |
| Enterprise application published with Application Proxy | **Admin >> Mobile Applications >> Proxy Support**: Application ID and External URI |

PAM requests a token for `api://<Proxy Application ID>/.default` with the Cloud Services client ID and secret. It then calls the External URI with that token. **Validate and Save** succeeds only when that call returns HTTP 200.

## Before you start

- Use a Microsoft Entra account that can create app registrations, grant admin consent, and publish Application Proxy applications.
- Install the **Application Proxy connector** on a Windows server that can open the internal PAM address. In the Entra admin center, go to **Enterprise applications >> Application proxy >> Download connector service**. The connector must show as **Active** before you publish the application.
- Decide the internal PAM URL that the connector can reach, for example `https://pam.corp.contoso.com/`.
- On the PAM server, you need permission to edit `conf\builtin.properties` and restart the Securden PAM service.

## 1. Create the Cloud Services app registration

This is the application PAM uses to sign in and request the proxy token.

1. Open [https://entra.microsoft.com](https://entra.microsoft.com).
2. Go to **Identity >> Applications >> App registrations >> New registration**.
3. Name it `Securden PAM Cloud Service`.
4. Supported account types: **Accounts in this organizational directory only**.
5. Click **Register**.
6. Go to **Authentication >> Add a platform >> Web**.
7. Set the Redirect URI to the PAM address that is open in the browser, plus `/onedrivedirauthdone`.
   Example: `https://pam.corp.contoso.com/onedrivedirauthdone`
   If you will open PAM through the Application Proxy address while saving Cloud Services, use that external address instead.
8. Go to **API permissions >> Add a permission >> Microsoft Graph >> Delegated permissions**.
9. Select **User.Read**, click **Add permissions**, then **Grant admin consent**.
10. On **Overview**, copy:
    - **Application (client) ID** — Cloud Services **Client ID**
    - **Directory (tenant) ID** — Cloud Services **Tenant ID**
11. Go to **Certificates & secrets >> Client secrets >> New client secret**.
12. Add a description such as `Securden PAM`, choose an expiry, and click **Add**.
13. Copy the **Value** immediately. This is the Cloud Services **Client Secret**. The Secret ID is not the secret.

The redirect URI and **User.Read** are used when PAM signs in to Microsoft while saving the cloud service. Proxy skip itself uses the application permission created in the next section.

## 2. Create the Application Proxy enterprise application

This is the application Entra publishes outside the network. Its client ID and external URL are what the Proxy Support page asks for.

1. Go to **Identity >> Applications >> Enterprise applications >> New application**.
2. Select **Add an on-premises application** (Application Proxy).
3. Name it `Securden PAM Application Proxy`.
4. **Internal URL**: the PAM address the connector can reach, for example `https://pam.corp.contoso.com/`.
5. **External URL**: keep the address Entra generates, or set a free `*.msappproxy.net` host. Copy this full URL. This is the **External URI** in PAM.
6. **Pre Authentication**: Microsoft Entra ID.
7. **Connector Group**: the group that contains the active connector.
8. Turn on **Translate URLs in headers**.
9. Click **Create**. Wait until the application shows as published.

## 3. Allow the Cloud Service app to call the proxy app

PAM does not use a secret from the proxy application. It asks the Cloud Service app for a token whose scope is `api://<proxy client ID>/.default`. That scope exists only after these steps.

1. Open **Enterprise applications >> Securden PAM Application Proxy >> Properties** and copy the **Application ID**. This is the Proxy Support **Application ID**.
2. Go to **App registrations** and open the application with that same Application ID. Entra created it when the on-premises application was published.
3. Go to **Expose an API**.
4. Next to **Application ID URI**, click **Set**. Accept `api://<this Application ID>` and click **Save**. The URI must be `api://` plus the proxy Application ID, with nothing after it.
5. Click **App roles >> Create app role**.
   - Display name: `Proxy access`
   - Allowed member types: **Applications**
   - Value: `Proxy.Access`
   - Description: `Lets Securden PAM request an application token for this proxy.`
   - Enable the role and click **Apply**.
6. Go to **App registrations >> Securden PAM Cloud Service >> API permissions >> Add a permission**.
7. Choose **APIs my organization uses**, search for `Securden PAM Application Proxy`, and select it.
8. Choose **Application permissions**, select **Proxy access**, and click **Add permissions**.
9. Click **Grant admin consent** and confirm.

If the enterprise application requires assignment, open **Enterprise applications >> Securden PAM Application Proxy >> Users and groups** and assign the users who will use the mobile application.

Keep these five values. The client secret is shown only once.

```text
Tenant ID                         : ........
Cloud Services Client ID          : ........
Cloud Services Client Secret      : ........
Proxy Support Application ID      : ........
Proxy Support External URI        : https://....msappproxy.net/
```

## 4. Trust the external address on the PAM server

PAM accepts logins only from hosts it trusts. Sign-in through the Application Proxy address fails with **CSRF verification failed** until the external origin is added.

1. On the Securden PAM server, open `conf\builtin.properties` in the PAM installation directory.
2. Add the external origin, without a trailing path:

```properties
CSRF_TRUSTED_ORIGINS = ["https://securden-pam-contoso.msappproxy.net"]
```

If the property already exists, add this origin inside the existing list. Do not remove the origins that are already there.

3. Save the file.
4. Restart the Securden PAM service.
5. Open a new browser window and confirm that `https://securden-pam-contoso.msappproxy.net/` reaches the PAM login page.

Use the host from the External URI you copied. Do not include `/login` or a trailing slash inside the origin value.

## 5. Add the Cloud Service in PAM

1. Sign in to PAM as an administrator.
2. Open **Admin >> Cloud Services >> Add**.
3. **Cloud Service Identifier**: a name you will recognize, such as `Contoso Entra`.
4. **Cloud Service Provider**: **Microsoft**.
5. Under **Choose the required scope**, turn on **Proxy Support for Mobile Applications**.
6. **Tenant ID**: Directory (tenant) ID from the Cloud Service app.
7. Select **Enter Client ID and Client Secret**.
8. **Client ID**: Application (client) ID of **Securden PAM Cloud Service**.
9. **Client secret**: the secret **Value** from that same app registration.
10. Save the cloud service and complete the Microsoft sign-in if PAM opens it.

The profile is not ready for Proxy Support until this save succeeds.

## 6. Add Proxy Support in PAM

1. Open **Admin >> Mobile Applications >> Proxy Support >> Add**.
2. **Configured Proxy Support Profile**: the Cloud Service identifier saved above.
3. **Application ID**: Application (client) ID of **Securden PAM Application Proxy**. Do not paste the Cloud Services client ID here.
4. **External URI**: the external URL from Application Proxy, for example `https://securden-pam-contoso.msappproxy.net/`.
5. Click **Validate and Save**.

Validation requests the token, then opens the External URI with it. The profile is saved only when that response is HTTP 200. A successful save shows **Proxy Support for Mobile Application configured**.

## 7. Check the mobile application

1. Install the Securden Android or iOS application.
2. When it asks for the server URL, enter the PAM server URL.
3. Sign in. The configured profile supplies the proxy token, so the mobile application can pass the Entra proxy without a separate proxy sign-in on each connection.

## If Validate and Save fails

| What you see | What to check |
| --- | --- |
| Unable to configure Proxy Support for Mobile Application | The token request failed, or the External URI did not return HTTP 200. |
| Client ID and secret were swapped | Cloud Services must use the **Cloud Service** app. Proxy Support **Application ID** must be the **Application Proxy** app. |
| Application ID URI is different from `api://<Application ID>` | PAM always requests `api://<the Application ID you typed>/.default`. |
| Admin consent was not granted | **Securden PAM Cloud Service >> API permissions** must show **Proxy access** as granted. |
| No connector, or the connector is inactive | **Enterprise applications >> Application proxy**. The connector server must be able to open the internal PAM URL. |
| External URI returns 302 or 401 | PAM accepts only HTTP 200. Publish an internal address that returns 200 to the connector, and confirm the external URL belongs to this application. |
| Login through the external URL shows **CSRF verification failed** on `/loginSubmit` | Add the external origin to `conf\builtin.properties` as in section 4, restart the PAM service, and sign in again in a new browser window. **Translate URLs in headers** must be on. |
| Secret was copied from Secret ID | Use the secret **Value** shown when the secret was created. Create a new secret if that value was not saved. |
| Cloud Services save opens `.../onedrivedirauthdone` and shows **Access Denied** | On the Cloud Service app registration, add a Web redirect URI of `https://<the PAM address in the browser>/onedrivedirauthdone`, add delegated **User.Read**, and grant admin consent. Save the cloud service again and complete the Microsoft sign-in. |
