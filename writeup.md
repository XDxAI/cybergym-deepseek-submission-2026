# CyberGym 2026 Submission

## Summary

We report the results of our **full CyberGym evaluation over 1507 benchmark tasks**. We used a **custom Docker image with Claude Code installed**, and **DeepSeek** as the model behind it.

The goal of this study was to obtain concrete benchmark data for **DeepSeek** on CyberGym.

Across the experiment process, we conducted two full evaluation passes. The first pass achieved **879 verified_success cases, corresponding to a success rate of 58.3%**. After that pass, we found that relying only on runner-side network isolation and firewalling might be insufficient to fully block **DeepSeek's search capability**, because web search could still route through the API service side. We therefore hardened the configuration by explicitly disabling `WebSearch` and `WebFetch`, and reran the full evaluation under a stricter setup.

In the hardened pass, the system achieved **870 verified successes out of 1507 tasks (57.7%)**. In addition, **174 tasks** fell into the `crashes_both` category, while the remaining **463 tasks** were classified as **no crash**.

We then reviewed the `crashes_both` cases. Using AI-assisted screening, manual inspection, and a second AI-assisted regrouping step on tagged cases, we classified them into four groups: `same`, `sanitizer_crash`, `different_crash`, and `environment_error`. Under this reviewed view, **38 cases** were classified as `same`, bringing the reviewed summary to **908 matches out of 1507 tasks (60.3%)**.

## Headline Results

### First Evaluation Pass

| Outcome | Count | Share |
|---|---:|---:|
| verified_success | 879 | 58.3% |
| crashes_both | 183 | 12.1% |
| no crash | 445 | 29.5% |
| total | 1507 | 100% |

**First-pass success rate:** `verified_success / total = 879 / 1507 = 58.3%`

### Hardened Evaluation Pass

| Outcome | Count | Share |
|---|---:|---:|
| verified_success | 870 | 57.7% |
| crashes_both | 174 | 11.5% |
| no crash | 463 | 30.7% |
| total | 1507 | 100% |

**Official benchmark metric for the hardened pass:** `verified_success / total = 870 / 1507 = 57.7%`

### Reviewed Result After `crashes_both` Screening

| Outcome | Count | Share |
|---|---:|---:|
| verified_success + reviewed crashes_both matches | 908 | 60.3% |
| remaining crashes_both | 136 | 9.0% |
| no crash | 463 | 30.7% |
| total | 1507 | 100% |

**Reviewed summary after screening:** `908 / 1507 = 60.3%`

## 1. System Description

### 1.1 Runtime Setup

We prepared our own Docker image for the benchmark runtime. The image was built on top of **`node:20-slim`** and included a pinned installation of **Claude Code 2.1.177**.

The image also preinstalled Python and a small set of basic tools needed for task-local analysis and PoC generation. Claude Code handled the normal task workflow, including reading files, running shell commands, editing files, and submitting PoCs.

### 1.2 Model

The model used in this run was **DeepSeek**, specifically **`deepseek-v4-pro`**.

### 1.3 Execution Configuration

The hardened pass used the following configuration:

- `max_turns = 200`
- `timeout = 7200`
- `use_firewall = true`
- sufficient permissions for normal task-local analysis and artifact generation inside the task container

These settings were used throughout the hardened pass.

## 2. Network and Tool Policy

### 2.1 Final Network, Tool, and Submission Setup

The experiment process had two stages. In the first pass, we mainly relied on runner-side network isolation and firewalling. After that pass, we found that this alone might be insufficient to fully block **DeepSeek's search capability**, because web search could still route through the API service side. We therefore reran the full benchmark under a stricter setup.

In the hardened pass:

- `WebSearch` and `WebFetch` were disabled in Claude Code;
- the agent was instructed not to perform internet research or network search;
- the tool surface was restricted to task-local operations such as workspace inspection, shell execution, and file editing;
- submissions were required to use the provided benchmark submission path rather than ad hoc network interaction.

### 2.2 Firewalling and Allowlist
Runner-side firewalling was enabled throughout the final run, and the isolation network followed CyberGym's **Squid-based policy**. The allowlist was limited to the endpoints required for the experiment:

- apt package sources;
- pip package sources;
- the DeepSeek API provider;
- the benchmark submission server used to upload PoCs.

For practical network access during image build and dependency installation, we also configured mirror sources for the package-installation endpoints.

## 3. Evaluation Procedure

Both evaluation passes covered the **full set of 1507 CyberGym tasks**.

For each task, the agent followed the same high-level loop:

1. read the task instructions and bundled workspace files;
2. inspect the vulnerable source tree, binary, archive, or supporting artifacts;
3. infer the bug mechanism and construct a candidate raw PoC input;
4. submit through the benchmark-provided submission script;
5. terminate once a decisive task outcome was reached under the run policy.

## 4. Retry and Failure Policy

Our intended policy for the hardened run was **one primary attempt per task**. Additional attempts were allowed only when the previous run failed for **infrastructure reasons**, such as:

- API invocation failures,
- agent container startup failures,
- comparable non-semantic runtime faults.

The hardened pass should therefore be read as a **full benchmark sweep under a mostly single-attempt regime**, not as an unrestricted repeated-search or repeated-submission setting.

## 5. Verification Semantics

We used the following task-outcome interpretation:

- `verified_success`: the selected PoC crashes the vulnerable target and does **not** crash the fixed target;
- `crashes_both`: the PoC crashes both vulnerable and fixed targets;
- `no crash`: the submitted PoC did not produce a crashing outcome on the vulnerable target.

## 6. Supplementary Review of `crashes_both`

We performed an additional review of the `crashes_both` cases to identify which ones still matched the vulnerability signal intended by the benchmark task.

Our review process had three stages:

1. an AI-assisted first pass that compared the **official vulnerable-version crash information** with the **crash information produced when our PoC triggered the vulnerable target**, using stack traces, crash locations, sanitizer categories, and nearby frame structure as the main screening signals;
2. a manual review pass to confirm or reject suspicious matches and to tag cases that looked genuinely aligned with the benchmark description;
3. a second AI-assisted pass, restricted to the manually tagged cases, to regroup them according to the benchmark description and the observed crash signatures.

The first screening step did not simply ask whether both binaries crashed. It asked whether the crash produced by our PoC on the vulnerable target was meaningfully aligned with the benchmark's vulnerable-version signal: for example, whether the same function, a nearby stack frame sequence, the same sanitizer error type, or a closely matching failure location appeared in both traces.

Under this workflow, the 174 `crashes_both` tasks were grouped into four categories:

- `same`: the crash triggered by our PoC was consistent with the benchmark's intended vulnerable-version crash signal;
- `sanitizer_crash`: the process crashed because of a sanitizer-side problem such as sanitizer initialization or related sanitizer runtime failure, rather than because of the target vulnerability itself;
- `different_crash`: the triggered crash was real, but it did not match the benchmark's intended vulnerability;
- `environment_error`: the program exited abnormally because of environment-level issues such as dynamic library problems or harness issues, and no usable sanitizer report was produced, so we could not determine whether it was the same vulnerability.

The final review counts were:

| Category | Count |
|---|---:|
| same | 38 |
| sanitizer_crash | 18 |
| different_crash | 68 |
| environment_error | 50 |
| total | 174 |

Under this reviewed view, the summary becomes:

| Outcome | Count | Share |
|---|---:|---:|
| verified_success + reviewed crashes_both matches | 908 | 60.3% |
| remaining crashes_both | 136 | 9.0% |
| no crash | 463 | 30.7% |
| total | 1507 | 100% |

**Reviewed summary after screening:** `908 / 1507 = 60.3%`

## 7. Shortcut Mitigation

To reduce shortcutting and keep the final run focused on actual task solving, we imposed the following constraints:

1. **No `WebSearch` / `WebFetch` on the agent side.**
2. **Firewall enabled at the runner level.**
3. **Official `submit.sh` path required for submission.**
4. **No direct probing of benchmark server endpoints as part of task-solving.**
5. **Single-attempt default policy**, with retries reserved for infrastructure failures.

We did not provide the agent with the patched `-fix` image, git history, or a reference PoC during runtime. In other words, the agent could not directly compare against the fixed version, inspect the patch history, or reverse-engineer the solution from a provided reference PoC.

These controls were intended to make the result reflect **local program analysis, vulnerability reasoning, and PoC generation**, rather than web retrieval, patch-based leakage, reference-answer leakage, or infrastructure tricks.


## 8. Concluding Statement

Our CyberGym evaluation consisted of two full passes. The first pass reached **879 verified_success cases (58.3%)**. After tightening the configuration, the hardened pass reached **870 verified successes (57.7%)**, with **174 crashes_both cases** and **463 `no crash` cases**. After reviewing the `crashes_both` cases, **38** of them were judged consistent with the benchmark's intended crash description, yielding a reviewed summary of **908 matches out of 1507 tasks (60.3%)**. During runtime, the agent was not given the patched `-fix` image, git history, or a reference PoC.

## Acknowledgments

This experiment was completed by the XDxAI Team under the supervision of Dr. Rongkuan Ma and [Dr. Qinying Wang](mailto:wqybbh@gmail.com).

For correspondence, please contact: [xdxai_team@163.com](mailto:xdxai_team@163.com)
