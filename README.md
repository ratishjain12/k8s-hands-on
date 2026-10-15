# Kubernetes, hands-on

A learning path for engineers who want to understand Kubernetes by building on it, not by copying YAML.
It is written for AI/backend engineers, so later phases cover GPU scheduling, model serving, and autoscaling.

## How this works

- **No copy-paste.** Every manifest in `labs/` is typed by the learner. The course gives tasks and questions, not solutions.
- **Predict, then verify.** Each exercise asks what you expect to happen before you run it.
- **Break things on purpose.** The most useful notes are about what failed and why.
- **Local first.** Everything through phase 4 runs on a local [kind](https://kind.sigs.k8s.io/) cluster at no cost.

## Layout

| Path | Purpose |
|---|---|
| [setup/](setup/README.md) | Install and verify tooling |
| [ROADMAP.md](ROADMAP.md) | Phases and progress tracker |
| `phases/` | Concepts, exercises, and hints for each phase |
| `labs/` | The learner's own manifests and code, one folder per exercise |
| `notes/` | Your notes, one folder per phase: concepts learned, and a debug log of what broke, why, and the fix |
| `projects/` | Larger projects that combine several phases |

## Using this repo

1. Do [setup](setup/README.md).
2. Work through [ROADMAP.md](ROADMAP.md) in order.
3. For each phase: read its `README.md`, attempt `exercises.md`, and open `hints.md` only when stuck.
4. Write your work in `labs/<phase>/` and your findings in `notes/<phase>/`.

## Prerequisites

Comfort with the terminal, basic Docker (images, containers, ports), and YAML.

## License

[MIT](LICENSE)
