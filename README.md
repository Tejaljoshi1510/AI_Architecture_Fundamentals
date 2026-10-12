# AI Architecture Fundamentals: RAG, MCP and Client AI Guardrails

**Scope:** LLM fundamentals, Retrieval-Augmented Generation (RAG), Model Context Protocol (MCP), and secure AI usage in client projects.

## 1. Introduction

Modern AI applications combine Large Language Models (LLMs) with external knowledge sources, tools, and security controls. Understanding these components is essential before integrating AI into enterprise environments.

An LLM generates responses based on learned patterns. However, it may lack current or private organizational information, generate incorrect answers, or misinterpret untrusted input. RAG addresses knowledge retrieval, MCP standardizes interaction with external systems, and AI guardrails help manage security, privacy, and operational risks.

These technologies solve different problems and can be combined to build controlled, enterprise-grade AI applications.

## 2. LLM Architecture Fundamentals

### 2.1 Tokenization and Context Window

**Tokenization** converts input text into tokens, which may represent whole words, parts of words, punctuation, or other text units. The model processes these tokens rather than directly processing human-readable sentences.

A **context window** is the amount of tokenized information a model can consider in a single interaction. It can include instructions, conversation history, retrieved documents, and tool results, as well as space needed for generated output, depending on the model.

**Engineering considerations:**

* Large prompts consume more context and can increase latency and cost.
* Excessive conversation history or retrieved documents may push important information out of the usable context.
* Context limits and token accounting vary by model.
* Relevant information should be selected and prioritized rather than indiscriminately sending all available documents.

### 2.2 Training vs. Inference

* **Training:** The model learns statistical patterns from training data by adjusting its parameters through an optimization process.
* **Inference:** The trained model processes a new input and generates an output using its learned parameters and the supplied context.

Training an LLM on general data does not automatically give it access to a client's private documents, live databases, or current infrastructure state. Such access requires an explicitly designed integration, such as RAG or approved tool calls.

### 2.3 Hallucinations and Failure Vectors

A hallucination occurs when a model generates information that appears plausible but is incorrect, unsupported, or fabricated.

Common causes include:

* Missing or ambiguous information in the prompt.
* Outdated or incomplete model knowledge.
* Weak retrieval results or irrelevant source documents.
* Conflicting evidence in the supplied context.
* The model presenting an inference as an established fact.

Hallucinations cannot be eliminated simply by writing a better prompt or adding RAG. Risk can be reduced through reliable sources, evidence-based responses, output validation, evaluation, and human review for high-impact decisions.

## 3. Retrieval-Augmented Generation (RAG)

### 3.1 What is RAG-

RAG is an architecture that retrieves relevant information from an external knowledge source and supplies it to an LLM as context when generating a response.

It is particularly useful when an organization needs an AI assistant to answer questions using internal documentation, technical runbooks, policies, or product specifications without relying exclusively on the model's training knowledge.

### 3.2 RAG Architecture and Workflow

RAG has two main phases: **document ingestion** and **query-time retrieval and generation**.

**A. Document ingestion (offline or incremental process)**

1. **Data ingestion:** Collect approved documents from sources such as knowledge bases, PDFs, technical documentation, or internal repositories.
2. **Parsing and chunking:** Extract text and divide documents into manageable, meaningful sections. Chunk size and overlap influence retrieval quality.
3. **Embedding generation:** Convert each chunk into a numerical vector representing its semantic characteristics.
4. **Indexing:** Store vectors alongside the original text or a reference to it, plus metadata such as document ID, version, access permissions, and source.
5. **Index maintenance:** Update or remove indexed content when the source changes, and maintain appropriate access controls.

**B. Query-time retrieval and generation**

1. The user submits a question.
2. The application authenticates the user and applies relevant access restrictions.
3. The question is converted into an embedding, where vector-based retrieval is used.
4. The retrieval system identifies relevant chunks using similarity search, metadata filters, keyword search, or a combination of these techniques.
5. Optionally, a reranker reorders the candidate results to prioritize the most relevant evidence.
6. The application builds a prompt containing the question, selected evidence, and instructions.
7. The LLM generates an answer based on the supplied context.
8. The application returns the answer and, where supported, source references. Validation or human approval may be required before consequential actions.

### 3.3 Core RAG Components

* **Embeddings:** Numerical representations used to compare semantic similarity between text or other supported data.
* **Vector database:** Stores and searches embeddings, often with metadata and associated document references.
* **Semantic search:** Retrieves conceptually related content even when the wording differs from the query.
* **Chunking:** Controls the granularity of information retrieved. Chunks that are too small may lose context; chunks that are too large may contain irrelevant information.
* **Reranking:** Uses a more targeted relevance assessment to improve the ordering of retrieved candidates.
* **Grounding:** Supplies supporting evidence to the LLM so responses can be based on retrieved information rather than unsupported assumptions.

### 3.4 Production Benefits and Limitations

**Benefits**

* Provides access to changing and organization-specific information.
* Can improve factual relevance and make answers easier to verify.
* Allows knowledge to be updated by maintaining the index instead of retraining the LLM.
* Supports source attribution when document references are preserved and correctly associated with answers.

**Limitations and failure scenarios**

* Incorrect parsing, stale indexes, poor chunking, or weak retrieval can produce inaccurate answers.
* A relevant document may exist but still not be retrieved.
* Similarity scores alone do not guarantee factual correctness.
* Retrieved documents can contain malicious prompt-injection instructions.
* Access restrictions must be enforced during retrieval, not just when presenting the final answer.
* RAG adds retrieval infrastructure, indexing work, monitoring, and latency.

**Production controls:** Evaluate retrieval precision and recall, measure answer correctness and groundedness, preserve source metadata, enforce document-level authorization, refresh indexes reliably, and provide a clear fallback when evidence is insufficient.

## 4. Model Context Protocol (MCP)

### 4.1 What is MCP?

The Model Context Protocol is an open protocol that standardizes how AI applications connect to external tools and data sources.

Without a common protocol, developers may need to implement and maintain separate integrations for every AI application and external system. MCP provides a common interface for discovering available capabilities and exchanging structured requests and responses.

MCP does not itself make an LLM more intelligent, guarantee that tool calls are safe, or replace application-level authorization.

### 4.2 MCP Architecture

The main components are:

* **MCP Host:** The AI application that manages the model interaction, user experience, and overall control flow.
* **MCP Client:** The component within the host that communicates with an MCP server.
* **MCP Server:** Exposes approved capabilities to the client, such as querying a database, accessing permitted files, or invoking an API.

MCP uses structured protocol messages, based on JSON-RPC, and supports transports such as standard input/output (stdio) for local integrations and Streamable HTTP for remote integrations.

### 4.3 MCP Primitives

* **Tools:** Executable operations that the model can request through the host, such as running a read-only database query or creating a ticket. The host and server must enforce authorization and execution policies.
* **Resources:** Data or contextual information exposed by a server, such as documentation, files, or database records.
* **Prompts:** Reusable prompt templates or workflows exposed by an MCP server.

The model does not automatically gain unrestricted access to every connected system. The host controls the interaction, while the server and underlying system should enforce their own permissions.

### 4.4 MCP Execution Flow

1. The AI host establishes a connection to an approved MCP server.
2. The client and server negotiate capabilities and discover available tools or resources.
3. The user requests an operation, such as retrieving deployment status.
4. The model may propose an appropriate tool call.
5. The host evaluates whether the call is allowed and applies any required approval policy.
6. The MCP client sends a structured request to the server.
7. The server validates the request, checks permissions, and invokes the underlying system.
8. The server returns the result to the client.
9. The host supplies the result to the model, which generates a user-facing response.

The precise approval and authorization steps depend on the application design. Tool discovery or protocol support must not be treated as proof that a requested operation is safe.

### 4.5 Production Risks and Controls

MCP integrations can introduce risks such as excessive permissions, malicious tool descriptions, compromised servers, sensitive data exposure, and unauthorized actions.

Recommended controls include:

* Allowlist approved MCP servers and tools.
* Apply least-privilege access to files, databases, APIs, and cloud resources.
* Prefer read-only access when write operations are unnecessary.
* Validate inputs and constrain tool arguments.
* Require explicit confirmation for sensitive or destructive actions.
* Protect credentials and use approved authentication mechanisms.
* Validate server identity and secure remote connections.
* Log tool invocations and relevant security events without unnecessarily recording secrets.
* Restrict network access and prevent unauthorized data exfiltration.

## 5. RAG vs. MCP

| Aspect             | RAG                                                           | MCP                                                                |
| ------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------ |
| Primary purpose    | Retrieve relevant knowledge to ground responses               | Standardize access to external capabilities and context            |
| Main mechanism     | Embeddings, search, retrieval and context construction        | Protocol-based client/server communication                         |
| Typical components | Document pipeline, retriever, vector index, LLM               | Host, client, server, tools and resources                          |
| Example            | Answer a question using an internal operations runbook        | Query an approved monitoring API                                   |
| Main risks         | Poor retrieval, stale data, unauthorized or malicious content | Excessive permissions, unsafe tool calls, compromised integrations |
| Main controls      | Retrieval evaluation, source attribution, access filtering    | Authentication, authorization, least privilege, tool validation    |

**Key distinction:** RAG addresses what relevant information should be supplied to the model. MCP addresses how an AI application can interact with external systems through standardized interfaces.

They are complementary. An MCP server may expose a search service that retrieves RAG-indexed information, while a separate approved tool provides live infrastructure status.

## 6. Client AI Guardrails

Before using AI on client projects, technical capability must be balanced with data confidentiality, intellectual property protection, and operational safety.

### 6.1 Data Classification and Privacy

* Use only AI tools and accounts approved by the client and organization.
* Follow the applicable data-classification and AI-use policies before submitting any content.
* Never enter credentials, private keys, API tokens, or other secrets into an AI prompt.
* Do not upload confidential source code, customer records, production logs, or internal architecture information unless explicitly authorized under the approved workflow.
* Minimize, redact, or anonymize data wherever practical.
* Verify the approved service's data-retention, access, logging, and training-use terms rather than assuming that all AI services handle data identically.

### 6.2 Prompt Injection and Untrusted Content

Prompt injection occurs when malicious instructions are embedded in user input, retrieved documents, webpages, emails, or tool results to influence model behavior.

For example, an internal document might contain instructions to ignore the user's request and disclose confidential data.

Mitigations:

* Treat retrieved documents and external tool results as untrusted data, not as authoritative system instructions.
* Separate trusted application instructions from user-supplied and retrieved content.
* Enforce permissions and data filtering outside the model.
* Restrict tool capabilities and validate proposed actions before execution.
* Test indirect prompt-injection scenarios during security reviews.

Prompt wording alone is not a reliable security boundary.

### 6.3 Intellectual Property and Output Validation

* Use only approved repositories and services for client code and artifacts.
* Review generated code for security flaws, licensing concerns, dependency risks, and compatibility with project standards.
* Test AI-generated scripts and infrastructure changes in a controlled environment before deployment.
* Never assume generated output is correct, original, secure, or production-ready.
* Require qualified human review for significant architecture, security, or production changes.

### 6.4 Human Oversight and Auditability

For sensitive or high-impact use cases, maintain traceability for relevant prompts, retrieved sources, tool calls, approvals, and final actions, subject to client privacy and retention policies.

AI should not independently execute destructive infrastructure operations, change production access policies, or expose sensitive data without appropriate controls and authorization.

## 7. Example: AI Assistant for Cloud Operations

Consider an internal assistant that helps engineers troubleshoot failed CI/CD deployments.

**RAG component:** Index approved runbooks, incident documentation, and deployment procedures. When an engineer asks why a pipeline failed, retrieve relevant troubleshooting guidance and generate an evidence-based explanation.

**MCP component:** Connect to approved monitoring or CI/CD APIs to retrieve live build status, error summaries, and deployment metadata. Write operations, such as restarting a deployment, should require the appropriate permissions and approval.

**Guardrails:** Redact credentials from logs, enforce the requesting engineer's access rights, treat retrieved log content as untrusted, validate tool arguments, and require approval before executing risky operations.

**Failure scenario:** A log contains a malicious instruction to send an access token to an external endpoint. The assistant must treat that instruction as untrusted content, avoid disclosing the token, and prevent unauthorized network or tool actions.

This architecture combines knowledge retrieval with controlled operational access while keeping authorization and execution safeguards outside the LLM's generated text.

## 8. Conclusion

LLMs provide language-generation capabilities, RAG grounds responses in retrieved information, and MCP standardizes how AI applications connect to external tools and data sources.

Neither RAG nor MCP automatically guarantees accuracy, confidentiality, or safe execution. Production readiness requires secure data handling, access control, input and output validation, retrieval evaluation, tool restrictions, auditability, and human oversight.

Before using these technologies on client projects, the engineer should understand the end-to-end data flow, identify trust boundaries, evaluate failure scenarios, and follow the approved client AI governance process.
