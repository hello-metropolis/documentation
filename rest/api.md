# Metropolis
## REST API

### Authentication

#### Bearer Token

The Metropolis REST API requires you pass a JWT token to authenticate.  

This example will use the following values as example `public` and `private` keys, but you should replace them with your own keys.


|---|---|
| Example Public Key  | `03c69016`  |
| Example Private Key  | `ab86e391`  |


**Header**

The header of the JWT looks like:

```json
{"alg":"HS256","typ":"JWT"}
```

**Body**

The body of the JWT includes both the timestamp for when the token was issued at, `iat`, and also your account's public key as a `sub`.

```json
{"iat":1591991988, "sub":"03c69016"}
```

**Signature**

Concatenate the header and payload separated by a period, then take the SHA256 HMAC with your secret key and base64 encode the value.  

```
payload = '{"alg":"HS256","typ":"JWT"}.{"iat":1591991988, "sub":"03c69016"}'
signature = sha256hmac(payload, secret)
```

**Full Token**

Base64 encode each of the pieces and concatenate them.  Example:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE1OTE5OTE5ODgsInN1YiI6IjAzYzY5MDE2In0.3C_obnCb31-2s_hICOG_DQNIO9j_gwLkivcPKbm-O8I
```

This token will be used as the `Bearer Token` in API requests.

**Generating Tokens**

You can use [JWT.io](https://jwt.io/) to test your implementation or use any JWT library to generate a token to use.

#### Headers

Each API request should include the HTTP Header `Authorization: Bearer BEARER_TOKEN` and replace the `BEARER_TOKEN` value with your generated token.

For example, this could look like:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE1OTE5OTE5ODgsInN1YiI6IjAzYzY5MDE2In0.3C_obnCb31-2s_hICOG_DQNIO9j_gwLkivcPKbm-O8I
```

**Errors**

| Message | Description |
|---|---|
| `Missing authorization header`  | Means that the authorization header has not been passed through the API request at all.  |
|  `Token expired` |  If the token is valid, but was generated too long ago to be value the message will indicate this and also share the age of the token. |
|  `Invalid signature` |  Indicates that the signature of the JWT does not match what is expected.  This message can come up for any reason that the JWT is passed as expected but is not a valid one. |



#### Trigger Events

Trigger events can be registered with Metropolis server.  If there are no actions to take (`event_links` that are keyed off the event) the event will be stored in the database, but no updates to sandboxes will happen.

However, if there are event links registered relevant actions will be applied to either the deployment or composition.

`POST /api/trigger_events`

|   |   |
|---|---|
| ref | The branch or tag ref that triggered the workflow run.  This is a value that `git checkout` will respect. |
| branch | The branch or tag that triggered the workflow run. |
| sha | The specific SHA hash for the action that happened. |
| actor | The user who triggered the workflow. |
| repo | An identifier for the GitHub workflow. |
| event_name | Type of an event that is being triggered.  This must be one of `pull_request`, `push` or `delete_branch`. |
| source | Source of the Triggered Event.  Must be `github action`. |


#### Example

Replace `PUBLIC_KEY` and `SECRET` with your values and the following bash code can be used to trigger an API request from your command line.

```bash
PUBLIC_KEY="03c69016"
SECRET="ab86e391"

HEADER=`echo -n '{"alg":"HS256","typ":"JWT"}' | openssl base64 -e -A | sed s/\+/-/ | sed -E s/=+$//`
TIMESTAMP=`date +%s`
PAYLOAD=`echo -n "{\"iat\":$TIMESTAMP,\"sub\":\"${PUBLIC_KEY}\"}" | openssl base64 -e -A| sed s/\+/-/ | sed -E s/=+$// `
SIGNATURE=`echo -n "$HEADER.$PAYLOAD" | openssl dgst -sha256 -hmac $SECRET -binary | openssl base64 -e -A | sed s/\+/-/ | sed -E s/=+$// | sed 's/\//_/g'`
BEARER_TOKEN="$HEADER.$PAYLOAD.$SIGNATURE"

curl http://hellometropolis.com/api/trigger_events -H "Content-Type: application/json" -H "Authorization: Bearer $BEARER_TOKEN" -d \
 "{\"branch\":\"refs/heads/branch123\",\"sha\":\"0888228bd58b5e5bd9f8d1e5cfed223cfc6ab55e\",\"actor\":\"kenmazaika\",\"repo\":\"hello-metropolis/quickstart\",\"event_name\":\"push\",\"source\":\"github action\"}"
```
