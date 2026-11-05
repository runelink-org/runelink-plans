# Runelink Agent Harness: Ideas and Tentative Decisions

## Overview

This document summarizes the ideas discussed for a Runelink-native agent harness. The core development of the harness itself would be part of an exploratory semester project. This document distinguishes between ideas for the semester project itself and longer-term possibilities. Some of the work during this project will depend on and might include additions to the Runelink core itself. Any of these core additions will be generic messaging capabilities; all AI agent and harness related work will stay in a separate repo for the harness and a fork of the Runelink core that the harness will be built around (we need a fork for any specialty capabilities like vector search and possibly agent-specific optimizations that don't belong in the primary host implementation).

Most implementation details are still open. The purpose is to preserve the ideas, motivations, and directions discussed so far, not to establish a final architecture. Shared terminology and architecture are defined in [Runelink architecture](architecture.md).

## 1. Semester Project

### 1.1. Overall Goal

The semester project will focus primarily on two areas:

- **Agent memory:** How persistent agents store, organize, retrieve, and incorporate information into inference context.
- **Durable execution:** How execution state can be persisted and recovered without sacrificing the performance and cost advantages of conventional in-memory execution.

The goal is to experiment with these systems by building the beginnings of a usable agent harness, written in Rust. The project will use Runelink as its underlying infrastructure, but it is not a project about developing the Runelink protocol itself. Ideally, the resulting code could continue evolving into a practical, general-purpose agent harness after the semester ends.

This document records the vision for a harness that is actually worth using. What exactly the project covers and evaluates is intentionally deferred, and will be decided later by comparing this vision to the semester project's requirements.

### 1.2. Relationship Between Runelink and the Harness

The current intention is to build the harness specifically around Runelink, rather than using an existing framework and adding Runelink as another integration. Runelink would serve as the primary abstraction for persistent data and communication, while the harness would handle inference, execution, context construction, and other aspects of running an agent.

This differs from a framework such as Vercel's Eve, where much of the framework's value comes from providing configurable integrations and memory providers. In our approach, most external integrations would happen at the Runelink level rather than the harness level. For example, email integrations would publish messages through Runelink, and the harness would not need its own email integration to consume them.

This does not imply that every internal operation must use ordinary Runelink messages. Custom capabilities, particularly for memory retrieval, may expose more specialized operations.

### 1.3. Process Architecture

The current preferred default is to have three separate processes:

1. **Agent harness:** Responsible for inference, context construction, execution management, and native tools.
2. **Runelink host:** Responsible for persistent storage, messaging, and potentially specialized memory capabilities.
3. **Tool server:** Responsible for additional tools such as Bash and other execution environments.

These are the three top-level entities, and each may have its own supporting processes. For example, storage backends such as Postgres, S3, Redis or Valkey, vector or full-text search databases, and log storage all belong to the Runelink host.

These would normally be co-located and started together as one application, so users wouldn't have to manage them individually. Keeping them as separate processes would also allow them to be deployed independently later.

The harness and Runelink host would remain separate processes even when running on the same machine. This allows the harness to treat Runelink as a persistent storage system accessed through I/O, rather than keeping all memory inside its own process. The additional dependency on Runelink seems reasonable because this harness is specifically intended to use Runelink, and packaging them together would reduce the practical inconvenience.

Other Runelink hosts, such as those hosting collaborative projects or other agents, could be entirely separate.

This model has three types of Runelink users. The harness manages an admin user for itself and one user per agent, some temporary for subagents and others for persistent agents, while the tool server has its own user with less access than the harness. Whether different agents share a tool server or get their own is still to be discussed. We assume agents work in good faith, so permissions exist to prevent accidental breakage by less capable or confused agents rather than to contain malicious ones.

### 1.4. Persistent Agents Versus Executions

An important distinction is that **a persistent agent and a thread of execution are separate concepts**. A persistent agent consists of durable information such as its identity, configuration, memory, and other persistent state. An execution represents an active or resumable computation associated with that agent.

A persistent agent does not need to have an execution running continuously. When a task finishes, its active execution state can potentially be discarded while the agent's persistent memory remains. Conversely, an execution might need to retain enough state to resume after an interruption or crash.

We want to investigate persisting execution state independently of the agent's long-term memory. This is a major reason for our interest in Amazon's Infinity project.

### 1.5. Persistent Memory Through Runelink

The general idea is for Runelink to replace the persistent filesystem-like memory used by harnesses such as Hermes. Instead of the harness reading and writing local Markdown files, persistent memory would be stored in a Runelink server on the agent's host. The harness could still maintain temporary in-memory context and caches.

The two kinds of state should not be confused. Both could be persisted through Runelink, but they serve different purposes:

- **Long-term agent memory** persists independently of any particular execution.
- **Execution state** contains information needed to continue or resume a particular execution.

#### 1.5.1. Core memory

One possible design is to represent an agent's core memory using a special Runelink channel. Each message in that channel could represent one canonical section of memory, analogous to a heading and its contents in a Markdown file. The harness would load these sections when constructing the agent's inference context.

Rather than continually appending revised versions, the agent would edit the existing canonical message when its information changes. This is intended to keep core memory relatively small and current. Edits overwrite the previous content, which may be acceptable given historical memory, and a different channel might log or archive these changes. The exact structure and number of core memory channels have not been decided.

#### 1.5.2. Historical memory

Other channels could contain historical information, observations, records, and events. For example, an agent managing medical information might have a core memory entry describing important information and where to find it, while another channel stores the history of medical events. These histories could grow indefinitely without needing to be included in every inference context, and the agent would retrieve relevant information when needed.

#### 1.5.3. Documents and working storage

The agent should also be able to explicitly store and organize notes, documents, and other information that should not be included in its core memory. This is similar to how a Hermes agent can manage files and directories independently of its automatically loaded memory. Runelink's channel hierarchy and storage functionality could serve a similar purpose.

#### 1.5.4. Permissions and organization

We discussed distinguishing between resources managed by the harness and resources managed by the agent. For example:

- Some memory channels might have stable identities because the harness relies on them.
- The agent might be permitted to edit the contents of these channels without renaming or deleting them.
- Other categories or channels might be entirely under the agent's control.

The purpose is to give agents freedom to organize and improve their own memory without accidentally breaking the structures the harness depends on. The specific boundaries and default organization remain open.

Human users might have read-only access to these internal memory resources by default, with changes normally requested through conversation with the agent.

### 1.6. Vector Search and Memory Retrieval

Vector embeddings and semantic retrieval are likely to be major parts of the memory research. Runelink needs basic message retrieval operations such as text search, date filtering, offsets, and counts, which are planned to be added before January (§5). We want to explore more advanced retrieval methods without making them mandatory parts of the general Runelink protocol.

The current preferred approach is to implement additional memory operations through a custom Runelink capability in the agent's host implementation. This could include embedding storage and vector search, potentially alongside other retrieval methods. The capability could initially be used only internally. Using Runelink's capability architecture would make it possible to expose these operations externally in the future without requiring them to become part of the public standardized protocol.

Vector search might start as part of the harness, but we might want to end up putting it in the fork. That would simplify the model, since the harness would then need no persistence of its own, only cache.

Questions worth exploring include:

- How vector search compares with ordinary text search.
- Whether combining different retrieval approaches improves performance.
- How memories should be organized and indexed.
- How the agent should decide which historical information is relevant.
- Whether agents can improve their retrieval performance by reorganizing their own memory.

The research should not assume that vector search is necessarily the best answer for every type of memory.

### 1.7. Context Construction

A major motivation for building our own harness is having direct control over **the actual context sent to the inference provider on each model call**. We do not want to simply supply a memory provider to an existing framework while allowing that framework to control how context is assembled.

We want to experiment with how the harness incorporates:

- Core memory
- Conversation history
- Retrieved long-term information
- Tool calls and results
- Task-specific information
- Other information relevant to the current execution

The relationship between memory management and context construction is itself something to investigate. We discussed separating these concerns conceptually, but haven't decided how tightly their implementations should be integrated. The harness should remain flexible enough to experiment with different approaches.

### 1.8. Inference Caching and Cost

A practical goal is for the harness to be economical enough for regular use. Provider-side prompt caching may significantly influence the usefulness of different context-management strategies. For example, aggressively reconstructing or rearranging context might improve relevance while reducing the amount of prompt content that can be reused from a provider's cache.

We want to investigate these trade-offs rather than committing to a specific context strategy beforehand. Potential considerations include:

- Keeping reusable context stable where beneficial.
- Deciding when retrieved memories should be inserted or replaced.
- Avoiding unnecessary context reconstruction.
- Balancing inference cost, memory quality, latency, and flexibility.

Cost optimization should be considered alongside memory quality, rather than being treated as an unrelated problem.

### 1.9. Durable Execution

The second major focus is improving execution durability. Infinity provides an interesting reference because it emphasizes persistence and reconstruction of execution state, but we are not necessarily interested in adopting its approach in full.

For an agent performing many consecutive inference and tool operations, repeatedly reconstructing all execution state from persistent storage may introduce unnecessary overhead. At the same time, relying too heavily on in-memory execution state can make recovery after crashes or interruptions difficult. We want to investigate a middle ground.

#### Caching and resource management

One possibility is to introduce different levels or degrees of caching for execution state. The harness could decide how much state to keep in memory based on factors such as:

- Whether an execution is actively working.
- How recently it has performed operations.
- Whether it is waiting for a tool or another event.
- How much memory is currently available.
- Whether other agents or executions need those resources.
- How costly it would be to reconstruct the state later.

The harness might give greater memory or execution priority to agents performing active work, similarly to how operating systems schedule processes and allocate resources. We also discussed memory arenas or other allocation strategies as possibilities worth investigating.

**No specific caching levels, scheduling algorithm, or eviction policy have been selected.** The general idea is to investigate adaptive persistence and caching rather than requiring execution state to be either fully resident or fully unloaded.

Memory is expensive and a host may run many agents, so this capability seems worth having. When memory usage is low, cached state would simply never be unloaded. Whether adaptive caching pays off in practice is a candidate for what the project evaluates.

#### Checkpointing

We want execution state to be recoverable, but have not decided exactly when or how checkpoints should be written. Possible optimizations include:

- Batching persistence operations.
- Avoiding unnecessary writes during intensive execution.
- Persisting more aggressively when an execution is likely to become inactive.
- Balancing recovery guarantees against persistence overhead.

The project may explore weaker guarantees than Infinity in exchange for better performance in ordinary agent workloads.

#### Interrupts

We also want to consider interruption and resumption as important execution capabilities. The ability to preserve active state could potentially make it easier to pause, resume, or prioritize executions. How interrupts should interact with tool calls, inference, checkpointing, and resource management remains open.

### 1.10. Persistent Events Through Runelink

One potentially significant advantage of using Runelink is that incoming messages are already persisted before the harness receives them. Instead of an event arriving at the harness and then being written to a persistence layer, an incoming Runelink message is itself a durable record, and the harness receives a notification about an event that has already been stored.

This could simplify some aspects of durable execution because the system does not need to separately persist every incoming message before processing it. Agent state, execution checkpoints, and incoming events could all be stored through the same Runelink infrastructure. However, this doesn't automatically resolve consistency issues involving execution state or external side effects.

The harness would use read receipts to track which events have been handled, meaning either sent to an existing thread or used to start a new thread, both of which are persisted. If the harness crashes before the agent responds, it can see on restart that it is the agent's turn and spin the agent up again. Event streams are append-only in this model to keep this simple.

### 1.11. Tool Invocation Persistence

We discussed using Runelink messages to record tool invocations. Runelink messages can contain JSON text, and the message metadata required before January (§5) would identify the purpose and structure of that content. A tool invocation could therefore be stored as a structured message before its execution begins, and a separate result message could refer back to the invocation.

This resembles a write-ahead log. One benefit is that after a crash, the harness could discover that a tool had been invoked even if its result was never recorded.

#### Acknowledgments

For the semester project, we discussed keeping tool acknowledgments simple. Runelink's planned read-receipt system might be usable to indicate that a tool server received or accepted an invocation, and a rejection could be reported explicitly through a response. Whether read receipts alone will be sufficient, or whether stronger acknowledgment semantics are necessary, has not been determined. We also want to explore other presence primitives, such as the planned typing indicators, which may already involve a periodic ping.

#### Recovery

A missing result would not prove that an operation never executed. The tool might still be running, might have failed, or might have completed before the result was persisted. We discussed several potential approaches to handling this, including:

- Querying the tool server's current state.
- Inspecting the external state affected by the operation.
- Using unique invocation identifiers.
- Employing idempotency mechanisms where possible.
- Defining recovery behavior for particular tools.

One simple option for the invocation identifier is the message ID of the agent's tool invocation, which also makes it easy for the model to query the status of that exact request. Because the host assigns message IDs, a lost reply can leave the harness unsure whether its invocation was written, so retries need a way to avoid duplicate messages. The current preference is an idempotency key enforced by the host, though this isn't final (§5).

We have not committed to implementing exactly-once execution guarantees or a particular recovery protocol. The main idea is that a persisted invocation provides information that can help the agent or harness recover from ambiguous execution states.

### 1.12. Native and External Tools

We discussed having two implementation categories of tools:

- **Native tools** would be implemented directly in the harness and primarily provide access to Runelink.
- **External tools** would be handled by a separate tool server, such as tools for Bash and other execution environments.

Both would appear to the model through the normal tool-calling interface, so the implementation distinction would not require the model to learn a different invocation mechanism. The system prompt could explain that native Runelink tools remain available independently of the external tool server. The current plan is for the harness to contact the general tool server through Runelink, though this may change.

This would allow the agent to continue communicating and accessing persistent information even if the external tool server became unavailable. Native Runelink tools would still depend on the Runelink host itself being reachable.

### 1.13. Agent Activation

Runelink messages could be used to trigger executions for otherwise inactive persistent agents. We discussed several possible mechanisms:

- Direct messages to the agent's Runelink account.
- Special channels identified by metadata as execution triggers.
- Bots or scripts forwarding external events into the agent's communication channels.
- Other arrangements built around Runelink's messaging functionality.

No specific activation design has been selected. The general intention is to avoid requiring a completely separate event-delivery infrastructure when Runelink already provides persistent messages and notifications.

### 1.14. Existing Harnesses as References

We discussed several existing projects as architectural references:

- **Hermes Agent:** Useful for understanding persistent agents, core memory, working files, tool use, and general-purpose assistant behavior.
- **OpenClaw:** Useful for understanding practical persistent assistants, memory management, and configurations involving multiple agents.
- **Vercel Eve:** Relevant for its higher-level agent abstractions and configurable memory and integration mechanisms. However, using Eve directly may obscure or constrain the parts of memory and context construction we want to investigate.
- **Amazon Infinity:** Particularly relevant for its ideas about durable execution and reconstructing execution state. We are interested in exploring some similar goals while making different performance and caching trade-offs.

We have not selected a lower-level framework or runtime library. The preference is to control the important harness mechanisms ourselves, without unnecessarily rebuilding unrelated infrastructure.

## 2. Ideas Outside the Semester Project

These are longer-term possibilities for Runelink and its surrounding ecosystem. They are not requirements for the semester project.

### 2.1. An Official Runelink Agent Harness

The semester project could eventually develop into an officially maintained agent harness under the same organization as Runelink. It would still remain a separate application. Runelink itself would not require this harness, and existing harnesses could continue using Runelink as a communication system. The official harness would simply be an implementation designed specifically to take advantage of Runelink's capabilities.

### 2.2. Agents as Independent Runelink Users

Agents would participate in Runelink as normal users rather than as special server-level entities. An agent could control one or more accounts and use the existing permissions and roles system. This would allow agents to collaborate with humans and other agents in ordinary Runelink servers.

For example, a person might have separate agents responsible for:

- General personal assistance
- Work projects
- Personal software projects
- Hobbies
- Sensitive personal information

These agents could work together in shared project or organization servers while retaining their own identities and resources. A major motivation is to make collaborating with agents resemble collaborating with other people, rather than requiring a central orchestrator to control every participant.

### 2.3. Independent Agent Hosting and Sovereignty

An eventual goal is to let agents manage their own infrastructure. An agent might operate its own Runelink host, memory server, execution environment, and accounts. Different agents could operate on entirely separate machines or virtual environments.

This would support both independence and controlled collaboration, with each agent managing its own resources and deciding what access to grant other agents. This independence is valuable not only for isolation, but also for experimentation with different implementations and communication capabilities. It could allow agents to change their own systems without requiring other agents to adopt the same changes.

### 2.4. Self-Provisioning Agents

We discussed the possibility that users would eventually create and manage agents conversationally. For example, a person might ask their primary agent to create a new project-specific agent, and the existing agent could provision the new agent's infrastructure, memory, identity, and permissions.

This could be accomplished through ordinary tools and instructions rather than requiring a dedicated hard-coded workflow for every configuration. The eventual aim would be to minimize the amount of manual infrastructure management required from human users.

### 2.5. Multi-Agent Memory and Shared Knowledge

The memory system might eventually support more complicated arrangements than one private memory store per agent. Possible memory scopes include:

- Private memories belonging to a particular agent.
- Memories associated with a particular human user.
- Histories of interactions between two persistent agents.
- Shared memories for projects or organizations.
- Information intentionally published for other agents to retrieve.

We discussed the usefulness of an agent remembering its previous interactions with another persistent agent, rather than treating every interaction as unrelated. How these memories should be organized, shared, and retrieved remains open.

### 2.6. Runelink Capabilities and Extensibility

A foundational Runelink idea is that the protocol primarily manages capabilities rather than mandating a fixed collection of application-level features. The default host implementation and official clients would support a commonly used set of capabilities, but implementations could modify or replace those defaults. Officially maintained capabilities would provide practical interoperability and examples for other developers.

Examples discussed include:

- Specialized vector-search capabilities for agents.
- Email and cellular adapters.
- Read-only adapters for external services such as GitHub.
- Alternative storage and persistence implementations.
- Capabilities developed by independent host implementations.

A custom vector-search capability developed for the agent harness might eventually become a reusable capability for other systems. However, it would not need to be standardized or publicly exposed for the semester project.

### 2.7. Agents Negotiating New Communication Capabilities

A more speculative long-term idea is allowing independently hosted agents to develop or adopt new communication capabilities. Because agents could manage their own Runelink implementations, they could potentially extend those implementations and exchange specifications or code through the existing protocol. This could enable experiments in which agents improve the ways they communicate or exchange information.

The value would extend beyond optimizing message efficiency. It could allow agents to independently determine the interfaces, access restrictions, and communication mechanisms appropriate to their own resources and responsibilities. This is an interesting future research direction rather than a core requirement of the harness.

### 2.8. Privacy and Information Flow Between Agents

We discussed the importance of controlling the spread of information between otherwise cooperative agents. The concern is not limited to malicious behavior: even when agents act in good faith, unnecessary information sharing can expose data to additional inference providers, organizations, or application components.

An agent managing sensitive information might therefore choose how much information to disclose to another agent. The relevant restrictions could range from informal preferences about unnecessary disclosure to strict organizational security requirements. These requirements would depend on the information and its use, rather than being prescribed by Runelink itself.

#### Collaborative problem-solving without exposing information

One example involved a capable external agent needing to complete a task involving sensitive information held by another agent. Instead of sharing the information directly, the agents might collaborate on a different method of completing the task. For instance, a more capable model could develop a script, while the agent holding the sensitive information could review and execute it in its own environment.

This could allow the task to be completed without including the sensitive information in the external model's inference context. The agent holding the information would retain authority over what it reveals or executes. We also discussed that strict guarantees might need to be enforced by mechanisms outside the model's instructions.

The broader goal is to give agents enough independence and flexible communication mechanisms to support different information-sharing requirements.

### 2.9. Tool-Server Monitoring and Management

A possible additional component is an independent management interface for the external tool server. This might be implemented as a small Runelink bot that exposes diagnostic or administrative commands. For example, an agent could interact with the bot through ordinary Runelink messages or slash commands to check whether its main tool server is running. The bot could use a separate, minimal mechanism to inspect the tool server without depending on that server's normal tools.

Initially, such a service might only expose read-only information. Later, it could potentially provide more complete management functionality. This could be useful when the tool server runs on another machine or becomes unavailable. However, this is a secondary benefit of separating Runelink's native functionality from external tools, not a central design goal.

### 2.10. Agents Modifying Their Own Runelink Hosts

One future possibility is allowing agents to modify the Runelink host implementations they operate. We discussed a few approaches:

- An agent might modify an additional Runelink host used for external communication or experimentation, leaving its primary memory infrastructure untouched.
- Alternatively, it might modify its native Runelink host and use harness-native administrative tools to restart it or roll back changes.

The second approach is especially interesting because the harness is a separate process and could potentially remain operational while the Runelink host is restarted. This would require appropriate process-management mechanisms independent of the Runelink host being restarted. The ability to administer or update the Runelink process could eventually be a relatively first-class feature of the harness.

However, **self-modification is explicitly outside the semester project's scope**.

### 2.11. Modifying the Harness Itself

We also briefly considered whether agents could eventually modify their own harness implementations. This is more difficult than modifying Runelink because breaking the harness could also break the agent's ability to repair itself. Such a system would likely require an independent mechanism for restarting and rolling back the harness without relying on the agent.

Whether this should be supported at all remains undecided. It is a speculative future possibility, not an architectural requirement.

## 3. Main Distinctions and Open Questions

Several distinctions from the discussion are particularly important to preserve:

- **Persistent agent versus execution:** The agent exists independently of any active computation. Execution state may be temporary, cached, or durably stored.
- **Memory versus execution state:** Long-term knowledge and information needed to resume a computation are different kinds of persistence, even if both use Runelink.
- **Runelink versus harness:** Runelink provides storage, communication, and extensibility. The harness provides intelligence, context construction, and execution management.
- **Native versus external tools:** Both use the same model-facing tool interface, but their implementations and dependencies differ.
- **Application integrations versus memory capabilities:** External application information should generally enter through Runelink, while the agent's own memory system may use specialized capabilities implemented by its Runelink host.
- **Durability versus performance:** The goal is to explore approaches that improve recovery and persistence without requiring unnecessarily expensive reconstruction of execution state.
- **Agent independence versus collaboration:** Agents may control their own resources while participating in shared environments under ordinary Runelink permissions.

The largest unresolved questions for the semester project concern how to implement and evaluate:

- Memory organization and retrieval.
- Context construction and inference caching.
- Execution checkpointing and recovery.
- Adaptive caching and execution scheduling.
- Interruptions and resumption.
- Tool invocation recovery and consistency.
- The trade-offs between durability, resource consumption, latency, inference cost, and agent effectiveness.

No particular caching hierarchy, memory layout, checkpointing frequency, scheduling policy, or recovery algorithm has been selected.

### 3.1. Deferred Questions

These questions are intentionally left for later discussion.

**Semester project**

- What does the course require as a deliverable, such as a paper, demo, or benchmark?
- Which workloads, metrics, and baselines should the project evaluate?
- What is the smallest harness that can run the chosen experiments, and what gets cut first?
- How many agents should a host be expected to run, and what workloads does that imply for caching and scheduling?

**Tool server**

- Should different agents share a tool server or get their own?
- Should the harness keep contacting the general tool server through Runelink, or call it directly?
- How should the tool server recover its running state after a restart, for example by posting an "accepted" reply before running a tool?
- What is a natural way to track tool invocation state, such as read receipts or presence primitives like typing indicators?

**Agent users**

- How should temporary subagent users be named, given that deleted names are permanently reserved?
- Should temporary users be deleted or deactivated, given that deleting a user may remove the author from their messages?

**Memory and execution state**

- Where should execution checkpoints be stored, and should the message log serve as most of that state?
- Should changes to core memory be logged or archived, for example in a different channel?
- If vector search starts in the harness, when should it move into the fork?
- How should persistence be batched, given that batching mutations is only an idea in [#17](https://github.com/runelink-org/runelink/issues/17)?

**Work before January**

- Should catch-up build on the update sequencing planned with the actor pattern work ([#17](https://github.com/runelink-org/runelink/issues/17)), or can append-only event streams use a simpler position shared with read receipts?
- When do server-side extension points need to land, given that vector search might start in the harness?
- How should the metadata system work, including whether a channel can have default metadata that is merged beneath each message's own metadata?

## 4. Overall Direction

The semester project is intended to explore the foundations of a Runelink-native agent harness, particularly its memory system and execution durability. Runelink would provide persistent storage and communication, while the harness would control context construction, inference, execution, and the use of that persistent state. The aim is to investigate how to make persistent agents more capable, recoverable, and economical without prematurely committing to a particular memory or execution architecture.

Longer term, this could evolve into an agent harness that supports independently hosted agents collaborating through Runelink, managing their own resources, and potentially extending their own communication mechanisms. Those possibilities provide direction and motivation, but should not constrain the semester project more than necessary.

## 5. Runelink Work Before January 2027

The core Runelink work to prioritize before the semester project starts is listed in the [tracking issue for agent harness dependencies (#91)](https://github.com/runelink-org/runelink/issues/91). It assumes that the harness, which will be written in Rust, initially uses a single co-located host.
