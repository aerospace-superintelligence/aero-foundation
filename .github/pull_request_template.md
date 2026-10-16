## Summary

<!-- What this changes and why. Link the issue: "Closes #123". -->

## Track

- [ ] data
- [ ] verifiers
- [ ] eval
- [ ] certification
- [ ] training / infrastructure
- [ ] docs / site

## Evidence

<!-- How you checked it. Commands run, eval or verifier output, before/after numbers.
     Claims about model capability need a reproducible run: config, seed, commit, tool versions. -->

## Checklist

- [ ] Every new data source has a license recorded in the provenance manifest; nothing restricted (SAE, RTCA, AIAA, paid textbooks) or export-controlled is included.
- [ ] Verifier or reward changes include tests, including at least one case that should get **zero** reward.
- [ ] Tool versions are pinned and any new tool binary is hashed.
- [ ] No eval or held-out task content leaks into training data.
- [ ] Every cited source has a working link; nothing is quoted at length.
- [ ] Docs updated where behavior changed.
