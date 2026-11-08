# OPNsense

OPNsense runs both as a VM on my proxmox node and bare-metal on my old lenovo-desktop, performing the following roles:
- Inter-VLAN and site-to-site routing
- Firewall rule enforcement and logging
- VPN tunneling

## Policy

**The core idea:**
Management is the only trusted zone. Everything else is treated as hostile and blocked until a rule says otherwise. Each zone gets what it needs and nothing else.
Any aditional rules are host-specific, e.g. if an agent machine needs access to my Gitea service, it gets its own dedicated rule "agent1_host -> gitea_host:gitea_port".

<details>
<summary>Home</summary>

| Rule | Src zone | Dst | Action |
| --- | --- | --- | --- |
| management -> any | management | any | pass |
| services -> dns | services | firewall:53 | pass |
| services -X-> LAN | services | 10.0.0.0/16 | block |
| services -> WAN | services | any | pass |
| infra -> ntp | infra | firewall:123 | pass |
| infra -X-> any | infra | any | block |
| family -> dns | family | firewall:53 | pass |
| family -> streaming | family | pi-a:streaming_ports | pass |
| family -X-> LAN | family | 10.0.0.0/16 | block |
| family -> WAN | family | any | pass |
| guest -> dns | guest | firewall:53 | pass |
| guest -X-> LAN | guest | 10.0.0.0/16 | block |
| guest -> WAN | guest | any | pass |
| iot -> dns | iot | firewall:53 | pass |
| iot -X-> LAN | iot | 10.0.0.0/16 | block |
| iot -> WAN | iot | any | pass |
| cctv -X-> any | cctv | any | block |

</details>

<details>
<summary>Powerhouse</summary>

| Rule | Source | Dest | Action | Via |
| --- | --- | --- | --- | --- |
| management -> any | management | any | pass | |
| services -> dns | services | firewall:53 | pass | |
| LLM-access -> ollama | LLM-access | ollama_hosts:ollama_port | pass | |
| LLM-access -> comfyUI | LLM-access | comfyUI_hosts:comfyUI_port | pass | |
| services -> dns | services | firewall:53 | pass | |
| services -> research | services | research_tools:research_ports | pass | |
| services -X-> LAN | services | LAN | block | |
| services -> WAN | services | any | pass | |
| agent-web -> LiteLLM | agent-web | LLM-access:LiteLLM_port | pass | |
| agent-web -X-> LAN | agent-web | LAN | block | |
| agent-web -> agent-VPN dns | agent-web | agent-VPN-GW:53 | pass | agent-VPN-GW |
| agent-web -X-> dns | agent-web | any:53 | block | |
| agent-web -> WAN via agent-VPN | agent-web | any | pass | agent-VPN-GW |
| agent-web -x-> WAN | agent-web | any | block | |
| agent-sandbox -> LiteLLM | agent-sandbox | LiteLLM:LiteLLM_port | pass | |
| agent-sandbox -X-> any | agent-sandbox | any | block | |
| workbench -> workbench-VPN DNS | workbench | workbench-VPN-GW:53 | pass | workbench-VPN-GW |
| workbench -X-> DNS | workbench | any:53 | block | |
| workbench -X-> LAN | workbench | LAN | block | |
| workbench -> WAN via workbench-VPN | workbench | any | pass | workbench-VPN-GW |
| workbench -x-> WAN | workbench | any | block | |
| intake -> intake-VPN DNS | intake | intake-VPN-GW:53 | pass | intake-VPN-GW |
| intake -X-> DNS | intake | any:53 | block | |
| intake -X-> LAN | intake | LAN | block | |
| intake -> WAN via intake-VPN | intake | any | pass | intake-VPN-GW |
| intake -x-> WAN | intake | any | block | |

</details>