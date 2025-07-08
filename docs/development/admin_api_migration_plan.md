# Migrating Synapse Admin to Tuwunel

This document outlines the steps required to adapt the Synapse Admin UI to work with Tuwunel, an alternative Matrix homeserver implementation.

## Table of Contents
1. [Overview](#overview)
2. [Admin UI Modifications](#admin-ui-modifications)
3. [Configuration Changes](#configuration-changes)
4. [Testing Strategy](#testing-strategy)
5. [Deployment](#deployment)
6. [Contributing](#contributing)
7. [License](#license)
8. [Appendix: Asynchronous Handling, Error Mapping, and Limitations](#appendix-asynchronous-handling-error-mapping-and-limitations)


# Overview

Synapse Admin is designed to work with Synapse's Admin API. To make it compatible with Tuwunel, we need to:
1. Ensure Tuwunel implements compatible Admin API endpoints
2. Modify the Admin UI to work with Tuwunel's API responses
3. Update authentication and configuration

# Admin Functionality Comparison

Tuwunel implements admin functionality differently from Synapse's REST API approach. Here's a comparison of how admin features are implemented in both systems. The reference for Synapse is mainly from [Synapse—The Admin API](https://element-hq.github.io/synapse/latest/usage/administration/admin_api/index.html). The Synapse documentation is also providing the sequence for the functionalities compared. Note that not the complete description is referenced, rather the ones understood as relevant for the tasks ahead.

## 1. Authenticate as a server admin

### Synapse

Many of the API calls in the admin api will require an access_token for a server admin. Note that a server admin is distinct from a room admin.

An existing user can be marked as a server admin by updating the database directly.


### Tuwunel

In Tuwunel a user can be given admin privileges for the server by using the command the admin console /which can be activated either server side 
```
admin users make-user-admin <user-ID>
```
or as commands in the `#admin` rooms for a given server.
```
!admin users make-user-admin <user-ID>
```


### Conclusion

While the basic functionality is the same, in Synapse it is not via the Admin API. Thus, there is currently no need for any activities in this respect.

## 2. Access Method

### Synapse

- Uses RESTful HTTP API endpoints under `/_synapse/admin/v1/`
- Uses HTTP request-response

### Tuwunel

- Uses Matrix room commands in the `#admins` room. 
- Uses Matrix events which are asynchronous

### Conclusion

There must be a kind of generic **API compatibility layer** allowing to access via the API. It **must**:
- **Real-time vs Request-Response**. Map REST endpoints to appropriate Tuwunel commands. **Translates REST API calls** to Tuwunel's room commands. All REST requests must issue a corresponding Matrix command with a unique correlation tag
- Handle the different response formats, including error mapping (see [Appendix](#error-mapping-table)) **Transforms responses** to match Synapse's API format. **Implements async request/response correlation** via UUIDs/tags in both commands and responses
- Manage session state for command-response correlation: **Maintains session state** with the Tuwunel admin room. 
- **Maps errors** and handles timeouts (see [Appendix](#error-mapping-table)). Implement robust error handling, including timeouts and retries.
- Provide clear user feedback for long-running or failed commands.

#### Error Mapping Table

| Tuwunel Error Text               | HTTP Status | Example Synapse Error Code | Notes                        |
|----------------------------------|-------------|----------------------------|------------------------------|
| "Permission denied"              | 403         | M_FORBIDDEN                | Not admin or not in room     |
| "User not found"                 | 404         | M_NOT_FOUND                |                              |
| "Malformed command"              | 400         | M_BAD_JSON                 |                              |
| "Timeout waiting for response"   | 504         | M_LIMIT_EXCEEDED           | Retryable error              |
| "Internal error"                 | 500         | M_UNKNOWN                  |                              |

#### Asynchronous Handling

All admin operations via Matrix are asynchronous. The compatibility layer must:

  - Attach a unique correlation tag/UUID to every command issued.
  - Track all outstanding requests, matching them to received responses via the tag.
  - Implement a configurable timeout (e.g. 10 seconds); if the timeout expires, return HTTP 504 to the client.
  - Allow for retries or cancellation if a request fails or is not acknowledged.

#### Rate Limiting
  - Synapse has built-in rate limiting
  - Tuwunel may have different limits on room commands

   **Requirement**: Implement appropriate queuing and backoff

## 3. Authentication

### Synapse

Requires explicit authentication details. It supports different authentication types (password, SSO, application service, OAuth), in the end access tokens.
Additionally, Synapse uses the `as_token` for authenticating `appservice` requests.

Note that Synapses session management, or at least the login identification and authentification are potentially different, allowing other flows. This needs to be evaluated and considered.

### Tuwunel
The assumption is that the authentication happens not at all when starting the admin console on the server directly or takes either place once, when entering the admin room (Tuwunel uses Matrix client authentication and admin room membership).

### Conclusion
- Tuwunel should maintain compatibility with this authentication method
- Consider adding additional authentication methods (JWT, etc.)
- **Handles authentication** using Tuwunel's admin room access

**Authentication flow**
- The adapter requires membership in the admin room
- Session management must consider either persistent client sessions or service-account usage.
- Support token refresh and fallback if admin room access is lost

**Session and Authentication**
- The adapter may use a persistent Matrix client session (recommended) or a service account.
- Admin room membership must be checked on startup and at regular intervals.
- If the session is lost, return HTTP 503 or prompt for re-authentication.




    
## 4. Feature Limitations

- Tuwunel may not fully support some advanced Synapse features.
- Unsupported features should be clearly flagged in the UI and API with HTTP 501 or a similar error.
- Always document known gaps and update this plan as new features are added.

# Object Details

Different classes of objects have different implementations, both in Synapse but also in Tuwunel. Here these differences will be detailed out.

## 1. User Management

| Feature             | Synapse API                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Tuwunel Equivalent                                                                           | Notes                                                                                                                                                                       |
|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **List Users**      | `GET /_synapse/admin/v2/users`<br>Parameters: <br>- `from`: Pagination start<br>- `limit`: Max users to return<br>- `guests`: Include guest users<br>- `deactivated`: Include deactivated users<br>- `name`: Search by name/localpart                                                                                                                                                                                                                           | `!admin users list [--deactivated]`                                                          | **Tuwunel Enhancement Opportunities**:<br>- Add pagination support (`--from`, `--limit`)<br>- Add guest user filtering (`--include-guests`)<br>- Add search functionality   |
| **View User**       | `GET /_synapse/admin/v2/users/{userId}`<br>Returns: <br>- User ID and localpart<br>- Display name<br>- Avatar URL<br>- Admin status<br>- Deactivated status<br>- Shadow-banned status<br>- User type (support, bot, etc.)<br>- Creation timestamp (ms)<br>- Password hash details<br>- ThreepIDs (email/phone)<br>- External IDs (from identity servers)                                                                                                        | `!admin users show @user:domain`                                                             | **Tuwunel Enhancement Opportunities**:<br>- Add detailed device information<br>- Include session information<br>- Add last seen timestamp<br>- Include IP information       |
| **Create User**     | `PUT /_synapse/admin/v2/users/{userId}`<br>Body Parameters:<br>- `password`: User's password (optional for SSO/appservice)<br>- `admin`: Whether user is admin<br>- `deactivated`: Create as deactivated<br>- `user_type`: Type of user (support, bot, etc.)<br>- `displayname`: Initial display name<br>- `threepids`: [{"medium":"email", "address":"user@example.com"}]<br>OR for appservice users:<br>- `auth`: `{ "type": "m.login.application_service" }` | `!admin users create @user:domain [password] [--admin] [--deactivated] [--displayname=NAME]` | **Authentication Types**:<br>- Password-based: Requires password field<br>- Appservice: Uses auth.type="m.login.application_service"<br>- SSO: Requires external auth setup |
| **Update User**     | `PUT /_synapse/admin/v2/users/{userId}`<br>Body: `{ "displayname": "...", "admin": bool }`                                                                                                                                                                                                                                                                                                                                                                      | `!admin users set @user:domain displayname "New Name"`                                       | Tuwunel uses separate commands for different attributes                                                                                                                     |
| **Deactivate User** | `POST /_synapse/admin/v1/deactivate/{userId}`<br>Body: `{ "erase": bool }`                                                                                                                                                                                                                                                                                                                                                                                      | `!admin users deactivate @user:domain [--erase]`                                             | Both support optional message history erasure                                                                                                                               |
| **Reset Password**  | `POST /_synapse/admin/v1/reset_password/{userId}`<br>Body: `{ "new_password": "...", "logout_devices": bool }`                                                                                                                                                                                                                                                                                                                                                  | `!admin users password @user:domain newpassword [--no-logout]`                               | Both support optional device session invalidation                                                                                                                           |
| **User Login**      | N/A (Handled by client)                                                                                                                                                                                                                                                                                                                                                                                                                                         | `!admin users login @user:domain [device_id]`                                                | Tuwunel allows admin-issued login tokens                                                                                                                                    |

### Detailed Comparison:

**User Creation**
- **Synapse**:
    - Requires explicit authentication details
    - Supports different authentication types (password, SSO, application service)
    - Can set admin status during creation
    - Returns 400 if the user already exists
    - Example:
      ```http
      PUT /_synapse/admin/v2/users/@newuser:example.com
      {
        "password": "s3cr3t",
        "admin": false,
        "displayname": "New User"
      }
      ```

- **Tuwunel**:
    - Simpler command-based interface
    - Password is optional (can be set later)
    - Admin flag can be set with `--admin`
    - Example:
      ```
      !admin users create @newuser:example.com s3cr3t --displayname "New User"
      ```

**Password Management**
- **Synapse**:
    - Separate endpoint for password reset
    - Can optionally log out all devices
    - Requires admin privileges
    - Example:
      ```http
      POST /_synapse/admin/v1/reset_password/@user:example.com
      {
        "new_password": "new_s3cr3t",
        "logout_devices": true
      }
      ```

- **Tuwunel**:
    - Simple command with an optional logout flag
    - Can be done by user (own account) or admin (any account)
    - Example:
      ```
      !admin users password @user:example.com new_s3cr3t --no-logout
      ```

## 2. Room Management

| Feature          | Synapse API                                                                                                                                                                                                                                                                                                                                                                                                                        | Tuwunel Equivalent                 | Notes                                                                                                                                                                                                      |
|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **List Rooms**   | `GET /_synapse/admin/v1/rooms`<br>Parameters:<br>- `from`: Pagination start<br>- `limit`: Max rooms to return<br>- `order_by`: `name`, `canonical_alias`, `joined_members`, `joined_local_members`, `version`, `creator`, `encryption`, `federatable`, `public`, `join_rules`, `guest_access`, `history_visibility`, `state_events`<br>- `search_term`: Filter by room ID or alias<br>- `dir`: `f` (ascending) or `b` (descending) | `!admin rooms list`                | **Tuwunel Enhancement Opportunities**:<br>- Add pagination support (`--from`, `--limit`)<br>- Add sorting options (`--order-by=joined_members --dir=desc`)<br>- Add search functionality (`--search=term`) |
| **Room Details** | `GET /_synapse/admin/v1/rooms/{roomId}`<br>Returns:<br>- Room ID and version<br>- Room state (join rules, guest access, etc.)<br>- Room metadata (creator, encryption, etc.)<br>- Room statistics (events, state events, etc.)<br>- Room state (current state events)<br>- Room members (with pagination)                                                                                                                          | `!admin rooms show !room:domain`   | **Tuwunel Enhancement Opportunities**:<br>- Add detailed member listing<br>- Include room state information<br>- Add room statistics                                                                       |
| **Delete Room**  | `DELETE /_synapse/admin/v1/rooms/{roomId}`<br>Parameters:<br>- `room_id`: The room ID to delete<br>- `new_room_user_id`: Optional user ID to create a new room and move users there<br>- `room_name`: Name for the new room (if new_room_user_id is set)<br>- `message`: Message to send to the room before deletion                                                                                                               | `!admin rooms delete !room:domain` | **Tuwunel Enhancement Opportunities**:<br>- Add option to move users to a new room<br>- Add custom message before deletion                                                                                 |
| **Purge Room**   | `POST /_synapse/admin/v1/purge_room`<br>Body:<br>- `room_id`: The room ID to purge<br>- `delete_local_events`: Whether to purge only local events                                                                                                                                                                                                                                                                                  | `!admin rooms purge !room:domain`  | **Tuwunel Enhancement Opportunities**:<br>- Add selective event purging<br>- Add option to preserve certain event types                                                                                    |

### Room Management Details

**Room List Response Example (Synapse)**:
```json
{
  "offset": 0,
  "rooms": [
    {
      "room_id": "!abc123:example.com",
      "name": "My Room",
      "canonical_alias": "#room:example.com",
      "joined_members": 42,
      "joined_local_members": 10,
      "version": "6",
      "creator": "@user:example.com",
      "encryption": "m.megolm.v1.aes-sha2",
      "federatable": true,
      "public": false,
      "join_rules": "invite",
      "guest_access": "can_join",
      "history_visibility": "shared",
      "state_events": 1234
    }
  ],
  "total_rooms": 1,
  "next_batch": 50,
  "prev_batch": 0
}
```

**Tuwunel Enhancement Plan**:
1. **Room Listing**
    - Implement pagination with `--from` and `--limit`
    - Add sorting options with `--order-by` and `--dir`
    - Add search functionality with `--search`

2. **Room Details**
    - Expand output to include all relevant room states
    - Add member listing with pagination
    - Include room statistics (messages, state events, etc.)

3. **Room Actions**
    - Add an option to move users during room deletion
    - Add custom messages for room actions
    - Implement selective event purging

Note that all async commands must be correlated and timeouts handled.

## 3. Federation

| Feature                 | Synapse API                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Tuwunel Equivalent                    | Notes                                                                                                                                                                                            |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **List Destinations**   | `GET /_synapse/admin/v1/federation/destinations`<br>Parameters:<br>- `from`: Pagination start<br>- `limit`: Max destinations to return<br>- `order_by`: `destination`, `retry_last_ts`, `retry_interval`, `failure_ts`, `last_successful_stream_ordering`<br>- `dir`: `f` (ascending) or `b` (descending)                                                                                                                                                                              | `!admin federation destinations`      | **Tuwunel Enhancement Opportunities**:<br>- Add pagination support (`--from`, `--limit`)<br>- Add sorting options (`--order-by=retry_last_ts`)<br>- Add filtering by status (`--status=failing`) |
| **Destination Details** | `GET /_synapse/admin/v1/federation/destinations/{destination}`<br>Returns:<br>- `destination`: The server name<br>- `failure_ts`: Timestamp of last failure (ms since epoch)<br>- `retry_last_ts`: When the last retry was attempted<br>- `retry_interval`: Current retry interval in ms<br>- `last_successful_stream_ordering`: Stream ordering of last successful send<br>- `pending_pdus`: Number of PDUs waiting to be sent<br>- `pending_edus`: Number of EDUs waiting to be sent | `!admin federation show example.org`  | **Tuwunel Enhancement Opportunities**:<br>- Add detailed queue information<br>- Include recent error messages<br>- Add connection statistics                                                     |
| **Reset Connection**    | `POST /_synapse/admin/v1/federation/destinations/{destination}/reset_connection`<br>Parameters:<br>- `retry_last_ts`: Optional timestamp to set for last retry attempt<br>- `retry_interval`: Optional interval to set for next retry<br>- `failure_ts`: Optional timestamp to set for last failure                                                                                                                                                                                    | `!admin federation reset example.org` | **Tuwunel Enhancement Opportunities**:<br>- Add options to set retry parameters<br>- Add force reconnect option                                                                                  |

### Federation Details

**Destination List Response Example (Synapse)**:
```json
{
  "destination": "example.org",
  "failure_ts": 1640995200000,
  "retry_last_ts": 1641081600000,
  "retry_interval": 3600000,
  "last_successful_stream_ordering": 12345678,
  "pending_pdus": 5,
  "pending_edus": 2
}
```

**Tuwunel Enhancement Plan**:
1. **Destination Listing**
    - Implement pagination with `--from` and `--limit`
    - Add sorting options (`--order-by=failure_ts --dir=desc`)
    - Add filtering by status (`--status=failing`)

2. **Connection Management**
    - Add detailed error information in `show` command
    - Include queue statistics (PDUs/EDUs waiting)
    - Show last successful sync information

3. **Troubleshooting**
    - Add command to view recent federation transactions
    - Implement federation send queue inspection
    - Add federation send retry controls

Note that all async commands must be correlated and timeouts handled.

## 4. Appservices

| Feature                 | Synapse API                                                                                                                                    | Tuwunel Equivalent                               | Notes                                                                                                                                                                    |
|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Register Appservice** | `POST /_synapse/admin/v1/appservices`<br>Body: Full appservice registration YAML                                                               | `!admin appservices register [yaml]`             | **Tuwunel Enhancement Opportunities**:<br>- Support YAML input from file or stdin<br>- Add validation of registration data<br>- Generate appservice tokens automatically |
| **List Appservices**    | `GET /_synapse/admin/v1/appservices`<br>Returns:<br>- `appservices`: Array of registered appservices<br>- `total`: Total number of appservices | `!admin appservices list`                        | **Tuwunel Enhancement Opportunities**:<br>- Add detailed output format<br>- Include status information (online/offline)<br>- Show last activity timestamp                |
| **Get Appservice**      | `GET /_synapse/admin/v1/appservices/{appserviceId}`<br>Returns full appservice configuration                                                   | `!admin appservices show <name>`                 | **Tuwunel Enhancement Opportunities**:<br>- Add detailed status information<br>- Include connection statistics<br>- Show recent errors                                   |
| **Update Appservice**   | `PUT /_synapse/admin/v1/appservices/{appserviceId}`<br>Body: Updated appservice configuration                                                  | `!admin appservices update <name> [yaml]`        | **Tuwunel Enhancement Opportunities**:<br>- Support partial updates<br>- Add dry-run option<br>- Validate configuration before applying                                  |
| **Delete Appservice**   | `DELETE /_synapse/admin/v1/appservices/{appserviceId}`                                                                                         | `!admin appservices unregister <name> [--purge]` | **Tuwunel Enhancement Opportunities**:<br>- Add confirmation prompt<br>- Add --force flag to skip confirmation<br>- Add --purge option to clean up data                  |

### Appservices Details

**Appservice Registration Example (Synapse)**:
```yaml
id: "my_appservice"
url: "http://my-appservice:8000"
as_token: "your_as_token"
hs_token: "your_hs_token"
sender_localpart: "_appservice"
namespaces:
  users:
    - exclusive: true
      regex: "@_appservice\\.(.*)"
  aliases: []
  rooms: []
```

**Tuwunel Enhancement Plan**:
1. **Registration & Management**
    - Support YAML input from files or direct input
    - Add validation for required fields
    - Generate secure tokens if not provided

2. **Status Monitoring**
    - Track appservice connection status
    - Monitor message queue sizes
    - Log and display recent errors

3. **Advanced Features**
    - Rate limiting configuration
    - Connection timeouts and retries
    - Webhook URL management

### Implementation Notes

1. **Rate Limiting**
    - Synapse has built-in rate limiting for appservices
    - Tuwunel should implement similar controls
    - Consider per-appservice rate limiting

2. **Error Handling**
    - Detailed error responses for configuration issues
    - Logging for failed requests
    - Retry mechanisms for transient failures

Note that all async commands must be correlated and timeouts handled.

## Admin UI Modifications

Unchanged, but recommends surfacing pending and failed async operations clearly to the user.

## Configuration Changes

### Environment Variables

(Recommend adding settings for timeouts, retry, and feature flags for unsupported functionality.)

### Feature Flags

Unchanged but includes flags for features currently unsupported by Tuwunel.

## Testing Strategy

**Enhanced:**
- Develop a mock Matrix client to simulate Tuwunel admin room events, including both success and error scenarios.
- Add fixtures for all API request/response pairs and edge cases.
- Unit and integration tests must cover correlation, timeouts, and error mapping.

## Deployment

(Unchanged)

## Troubleshooting

**Expanded:**
- If an API response times out, check Matrix connectivity and admin room status.
- If commands are not correlated, ensure correlation tags are unique and consistent in both requests and events.

## Contributing

When contributing to the Tuwunel compatibility layer:
1. Follow the existing code style
2. Add tests for new functionality, including async and error cases
3. Document any Tuwunel-specific behaviour or limitations
4. Update this migration guide as needed

## License

[Apache License like Tuwunel](/./LICENSE)

## Appendix: Asynchronous Handling, Error Mapping, and Limitations



### Test Infrastructure

- Mock-driven tests should cover all request/response flows.
- Simulate both normal and error scenarios, including timeouts and malformed events.

## Appendix: Todos (CR)

This chapter gives an overview about the tasks & todos to be done and how these are split into local Cr and git commits

- [x] 01 Document initial approach for including Synapse Admin API in Tuwunel
- [X] 02 Adopt configuration file for Admin API settings
- [ ] 03 Describe Synapse Admin features in details, to have a base for starting
- [ ] 04
- [ ] 05
- [ ] 06
- [ ] 07
- [ ] 08

## Appendix: Footnotes and References

[¹] Synapse Admin documentation https://github.com/matrix-org/synapse/tree/develop/docs/usage/administration/admin_api

---
*Last updated: July 2025*
