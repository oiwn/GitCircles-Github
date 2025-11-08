# Task 05 – Blockchain Stub

This task captures the contract for the blockchain handoff layer that upstream notification/process automation code will call. Implementation details (node configuration, encryption, etc.) stay out of scope, but every consumer must rely on the same interface and signals.

## Integration Boundary
- Add a `src/blockchain.rs` module (re-exported via `lib.rs`) that exposes the async entry point:
  ```rust
  pub struct TokenTransferRequest {
      pub project_id: String,
      pub pr_number: u64,
      pub repo: String,
      pub recipient_wallet: String,
      pub amount_tokens: u64,
      pub memo: Option<String>, // e.g. tx note or explorer hint
  }

  pub struct TokenTransferReceipt {
      pub tx_hash: String,
      pub explorer_url: String,
      pub confirmed: bool,
  }

  pub async fn send_tokens(request: TokenTransferRequest) -> Result<TokenTransferReceipt, BlockchainError>;
  ```
- The stub does not actually broadcast transactions yet; for Task 05 it should log intents and return deterministic fake receipts so higher layers (notification + audit) can be exercised.
- All callers (wallet workflow, automation worker, tests) must treat the module as the only way to initiate transfers, even when mocked.

## Secure Key Storage & Connectivity
- The stub should document its dependencies without implementing them:
  - `KeyVault`: abstraction for loading the Ergo testnet private key; actual storage can be HSM, OS keychain, or encrypted file, but the interface must allow injecting a mock for local development.
  - `ErgoNodeClient`: trait describing the JSON-RPC endpoint (URL, auth header) so environments can point at different nodes.
- The module must validate that both dependencies are configured before attempting a transfer, returning a configuration error otherwise. No secret material should be logged; use opaque identifiers when emitting traces.

## Success/Failure Signals
- `Ok(TokenTransferReceipt)` indicates the signed transaction was accepted by the network (or stubbed equivalent). `confirmed` should be `false` for immediate responses and flipped to `true` once the stub observes a confirmation poll—callers can decide whether to wait.
- Errors should map to high-level variants so upstream automation can react:
  - `BlockchainError::Connectivity` – node unreachable, TLS issues, timeouts.
  - `BlockchainError::Signing` – unable to access key vault or sign payload.
  - `BlockchainError::Rejected` – node explicitly rejected the tx (bad wallet, insufficient balance).
  - `BlockchainError::Internal` – catch-all for unexpected failures; include a correlation id for observability.
- Upstream tasks should treat `Connectivity` as retryable, `Signing` as operator action required, and `Rejected` as terminal for the current PR payout attempt. When the stub is still fake, it should surface these variants through feature flags so alerting and notification paths get exercised early.

## Guidance From `ergoplatform/oracle-core`
Use the `oracle-core` repository as a concrete reference for how Ergo integrations are structured:

- **Node gateway pattern** (`core/src/node_interface/node_api.rs:18-330`): The project wraps `ergo_node_interface` inside a `NodeApiTrait` + `NodeApi` struct that exposes high-level methods (`get_unspent_boxes_by_address`, `get_state_context`, `sign_and_submit_transaction`). This isolates HTTP calls and signing logic, while callers receive a trait they can mock. Our `ErgoNodeClient` abstraction should follow the same model so the blockchain stub can swap between the real node and test doubles.
- Relevant snippet:
  ```rust
  pub trait NodeApiTrait {
      fn sign_and_submit_transaction(
          &self,
          transaction_context: TransactionContext<UnsignedTransaction>,
      ) -> Result<TxId, NodeApiError>;
  }

  pub struct NodeApi {
      pub node: NodeInterface,
  }

  impl NodeApi {
      pub fn new(node_url: &Url) -> Self {
          let node = NodeInterface::from_url("", node_url.clone());
          Self { node }
      }

      pub fn sign_and_submit_transaction(
          &self,
          transaction_context: TransactionContext<UnsignedTransaction>,
      ) -> Result<TxId, NodeApiError> {
          let tx = self.sign_transaction(transaction_context)?;
          self.submit_transaction(&tx)
      }
  }
  ```

- **Secret + network configuration** (`core/src/oracle_config.rs:30-200`): Secrets come from env vars (`ORACLE_WALLET_SECRET`, `ORACLE_WALLET_MNEMONIC`) or YAML, then the code derives the NetworkAddress and change address at startup. Replicate this pattern by resolving keys once (preferably via the `KeyVault` abstraction) and caching derived addresses, ensuring nothing sensitive gets logged.
- Relevant snippet:
  ```rust
  fn set_mnemonic_secret(&mut self) -> Result<(), OracleConfigFileError> {
      if let Ok(secret) = std::env::var("ORACLE_WALLET_SECRET") {
          let secret_bytes = base16::decode(&secret)?;
          if let Ok(secret_key) = SecretKey::from_bytes(&secret_bytes) {
              self.oracle_secret_key = Some(secret_key);
              return Ok(());
          }
      }

      if let Some(secret) = &self.oracle_secret {
          let secret_bytes = base16::decode(secret)?;
          if let Ok(secret_key) = SecretKey::from_bytes(&secret_bytes) {
              self.oracle_secret_key = Some(secret_key);
              return Ok(());
          }
      }

      if let Ok(mnemonic) = std::env::var("ORACLE_WALLET_MNEMONIC") {
          if let Ok(secret_key) = self.derive_secret_from_mnemonic(&mnemonic) {
              self.oracle_secret_key = Some(secret_key);
              return Ok(());
          }
      }

      if let Some(mnemonic) = &self.oracle_mnemonic {
          if let Ok(secret_key) = self.derive_secret_from_mnemonic(mnemonic) {
              self.oracle_secret_key = Some(secret_key);
              return Ok(());
          }
      }

      Err(OracleConfigFileError::MissingMnemonicSecret)
  }
  ```

- **Transaction construction flow** (`core/src/cli_commands/transfer_oracle_token.rs:1-185`): They gather unspent boxes, run `SimpleBoxSelector`, build an unsigned transaction with `TxBuilder`, embed context extensions, and finally wrap it in `TransactionContext` before calling `sign_and_submit_transaction`. Our payout flow should be conceptually identical: collect funding boxes, build `TransactionContext`, then delegate to `send_tokens`.
- Relevant snippet:
  ```rust
  let unspent_boxes = node_api.get_unspent_boxes_by_address(
      &oracle_address.to_base58(),
      target_balance,
      [].into(),
  )?;
  let box_selector = SimpleBoxSelector::new();
  let selection = box_selector.select(unspent_boxes, target_balance, &[])?;
  let mut input_boxes = vec![in_oracle_box.get_box().clone()];
  input_boxes.append(selection.boxes.as_vec().clone().as_mut());
  let box_selection = BoxSelection {
      boxes: input_boxes.clone().try_into().unwrap(),
      change_boxes: selection.change_boxes,
  };
  let mut tx_builder = TxBuilder::new(
      box_selection,
      vec![oracle_box_candidate],
      height.0,
      target_balance,
      change_address,
  );
  let tx = tx_builder.build()?;
  let context = TransactionContext::new(tx, input_boxes, vec![])?;
  let tx_id = node_api.sign_and_submit_transaction(context)?;
  ```

- **Explorer linkage & confirmation polling** (`core/src/explorer_api.rs:1-129`): After broadcasting, they generate an explorer URL and optionally poll until the tx is indexed. `TokenTransferReceipt.explorer_url` should come from the same logic (derive base URL from config, append `/en/transactions/{tx}`), and `confirmed` can be driven by an explorer poll similar to `wait_for_tx_confirmation`.
- Relevant snippet:
  ```rust
  pub(crate) fn ergo_explorer_transaction_link(tx_id: TxId, prefix: NetworkPrefix) -> String {
      let url = ORACLE_CONFIG
          .explorer_url
          .clone()
          .unwrap_or_else(|| default_explorer_url(prefix));
      let tx_id_str = String::from(tx_id);
      url.join("en/transactions/")
          .unwrap()
          .join(&tx_id_str)
          .unwrap()
          .to_string()
  }

  pub fn wait_for_tx_confirmation(tx_id: TxId) {
      wait_for_txs_confirmation(vec![tx_id]);
  }
  ```

- **Test scaffolding** (`core/src/node_interface/test_utils.rs:1-97`): `MockNodeApi` implements the trait, signs locally, and records submitted transactions. Lean on this approach when building integration tests for our stub—define a mock `ErgoNodeClient` that returns deterministic boxes/tx ids so CLI + notification flows can run without a live node.
- Relevant snippet:
  ```rust
  pub struct MockNodeApi<'a> {
      pub unspent_boxes: Vec<ErgoBox>,
      pub secrets: Vec<SecretKey>,
      pub submitted_txs: &'a RefCell<Vec<Transaction>>,
      pub chain_submit_tx: Option<&'a mut ChainSubmitTx<'a>>,
      pub ctx: ErgoStateContext,
  }

  impl NodeApiTrait for MockNodeApi<'_> {
      fn sign_and_submit_transaction(
          &self,
          transaction_context: TransactionContext<UnsignedTransaction>,
      ) -> Result<TxId, NodeApiError> {
          self.sign_transaction(transaction_context)
              .and_then(|tx| self.submit_transaction(&tx))
      }
  }
  ```

## Guidance From `sigma-rust`
We cannot depend directly on `oracle-core`, but the `sigma-rust` workspace already exposes the primitives needed for a first-class Ergo client. Use the following snippets as references while implementing the blockchain stub:

- **Wallet + signing (`ergo-lib/src/wallet.rs:73-121`)**:
  ```rust
  use ergo_lib::wallet::{Wallet, signing::TransactionContext};

  // Load operator wallet from mnemonic (or inject secrets via KeyVault)
  let wallet = Wallet::from_mnemonic(mnemonic_phrase, "")?;

  // Build the unsigned transaction via TxBuilder (see next snippet)
  let tx_ctx = TransactionContext::new(unsigned_tx, input_boxes, vec![])?;

  // Fetch ErgoStateContext from the node, then sign
  let signed_tx = wallet.sign_transaction(tx_ctx, &state_context, None)?;
  ```
  `Wallet::from_secrets` is available if the KeyVault returns raw `SecretKey` bytes, and deterministic signing helpers exist for constrained environments.

- **Transaction builder (`ergo-lib/src/wallet/tx_builder.rs:1-120`)**:
  ```rust
  use ergo_lib::wallet::{box_selector::BoxSelection, tx_builder::TxBuilder};

  let box_selection = BoxSelection { boxes, change_boxes };
  let mut tx_builder = TxBuilder::new(
      box_selection,
      vec![payout_candidate],
      current_height,
      fee_amount,
      change_address,
  );
  tx_builder.set_context_extension(input_box_id, ctx_ext); // optional
  let unsigned_tx = tx_builder.build()?; // adds change + miner fee boxes
  ```
  Pair this with `SimpleBoxSelector` to gather UTXOs fetched from the node via REST.

- **REST node client (`ergo-rest/src/api.rs`, `ergo-rest/src/api/node.rs:24-60`)**:
  ```rust
  use ergo_lib::ergo_rest::{api::node, NodeConf};

  pub async fn get_info(node_conf: NodeConf) -> Result<NodeInfo, NodeError> {
      let url = node_conf.addr.as_http_url().join("info")?;
      let client = build_client(&node_conf)?;
      Ok(set_req_headers(client.get(url), node_conf)
          .send()
          .await?
          .json::<NodeInfo>()
          .await?)
  }
  ```
  The same pattern (build reqwest client, attach API key header, serialize JSON) should be reused to add missing endpoints such as `POST /transactions` and `/wallet/boxes/unspent/byAddress`. `NodeConf` already carries the peer address, API key, and timeout values.

- **Typed responses** – REST helpers deserialize straight into `ergo_chain_types::Header`, `ergo_merkle_tree::MerkleProof`, `ergo_nipopow::NipopowProof`, and so on (see imports at the top of `ergo-rest/src/api/node.rs`). That keeps our persistence/audit layers aligned with upstream Ergo types without custom parsing. `NodeConf` itself is just:
  ```rust
  #[derive(PartialEq, Eq, Debug, Clone, Copy)]
  pub struct NodeConf {
      pub addr: PeerAddr,
      pub api_key: Option<&'static str>,
      pub timeout: Option<Duration>,
  }
  ```

- **Cross-platform validation** – the WASM and iOS bindings under `bindings/` construct the same `NodeConf` objects and call these REST APIs, demonstrating that the approach works outside Rust and giving us additional reference implementations if we later surface blockchain interactions beyond the CLI. Example (`bindings/ergo-lib-wasm/src/rest/api.rs`):
  ```rust
  #[wasm_bindgen]
  pub fn get_info(node: &NodeConf) -> js_sys::Promise {
      future_to_promise(async move {
          let info = node::get_info(node.0).await?;
          Ok(JsValue::from(info))
      })
  }
  ```

Action items: add `ergo-lib = { version = "...", features = ["rest"] }` to `Cargo.toml`, wrap `ergo_rest::api` calls inside our `ErgoNodeClient` trait, and use the `Wallet` + `TxBuilder` APIs for transaction authoring. This keeps GitCircles fully aligned with sigma-rust while satisfying the “no oracle-core dependency” constraint.
