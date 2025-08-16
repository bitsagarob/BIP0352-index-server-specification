# BIP352 Silent Payments Index Server Specification (WIP)

## Abstract

This specification defines a unified standard for Silent Payments indexing services, enabling wallet developers to implement efficient Silent Payments receive support through standardized APIs. The specification covers three service deployment models and their corresponding API requirements.

## Motivation

The goal of this specification is to provide a unified standard for Silent Payments wallet and service developers to implement receive support. The Electrum protocol provides a great model for how wallet-server communication works today, demonstrating efficient client-server interactions for lightweight Bitcoin wallets. This specification is not meant to replace the Electrum protocol but to extend it to support Silent Payments, leveraging its proven architecture to enhance privacy and efficiency.

Although it might appear that index servers are only necessary for [light client support](https://github.com/setavenger/BIP0352-light-client-specification), reducing scanning operations for wallets improves the user experience, speeds up, and simplifies wallet receiving implementations. Minimizing the time it takes for users to confirm on-chain balance when opening a wallet aligns with existing expectations for user experience. Another benefit of having a common specification for tweak services is that developers can iterate on wallet or server improvements once, without requiring extensive coordination for upgrade compatibility. Additionally users and wallet developers will have the choice to migrate to the best service provider without requiring architectural changes.

## Current Landscape

[blindbit](https://github.com/setavenger/blindbit-oracle), [cake wallet](https://github.com/cake-tech/blockstream-electrs) and [silentiumd](https://github.com/louisinger/silentiumd) have completed indexing server solutions that are being leveraged by wallet developers today. The current approaches successfully dervive tweaks and perform additional scan operations but there is no clear definition of what role each service provides to the wallet. A wallet team could not swap services without signficant changes to wallet design. Once this specification is solidified each of the current implementations will have a framework to align data formats and validate data integrity.

- blindbit-oracle
  - dana-wallet
  - BDK + kyoto
- cake-esplora
  - cake wallet

## Hosted Service Scanner Models

### Self hosted

A user can setup a bitcoin node and add an indexing service to same infrastructure, wallet registers scan private key and silent payment address + birthdate with service for most efficient initialization and UTXO management.

### Hosted Services

Some users may choose to forgo setting up bitcoin node infrastructure and choose to outsource tweaks for a range of blocks. Hosted services providing Block Processor and Block Filter Manager roles can be used anonymously if care is taken by the wallet developer.

## Service Roles

There are three distinct service roles (steps) required to find silent payments. Either the wallet or service has to do the work. The following section details different service components and describes the key role it provides in scanning for payments. Even though the services can be combined into one [Stack](#stacks) of capabilities all standard endpoints for a given service must be implemented. Additional endpoints can be offered but will not be captured in this specification. Each wallet implementation will have to choose which Stack to build based on the wallet / user objectives.

### Roles:

##### Block Processor

- *requires access to full node*
- Compute tweaks for all blocks
- Serve Tweaks and relevant TXOs for each block
- Examples
  - blindbit-oracle, Electrs, ElectrumX, Esplora, Fullcrum, silentiumd

##### Block Filter Manager

- *requires access to full node or BP*
- Serve CBF (BIP158) headers & filters
- Examples
  - blindbit-oracle, kyoto (bdk-sp), silentiumd?

##### SP Scanner

- *requires access to full node or BP or BFM*
- Wallet shares: scan_sk, scan_pk, spend_pk with service
  - Permenant (Hosted) or Ephemeral (Session)
- Find owned UTXOs since last scan or for a range of blocks
- Examples
  - [sp-client::SpScanner](https://github.com/cygnet3/sp-client/blob/master/src/scanner/scanner.rs), [sp-poc](https://github.com/nymius/sp-poc/tree/feat/use_bdk_sp) (Kyoto + bdk-sp)

---

#### Service Role Comparison

|                                             |                                                                                       | Service Roles                                            |                               |
| --------------------------------------------| ------------------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------- |
| **Feature**                                 | **Block Processor**                                                                   | **Block Filter Manager**                                 | **SP Scanner**                |
| *Privacy*                                   | Yes                                                                                   | Yes                                                      | No                            |
| *Security (Trust)*                          | N/A                                                                                   | N/A                                                      | Yes                           |
| *Dust*                                      | Possible                                                                              | Possible                                                 | Possible                      |
| *Spent Tx Filtering*                        | Possible                                                                              | Possible                                                 | Possible                      |
| *Payment Notifications*                     | No                                                                                    | Maybe                                                    | Yes                           |
| *Mempool Monitoring*                        | No                                                                                    | Maybe                                                    | Yes                           |
| *Performance / UX (Offline catch up delay)* | Slow - sync required                                                                  | Depends on wallet implementation                         | Depends on wallet implementation |
| *Server Storage Requirements*               | Large                                                                                 | Small                                                    | Small                         |
| *Client Bandwidth  (wallet <-> service)*    | Large/Moderate                                                                        | Large/Moderate                                           | Small                         |
| *Client Transaction Rate*                   | < 2 per day                                                                           | < 2 per day                                              | unlimited                     |
| *Labels*                                    | No                                                                                    | No                                                       | Yes                           |

## Stacks

Stack implementations combine one or more service roles to create unique offerings for different privacy objectives. Each of these stacks would likely have different endpoint requirements.

### My Scanner (Personalized)

- combines ***SP Scanner + Block Filter Manager + Block Processor***

1. Configured to support a single wallet or small group of wallets
   1. Wallet birthdate limits scanning requirements
   2. Register scan key pair + spend public key with server
2. Trusted service
   1. Wallet can assume that any data received from the service does not need to be verified
   2. Wallet would receive a list of UTXOs
   3. User could be notified when payment is detected by server
3. Technical User
   1. Willing to learn how to optimize for their situation (configure)
4. Additional Services
   1. Wallet Recovery
      1. Fast - based on filtered UTXO set
   2. Mempool Monitoring
   3. Receive Notifications

### Remote Scanner (Ephemeral)

- combines ***SP Scanner + Block Filter Manager + Block Processor***

1. Configured to respond to any wallet
   1. Must index entire chain from Taproot activation
   2. Wallet request UTXOs from service
2. Service is trusted
   1. To preserve forward privacy a user should rotate wallet when leaving service
3. Novice or Technical User
   1. Share scan key pair + spend public key with **server during a session**
   2. Relies on well organized UI/UX to educate user on *"scanning concepts"*
4. Additional Services
   1. Wallet Recovery
      1. Wallet should maintain SP transaction archive
   2. Mempool Monitoring
      1. May be rate limited to a few checks per minute or max per day
   3. Labels
      1. Each additional label requires more compute to scan / transaction
      2. May be rate limited or require additional fee

### Serve Tweaks for any number of users (Anonymous)

- combines ***Block Filter Manager + Block Processor***

1. Configured to respond to any wallet
   1. Must index entire chain from Taproot activation
   2. Wallet request tweaks for a block or range of blocks, use block filters to identify transactions
2. Service cannot be fully trusted
   1. Users should be able to verify received data against other sources or prove legitimacy independently
3. Novice or Technical User
   1. Simple to understand workflow to configure and use (set it and forget)
   2. Relies on well organized UI/UX to educate user on *"scanning concepts"*
4. Wallet Recovery
   1. Wallet should maintain SP transaction archive
   2. Services may rate limit bulk scanning

---

#### Stack Comparison

|                                             |                                                                    | Silent Payment Stack                                                                         |                                                                                                          |
| ------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Feature**                                 | **My Scanner (Personalized)**                                      | **Remote Scanner (Ephemeral)**                                                               | **Tweak Server (Anonymous)**                                                                             |
| *Privacy*                                   | Full <br>(register spend public key & scan private key) <br>Self Hosted  | Privacy from all but Indexer Service <br>(share spend public key & scan private key) for a session<br>Hosted | Mask interest in ScriptPubKeys and Transaction <br>(Care should be taken when filtering for blocks) <br>Hosted |
| *Skill (User)*                              | Experienced / Technical                                            | Novice / Moderate                                                                            | Novice / Moderate                                                                                        |
| *Security (Trust)*                          | Trusted (Self hosted)                                              | Trusted (Paid/Free)                                                                          | Untrusted (Free) <br>Should download full block on filtered match (no simplified utxo)                   |
| *Wallet Requirements*                       | Send / Spend SP <br>[Fetch Payments](#my-scanner-integration) <br>Manage UTXOs | Send / Spend SP <br>[Remote Scan](#remote-scanner-integration) <br>Manage UTXOs          | Send / Spend SP <br>[Scan For ScriptPubKeys](#tweak-server-integration) <br>Manage UTXOs                |
| *Wallet Backup / Recovery*                  | Label backup is necessary, recovering transactions should be quick | Most important <br>Label backup + utxo backup would be best <br>(restore could be rate limited) | Most important <br>Label backup + utxo backup would be best <br>(restore could be rate limited)          |
| *Dust*                                      | wallet preference                                                  | wallet preference                                                                            | wallet preference                                                                                        |
| *Spent Tx Filtering* (Cut Through)          | wallet preference                                                  | wallet preference                                                                            | Possible                                                                                                 |
| *Payment Notifications*                     | Optional                                                           | Optional                                                                                     | Not Feasible                                                                                             |
| *Mempool Monitoring*                        | Optional                                                           | Optional                                                                                     | Not Feasible                                                                                             |
| *Performance / UX (Offline catch up delay)* | Wallet can have same UX as traditional payment                     | Wallet will need to sync (catch up) to find all UTXOs - distributed scan.                    | Wallet will need to sync (catch up) to find all UTXOs                                                    |
| *Server Storage Requirements*               | Medium                                                             | Large                                                                                        | Large                                                                                                    |
| *Client Bandwidth  (wallet <-> service)*    | Low                                                                | Low                                                                                          | Moderate                                                                                                 |
| *Client Transaction Rate*                   | unlimited                                                          | < ~20 day                                                                                    | < 1 day                                                                                                  |
| *Labels*                                    | Supported - register labels with service                           | Managed by wallet                                                                            | Should labels be offered in wallet at all?                                                               |

## Server API endpoints

API endpoints are grouped by Service Roles, thought has been given to include what each implementation should support "Required" vs "Optional".

|                                   |                | Service Roles            | | | | | | |
| --------------------------------- | :------------: | :----------------------: | :-----------------: | -------------------------------------------------------------------------------------------------------- | :-----------------: | :---------------: | :------------: | :--------------: |
| **Endpoint**                      | **SP Scanner** | **Block Filter Manager** | **Block Processor** | ***Description***                                                                                        | **blindbit-oracle** | **blindbit-scan** | **silentiumd** | **cake esplora** |
| /info                             |                |      Required       |      Required         | returns basic information about the indexing server instance                                                |          Y          |        Similar    |     Similar    |                  |
| /block-hash/:blockheight          |                |      Required       |      Required         | returns the block-hash for a certain block-height - used by wallet to detect reorg                          |          Y          |                   |                |                  |
| /tweaks/:blockheight?dustLimit&filterSpent|         |                     |      Required         | returns tweak data; optional parameters filterSpent + dustLimit.                                            |          Y          |          Uses     |     Similar    |     Similar      |
| /filter/utxos/:blockheight        |                |      Required       |                       | returns a BIP 158 compliant filter of script pub keys which received funds                                  |          Y          |          Uses     |                |                  |
| /utxos/:blockheight               |    Optional?   |                     |                       | UTXO data for that block (cut down to the essentials needed to spend)                                       |          Y          |          Uses     |                |                  |
| /filter/spent/:blockheight        |                |      Optional?      |                       | returns a filter for shortened spent outpoint hashes; optional validation of spent outpoints                |          Y          |          Uses     |     Similar    |                  |
| /spent-index/:blockheight         |    Optional?   |                     |                       | returns the spent outpoints index                                                                           |          Y          |          Uses     |                |                  |
| **Proposed** | | | | | | | | |
| /scan/:keys/:block_range/[:labels]  |        Y       | | | scan a range of blocks given scan_sk + sp address; returns UTXOs or busy | | | |
| /scan-bulk/:keys/:block_range/[:labels]|        Y       | | | start async scan for UTXOs; returns a scan token and expected duration   | | | |
| /scan-check/:token                  |        Y       | | | provide scan token to collect UTXOs or updated duration                  | | | |
| /scan-cancel/:token                 |        Y       | | | provide scan token to cancel bulk scan; wallet courtesy                  | | | |
| /scan-mempool/:keys/[:labels]       |        Y       | | | subscribe to unconfirmed scanning given scan_sk + sp address             | | | |
| **Electrum** | | | | | | | | |
| rpc blockchain.tweaks.subscribe   |                |                     |          Y            |                                                                                                             |                     |                   |                |         Y        |
| rpc blockchain.scripthash.get_balance|     Y       |                     |                       |                                                                                                             |                     |                   |                |         Y        |
| rpc blockchain.scripthash.get_history|     Y       |                     |                       |                                                                                                             |                     |                   |                |         Y        |
| **?** | | | | | | | | |
| /register                         |    Optional?   |                     |                       | handles registering new wallets; returns unique wallet identifier                                           |                     |        Y          |                |                  |
| /wallet/utxos?restore             |    Optional?   |                     |                       | serves UTXOs since last check; restore all known UTXOs                                                      |                     |        Y          |                |                  |
| /labels                           |    Optional?   |                     |                       | add or return list of labels                                                                                |                     |        Y          |                |                  |

### API Request/Response Schemas

---

#### Core Information Endpoints (WIP)

##### GET /info

```json
{
  "version": "1.0.0",
  "network": "mainnet|testnet|regtest",
  "block_height": 850000,
  "services": ["block_processor", "block_filter_manager", "sp_scanner"],
  "dust_limit": 546,
  "filter_spent": "optional|N/A" # indicates if server supports optional filtering
}
```

##### GET /block-hash/:blockheight

```json
{
  "height": 850000,
  "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
  "status": 0|1
}
```

#### Tweak Data Endpoints

##### GET /tweaks/:blockheight?dustLimit=:amount&filterSpent=:bool

```json
{
  "height": 850000,
  "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
  "dust_limit": 546, # zero indicates disabled, requested limit is echoed for confirmation
  "filter_spent": 0|1, # zero indicates off, one == on
  "tweaks": [
     "02abc123...",
     "03def456...",
  ],
  "checksum or count": 1234,
}
```

#### Filter Endpoints

##### GET /filter/utxos/:blockheight

```json
{
  "height": 850000,
  "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
  "data": "hex_encoded_filter_bytes",
}
```

##### GET /filter/spent/:blockheight

```json
{
  "height": 850000,
  "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
  "data": "hex_encoded_filter_bytes",
  "checksum or data length": 12345,
}
```

#### Ephemeral Scanner Service Endpoints

##### POST /scan/:keys/:block_range/[:labels]

Request:
```json
{
  "scan_pk": "02abc123...",  
  "scan_sk": "02abc123...",
  "spend_pk": "03def456...",
  "start_height": 800000,
  "end_height": 800002,
  "labels_index" : [1, 2, 3] # optional label index
}
```
Response (complete):
```json
{
  "status": "complete",
  "utxos": [
    {
      "txid": "a1b2c3...",
      "output_index": 0,
      "value": 100000,
      "output_pubkey": "03def456...", #script pub key
      "height": 850000,
      "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
      "spent": false
    }
  ]
}
```
Response (busy):
```json
{
  "status": "busy",
  "message": "try bulk scan"
}
```

##### POST /scan-bulk/:keys/:block_range/[:labels]

Request:
```json
{
  "scan_pk": "02abc123...",  
  "scan_sk": "02abc123...",
  "spend_pk": "03def456...",
  "start_height": 800000,
  "end_height": 800102,
  "labels_index" : [1, 3, 7] # optional label index
}
```
Response:
```json
{
  "duration": 100000,
  "token": "02abc123..."
}
```

##### POST /scan-check/:token

Request:
```json
{
  "token": "02abc123..."
}
```
Response (complete):
```json
{
  "status": "complete",
  "utxos": [
    {
      "txid": "a1b2c3...",
      "output_index": 0,
      "value": 100000,
      "output_pubkey": "03def456...", #script pub key
      "height": 850000,
      "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
      "spent": false
    },
    {
      "txid": "a1b2c3...",
      "output_index": 3,
      "value": 100000,
      "output_pubkey": "03def456...", #script pub key
      "height": 850001,
      "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
      "spent": false
    }
  ]
}
```
Response (busy):
```json
{
  "status": "busy",
  "duration": 10000,
  "token": "02abc123..."
}
```

##### POST /scan-cancel/:token

Request:
```json
{
  "token": "02abc123..."
}
```
Response:
```json
{
  "status": "complete"
}
```

##### STREAM /scan-mempool/:keys/[:labels]
Request:
```json
{
  "scan_pk": "02abc123...",
  "scan_sk": "02abc123...",
  "spend_pk": "03def456...",
  "labels_index" : [2, 3] # optional label index
}
```
Response (on update):
```json
{
  "status": "active",
  "last_check": <timestamp>,
  "next_check": <timestamp>,
  "utxos": [
    {
      "txid": "a1b2c3...",
      "output_index": 0,
      "value": 100000,
      "output_pubkey": "03def456...", #script pub key
      "height": 850000,
      "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
      "spent": false
    }
  ]
}
```
Response (failure):
```json
{
  "status": "error",
  "last_check": <timestamp>,
  "message": "error string"
}
```

#### Scanner Service Endpoints

##### POST /register

Request:
```json
{
  "scan_pk": "02abc123...",  
  "scan_sk": "02abc123...",
  "spend_pk": "03def456...",
  "birthday": 850000,
  "label": "my_wallet"
}
```
Response:
```json
{
  "wallet_id": "uuid-string",
  "registered": true,
  "scan_from": 850000
}
```

##### GET /wallet/utxos?wallet_id=:id&restore=:bool

```json
 {
  "wallet_id": "uuid-string",
  "last_scan_height": 850100,
  "utxos": [
    {
      "txid": "a1b2c3...",
      "output_index": 0,
      "value": 100000,
      "output_pubkey": "03def456...", #script pub key
      "height": 850050,
      "hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
      "spent": false
    }
  ]
}
```

#### Error Response Format

```json
{
  "error": {
    "code": 400,
    "message": "Invalid block height",
    "details": "Block height must be between 0 and current tip"
  }
}
```

## Diagrams

### Wallet Integration Patterns

##### [Tweak Server Integration](https://github.com/silent-payments/BIP0352-index-server-specification/blob/main/diagrams/dana-blindbit.mmd)

```
(per block)
1. Fetch tweaks `/tweaks-index/:height`
2. Compute the possible scriptPubKeys for n = 0 and each label n+1
3. Fetch filter `/filter/utxos/:height` (BIP 158)
4. Match scriptPubKeys against filter
    - If no match: go to 1. with height + 1
    - Else: continue with 5.
5. Fetch block, verify owned UTXOs and add to wallet
    - Go to 1. with height + 1
```

##### [Remote Scanner Integration](https://github.com/silent-payments/BIP0352-index-server-specification/blob/main/diagrams/remote_scanner_sequence.mmd)

```
1. Wallet requests a block or range of blocks - provide sp address and scan private key `/scan/:keys/:range`
    1.1. Service will respond with 1 of the following:
        - utxos found for each block
        - "busy - requests too large" - upon receiving this response wallet can try another service or request bulk scan
    1.2. Bulk scan - wallet provide sp address and scan private key `/scan-bulk/:keys/:range`
        - Service responds with estimated scan duration and bulk scan token
        - Wallet records token and sets timer for duration 
            - on timeout wallet checks `/scan-check/:token`
2. Wallet subscribes to unconfirmed scanning `/scan-mempool/:keys`
    2.1. Service will scan for utxos every # seconds
        - For each UTXO found matching a key; stream message to wallet
3. Wallet maintain local UTXO state for spending
```

##### [My Scanner Integration](https://github.com/silent-payments/BIP0352-index-server-specification/blob/main/diagrams/my_scanner_sequence.mmd)

```
1. Register wallet sp address and scan private key with /register endpoint
2. Periodically query `/wallet/utxos` for updates
3. Handle notifications of unconfirmed transactions if supported
4. Maintain local UTXO state for spending
```

## Testing

Reference implementations or specific test vectors for section 2 and 3. Section 1 is fully defined in BIP352 Spec

1. Verify Tweak Derivation - [BIP352 Test Vectors](https://github.com/bitcoin/bips/blob/master/bip-0352/send_and_receive_test_vectors.json)
2. Block Processing
   1. Block Vectors
      1. expected tweaks for a given block
   2. Pruning
      1. Spent Tx Filtering (cut-through)
      2. Dust

### Reference Test Vectors

#### Block Processing Test Vector

- [Embedded Service Comparison [Block Test Vectors]](https://github.com/silent-payments/tweak-service-auditor/issues/1)

#### Interoperability Tests

- Service switching without data loss
- Reorg handling consistency
- Rate limiting behavior verification

## Security and Authentication

### Authentication Models

###### Anonymous Services (Tweak Servers)

- No authentication required for public endpoints
- Rate limiting applied per IP address
- No user registration or key storage

###### Scanner Services

- Token or wallet ID based authentication
- Key storage should be ephemeral for the life of a session
  - Use token for fetching scan results

### Security Considerations

##### Rate Limiting Standards

- Anonymous services: Maximum 10 blocks / minute
- Scanner services: Depends on load and service offering

##### Privacy Protection

- Services should not log request patterns that could deanonymize users
- Tweak servers should implement request batching to reduce timing correlation

##### Data Integrity

- Block hash verification required for reorg detection
- Tweak data should be verifiable against full node data
- Checksums recommended for bulk data transfers (How does a wallet know all tweaks were received for a given block request?)


### Other Considerations

- Pagination for large bulk results