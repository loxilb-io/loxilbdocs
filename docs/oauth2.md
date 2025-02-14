## 1. Overview

Due to continuous requests for Oauth2 authentication in the LoxiLB REST API, a Oauth2 token-based authentication feature has been added to enhance security. With this update, authentication is now mandatory for API requests, improving the overall security of LoxiLB operations.

### Differences from the Previous API & Token based API

Previously, the LoxiLB API could be used with user/password based authentication or non-authentication. However, with this update, Oauth2 authentication has been introduced. The new authentication method requires users to go through a Oauth2 login process(currently google+ and github supports) to obtain a token, which must be used for API requests. This feature can be enabled using an option(`--oauth2, --oauth2provider=google,github`) and environment variables (`OAUTH2_GOOGLE_CLIENT_ID, OAUTH2_GOOGLE_CLIENT_SECRET, OAUTH2_GOOGLE_REDIRECT_URL, OAUTH2_GITHUB_CLIENT_ID, OAUTH2_GITHUB_CLIENT_SECRET, OAUTH2_GITHUB_REDIRECT_URL`) for Oauth2 authentication, and if not activated, the API can be used as before.

OAuth2 authentication-based LoxiLB does not need a separate database like the token-based approach. It seamlessly utilizes OAuth2 authentication tokens.

### Key Changes

- **New API Endpoints Added**:
    - `GET /oauth/{provider}`: OAuth2 Log In and issues a token

- **API requests requiring authentication must include the token in the `Authorization` header**
- **Authentication can be enabled/disabled through an option when running LoxiLB**

## 2. OAuth2 Authentication Overview

OAuth2 authentication in LoxiLB enables seamless authentication using external identity providers such as **Google** and **GitHub**. Users authenticate through OAuth2 providers, which issue a token that must be included in API requests.

### How OAuth2 Authentication Works
1. **User initiates authentication** by accessing `GET /oauth/{provider}` (e.g., `GET /oauth/google`).
2. **User is redirected** to the chosen provider (Google/GitHub) for login.
3. **Upon successful authentication**, the provider returns an authorization code to LoxiLB.
4. **LoxiLB exchanges the authorization code** for an access token.
5. **LoxiLB issues an OAuth2 token** for the user, which is used for API authentication.
6. **Users include the token in the `Authorization` header** for subsequent API requests.

### Enabling OAuth2 Authentication in LoxiLB
To enable OAuth2 authentication, use the following flags when running LoxiLB:
```
./loxilb --oauth2 --oauth2provider=google \
        --oauth2clientid=<CLIENT_ID> \
        --oauth2clientsecret=<CLIENT_SECRET> \
        --oauth2redirecturl=<REDIRECT_URL>
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
     -H "Authorization: Bearer <your-token>"
```

### Token Expiry & Renewal
- OAuth2 tokens are managed by the provider (Google/GitHub), and their expiry policies depend on the provider's settings.
- When an OAuth2 token expires, users must reauthenticate through the provider.

### Advantages of OAuth2 Authentication
- No need for LoxiLB to manage user credentials.
- Secure and widely used authentication mechanism.
- Supports federated authentication through external identity providers.
- Eliminates the need for separate user databases in LoxiLB.

