1. Prompt tối ưu
   Act as a Senior System Analyst expert in Web Security and Enterprise Software Specification.

   Write a comprehensive and technical Functional Requirement (FR) specification document for the Login module of the "Shop AI" project.

   The system architecture implements JSON Web Token (JWT) for authentication and Role-Based Access Control (RBAC) for authorization. The login process must be secure, resilient, and handle edge cases gracefully to prevent unauthorized access and maintain a smooth user experience.

   You must strictly specify the system behavior, data flow, and error messages for the following critical scenarios:
   1. Brute-Force Protection: The system must track failed attempts. If a user enters an incorrect password more than 5 consecutive times, specify the lockout duration, account status update, and notification mechanism.
   2. Token Expiration Handling: Specify how the system handles an expired JWT token during an active session. Detail the silent refresh mechanism (using Refresh Tokens) or the graceful redirection to the login page without losing critical user state.
   3. Inactive Account Access: Define the precise system validation workflow and error response when a user attempts to log in with an account whose status is explicitly set to 'Inactive' or 'Suspended'.

   Generate the response in a structured Markdown format with the following sections:
   - 1. Functional Overview
   - 2. Technical Prerequisites (JWT & RBAC Strategy)
   - 3. Detailed Workflow (Main Flow)
   - 4. Exception Handling & Edge Cases (Structured in a comparison table for the 3 cases above, containing: Trigger, System Validation Action, and API Response/UI Behavior).

2. Functional Requirements do AI sinh ra

   # FUNCTIONAL REQUIREMENT SPECIFICATION: AUTHENTICATION MODULE (SHOP AI)

   ## 1. Functional Overview

   The Authentication module provides secure entry to the Shop AI platform. It verifies user credentials, generates secure session tokens, and evaluates user roles to enforce Role-Based Access Control (RBAC) across the system ecosystem.

   ## 2. Technical Prerequisites (JWT & RBAC Strategy)
   - **Authentication Protocol:** Stateless authentication utilizing JSON Web Tokens (JWT). Upon successful login, the system issues an `access_token` (short-lived: 15 minutes, stored in memory/secure cookie) and a `refresh_token` (long-lived: 7 days, stored in an HttpOnly, SameSite=Strict cookie).
   - **Authorization Model:** RBAC architecture. The JWT payload must securely include the user's unique identifier and an array of assigned roles (e.g., `["ROLE_CUSTOMER", "ROLE_ADMIN"]`) to allow instantaneous front-end and back-end route guarding.

   ## 3. Detailed Workflow (Main Flow)
   1. **Request Submission:** The user submits their identifier (email/username) and password via the HTTPS POST `/api/v1/auth/login` endpoint.
   2. **Credential Verification:** The system fetches the user record, hashes the incoming password using bcrypt, and compares it against the stored hash.
   3. **Status Check:** The system verifies that the account status is active.
   4. **Token Generation:** The system generates the JWT Access Token and Refresh Token, embedding user identity and RBAC roles.
   5. **Success Response:** The system returns a `200 OK` status code, delivers the access token in the response body, sets the refresh token cookie, and updates the `last_login_at` timestamp in the database.

   ## 4. Exception Handling & Edge Cases

   The following matrix specifies how the system handles complex edge cases and security exceptions:

   | Exception Case                    | Trigger Condition                                                                             | System Validation & Business Logic Action                                            | API Response & UI Behavior |
   | --------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------- |
   | **EX-01: Brute-Force Protection** | User enters an incorrect password greater than 5 consecutive times within a 15-minute window. | - Increment the `failed_login_attempts` counter in the database on each failure.<br> |

   <br>- Upon the 6th failed attempt, change `account_status` to `LOCKED`. <br>

   <br>- Set a `lockout_until` timestamp to Current Time + 30 minutes.<br>

   <br>- Log a security alert in the system logs. | - **HTTP Status:** `423 Locked`<br>

   <br>- **JSON Payload:** `{"error": "Account locked", "message": "Too many failed attempts. Try again after 30 minutes."}`<br>

   <br>- **UI Action:** Disable the login button, display a countdown timer, and trigger an automated security alert email to the user. |
   | **EX-02: Token Expiration during Active Session** | The `access_token` expires while the user is actively interacting with the Shop AI web interface. | - The client-side Interceptor intercepts a `401 Unauthorized` response from any business API.<br>

   <br>- The client automatically makes a silent POST request to `/api/v1/auth/refresh` sending the HttpOnly `refresh_token`.<br>

   <br>- Back-end validates the refresh token against the rotation whitelist in Redis.<br>

   <br>- If valid, a new `access_token` is issued. If expired/invalid, clear session. | - **Successful Refresh:** New token issued silently; the original business API request is retried seamlessly with zero user interruption.<br>

   <br>- **Failed Refresh (HTTP 401):** Clear all local session storage and redirect the user to `/login?session=expired` with a toast message: _"Your session has expired. Please log in again."_ |
   | **EX-03: Inactive Account Login Attempt** | A user with an `account_status` set to `INACTIVE` or `SUSPENDED` enters correct credentials. | - The system successfully matches the password hash.<br>

   <br>- The system evaluates the user status flag before generating tokens.<br>

   <br>- Token generation is immediately aborted to prevent unauthorized access hooks.<br>

   <br>- Increment an audit log tracking unauthorized access attempts on suspended records. | - **HTTP Status:** `403 Forbidden`<br>

   <br>- **JSON Payload:** `{"error": "Account Inactive", "message": "Your account has been deactivated. Please contact support."}`<br>

   <br>- **UI Action:** Clear the password field, keep the user on the login screen, and display a prominent red error banner with a hyperlink to the customer support portal. |
