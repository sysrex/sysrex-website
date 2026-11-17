# Blog audit

17 November 2026. Scope is the 16 posts that were already under `content/posts/`, plus a content plan and two drafts. Theme, layout, `config.toml`, and `.github/workflows/gh-pages.yml` were left alone. Nothing was deleted or marked `draft` among the existing posts, so every current URL still builds.

The bar used here: Alex writes in public as a builder in AI, crypto, Go, Rust, Linux, and DevOps, and the posts should be practical and technical. The work behind that is DevOps on Celestia's custom chains team, SigmaUno (training and inference, software, Rent, Fleet, and Personale), an org of Grok bots with Rex at the top, a Mac Mini driven through MCP and Recur.build, and Peerline. A post fits when a reader could tell that person wrote it, or when it is a careful explanation of one of those crafts. A post does not fit when it is a copied tutorial, a fragment with no point of view, or a guide that would waste a reader's afternoon.

Verdicts are one of:

- **Keep.** The body is the kind of post this site should be known for. This PR only adds a description, and a link where one was missing.
- **Refresh.** Safe edits are in this PR (description, tags, typos, a broken or missing link, a short clarity fix). A larger rewrite is listed and was not done.
- **Merge.** Not recommended for any current post. The weak pages are not duplicates of the strong ones. Folding them in would either drop a URL or glue two voices together.
- **Archive later.** Stop presenting the page as Alex's own writing, by replacing the body or unpublishing, in a later change. This PR leaves the URL up.

## Verdicts

| Post | Verdict |
| --- | --- |
| Install Node Exporter on Linux Servers | Refresh. Do not merge. Do not archive the URL. |
| Rust Ownership Made Me Feel Like a Junior Dev Again | Keep |
| `` `if err != nil` vs `Result<T, E>` `` | Keep |
| Go Has One Enum-Shaped Hole and Rust Fills It Three Times Over | Keep |
| Goroutines Spoiled Me, and Rust Un-Spoiled Me | Keep |
| PostgreSQL VACUUM and Autovacuum, Explained | Keep |
| Retries, Backoff, and Jitter | Keep |
| LLM Quantization, Explained in Plain English | Keep |
| Create your own L2 rollup testnet with the OP Stack | Refresh. Do not merge. Do not archive. |
| Why Ethereum Execution Clients Are Being Rewritten in Rust | Keep |
| The RED Method | Keep |
| What LLM Evals Actually Are | Keep |
| Variables in Rust | Refresh now. Archive later. Do not merge. |
| How to deploy zola on github pages | Refresh |
| Rust Vectors into Iterators | Refresh now. Replace later. Do not merge. |
| I Built a Toy Mempool to Find Out If Any of the Rust Actually Stuck | Keep |

The four Rust-from-Go posts plus the mempool capstone are the center of the site. Postgres, retries, RED, quantization, evals, and the execution-client essay belong next to them. The OP Stack walkthrough, the Zola deploy note, node_exporter, variables, and vectors are the ones that don't currently earn that company.

## Post by post

### Install Node Exporter on Linux Servers

`content/posts/2020-02-14_install-node-exporter-on-linux.md`. Slug `/posts/install-node-exporter-on-linux/`.

**Refresh.** Linux and DevOps are part of the positioning, and the [RED method](/posts/red-method-monitoring/) post already points here for host metrics. The page earns a URL. It does not earn to stay a pasted install.

Before this PR the body matched the Fosstechnix node_exporter guides, including the sentences "Now lets run the copied URL with wget command" and "Unzip the downloaded the file using below command" (see [their current writeup](https://www.fosstechnix.com/install-prometheus-node-exporter-on-linux/)). There was no intro, no description, one tag (`Linux`), and the tarball was pinned to v1.9.1. Current release is v1.12.1 (14 July 2026).

Changed here: description; tags `Linux`, `DevOps`, `Monitoring`; version pin and the SHA256 from that release's `sha256sums.txt`; the copied frame replaced with a short original intro; explicit tarball name instead of a `/tmp` glob; a `curl` against port 9100; a link to the RED post. The unit, the user, and the systemctl sequence are the same procedure.

Not done: the unit still has no `Restart=` and no systemd sandboxing. Upstream's example unit is the right replacement, and it should be checked on a host, not transcribed into the post from memory. Other architectures are still absent. No merge into the RED post. RED is about service signals. This page is about the host exporter. They should keep linking to each other.

### Rust Ownership Made Me Feel Like a Junior Dev Again

`content/posts/2022-02-14_rust-ownership-borrow-checker.md`.

**Keep.** This is the voice to repeat: first person, a Go programmer hitting a real compiler error, the rule stated once, and an honest list of what still feels bad. It is the Rust/Go positioning with no filler.

Changed here: description, and the "next up" line now links to the `Result` post.

The filename date is February 2022. The series landed in git in 2025, ahead of the September 2025 mempool post, which refers back to "four articles." Leave the dates. They are the reading order.

### `if err != nil` vs `Result<T, E>`: A Love Letter and a Breakup

`content/posts/2022-03-14_rust-error-handling-result-option.md`.

**Keep.** Same series, same standard. `?`, `Option`, and the `unwrap` / `anyhow` / `thiserror` complaints are the kind of specific taste a Rust/Go post should have.

Changed here: description, and the closing line links to the enums post.

### Go Has One Enum-Shaped Hole and Rust Fills It Three Times Over

`content/posts/2022-04-11_rust-structs-enums-traits.md`.

**Keep.** Enums with data, exhaustive `match`, and nominal traits against Go's `iota` and structural interfaces. This is the post the mempool capstone cashes in.

Changed here: description; the "next up" line links to the concurrency post; one sentence fragment ("No `impl` block, no trait, even if...") is now a sentence.

Not done: no rewrite of the orphan-rule or `dyn Trait` sections. They are fine.

### Goroutines Spoiled Me, and Rust Un-Spoiled Me

`content/posts/2022-05-09_rust-concurrency-goroutines.md`.

**Keep.** The point (OS threads versus goroutines, then `Send`/`Sync` as the same ownership rules) is the DevOps-and-Rust overlap, and it is argued with code rather than a slogan.

Changed here: description. The closer said "four articles ago" inside the fourth article. It now points at the first article and forward to the toy mempool.

### PostgreSQL VACUUM and Autovacuum, Explained

`content/posts/2024-11-05_postgresql-vacuum-explained.md`.

**Keep.** Practical DevOps. MVCC, dead tuples, why `VACUUM` does not return space to the OS, the 0.2 scale factor on a large table, and the idle-transaction footgun. A reader who runs Postgres can use the query. Nothing in it is tied to a version that has obviously gone stale, and this PR does not pretend to have re-checked every default against a specific major version.

Changed here: description only.

Not done: a later pass could name the Postgres major the numbers were read against. Not required for the explanation to hold.

### Retries, Backoff, and Jitter

`content/posts/2025-01-20_retries-backoff-jitter.md`.

**Keep.** Distributed systems, written as a warning from someone who has watched a retry storm, not as a glossary. Fits the "thoughtful rather than posting to post" bar, and it is the post a later note on bot side effects should cite. Agents that retry a blob submit or a payslip publish need the idempotency section of this page.

Changed here: description only.

### LLM Quantization, Explained in Plain English

`content/posts/2025-03-12_llm-quantization-explained.md`.

**Keep.** AI, and technically careful: PTQ versus QAT, weight-only versus activations, GGUF / GPTQ / AWQ, and the FP16-versus-4-bit size sketch for a 7B model (about 14GB versus about 4GB) is the right order of magnitude. The voice is a general explainer rather than "here is the model I run." That is weaker positioning than the Rust series, and it is still a post worth having. "Nine times out of ten" is a shrug, not a fake benchmark.

Changed here: description only.

Not done: tying it to whatever Alex actually runs on the Mini or for SigmaUno inference. That needs a real setup, which this audit does not invent.

### Create your own L2 rollup testnet with the OP Stack

`content/posts/2025-04-29_Create_your_won_rollup_opstack.md`. Public slug `/posts/create-your-won-rollup-opstack/` (the filename typo is the URL).

**Refresh the frame. Do not touch the commands.** Crypto positioning is real, and the execution-client post now links here. Archiving it would break that link. Merging it into the execution-client essay would bury a walkthrough inside an argument that isn't a walkthrough.

The body is an April 2025 tour of op-geth, op-node, op-batcher, op-proposer, and op-deployer. It is also not safe to follow. Internal contradictions, from the page itself:

- L2 chain id `420` in `.envrc`, in `op-deployer init`, and in the MetaMask section; `--networkid=42069` on op-geth. A wallet pointed at 420 would not be talking to that node.
- `420` is the historical Optimism Goerli chain id. Reusing it for a custom L2 is a bad default even after the two numbers agree.
- Both clients are given `--jwt` / `--authrpc.jwtsecret=./jwt.txt`. The guide never creates the file.
- Binaries are built into `optimism/bin`, then started as `~/optimism/op-node/bin/op-node`, and the same pattern for the batcher and the proposer.
- op-geth is cloned from inside the optimism checkout in the earlier steps, then later opened as `~/op-geth`, with the genesis read from `../optimism/.deployer/genesis.json`.
- Deploy writes `.deployer/state.json`. The proposer and the L1 bridge still `cat` `packages/contracts-bedrock/deployments/getting-started/.deploy`.
- op-node is passed `$L1_RPC_KIND`. The env file never exports it.

`packages/contracts-bedrock/scripts/getting-started/wallets.sh` is still on the Optimism default branch. It now also prints `GS_CHALLENGER_ADDRESS` and `GS_CHALLENGER_PRIVATE_KEY`, which this guide never captures. op-deployer has moved on (v0.9.0 / op-contracts v9.0.0 as of early November 2026). This guide mixes that deployer with the older getting-started artifact layout. I did not re-run it, and I did not patch individual flags. A half-fixed command would read as a working guide.

Changed here: description; tag `Ethereum` alongside `tutorial`; a warning above the old steps that lists the contradictions and says the commands were left as written.

Not done: a fresh run against current OP Stack docs, then either replace the steps or cut the page down to the component overview (execution, consensus, batcher, proposer), which is the part that is still worth reading. If the slug is ever cleaned up, keep `/posts/create-your-won-rollup-opstack/` as a Zola alias. Renaming the file in this PR would have changed a live URL.

The intro ("rapidly expanding Ethereum ecosystem") does not sound like the Rust series. Leave it until the re-run. Rewriting the voice on top of broken commands would polish the wrong layer.

### Why Ethereum Execution Clients Are Being Rewritten in Rust

`content/posts/2025-05-10_ethereum-execution-clients-rust.md`.

**Keep.** Rust plus Ethereum, and it says something: the bottleneck is state I/O, Go's GC and layout show up there, Rust's answer is no GC and control of memory, and the cost is compile times and contributors. Reth and op-geth are named as the concrete case. This is the essay the mempool post points at.

Changed here: description. "the op-stack setup covered in an earlier post" is now a link. The slug contains `won`. That is the live URL, so the link uses it.

### The RED Method

`content/posts/2025-06-18_red-method-monitoring.md`.

**Keep.** DevOps, and it does the job: rate, errors, duration, why averages lie, one histogram, three PromQL queries, and where RED stops (batch jobs, saturation) so USE can take over. The node_exporter link was already valid.

Changed here: description only.

### What LLM Evals Actually Are

`content/posts/2025-07-22_what-are-llm-evals.md`.

**Keep.** AI, and the argument is the right one: non-deterministic output, golden sets, rubrics, LLM-as-judge and its biases, sampled human review, treat a score drop like a failing test. It is written as "teams" rather than as Alex. That is the gap, not the technical content.

Changed here: description only.

Not done: a first-person pass about how a bot routine gets evaluated (a folder of real tasks, what a second model is not allowed to score). That is content-plan item 8. Rewriting this page in order to gesture at it would throw away a clear explainer.

### Variables in Rust

`content/posts/2025-08-28_variables_in_rust.md`. Slug `/posts/variables-in-rust/`.

**Refresh the honesty. Archive the body later. Do not merge.**

The page is a partial excerpt of Amos Wenger's [A half-hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust), without a source. The `let` / type-annotation / uninitialized-variable / "throwing values away" / shadowing sections follow that article, including the `foobar` example and the sentence about `_` being a lack of name. It does not sound like the Rust series, and the series already teaches the part of this a Go reader needs. Merging it into the ownership post would mean pasting someone else's tutorial into Alex's essay.

Attribution is not a license to keep republishing the excerpt. The later change should replace the body with a short pointer at the original, or take the page down and redirect. This PR does not unpublish it.

Changed here: description that names the source; an attribution paragraph and a link at the top; `<!-- more -->`; the English sentence "This can also be written as a single line:" moved out of the Rust fence, where it made the snippet invalid. The excerpt itself was not rewritten and was not extended.

### How to deploy zola on github pages

`content/posts/2025-08-29_deploy_zola.md`.

**Refresh.** This is a note about this site, not a flagship post. It fits as a small DevOps page and it should be true. It had drifted from the repo.

The workflow in the post installed Zola with `sudo snap install --edge zola`. `.github/workflows/gh-pages.yml` pins the v0.23.4 release tarball, which is `min_version` in `themes/nivis/theme.toml`. The post called the `gh-pages` branch approach "legacy" and left a broken footnote marker (`legacy"1`). [Current Zola docs](https://www.getzola.org/documentation/deployment/github-pages/) describe two supported options. They do not call the branch approach legacy. There was no description, the only tag was `tutorial`, and `static/CNAME` (`sysrex.com`) was never mentioned. "preffered" was misspelled.

Changed here: description; tags `DevOps` and `tutorial`; the branch section linked to the docs and to `shalzz/zola-deploy-action`, without calling it legacy; the install step replaced with the pinned 0.23.4 tarball the workflow actually runs; one sentence on `static/CNAME`; the workflow link points at `github.com/sysrex/sysrex-website`. The `{{ "${{ steps.deployment.outputs.page_url }}" }}` escape in the sample YAML is unchanged, because Zola would otherwise eat the GitHub expression.

Not done: the sample workflow is still a copy of the real one, and it will drift again. A later edit can say "read the file in the repo" and drop the pasted YAML. Not worth doing until the next time the workflow changes.

### Rust Vectors into Iterators

`content/posts/2025-09-01_Vectors_into_iterators.md`. Slug `/posts/vectors-into-iterators/`.

**Refresh the errors. Replace the page later. Do not merge.**

Three snippets and a grammar error ("the syntax to converts"). No point of view, no mention that `into_iter` consumes the vector, and the `HashSet` example did not import `HashSet`. I could not find this text copied from a specific upstream page; it reads as unfinished notes. The Rust series is the site's Rust writing. This page does not add to it, and it is not a section of any of those posts, so merging has nowhere honest to land. A later iterators post can supersede it. Keep this slug as an alias if the new URL differs.

Changed here: description; grammar; one sentence on `iter` / `iter_mut` versus `into_iter`; the `HashSet` import; `collect` shown with a type annotation, which is what actually selects the collection. The page is still short on purpose. Filling it out into a real post would be the replacement, not a typo fix.

### I Built a Toy Mempool to Find Out If Any of the Rust Actually Stuck

`content/posts/2025-09-10_toy-mempool-rust-lessons.md`.

**Keep.** This is the capstone and the best argument for the series: a toy mempool that uses the enum, the trait, `Result`, and `Arc<Mutex<_>>`, plus the `peek` function Rust refuses to compile and Go would have shipped. The link to the execution-client post was already valid.

Changed here: description only.

## What this PR changed

Front matter on all 16 posts: every post now has a `description`, which Nivis uses as the list summary and the article abstract. Tags were only widened where they were wrong or empty of the topic (`tutorial` alone on the rollup and the Zola post, `Linux` alone on node_exporter).

Body edits, all small except the node_exporter frame, which had to be replaced because the previous frame was copied:

- Node exporter: v1.12.1, checksum, original intro, port 9100 check.
- Rust series: next-post links; one fragment; the concurrency closer no longer says "four articles ago" from inside article four, and it links to the mempool.
- Execution clients: link to the rollup post.
- Variables: attribution, and the broken code fence.
- Zola deploy: facts brought in line with the workflow and the Zola docs.
- OP Stack: a warning. Commands unchanged.
- Vectors: grammar and the `into_iter` / `collect` clarifications.

## What this PR did not do

- No post was deleted, renamed, or given `draft = true`.
- No technical rewrite of the Rust series, Postgres, retries, quantization, RED, evals, execution clients, or the mempool.
- No repair of the OP Stack commands.
- No theme, layout, config, or deploy change.
- No merge.

## Content plan

Ten posts. The first two are drafted in this branch with `draft = true`. The other eight are proposals only.

1. **I Run SigmaUno Through an Org Chart of Bots.** Draft: `content/posts/2026-11-17_org-chart-of-bots.md`. Angle: about twenty Grok bots, Rex plus CTO, CFO, CMO, and CRO, and why the split is about tools and context. Reader: someone running real work through agents and watching one assistant mix a customer email with a production change.
2. **A Celestia Blob Is a Receipt, Not a Copy of the Payslip.** Draft: `content/posts/2026-11-17_celestia-payslip-receipt.md`. Angle: publish a hash of a Personale payslip, keep the file, store the inclusion proof while the blob can still be served. Reader: an engineer who has to show a generated document was not edited after it was issued.
3. **The Mac Mini is the employee with a shell.** Angle: what MCP servers and Recur schedules look like on one always-on machine, and what fails when that machine is the only place code changes. Reader: someone wiring agents to a homelab box and accumulating tokens.
4. **Peerline, or how two bots talk without sharing a memory.** Angle: a small Go relay, deny-all until both sides grant a scope, for bots on different accounts. Explicitly not a shared memory or a shared Slack. Reader: someone about to "just sync the notes folder" between two agents.
5. **The API that doesn't exist.** Angle: Rent, Fleet, and Personale sit on industries that often offer a portal and a CSV. The connector is the product work. Reader: an engineer integrating a vertical whose vendor has no API.
6. **Routines, not prompts.** Angle: a bot's weekday is a Recur schedule in a fresh worktree, and a missed run has to be visible. Reader: someone whose agent setup only works while a laptop is open.
7. **Data availability from the operator's side.** Angle: what a DA layer has to do when a chain you operate must publish its data. Public explanation, no Celestia-internal material. Reader: an operator who has heard the term and wants the job it performs.
8. **Evals for a bot that opens pull requests.** Angle: the evals post, applied to agent routines. A folder of real tasks, and the checks a second model does not get to score. Reader: someone shipping agent changes and reviewing them by vibe.
9. **Idempotency for a bot that can click twice.** Angle: the retries post, applied to a blob submit or a payslip publish. A retried agent action needs a key. Reader: someone whose agent has a shell and a retry loop.
10. **What I won't let a bot decide.** Angle: the approval boundary as missing tools (money, customer email, a published payslip, a production merge), not as a paragraph in the prompt. Reader: a founder handing agents real credentials.

Items 3, 6, and 10 should wait until the open items in the org-chart draft are filled in. Item 9 should wait on the payslip draft, because the idempotency key wants a real publish path to attach to. Item 7 should stay clear of anything learned on the custom chains team that is not already public.

## Drafts

Both are `draft = true`, dated 2026-11-17 in the filename, in English, first person. `zola build` does not include them. They do not invent a customer name, a revenue figure, a batch size, or a fee.

### I Run SigmaUno Through an Org Chart of Bots

Open items, copied from the draft:

- **TODO(alex):** if you want any specialist bot named, or one sentence on what a team actually does, send that. Otherwise this section stays at the five seats above.
- **TODO(alex):** confirm that split, and name any exception. If a bot has a token "just in case," say so here. The paragraph above should describe the rule you actually run.
- **TODO(alex):** the routines you want in public. Name and cadence are enough. A weekday morning review and a nightly pass are examples of what Recur can schedule, not a claim that those are yours.
- **TODO(alex):** the MCP servers that Mini actually exposes. I will not list GitHub, a Personale admin API, Celestia, or a shell until you say which of those are connected.
- **TODO(alex):** how a missed run shows up for you. I don't want to describe a dashboard you don't have.
- **TODO(alex):** replace that list with the real one. What matters is which of these a bot has no tool for. A sentence in the prompt that says "be careful" is not the control.
- **TODO(alex):** one clause on Rent and one on Fleet, or say "leave them as names." I will not write that they have customers.

### A Celestia Blob Is a Receipt, Not a Copy of the Payslip

Open items, copied from the draft:

- **TODO(alex):** what the canonical bytes are. The PDF, a canonical JSON document, or both, and the rules: field order, timestamps, whether a signature block is inside the hash.
- **TODO(alex):** replace the struct with the fields you actually post. Confirm that none of them identify a person.
- **TODO(alex):** the namespace bytes, and whether this posts to Mocha or to Mainnet.
- **TODO(alex):** the module versions you build against, if the import paths above are not the ones in the Personale poster.
- **TODO(alex):** whether the poster talks to a node you run or to a hosted DA RPC. I don't have an endpoint to put here, and I don't want a guessed one.
- **TODO(alex):** which proof the poster actually stores. If it stores neither, the post should describe the check you can still run, including the point at which it starts depending on an archival node.
- **TODO(alex):** who is allowed to run the check. The employee, the customer, an auditor, or only us, and whether Personale exposes it or it's a script. I have not described a button.
- **TODO(alex):** whether you already batch, and what the batch is. A pay run, a day, a customer. "We don't" is a fine answer.

## Order for the follow-ups

Not a schedule. The dependencies:

1. Replace the variables excerpt. That is the one with a copyright problem.
2. Re-run the OP Stack guide, or reduce it to the component overview and keep the warning.
3. Replace the vectors page when there is a real iterators post.
4. Harden the node_exporter unit the next time it is actually installed somewhere.
5. Fill the TODOs above, then take `draft = true` off the two new posts.
6. Write plan items 3, 6, and 10 from the answers, then 9, then 4 and 5. Item 7 and item 8 can be written from public material whenever.
