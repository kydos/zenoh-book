# Security

This chapter covers Zenoh's security features: TLS transport, mutual authentication, access control lists, and zero-trust configuration.

## TLS Transport

Zenoh supports TLS 1.3 on the `tls://` transport endpoint. TLS encrypts all data in transit and authenticates the server to the client. Use TLS whenever Zenoh nodes communicate over an untrusted network.

Configure a router to listen with TLS:

```json5
// zenohd config with TLS
{
  mode: "router",
  listen: {
    endpoints: ["tls/0.0.0.0:7448"]
  },
  transport: {
    unicast: {
      tls: {
        server_certificate: "/etc/zenoh/server.pem",
        server_private_key: "/etc/zenoh/server_key.pem",
        root_ca_certificate: "/etc/zenoh/ca.pem"
      }
    }
  }
}
```

Client connecting with TLS:

```json5
{
  mode: "client",
  connect: {
    endpoints: ["tls/router.example.com:7448"]
  },
  transport: {
    unicast: {
      tls: {
        root_ca_certificate: "/etc/zenoh/ca.pem"
      }
    }
  }
}
```

The client verifies the server certificate against `root_ca_certificate`. If verification fails, the connection is rejected.

## Mutual Authentication (mTLS)

Mutual TLS requires both the server and the client to present certificates. The server verifies the client certificate, enabling identity-based access decisions at the transport layer.

Server config — require client certificate:

```json5
{
  transport: {
    unicast: {
      tls: {
        server_certificate: "/etc/zenoh/server.pem",
        server_private_key: "/etc/zenoh/server_key.pem",
        root_ca_certificate: "/etc/zenoh/ca.pem",
        client_auth: true
      }
    }
  }
}
```

Client config — provide certificate:

```json5
{
  transport: {
    unicast: {
      tls: {
        client_certificate: "/etc/zenoh/client.pem",
        client_private_key: "/etc/zenoh/client_key.pem",
        root_ca_certificate: "/etc/zenoh/ca.pem"
      }
    }
  }
}
```

With `client_auth: true`, the router rejects any client that does not present a valid certificate signed by the configured CA. The certificate's Common Name (CN) is available to the ACL subsystem for per-identity rules.

## Access Control Lists

Zenoh supports per-certificate, per-interface, and per-key-expression access control. Configure ACLs in the router JSON5 config.

```json5
{
  access_control: {
    enabled: true,
    default_permission: "deny",
    rules: [
      {
        id: "sensor-pub",
        messages: ["put", "delete"],
        key_exprs: ["sensors/**"],
        flows: ["egress"],
        permission: "allow"
      },
      {
        id: "sensor-sub",
        messages: ["declare_subscriber"],
        key_exprs: ["sensors/**"],
        flows: ["ingress"],
        permission: "allow"
      }
    ],
    subjects: [
      {
        id: "sensor-node",
        interfaces: ["eth0"],
        cert_common_names: ["sensor-device-01"]
      }
    ],
    policies: [
      {
        subjects: ["sensor-node"],
        rules: ["sensor-pub", "sensor-sub"]
      }
    ]
  }
}
```

With `default_permission: "deny"`, only operations explicitly listed in an `"allow"` rule proceed. This is the zero-trust posture: deny everything, permit only what is explicitly needed.

### ACL Fields

| Field | Description |
|-------|-------------|
| `messages` | Message types: `put`, `delete`, `declare_subscriber`, `declare_queryable`, `get` |
| `key_exprs` | Key expression patterns this rule applies to |
| `flows` | `ingress` (received by router) or `egress` (sent from router) |
| `cert_common_names` | mTLS certificate CN values for this subject |
| `interfaces` | Network interface names for this subject |

## Token-Based Authentication

Zenoh supports username/password authentication in environments where certificate management is not feasible.

```json5
{
  transport: {
    unicast: {
      auth: {
        usrpwd: {
          user: "zenoh-client",
          password: "s3cr3t",
          dictionary_file: "/etc/zenoh/users.json5"
        }
      }
    }
  }
}
```

The `dictionary_file` maps usernames to hashed passwords. Token-based auth is a lower-security option compared to mTLS: it does not provide per-message integrity or forward secrecy unless combined with TLS.

## Zero-Trust Summary

A zero-trust Zenoh deployment combines all of the above:

1. All transports use TLS with mutual authentication (`client_auth: true`).
2. Access control is enabled with `default_permission: "deny"`.
3. Explicit allowlist rules are defined per certificate identity and key expression.
4. No unauthenticated peer connections are permitted.
5. Admin space write access is disabled or protected by ACL rules.

This configuration ensures that a compromised node cannot publish to unauthorized topics, subscribe to data it does not own, or modify router configuration at runtime.
