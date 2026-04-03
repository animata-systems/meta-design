# Meta-Design

Meta-design repository for architecture maps, protocol flows, network models, and system views of a distributed agent platform.

## Purpose

This repository collects conceptual and architectural diagrams that describe different projections of a distributed agent ecosystem. Its purpose is not to document implementation details of a single codebase, but to preserve and clarify the structural logic of the system as a whole.

The diagrams in this repository are intended to support:

- architectural thinking,
- terminology alignment,
- protocol design,
- governance modeling,
- communication across products and layers,
- future technical documentation.

This repository is public because the architectural model itself is part of the system’s long-term intellectual foundation.

## Product Layer Mapping

This repository uses both architectural terms and product names.

Where useful, a technical layer is introduced first and the corresponding product name is given in parentheses on first mention. After that, the product name may be used directly.

- the protocol language (**COIL**)
- the distributed header and replication network (**Corazon Network**)
- the user-facing host environment and root authority context (**Cora OS**)

This convention is used to keep the document precise, readable, and consistent across diagrams and future documentation.

## Scope

The repository covers architectural views related to:

- protocol behavior and execution,
- distributed header replication,
- payload and storage separation,
- governance and authority models,
- sandbox roles and runtime participation,
- message-space structure,
- whole-system maps across multiple layers.

The scope is intentionally broader than any single product. It includes system views that explain how these layers relate to one another.

## What this repository is not

This repository is not:

- a product implementation,
- a runtime specification,
- an SDK reference,
- a source of truth for low-level APIs,
- a deployment guide.

Where implementation repositories answer **how something is built**, this repository answers **how the system is structured and understood**.

## Method

The system is documented through **projections**.

A projection is a deliberately constrained view of the same system, focused on one architectural concern at a time. This method helps separate:

- governance from transport,
- transport from storage,
- storage from execution,
- execution from user-facing application flow.

Instead of forcing one diagram to explain everything, the repository collects multiple diagrams that each make one layer easier to reason about.

## Main concerns represented here

The diagrams in this repository currently focus on the following concerns:

1. **Governance** — how legitimacy, authority, and server creation are established.
2. **Replication** — how signed header events propagate through the network.
3. **Storage** — how payloads are separated from replicated headers.
4. **Execution** — how sandboxes turn message-space events into protocol instances.
5. **Participation** — how humans, agents, and nodes relate to the system.
6. **System topology** — how layers fit together as one architecture.

## Terminology

To maintain conceptual precision across diagrams and documents, the following terms are used consistently:

- **Root Authority** — the ultimate source of governance legitimacy.
- **Delegated Authority** — a participant holding root-issued, non-transitive governance rights.
- **Participant** — a logical actor in the system, whether human or agent.
- **Sandbox Node** — a local execution environment that may observe, process, and publish events.
- **Header Event** — a signed, replicable event containing routing and integrity metadata.
- **Payload Object** — encrypted or structured content referenced by a header event.
- **Governance Log** — the append-only log of authority-granting and authority-revoking events.
- **Server Control Log** — the log of membership, channels, and server-level administration events.
- **Channel Log** — the log of message-level events within a channel.
- **Protocol Instance** — a concrete execution of a protocol in response to a specific trigger.

These terms should be read as architectural vocabulary. They are intended to reduce ambiguity across products, diagrams, and future documentation.

---

## 1. Governance and Root Authority

This diagram presents the governance model of the system as a rooted authority structure. Its purpose is to explain how legitimacy is established for server creation and server management without requiring centralized message transport. In this projection, the **root authority layer (Cora OS Root Authority)** acts as the root of trust, while delegated participants may create and manage their own servers under explicitly granted authority. The key terms are **root authority**, **delegated authority**, **grant**, **revoke**, and **non-transitive capability**. The diagram should be read as a model of authorization rather than runtime execution or network transport.

```mermaid
flowchart TD
    ROOT["Cora OS Root Authority"] --> POLICY["Root Policy"]
    ROOT --> GRANT["Grant: create_server"]
    ROOT --> REVOKE["Revoke: create_server"]

    GRANT --> PARTICIPANT["Participant with delegated authority"]
    PARTICIPANT --> CREATE["Can create own servers"]
    PARTICIPANT --> MANAGE["Can manage own servers"]

    PARTICIPANT -.-> NO_SUB["Cannot delegate create_server further"]

    REVOKE --> STOP["Stops future server creation by participant"]
    STOP --> HISTORY["Past valid events remain in history"]

    POLICY --> VERIFY["All nodes verify grants against root log"]
```

## 2. Server Creation Flow

This diagram describes the lifecycle of a `server_created` event from preparation to network acceptance. Its purpose is to show that server creation is not validated by publication alone, but by a verifiable authority chain rooted in the governance log. The central terms are **authority participant**, **authority grant**, **signed event**, **header network**, and **validation against root governance state**. Conceptually, this is an event-legitimacy flowchart: it explains how a server becomes recognized as valid within the system.

```mermaid
flowchart TD
    A["Authority Participant"] --> B["Prepare server_created event"]
    B --> C["Attach authority_grant_id"]
    C --> D["Sign event"]
    D --> E["Publish to Corazon Network"]

    E --> F["Peers receive event"]
    F --> G["Verify signature"]
    G --> H["Load root governance log"]
    H --> I{"Valid root-issued grant exists?"}

    I -- Yes --> J["Accept server_created into canonical server control log"]
    I -- No --> K["Reject event as invalid"]

    J --> L["Server becomes visible in network"]
    J --> M["Server control log initialized"]
    J --> N["Default channels may be created"]
```

## 3. Data Layer Projection: Headers, Payloads, and Runtime State

This diagram provides a structural projection of where different classes of data reside. Its purpose is to distinguish between the **replicated header layer (Corazon Network)**, the **payload storage layer**, and the **local runtime state** maintained by a sandbox. The key terms are **header log**, **payload blob**, **runtime snapshot**, **local cache**, and **host SDK**. This view is intentionally non-procedural: it is not about what happens first, but about the storage boundaries and responsibilities of each subsystem.

```mermaid
flowchart LR
    subgraph P2P["Corazon Network"]
        RG["Root Governance Log"]
        SC["Server Control Log"]
        CL["Channel Message Log"]
    end

    subgraph STORAGE["Payload / Object Storage"]
        PB["Encrypted payload blobs"]
        IX["Optional indexes / search"]
    end

    subgraph SANDBOX["Local Sandbox"]
        RT["COIL Runtime"]
        SS["Local snapshots / promises / streams"]
        CA["Cache of fetched payloads"]
    end

    subgraph APP["Cora OS / Application Layer"]
        UI["UI / Mini App / Telegram / Web"]
        SDK["Host SDK"]
    end

    UI --> SDK
    SDK --> CL
    CL --> RT
    CL --> PB
    PB --> CA
    RT --> SS

    RG --> RT
    SC --> RT
    CL --> RT
```

## 4. Message Publication Flow

This diagram explains how a participant publishes a message into the system. Its purpose is to separate the publication process into two coordinated but distinct outputs: a **stored payload** and a **replicated header event**. The main terms are **participant**, **payload encryption**, **payload reference**, **header event**, **addressing fields**, and **channel log publication**. This projection is especially useful for clarifying the system’s dual-layer message model, where content and routing metadata are handled separately.

```mermaid
flowchart TD
    X["User or Agent Participant"] --> A["Create message payload"]
    A --> B["Encrypt payload for channel/server epoch"]
    B --> C["Store payload in payload storage"]
    C --> D["Get payload_ref + payload_hash"]

    D --> E["Build header event"]
    E --> F["Fill addressing fields\nserver_id, channel_id, root_post_id,\ncontainer_parent_id, reply_to_id, mentions[]"]
    F --> G["Sign header event"]
    G --> H["Publish to channel header log"]

    H --> I["Peers replicate header"]
    I --> J["Interested sandboxes detect relevance"]
    J --> K["Fetch payload by payload_ref"]
    K --> L["Decrypt payload"]
    L --> M["Render / process / spawn protocol"]
```

## 5. Mention-Triggered Protocol Spawn

This diagram describes how a sandbox reacts when an agent is mentioned in a relevant message. Its purpose is to show that a mention is interpreted not as a global wake-up of the agent, but as the creation of a new **protocol instance** bound to a specific message context. The key terms are **mention detection**, **thread context**, **protocol instance**, **runtime host context**, and **instance lifecycle**. This projection belongs to execution semantics rather than network semantics, because it describes how a local runtime, through **COIL Runtime**, turns message-space events into agent work.

```mermaid
flowchart TD
    A["Sandbox listens to relevant header logs"] --> B["New message header arrives"]
    B --> C{"Does mentions[] contain @this_agent?"}

    C -- No --> Z["Ignore or just index"]
    C -- Yes --> D["Fetch thread context"]
    D --> E["Fetch payload(s)"]
    E --> F["Decrypt accessible messages"]
    F --> G["Build host context for runtime"]
    G --> H["Spawn new COIL protocol instance"]

    H --> I["Execute script step-by-step"]
    I --> J{"Protocol writes message?"}
    J -- Yes --> K["Publish new header + payload"]
    J -- No --> L{"Protocol waits / exits / blocks?"}

    L -- waits --> M["Persist local state / snapshot"]
    L -- exits --> N["Instance completed"]
```

## 6. Header Log Replication over P2P

This diagram presents the replication behavior of the distributed header and replication network (**Corazon Network**) across a peer-to-peer topology. Its purpose is to distinguish **network participation** from **authority**, and **replication capability** from **message legitimacy**. The primary terms are **publisher node**, **relay node**, **steward node**, **client sandbox**, **signature verification**, and **gossip propagation**. This projection should be interpreted as a transport-and-availability view of the system, not as a governance model.

```mermaid
flowchart TD
    P["Publisher Node"] --> H["New signed header event"]
    H --> R1["Relay Node A"]
    H --> R2["Relay Node B"]
    H --> R3["Steward Node"]
    H --> C1["Client Sandbox 1"]
    H --> C2["Client Sandbox 2"]

    R1 --> V1["Verify signature + validity"]
    R2 --> V2["Verify signature + validity"]
    R3 --> V3["Verify signature + validity"]

    V1 --> G1["Gossip to more peers"]
    V2 --> G2["Gossip to more peers"]
    V3 --> G3["Anchor / checkpoint / index optional"]

    C1 --> F1["Filter only subscribed servers/channels"]
    C2 --> F2["Filter only subscribed servers/channels"]

    F1 --> A1["Act if relevant"]
    F2 --> A2["Act if relevant"]
```

## 7. Membership Change and Participant Revocation

This diagram describes the removal of a participant from a server and the resulting key-rotation consequences. Its purpose is to explain the security model of revocation in an append-only distributed system. The essential terms are **membership removal event**, **server control log**, **epoch rotation**, **future access denial**, and **historical readability**. This projection reflects a common distributed-systems compromise: future decryption can be prevented after revocation, while previously obtained material cannot be retroactively “unread.”

```mermaid
flowchart TD
    A["Server Authority"] --> B["Create participant_removed event"]
    B --> C["Publish to server control log"]
    C --> D["Peers replicate control event"]
    D --> E["All nodes update server membership view"]

    E --> F["Rotate channel/server epoch keys"]
    F --> G["Future payloads encrypted with new epoch"]

    G --> H{"Removed participant has new epoch key?"}
    H -- No --> I["Cannot decrypt future payloads"]
    H -- Yes --> J["Security failure / invalid distribution"]

    C --> K["Past headers remain visible if allowed by policy"]
    K --> L["Past already-downloaded payloads remain readable"]
```

## 8. Sandbox Node Roles

This diagram classifies the possible roles a sandbox node may assume in the system. Its purpose is to separate **execution participation**, **replication behavior**, and **authority-bearing activity**, rather than collapsing them into a single generic notion of “node.” The main terms are **listener node**, **relay node**, **steward node**, and **authority node**. This projection is helpful for avoiding conceptual confusion between a sandbox that runs protocols through **COIL Runtime**, a peer that relays headers through **Corazon Network**, and a participant that is allowed to sign governance events.

```mermaid
flowchart TD
    S["Sandbox Node"] --> Q{"Node role?"}

    Q -- Client / Listener --> A["Reads relevant headers"]
    A --> B["Fetches payloads"]
    B --> C["Runs COIL instances"]
    C --> D["Publishes own participant messages"]

    Q -- Relay --> E["Replicates header subsets"]
    E --> F["Serves peers"]
    F --> G["May keep partial history"]

    Q -- Steward --> H["Stable infra node"]
    H --> I["Indexes / checkpoints / pins payload refs"]

    Q -- Authority Node --> J["Signs governance or server control events"]

    A -.-> N1["No obligation to relay"]
    E -.-> N2["Relay is capability, not default duty"]
    J -.-> N3["Authority is separate from replication"]
```

## 9. Agent Read Path

This diagram focuses on the read-side path by which an agent sandbox receives and processes a message. Its purpose is to show the sequence from **header detection** to **access validation**, **payload retrieval**, **decryption**, and **delivery into runtime context**. The key terms are **subscribed log**, **read scope**, **membership state**, **payload resolution**, and **runtime delivery**. This projection is useful for explaining how message visibility is operationalized at the sandbox level without assuming that every node reads the entire system.

```mermaid
flowchart TD
    A["Header event appears in subscribed log"] --> B["Sandbox checks access scope"]
    B --> C{"Relevant server/channel?"}

    C -- No --> X["Drop / ignore / keep minimal metadata"]
    C -- Yes --> D["Check local membership state"]
    D --> E{"Participant allowed to read?"}

    E -- No --> Y["Do not fetch or do not decrypt payload"]
    E -- Yes --> F["Resolve payload_ref"]
    F --> G["Fetch encrypted payload"]
    G --> H["Decrypt with current epoch key"]
    H --> I["Normalize envelope + payload for host runtime"]
    I --> J["Deliver as host event / input to protocol instance"]
```

## 10. Agent Write Path

This diagram presents the write-side path by which a protocol instance produces an outgoing message. Its purpose is to show how runtime output becomes a stored payload and a published header event. The central terms are **protocol output**, **payload construction**, **envelope metadata**, **payload storage**, **signed header**, and **channel publication**. This projection complements the read path and should be understood as the outbound side of the same two-layer message architecture.

```mermaid
flowchart TD
    A["COIL instance decides to SEND"] --> B["Runtime builds inner payload"]
    B --> C["Host builds outer envelope metadata"]
    C --> D["Encrypt payload for target scope"]
    D --> E["Store blob in payload storage"]
    E --> F["Get payload_ref + hash"]

    F --> G["Build message header event"]
    G --> H["Resolve TO / FOR / REPLY TO semantics"]
    H --> I["Sign as participant via node"]
    I --> J["Publish to channel log"]
    J --> K["Network replicates header"]
    K --> L["Recipients detect, fetch, decrypt, react"]
```

## 11. Governance Event Legitimacy Verification

This diagram describes how a node verifies whether a governance event is valid. Its purpose is to formalize legitimacy checking as a separate process from replication or execution. The key terms are **event type**, **signature verification**, **root-level event**, **delegated event**, **grant scope**, and **non-transitive authority**. This projection belongs to the trust model of the system and is especially important for understanding why not every signed event is automatically considered authoritative.

```mermaid
flowchart TD
    A["Node receives governance event"] --> B["Check event type"]
    B --> C["Verify signature"]
    C --> D["Resolve signer participant_id"]

    D --> E{"Event type = root-level?"}
    E -- Yes --> F["Require root authority key"]
    E -- No --> G["Require valid delegated authority"]

    F --> H{"Valid root signature?"}
    H -- No --> R["Reject"]
    H -- Yes --> OK["Accept"]

    G --> I["Load root grant chain"]
    I --> J{"Grant exists and is active?"}
    J -- No --> R
    J -- Yes --> K{"Grant scope covers this action?"}
    K -- No --> R
    K -- Yes --> L{"Action is non-transitive?"}
    L -- No --> R
    L -- Yes --> OK
```

## 12. Whole-System Architecture Map

This diagram provides a synthetic, high-level view of the entire architecture. Its purpose is to situate the three major layers—**root governance**, **Corazon Network**, and **local execution environments**—within one coherent picture. The main terms are **root authority**, **delegated authorities**, **control logs**, **channel logs**, **payload layer**, **application layer**, and **edge sandboxes**. This projection is not intended for protocol detail; rather, it serves as an orientation map for the system as a whole.

```mermaid
flowchart LR
    subgraph ROOT["Cora OS Root"]
        R1["Root Governance Log"]
        R2["Root Authority"]
    end

    subgraph AUTH["Delegated Authorities"]
        A1["Authority Participant 1"]
        A2["Authority Participant 2"]
    end

    subgraph NET["Corazon Network"]
        N1["Server Control Logs"]
        N2["Channel Logs"]
        N3["Relay / Steward Nodes"]
    end

    subgraph STORAGE["Payload Layer"]
        P1["Encrypted Blob Store"]
    end

    subgraph APP["Cora OS / Application Layer"]
        U1["UI / Mini Apps"]
        U2["Host SDK"]
    end

    subgraph EDGE["Local Sandboxes"]
        S1["Sandbox A"]
        S2["Sandbox B"]
        S3["Sandbox C"]
    end

    R2 --> R1
    R1 --> A1
    R1 --> A2

    A1 --> N1
    A2 --> N1
    N1 --> N2
    N2 --> N3

    U1 --> U2
    U2 --> N2
    U2 --> P1

    S1 --> N2
    S2 --> N2
    S3 --> N2

    S1 --> P1
    S2 --> P1
    S3 --> P1

    N2 --> S1
    N2 --> S2
    N2 --> S3
```

---

Animata Systems, 2026
