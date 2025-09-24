# Ad Network on ICP: Decentralized Advertising and Monetization Platform with Chain Fusion

The Ad Network is a decentralized advertising and monetization platform built on the Internet Computer Protocol (ICP) blockchain. It enables advertisers to create and distribute ads in various formats (e.g., banners, full-page) directly on-chain, while allowing developers to monetize their web apps, games, or projects by displaying these ads and earning tokens for verified views. Ads are stored in an ICP canister, served via a fair round-robin system, and tracked transparently on the blockchain to prevent fraud and ensure equitable distribution.

This project showcases advanced **chain fusion** capabilities, integrating Solana blockchain interactions seamlessly into an ICP dApp. Using ICP's HTTPS outcalls for querying Solana RPC nodes and threshold Ed25519/Schnorr signing for authorizing Solana transactions, the platform supports dual ecosystems: ICP (with Internet Identity or Plug Wallet) for native ICP payments, and Solana (with Phantom Wallet) for SOL payments. This allows users to authenticate and transact in their preferred chain without leaving the ICP-hosted app, demonstrating ICP's role as a "world computer" that fuses multiple blockchains.

Key technical achievements:
- **Chain-Key Cryptography for Solana Integration**: Leverages ICP's threshold Ed25519 and Schnorr signing to generate Solana-compatible signatures on-chain. Private keys are secret-shared across ICP subnet nodes, enabling secure signing without exposing keys. Derivation paths use Solana pubkeys for unique, per-user keys.
- **SOL RPC Canister Bridge**: Uses a dedicated ICP canister (`tghme-zyaaa-aaaar-qarca-cai`) for fault-tolerant Solana RPC calls via HTTPS outcalls, aggregating responses from multiple providers (e.g., Helius, Alchemy) with consensus strategies (e.g., equality) to avoid single points of failure and handle network variance.
- **Dual Authentication & Payments**: ICP users pay/receive in ICP; Phantom users in SOL. Bridging handled by a Solana-ICP POC canister (`sh73c-3qaaa-aaaao-qke6a-cai`) for deriving addresses, nonces, and transfers. Nonces prevent replays; signatures verified with Ed25519.
- **View Verification Token System**: Prevents fake views with ephemeral tokens; records only after 5 seconds, using on-chain logic for fairness. Tokens generated via Motoko canister.
- **Dynamic Pricing & Payouts**: Advertisers prepay views (0.001 ICP / 0.00015 SOL per view); developers earn ~71% (0.00071 ICP / 0.00011 SOL per view). Payouts via secure backend transfers, with dynamic cycles for cost efficiency.
- **Round-Robin Ad Serving**: Fair distribution based on purchased views, with real-time tracking (active ads, views, balances) via canister queries.
- **Image Handling & Validation**: Frontend resizes/validates images to exact ad specs (e.g., 1090x225 for horizontal banners) before base64 encoding and on-chain submission.
- **Embed Script**: Lightweight JS embed (`ad-network-embed-bundled.js`) for easy ad integration in web projects; fetches/displays ads with click handling.

The full dApp (built with Construct 3) is deployed at [https://jrldj-qaaaa-aaaam-aaqvq-cai.icp0.io/](https://jrldj-qaaaa-aaaam-aaqvq-cai.icp0.io/) and can be accessed through its custom domain [AdNetwork.online](https://AdNetwork.online/). For developers, a simple embed script allows easy integration—see the example at [https://github.com/dickhery/Ad-Network-Embed](https://github.com/dickhery/Ad-Network-Embed).

This project was entered into the National Round of the World Computer Hacker League 2025 Hackathon, highlighting ICP's chain fusion for multi-chain dApps.

## How Chain Fusion Works: Solana Integration on ICP

ICP's unique architecture enables "chain fusion" by allowing canisters to interact with external blockchains like Solana without oracles or bridges. Here's a detailed breakdown:

1. **Authentication Modes**:
   - **ICP (Internet Identity/Plug)**: Uses ICP principals for identity; payments via ICP ledger canister (`ryjl3-tyaaa-aaaaa-aaaba-cai`).
   - **Solana (Phantom Wallet)**: Authenticates via Phantom; derives an ICP principal from Solana pubkey using SHA-224 hashing + Ed25519 type byte (in `lib.rs` of SOL bridge). This enables Solana users to interact with ICP canisters as if they had an ICP identity. Pubkey stored in stable memory mappings for ownership checks.

2. **Solana RPC Interactions**:
   - HTTPS outcalls from ICP canisters query Solana nodes via the SOL RPC canister (`tghme-zyaaa-aaaar-qarca-cai`), which aggregates from providers like Helius/Alchemy/Ankr.
   - Supported methods: `getBalance`, `getSlot` (with rounding for consensus), `getBlock` (for blockhash), `sendTransaction` (with base64 encoding), `getSignatureStatuses` (for confirmations).
   - Consensus: Uses strategies like `Equality` or `Threshold` to reconcile responses; dynamic cycles estimation for cost efficiency (parsed from errors).
   - Example: In `lib.rs`, `sol_get_balance_lamports` encodes args, calls dynamically with cycles, decodes multi-results.

3. **Signing Solana Transactions**:
   - Uses ICP's threshold Schnorr (Ed25519 variant): Keys secret-shared; signatures via `sign_with_schnorr` management canister call.
   - Derivation: Solana pubkey seeds the path for `schnorr_public_key`; dynamic cycles for PK/sign calls to handle variable costs.
   - Serialization: Transactions serialized (header, accounts, blockhash, instructions) in `lib.rs`; signed message includes nonce/service fee.
   - Verification: Ed25519 signatures for Phantom actions; `verify_signature` uses `ed25519_dalek`.
   - Confirmations: `getSignatureStatuses` for polling; nonce bump on success (manual for II/Plug via `bump_nonce_ii`).

4. **Payments & Bridging**:
   - **ICP Path**: Direct transfers via ledger; subaccounts derived from Solana pubkey for bridging.
   - **SOL Path**: Deposits to derived addresses (via `get_sol_deposit_address`); transfers via serialized TXs signed on ICP and submitted via RPC (`sol_send_transaction_b64`).
   - Fees: Service fees (e.g., 0.0001 ICP) transferred first; nonces bumped post-success.
   - Nonce Management: Stable BTreeMap for per-pubkey nonces; read/init/bump in `lib.rs`.

5. **Security**:
   - Ownership: Maps Solana pubkeys to ICP principals; requires linking via signed message.
   - Authorization: Phantom sig or owner check; traps on invalid pubkeys/signatures.
   - Timeouts: Polling for nonce changes confirms TXs after timeouts (in frontend JS).
   - Consensus Issues: Handles ICP replica variance with response transformations (e.g., slot rounding).

This fusion allows seamless toggling: ICP users see ICP balances/pricing; Solana users see SOL—all within one ICP dApp. For example, ad creation in Phantom mode signs a message with nonce, pays SOL via bridged transfer, and calls `_phantom` canister methods.

## Repository Structure

The main repo contains all components for the Ad Network, including the SOL bridge, ICP transfer bank, Ad Network canister, and frontend.

- **Ad_Network_rollup**: Rollup config and sources for bundling the frontend JS (`ic_ad_network_bundle.js`).
- **ICP_SOL_Bridge**: SOL-ICP bridge canister (`lib.rs`); handles signing/transfers.
- **icp_transfer_bank_ad_network**: ICP transfer backend (`lib.rs`); restricted to Ad Network.
- **ICP-Ad-Network-Canister**: Motoko backend (`main.mo`); core ad logic.
- **AdNetworkOnIcp**: Construct 3 frontend (event sheets, layouts); deployable as asset canister.
- **Ad-Network-Embed**: Lightweight embed script (`ad-network-embed-bundled.js`) for ad integration.

For canister methods/details, see sub-READMEs or code (e.g., `main.mo` for ad ops, `lib.rs` for signing/transfers).

## Key Features in Detail

- **Ad Formats**: Banners/full-page; validation/resizing in JS (`handleImageSelection`).
- **Serving**: Round-robin; `getNextAd` returns ad + token.
- **Verification**: 5s timeout; `recordViewWithToken` validates.
- **Payments**: Prepay views; earn 71%. Dual ICP/SOL via bridge.
- **Dashboard**: Stats via queries (e.g., `getTotalActiveAds[_phantom]`).
- **Embed Script**: Fetches/displays ads with click handling; no auto-cycling.
- **Security**: Threshold signing, nonce checks, restricted transfers (e.g., `require_owner` in `lib.rs`).

## How It Works

### Advertisers
1. Auth; create ads (`confirmCreateNewAd` -> `doCreateNewAd` signs/pays).
2. Pay ICP/SOL; top up (`confirmTopUpAdViews` -> `doTopUpAdViews`).

### Developers
1. Register (`registerProject[_phantom]`).
2. Embed script: Fetches ad (`fetchAndShow`); records after 5s.
3. Cash out (`confirmCashOutAllProjectsViews` -> `doCashOutAllProjectsViews`).

### Chain Fusion Flow
- Phantom: Derive principal (`get_pid`); sign messages (`signMessage`); bridge transfers (`transfer_sol` serializes/signs/submits).
- RPC: Aggregate queries (`sol_get_balance_lamports` encodes/dynamic cycles/decodes).
- Sign: `sign_with_schnorr_dynamic` with derivation; verifies Ed25519.

## Getting Started: Full Deployment & Testing

Prerequisites: Install [dfx](https://internetcomputer.org/docs/current/developer-docs/getting-started/install) (ICP SDK), Node.js/npm, Rust/cargo. Fund a cycles wallet for deployments. Ensure you have a cycles wallet with sufficient cycles for canister creation and operations.

### 1. Clone the Repo
```
git clone https://github.com/dickhery/ICP_Ad_Network.git
cd ICP_Ad_Network
```

This single repo contains all directories needed for deployment.

### 2. Install Dependencies for Bundling
Navigate to the root of the `Ad_Network_rollup` directory and run:
```
npm cache clean --force
rm -rf node_modules package-lock.json
npm install --save-dev rollup @rollup/plugin-node-resolve @rollup/plugin-commonjs @rollup/plugin-json @rollup/plugin-replace @rollup/plugin-terser @rollup/plugin-babel @babel/core
```

This installs the necessary dependencies for bundling the frontend JS.

### 3. Deploy the SOL Bridge Canister (`ICP_SOL_Bridge`)
- Navigate to the root of the `ICP_SOL_Bridge` directory.
- Run `dfx deploy --network ic` (this creates the initial `.env` file with canister IDs).
- Source the environment variables for development:
  ```
  set -a
  source ./.env
  set +a
  env | grep -E 'CANISTER_ID|DFX_|SOL_|VITE_|INTERNET_IDENTITY|LEDGER'  # Verify
  ```
- For production, use `.env.production`:
  ```
  set -a
  source ./.env.production
  set +a
  ```
- Redeploy with `dfx deploy --network ic` to ensure the canister is live and configured.
- Note the `sol_icp_poc_backend` canister ID from `canister_ids.json` for later updates.

### 4. Deploy the ICP Transfer Bank (`icp_transfer_bank_ad_network`)
- Navigate to the root of the `icp_transfer_bank_ad_network` directory.
- Run `cargo build` to compile the Rust code.
- Deploy with `dfx deploy --network ic`.
- This canister handles ICP transfers; restricted to Ad Network calls.
- Note the `icp_transfer_backend` canister ID from `canister_ids.json`.

### 5. Deploy the Ad Network Canister (`ICP-Ad-Network-Canister`)
- Navigate to the root of the `ICP-Ad-Network-Canister` directory.
- Run `npm install` to install dependencies.
- Deploy with `dfx deploy --network ic`.
- This is the core backend for ad logic.
- Note the `ad_network_backend` canister ID from `canister_ids.json`.

**Important: Update Admin Principals**
- In `main.mo` (in `ICP-Ad-Network-Canister`), the admin principals are hardcoded. Replace them with principals you control (e.g., your ICP identity):
  ```
  stable var admins : [Principal] = [
    Principal.fromText("your-principal-here"),
    // Add more if needed
  ];
  ```
- Redeploy after changes: `dfx deploy --network ic`.

### 6. Bundle the Frontend JS (`Ad_Network_rollup`)
- Navigate to the root of the `Ad_Network_rollup` directory.
- Run `npx rollup -c` to bundle `ic_ad_network_bundle.js`.
- Copy the generated `ic_ad_network_bundle.js` to the `AdNetworkOnIcp` directory (or ensure it's referenced in the Construct 3 project).

### 7. Deploy the Frontend (`AdNetworkOnIcp`)
- Navigate to the root of the `AdNetworkOnIcp` directory.
- Deploy with `dfx deploy --network ic` (deploys as an asset canister).
- Access the dApp at `https://{frontend_canister_id}.icp0.io/` (or local equivalent).

### 8. Update Canister IDs Across Files (For New Deployments)
If this is a fresh deployment with new canister IDs:
- Open the repo in a text editor.
- Replace `qanay-uyaaa-aaaag-qbbwa-cai` (Ad Network) in all files with the new `ad_network_backend` ID from `ICP-Ad-Network-Canister/canister_ids.json`.
- Replace `j24mu-2qaaa-aaaal-aae5a-cai` (ICP Transfer) with the new `icp_transfer_backend` ID from `icp_transfer_bank_ad_network/canister_ids.json`.
- Replace `sh73c-3qaaa-aaaao-qke6a-cai` (SOL PoC) with the new `sol_icp_poc_backend` ID from `ICP_SOL_Bridge/canister_ids.json`.
- Also update IDs in `.env` and `.env.production` files.
- Redeploy all canisters (`dfx deploy --network ic` in each root) to propagate changes.

### 9. Set Initial Password for Beta Login Screen
- After deployment, the app has a temporary password-protected beta login screen (to be removed at launch).
- Authenticate as an admin (using a principal from the `admins` array in `main.mo`).
- In the app, navigate to the Admin panel (visible only to admins).
- Enter and set a new password via the "Set Password" button (calls `set_password` in `main.mo`).
- Use this password to access the app until the screen is removed.

### 10. Testing
- Local: `dfx start --background`; access `http://localhost:4943?canisterId={frontend_id}`.
- Auth: II/Plug for ICP; Phantom for SOL.
- Advertiser: Create ad (upload/resize images, pay views).
- Developer: Register project, embed script, fetch ad (5s view), cash out.
- Verify: Check balances (ICP ledger/Solana explorer), views in dashboard.
- Edge: Test timeouts (poll nonces), invalid sigs, low balances.

## Future Plans
- Video ads, bidding, plugins, DAO.

## Contributing
Fork, PR (e.g., canisters/API).

## License
MIT. See [LICENSE](LICENSE).