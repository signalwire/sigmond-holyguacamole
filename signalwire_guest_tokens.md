# SignalWire Guest Tokens API Documentation

## Create Subscriber Guest Token

Creates a Subscriber Guest Token for accessing SignalWire Fabric services. The token is authorized using an existing API token and provides limited permissions for guest users.

### Endpoint

```
POST /guests/tokens
```

**Base URL:** `https://{space_name}.signalwire.com/api`

### Authentication

This endpoint requires HTTP Basic Authentication using your SignalWire credentials:
- **Username:** Your Project ID
- **Password:** Your API Token

### Request

#### Headers

```
Content-Type: application/json
Authorization: Basic {base64_encoded_credentials}
```

#### Request Body

| Parameter | Type | Required | Description | Constraints |
|-----------|------|----------|-------------|-------------|
| `allowed_addresses` | `array[string]` | **Yes** | List of UUIDs representing the allowed Fabric addresses | Maximum 10 UUIDs |
| `expire_at` | `string` | No | Expiration date and time of the token in ISO 8601 format | Must be a valid datetime |

#### Example Request

```json
{
  "allowed_addresses": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001"
  ],
  "expire_at": "2024-12-31T23:59:59Z"
}
```

### Response

#### Success Response (201 Created)

**Content-Type:** `application/json`

```json
{
  "token": "eyJhbGciOiJkaXIiLCJlbmMiOiJBMjU2R0NNIiwiY2giOiJwdWMuc2lnbmFsd2lyZS5jb20iLCJ0eXAiOiJTQVQifQ..8O4EJs349q97jAcd.H4GNrC6gsWdz91ArWF9ce00Cm62iHfsrFRRUUGW3e96j9C3IphiJXvHYHTmD4qMt8czZ8cniF8c53vVAIZF-yBQibejiMxwnqW6KkLct2EJoPUf9g-wQwM0-lGGj9iPx_7yprkQekFK-7svkLcKlo1voZyavxIsWQlXByppmR_ospVx2u8jbAab0ZjKJNEnr1yPF9oNkyMAnkpkS8k8PwKaxUHBc5SGumKlexUjL3ixZDR6UOcbApVXxrB-DmQBs3otOT7hzME7oKvR-6Xy0XJ1pt4Of7MEzNBUK5Z5NMjtFiA8IqwDlNJz3I5gn8hbjSZwSMJHRJGx2DKpNKiu6fcd-3i2VwCpnKHaNUybMJ5gV3cTNfTFJQBSearCLv-7gMx6Gqy9FF_Hm2bGlfnjTQ9BCsCqXBkQ9EQD6yboi2uUhPyLmpzPqlrBc9ik0c3qR5ey5Jym_VnZXaT_S5NxjzIjLzvs33GOKiooGMsBWOm6mzTPcf398xaSErT4dF2wXwtZANou7Dt4BoTKa.DcLVYpma-iItaGhaOStu9A"
}
```

#### Response Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | `string` | **Yes** | The generated guest token string |

#### Error Responses

##### 401 Unauthorized
```json
{
  "error": {
    "type": "authentication_error",
    "code": "unauthorized",
    "message": "Access is unauthorized",
    "details": "Invalid or missing authentication credentials"
  }
}
```

##### 404 Not Found
```json
{
  "error": {
    "type": "not_found_error",
    "code": "resource_not_found",
    "message": "The server cannot find the requested resource",
    "details": "The specified endpoint or resource does not exist"
  }
}
```

##### 422 Unprocessable Entity
```json
{
  "error": {
    "type": "validation_error",
    "code": "invalid_parameters",
    "message": "Client error",
    "details": "Invalid request parameters provided",
    "parameter": "allowed_addresses"
  }
}
```

##### 500 Internal Server Error
```json
{
  "error": {
    "type": "server_error",
    "code": "internal_error",
    "message": "Server error",
    "details": "An unexpected error occurred on the server"
  }
}
```

### Usage Examples

#### cURL Example

```bash
curl -X POST \
  https://your-space.signalwire.com/api/guests/tokens \
  -u "PROJECT_ID:API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "allowed_addresses": [
      "550e8400-e29b-41d4-a716-446655440000",
      "550e8400-e29b-41d4-a716-446655440001"
    ],
    "expire_at": "2024-12-31T23:59:59Z"
  }'
```

#### JavaScript (Node.js) Example

```javascript
const axios = require('axios');

const config = {
  method: 'post',
  url: 'https://your-space.signalwire.com/api/guests/tokens',
  auth: {
    username: 'PROJECT_ID',
    password: 'API_TOKEN'
  },
  headers: {
    'Content-Type': 'application/json'
  },
  data: {
    allowed_addresses: [
      '550e8400-e29b-41d4-a716-446655440000',
      '550e8400-e29b-41d4-a716-446655440001'
    ],
    expire_at: '2024-12-31T23:59:59Z'
  }
};

axios(config)
  .then((response) => {
    console.log('Guest token created:', response.data.token);
  })
  .catch((error) => {
    console.error('Error creating guest token:', error.response?.data || error.message);
  });
```

#### Python Example

```python
import requests
import json
from datetime import datetime, timezone

url = "https://your-space.signalwire.com/api/guests/tokens"

payload = {
    "allowed_addresses": [
        "550e8400-e29b-41d4-a716-446655440000",
        "550e8400-e29b-41d4-a716-446655440001"
    ],
    "expire_at": "2024-12-31T23:59:59Z"
}

headers = {
    'Content-Type': 'application/json'
}

response = requests.post(
    url,
    auth=('PROJECT_ID', 'API_TOKEN'),
    headers=headers,
    data=json.dumps(payload)
)

if response.status_code == 201:
    token_data = response.json()
    print(f"Guest token created: {token_data['token']}")
else:
    print(f"Error: {response.status_code} - {response.text}")
```

### Important Notes

1. **Security**: Guest tokens provide limited access to Fabric services. Always use HTTPS for API requests.

2. **Token Expiration**: If no `expire_at` is specified, the token may have a default expiration period. Always specify an appropriate expiration time for security.

3. **Address Limits**: You can specify up to 10 allowed Fabric addresses per guest token.

4. **Token Storage**: Store the returned token securely and never expose it in client-side code or public repositories.

5. **API Token Permissions**: Your API token must have appropriate permissions to create guest tokens.

### Related Endpoints

- **List Guest Tokens**: `GET /guests/tokens`
- **Delete Guest Token**: `DELETE /guests/tokens/{token_id}`
- **Video Room Tokens**: `POST /video/conferences/{room_id}/conference_tokens`

### Support

For additional support and documentation, visit:
- [SignalWire Documentation](https://developer.signalwire.com/)
- [SignalWire Community](https://signalwire.community/)
- [Support Portal](https://support.signalwire.com/)
