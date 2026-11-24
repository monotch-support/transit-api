# Authentication

## Encryption

The Transit API is exposed using HTTPS only. SSL (TLS) is used to encrypt all in-flight data.

## Basic authentication

The Transit API uses HTTP Basic authentication. Every request must include the `Authorization` header with the base64 encoding of `username:password`:

```http
Authorization: Basic <base64-encoding of username:password>
```

## Obtaining credentials

Credentials are shared separately. You will receive:

- A **username and password** for Basic authentication
- A **base URL** for the API endpoints; the base URL is specific to each client
