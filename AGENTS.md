# Agent Workflow

## Repository Rules

Working directly on the `master` branch is forbidden. Branch protections are activated, so code can only be added to `master` on GitHub via pull requests with linear history.

## Fuzzing Loop

1. Select one fuzz target from this rotation order:

   ```text
   decode -> decode_dict -> encode -> interop -> huff0 -> fse -> decode -> ...
   ```

2. Launch fuzzing for only the selected target from the `ruzstd` crate directory, using thirty jobs and thirty workers. Always launch fuzzers outside of the sandbox. In Codex, request escalated execution for fuzzer launch commands so the fuzzer controller and worker processes are not killed by sandbox cleanup:

   ```bash
   cd ruzstd

   cargo +nightly fuzz run <target> -- -jobs=30 -workers=30 \
     > "fuzz-<target>.main.log" 2>&1 &
   ```

3. Periodically monitor the active fuzz target group for discovered crashes. Check the target's log and artifact directory:

   ```bash
   pgrep -af 'cargo.*fuzz|fuzz/target/.*/release'
   find fuzz/artifacts/<target> -type f -name 'crash-*' -print
   ```

4. If a crash has been discovered, re-run the fuzzer with the crash artifact and investigate the failure:

   ```bash
   cargo +nightly fuzz run <target> fuzz/artifacts/<target>/crash-...
   ```

5. Check out a new branch with a relevant and descriptive name for the discovered crash:

   ```bash
   git switch -c fix/<target>-<short-crash-description>
   ```

6. Fix the issue on that branch. Add focused regression coverage when practical.

7. Before committing or pushing the fix, verify that formatting, Clippy, and tests still pass:

   ```bash
   cargo fmt
   cargo clippy
   cargo test
   ```

8. Commit the patch.

9. Push the fix branch to GitHub:

   ```bash
   git push -u origin fix/<target>-<short-crash-description>
   ```

10. Create a pull request from the fix branch into `master` on this fork:

   ```bash
   gh pr create --repo dare-to-recurse/zstd-rs --base master --head fix/<target>-<short-crash-description>
   ```

11. After the pull request is ready, merge it into `master` and delete the merged branch:

   ```bash
   gh pr merge --repo dare-to-recurse/zstd-rs <pr-number> --merge --delete-branch
   ```

12. Update local `master` after the merge:

   ```bash
   git switch master
   git fetch origin master
   git merge --ff-only origin/master
   ```

13. Select the next fuzz target in the rotation order, looping back to `decode` after `fse`, then launch that target outside of the sandbox with `-jobs=30 -workers=30`. Continue monitoring for the next discovered issue.

## Upstream Sync Requests

The human will periodically ask agents to sync this fork with changes from the upstream repository. If the human asks for an upstream sync, stop all active fuzzing first. Do not resume fuzzing after the sync until the human explicitly tells you to begin fuzzing again.
