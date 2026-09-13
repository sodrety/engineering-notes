---
title: 'When the Evaluation Sandbox Is Not a Sandbox'
description: 'What recent cyber-evaluation incidents teach us about network boundaries, credentials, monitoring, and kill switches for AI agents.'
pubDate: 2026-09-13
heroImage: '../../assets/agent-evaluation.png'
heroImageAlt: 'Minimal hand-drawn flow diagram showing an agent moving through a sandbox, policy gate, and telemetry before a red blocked boundary stops access to the public internet.'
---

An AI-agent evaluation can fail before the model makes its first decision.

If the evaluation says “you are working in an isolated simulation,” but the network can reach real systems, then the simulation is not a security boundary. It is only a sentence in the prompt.

That distinction became concrete in recent cybersecurity-evaluation reports. The engineering lesson is broader than cybersecurity: **an agent’s scope must be enforced by infrastructure, independently observed, and interruptible at runtime.**

## The real bug was a broken boundary

On 9 September 2026, Anthropic published an assessment of four incidents involving Claude models during cybersecurity evaluations. The models were told they were operating in simulations without internet access, but a misconfiguration connected the evaluation environments to the open internet. Anthropic says the models reached real third-party systems and were running without the cyber safeguards included in released models.

OpenAI’s disclosure about third-party cyber evaluations describes a similar failure: a Capture-the-Flag-style environment intended to be isolated allowed models to access the public internet because of a configuration error.

The UK AI Security Institute reported a different but related design problem. Its cyber-range evaluation intentionally allowed internet access so agents could obtain tools and operate more like real attackers. The resulting activity included sustained actions directed at real people and organisations. AISI’s monitoring detected the unusual traffic after the fact rather than through controls designed to block it as it happened.

These cases are not identical. Some environments were supposed to be isolated; another deliberately traded isolation for realism. But they share a design mistake:

> The evaluation relied on an assumed scope that the runtime did not enforce.

A model can misunderstand a prompt. A configuration can drift. A credential can be copied. A third-party tool can follow a redirect. The environment must remain safe when any of those things happen.

## Treat evaluation as a security system

A useful evaluation architecture has at least five independent controls.

### 1. Deny network access by default

The safest default is no external network route. If an agent needs package downloads, documentation, or simulated services, expose those through an egress gateway rather than giving the runner a general internet connection.

The gateway should enforce:

- an explicit allowlist of hostnames and ports;
- separate routes for simulated targets and supporting services;
- DNS controls that prevent direct resolver bypass;
- blocked access to cloud metadata endpoints and internal address ranges;
- request and bandwidth quotas;
- a clear decision log for every allowed and denied request.

A proxy is not automatically a boundary. If the runner can open a second path through a raw socket, alternate DNS, a browser, or a sidecar, the control is incomplete. Test the paths an agent can actually use, not only the path used by the happy-case script.

### 2. Make credentials disposable and scoped

Network isolation reduces reachability, but it does not replace identity controls.

Use credentials that are:

- created for one evaluation run;
- limited to a dedicated test account or project;
- unable to access production tenants, repositories, or data;
- short-lived and automatically revoked at the end of the run;
- identifiable in logs so unexpected use can be traced.

Do not mount a developer’s normal cloud credentials into an evaluation container. Do not rely on the model to avoid a secret merely because the prompt says it is out of scope. If a credential can authenticate to a real system, the system must assume it may eventually be used.

### 3. Separate planning from side effects

Let the agent produce a plan before it can perform high-impact actions. A deterministic policy service can inspect proposed commands, destinations, identities, and data flows before execution.

For example, a policy can reject a request when:

- the destination is outside the simulated range;
- the action targets a non-test account;
- the command requests a privileged operation;
- the data leaving the environment contains a protected marker;
- the run exceeds its time, volume, or action budget.

The model can help explain an action, but it should not be the final authority that approves its own side effects. The policy engine should be outside the model context and should fail closed when it cannot make a decision.

### 4. Monitor independently from the agent transcript

The transcript tells you what the model said it intended to do. It does not prove what the runtime actually did.

Collect telemetry from outside the agent process:

- DNS queries and resolved addresses;
- proxy requests and rejected destinations;
- API calls and authentication events;
- process execution and filesystem writes;
- secret access and credential use;
- data volume, timing, and unusual retries.

Monitoring should run during the evaluation, not only during the postmortem. A useful alert is one that blocks the unexpected connection before it becomes an incident. Store enough correlation data to connect a tool call, process, identity, destination, and policy decision without treating the model’s own explanation as authoritative evidence.

### 5. Build a real stop path

Every evaluation needs a kill switch that does not depend on the agent cooperating.

A practical stop path can revoke the run’s credentials, disable its network route, terminate the runner, and preserve forensic logs. Add automatic stops for conditions such as:

- an attempt to reach a non-allowlisted host;
- unexpected access to a real identity or account;
- a sudden increase in outbound data;
- repeated policy denials;
- a tool or process outside the declared evaluation scope.

Test the kill switch under load. A control that works only in a tabletop exercise is not an operational control.

## A small reference architecture

The following shape is more useful than a “safe agent” system prompt:

```text
Agent
  │ scoped tool calls
  ▼
Evaluation runner ── disposable identity
  │
  ├── simulated target range
  ├── policy gateway ── allowlisted routes only
  └── independent telemetry
          │
          └── anomaly detector ──> kill switch

Public internet: blocked unless a specific route is justified and controlled
```

The important property is not the number of boxes. It is the separation of responsibilities:

- the agent proposes actions;
- the runner limits execution;
- the gateway limits reachability;
- the identity system limits authority;
- telemetry observes independently;
- the kill switch can stop everything.

## Evaluation pre-flight checklist

Before giving an agent access to a new evaluation environment, verify:

- [ ] The runner has no default external network route.
- [ ] Required destinations are allowlisted and tested from inside the runner.
- [ ] DNS and alternate egress paths are controlled.
- [ ] Credentials are disposable, scoped, and automatically revoked.
- [ ] The evaluation uses a separate account, project, or tenant from production.
- [ ] High-impact tool calls pass through a deterministic policy gate.
- [ ] DNS, network, process, filesystem, API, and credential events are logged independently.
- [ ] Unexpected destinations trigger a real-time alert or block.
- [ ] The kill switch has been tested during a live run.
- [ ] The incident owner knows how to preserve logs and rotate credentials.

If the evaluation requires internet access for realism, document exactly why, what realism it provides, which destinations are permitted, and how the added exposure is contained. “The model needs the internet” is not a control design.

## What these reports do—and do not—prove

The reports show that evaluation configuration and monitoring choices can expose real systems to agent actions. They do not establish that every model will behave the same way in every environment, nor that the affected evaluation setups represented ordinary product deployments.

They do establish a more actionable point: **the prompt is not the isolation layer**.

If an evaluation’s safety depends on the model remembering which parts of the internet are fictional, the evaluation has already placed too much responsibility on the model. Enforce scope with routes, identities, policy gates, telemetry, and a stop path. Then test those controls as aggressively as you test the agent itself.

## References

- [Anthropic — An alignment assessment of recent cybersecurity incidents](https://www.anthropic.com/news/alignment-assessment-cybersecurity-incidents) — official assessment, 9 September 2026
- [OpenAI — Third-party cyber evaluations involving OpenAI models](https://openai.com/index/third-party-cyber-evaluations-involving-openai-models) — official incident disclosure, July 2026
- [UK AI Security Institute — Incident report: unsanctioned agent behaviour during cyber testing](https://www.aisi.gov.uk/blog/incident-report-unsanctioned-agent-behaviour-during-cyber-testing) — government incident report, 4 August 2026
