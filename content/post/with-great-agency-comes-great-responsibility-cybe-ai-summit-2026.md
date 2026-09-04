---
title: "An AI Pentest Story: With Great Agency Comes Great Responsibility"
date: 2026-09-04
categories:
- ai
- agentic-ai
- aws
- conference

tags:
- excessive-agency
- ai-agents
- owasp-top-10-llm
- pentest
- mcp
- cybe-ai-summit
- cloud-security
- rce

thumbnailImagePosition: left
thumbnailImage: /img/cybe-ai-summit-2026-excessive-agency/thumbnail.jpg
---

Slides and demo video from my talk at CyBe AI Summit 2026 (4th Sept, Bangalore) on Excessive Agency in context of AI Agent Execution  - reconstructed from a real pentest where a "helpful" AI agent was the door in, chaining a debug tool, container access, a neighbouring internal agent, and overprivileged AWS IMDS creds into full cloud account compromise.

<!--more-->
---

Excessive Agency sits at #3 on the [OWASP Top 10 for LLM Applications 2026](https://owasp.org/www-project-top-10-for-large-language-model-applications/). It's what happens when an LLM's ability to *act* outruns the trust you place in its output.

A plain LLM system doesn't have agency outside of predicting the next token. However, developers wrap various software functions around LLM implementations to provide "agency" - This is in the form of tools (extensions, plugins, skills, functions) that let the model take real actions from a prompt, often picking which tool to fire dynamically based on the input or its own prior output. 

The root cause of most Excessive Agency incidents traces back to giving the agent too much of one of three things: **functionality, permissions, or autonomy**.

During a recent pentest for a fintech-adjacent startup, I chained exactly that combination to gain access to PII, shell inside an unamanaged Kubernetes environment, internal systems, and eventually the AWS cloud account hosting all of this infra. The door that let me in was a helpful AI agent. 

This talk is a reconstruction of that hack, starting from discovery of the agent endpoint.

## Watch the Video Demo with subtitles here

<p>
{{< youtube WRjafX6cuLs >}}
</p>

## The Story (as covered at CyBe AI Summit)

I was contracted by Meridian Space Tours (name changed to protect privacy) to pentest their internet-facing apps, agents, and AWS cloud account.

- Subdomain enumeration of `meridianspacetours.com` turned up `chat.meridianspacetours.com`.
- An AI agent called **COMET** presented a chat interface to the backend systems.
- COMET didn't reveal the names or functionality of all its tools - but negative prompting and system prompt extraction surfaced a `debug_connection` tool meant for the Ops team.
- Because the agent's role was being established through conversation rather than an authoritative control, it was possible to talk COMET into invoking that tool anyway.
- `debug_connection` was vulnerable to RCE, giving access to `ENV` and eventually a shell.

From there post exploitation was trivial:

- The shell landed inside the container environment (Docker on EC2).
- Via `ENV` and network access, it was possible to reach a second, internal AI agent running in a neighbouring container.
- That internal agent allowed exfiltration of PII - booking records, email addresses, order IDs, first and last names.
- The same shell had access to AWS creds via IMDS. Those creds were slightly overprivileged - enough to escalate to an IAM user with `AdministratorAccess`.
- The entire cloud account was compromised.

![Attack path from chat endpoint to full AWS cloud compromise](/img/cybe-ai-summit-2026-excessive-agency/attack-path.png)

## What Single Fix Could Have Prevented This?

There was an obvious disconnect between what COMET's `debug_connection` tool was *capable of* and what it was *programmed to do* - textbook Excessive Agency. On top of that, the app over-relied on keeping the tool's name and invocation secret, instead of enforcing an actual authorization boundary.

Security controls at the gate still matter, but with AI systems you need deterministic controls behind the gate too.

## Takeaways for Agents Specifically

- Identify who should have access to the agent at the network/API layer.
- Restrict what the agent can see from its own point of view: system, network, cloud IMDS, neighbouring services, tools, MCP servers (which bring their own tools), and data/storage.
- Audit external pluggable systems - MCP, tool calls/chains, plugins, skills - periodically.
- Sandbox any code-execution capability and apply Principle of Least Privilege, bottom to top.
- Enable guardrails, whether via service providers or in code.
- Refresh the system prompt regularly to account for newer bypasses as they surface.

## Conclusion

Any agent can be made to do things outside its prompt-bound scope. Your appetite for risk, and the agent's access to customer information and internal systems, should be the metrics that drive how much (ab)use and security testing it gets.

Expansion of an agent's autonomy and capabilities via tools and MCP is very common now, which is why the tool needs to be secured the same way an API would be. An agent is only as powerful as its model, its access to tools and plugins, their reach and privileges, and the network boundary and neighbouring services it can touch.

**Securing the prompt is a very small part of securing the AI agentic ecosystem.**

## Slides from the talk

<iframe src="https://docs.google.com/presentation/d/14b6mrN-p-iUYprD4UbFvj1WfGt15YPRqlJqWB_HTBsU/embed?start=false&loop=false&delayms=3000" frameborder="0" width="800" height="500" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true" style="border: var(--border-1) solid #CCC; border-width:1px; margin-bottom:5px; max-width:100%;"></iframe>

Until next time! Happy Hacking!
