# Advance Rust for Solana (Solana Turbine (aka Web3 Builders Alliance))

I have been taking this awesome course since past 3 weeks;
lets explore what it was all about and what we have been learning in it:

## Week 1 - [Invoke Context](https://github.com/anza-xyz/agave/blob/master/program-runtime/src/invoke_context.rs)
[Invoke Context Notes](https://github.com/dattgoswami/solana-notes/blob/main/invoke_context.md)
After going through some Rust concepts, our first task was to see if there are any improvements/optimizations that we could do. So for this refactoring my approach was to check the entire codebase for hotspots as to where `invoke_context` is being utilized and what are the possible changes that I could make to those functions, viz. get_compute_budget(), get_check_aligned(), get_sysvar_cache(), etc.
These are the ones calling `get_check_aligned()` mostly:
agave/programs/bpf_loader/src/syscalls/cpi.rs
agave/programs/bpf_loader/src/syscalls/logging.rs
agave/programs/bpf_loader/src/syscalls/mem_ops.rs
agave/programs/bpf_loader/src/syscalls/mod.rs

Getting to the actual refactoring, I was thinking of:
1. Adding comments
2. Improving error handling / error messages
3. Breaking bigger blocks/functions into smaller chunks to improve readability and maintainability
4. Reading tests to understand how its building the state `test_process_instruction()` to then be able to write more tests 

Solution:
actual implementation:
```
    pub fn get_check_aligned(&self) -> bool {
        self.transaction_context
            .get_current_instruction_context()
            .and_then(|instruction_context| {
                let program_account =
                    instruction_context.try_borrow_last_program_account(self.transaction_context);
                debug_assert!(program_account.is_ok());
                program_account
            })
            .map(|program_account| *program_account.get_owner() != bpf_loader_deprecated::id())
            .unwrap_or(true)
    }
```
removing unwrap_or and using map_or instead:
```
        .map_or(true, |program_account| *program_account.get_owner() != bpf_loader_deprecated::id())
```
Looked at some of the PRs in the Agave / Solana Labs codebase to find some that might be relevant to runtime / invoke context:
i) https://github.com/anza-xyz/agave/pull/1180
ii) https://github.com/anza-xyz/agave/pull/180
iii) https://github.com/solana-labs/solana/issues/34057
iv) https://github.com/anza-xyz/agave/pull/2441/files

Solana programs are compiled to BPF bytecode, the `bpf_loader/src/lib.rs` its `load_program_from_bytes` [method](https://github.com/anza-xyz/agave/blob/a10cd5548d2e21d10b3e43a52af2684333425f26/programs/bpf_loader/src/lib.rs#L63) loads this byte code

Also, compared the codebase of Agave, Rakurai & the Solana Labs client.
https://github.com/solana-labs/solana/
https://github.com/anza-xyz/agave
https://github.com/rakurai-io/rakurai-agave

## Week 2 - [ReplayStage](https://github.com/anza-xyz/agave/blob/master/core/src/replay_stage.rs)
[Replay Stage Notes](https://github.com/dattgoswami/solana-notes/blob/main/replay_stage.md)
Watched Justin Starry's video from the Artisan cohort from Q2, 2024. [TVU - Transaction Validation Unit](https://docs.solanalabs.com/validator/tvu) uses the Replay Stage. There were two groups one was looking into optimizing the [main replay loop]((https://github.com/anza-xyz/agave/blob/a10cd5548d2e21d10b3e43a52af2684333425f26/core/src/replay_stage.rs#L654) and optimizing using dash map (instead of hash map) and in the other one we were looking into metrics with Ernest, Jack, Nate, Tim.

In specific the [measure](https://github.com/anza-xyz/agave/tree/master/measure) and [metrics](https://github.com/anza-xyz/agave/tree/master/metrics).
`measure.rs` is where the timing related utilities reside [measure.rs](https://github.com/anza-xyz/agave/blob/master/measure/src/measure.rs)
metrics is where the logging related macros reside [datapoint.rs](https://github.com/anza-xyz/agave/blob/master/metrics/src/datapoint.rs)

I was looking into `testnet-automation.sh` & `init-metrics.sh` this is where InfluxDB(time series database) and Grafana dashboards are being setup. [README](https://github.com/anza-xyz/agave/blob/master/metrics/README.md) mentions the cluster and the fee market information that we can check on these logs. 

Public dashboards:
https://metrics.solana.com/d/monitor-edge/cluster-telemetry
https://metrics.solana.com/d/0n54roOVz/fee-market
https://metrics.solana.com/d/UpIWbId4k/ping-result

## Week 3 - SVM API & Building our own L2 (Rollup that uses SVM for execution)

There was this example of state channels [Paytube](https://github.com/buffalojoec/paytube) that we were independently looking into. The image in the README is important and the tests in this were particularly helpful.
Also, this article by Joe lays out the detail of the [runtime](https://fluff-ranunculus-275.notion.site/The-Agave-Runtime-d1f8d3608e5d4529b120e09e80b48887) very well. I'd recommend that everyone should check it out.

[Mollusk](https://github.com/buffalojoec/mollusk) is a minimal version of Paytube, where in a program execution pipeline is provisioned of the lower-level SVM components. Its just calling `process_instructions` (processing instructions) using BPF Loader.  

For the L2 we (Ahzam, Andrew, Jimii, Jonathan and I) started thinking about the key components that we would need:
1. How would we store the Accounts for the L2(rollup)?
2. Do we need on chain programs(escrow, state store)?
3. How does a rollup differ from a sidechain or a state channel?
4. Are we going to need a Data Availability layer?
5. Which L1 are we using for settlement/consensus?
6. What part of the PayTube example can we use for the rollup?
7. What are different types of rollups? Optimistic / zk
	When to use one vs the other?
8. How would the L2 post transaction data to L1? Compressed TXN, TXNs, state diff
9. What other type of L2's exist? Rollup, State Channel, Side Chain, Valadium
10. Reading up on papers on Ephemeral Rollup from MagicBlock
11. Reading up on Eclipse, Risc0, Espresso, Spicenet, Termina
12. Do we need a Bridge?
13. Do we need a sequencer?
We were discussing about Gaming & NFT rollup.

In the next session we were working on the architecture diagram for gaming rollup and finalizing the key blocks that we would need. 
We decided to use:
Solana L1 for settlement //skipped for POC
Optimistic Rollup //but then we looked into Sovereign SDK
Celestia for DA as its mature(other options NearDA, Walrus, EigenDA, Avail)
	-Ahzam looked into its docs and only Sovereign SDK made sense
	-we decided to use it and build a sovereign rollup for POC and then swap these components later with Solana L1 settlement

Irvin gave a very interesting talk on L2's. Check out his paper on [SVM Rollups](https://arxiv.org/abs/2405.08882)

Note: Thank you instructors from [Solana Turbine](https://x.com/solanaturbine) -> [japarjam](https://x.com/japarjam), [Berg](https://x.com/bergabman), Nate, Dean, all the colleagues in the cohort and Joe from Anza! 

References / Further Deep Dive:
[SVM API](https://www.anza.xyz/blog/anzas-new-svm-api)
[Modular SVM repo](https://github.com/buffalojoec/modular-svm)

[Solana Consensus](https://www.youtube.com/watch?v=StDx4VhZIVk)

[SVM Community Call on Interoperability](https://youtu.be/2nbY6clDKIY?si=C8JH5GbI_-qx0jPt)

PR's for interesting SIMDs:
https://github.com/solana-foundation/solana-improvement-documents/pull/161
https://github.com/solana-foundation/solana-improvement-documents/pull/162
https://github.com/solana-foundation/solana-improvement-documents/pull/163
https://github.com/solana-foundation/solana-improvement-documents/pull/64

https://github.com/solana-labs/solana/issues/34196
https://github.com/anza-xyz/agave/issues/389

Runtime Docs:
https://github.com/anza-xyz/agave/blob/master/docs/src/runtime/programs.md
https://github.com/anza-xyz/agave/blob/master/docs/src/runtime/sysvars.md

https://docs.optimism.io/stack/explainer
https://lightning.network/docs/
https://dev.risczero.com/api/getting-started
https://docs.spicenet.io/
https://termina.gitbook.io/documentation
https://mystenlabs.com/blog/announcing-walrus-a-decentralized-storage-and-data-availability-protocol
https://github.com/MystenLabs/example-walrus-sites

https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/rollup-interface/specs/overview.md
https://sovereign.mirror.xyz/pZl5kAtNIRQiKAjuFvDOQCmFIamGnf0oul3as_DhqGA
