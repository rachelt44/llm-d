# llm-d

## Skills (Agent Skills standard)
- Skills live in `skills/`. Each skill is a folder with required `SKILL.md` (YAML frontmatter + Markdown).
- Use the skill by reading `SKILL.md` and following the instructions.

## Available Skills
- [llm-d-kubertenes-deployment](skills/llm-d-kubernetes-deployment/SKILL.md) - Deploy and configure llm-d (high-performance distributed LLM inference) on Kubernetes clusters using Well-lit Path guides for production-ready LLM serving with optimizations like intelligent inference scheduling, prefill/decode disaggregation, wide expert parallelism, and tiered prefix caching.
- [run-llm-d-benchmark](skills/run-llm-d-benchmark/SKILL.md) - Runs a workload against an already deployed llm-d stack (high-performance distributed LLM inference), using one of several available benchmark harnesses.
