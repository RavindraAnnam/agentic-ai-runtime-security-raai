# Runtime Threat Matrix

| Threat | Attack Scenario | Impact | Recommended Control |
|---|---|---|---|
| Prompt Injection | Hidden instructions manipulate agent behavior | Data leakage, policy bypass | Context filtering, instruction hierarchy, input validation |
| Tool Abuse | Agent invokes tools beyond approved purpose | Fraud, unauthorized action | Tool allow lists, scoped permissions, tool approval |
| Memory Poisoning | Malicious memory influences future actions | Persistent manipulation | Memory validation, TTL, source trust scoring |
| API Abuse | Agent calls sensitive APIs incorrectly | Unauthorized access, service disruption | API gateway, RBAC, rate limits, audit logging |
| Agent Impersonation | Rogue agent behaves as trusted agent | Trust chain compromise | Agent identity, certificates, workload identity |
| Multi-Agent Escalation | Compromised agent influences other agents | Lateral movement | Agent trust boundaries, delegation control |
| Data Exfiltration | Agent leaks sensitive data through output | Privacy/compliance issue | DLP, output inspection, redaction |
| Workflow Hijacking | Agent changes business logic or approval flow | Operational disruption | Human-in-the-loop, workflow validation |
| Excessive Autonomy | Agent performs high-risk actions without review | Business and compliance risk | Risk-based approval gates |
| Weak Observability | No trace of agent decisions/actions | Incident response failure | Runtime logs, SIEM integration, decision tracing |
