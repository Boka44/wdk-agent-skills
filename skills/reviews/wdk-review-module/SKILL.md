---
name: wdk-review-module
description: Pre-review a WDK wallet or protocol module against the contribution guide, checking every rule and checklist item and reporting pass, fail, not run, or n/a. Use before requesting a WDK source review.
user-invocable: true
argument-hint: "[module root directory]"
---

# WDK Module Pre-Review

Review the module at the provided path (or the current project if none given) against the [contribution guide](https://docs.wdk.tether.io/tools/create-wdk-module/). Read the guide first. If this skill and the guide disagree, the guide wins. If the page has no "Pre-Review Checklist" section, say so and review against this skill alone.

## Before you start

- Run every command from the module root.
- Install dependencies with `npm ci`, or `npm install` if there is no lockfile.
- Use a `bare` at least as new as the highest `engines.bare` in the installed dependency tree. Check with `bare --version`. If it is older, install `bare` in a scratch directory and use that one. An old `bare` is a reason for not run, never for fail.

## Results

Give every item one result, with its evidence (the command and outcome, or the file and line):

- **pass**: you ran the command or read the code, and it holds.
- **fail**: it does not hold. Give the smallest fix.
- **not run**: you could not check it. Say what is missing.
- **n/a**: it does not apply to this module type, or the module has no such code path. Say why.

Code that is still a stub, a `TODO`, or returns `undefined` where the contract returns a value is a fail ("not implemented"), not a pass or n/a. When one cause fails several items, give each its result and write "same cause as Mx" instead of repeating the fix.

A claim in a README is not evidence that the code does it. Do not use real funds, real keys, or paid APIs unless the user says so. End with the count of each result, then the list of failures.

## Rules

### All modules

- **M1. Run on Bare.** `exports["."].bare` points to an entry that exists and loads `bare-node-runtime`. This is a static check; M19 runs it.
- **M2. Resolve one copy of `@tetherto/wdk-wallet`.** `package.json` declares a version range, not an exact version. No WDK package it depends on pins `@tetherto/wdk-wallet` to an exact version. If one does, fail and name it.
- **M3. Never lose a transaction.** After a submission, every result and error path keeps the transaction identifier and partial-execution details. A failed submission is distinct from an unknown broadcast outcome. If the module hands submission to the host account, pass when no code after the submission can throw or drop the result. n/a if the module adds and overrides no submission path.
- **M4. Add, don't redefine.** The module extends the base class for its type in the latest `@tetherto/wdk-wallet` release (`npm view @tetherto/wdk-wallet version`). A wallet built on another WDK wallet, such as `@tetherto/wdk-wallet-evm`, extends that package's classes instead. Each override keeps the parameters, meaning, and return shape of the method it replaces, and constructors keep every overload of the parent.
- **M5. Leave secrets and logs to the host.** `grep -rnE "console\.[a-z]+\(" src index.js bare.js` finds nothing. Provider requests and telemetry are disclosed in the README.

### Protocol modules

- **M6. Sign with the host's account.** Signing and submission go only through the account given to the constructor. There is no seed or private key input, and no second signer.
- **M7. Report status honestly.** An unrecognized status in a valid response maps to `pending` for swidge and SDA, or `in_progress` for fiat. n/a for swap, bridge, and lending.

### Wallet modules

- **M8. Override both account factories.** A variant that changes the account class overrides `getAccount()` and `getAccountByPath()`. Every factory, including `toReadOnlyAccount()`, returns the module's own classes.
- **M9. Clear their own state.** Extra keys or sessions are cleared in `dispose()`, after `super.dispose()`.

## Checklist

### Package

- **M10. Name.** `name`, with or without an `@scope/` prefix, matches what the CLI generates: `wdk-wallet-<chain>[-<variant>]`, `wdk-protocol-<swap|bridge|lending>-<name>-<chain>`, or `wdk-protocol-<fiat|swidge|sda>-<name>`. `npm view @tetherto/<name without scope> version` returns 404, because official package names are reserved, unless the module is that `@tetherto` package. If the package is already published, `npm view <name> maintainers` lists the maintainers for the user to confirm.
- **M11. Scaffold leftovers.** `find . -path ./node_modules -prune -o -name '*{{*' -print` finds nothing. A match means the project was generated on Windows by `create-wdk-module` 1.0.0-beta.4 or earlier; regenerate it on macOS or Linux, or with a release newer than 1.0.0-beta.4. `grep -rnIE "TODO|FIXME" --exclude-dir=node_modules --exclude-dir=.git .` finds nothing. The scaffold leaves `TODO` markers in `src/`, `tests/`, and `SECURITY.md`, and each one must be resolved and removed. List every match.
- **M12. Lint.** `npm run lint` passes. Note what `standard.ignore` leaves out of lint.
- **M13. Dependencies.** Every entry in `dependencies` is imported and used by `src/`, `types/`, `index.js`, or `bare.js`. Nothing they import is only in `devDependencies`.
- **M14. Packed install.** Pass when the install succeeds and the last command prints exactly one version:

  ```bash
  DIR="$(mktemp -d)"
  TARBALL="$DIR/$(npm pack --silent --pack-destination "$DIR")"
  (cd "$DIR" && npm init -y >/dev/null && npm install "$TARBALL" @tetherto/wdk && npm ls @tetherto/wdk-wallet --all | grep -o 'wdk-wallet@[^ ]*' | sort -u)
  ```

  `npm ls` exits 0 even when it finds two versions, so read the printed list. If a dependency pins the second copy, name it.

### Behavior

- **M15. Tests.** `npm test` passes, and `grep -rnE "(test|it)\.todo" tests` finds nothing. Jest reports a suite of only `todo` tests as passed, so a pass needs real tests.
- **M16. Main flow.** A test runs the main flow through `@tetherto/wdk` and a real wallet package, with no mocks or sender stubs, on a fork at a known block. If the provider or chain can't run on a fork, a live run on a testnet or mainnet replaces it. Give the block height, or the network and transaction hash, for the review request. The main flow is:
  - **wallet:** `registerWallet()`, `getAccount()`, read a balance, sign, and send one transfer;
  - **protocol:** `registerWallet()`, `registerProtocol()`, get the protocol from the account, and run its main operation, such as `swap()`, `bridge()`, `supply()`, `buy()`, or `createDepositAddress()`.
- **M17. Live coverage.** Live tests include at least one read or quote per supported network, and one complete write per flow kind on at least one network. The README lists any missing live coverage as a known gap, with the reason.
- **M18. Disposal.** If the module holds extra state, tests dispose through both the account and the manager and assert the state is gone.
- **M19. Bare runtime.** The M16 flow runs from the packed Bare entry with `bare <file>`, not through `npm test`, or the README documents an approved waiver. A test that only imports the entry does not count. If M16 has no flow, fail with "same cause as M16".

### Documentation

- **M20. README.** It has install steps, a runnable example, configuration, supported chains and operations, errors with caller actions, fees and fee cap support, known limitations, supported WDK versions, and support contacts. An example is runnable when it runs as written once the documented configuration and environment variables are set. List what is missing.
- **M21. Changelog.** `CHANGELOG.md` records each release, with migration steps for breaking changes.
- **M22. Security policy.** `SECURITY.md` says how to report a vulnerability privately.
- **M23. Caveats.** Each caveat that applies (bundler, Bare, credentials, real funds) exists and sits next to the run commands, not only in example comments. A missing caveat is a fail.
- **M24. Credentials.** The repository holds no real credentials. If tests or examples read `process.env`, `.env.example` lists every variable they read.
- **M25. Example assets.** Examples use Tether assets or neutral placeholders.
