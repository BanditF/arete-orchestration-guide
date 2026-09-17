# Understanding AI orchestration

This small repository explains what an orchestrator does by walking through one realistic workflow.

An orchestrator coordinates tools and models. It breaks a larger request into smaller tasks, assigns each task to an appropriate worker, combines the results, and pauses when a human decision is required.

Start with [ORCHESTRATOR.md](ORCHESTRATOR.md). It contains the explanation and the example flow. The executable-looking YAML is there to make the routing concrete; it is a communication example, not a requirement to install Arete.

The [flow file](flows/issue-to-reviewed-pr.yaml) shows the same workflow separately in machine-readable form.

This repository contains no private infrastructure, credentials, host details, or project source.
