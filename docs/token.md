## 1. Overview

LoxiLB REST API now supports OAuth2 token-based authentication feature to enhance security. With this update, authentication is now mandatory for API requests, improving the overall security of LoxiLB operations.

### Differences from the Previous API

Previously, the LoxiLB API could be used without authentication. However, with this update, authentication has been introduced. The new authentication method requires users to go through a login process to obtain a token, which must be used for API requests. This feature can be enabled using an option, and if not activated, the API can be used as before.

### Key Changes

- **New API Endpoints Added**:
    - `POST /auth/login`: Logs in and issues a token
    - `POST /auth/logout`: Logs out and revokes the token
    - `GET /auth/users`: Retrieves the list of users
    - `POST /auth/users`: Adds a new user
    - `DELETE /auth/users/{id}`: Deletes a specific user
    - `PUT /auth/users/{id}`: Modifies a specific user's information
- **API requests requiring authentication must include the token in the `Authorization` header**
- **Authentication can be enabled/disabled through an option when running LoxiLB**

## 2. Token Authentication Overview

LoxiLB REST API uses **JWT (JSON Web Token)** for token generation, while authentication is based on values stored in the database. To use authentication, users must first create an account using the `/auth/users` API. Then, they can log in by providing their user ID and password in the request body of the `POST /auth/login` API, which returns a token. The issued token is stored in both the database and cache, and after authentication, API requests are processed accordingly.

### User Role Management

Each user account has a specific **role**, with two types available: `admin` and `viewer`.

- **`admin` Role**: Has full administrative privileges and can perform all API requests.
- **`viewer` Role**: Can only perform `GET` requests and is restricted from making configuraion.

Role-Based Access Control (RBAC) is implemented to enhance security by segmenting user permissions.

### Token Issuance and Storage Method

- Upon successful login, a JWT token is issued.
- The issued token is stored in **both the database and cache**, along with its creation timestamp.
- When making API requests, the `Authorization` header must include the token, which is validated against the database or cache.

### Token Expiry and Renewal Policy

- The default validity period of an issued token is **1 hour**.
- There is **no refresh mechanism**, and once a token expires, users must **log in again** to obtain a new one.

## 3. Token Issuance Method

To obtain a token, the `--userservice` option must be enabled when running LoxiLB. When activated, the user authentication and token issuance features will function.

LoxiLB uses **MySQL** to store and manage user information. The database must be properly connected, and the following execution options can be used to configure database connection details.

### LoxiLB Execution Options and Example

| Option | Description | Example | Default |
| --- | --- | --- | --- |
| `--userservice` | Enable user authentication service | `--userservice` | `false` |
| `--databasehost` | Database host address | `--databasehost="127.0.0.1"` | `127.0.0.1` |
| `--databaseport` | Database port number | `--databaseport=3306` | `3306` |
| `--databaseuser` | Database username | `--databaseuser="root"` | `root` |
| `--databasepasswordpath` | Path to database password | `--databasepasswordpath="/etc/loxilb/mysql_password"` | `/etc/loxilb/mysql_password` |
| `--databasename` | Database name | `--databasename="loxilb_db"` | `loxilb_db` |

### Execution Example

To enable the user authentication service and configure MySQL database connection, run LoxiLB as follows:

```
./loxilb --userservice \
         --databasehost="127.0.0.1" \
         --databaseport=3306 \
         --databaseuser="root" \
         --databasepasswordpath="/etc/loxilb/mysql_password" \
         --databasename="loxilb_db"
```
Note: The databasepasswordpath option reads the plain text value from the specified file as the password. Please be cautious about file security and potential leaks. It is recommended to set the file permissions to 600 using `chmod 600 /etc/loxilb/mysql_password`. 
### Database Initialization

To use LoxiLB's authentication feature, the necessary database and tables must be set up in MySQL. Execute the SQL script provided [here](docs/db_setup.md) to initialize the database.

### Token Issuance Process

1. **Enable `-userservice` option when running LoxiLB and configure database connection**
2. **Create a user account** (`POST /auth/users` API call)

Example request:

```
curl -X POST "http://<loxilb-ip>:<port>/netlox/v1/auth/users" \
     -H "Content-Type: application/json" \
     -d '{"username": "loxilb", "password": "securepassword", "role": "admin"}'
```

Example response:

```
{
  "Result": "Success"
}
```

3. **Login Request** (`POST /auth/login` API call with user ID and password)
4. **JWT Token Issuance and Response Return**
5. **Include the token in the `Authorization` header for subsequent API requests**

Example request:

```
curl -X POST "http://<loxilb-ip>:<port>/netlox/v1/auth/login" \
     -H "Content-Type: application/json" \
     -d '{"username": "loxilb", "password": "securepassword"}'
```

Example response:

```
{
  "token": "eyJhbGciOiJIUzI1..."
}
```



## 4. Using the Token in API Calls

To make authenticated API requests, include the token in the `Authorization` header. After obtaining the token from the login response, attach it to API requests as follows:

Example request:

```
curl -X GET "http://<loxilb-ip>:<port>/netlox/v1/config/loadbalancer/all" \
     -H "Authorization: <your-token>"
```

### Request Header Explanation

| Header | Description |
| --- | --- |
| `Authorization` | The token obtained after login|

If an incorrect or expired token is used, the server will respond with an error message indicating that authentication has failed. Ensure that the token is valid and has not expired before making API requests.

## 5. Token Expiry and Renewal Policy

### Token Expiry Duration

- By default, the issued token is **valid for 1 hour**.
- After 1 hour, the token automatically expires and cannot be used for authentication.
- If an expired token is used for an API request, an error response will be returned indicating authentication failure.

### No Token Renewal Mechanism

- LoxiLB **does not provide a refresh mechanism** for tokens.
- Once a token expires, the user **must log in again** to obtain a new token.
- This design enhances security by requiring users to periodically authenticate.

### Automatic Logout Policy

- LoxiLB **does not automatically log out users** when their token expires.
- However, expired tokens will no longer be valid for API requests, effectively requiring re-authentication.

### Recommendations for Token Management

- **Monitor Token Expiry:** Ensure applications using LoxiLB APIs handle token expiration correctly and obtain a new token when necessary.
- **Handle Expired Token Errors Gracefully:** If a `401 Unauthorized` response is received, prompt the user to log in again.
- **Use Secure Storage:** Store tokens securely and avoid exposing them in logs or front-end applications.
- **Implement Periodic Re-authentication:** To minimize authentication failures due to token expiry, consider implementing automatic re-authentication processes in applications using LoxiLB APIs.

LoxiLB’s token-based authentication system is designed to improve security while maintaining simplicity in user management. Future updates may include additional security enhancements such as token refresh mechanisms or stricter session management.

## 6. Security Considerations

### Use of HTTPS

- Always use **HTTPS** for login and API requests to prevent data interception and man-in-the-middle attacks.
- Avoid using HTTP, as authentication credentials can be exposed.

### Secure Token Storage

- Tokens should be securely stored and **never hardcoded in source code**.
- Avoid storing tokens in local storage or session storage in browsers, as they can be accessed by malicious scripts.
- Prefer storing tokens in **HTTP-only, secure cookies** when applicable.

### Preventing Token Theft

- Tokens should **only be sent over secure connections (HTTPS)**.
- Implement **short-lived tokens** and refresh them periodically.
- Consider binding tokens to a user’s IP address or device fingerprint to prevent misuse.

### Role-Based Access Control (RBAC)

- LoxiLB enforces **Role-Based Access Control (RBAC)** to restrict unauthorized actions.
- The `admin` role allows all operations, while the `viewer` role only permits `GET` requests.
- Ensure users have the minimum necessary privileges.

### Protection Against Brute-Force Attacks

- Implement **rate limiting** on authentication endpoints to prevent brute-force attacks.
- Consider adding **account lockout mechanisms** after multiple failed login attempts.
- Encourage the use of **strong passwords** and multi-factor authentication (MFA).

### Token Expiration and Revocation

- Tokens expire **after 1 hour** by default to minimize security risks.
- If a token is compromised, **invalidate and regenerate** it immediately.
- Provide a logout mechanism (`POST /auth/logout`) to manually revoke a token.

## 7. Troubleshooting

### Token Not Provided in API Request

- **Error Code:** `401 Unauthorized`
- **Response Example:**

```
{
  "code": 401,
  "message": "unauthenticated for invalid credentials"
}
```

- **Cause:** The API request did not include the `Authorization` header with a valid token.
- **Solution:** Ensure that you have logged in and obtained a valid token. Include the token in the `Authorization` header of your request.

---

### Unauthorized User Trying to Access a Restricted API

- **Error Code:** `403 Forbidden`
- **Response Example:**

```
{
  "code": 403,
  "message": "Permission denied"
}
```

- **Cause:** A user with insufficient privileges (`viewer` role) attempted to perform an action requiring `admin` privileges.
- **Solution:** Verify that the logged-in user has the necessary permissions. If elevated permissions are needed, log in with an `admin` account and obtain a new token.

---

### Expired or Invalid Token Used

- **Error Code:** `500 Internal Server Error`
- **Response Example:**

```
{
  "code": 500,
  "message": "Token not found"
}
```

- **Cause:** The token used in the request is either expired or incorrect.
- **Solution:** Log in again using the `POST /auth/login` API to obtain a new token. Ensure that the correct token is included in API requests.

---

### Additional Troubleshooting Steps

- **Verify HTTP Headers:** Check whether the HTTP request header is correctly set when making an API call (`H 'Authorization: <token>'`).
- **Check Token Expiry:** If the token has expired, log in again to obtain a new token.
- **Review LoxiLB Logs:** Check the LoxiLB logs for authentication-related error messages.
- **Confirm Database Connection:** Ensure that the authentication service can connect to the database where tokens are stored.

Following these steps will help resolve most authentication-related issues in LoxiLB.
