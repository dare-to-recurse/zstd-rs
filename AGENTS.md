# Agent Workflow

## Fuzzing Loop

1. Launch fuzzing for all six fuzz targets in parallel from the `ruzstd` crate directory, using five jobs and five workers per target:

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

9. Re-launch the fuzz target group that crashed with `-jobs=5 -workers=5`, then continue monitoring for the next discovered issue in any fuzz target group.
