# K 40 RF API

The K 40 RF is an internet gateway for controlling and monitoring your heating system or ventilation unit remotely.

You can use the K 40 RF Local API to read data from your heating system over your local network.
This guide takes you from gateway discovery to your first authenticated request.

**Important:** For now, the API provides read-only access and the resources available depend on your connected appliance and system configuration.

## Supported products

This documentation covers the following brand-specific variations:

| Brand                   | Product Name        | Technical documentation                                                          |
| ----------------------- | ------------------- | -------------------------------------------------------------------------------- |
| Bosch                   | Connect-Key K 40 RF | [Installation Manual](https://www.docs.bosch-thermotechnology.com/mc/7738114013) |
| Buderus                 | MX400               | [Installation Manual](https://www.docs.bosch-thermotechnology.com/mc/7738113982) |
| IVT, Vulcano, Worcester | K 40 RF             | [Installation Manual](https://www.docs.bosch-thermotechnology.com/mc/7738113983) |

**Important:** throughout this documentation we use _gateway_ to refer to all variations.

## Before you start

To follow the next steps, you need:

- A [supported gateway](#supported-products) paired with your heating appliance.
- Gateway firmware version `15.00.01` or newer.
- A computer on the same local network as your gateway.
- Access to a `bash` or `zsh` shell with `curl` installed.
- The `Login` and `Pass` values printed on the gateway label.
- Physical access to the gateway.

## Make your first request

### 1. Discover your gateway

#### Option 1: Use the mDNS hostname

Discover your gateway through the mDNS service `_hvac-open-api._tcp.local.`.
The discovery procedure depends on your operating system.

Set `K40_HOST` to the returned hostname, without any trailing dot or port:

```bash
export K40_HOST="replace-with-host.local"
```

#### Option 2: Use the gateway's IP address

Find your gateway's IP address in your router's device list.
Set `K40_HOST` to that IP address:

```bash
export K40_HOST="replace-with-ip-address"
```

**Important:** The gateway uses two HTTPS ports:

| Port   | Purpose                                 |
| ------ | --------------------------------------- |
| `9442` | Create, list, and revoke access tokens. |
| `9443` | Read API resources.                     |

### 2. Use HTTPS with certificate verification disabled

The examples use `--insecure` because of a [known certificate limitation](#known-limitation-https-certificate-verification).
The connection remains encrypted, but `curl` does not verify the gateway's identity.

Use these requests only on a trusted local network.
Include `--insecure` in requests to both ports, as shown in the examples.

### 3. Prepare your credentials

Replace the example values with the `Login` and `Pass` values from your gateway label.
Choose a client name that identifies your application:

**Important:** Remove all hyphens from the `Pass` value.

```bash
export K40_LOGIN="replace-with-your-login"
export K40_PASSWORD="replace-with-your-pass-without-hyphens"
export K40_CLIENT_NAME="replace-with-your-client-name"
```

### 4. Create your access token

1. Press the WLAN and Wireless buttons simultaneously for 1 second.
2. Make sure that the LEDs briefly blink blue.
3. Within 5 minutes, run the request that follows.

The request sends form-encoded credentials to port `9442`.

```bash
curl \
  --silent --show-error --fail-with-body \
  --insecure \
  --request POST "https://${K40_HOST}:9442/auth/token" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "username=${K40_LOGIN}" \
  --data-urlencode "password=${K40_PASSWORD}" \
  --data-urlencode "client_name=${K40_CLIENT_NAME}"
```

A successful response contains your access token:

```json
{
  "access_token": "your-access-token",
  "token_type": "Bearer",
  "scope": "open_api.read"
}
```

Copy the `access_token` value from your response into an environment variable:

```bash
export K40_ACCESS_TOKEN="your-access-token"
```

In case of `412 physical_proximity_unproven`, repeat the button sequence before you retry the request.

The access token does not expire. Store it securely and treat it like a password.

### 5. Make your first authenticated request

Request `/gateway/brand`, which is available on every supported gateway.
Use your access token in the `Authorization` header on port `9443`:

```bash
curl \
  --silent --show-error --fail-with-body \
  --insecure \
  --header "Authorization: Bearer ${K40_ACCESS_TOKEN}" \
  "https://${K40_HOST}:9443/gateway/brand"
```

A successful request returns HTTP `200` and a JSON resource like this:

```json
{
  "id": "/gateway/brand",
  "type": "stringValue",
  "writeable": 0,
  "value": "Buderus"
}
```

`value` contains your gateway's brand, which can differ from this example.
`id` identifies the resource.
`type` describes its value type, and `writeable: 0` identifies a read-only resource.

You now have an authenticated connection to the Local API.
Use the [OpenAPI specification](../openapi/k-40-rf.yaml) to explore more operations.
Reuse your access token for subsequent requests.

## List and revoke tokens

Use `K40_HOST` and `K40_ACCESS_TOKEN` from the [first-request guide](#make-your-first-request) for these requests.
Both requests use port `9442` and require your access token in the `Authorization` header.

### List tokens

Request `/auth/token` to get the token list:

```bash
curl \
  --silent --show-error --fail-with-body \
  --insecure \
  --header "Authorization: Bearer ${K40_ACCESS_TOKEN}" \
  "https://${K40_HOST}:9442/auth/token"
```

A successful response contains a JSON array like this:

```json
[
  {
    "token_type": "Bearer",
    "token_id": "1",
    "client_name": "eu-dataact",
    "created_at": 1775134322
  }
]
```

Use `client_name` to identify the application associated with a token.
Use `token_id` to select a token for revocation.

### Revoke a token

From the [token list](#list-tokens), copy the `token_id` of the token you want to revoke:

```bash
export K40_TOKEN_ID="replace-with-token-id"
```

Send a form-encoded POST request to `/auth/revoke` with that `token_id` and `usage_type=private`:

```bash
curl \
  --silent --show-error --fail-with-body \
  --insecure \
  --request POST "https://${K40_HOST}:9442/auth/revoke" \
  --header "Authorization: Bearer ${K40_ACCESS_TOKEN}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "token_id=${K40_TOKEN_ID}" \
  --data-urlencode "usage_type=private"
```

The revoked token no longer grants API access.

## Troubleshooting

### Gateway discovery fails

1. Make sure that your computer and gateway use the same local network.
2. Search for the exact mDNS service `_hvac-open-api._tcp.local.`.
3. If mDNS returns no result, look for the gateway in your router's device list.
4. Update `K40_HOST` with the gateway's current hostname or IP address.

### Hostname resolution fails

Errors such as `BIO_lookup_ex`, `Could not resolve host`, or `Resolving timed out` indicate a hostname resolution failure.
This failure occurs before the HTTPS connection.

1. Make sure that `K40_HOST` contains the resolved hostname, not the service instance name.
2. If hostname resolution still fails, use the [gateway's IP address](#option-2-use-the-gateways-ip-address).

### Authentication works but resource requests fail

The gateway serves authentication and API resources on separate ports.
A successful token request on port `9442` does not establish connectivity to port `9443`.

1. Make sure that `/auth/*` requests use port `9442`.
2. Make sure that resource requests use port `9443`.
3. Make sure that your firewall and network isolation rules permit connections to both ports.

### Known limitation: HTTPS certificate verification

The gateway can present a certificate with a device identifier instead of its mDNS hostname or IP address.
In this case, HTTPS hostname verification fails even if the client trusts the certificate.

Use `--insecure` for requests to ports `9442` and `9443`, as shown in [step 2](#2-use-https-with-certificate-verification-disabled).
The connection remains encrypted, but the client cannot verify the gateway's identity.

### HTTP errors

| Status | Error or symptom              | What to do                                                                                                             |
| ------ | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `400`  | `invalid_grant`               | Make sure that `K40_LOGIN` and `K40_PASSWORD` match the label. Remove all hyphens from the password.                   |
| `401`  | `Unauthorized`                | Make sure that your bearer token is valid and the request uses the correct port and header.                            |
| `403`  | `Forbidden`                   | Make sure that `/auth/*` requests use port `9442`. Otherwise, your token lacks permission for the request.             |
| `412`  | `physical_proximity_unproven` | Repeat the [button sequence and token request](#4-create-your-access-token) within the five-minute window.             |
| `415`  | `Unsupported Media Type`      | Send token requests as `application/x-www-form-urlencoded`.                                                            |
| `507`  | `Token database full`         | [Revoke unused tokens](#revoke-a-token) before you create another token.                                                |

For other HTTP errors, see the relevant operation in the [API reference](../openapi/k-40-rf.yaml).

### Gateway status

Your gateway must be paired with your appliance and connected to the local network.
A blinking green LED indicates no connection to the Bosch server infrastructure.
It does not prevent local API access if the local network connection works.

For factory-reset instructions or device support, use your [product documentation](#supported-products).

## Next steps

- Explore all available operations in the [OpenAPI specification](../openapi/k-40-rf.yaml).
- Ask questions or share your feedback in [GitHub Discussions](https://github.com/bosch-home-comfort/api-docs/discussions).
