# Autonomous SOC Triage Agent (LlamaIndex + Ollama)

An autonomous Tier-2 Security Operations Center (SOC) Triage Agent built with **LlamaIndex** (`AgentWorkflow` & event-driven `Workflow`) and powered by a local open-source LLM via **Ollama**. It parses raw security logs, enriches Indicators of Compromise (IOCs), correlates SIEM history, consults internal SOC playbooks via RAG (ChromaDB), and executes safe containment or ticket escalation.

## 1. Open-Source LLM & Provider

* **Model:** [`qwen2.5:14b`](https://ollama.com/library/qwen2.5:14b) (via [Ollama](https://ollama.com/library/qwen2.5); `qwen2.5:7b` can be used on lower-VRAM hardware)
* **Provider:** **Alibaba Cloud** ([Qwen](https://www.alibabacloud.com/en/solutions/generative-ai/qwen?_p_lc=1))
* **Why Local Ollama for SOC?** Security telemetry contains sensitive internal IPs, hostnames, and user identities. Running `qwen2.5:14b` locally through [LlamaIndex's Ollama integration](https://developers.llamaindex.ai/python/framework-api-reference/llms/ollama/) (`FunctionCallingLLM`) keeps all log data private with zero external LLM API calls, while delivering strong structured JSON/log parsing and a 128K-token context window.

## 2. Agent Tools

| Tool Name | Type | Purpose |
| :--- | :--- | :--- |
| `parse_security_log` | `FunctionTool` | Extracts IPs, file hashes, hostnames, and usernames from raw SIEM/endpoint logs |
| `check_threat_intel` | `FunctionTool` | Checks extracted IPs, domains, and hashes against Threat Intelligence feeds |
| `query_siem_history` | `FunctionTool` | Retrieves recent SIEM events for a host to detect lateral movement or prior alerts |
| `soc_playbooks` | `QueryEngineTool` | Queries internal SOC Incident Response Playbooks stored in ChromaDB via RAG |
| `isolate_host` | `FunctionTool` | Isolates a compromised endpoint via EDR/NAC (`dry_run=True` by default for safety) |
| `create_soc_ticket` | `FunctionTool` | Escalates confirmed incidents with severity (`P1`–`P4`), summary, and IOCs |

## 3. System Prompt

```text
You are an Autonomous Tier-2 SOC Triage Agent specializing in network security, endpoint telemetry, and log analysis.

Your mission is to triage incoming security alerts, correlate raw logs with threat intelligence and internal SOC playbooks, determine whether an alert is a True Positive, Benign Positive, or False Positive, and recommend or execute containment actions safely.

## Core Triage Workflow
1. Parse & Extract IOCs: Always start by calling `parse_security_log` on raw log lines or alert payloads to extract IPs, domains, hashes, usernames, hosts, and event IDs.
2. Enrich Indicators: Use `check_threat_intel` on every extracted external IP, domain, or file hash. Never guess reputation from memory.
3. Correlate Context: Call `query_siem_history` for the affected host or user to check for lateral movement, brute-force patterns, or prior alerts.
4. Consult Playbooks: Query `soc_playbooks` (RAG) to match the observed behavior against official SOC severity criteria, MITRE ATT&CK mappings, and escalation procedures.
5. Decide & Act:
   - False Positive / Benign: Document evidence and close without containment.
   - Low / Medium (P3/P4): Create a ticket via `create_soc_ticket` with full IOCs and recommended analyst follow-up.
   - High / Critical (P1/P2) with confirmed malicious IOC or active C2/ransomware: Call `isolate_host` (with `dry_run=True` unless explicit auto-containment is authorized) and immediately call `create_soc_ticket` with severity P1/P2.

## Guardrails & Output Format
- Zero Fabrication: Never invent IP addresses, hashes, timestamps, or CVE/MITRE IDs not present in tool outputs.
- Safety First: Treat `isolate_host` as a high-impact action; only trigger it when threat intel or SIEM correlation confirms malicious activity.
- Structured Final Report: Conclude every investigation with Verdict, Severity, MITRE ATT&CK Mapping, Key Evidence & IOCs, and Actions Taken / Recommended Next Steps.
```

## 4. Implementation Overview (`autonomous_soc_triage_agent.ipynb`)

The notebook follows a modular LlamaIndex agentic architecture:
1. **Setup & Local LLM:** Configures `Ollama(model="qwen2.5:14b")` and `BAAI/bge-small-en-v1.5` embeddings.
2. **Playbook RAG Pipeline:** Ingests SOC runbooks into `ChromaVectorStore` via `IngestionPipeline` and `SentenceSplitter`.
3. **Single & Multi-Agent Workflows:** Demonstrates a stateful `AgentWorkflow` with `Context` memory, plus a multi-agent handoff between `triage_analyst` and `containment_responder` (`ReActAgent`).
4. **Deterministic Event-Driven Pipeline:** Implements `SOCTriageWorkflow` (`StartEvent` → `ParsedLogEvent` → `EnrichedAlertEvent` → `StopEvent`) with `draw_all_possible_flows`.

## Quickstart

1. **Install and start Ollama**, then pull the model:
   ```bash
   ollama pull qwen2.5:14b
   ```
2. **Install Python dependencies** (or run the first cell in the notebook):
   ```bash
   pip install "llama-index-core>=0.14.25,<0.15" "llama-index-llms-ollama>=0.5" \
     "llama-index-embeddings-huggingface>=0.8,<0.9" "llama-index-vector-stores-chroma>=0.6,<0.7" \
     chromadb "llama-index-utils-workflow>=0.11,<0.12" python-dotenv
   ```
3. **Open and run `autonomous_soc_triage_agent.ipynb`** in Jupyter or VS Code.
