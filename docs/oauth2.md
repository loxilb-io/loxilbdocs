## 1. Overview

LoxiLB REST API now supports OAuth2 token-based authentication feature to enhance security. With this update, authentication is now mandatory for API requests, improving the overall security of LoxiLB operations.

### Differences from the Previous API & Token based API

Previously, the LoxiLB API could be used with user/password based authentication or non-authentication. However, with this update, Oauth2 authentication has been introduced. The new authentication method requires users to go through a Oauth2 login process(currently google+ and github supports) to obtain a token, which must be used for API requests. This feature can be enabled using an option(`--oauth2, --oauth2provider=google,github`) and environment variables (`OAUTH2_GOOGLE_CLIENT_ID, OAUTH2_GOOGLE_CLIENT_SECRET, OAUTH2_GOOGLE_REDIRECT_URL, OAUTH2_GITHUB_CLIENT_ID, OAUTH2_GITHUB_CLIENT_SECRET, OAUTH2_GITHUB_REDIRECT_URL`) for Oauth2 authentication, and if not activated, the API can be used as before.

OAuth2 authentication-based LoxiLB does not need a separate database like the token-based approach. It seamlessly utilizes OAuth2 authentication tokens.

### Key Changes

- **New API Endpoints Added**:
    - `GET /oauth/{provider}`: OAuth2 Log In and issues a token

- **API requests requiring authentication must include the token in the `Authorization` header**
- **Authentication can be enabled/disabled through an option when running LoxiLB**

## 2. Prerequisites(e.g. google)

### **2.1 Google Cloud Account and Project Setup**

To use Google OAuth, you need a **Google Cloud Platform (GCP) account** and a **project**.

### **Creating a Google Cloud Account**

- Go to Google Cloud Console.

### **Creating a New Project**

- In **Google Cloud Console**, click **Select a project** → **New Project**.
- Enter a **Project Name**, select the **Organization** and **Location**, then click **Create**.

### **Selecting the Project**

- In the top-left **Select a project** menu, choose the newly created project.

---

### **2.2 Creating an OAuth 2.0 Client ID**

To use Google OAuth with Loxilb, you need an **OAuth 2.0 Client ID**. Below are the steps to create it in Google Cloud.

### **Setting Up the OAuth Consent Screen**

1. In **Google Cloud Console**, go to **APIs & Services** → **Credentials**.
2. Click the **OAuth consent screen** tab.
3. Choose the **Application Type**:
    - **Internal**: Only Google Workspace users can authenticate.
    - **External**: Any Google account user can authenticate.
4. Enter the **Application Name** and click **Save and Continue**.

### **Creating an OAuth 2.0 Client ID**

1. On the **Credentials** page, click **Create Credentials** → **OAuth Client ID**.
2. Select **Application Type**: **Web Application**.
3. Enter a **Client Name** (e.g., `loxilb-oauth-client`).
4. Add an **Authorized Redirect URI**:
    - Input the redirect URL
    - e.g. `http://localhost:11111/netlox/v1/oauth/google/callback`.
5. Click **Create** to generate the **Client ID** and **Client Secret**.

### **Saving the Client ID and Secret**

- Once the **Client ID** and **Client Secret** are generated, copy and save them securely.
- These credentials will be used in the **Loxilb configuration**.

If there is already a project in Google Cloud and you plan to use it, there is no issue with using the existing project and simply adding the application to it.

## 3. OAuth2 Authentication Overview

OAuth2 authentication in LoxiLB enables seamless authentication using external identity providers such as **Google** and **GitHub**. Users authenticate through OAuth2 providers, which issue a token that must be included in API requests.

### How OAuth2 Authentication Works
1. **Turn on LoxiLB using credentials**
2. **User initiates authentication** by accessing `GET /oauth/{provider}` (e.g., `GET http://localhost:11111/netlox/v1/oauth/google`).
3. **User is redirected** to the chosen provider (Google/GitHub) for login.
4. **Upon successful authentication**, the provider returns an authorization code to LoxiLB.
   If the redirect URL is set to the callback API as shown in the example above, the following process will be handled automatically, and a JSON-formatted token will be issued.
5. **LoxiLB exchanges the authorization code** for an access token.
6. **LoxiLB issues an OAuth2 token** for the user, which is used for API authentication.
7. **Users include the token in the `Authorization` header** for subsequent API requests.

### Enabling OAuth2 Authentication in LoxiLB
To enable OAuth2 authentication, use the following flags when running LoxiLB:
```
./loxilb --oauth2 --oauth2provider=google \
        --oauth2google-clientid=<CLIENT_ID> \
        --oauth2google-clientsecret=<CLIENT_SECRET> \
        --oauth2google-redirecturl=<REDIRECT_URL>
```
Alternatively, environment variables can be set:
```
export OAUTH2_GOOGLE_CLIENT_ID=<your_google_client_id>
export OAUTH2_GOOGLE_CLIENT_SECRET=<your_google_client_secret>
export OAUTH2_GOOGLE_REDIRECT_URL=<your_google_redirect_url>
export OAUTH2_GITHUB_CLIENT_ID=<your_github_client_id>
export OAUTH2_GITHUB_CLIENT_SECRET=<your_github_client_secret>
export OAUTH2_GITHUB_REDIRECT_URL=<your_github_redirect_url>
./loxilb --oauth2 --oauth2provider=google,github 
```

### Example API Usage
After obtaining the OAuth2 token, use it in API requests as follows:
```
curl -X GET "http://<loxilb-ip>:<port>/netlox/v1/config/loadbalancer/all" \
     -H "Authorization: <your-token>"
```

### Token Expiry & Renewal
- OAuth2 tokens are managed by the provider (Google/GitHub), and their expiry policies depend on the provider's settings.
- When an OAuth2 token expires, users must reauthenticate through the provider.

### Advantages of OAuth2 Authentication
- No need for LoxiLB to manage user credentials.
- Secure and widely used authentication mechanism.
- Supports federated authentication through external identity providers.
- Eliminates the need for separate user databases in LoxiLB.

