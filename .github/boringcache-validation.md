# llama.cpp CPU Docker cache validation

This fork compares the upstream full CPU amd64 Docker build and push with two compiler-cache integrations. Both integrations apply [the same ccache patch](boringcache-ccache.patch). Upstream source is selected separately from the validation branch, using [boringcache-upstream](boringcache-upstream) and the captured sequence in [boringcache-commits.json](boringcache-commits.json).

| Provider | Layer cache | Compiler cache |
| --- | --- | --- |
| Registry | Fork-owned GHCR registry cache, `mode=max` | None, unchanged upstream Dockerfile |
| Ccache | Separate fork-owned GHCR registry cache, `mode=max` | `/root/.ccache` injected and extracted by buildkit-cache-dance, persisted through actions/cache |
| BoringCache | Managed BoringCache Docker layer cache | The same `/root/.ccache` mount, persisted by the Docker adapter |

The ccache patch installs ccache, sets its directory and 5 GB limit, mounts that directory during compilation, and prints statistics after resetting them before compilation. Upstream CMake already discovers ccache when installed. GCC 14, Release mode, all CPU variants, dynamic backends, disabled build tests, build parallelism, the UI build, full-image packaging, and Python dependencies remain as upstream defines them. No other dependency cache is added.

| Phase | Source | Change |
| --- | --- | --- |
| Cold | `4b1622afb7dccc819df968d1ef9bb5ebeeac5538` | Seed, four first-parent commits before captured master |
| Warm | Same seed, same source archive | Unchanged layer reuse on fresh builders |
| Rolling 1 | `13b4d7135a6351f81e1eccf6361a4eafd50350eb` | Metal transfer-buffer release |
| Rolling 2 | `42d958167a748f2c04b1f888e84e7a58f609ddcb` | CUDA MMVQ routing |
| Rolling 3 | `2b36825cbc39b06ed9512b487a67127206e474af` | Python conversion metadata |
| Rolling 4 | `d775ebf363f777bc862a8659b6c55eec9419cf35` | Server embedding-request validation |

The first two forward changes affect backends outside this CPU build. The conversion change affects the full package. The final change modifies compiled server source. These are real upstream changes; the workflow does not synthesize a mutation or force a Docker cache miss. This short sequence is a screening test. It does not sample successive daily publication revisions.

Each arm uses a fresh standard GitHub-hosted Ubuntu 24.04 amd64 runner and a fresh builder. A source job archives the complete checkout, including `.git`, once per source revision. Every arm downloads the same archive. Warm runs reuse the cold run's exact archive through the `source_run` input, so fresh-checkout Git metadata cannot invalidate the unchanged context. The workflow checks source and archive identity and retains the effective Dockerfile and patch. `BUILD_DATE` is fixed to the source commit date for all arms and the unchanged repeat; `APP_VERSION` is the checked-out source's Git description. Daily upstream jobs use their publication time and source tag, so their UI/label invalidation can differ from this controlled repeat.

The full image is published by digest to `ghcr.io/boringcache/llama-cpp-validation`. No upstream image tags or release tags are published. Registry cache exports fail on error. BoringCache cache errors fail the build; warm additionally requires a cache hit. Cold and rolling publish cache state; warm only restores it. Conventional mount extraction and actions/cache saving happen in post steps, so include those steps in full job timing and verify their outcomes in the raw logs.

The image check pulls the published digest, checks amd64 and the source-revision label, runs `llama-cli --version` and `llama-server --version`, and records binary/library checksums. This is a packaging and startup check. It does not establish inference correctness or performance. Compare output checksums across the same-source arms after each run and investigate any difference before interpreting timings.

The workflow retains build-and-push wall time, runner details, source/workflow/run identities, Docker versions, native ccache output in job logs, image metadata and output checksums, and product-emitted BoringCache evidence. Report source preparation, setup, image checks, mount extraction, cache saving, and complete job/workflow time separately. Read live compiler statistics only when the compile RUN executes; a Docker layer hit can replay a prior ccache report. Missing transfer-byte measurements are unavailable, not zero.

Dispatch [boringcache-validation.yml](workflows/boringcache-validation.yml) on `boringcache-validation` with `phase=connect` and approve the fork's GitHub Machine connection to `boringcache/llama-cpp-validation`. The pinned Action uses its reviewed release's default CLI. An exact `cli_version` is only a dispatch-time override.

After connection, run `phase=cold` and retain its run ID. Then run `phase=warm` with `source_run` set to that cold run ID. Advance `boringcache-upstream` through the four recorded commits one at a time, committing and pushing each source pin, and dispatch `phase=rolling` after the preceding run completes. Keep all three cache identities unchanged throughout the sequence. Repeating a cold comparison requires unused cache identities for all three arms.

GitHub requires a manually dispatched workflow to exist on the fork's default branch. Only the dispatch workflow needs registration there; every job refuses to build from a branch other than `boringcache-validation`. The upstream default branch is `master`. The upstream source checkout contains no validation configuration.

This screening test covers the full CPU amd64 image build and push. It excludes the subsequent light and server image steps, architecture merging, attestations, GPU inference, CUDA, and ROCm. It cannot establish complete publication-job savings or attribute every benefit of adding ccache to BoringCache storage. Add daily source revisions and the upstream CUDA 12.8.1 amd64 build only after CPU output and cache behavior are validated.
