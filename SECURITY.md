# Security policy

## Reporting a vulnerability

Please **do not open a public issue** for security problems.

Report privately through GitHub: go to the repository's **Security** tab and choose **Report a vulnerability**, or use [this link](https://github.com/aerospace-superintelligence/aero-foundation/security/advisories/new). We aim to acknowledge reports within 5 business days.

Include what you found, how to reproduce it, and the impact you expect.

## What counts as a security issue here

Beyond the usual software vulnerabilities, this project treats the following as security issues:

- **Verifier escapes.** A model output, tool input file, or generated code path that escapes the verifier sandbox, reads files or network resources it shouldn't, or executes arbitrary commands on the training host.
- **Reward exploits.** A way to obtain high reward without solving the task, such as reading the reference answer, crashing a tool into a default "pass", or exploiting a parser. Low-severity exploits can be filed publicly with the `reward-integrity` label; anything that also reaches the host goes through private reporting.
- **Data poisoning or provenance tampering.** Changes that would silently insert unlicensed, export-controlled, or malicious content into a corpus, or that alter provenance manifests or hashes.
- **Leaked credentials** in the repository, issues, or CI logs.

## Scope and intended use

aero-foundation is a research project for **ground-based engineering tools**. Nothing in this repository is airborne software, and no output of the model is qualified for certification credit. Treat every model output as an unverified proposal.

## Supported versions

The project is pre-release. Only the `main` branch receives fixes.
