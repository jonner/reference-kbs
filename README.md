# reference-kbs

A reference implementation of the [KBS](https://github.com/confidential-containers/kbs/).

## Attestation Server

While the [KBS](https://github.com/confidential-containers/kbs/) doesn't define the location of the component that does the attestation of the TEE measurements, allowing it to be a separate component from KBS, in this implementation that functionality is provided locally.

## Using reference-kbs

The following endpoints are available:

### /register_workload

In order to attest a workload, it must first be registered with the attestation
server. To do this, issue a POST request to the `/register_workload` endpoint.
This endpoint takes the following JSON input:

```json
{
  "workload_id": "my-workload-id",
  "launch_measurement": "string",
  "tee_config": "string",
  "passphrase": "string"
}
```

The launch measurement is inserted into the `measurements` database. The
`tee_config` is inserted into the `configs` database. The `passphrase` is
inserted into the `secrets` database.

### /auth

In order to authenticate a given workload, issue  POST request to the `/auth`
endpoint. This endpoint takes the following JSON data:

#### SEV:
```
{
  "build": {
    "version": {
      "major": "u8",
      "minor": "u8"
    },
    "build": "u8"
  },
  "chain": {
    "ca": {
      "ask": {
        "version": "u32",
        "v1": "v1cert"
      },
      "ark": {
        "version": "u32",
        "v1": "v1cert"
      }
    },
    "sev": {
      "pdh": {
        "version": "u32",
        "v1": "v1cert"
      },
      "pek": {
        "version": "u32",
        "v1": "v1cert"
      },
      "oca": {
        "version": "u32",
        "v1": "v1cert"
      },
      "cek": {
        "version": "u32",
        "v1": "v1cert"
      }
    }
  },
  "workload_id": "my-workload-id"
}
```
#### SNP:
```
{
  "workload_id": "my-workload-id"  
}
```

Verifies that a given workload exists in the database and looks up the TeeConfig
for that workload. It then creates an attester and a session, and returns a
challenge to the requestor. The challenge response contains a nonce in addition
to some TEE-specific data:

```
{
  "nonce": "sessionid",
  "extra-params": "tee-specific"
}
```

### /attest

The `attest` endpoint is used to actually attest a particular workload. The
client should issue an HTTP POST request to the `/attest` endpoing with  a valid
`session_id` cookie. The body of the request should include the following JSON
input:

```
{
  "tee-pubkey": {
    "alg": "string",
    "k-mod": "string",
    "k-exp": "string"
  },
  "tee-evidence": "tee-specific evidence as a string"
}
```

When the server recieves the request, it checks the session_id for a valid
session, and extracts the workload id from the session. The server then fetches
the pre-registered measurement for that workload from the database. It then
compares the given `tee-evidence` against the expected measurement and validates
all of the certificates, etc. It also saves the provided public key so that it
can encrypt secrets to send back to the client later.

### /key/<key_id>

This endpoint allows you fetch secret keys associated with a particular
workload with an HTTP GET request. This endpoint expects a valid session
that has previously been attested. The session is specified via a `session_id`
cookie. The id of the key to fetch is specified in the request URL.

If the session is valid and has already been attested, it looks up the specified
key from the secrets database, encrypts it using the public key that was
supplied during the attestation, and returns it the encrypted value in the
response.
