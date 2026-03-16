# satspay-registry

The SatsPay phone-to-wallet registry. Maps hashed phone numbers to Stacks wallet addresses so that once a recipient has claimed once and registered their wallet, all future sends to their phone number go **directly** to their wallet — no escrow, no claim link, instant delivery.

---

## Overview

The registry solves a UX problem: the first time you send to a phone number, the recipient has to open a link and connect a wallet. That's fine. But the 10th time you send to the same person — your employee, your supplier, your family member — they shouldn't have to do anything. The money should just arrive.

```
First send:
  Sender → phone number → escrow → recipient claims → registers wallet

All future sends:
  Sender queries registry → gets recipient's address → sends directly (no escrow)
```

The registry is also how SatsPay respects privacy: phone numbers are stored as SHA-256 hashes, not plaintext. The contract knows *that* a phone number is registered, but not *which* phone number it is.

---

## Contract Architecture

### Storage

```clarity
;; Maps hashed phone numbers to wallet addresses
(define-map phone-registry
  { phone-hash: (buff 32) }
  {
    owner: principal,
    registered-at: uint,
    active: bool
  }
)

;; Reverse lookup: wallet address → phone hash (for deregistration)
(define-map address-to-phone
  { owner: principal }
  { phone-hash: (buff 32) }
)
```

---

### Public Functions

#### `register`
Called when a recipient claims for the first time and chooses to register their wallet. Links their phone hash to their Stacks address so future senders can skip escrow.

```clarity
(define-public (register (phone-hash (buff 32))))
```

**Parameters:**
| Parameter | Type | Description |
|---|---|---|
| `phone-hash` | `(buff 32)` | SHA-256 hash of the recipient's phone number |

**What it does:**
1. Verifies `phone-hash` is not already registered to another address
2. Checks the caller (`tx-sender`) doesn't already have a registered phone
3. Writes to `phone-registry` mapping this hash → caller's address
4. Writes to `address-to-phone` for reverse lookup
5. Emits a `phone-registered` print event

**Design note:** Registration is **self-sovereign** — only the recipient can register their own phone. The backend cannot register on their behalf. This is enforced because the contract uses `tx-sender` as the wallet address, so the recipient must sign the transaction themselves.

**Errors:**
| Code | Name | Meaning |
|---|---|---|
| `u100` | `err-already-registered` | This phone hash is already registered to another address |
| `u101` | `err-address-has-phone` | This wallet already has a registered phone number |

---

#### `deregister`
Removes a phone-to-wallet mapping. Called if a user changes phone numbers or wants to unlink their wallet.

```clarity
(define-public (deregister))
```

**No parameters** — the contract uses `tx-sender` to find and remove the registration.

**What it does:**
1. Looks up the caller's phone hash via `address-to-phone`
2. Deletes the `phone-registry` entry
3. Deletes the `address-to-phone` entry
4. Emits a `phone-deregistered` print event

**Errors:**
| Code | Name | Meaning |
|---|---|---|
| `u200` | `err-not-registered` | Caller has no registered phone number |

---

#### `update-address`
Called when a user gets a new wallet but keeps their phone number. Updates the mapping to point to their new address.

```clarity
(define-public (update-address
  (phone-hash (buff 32))
  (new-address principal)))
```

**Parameters:**
| Parameter | Type | Description |
|---|---|---|
| `phone-hash` | `(buff 32)` | Their phone hash (proves they know the phone number) |
| `new-address` | `principal` | Their new Stacks wallet address |

**What it does:**
1. Verifies the current registration for `phone-hash` belongs to `tx-sender`
2. Updates the `phone-registry` to point to `new-address`
3. Updates both `phone-registry` and `address-to-phone` mappings
4. Emits `address-updated` event

**Errors:**
| Code | Name | Meaning |
|---|---|---|
| `u300` | `err-not-registered` | This phone hash isn't registered |
| `u301` | `err-not-owner` | Caller is not the current owner of this registration |

---

### Read-Only Functions

#### `get-address-for-phone`
The most-called function in the protocol. The sender's frontend calls this before every send to check whether the recipient is registered (skip escrow) or not (use escrow).

```clarity
(define-read-only (get-address-for-phone (phone-hash (buff 32)))
  (map-get? phone-registry { phone-hash: phone-hash }))
```

Returns:
```clarity
(some {
  owner: principal,       ;; The recipient's wallet address
  registered-at: uint,    ;; Block height when they registered
  active: bool            ;; Always true for active registrations
})
;; or none if not registered
```

#### `is-registered`
Simple boolean check. Returns `true` if a phone hash has a registered wallet.

```clarity
(define-read-only (is-registered (phone-hash (buff 32)))
  (is-some (map-get? phone-registry { phone-hash: phone-hash })))
```

#### `get-phone-for-address`
Reverse lookup — given a wallet address, what phone hash is it registered to?

```clarity
(define-read-only (get-phone-for-address (address principal))
  (map-get? address-to-phone { owner: address }))
```

---

## Events (Print Statements)

| Event | Emitted by | Payload |
|---|---|---|
| `phone-registered` | `register` | `{ phone-hash, owner, registered-at }` |
| `phone-deregistered` | `deregister` | `{ phone-hash, owner }` |
| `address-updated` | `update-address` | `{ phone-hash, old-address, new-address }` |

---

## How the Send Flow Uses the Registry

In the frontend, before building a transaction, the app queries the registry:

```typescript
import { callReadOnlyFunction, bufferCVFromString } from '@stacks/transactions';
import { sha256 } from 'js-sha256';

async function getSendDestination(phoneNumber: string) {
  const phoneHash = Buffer.from(sha256(normalizePhone(phoneNumber)), 'hex');

  const result = await callReadOnlyFunction({
    contractAddress: 'ST...',
    contractName: 'satspay-registry',
    functionName: 'get-address-for-phone',
    functionArgs: [bufferCV(phoneHash)],
    network,
  });

  if (result.type === ClarityType.OptionalSome) {
    // Recipient is registered — send directly to their wallet
    const recipientAddress = result.value.data.owner;
    return { type: 'direct', address: recipientAddress };
  } else {
    // Not registered — use escrow flow
    return { type: 'escrow', phoneHash };
  }
}
```

---

## Security Considerations

### Phone Number Hashing
Phone numbers must be normalized before hashing to prevent a single phone number from having multiple hashes:

```typescript
function normalizePhone(phone: string): string {
  // Remove all non-digit characters, then add country code
  const digits = phone.replace(/\D/g, '');
  // Ensure leading country code (234 for Nigeria)
  return digits.startsWith('234') ? `+${digits}` : `+234${digits}`;
}

const hash = sha256(normalizePhone(phone)); // always consistent
```

### No Admin Override
The registry has no admin key and no ability to forcibly register or deregister a phone number. This is intentional — the user's registration is sovereign. Not even SatsPay the company can change where your funds are sent.

### One Phone Per Wallet, One Wallet Per Phone
The contract enforces a 1:1 relationship. A phone can only map to one address, and an address can only have one phone. This prevents confusion and double-registration attacks.

---

## Testing

```bash
clarinet test tests/satspay-registry_test.ts
```

### Test Coverage

| Test | Description |
|---|---|
| `register-success` | Valid registration stores mapping correctly |
| `register-duplicate-phone` | Rejects second registration of same phone hash |
| `register-duplicate-address` | Rejects second registration from same wallet |
| `deregister-success` | Removes both mappings |
| `deregister-not-registered` | Rejects deregister if not registered |
| `update-address-success` | Updates wallet address for registered phone |
| `update-address-wrong-owner` | Rejects update from non-owner |
| `get-address-for-phone-found` | Returns correct address for registered phone |
| `get-address-for-phone-not-found` | Returns none for unregistered phone |
| `is-registered-true` | Returns true for registered phone |
| `is-registered-false` | Returns false for unregistered phone |

---

## Deployment

```bash
# Testnet
clarinet deployments apply --testnet

# Mainnet
clarinet deployments apply --mainnet
```

Contract address: `ST<deployer-address>.satspay-registry`

---

## License

MIT — see root [LICENSE](../../LICENSE)
