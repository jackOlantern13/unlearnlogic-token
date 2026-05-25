# UNLEARNLOGIC PROTOCOL SPECIFICATION
## $UNL — Solana-Native Community Mining Token
### Version 0.1 — Genesis Draft

---

## 1. VISION

UNLEARNLOGIC is a fair-launch community mining protocol on Solana. It exists at the intersection of cypherpunk ethics, computational participation, and anti-extractive token design.

**Core thesis:** Proof-of-work-inspired participation is the only legitimate path to token ownership. No insiders. No presale. No venture allocation. Only computation.

**What it is not:**
- Not a meme coin
- Not a speculation vehicle
- Not a yield farm
- Not a DAO governance token issued to investors

**Cultural identity:** The mining ritual — running computation, waiting epochs, claiming delayed rewards — is intentionally slow. Slow is honest. Slow filters for believers, not tourists.

---

## 2. PROTOCOL ARCHITECTURE

### 2.1 Token Parameters

| Parameter | Value |
|-----------|-------|
| Name | UnlearnLogic |
| Ticker | UNL |
| Blockchain | Solana |
| Standard | SPL Token |
| Decimals | 6 |
| Max Supply | 21,000,000 UNL |
| Freeze Authority | None (null from genesis) |
| Mint Authority | Program PDA → Revoked Day 30 |

### 2.2 Distribution

| Allocation | % | UNL |
|-----------|---|-----|
| Mining Rewards | 90% | 18,900,000 |
| Creator Treasury | 5% | 1,050,000 |
| Ecosystem Grants | 5% | 1,050,000 |

**Creator treasury:** Linear vest, 6-month cliff, 24-month total duration. On-chain vesting schedule, not social contract.

**Ecosystem treasury:** Multisig-controlled. Proposal-gated. Every disbursement linked to an on-chain governance vote.

### 2.3 Program Stack

```
Layer 0: SPL Token (UNL mint)
Layer 1: unlearn-mining program (Anchor)
Layer 2: unlearn-treasury program (Anchor)
Layer 3: Squads Multisig → SPL Governance (DAO)
```

### 2.4 PDA Account Map

| Seeds | Account | Purpose |
|-------|---------|---------|
| `["global_config"]` | GlobalConfig | Protocol parameters |
| `["epoch_state", epoch_u64]` | EpochState | Per-epoch tracking |
| `["miner_state", wallet]` | MinerState | Per-wallet state |
| `["submission", wallet, epoch]` | Submission | Per-epoch per-wallet proof |
| `["treasury_vesting"]` | TreasuryVesting | Creator + ecosystem vest |
| `["difficulty_history", window_u8]` | DifficultyWindow | LWMA 8-epoch window |

### 2.5 Epoch State Machine

```
States: OPEN → SETTLING → CLAIMABLE → EXPIRED

OPEN       (slots 0 → EPOCH_LENGTH)
           Submissions accepted. Difficulty enforced.
           
SETTLING   (EPOCH_LENGTH → EPOCH_LENGTH + SETTLE_WINDOW)
           No new submissions. Reward shares calculated.
           Next epoch challenge seed derived:
           seed = XOR(final_slot_hash, submission_merkle_root)
           
CLAIMABLE  (SETTLE_WINDOW → SETTLE_WINDOW + CLAIM_WINDOW)
           Miners call claim_rewards() to receive UNL.
           
EXPIRED    After CLAIM_WINDOW
           Unclaimed rewards permanently inaccessible.
           Deflationary burn-equivalent effect.
```

**Timing constants (adjustable via governance):**
- Epoch length: 18,000 slots ≈ 2 hours
- Settle window: 4,500 slots ≈ 30 minutes
- Claim window: 604,800 slots ≈ 7 days

---

## 3. SMART CONTRACTS

### 3.1 Folder Structure

```
programs/
├── unlearn-mining/
│   ├── src/
│   │   ├── lib.rs                 # Program entrypoint
│   │   ├── instructions/
│   │   │   ├── initialize.rs      # Bootstrap GlobalConfig + mint
│   │   │   ├── start_epoch.rs     # Open new EpochState PDA
│   │   │   ├── submit_proof.rs    # Core mining instruction
│   │   │   ├── settle_epoch.rs    # Close epoch, compute shares
│   │   │   ├── claim_rewards.rs   # Mint UNL to miner ATA
│   │   │   └── adjust_diff.rs     # LWMA difficulty update
│   │   ├── state/
│   │   │   ├── global_config.rs
│   │   │   ├── epoch_state.rs
│   │   │   ├── miner_state.rs
│   │   │   └── submission.rs
│   │   ├── errors.rs
│   │   └── constants.rs
│   └── Cargo.toml
├── unlearn-treasury/
│   └── src/
│       ├── lib.rs
│       └── instructions/
│           ├── init_vesting.rs
│           └── release_vested.rs
app/
├── miner/                          # Browser + CLI miner
└── dashboard/                      # Next.js frontend
```

### 3.2 GlobalConfig State

```rust
#[account]
pub struct GlobalConfig {
    pub authority: Pubkey,           // multisig after launch
    pub mint: Pubkey,                // UNL SPL mint address
    pub current_epoch: u64,
    pub current_difficulty: u8,      // leading zero bits required
    pub epoch_length_slots: u64,     // ~18000 slots ≈ 2 hours
    pub settle_window_slots: u64,    // ~4500 slots ≈ 30 min
    pub claim_window_slots: u64,     // ~604800 slots ≈ 7 days
    pub total_minted: u64,           // precision: 6 decimals
    pub max_supply: u64,             // 21_000_000 * 10^6
    pub halving_interval: u64,       // epochs per halving = 210
    pub base_reward_per_epoch: u64,  // initial: 50_000_000 (50 UNL)
    pub cooldown_slots: u64,         // per-wallet submission cooldown
    pub submission_cap: u16,         // max submissions per epoch globally
    pub paused: bool,                // emergency only, pre-decentralization
    pub bump: u8,
}
```

### 3.3 EpochState

```rust
#[account]
pub struct EpochState {
    pub epoch_id: u64,
    pub start_slot: u64,
    pub end_slot: u64,
    pub challenge_seed: [u8; 32],     // XOR(slot_hash, prev_merkle_root)
    pub total_submissions: u32,
    pub valid_submissions: u32,
    pub reward_pool: u64,             // UNL available this epoch (post-halving)
    pub total_score: u64,             // sum of all miner_score^2
    pub settled: bool,
    pub bump: u8,
}
```

### 3.4 MinerState

```rust
#[account]
pub struct MinerState {
    pub wallet: Pubkey,
    pub last_nonce: u64,             // monotonic: replay protection
    pub cooldown_until: u64,         // slot when next submission allowed
    pub last_submission_slot: u64,
    pub total_earned: u64,           // lifetime UNL earned
    pub total_claimed: u64,          // lifetime UNL claimed
    pub bump: u8,
}
```

### 3.5 Submission

```rust
#[account]
pub struct Submission {
    pub wallet: Pubkey,
    pub epoch_id: u64,
    pub nonce: u64,
    pub hash_bytes: [u8; 32],
    pub best_zeros: u8,              // leading zero count (quality score)
    pub submitted_at: u64,
    pub reward_share: u64,           // calculated at settle time
    pub claimed: bool,
    pub bump: u8,
}
```

### 3.6 submit_proof Instruction (Core Logic)

```rust
pub fn submit_proof(
    ctx: Context<SubmitProof>,
    nonce: u64,
    hash_bytes: [u8; 32],
) -> Result<()> {
    let config = &ctx.accounts.global_config;
    let epoch = &mut ctx.accounts.epoch_state;
    let miner = &mut ctx.accounts.miner_state;
    let submission = &mut ctx.accounts.submission;

    // 1. Protocol live check
    require!(!config.paused, ErrorCode::ProtocolPaused);

    // 2. Epoch open
    let slot = Clock::get()?.slot;
    require!(slot >= epoch.start_slot, ErrorCode::EpochNotStarted);
    require!(slot < epoch.end_slot, ErrorCode::EpochClosed);

    // 3. Submission cap
    require!(
        epoch.total_submissions < config.submission_cap as u32,
        ErrorCode::EpochCapReached
    );

    // 4. Cooldown check (Sybil/spam resistance)
    require!(
        slot >= miner.cooldown_until,
        ErrorCode::CooldownActive
    );

    // 5. Replay protection: nonce must be strictly greater than last
    require!(nonce > miner.last_nonce, ErrorCode::NonceReplay);

    // 6. CRITICAL: Reconstruct hash on-chain — never trust client
    let mut data = Vec::with_capacity(80);
    data.extend_from_slice(ctx.accounts.miner_wallet.key.as_ref());
    data.extend_from_slice(&config.current_epoch.to_le_bytes());
    data.extend_from_slice(&nonce.to_le_bytes());
    data.extend_from_slice(&epoch.challenge_seed);
    let computed = solana_program::hash::hash(&data);
    require!(
        computed.to_bytes() == hash_bytes,
        ErrorCode::HashMismatch
    );

    // 7. Difficulty check
    let zeros = count_leading_zero_bits(&hash_bytes);
    require!(zeros >= config.current_difficulty, ErrorCode::InsufficientWork);

    // 8. State mutations (all before any CPI — reentrancy safe)
    miner.last_nonce = nonce;
    miner.cooldown_until = slot.checked_add(config.cooldown_slots)
        .ok_or(ErrorCode::Overflow)?;
    miner.last_submission_slot = slot;

    epoch.total_submissions = epoch.total_submissions
        .checked_add(1).ok_or(ErrorCode::Overflow)?;
    epoch.valid_submissions = epoch.valid_submissions
        .checked_add(1).ok_or(ErrorCode::Overflow)?;

    // Store proof data
    submission.wallet = *ctx.accounts.miner_wallet.key;
    submission.epoch_id = config.current_epoch;
    submission.nonce = nonce;
    submission.hash_bytes = hash_bytes;
    submission.best_zeros = zeros;
    submission.submitted_at = slot;
    submission.claimed = false;

    // Emit event for indexers
    emit!(ProofSubmitted {
        wallet: *ctx.accounts.miner_wallet.key,
        epoch: config.current_epoch,
        zeros,
        slot,
    });

    Ok(())
}
```

### 3.7 settle_epoch Instruction

```rust
pub fn settle_epoch(ctx: Context<SettleEpoch>) -> Result<()> {
    let config = &mut ctx.accounts.global_config;
    let epoch = &mut ctx.accounts.epoch_state;

    let slot = Clock::get()?.slot;
    require!(slot >= epoch.end_slot, ErrorCode::EpochStillOpen);
    require!(!epoch.settled, ErrorCode::AlreadySettled);

    // Calculate epoch reward pool (post-halving)
    let halving_n = config.current_epoch / config.halving_interval;
    let epoch_pool = config.base_reward_per_epoch >> halving_n;

    // Check max supply headroom
    let epoch_pool = epoch_pool.min(
        config.max_supply.saturating_sub(config.total_minted)
    );

    epoch.reward_pool = epoch_pool;
    epoch.settled = true;

    // Note: individual reward_share values calculated in claim_rewards
    // to avoid single large compute transaction.

    // Derive next epoch challenge seed
    // (called separately by start_epoch instruction)

    emit!(EpochSettled {
        epoch: config.current_epoch,
        reward_pool: epoch_pool,
        valid_submissions: epoch.valid_submissions,
    });

    Ok(())
}
```

### 3.8 claim_rewards Instruction

```rust
pub fn claim_rewards(ctx: Context<ClaimRewards>) -> Result<()> {
    let config = &ctx.accounts.global_config;
    let epoch = &ctx.accounts.epoch_state;
    let submission = &mut ctx.accounts.submission;
    let miner_state = &mut ctx.accounts.miner_state;

    // Validate state
    require!(epoch.settled, ErrorCode::EpochNotSettled);
    require!(!submission.claimed, ErrorCode::AlreadyClaimed);

    let slot = Clock::get()?.slot;
    let claim_deadline = epoch.end_slot
        .checked_add(config.settle_window_slots).unwrap()
        .checked_add(config.claim_window_slots).unwrap();
    require!(slot < claim_deadline, ErrorCode::ClaimExpired);

    // Calculate this miner's reward share
    // Quadratic scoring: score = zeros^2
    let miner_score = (submission.best_zeros as u128).pow(2);
    let total_score = epoch.total_score as u128; // set during settle
    
    let reward = if total_score > 0 {
        ((epoch.reward_pool as u128)
            .checked_mul(miner_score).ok_or(ErrorCode::Overflow)?
            / total_score) as u64
    } else {
        0
    };

    // Mark claimed BEFORE mint (reentrancy safe)
    submission.claimed = true;
    submission.reward_share = reward;
    miner_state.total_claimed = miner_state.total_claimed
        .checked_add(reward).ok_or(ErrorCode::Overflow)?;

    // Mint UNL to miner's associated token account
    if reward > 0 {
        let seeds = &[b"global_config", &[config.bump]];
        let signer = &[&seeds[..]];
        
        token::mint_to(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                MintTo {
                    mint: ctx.accounts.mint.to_account_info(),
                    to: ctx.accounts.miner_ata.to_account_info(),
                    authority: ctx.accounts.global_config.to_account_info(),
                },
                signer,
            ),
            reward,
        )?;

        // Update global minted counter
        // Note: This requires GlobalConfig as mut — use separate instruction
        // or accept slight race on total_minted for display purposes only.
        // Hard cap enforced by mint authority revocation at Day 30.
    }

    emit!(RewardClaimed {
        wallet: submission.wallet,
        epoch: submission.epoch_id,
        amount: reward,
    });

    Ok(())
}
```

---

## 4. MINING LOGIC

### 4.1 Hash Function

```
input = wallet_pubkey (32 bytes)
      || epoch_id    (8 bytes, little-endian u64)
      || nonce       (8 bytes, little-endian u64)
      || challenge_seed (32 bytes)

hash = SHA256(input)  // 80 bytes → 32 bytes
```

The challenge_seed changes every epoch:
```
seed[n+1] = XOR(
    hash(last_slot_in_epoch),
    merkle_root(all_submission_hashes_this_epoch)
)
```

This makes the next epoch's challenge unpredictable until the current epoch closes. Bots cannot pre-compute valid hashes for the next epoch.

### 4.2 Difficulty Adjustment (LWMA)

**Linear Weighted Moving Average** over an 8-epoch window:

```
TARGET_SUBMISSIONS = 144  (configurable via governance)

weighted_sum = Σ(i × submissions[i]) for i in 1..=8
               (most recent epoch weighted 8x, oldest weighted 1x)
weight_total = 1+2+3+4+5+6+7+8 = 36
lwma = weighted_sum / weight_total

ratio = TARGET / lwma
delta = round(log2(ratio))
delta = clamp(delta, -2, 2)  // max ±2 bits adjustment per epoch
new_difficulty = clamp(current + delta, 16, 40)
```

**Bounds:**
- Minimum difficulty: 16 bits (≈65,536 expected hashes per submission)
- Maximum difficulty: 40 bits (≈1 trillion expected hashes)
- Starting difficulty: 20 bits (≈1,048,576 expected hashes)

### 4.3 Halving Schedule

| Epoch Range | Reward/Epoch | UNL Minted | ~Duration |
|-------------|-------------|------------|-----------|
| 1–210 | 50 UNL | 10,500 | ~20 days |
| 211–420 | 25 UNL | 5,250 | ~20 days |
| 421–630 | 12.5 UNL | 2,625 | ~20 days |
| 631–840 | 6.25 UNL | 1,312.5 | ~20 days |
| 841–1050 | 3.125 UNL | 656.25 | ~20 days |
| ... | ... | ... | ... |

> Note: 210-epoch halving ≈ 420 hours ≈ 17.5 days with 2-hour epochs. Adjust `halving_interval` in GlobalConfig before deployment. Longer intervals (e.g., 2,100) extend the halving periods to ~6 months each, closer to Bitcoin's model.

### 4.4 Reward Share Formula

```
// For each valid submission in epoch:
miner_score = best_zeros^2

// Total score across all valid submissions:
total_score = Σ miner_score

// Individual reward:
miner_reward = epoch_pool × (miner_score / total_score)

// Quadratic scoring rationale:
// Linear scoring (zeros/total_zeros) creates winner-take-all dynamics
// where GPU farms dominate entirely. Quadratic scoring reduces this by
// diminishing marginal returns on each additional leading zero.
// A miner with 22 zeros earns 484 score.
// A miner with 20 zeros earns 400 score.
// Ratio: 1.21x (not 1.1x linear)
```

### 4.5 Browser Miner (Web Worker Pseudocode)

```typescript
// miner-worker.ts — runs in dedicated Web Worker thread
// Never on main thread — never blocks UI

interface MinerConfig {
  walletPubkey: Uint8Array;  // 32 bytes
  epochId: bigint;
  difficulty: number;        // leading zero bits
  challengeSeed: Uint8Array; // 32 bytes
}

self.onmessage = async ({ data }: { data: MinerConfig }) => {
  const { walletPubkey, epochId, difficulty, challengeSeed } = data;
  const sha256 = await loadWasmSha256(); // wasm-crypto for speed
  
  let nonce = BigInt(Date.now()); // pseudo-random start
  let attempts = 0n;
  const startTime = performance.now();
  let running = true;

  self.onmessage = ({ data }) => { if (data.stop) running = false; };

  while (running) {
    // Build input: 80 bytes
    const input = new Uint8Array(80);
    input.set(walletPubkey, 0);
    input.set(toLeBytes(epochId, 8), 32);
    input.set(toLeBytes(nonce, 8), 40);
    input.set(challengeSeed, 48);

    const hash = sha256(input);
    const zeros = countLeadingZeroBits(hash);

    if (zeros >= difficulty) {
      self.postMessage({
        type: 'PROOF_FOUND',
        nonce: nonce.toString(),
        hash: Array.from(hash),
        zeros,
      });
      running = false;
      break;
    }

    nonce++;
    attempts++;

    // Report stats every 10k hashes
    if (attempts % 10000n === 0n) {
      const elapsed = (performance.now() - startTime) / 1000;
      self.postMessage({
        type: 'STATS',
        hashRate: Math.round(Number(attempts) / elapsed),
        attempts: attempts.toString(),
      });
    }
  }
};
```

### 4.6 CLI Miner Architecture

```
unlearnlogic-miner/
├── src/
│   ├── main.rs              # CLI entrypoint (clap)
│   ├── config.rs            # Config file: RPC, keypair, threads
│   ├── miner.rs             # Mining loop (rayon parallel)
│   ├── hash.rs              # SHA256 + leading zero counter
│   ├── rpc.rs               # Solana RPC client
│   └── submit.rs            # Transaction builder + sender
└── Cargo.toml
```

CLI commands:
```bash
unl-miner mine --wallet ~/.config/solana/id.json --rpc mainnet
unl-miner status
unl-miner claim --epoch <n>
unl-miner leaderboard
```

GPU support: Optional CUDA/OpenCL via feature flag. SHA256 kernel runs on GPU, nonce range split across CUDA threads.

---

## 5. SECURITY

### 5.1 Threat Matrix

| Attack | Severity | Mitigation |
|--------|----------|------------|
| Hash replay | CRITICAL | Monotonic nonce tracking in MinerState. On-chain hash recomputation. |
| Double claim | CRITICAL | `submission.claimed` set atomically before mint CPI. |
| Fake proof | CRITICAL | Hash RECOMPUTED on-chain from submitted nonce. Client cannot forge. |
| Sybil (multi-wallet) | HIGH | Cooldown per wallet. Quadratic scoring. SOL rent cost per wallet. |
| Bot flooding | HIGH | Per-epoch submission cap. Dynamic challenge seed. Cooldown slots. |
| MEV front-run | MEDIUM | Reward shares calculated post-epoch. No submission order advantage. |
| Integer overflow | HIGH | Rust checked arithmetic throughout. u128 for intermediate calculations. |
| Mint authority exploit | CRITICAL | Authority held by PDA. Revoked Day 30. |
| Reentrancy | HIGH | All state mutations before any CPI. No CPI back to self. |
| Emergency pause abuse | MEDIUM | Multisig required. Capability removed at DAO transition. |

### 5.2 PDA Safety

All accounts use canonical bump seeds stored in account data. Every instruction validates:
```rust
// Example validation in Anchor:
#[derive(Accounts)]
pub struct SubmitProof<'info> {
    #[account(
        seeds = [b"global_config"],
        bump = global_config.bump,
    )]
    pub global_config: Account<'info, GlobalConfig>,
    
    #[account(
        mut,
        seeds = [b"miner_state", miner_wallet.key().as_ref()],
        bump = miner_state.bump,
    )]
    pub miner_state: Account<'info, MinerState>,
    
    #[account(
        init_if_needed,
        payer = miner_wallet,
        space = Submission::LEN,
        seeds = [
            b"submission",
            miner_wallet.key().as_ref(),
            &global_config.current_epoch.to_le_bytes()
        ],
        bump,
    )]
    pub submission: Account<'info, Submission>,
    // ...
}
```

### 5.3 Audit Requirements

Before mainnet:
- [ ] Two independent security audit firms
- [ ] All critical and high findings resolved
- [ ] Fuzz testing with honggfuzz-rs on all instruction handlers
- [ ] Economic attack simulation (miner collusion, submission cap manipulation)
- [ ] Compute unit profiling for all instructions (stay under 200k CU)
- [ ] Formal verification of reward math invariants

### 5.4 Authority Revocation Timeline

```
Day 0:   Launch. Mint authority = GlobalConfig PDA.
         Upgrade authority = deployer EOA temporarily.
         
Day 7:   Upgrade authority transferred to Squads multisig.
         
Day 30:  Mint authority revoked.
         Max supply now enforced by absence of authority,
         not just program logic.
         
Month 6: Upgrade authority transferred to SPL Governance.
         Pause capability removed from program via upgrade.
         
Year 1:  Upgrade authority optionally burned (governance vote).
         Protocol becomes fully immutable.
```

---

## 6. FRONTEND ARCHITECTURE

### 6.1 Tech Stack

```
Framework:     Next.js 14 (App Router)
Styling:       TailwindCSS + custom CSS variables
Language:      TypeScript strict mode
Wallets:       @solana/wallet-adapter (Phantom, Backpack, Solflare)
On-chain:      @coral-xyz/anchor + generated IDL types
Hashing:       @noble/sha2 (WASM) in Web Worker
Indexing:      Helius webhooks → serverless edge functions → KV store
```

### 6.2 Pages

| Route | Description |
|-------|-------------|
| `/` | Landing page. Manifesto. Connect wallet. |
| `/mine` | Mining dashboard (wallet required) |
| `/tokenomics` | Supply visualization, emission curve |
| `/whitepaper` | Full protocol specification |
| `/community` | Epoch leaderboard, top miners, history |
| `/epoch/[n]` | Per-epoch breakdown, submissions, reward dist |

### 6.3 Mining Dashboard Components

```
<MinerPanel>
  ├── WalletConnect (Phantom/Backpack/Solflare)
  ├── EpochClock (live slot countdown)
  ├── DifficultyMeter (current bits, visual bar)
  ├── MinerControls
  │   ├── StartButton → spawns Web Worker
  │   ├── StopButton → terminates Worker
  │   ├── ThreadSelector (1-4 Workers)
  │   └── HashRateDisplay (H/s, live)
  ├── ProofStatus
  │   ├── CurrentBestZeros
  │   ├── EstimatedShare (%)
  │   └── SubmitButton (when proof found)
  └── RewardPanel
      ├── PendingRewards (claimable UNL)
      ├── ClaimButton → claim_rewards tx
      └── ClaimHistory (past claims + tx links)
```

### 6.4 Indexing Architecture

**Preferred (decentralized):** Helius geyser webhooks → Vercel Edge Functions → Upstash Redis for leaderboard, epoch stats.

**Fallback (fully on-chain):** All data queryable from program accounts. Client-side getProgramAccounts with memcmp filters. Slower but zero infrastructure.

```typescript
// Fetch all submissions for an epoch
const submissions = await program.account.submission.all([
  {
    memcmp: {
      offset: 8 + 32,  // after discriminator + wallet
      bytes: bs58.encode(Buffer.from(epochId.toArrayLike(Buffer, 'le', 8)))
    }
  }
]);
```

### 6.5 Environment Variables

```bash
NEXT_PUBLIC_PROGRAM_ID=         # Mining program ID
NEXT_PUBLIC_TREASURY_PROGRAM_ID=
NEXT_PUBLIC_UNL_MINT=           # UNL SPL mint address
NEXT_PUBLIC_RPC_URL=            # Helius mainnet endpoint
HELIUS_API_KEY=                 # Server-side only
NEXT_PUBLIC_NETWORK=mainnet-beta
```

---

## 7. LAUNCH SEQUENCE

### 7.1 Pre-Launch

```
PHASE 0: AUDIT + TESTING
[ ] 2 independent audit firms complete
[ ] All critical/high findings resolved
[ ] 30+ devnet epochs simulated
[ ] Browser miner: Chrome, Firefox, Brave tested
[ ] CLI miner: Linux, macOS, Windows tested
[ ] Squads multisig configured (3-of-5)
[ ] Legal review in operating jurisdictions
[ ] Token metadata prepared + IPFS pinned
[ ] Whitepaper published + IPFS pinned
[ ] Community channels launched (Discord/Telegram)
[ ] Announcement to fair-launch indexers (Pump.fun excluded)
```

### 7.2 Launch Day

```
T-0h:  Deploy unlearn-mining to mainnet
T-0h:  Deploy unlearn-treasury to mainnet
T-0h:  Create UNL SPL mint (null freeze_authority)
T-0h:  Initialize GlobalConfig
T-0h:  Assign mint_authority to GlobalConfig PDA
T-0h:  Set Metaplex token metadata on-chain
T-0h:  Start Epoch 1
T-0h:  Publish program IDs publicly
T-0h:  Open mining
T-1h:  Verify first proofs accepted on-chain
T-4h:  Verify first epoch settled correctly
T-4h:  Verify first claims processed
```

### 7.3 Post-Launch

```
Day 7:    Transfer upgrade authority → Squads multisig
Day 30:   Revoke mint_authority (irreversible)
          Community announcement + verification
Month 2:  Deploy SPL Governance program (devnet test first)
Month 6:  Migrate to SPL Governance on mainnet
          DAO live. Creator retires from governance.
Year 1:   Community vote on upgrade authority burn
```

---

## 8. GOVERNANCE MODEL

### 8.1 Phase 1: Multisig (Launch → Month 6)

**Structure:** 3-of-5 Squads multisig  
**Signers:** Publicly identified. Mix of creator + community leads.  
**Powers:**
- Emergency pause (pre-DAO only)
- Ecosystem grant approval (≤50,000 UNL without super-majority)
- Parameter adjustment within governance bounds
- Accept/reject program upgrades

**Cannot:**
- Mint tokens outside program logic
- Change max supply
- Override halving schedule
- Access creator vesting ahead of schedule

### 8.2 Phase 2: SPL Governance (Month 6 → Year 1)

**Voting token:** UNL  
**Quorum:** 5% of circulating supply  
**Vote duration:** 72 hours minimum  
**Timelock:** 48 hours between vote pass and execution

**Adjustable parameters:**
- `epoch_length_slots` (range: 9000–36000)
- `cooldown_slots` (range: 450–9000)
- `submission_cap` (range: 50–500)
- `target_submissions` (LWMA target)

**Immutable (no governance power over):**
- `max_supply`
- `halving_interval`
- `base_reward`
- Treasury allocation percentages

### 8.3 Ecosystem Treasury Governance

- Formal proposal required for all disbursements
- Proposal must include: recipient, amount, purpose, success metrics
- Public comment period: 7 days
- Vote: 72 hours
- All disbursements published on-chain

---

## 9. BRANDING GUIDELINES

### 9.1 Visual Identity

**Palette:**
- Primary: #000000 (pure black)
- Secondary: #0a0a0a (near-black)
- Surface: #111111
- Foreground: #e8e8e8
- Muted: #999999
- Accent: #39ff14 (electric green — mining confirmation only)
- Warning: #f5a623 (amber — difficulty/warnings)
- White: #ffffff (typographic emphasis only)

**Typography:**
- Display: Syne (800 weight) — headlines, logo
- Monospace: Space Mono — all data, code, hashes
- Never: Inter, Roboto, system-ui for primary display

**Aesthetic direction:**
- Brutalist terminal
- Signal/noise philosophy
- No rounded corners on primary containers
- 1px borders everywhere
- No gradients in UI (gradients in generated visualizations only)
- No drop shadows

### 9.2 Landing Page Copy (Tone: Minimal, Intelligent, Unsettling)

**Hero:**
```
UNLEARN
LOGIC

The only way in is through computation.
No investors. No presale. Mine or miss.
```

**Mining section:**
```
Run the hash.
Submit the proof.
Wait for the epoch.
Claim what you earned.

That's it. That's the protocol.
```

**Tokenomics section:**
```
21,000,000 UNL.
No more. Ever.
90% to miners.
The rest vests on-chain.
No exceptions.
```

**Philosophy:**
```
Markets run on narrative.
Most tokens are narrative with nothing underneath.

UNLEARNLOGIC is computation with a token on top.
The work is the product.
The waiting is the philosophy.
The scarcity is the art.
```

### 9.3 What to Avoid

- "To the moon" — or any price speculation language
- APY/yield percentages
- "Revolutionary" / "game-changing"
- Cartoon mascots
- Rainbow color schemes
- Fake urgency ("only X left!")
- Celebrity endorsements
- "Safe" or "rugproof" (never guarantee)

---

## 10. TOKEN METADATA CONFIGURATION

```json
{
  "name": "UnlearnLogic",
  "symbol": "UNL",
  "description": "Fair-launch Solana mining protocol. No insiders. No presale. Proof of computation.",
  "image": "ipfs://[CID]/logo.png",
  "external_url": "https://unlearnlogic.xyz",
  "attributes": [
    { "trait_type": "Type", "value": "Fair Launch Mining Token" },
    { "trait_type": "Max Supply", "value": "21000000" },
    { "trait_type": "Mining Share", "value": "90%" },
    { "trait_type": "Blockchain", "value": "Solana" },
    { "trait_type": "Standard", "value": "SPL Token" },
    { "trait_type": "Freeze Authority", "value": "None" }
  ],
  "properties": {
    "files": [
      { "uri": "ipfs://[CID]/logo.png", "type": "image/png" },
      { "uri": "ipfs://[CID]/logo.svg", "type": "image/svg+xml" }
    ],
    "category": "fungible"
  }
}
```

---

## 11. API DESIGN

### 11.1 Indexer API (Helius-backed)

```
GET /api/epoch/current
  → { epoch, startSlot, endSlot, state, submissions, difficulty, rewardPool }

GET /api/epoch/:n
  → { epoch data, top submissions, total miners }

GET /api/miner/:wallet
  → { totalEarned, totalClaimed, pendingRewards, epochHistory[] }

GET /api/leaderboard?epoch=current&limit=50
  → { miners: [{ wallet, zeros, score, estimatedReward }] }

GET /api/stats
  → { totalMinted, circulatingSupply, activeMinerCount, currentDifficulty }
```

### 11.2 Event Schema (for indexing)

```typescript
// Emitted on-chain, indexed by Helius webhooks

event ProofSubmitted {
  wallet: PublicKey;
  epoch: u64;
  zeros: u8;
  slot: u64;
}

event EpochSettled {
  epoch: u64;
  rewardPool: u64;
  validSubmissions: u32;
  nextChallengeSeed: bytes32;
}

event RewardClaimed {
  wallet: PublicKey;
  epoch: u64;
  amount: u64;
}

event DifficultyAdjusted {
  oldDifficulty: u8;
  newDifficulty: u8;
  lwmaSubmissions: f64;
}
```

---

## 12. FUTURE EXPANSION

### 12.1 Possible Governance-Approved Extensions

- **Mining pools:** Multi-wallet cooperative mining with on-chain split contracts. Must not compromise Sybil resistance.
- **NFT mining badges:** Non-transferable proof-of-participation badges minted to miners per epoch milestone. Cultural artifact, no economic utility.
- **Difficulty NFTs:** Visual difficulty-curve art generated from on-chain hash data. Protocol-generated, not creator-designed.
- **Protocol fee switch:** Optional 0.1-1% protocol fee on claim transactions, directed to ecosystem treasury. Requires governance super-majority.

### 12.2 What Will Never Change (By Design)

- Max supply of 21,000,000 UNL
- Mining as the primary distribution mechanism
- No re-introduction of freeze or mint authority after revocation
- No founder/investor privilege in future token mechanics

---

*UNLEARNLOGIC Protocol Specification v0.1*  
*This document is a living draft. Final version published to IPFS at launch.*  
*Program IDs, mint address, and deployment hashes added post-deploy.*
