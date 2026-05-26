# fastverk

Bazel-native development from silicon to seL4 — a constellation of
`rules_*` modules + a bzlmod registry that wires them together.

Each module is a single concern. Compose them to get a stack that
goes all the way from a KiCad PCB through a soft-CPU netlist
through a freestanding cross-compile through the seL4 microkernel
and out to qemu — all reproducible, all in Bazel.

## Modules

### Public registry — production-ready building blocks

| Module | Purpose | Latest | Smoke |
|---|---|---|---|
| [`rules_uv`](https://github.com/fastverk/rules_uv) | Python via `uv` (build-from-source + prebuilt) | 0.7.3 | ✓ |
| [`rules_lean`](https://github.com/fastverk/rules_lean) | Lean theorem prover | — | ✓ |
| [`rules_postgres`](https://github.com/fastverk/rules_postgres) | PostgreSQL tooling | — | ✓ |
| [`rules_github`](https://github.com/fastverk/rules_github) | GitHub API integration | — | ✓ |
| [`rules_jsonschema`](https://github.com/fastverk/rules_jsonschema) | JSON Schema validation | — | ✓ |
| [`rules_openapi`](https://github.com/fastverk/rules_openapi) | OpenAPI code generation | — | ✓ |
| [`rules_mdbook`](https://github.com/fastverk/rules_mdbook) | mdbook documentation | — | ✓ |
| [`rules_storybook`](https://github.com/fastverk/rules_storybook) | Storybook UI | — | ✓ |
| [`rules_nextjs`](https://github.com/fastverk/rules_nextjs) | Next.js | — | ✓ |
| [`rules_vite`](https://github.com/fastverk/rules_vite) | Vite bundler | — | ✓ |
| [`rules_bun`](https://github.com/fastverk/rules_bun) | Bun package manager | — | ✓ |
| [`rules_chrome`](https://github.com/fastverk/rules_chrome) | Chrome integration | — | ✓ |
| [`rules_docker_compose`](https://github.com/fastverk/rules_docker_compose) | Compose orchestration | — | ✓ |
| [`rules_gitlab`](https://github.com/fastverk/rules_gitlab) | GitLab CI/API | — | ✓ |
| [`rules_cloudformation`](https://github.com/fastverk/rules_cloudformation) | AWS CloudFormation | — | ✓ |
| [`rules_jena`](https://github.com/fastverk/rules_jena) | Apache Jena RDF | — | ✓ |
| [`rules_rdf`](https://github.com/fastverk/rules_rdf) | RDF/semantic web | — | ✓ |
| [`rules_autoconf`](https://github.com/fastverk/rules_autoconf) | GNU autoconf | — | ✓ |
| [`rules_ci_ir`](https://github.com/fastverk/rules_ci_ir) | CI intermediate representation | — | ✓ |
| [`rules_tectonic`](https://github.com/fastverk/rules_tectonic) | Tectonic LaTeX | 0.1.1 | ✓ |

### Embedded stack — silicon → seL4 (currently private during incubation)

The vertical-slice constellation. Wired together they boot
`hello, world` from a microkit PD on seL4 in qemu-system-aarch64,
fully hermetically.

```
                    rules_board
                   (PCB + soft-CPU
                    + microkit
                    platform glue)
                       /  |  \
                      /   |   \
              rules_kicad |  rules_verilog
              (hermetic   |  (Verilator,
               KiCad via  |   Yosys)
               DMG +      |       \
               hdiutil)   |      rules_chisel
                          |      (Mill-driven
                          |       Chisel→Verilog)
                          |          |
                          |    rules_riscv_core
                          |    (curated Ibex /
                          |     Rocket presets)
                          |
                  rules_microkit
                  (microkit_pd /
                   microkit_image /
                   microkit_qemu_run)
                       /  |  \
                      /   |   \
        rules_cc_cross    |    rules_qemu
        (hermetic ARM-    |    (hermetic
         GNU 14.2.rel1)   |     Homebrew
                          |     bottle install
                          |     w/ install_name_tool
                          |     relocation)
                          |
                rules_microkit_tool        rules_sel4
                (vendor SDK 2.2.0          (source-build
                 with prebuilt              seL4 kernel —
                 kernels per board)         alt path)
```

**Smoke-verified path**: every link in the chain through `bazel
test //examples/hello_on_qemu:boot_test` (in [`rules_microkit`](https://github.com/fastverk/rules_microkit)).

### Premium — coming soon

`rules_naga`, `rules_render`, `rules_blender` ride on a separate
private registry. See [bazel-registry/ROADMAP.md](https://github.com/fastverk/bazel-registry/blob/main/ROADMAP.md)
for the bifurcation design.

## Quick start

Add to your `.bazelrc`:

```
common --registry=https://raw.githubusercontent.com/fastverk/bazel-registry/main/
common --registry=https://bcr.bazel.build/
```

Add to your `MODULE.bazel`:

```python
bazel_dep(name = "rules_uv", version = "0.7.3")
# … etc.
```

That's it. See each module's README for module-specific install
instructions.

## Tooling

- **[bazel-registry](https://github.com/fastverk/bazel-registry)** — the bzlmod registry itself + the `rels`
  command for cutting releases, auditing modules, and (soon)
  driving the org-wide development console (see [BOTNOC.md](./BOTNOC.md)).

## Philosophy

- **Bazel-native first.** Every workflow that crosses module
  boundaries should be expressible as Bazel targets, not as
  out-of-band scripts.
- **Hermetic by default.** Each module either pins its upstream
  artifact's sha256 + extracts deterministically, or vendors a
  source tarball with the same. Host-tool dependencies (`hdiutil`,
  `install_name_tool`, `codesign`) are limited to OS-provided
  utilities that don't drift.
- **Honest about gaps.** Modules ship at `0.0.x` with explicit
  "structural-only, no smoke" labels when they're not yet
  end-to-end verified. We don't pretend.
- **One thing per module.** `rules_cc_cross` does cross compilers.
  `rules_microkit` orchestrates microkit. `rules_board` glues
  KiCad + Verilog + microkit. Splitting beats coupling.

## Contributing

Each module has its own issues/PRs. For org-wide coordination
(cross-module bumps, registry-tier moves, agent dispatch), see
[BOTNOC.md](./BOTNOC.md).
