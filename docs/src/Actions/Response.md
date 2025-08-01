# Getting the Login Result from SIWF

After the user completes the authentication and authorization, the user is redirected to the callback URL with one additional parameter set: `authorizationCode`.

The `authorizationCode` can be used to retrieve the result of the login.

Note: The following instruction examples are based on Frequency Access.

## Quick Reference

- Staging-Testnet: `https://testnet.frequencyaccess.com/siwa/api/payload`
- Production-Mainnet: `https://www.frequencyaccess.com/siwa/api/payload`
- Response Structure: [`SiwfResponse`](../DataStructures/All.md#response)

## Step 1: Retrieve the Response

- `GET`:
  - Staging-Testnet: `https://testnet.frequencyaccess.com/siwa/api/payload?authorizationCode=[Parameter from callback URL]`
  - Production-Mainnet: `https://www.frequencyaccess.com/siwa/api/payload?authorizationCode=[Parameter from callback URL]`

## Step 2: Parsing the Response

Response Sections

- `userPublicKey`: The key for the user signing this request. This can be a Sr25519 public key or an Ethereum address.
- `payloads`: Signed payloads from the user
- `credentials`: User-approved, verified credentials from Frequency Access (or other SIWF-compatible services) such as email, phone, user keys, etc...

### `userPublicKey`

The user for the current session is identified via the public key.
If the user has a Frequency blockchain account (MSA), the user’s MSA Id can be retrieved from Frequency via this key.
If the user does not have an account, the payloads section with contain the payload to create the MSA Id.

While the `userPublicKey` may change, the MSA Id will _always_ be the same for the same user account.

## Step 3: Processing the Credentials

The `credentials` array will contain any [requested](../SignatureGeneration.md#step-3-optional-request-credentials-graph-key-email-phone) and approved credentials.
Each credential would be matched based on the `type` field.

These credentials follow the [DSNP Verifiable Credentials Specification](https://spec.dsnp.org/VerifiableCredentials/Overview.html).

- [`@dsnp/verifiable-credentials`](https://github.com/LibertyDSNP/dsnp-verifiable-credentials) TypeScript library for verifying these DSNP Credentials specifically
- Other [W3C Verifiable Credentials](https://www.w3.org/TR/vc-data-model-1.1/) verifiers may also be compatible.

### Verify the Credential

_Trust Model Note_: You may choose to just trust credentials issued by Frequency Access (or other SIWF-compatible services) given that the credential is fetched directly. These will have issuer `did:web:testnet.frequencyaccess.com` or `did:web:frequencyaccess.com`.
Note that some credentials such as `VerifiedGraphKeyCredential` or `VerifiedRecoverySecretCredential` do not need a proof.

#### Sr25519
1. Check that the `credentialSubject.id` matches the `userPublicKey` following the [`did:key` Method from the W3C](https://w3c-ccg.github.io/did-key-spec/#format)
  - Example: `f6cL4wq1HUNx11TcvdABNf9UNXXoyH47mVUwT59tzSFRW8yDH` is the [SS58](https://docs.substrate.io/reference/address-formats/) version with prefix `90` of the hex address `0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d`. `0xef01d43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d` is multicodec `sr25519-pub` hex which is multiformat `base58-btc` `z6QNzHod3tSSJbwo4e5xGDcnsndsR9WByZzPoCGdbv3sv1jJ`
2. Fetch the issuer DID Document following the [`did:web` Method from the W3C](https://w3c-ccg.github.io/did-method-web/)
  - Production-Mainnet should always be `did:web:frequencyaccess.com` which resolves to `https://frequencyaccess.com/.well-known/did.json`
  - Staging-Testnet should always be `did:web:testnet.frequencyaccess.com` which resolves to `https://testnet.frequencyaccess.com/.well-known/did.json`
3. Check that the key in the `proof.verificationMethod` is in the DID Document to verify that the key is still valid
4. Test that the `proof` validates according the to [W3C Verifiable Credentials Specification](https://www.w3.org/TR/vc-data-model-1.1/#verification)

#### Secp256k1
1. Check that the `credentialSubject.id` matches the `userPublicKey` following the [`did:ethr` definition](https://github.com/uport-project/ethr-did)
   - Example: `0x34c20Ac587999E44AFC39A239b8AB9f243e73c2A` is the Ethereum address associated with used `Secp256k1` key is represented as `did:ethr:0x34c20Ac587999E44AFC39A239b8AB9f243e73c2A`
2. Currently, there is no proof associated with these keys.

#### Graph Encryption Key Credential

If the user has permitted access to their private social graph per the request for `VerifiedGraphKeyPairCredential`, `credentials` will have a entry with `type` including `VerifiedGraphKeyPairCredential`.
This is an `x25519` key pair for use with the `curve25519xsalsa20poly1305` encryption algorithm from the [NaCl Library](http://nacl.cr.yp.to).
For more information on how the Graph data is structured after decryption, see the [DSNP Specification](https://spec.dsnp.org/DSNP/UserData.html).

This key pair is PII and should be stored with care, but must be stored to read the user's private graph. 
This credential does not need a proof.

#### Recovery Secret Credential
If the user wanted to set a recovery secret on-chain to recover their account in case of losing access, they can use this
verified credential.
For more information about the Recovery secret and how it is generated, see the [design doc](https://github.com/frequency-chain/frequency/blob/main/designdocs/recovery_system.md).
This credential does not need a proof.

### Example Credential

{{#include ../DataStructures/VerifiedPhone.md 0}}

## Step 4: Processing the Payloads

The payload section has several different types that can be returned:

- `login`: When the user already has a correct delegation
- `addProvider`*: When the user does not have a delegation (a new user signup) or when the delegation was needing to be changed
- `itemActions`*: When the user has user chain data to set or update
- `claimHandle`*: When the user needs to claim a new handle
- `recoveryCommitment`*: When the user needs to add a recovery commitment

\* Requires submission to Frequency

The payloads that require submission to Frequency should be submitted in one batch using [Capacity](https://docs.frequency.xyz/Tokenomics/ProviderIncentives.html#capacity-model).
The `addProvider` *must* always be *first* in the batch to ensure the correct permissions and delegations are in place for subsequent actions in the batch.

Signatures to Frequency have an expiration set to a future Frequency block number.
If the actions are not submitted before expiration, Frequency will reject the transactions and your application will need to request new signatures.

[Payload Examples and Signature Details](../Payloads.md)

### `login`

The correct delegation for the user already exists.
No submission to the chain is required, but the application *must* validate the signature to be sure that the user is authenticated.

The message signed follows [CAIP-122: Sign in With X](https://chainagnostic.org/CAIPs/caip-122) specification which is derived from [EIP-4361: Sign-In with Ethereum](https://eips.ethereum.org/EIPS/eip-4361).

#### Example Message with Placeholders

```text
{{domain}} wants you to sign in with your Frequency account:
frequency:{{chainReference}}:{{ss58Address or eip-55Address}}

URI: {{uri}}
Version: {{version}}
Nonce: {{nonce}}
Chain ID: frequency:{{chainReference}}
Issued At: {{issued-at}}
```

Inside the message, `{{domain}}` is the domain of the application requesting the sign in. `{{domain}}` should match the domain contained in the `URI` field.

#### Validation Steps

1. Perform an Sr25519/Secp256k1 signature verification using:
    - `userPublicKey`: The signing key or address
    - `payload.message`: The signed message parsing `\n` into `LF` line breaks
    - `signature.encodedValue`: The signature
2. Verify that the `userPublicKey` matches the second line of the message for the correct chain identifier
    - Production-Mainnet: `frequency:mainnet:`
    - Staging-Testnet: `frequency:testnet-paseo:`
3. Verify that the message starts with your domain based on [RFC 4501](https://www.rfc-editor.org/rfc/rfc4501.html) `dnsauthority`
4. Verify the `Issued At` value from the message is within your acceptable time range
5. Verify that `Nonce` value in the message is not being reused

#### Parsing the Message Data

1. Break the lines by `\n`
2. Match based on the prefix:
    - `Nonce: `
    - `Issued At: `
    - `URI: `

### `addProvider`

The user either needs a new delegation or a new MSA created with the delegation.

The `endpoint.extrinsic` field will distinguish between the two:
- `createSponsoredAccountWithDelegation`: New MSA
- `grantDelegation`: New/Updated Delegation

[See Frequency Documentation](https://frequency-chain.github.io/frequency/pallet_msa/index.html#extrinsics) on forming the transaction for the extrinsics.

### `itemActions`

Item actions will update the user's chain data for things such as their public key for the encrypted graph.
These actions *must* be submitted to the chain for the correct functioning of private graph and other systems.

[See Frequency Documentation](https://frequency-chain.github.io/frequency/pallet_stateful_storage/index.html#extrinsics) on forming the transaction for the extrinsics.


### `claimHandle`

The user wishes to claim a Frequency Handle.

[See Frequency Documentation](https://frequency-chain.github.io/frequency/pallet_handles/index.html#extrinsics) on forming the transaction for the extrinsics.

## Step 5: Frequency Transaction Submission

All payloads that are returned will be able to be processed using [Capacity on Frequency](https://frequency-chain.github.io/frequency/pallet_frequency_tx_payment/index.html#extrinsics).

Frequency submission is required for any of the following payloads:
- `addProvider`
- `itemActions`
- `claimHandle`
- `recoveryCommitment`

They can be submitted to the chain in one transaction using [`pay_with_capacity_batch_all`](https://frequency-chain.github.io/frequency/pallet_frequency_tx_payment/index.html#extrinsics).

## Step 6: Session Starts

Once the payloads have been validated, the user's authenticated session may start.
SIWF services do not manage user sessions.

### Examples

## New Frequency User Response

### Sr25519
{{#include ../DataStructures/Sr25519/Response-NewUser.md 0}}

### Secp256k1
{{#include ../DataStructures/Secp256k1/Response-NewUser.md 0}}

## New Application/Delegation Response

### Sr25519
{{#include ../DataStructures/Sr25519/Response-NewProvider.md 0}}

### Secp256k1
{{#include ../DataStructures/Secp256k1/Response-NewProvider.md 0}}

## Login Only Response

### Sr25519
{{#include ../DataStructures/Sr25519/Response-LoginOnly.md 0}}

### Secp256k1
{{#include ../DataStructures/Secp256k1/Response-LoginOnly.md 0}}