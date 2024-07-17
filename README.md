# Vettvangur.Audkenni API

## Overview
The `Vettvangur.Audkenni` API provides endpoints for handling authentication requests with Auðkenni's API. Below are the available endpoints and their respective details.

> [!NOTE]
> For .NET implementation we highly recommend our NuGet package [Vettvangur.AudkenniConnect](https://github.com/Vettvangur/Vettvangur.AudkenniConnect), which enables easier integration with these endpoints. The package supports:
> - .NET Framework 4.6.1 and higher
> - .NET Core 5.0 to 8.0

## Setup

#### API URL: https://audkenni.vettvangur.is/
### Authorization

To access the endpoints, developers need to include an authorization header with an API key provided by Vettvangur. Please contact some genius at Vettvangur to get an api key. The header should be set up as follows:

```http
Authorization: apiKey <actualApiKeyHere>
```

## Endpoints

### 1. `POST ~/authorize-code`

#### Used to forward an auth request to Auðkenni's API and returns a 4-digit (or 4 emojis) challenge (aka verification code).

#### Request
**Body Parameters:**
- `authType` (string, enum): Authentication type, can be "sim" or "app". Default: "sim".
- `phoneNumberOrSsn` (string): Phone number or SSN of the user. **Required**.
- `requestingApplication` (string): Alias/Name/ID of the requesting application. **Required**.
- `Challenge` (string): Optional challenge. A 4-digit (or 4 emojis) code is created if not provided.
- `useEmojiChallenge` (string): Default: `true`.
- `simChallengePrefix` (string): Default: `""`.
- `appAdditionalMessage` (string): Default: `""`.
- `lang` (string): Language code. Default: `is`.

#### Response
- `Identifier` (Guid): Important value to check the status of the auth process.
- `Challenge` (string): The 4-digit (or 4 emojis) code. Most important value for the client.
- `ChallengeHash` (string): Only used for app auth scenario.
- `AuthType` (enum): Authentication type, can be "sim" or "app".
- `RequestingApplication` (string): Alias/Name/ID of the requesting application.
- `Error` (string): Error message, if any.

**Example Response:**
```json
{
    "identifier": "D9953820-49D9-468B-833F-72B6E141D651",
    "challenge": "1234",
    "challengeHash": null,
    "authType": "sim",
    "requestingApplication": "VR",
    "error": null
}
```

### cURL examples

sim example:
```bash
curl --location 'https://<actual-api-url-here>/authorize-code' \
--header 'Content-Type: application/json' \
--header 'apiKey: ••••••' \
--data '{
    "authType": "sim",
    "phoneNumberOrSsn": "0000000",
    "requestingApplication": "VR"
}'
```

app example:
```bash
curl --location 'https://<actual-api-url-here>/authorize-code' \
--header 'Content-Type: application/json' \
--header 'apiKey: ••••••' \
--data '{
    "authType": "app",
    "phoneNumberOrSsn": "0000000000",
    "requestingApplication": "VR"
}'
```
---

### 2. `POST ~/authorize-simple`

#### Used to handle a *super-simplified* authentication process, managing every step required by Auðkenni, including polling until the user is authenticated. 
> [!WARNING] 
> This endpoint is not recommended for **app authentication** as the app will always display a 4-digit verification code, which the client application will never see. Only a message can be included for app authentication.

#### Request
**Body Parameters:**
- `authType` (string, enum): Authentication type, can be "sim" or "app". Default: "sim".
- `phoneNumberOrSsn` (string): Phone number or SSN of the user. **Required**.
- `requestingApplication` (string): Alias/Name/ID of the requesting application. **Required**.
- `ChallengeOrMessage` (string): Optional challenge or message.
- `lang` (string): Language code. Default: `is`.

#### Response
The endpoint does not return anything until one of the following occurs:
- The user enters their PIN on their device, and the API polls Auðkenni until the user is authenticated.
- The user cancels on their device.
- An error occurs.

**Example Response Model:**
```json
{
    "userInfo": {
        "name": "Jón Jónsson",
        "ssn": "0000000000"
    },
    "error": null
}
```

### cURL examples

sim example:
```bash
curl --location 'https://<actual-api-url-here>/authorize-simple' \
--header 'Content-Type: application/json' \
--header 'apiKey: ••••••' \
--data '{
    "authType": "sim",
    "phoneNumberOrSsn": "000-0000",
    "requestingApplication": "VR",
    "challengeOrMessage": "Staðfestingarkóði: 1234",
    "lang": "is"
}'
```

app example:
```bash
curl --location 'https://<actual-api-url-here>/authorize-simple' \
--header 'Content-Type: application/json' \
--header 'apiKey: ••••••' \
--data '{
    "authType": "app",
    "phoneNumberOrSsn": "000000-0000",
    "requestingApplication": "VR",
    "challengeOrMessage": "Authentication request from VR",
    "lang": "en"
}'
```
---

### 3. `POST ~/authorize-status`
#### Used to check the status of an ongoing authentication process. 
> [!NOTE] 
> No need to implement this if you want our api to handle the polling for you. Use `POST ~/authorize-status-poll` if you want automatic polling.

#### Request
**Body Parameters:**
- `identifier` (Guid): The identifier returned by the `~/authorize-code` endpoint. **Required**.
- `requestingApplication` (string): Alias/Name/ID of the requesting application. **Required**.
- `lang` (string): Language code. Default: `is`.

#### Response
- `FinishedPendingUser`: (bool): `true` if Authentication status is `Success`, otherwise `false`
- `Status`: (enum): Authentication status, can be `Pending`, `Success` or `Failed`.
- `UserInfo`: (UserInfo): 
    - `Name`: (string): Full name of the user retreived from Auðkenni
    - `Ssn`: (string): The SSN/Kennitala of the user retreived from Auðkenni
- `Error`: (string): The error message

#### Response Model:
```json
{
    "finishedPendingUser": true,
    "status": "Success",
    "userInfo": {
        "name": "Jón Jónsson",
        "ssn": "0000000000"
    },
    "error": null
}
```

### cURL example

```bash
curl --location --request GET 'https://<actual-api-url-here>/authorize-status' \
--header 'Content-Type: application/json' \
--header 'apiKey: ••••••' \
--data '{
    "identifier": "D9953820-49D9-468B-833F-72B6E141D651",
    "requestingApplication": "VR"
}'
```

---

### 4. `POST ~/authorize-status-poll`

#### This is the polling version of `~/authorize-status`. It repeatedly checks the status while it returns `Pending` until it returns a status of either `Success` (for successful authentication) or `Failed` (for an unsuccessful one due to error or user cancellation).

#### Request
**Body Parameters:**
- `identifier` (Guid): The identifier returned by the `~/authorize-code` endpoint. **Required**.
- `requestingApplication` (string): Alias/Name/ID of the requesting application. **Required**.
- `lang` (string): Language code. Default: `is`.

#### Response
- `FinishedPendingUser`: (bool): `true` if Authentication status is `Success`, otherwise `false`
- `Status`: (enum): Authentication status, can be `Pending`, `Success` or `Failed`.
- `UserInfo`: (UserInfo): 
    - `Name`: (string): Full name of the user retreived from Auðkenni
    - `Ssn`: (string): The SSN/Kennitala of the user retreived from Auðkenni
- `Error`: (string): The error message

#### Response Model:
```json
{
    "finishedPendingUser": true,
    "status": "Success",
    "userInfo": {
        "name": "Jón Jónsson",
        "ssn": "0000000000"
    },
    "error": null
}
```

### cURL example

```bash
curl --location --request GET 'https://<actual-api-url-here>/authorize-status-poll' \
--header 'Content-Type: application/json' \
--header 'apiKey: ••••••' \
--data '{
    "identifier": "D9953820-49D9-468B-833F-72B6E141D651",
    "requestingApplication": "VR"
}'
```
---
