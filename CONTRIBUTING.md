# Contributing to aero-foundation

Thank you for helping. This project trains a language model whose rewards come from engineering tools, so the bar for a contribution is the same as for any engineering change: state your assumptions, show your evidence, and make it reproducible.

## Ground rules

1. **A claim is a hypothesis until a check says otherwise.** Any statement about model capability must come with the run that produced it: commit, config, seed, tool versions, and the eval report.
2. **No restricted material.** Do not add, attach, paste, or train on SAE, RTCA, EUROCAE, AIAA, ESDU, IEEE, or commercial-textbook content, or anything export-controlled (ITAR/EAR) or marked with a limited distribution statement. If a public document is ambiguous, open an issue on the `data` track and wait for review.
3. **Never invent sources.** Every reference in `research/` must link to a page you opened. Paraphrase; don't quote at length.
4. **Rewards must be hard to game.** A verifier change needs tests that include inputs which should score zero: malformed output, physically implausible values, attempts to read the reference answer, timeouts.
5. **Traceability is a feature.** Data, tools, and models are versioned and hashed. If your change adds a data source, a tool, or a reward function, it also updates the matching manifest.

## Tracks

| Track | Covers | Label |
| --- | --- | --- |
| Data | Corpus sourcing, license tagging, filtering, dedup, decontamination | `track:data` |
| Verifiers | Tool wrappers, reward functions, sandboxing, cross-checks | `track:verifiers` |
| Eval | Benchmarks, held-out task sets, regression evals | `track:eval` |
| Certification | DO-330 qualification strategy, traceability, evidence | `track:certification` |

## Workflow

1. Find or open an issue using the **Research / engineering task** template. Say which track it belongs to and what "done" means.
2. Branch from `main` (`track/short-description`, for example `verifiers/avl-trim-reward`).
3. Keep pull requests small and single-purpose. Fill in the PR template's evidence section.
4. A maintainer listed in [`.github/CODEOWNERS`](.github/CODEOWNERS) reviews. Verifier, reward, and certification changes need explicit approval.

## Commit messages

Use the imperative mood and a short subject (`Add AVL trim-reward wrapper`). Explain *why* in the body when it isn't obvious.

## Licensing of contributions

The project is licensed under [Apache-2.0](LICENSE). By submitting a contribution you agree it is licensed under the same terms (Apache-2.0 section 5). Some verifier tools are GPL- or LGPL-licensed; we invoke them as separate processes and do not vendor or link their code into this repository. Don't change that without opening an issue first.

## Conduct

Be direct, be kind, and argue with evidence. Harassment or personal attacks are not tolerated; maintainers may remove content or contributors who don't follow this.
