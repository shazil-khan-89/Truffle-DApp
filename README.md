# Property Registry dApp — Truffle & Ganache

**MG3012 Blockchain Technology for Business · Assignment 3**
**Building and Deploying Smart Contracts using Truffle & Ganache**

| | |
|---|---|
| **Name** | Muhammad Shazil Khan |
| **Roll No** | 24i-5557 |
| **Class** | FT-05A (BSFT, 5th semester, Fall 2026) |
| **Instructor** | Dr. Usama Arshad |

---

## 1. Overview

This project takes the **Property Registry** smart contract from Assignment 2 (deployed with Remix on Sepolia) and moves it into a proper local development workflow:

- **Truffle** compiles, deploys (migrates) and tests the contracts
- **Ganache** runs a local Ethereum blockchain with 10 funded test accounts
- **MetaMask + ethers.js** connect a plain HTML/CSS/JS frontend to the deployed contracts

The contract records property registration, ownership transfers with zone-based fees, and municipal notices (fines) issued by a registrar. A "no-dues" rule blocks a transfer while a property has 3 or more unpaid notices.

> Student prototype only. It is not connected to CDA, RDA, LDA, PLRA or any real land record, and all amounts are Ganache test ETH.

### Contracts

| File | Contract | Purpose |
|---|---|---|
| `contracts/SmartContract.sol` | `PropertyRegistry` | Main contract from Assignment 2 (logic unchanged, only the network comment updated) |
| `contracts/PropertyInspection.sol` | `PropertyInspection` | **Bonus** second contract. Approved inspectors record site inspections. It calls `PropertyRegistry.getProperty()` on-chain to confirm the property exists, and uses the registry's `registrar()` for access control |

### PropertyRegistry functions

| Function | Type | Who can call | What it does |
|---|---|---|---|
| `registerProperty(id, type)` | write | anyone | Registers a property ID (3–20 chars) to the caller. Types: 0 Plot, 1 House, 2 Commercial |
| `transferProperty(id, newOwner, zone)` | write + payable | current owner | Transfers ownership. Fee = zone base fee × type multiplier, forwarded to the registrar |
| `issueNotice(id, noticeType)` | write | registrar only | Issues a notice: 0 Unpaid tax, 1 Illegal construction, 2 Encroachment |
| `payNotice(noticeId)` | write + payable | property owner | Pays the exact fine for a notice |
| `getProperty(id)` | view (free) | anyone | Returns owner, type, registration time, transfers, fees paid and unpaid notices |

**Fee schedule (demo values in test ETH)**

| Zone | Base fee | Plot ×1 | House ×2 | Commercial ×3 |
|---|---|---|---|---|
| Islamabad | 0.0010 | 0.0010 | 0.0020 | 0.0030 |
| Rawalpindi | 0.0006 | 0.0006 | 0.0012 | 0.0018 |
| Lahore | 0.0008 | 0.0008 | 0.0016 | 0.0024 |

Notice fines: unpaid property tax 0.001, illegal construction 0.002, encroachment 0.0005 ETH.

---

## 2. Folder structure

```
PropertyRegistry-Truffle/
├── contracts/
│   ├── SmartContract.sol          # PropertyRegistry (main contract from Assignment 2)
│   └── PropertyInspection.sol     # Bonus: second contract that calls the registry
├── migrations/
│   ├── 1_deploy_contract.js       # deploys PropertyRegistry
│   └── 2_deploy_inspection.js     # deploys PropertyInspection with the registry address
├── test/
│   └── propertyRegistry.test.js   # 13 automated Truffle/Mocha tests
├── frontend/
│   ├── index.html
│   ├── script.js                  # ethers.js v6 + MetaMask logic
│   ├── style.css
│   └── lib/ethers.umd.min.js      # ethers.js bundled locally (works offline)
├── screenshots/                   # compile, migrate, test and UI interaction
├── server.js                      # tiny static server (serves frontend + build artifacts)
├── truffle-config.js
├── package.json
└── README.md
```

---

## 3. Prerequisites

| Tool | Version used | Install |
|---|---|---|
| Node.js | v20 LTS | https://nodejs.org |
| Truffle | v5.11.5 | `npm install -g truffle` |
| Ganache | CLI v7.9 (the GUI works too) | CLI: `npm install -g ganache` · GUI: https://archive.trufflesuite.com/ganache/ |
| MetaMask | latest browser extension | https://metamask.io |

Check the installation:

```bash
node -v
truffle version
```

(See `screenshots/01_truffle_version.png`.)

---

## 4. Setup and deployment — step by step

### Step 1: Initialise the Truffle project

```bash
mkdir PropertyRegistry-Truffle
cd PropertyRegistry-Truffle
truffle init
```

This creates `contracts/`, `migrations/`, `test/` and `truffle-config.js` (`screenshots/02_truffle_init.png`). The Assignment 2 contract was then copied into `contracts/SmartContract.sol`.

### Step 2: Start Ganache

**Option A — Ganache GUI:** open Ganache → **Quickstart (Ethereum)**. It runs on `http://127.0.0.1:7545` with network id 5777.

**Option B — Ganache CLI** (same port and ids as the GUI):

```bash
npm run ganache
```

This runs `ganache --server.port 7545 --chain.chainId 1337 --chain.networkId 5777 --wallet.deterministic`, which always produces the same 10 accounts with 100 ETH each (`screenshots/03_ganache_running.png`). Keep this terminal open.

### Step 3: Configure Truffle for Ganache

`truffle-config.js`:

```js
networks: {
  development: {
    host: "127.0.0.1",
    port: 7545,          // Ganache
    network_id: "*",     // any network id (Ganache GUI uses 5777)
  },
},
compilers: {
  solc: {
    version: "0.8.20",
    settings: {
      optimizer: { enabled: true, runs: 200 },
      evmVersion: "paris",   // Ganache does not support the PUSH0 opcode from "shanghai"
    },
  },
},
```

`evmVersion: "paris"` matters: solc 0.8.20 targets Shanghai by default and emits `PUSH0`, which Ganache GUI rejects with an *invalid opcode* error during migration.

### Step 4: Compile

```bash
truffle compile
```

ABIs and bytecode are written to `build/contracts/` (`screenshots/04_truffle_compile.png`).

### Step 5: Migrate (deploy)

```bash
truffle migrate          # first deployment
truffle migrate --reset  # redeploy from scratch
```

The migrations run in order:

1. `1_deploy_contract.js` deploys `PropertyRegistry`. Ganache account 0 becomes the **registrar**.
2. `2_deploy_inspection.js` deploys `PropertyInspection`, passing `PropertyRegistry.address` into its constructor.

Truffle saves each deployed address into `build/contracts/*.json` under the network id (`screenshots/05_truffle_migrate.png`).

### Step 6: Run the frontend

```bash
npm run frontend
```

Open **http://localhost:3000** (`screenshots/07_frontend_server.png`).

`server.js` serves the `frontend/` folder and exposes `build/contracts/` at `/contracts/`. `script.js` fetches `PropertyRegistry.json` and `PropertyInspection.json` from there and reads the ABI and the address for the current network id, so the page always uses whatever the last `truffle migrate` deployed. This is the same idea as `lite-server` in the Truffle Pet Shop tutorial, without needing an extra package. (VS Code Live Server also works if you open the project root and go to `/frontend/index.html`.)

### Step 7: Connect MetaMask to Ganache

1. MetaMask → **Add network manually**
   - Network name: `Ganache Local`
   - RPC URL: `http://127.0.0.1:7545`
   - Chain ID: `1337`
   - Currency symbol: `ETH`
2. **Import account** using private keys from Ganache (key icon in the GUI, or the "Private Keys" list printed by `npm run ganache`). Import at least:
   - Account 0 — the registrar (deployer)
   - Account 1 — a property owner
   - Account 2 — a buyer
   - Account 3 — an inspector (bonus contract)
3. Click **Connect MetaMask** in the dApp. If MetaMask is on another network, the page asks it to switch to (or add) Ganache.

---

## 5. Using the dApp

| Section | Contract call | Notes |
|---|---|---|
| 1 · Register a property | `registerProperty` | ID is upper-cased by the frontend (e.g. `isb-f7-101` → `ISB-F7-101`) |
| 2 · Transfer ownership | `transferProperty` (payable) | Live fee quote read from `zoneBaseFee` × `typeMultiplier`; warns if you are not the owner or the property is blocked |
| 3 · Pay a notice fine | `payNotice` (payable) | Loads the notice and its exact fine before paying |
| 4 · Issue a notice | `issueNotice` | Only enabled when the connected wallet is the registrar |
| 5 · Look up a property | `getProperty` (view) | Shows owner, transfers, fees, transfer status, all notices (with Pay buttons) and inspections |
| Fee schedule | public arrays (view) | Read from the contract, not hard-coded |
| Site inspections (bonus) | `setInspector`, `recordInspection`, `getInspectionIds` | Second contract |
| On-chain activity | contract events | `PropertyRegistered`, `PropertyTransferred`, `NoticeIssued`, `NoticePaid`, `InspectorUpdated`, `InspectionRecorded` with block, time and tx hash |

The header shows the network, connected account, role (Registrar / Inspector / Citizen), balances and the latest block. Every transaction shows a toast with its hash, block number and gas used. Failed transactions show the contract's `require()` message, e.g. *"Too many unpaid notices: clear them first"*. The page refreshes itself on every new block.

---

## 6. Testing

### 6.1 Automated tests (Truffle + Mocha/Chai)

With Ganache running:

```bash
truffle test
```

**13 tests pass** (`screenshots/06_truffle_test.png`):

| Area | What is tested |
|---|---|
| Deployment | deployer is registrar; fee schedule loaded correctly |
| `registerProperty` | registers to caller and emits event; rejects duplicate IDs, IDs shorter than 3 / longer than 20 chars, invalid type; `getProperty` reverts for unknown IDs |
| `transferProperty` | charges zone × type fee, **registrar's balance increases by exactly the fee**, owner/transfer count/fees/record updated; rejects wrong fee, non-owner, self/zero address, invalid zone |
| Notices | only registrar can issue; only the owner can pay, only the exact amount, only once; non-existent notice rejected |
| No-dues rule | 3 unpaid notices block a transfer; paying one unblocks it |
| `PropertyInspection` (bonus) | linked to registry; only registrar manages inspectors; inspection stores the owner read from the registry; unapproved inspector rejected; **revert from the registry bubbles up** for unregistered IDs |

Each test deploys fresh contracts with `PropertyRegistry.new()`, so tests are independent. `truffle test` writes to a temporary build folder, so it does not change the addresses the frontend uses.

### 6.2 Manual test run (the one shown in the screenshots)

| # | Account | Action | Expected result | Screenshot |
|---|---|---|---|---|
| 1 | — | Open the dApp | Fee schedule, registrar, counts load without a wallet | `ui_01` |
| 2 | Acc 1 | Connect MetaMask | Role "Citizen / owner", balance shown | `ui_02` |
| 3 | Acc 1 | Register `LHR-GUL-7` (Commercial), `RWP-SAT-9` (Plot), `ISB-F7-101` (House) | 3 `PropertyRegistered` events | `ui_03` |
| 4 | Acc 1 | Transfer `ISB-F7-101` to Acc 2, Islamabad | Quote 0.002 ETH (0.001 × 2) | `ui_04` |
| 5 | Acc 1 | Submit the transfer | Owner = Acc 2, transfers 1, fees 0.002 ETH | `ui_05` |
| 6 | Acc 0 | Issue 3 notices on `RWP-SAT-9` | "Transfer blocked · 3 unpaid notices" | `ui_06` |
| 7 | Acc 1 | Try to transfer `RWP-SAT-9` | Reverted: *Too many unpaid notices: clear them first* | `ui_07` |
| 8 | Acc 1 | Pay notice #1 (0.001 ETH) | 2 unpaid, "Clear to transfer" | `ui_08` |
| 9 | Acc 0 | Approve Acc 3 as inspector | `InspectorUpdated` event | `ui_09` |
| 10 | Acc 3 | Record inspection on `LHR-GUL-7` | Inspection #1 listed under the property | `ui_10` |
| 11 | Acc 3 | Record inspection on `FAKE-123` | Reverted: *Property not registered* (from the registry contract) | `ui_11` |
| 12 | — | Activity log | All 11 dApp transactions listed with block, time, tx hash | `ui_12` |

`08_ganache_dapp_transactions.png` shows the same transactions arriving in Ganache (tx hash, gas used, block number, block time).

---

## 7. Screenshots

### Compilation, deployment and tests

| | |
|---|---|
| `truffle version` | ![](screenshots/01_truffle_version.png) |
| `truffle init` | ![](screenshots/02_truffle_init.png) |
| Ganache running on 7545 | ![](screenshots/03_ganache_running.png) |
| `truffle compile` | ![](screenshots/04_truffle_compile.png) |
| `truffle migrate` | ![](screenshots/05_truffle_migrate.png) |
| `truffle test` — 13 passing | ![](screenshots/06_truffle_test.png) |
| Frontend server | ![](screenshots/07_frontend_server.png) |
| Ganache processing dApp transactions | ![](screenshots/08_ganache_dapp_transactions.png) |

### dApp interaction

**Loaded before connecting (read-only calls)**
![](screenshots/ui_01_dapp_loaded_before_connect.png)

**Wallet connected**
![](screenshots/ui_02_wallet_connected.png)

**Property registered**
![](screenshots/ui_03_property_registered.png)

**Transfer fee quote (read from contract)**
![](screenshots/ui_04_transfer_fee_quote.png)

**Transfer confirmed — fee paid to registrar**
![](screenshots/ui_05_transfer_confirmed.png)

**Registrar issues notices — transfer blocked**
![](screenshots/ui_06_registrar_issued_notices.png)

**No-dues rule enforced by the contract**
![](screenshots/ui_07_transfer_blocked_unpaid_notices.png)

**Notice paid — transfer unblocked**
![](screenshots/ui_08_notice_paid_transfer_unblocked.png)

**Bonus: registrar approves an inspector**
![](screenshots/ui_09_registrar_approves_inspector.png)

**Bonus: inspection recorded through the second contract**
![](screenshots/ui_10_inspection_recorded_bonus_contract.png)

**Bonus: cross-contract check rejects an unregistered property**
![](screenshots/ui_11_inspection_rejected_unregistered.png)

**On-chain activity log (events)**
![](screenshots/ui_12_onchain_activity_log.png)

**Full page**
![](screenshots/ui_13_full_page.png)

---

## 8. Bonus work

1. **Multiple contracts:** `PropertyInspection.sol` is deployed by its own migration, receives the registry address in its constructor and makes on-chain calls into `PropertyRegistry` (`getProperty`, `registrar`). Revert reasons from the registry bubble up to the user.
2. **Automated test suite:** 13 Truffle tests covering every function, access rule, fee calculation and the no-dues rule.
3. **Frontend extras:** role detection, live fee quotes, notice preview, one-click Pay buttons on a property's notices, readable revert messages, event-based activity history, auto-refresh on new blocks, automatic MetaMask network switch/add, read-only mode without MetaMask, and ethers.js bundled locally so nothing depends on a CDN.

---

## 9. Commands summary

| Command | Purpose |
|---|---|
| `npm install -g truffle ganache` | Install Truffle and Ganache CLI |
| `truffle init` | Create the project skeleton |
| `npm run ganache` | Start Ganache CLI on port 7545 (or open Ganache GUI) |
| `truffle compile` | Compile contracts to `build/contracts` |
| `truffle migrate` / `truffle migrate --reset` | Deploy / redeploy to Ganache |
| `truffle test` | Run the 13 automated tests |
| `truffle console` | Interactive console, e.g. `const r = await PropertyRegistry.deployed()` |
| `npm run frontend` | Serve the dApp at http://localhost:3000 |

---

## 10. Troubleshooting

| Problem | Fix |
|---|---|
| `invalid opcode` during migrate | Keep `evmVersion: "paris"` in `truffle-config.js`, then `truffle migrate --reset` |
| `Could not connect to your Ethereum client` | Ganache is not running or not on port 7545 |
| Banner: "not deployed on network …" | MetaMask is on the wrong network, or `truffle migrate` has not been run against this Ganache instance |
| `nonce too high` / stuck transactions after restarting Ganache | MetaMask → Settings → Advanced → **Clear activity tab data** |
| Notice form disabled | Only the registrar (Ganache account 0) can issue notices |
| "Could not load PropertyRegistry.json" | Run `truffle migrate`, then start the site with `npm run frontend` (not by double-clicking `index.html`) |

---

## References

- Truffle Pet Shop tutorial — https://archive.trufflesuite.com/guides/pet-shop/
- Truffle documentation — https://archive.trufflesuite.com/docs/truffle/
- Ganache documentation — https://archive.trufflesuite.com/docs/ganache/
- ethers.js v6 documentation — https://docs.ethers.org/v6/
