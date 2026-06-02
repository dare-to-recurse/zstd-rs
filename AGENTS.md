# Agent Workflow

## Repository Rules

Working directly on the `master` branch is forbidden. Branch protections are activated, so code can only be added to `master` on GitHub via pull requests with linear history.

## Fuzzing Loop

1. Launch fuzzing for all six fuzz targets in parallel from the `ruzstd` crate directory, using five jobs and five workers per target. Always launch fuzzers outside of the sandbox. In Codex, request escalated execution for fuzzer launch commands so the fuzzer controller and worker processes are not killed by sandbox cleanup:

   ```bash
   cd ruzstd

   for target in decode decode_dict encode interop huff0 fse; do
     cargo +nightly fuzz run "$target" -- -jobs=5 -workers=5 \
       > "fuzz-${target}.main.log" 2>&1 &
   done
   ```

2. Periodically monitor each fuzz target group for discovered crashes. Check each target's logs and artifact directory:

   ```bash
   pgrep -af 'cargo.*fuzz|fuzz/target/.*/release'
   find fuzz/artifacts -type f -name 'crash-*' -print
   ```

3. If a crash has been discovered, re-run the fuzzer with the crash artifact and investigate the failure:

   ```bash
   cargo +nightly fuzz run <target> fuzz/artifacts/<target>/crash-...
   ```

4. Check out a new branch with a relevant and descriptive name for the discovered crash:

   ```bash
   git switch -c fix/<target>-<short-crash-description>
   ```

5. Fix the issue on that branch. Add focused regression coverage when practical.

6. Before committing or pushing the fix, verify that formatting, Clippy, and tests still pass:

   ```bash
   cargo fmt
   cargo clippy
   cargo test
   ```

7. Commit the patch.

8. Push the fix branch to GitHub:

   ```bash
   git push -u origin fix/<target>-<short-crash-description>
   ```

9. Create a pull request from the fix branch into `master` on this fork:

   ```bash
   gh pr create --repo dare-to-recurse/zstd-rs --base master --head fix/<target>-<short-crash-description>
   ```

10. After the pull request is ready, merge it into `master` and delete the merged branch:

   ```bash
   gh pr merge --repo dare-to-recurse/zstd-rs <pr-number> --merge --delete-branch
   ```

11. Update local `master` after the merge:

   ```bash
   git switch master
   git fetch origin master
   git merge --ff-only origin/master
   ```

12. Re-launch the fuzz target group that crashed outside of the sandbox with `-jobs=5 -workers=5`, then continue monitoring for the next discovered issue in any fuzz target group.

## Upstream Sync Requests

The human will periodically ask agents to sync this fork with changes from the upstream repository. If the human asks for an upstream sync, stop all active fuzzing first. Do not resume fuzzing after the sync until the human explicitly tells you to begin fuzzing again.
