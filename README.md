# Vettvangur.Audkenni API

## Overview
The `Vettvangur.Audkenni` API provides endpoints for handling authentication requests with Auðkenni's API. Below are the available endpoints and their respective details.

> [!NOTE]
> For .NET implementation we highly recommend our NuGet package [Vettvangur.AudkenniConnect](#vettvanguraudkenniconnect-nuget-package), which enables easier integration with these endpoints. The package supports:
> - .NET Framework 4.6.1 and higher
> - .NET Core 5.0 to 8.0

## `Setup`

### Authorization
#### API URL: https://audkenni.vettvangur.is/
To access the endpoints, developers need to include an authorization header with an API key provided by Vettvangur. Please contact some genius at Vettvangur to get an api key. The header should be set up as follows:

```http
Authorization: apiKey <actualApiKeyHere>
```

## `Endpoints`

### 1. `POST ~/authorize-code`

#### Used to forward an auth request to Auðkenni's API and returns a 4-digit (or 4 emojis) challenge (aka verification code).

#### Request
**Body Parameters:**
- `authType` (string, enum): Authentication type, can be "sim" or "app". Default: "sim".
- `phoneNumberOrSsn` (string): Phone number or SSN of the user. **Required**.
- `requestingApplication` (string): Alias/Name/ID of the requesting application. **Required**.
- `Challenge` (string): Optional challenge. A 4-digit (or 4 emojis) code is created if not provided. This only works if the authtype is "sim", app authentication always creates its own challenge code.
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

# Vettvangur.AudkenniConnect Nuget Package
Website: https://www.nuget.org/packages/Vettvangur.AudkenniConnect
## `About`
Currently, this package includes two ways for user authentication against Auðkenni's API; either via **sim** or **app**.<br/>
Both are available with or without a custom verification code (aka `challenge`).

## `Setup`
1. Install the package from NuGet: `Install-Package Vettvangur.AudkenniConnect`
1. Get your `ApplicationId` and `ApiKey` by contacting [Vettvangur](mailto:vettvangur@vettvangur.is) and add them to your app settings. <br/>
    Example for `appsettings.json`:
    ```json
    {
      "AudkenniConnect": {
        "ApplicationId": "your-provided-application-id",
        "ApiKey": "your-provided-api-key"
      }
    }
    ```
    Example for `web.config`/`app.config`:
    ```xml
    <appSettings>
        <add key="AudkenniConnect:ApplicationId" value="your-provided-application-id" />
        <add key="AudkenniConnect:ApiKey" value="your-provided-api-key" />
    </appSettings>
    ```
1. Register AudkenniConnect where you register your services:<br/>
    Example for `Startup.cs`:
    ```csharp
    var audkenniServiceProvider = new ServiceCollection().BuildAudkenniProvider(<fetch-api-key-from-appsettings-here>;
    services.AddSingleton(audkenniServiceProvider);
    services.AddScoped(sp => audkenniServiceProvider.GetService<AudkenniConnectService>());
    ```
    
1. Choose your method(s) - [see below](#choose-your-methods).

Please contact [Vettvangur](mailto:vettvangur@vettvangur.is) if you're having trouble with the setup.

## `High-level explanation of authorization steps:`
1. The client application notifies audkenni.is that a user wants to be authenticated, and gets a `challenge` code in return.
1. The client application displays the `challenge` code to the user, the user gets the request to his device and he provides his pin.
1. The client application polls the status of the authentication request until the user has finished answering the auth request on his device.

## `Choose your method(s)`
### `AuthorizeWithCodeAsync`
- Used for authentication with `challenge` code. <br/>
- Callenge code is only applicable if the authtype is "sim". For authtype "app" a challenge code will be generated for you. <br/>
- This method takes care of `step 1` from the high-level explanation above. <br/>
- Used in cohesion with `GetAuthorizationStatusAsync` or `PollAuthorizationStatusAsync`.

### `AuthorizeSimpleAsync`
- This method is used on its own for a *super-simplified* authentication process. <br/>
- This method takes care of `steps 1-3` from the high-level explanation above; except for returning the `challenge` code to the client, a custom `message` can be used instead. <br/>
- All steps of the auth-process that Auðkenni requires are handled automatically, **including polling of the auth-status until the user has finished answering the auth request on his device.**

**_NOTE:_** This method is **NOT RECOMMENDED for app authentication** as the app will always display a 4-digit `challenge`, which the client application will never see. Only a custom message can be included for app authentication.

### `GetAuthorizationStatusAsync` 
- Only use this method if you want to implement the polling yourself. <br/>
- If polling is implemented correctly then this method will take care of `step 3` from the high-level explanation above. <br/>
- Used in cohesion with `AuthorizeWithCodeAsync` to check if the user has authenticated. <br/>

### `PollAuthorizationStatusAsync`
- Use this method if you don't want to implement the polling yourself. <br/>
- This method will take care of `step 3` from the high-level explanation above. <br/>
- In simple terms, this method is just calling the same endpoint as `GetAuthorizationStatusAsync`, but multiple times with some delay between each call.

Please contact [Vettvangur](mailto:vettvangur@vettvangur.is) if you need help choosing the right methods for your use-case.
