---
caip: 285
title: JSON-RPC Provider Lifecycle Methods for Session Management
author: [Alex Donesky] (@adonesky1)
discussions-to: TBD
status: Draft
type: Standard
created: 2024-06-07
requires: 25, 217
---

## Simple Summary

CAIP-285 introduces new "lifecycle" methods and a new event for managing authorizations within an existing CAIP-25 session, allowing for addition, revocation, and retrieval of authorizations. These methods provide an alternative to session management via `sessionId`s, allowing `sessionId`s to be optional for CAIP-25.

## Abstract

This proposal aims to extend the CAIP-25 standard by defining new JSON-RPC methods for managing the lifecycle of authorizations within a session. These methods allow dapps and wallets to dynamically adjust authorizations, providing more granular control and better user experience. Additionally, it allows for session management without mandatory sessionIds, offering more flexibility in handling sessions.

## Motivation

The motivation behind this proposal is to enhance the flexibility of CAIP-25 by enabling management of session authorizations without sessionIds which don't map well to extension based wallet's dapp connections and could therefore add constraints and burdens to existing flows. The proposed methods provide an intuitive way to add, revoke, and retrieve authorizations within an existing session, simplifying the management of session lifecycles.

### Use Case Scenarios

1. **Dapp Initiated Adding Authorizations To an Existing Session:**

   - **Current Method:** Call `provider_authorize` again with an existing session identifier and new scopes/methods to add. This is actually somewhat ambiguous in CAIP-25 where it’s unclear if incremental authorizations adds should include the full object or just scopes to add: "This object gets updated, extended, closed, etc. by successive calls and notifications, each tagged by this identifier."
   - **Proposed Method:** Use `wallet_updateSession` to add new authorizations to the existing session. Alternatively, this could be achieved with more specific language indicating that a subsequent `provider_authorize` call with new scopes/authorizations could simply add to an existing session. If we pursue this latter approach one flow that will need clarity is what happens if a subsequent `provider_authorize` request includes `requiredScopes` which the respondent does not authorize. In this case I would expect that this does not revoke the existing session but rather only the new request is rejected.

2. **Wallet Initiated Adding Authorizations To an Existing Session:**

   - **Current Method:** CAIP-25 does not make it very clear how a respondent (wallet), can modify the authorizations of an existing session. The following excerpt is the closest we get: "The properties and authorization scopes that make up the session are expected to be persisted and tracked over time by both parties in a discrete data store, identified by an entropic identifier assigned in the initial response. This object gets updated, extended, closed, etc. by successive calls and notifications, each tagged by this identifier."
   - **Proposed Method:** Wallet publishes and caller/dapp listens for an event `wallet_sessionChanged` with the new full sessionScope.

3. **Wallet Initiated Authorizations Revocation:**

   - **Current Method:** "If a respondent (e.g. a wallet) needs to initiate a new session, whether due to user input, security policy, or session expiry reasons, it can simply generate a new session identifier to signal this notification to the calling provider."
   - Given this language it is unclear if a wallet can revoke authorizations without creating a new session. It is also therefore unclear if a wallet can revoke some subset of authorizations without creating a new session.
   - **Proposed Method:** Wallet publishes and caller/dapp listens for an event `wallet_sessionChanged` with the new full sessionScope.

4. **Dapp Initiated Authorizations Revocation:**

   - **Current Method:** "if a caller [dapp] needs to initiate a new session, it can do so by sending a new request without a sessionIdentifier."
   - **Proposed Method:** Use `wallet_revokeSession` to revoke specific authorizations or the entire existing session. A revokation of methods/notifications/accounts is achieved by sending the sessionScope with the methods/notifications/accounts to revoke. A revokation of a full scope is achieved by sending an empty object for that scope. A full revokation of the session is achieved by sending an empty object for the scopes param.

5. **Retrieving Current Session Authorizations:**

   - **Current Method:** Wallet and app both persist configuration across updates.
   - **Proposed Method:** Use `wallet_getSession` to retrieve the current authorizations of the session.

## Specification

### New Methods

#### `wallet_updateSession`

Adds new authorizations to an existing session.

**Parameters:**

- `sessionId` (string, optional): The session identifier.
- `scopes` (object, required): An object containing the new scopes to be added, formatted according to CAIP-217.

**Initial Session Scopes:**

```json
{
  "eip155:1": {
    "methods": ["eth_signTransaction"],
    "notifications": ["accountsChanged"],
    "accounts": ["eip155:1:0xabc123"]
  },
  "eip155:137": {
    "methods": ["eth_sendTransaction"],
    "notifications": ["chainChanged"],
    "accounts": ["eip155:137:0xdef456"]
  }
}
```

**Example Request:**

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "wallet_updateSession",
  "params": {
    "scopes": {
      "eip155:1": {
        "methods": ["eth_sendTransaction"],
        "notifications": ["chainChanged"]
      }
    }
  }
}
```

**Example Response:**

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "success": true,
    "updatedScopes": {
      "eip155:1": {
        "methods": ["eth_signTransaction", "eth_sendTransaction"],
        "notifications": ["accountsChanged", "chainChanged"],
        "accounts": ["eip155:1:0xabc123"]
      },
      "eip155:137": {
        "methods": ["eth_sendTransaction"],
        "notifications": ["chainChanged"],
        "accounts": ["eip155:137:0xdef456"]
      }
    }
  }
}
```

**Explanation:**

- The `wallet_updateSession` method adds the specified scopes to the current session's authorizations. If the scope already exists, the new methods and notifications are merged with the existing ones.

#### `wallet_revokeSession`

Revokes authorizations from an existing session.

**Parameters:**

- `sessionId` (string, optional): The session identifier.
- `scopes` (object, required): An object containing the scopes to be revoked, formatted according to CAIP-217.

**Initial Session Scopes:**

```json
{
  "eip155:1": {
    "methods": ["eth_signTransaction", "eth_sendTransaction"],
    "notifications": ["accountsChanged", "chainChanged"],
    "accounts": ["eip155:1:0xabc123", "eip155:1:0xdef456"]
  },
  "eip155:137": {
    "methods": ["eth_sendTransaction"],
    "notifications": ["chainChanged"],
    "accounts": ["eip155:137:0xdef456"]
  }
}
```

**Example Request:**

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "wallet_revokeSession",
  "params": {
    "scopes": {
      "eip155:1": {
        "methods": ["eth_sendTransaction"],
        "notifications": ["chainChanged"],
        "accounts": ["eip155:1:0xabc123"]
      }
    }
  }
}
```

**Example Response:**

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "success": true,
    "updatedScopes": {
      "eip155:1": {
        "methods": ["eth_signTransaction"],
        "notifications": ["accountsChanged"],
        "accounts": ["eip155:1:0xdef456"]
      },
      "eip155:137": {
        "methods": ["eth_sendTransaction"],
        "notifications": ["chainChanged"],
        "accounts": ["eip155:137:0xdef456"]
      }
    }
  }
}
```

**Explanation:**

- The `wallet_revokeSession` method removes the specified methods, notifications, and accounts from the current session's authorizations. If the scope no longer has any methods, notifications, or accounts after revocation, it is removed from the session. If no scopes remain in the session, the session is considered closed. If no accounts remain in a scope, and only write methods the entire scope is removed.

#### `wallet_getSession`

Retrieves the current authorizations of an existing session.

**Parameters:**

- `sessionId` (string, optional): The session identifier.

**Initial Session Scopes:**

```json
{
  "eip155:1": {
    "methods": ["eth_signTransaction"],
    "notifications": ["accountsChanged"],
    "accounts": ["eip155:1:0xabc123"]
  },
  "eip155:137": {
    "methods": ["eth_sendTransaction"],
    "notifications": ["chainChanged"],
    "accounts": ["eip155:137:0xdef456"]
  }
}
```

**Example Request:**

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "wallet_getSession",
  "params": {}
}
```

**Example Response:**

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "scopes": {
      "eip155:1": {
        "methods": ["eth_signTransaction"],
        "notifications": ["accountsChanged"],
        "accounts": ["eip155:1:0xabc123"]
      },
      "eip155:137": {
        "methods": ["eth_sendTransaction"],
        "notifications": ["chainChanged"],
        "accounts": ["eip155:137:0xdef456"]
      }
    }
  }
}
```

**Explanation:**

- The `wallet_getSession` method returns the current authorizations for the session. It lists all scopes along with their methods, notifications, and accounts.

### Events

#### `wallet_sessionChanged`

This event is published by the wallet to notify the caller/dapp of updates to the session authorization scopes. The event payload indicates how the scopes have changed, showing additions and removals in the authorizations. If a connection between the wallet and the caller/dapp is severed and the possiblity of missed events arises, the caller/dapp should immediately call `wallet_getSession` to retrieve the current session scopes.

**Event Payload:**

- `sessionId` (string, optional): The session identifier.
- `sessionScopes` (object of `scopeObject` objects, required): An object containing the updated session scopes, formatted according to CAIP-217.

**Initial Session Scopes:**

```json
{
  "sessionScopes": {
    "eip155:1": {
      "methods": ["eth_signTransaction"],
      "notifications": ["accountsChanged"],
      "accounts": ["eip155:1:0xabc123"]
    },
    "eip155:137": {
      "methods": ["eth_sendTransaction"],
      "notifications": ["chainChanged"],
      "accounts": ["eip155:137:0xdef456"]
    }
  }
}
```

**Example Event:**

```json
{
  "method": "wallet_sessionChanged",
  "params": {
    "sessionScopes": {
      "eip155:1": {
        "methods": ["eth_signTransaction", "eth_sendTransaction"],
        "notifications": ["accountsChanged"],
        "accounts": ["eip155:1:0xabc123"]
      },
      "eip155:137": {
        "methods": ["eth_sendTransaction"],
        "notifications": [],
        "accounts": ["eip155:137:0xdef456"]
      }
    }
  }
}
```

This event indicates how the scopes have changed by comparing the updated scopes with the initial session scopes. In the example, the method `eth_sendTransaction` was added to `eip155:1`, and the notification `chainChanged` was removed from `eip155:137`.

### Optional SessionIds

The `sessionId` parameter in the new lifecycle methods is optional. When not provided, the methods will operate on the current active session. This approach allows for more flexible session management without the overhead of tracking session identifiers.

## Security Considerations

The introduction of these lifecycle methods must ensure that only authorized parties can modify the authorizations of a session. Proper authentication and authorization mechanisms must be in place to prevent unauthorized access or modifications.

## Privacy Considerations

Managing authorizations within an existing session reduces the need to create multiple session identifiers, which can help minimize the exposure of session-related metadata. However, care must be taken to handle these methods in a way that does not inadvertently leak sensitive information.

## Changelog

- 2024-06-07: Initial draft of CAIP-285.

## Links

- [CAIP-25](https://chainagnostic.org/CAIPs/caip-25)
- [CAIP-217](https://chainagnostic.org/CAIPs/caip-217)

## Copyright

Copyright and related rights waived via [CC0](../LICENSE).
