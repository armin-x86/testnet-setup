# Custom Substrate-based RelayChain and Parachain Setup

This README outlines the steps to spin up your **custom Substrate-based RelayChain and a Parachain**. It is important to be familiar with the following concepts: **Accounts, Session Keys, Node Keys, Validator, Collator, PoV, PoS**, and other main Substrate concepts. The following documentation can help you get started:
- [Polkadot SDK Docs](https://docs.substrate.io/polkadot-sdk/)
- [Substrate Parachains Guide](https://paritytech.github.io/polkadot-sdk/book/whence-parachains.html)
- [Parachain Deployment Guide](https://paritytech.github.io/devops-guide/guides/parachain_deployment.html)

---

## **RelayChain Setup**

### Step 1: Prepare a Base ChainSpec
1. **Download the latest Polkadot binaries**:
   ```bash
   wget https://github.com/paritytech/polkadot-sdk/releases/download/polkadot-stable2412/polkadot
   wget https://github.com/paritytech/polkadot-sdk/releases/download/polkadot-stable2412/polkadot-execute-worker
   wget https://github.com/paritytech/polkadot-sdk/releases/download/polkadot-stable2412/polkadot-prepare-worker
   ```
   Place `polkadot-execute-worker` and `polkadot-prepare-worker` in the same directory as the `polkadot` binary.
2. **Generate the base ChainSpec**:
   ```bash
   polkadot build-spec --chain rococo-local > base.json
   ```
   Modify `base.json` to include:
   - Public IP address and the public part of the boot node's `NodeKey` in the `bootNodes` list.
3. **Generate NodeKey**:
   ```bash
   polkadot key generate-node-key
   # Or use subkey:
   subkey generate-node-key
   ```
4. **Generate Substrate Accounts**:
   ```bash
   subkey generate -n substrate
   Or:
   polkadot key generate -n substrate
   ```
   Save the **mnemonic phrase**. Use the **Public key (SS58)** in the ChainSpec for the `balances` and `session.keys`.
5. **Generate Session Keys**:
- Using RPC:
  ```
  curl -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "method": "author_rotateKeys", "params":[]}' http://localhost:9944
  ```
- Using CLI:
  ```
  polkadot key insert --keystore-path /home/ubuntu/keystore --key-type gran --scheme ed25519 --suri "mnemonic//DerivationPhrase"
  ```

6. **Update the ChainSpec**:
- Replace default accounts (`--alice`, `--bob`) in:
  - `session.keys`
  - `balances`
  - `sudo.key`
- Convert the modified ChainSpec to raw format:
  ```
  polkadot build-spec --chain base.json --raw > chainspec.json
  ```

7. **Node Key File**:
- Convert private part of NodeKey to binary:
  ```
  echo "private part of node-key" | xxd -r -p > nodeKey
  ```

---

### Step 2: Systemd Setup for Validators
1. **Systemd Unit**:
Configure a systemd service for each validator. One validator can act as a boot node with:
    ```
    --state-pruning=archive
    ```
2. Ensure the **P2P port** is open.

---

## **Parachain Setup**

### Step 1: Prepare Parachain ChainSpec
1. **Generate Base ChainSpec**:
    ```bash
    astar-collator build-spec --chain astar-dev > parachain-base.json
    ```

2. **Update Parachain ChainSpec**:
- Replace `--alice` and `--bob` with generated collator accounts.
- Add collator accounts to:
  - `balances`
  - `collatorSelection.invulnerables`
  - `session.keys`
  - `sudo.key`
- Convert to raw format:
  ```
  astar-collator build-spec --chain parachain-base.json --raw > parachain-chainspec
  ```

---

### Step 2: Register Parachain
1. **Export Genesis Files**:
    ```
    astar-collator export-genesis-state --chain parachain-chainspec > genesis-state
    astar-collator export-genesis-wasm --chain parachain-chainspec > genesis-wasm
    ```

2. **Polkadot.js Setup**:
Use `parasSudoWrapper.sudoScheduleParaInitialize`:
- `id`: Parachain ID (from ChainSpec)
- `genesisHead`: `genesis-state`
- `validationCode`: `genesis-wasm`
- `paraKind`: `true`

---

## **Archive Nodes and HAProxy**

### Archive Node Setup
1. Deploy two archive nodes and expose the RPC port (9944) only to the HAProxy server using firewall rules (e.g., UFW).
2. Share the ChainSpec files (`parachain-chainspec` and `relaychain-chainspec`) with archive nodes.

### HAProxy Load Balancer
1. **Setup HAProxy**:
- Configure it to load balance between the two archive nodes.
- Use SSL termination for HTTPS access.
- Example HAProxy config:
  ```haproxy
  frontend https_frontend
      bind *:443 ssl crt /etc/haproxy/certs/rpc.domain.com.pem
      default_backend astar_archive_backends

  backend astar_archive_backends
      balance roundrobin
      server archive_node_1 192.168.251.30:9944 check
      server archive_node_2 192.168.251.31:9944 check
  ```

2. **SSL Certificate**:
Use `certbot` to generate the SSL certificate:

---

Now the testnet is fully operational with:
- A custom relay chain
- A parachain registered to the relay chain
- Redundant archive nodes behind HAProxy


