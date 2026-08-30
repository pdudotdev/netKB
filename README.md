# YANA • Yet Another Network Agent

[![Version](https://img.shields.io/badge/version-1.2-1a1a2e)](https://github.com/pdudotdev/YANA/releases/tag/v1.2.0)
![License](https://img.shields.io/badge/license-GPLv3-1a1a2e)
[![Last Commit](https://img.shields.io/github/last-commit/pdudotdev/YANA?color=1a1a2e)](https://github.com/pdudotdev/YANA/commits/main/)

| | |
|---|---|
| **Platforms** | ![Cisco IOS](https://img.shields.io/badge/Cisco_IOS-0d47a1) ![Cisco IOS-XE](https://img.shields.io/badge/Cisco_IOS--XE-0d47a1) ![Arista EOS](https://img.shields.io/badge/Arista_EOS-0d47a1) ![Juniper JunOS](https://img.shields.io/badge/Juniper_JunOS-0d47a1) ![Aruba AOS](https://img.shields.io/badge/Aruba_AOS-0d47a1) ![Vyatta VyOS](https://img.shields.io/badge/Vyatta_VyOS-0d47a1) ![MikroTik RouterOS](https://img.shields.io/badge/MikroTik_RouterOS-0d47a1) ![FRR](https://img.shields.io/badge/FRR-0d47a1) |
| **Transport** | ![SSH](https://img.shields.io/badge/SSH%20CLI-1565c0) ![Scrapli](https://img.shields.io/badge/Scrapli-1565c0) |
| **Integrations** | ![MCP](https://img.shields.io/badge/MCP-1e88e5) ![ChromaDB](https://img.shields.io/badge/ChromaDB-1e88e5) ![JUnit XML](https://img.shields.io/badge/JUnit_XML-1e88e5) |

## Overview

RAG-powered network QA investigation tool for multi-vendor networks.

Run your tests with any framework. When something fails, YANA investigates - it queries live devices, searches protocol specs, follows diagnostic decision trees, and tells you why it failed and what to fix.

**Recommended models:**
- Opus 4.6+ (1M)

**Documentation:**
- [**WORKFLOW.md**](metadata/workflow/WORKFLOW.md) - operational flow
- [**GUARDRAILS.md**](metadata/guardrails/GUARDRAILS.md) - security controls

**What's new:**
- [**CHANGELOG.md**](CHANGELOG.md) - latest version

## Tech Stack

| Technology | Role |
|-----------|------|
| Python | Core language |
| FastMCP | MCP server exposing 8 tools |
| Claude | Reasoning, context, investigation |
| LangChain | RAG pipeline (chunking, embedding, retrieval) |
| ChromaDB | Vector database for knowledge base |
| Scrapli | Multi-vendor SSH transport |
| JUnit XML | Test results format |

## Scope

| Protocol | What's Checked |
|----------|---------------|
| **OSPF** | Adjacencies, area and area types, config, LSDB |
| **Routing** | Table, route maps, prefix lists, PBR, ACLs, ECMP |
| **Interfaces** | Up/down state, expected operational status |

## Test Network Topology

**Network diagram:**

![topology](metadata/topology/DBL-TOPOLOGY.png)

**Lab environment:**
- 16 devices defined in [**TOPOLOGY.yml**](TOPOLOGY.yml)
- 5 x Cisco IOS, 3 x Cisco IOS-XE, 4 x Arista cEOS, 2 x MikroTik CHR, 1 x Juniper JunOS, 1 x Aruba AOS-CX
- See [**lab_configs**](lab_configs/) for my test network's configuration

## Installation

**Prerequisites:** Python 3.11+

**Step 1 - Install and ingest:**
```bash
sudo apt install git make python3.12-venv -y
cd ~ && git clone https://github.com/pdudotdev/YANA
cd YANA && make setup
```

**Step 2 - Configure credentials:**
```bash
cp .env.example .env
# Edit .env with your device credentials
```

**Step 3 - Authenticate with Claude:**
```bash
claude auth login
```

**Step 4 - Register the MCP server:**
```bash
claude mcp add yana -s user -- /home/<user>/YANA/yana/bin/python /home/<user>/YANA/server/MCPServer.py
```

## MCP Tools

| Tool | Description |
|------|-------------|
| `get_status` | Report active data sources (inventory, intent, KB) |
| `list_devices` | List inventory devices, optionally filtered by CLI style |
| `get_ospf` | Query OSPF state (neighbors, LSDB, process config) on a live device |
| `get_interfaces` | Query interface up/down status on a live device |
| `get_routing` | Query routing table, route maps, prefix lists, PBR, and ACLs on a live device |
| `query_intent` | Retrieve network design intent from local JSON |
| `traceroute` | Trace the forwarding path from a device to a destination IP |
| `search_knowledge_base` | Search network knowledge base (RFCs, vendor guides) with protocol/vendor/topic filters |

## Customization

YANA is designed to work with your own test topology. Bring your own:

| What | Where | Format |
|------|-------|--------|
| **Device inventory** | `data/NETWORK.json` | JSON dict: device name -> host, platform, cli_style |
| **Design intent** | `data/INTENT.json` | JSON: router roles, OSPF areas, links, neighbors |
| **Protocol docs** | `docs/*.md` | Protocol data (re-ingest with `make ingest`) |
| **Test results** | `results/*.xml` | JUnit XML from any test runner |
| **Credentials** | `.env` | `ROUTER_USERNAME` and `ROUTER_PASSWORD` |

## QA Workflow

1. **Run your tests** with any framework (pytest, pyATS, Ansible, etc.)
2. **Place JUnit XML results** in `results/`
3. **Investigate failures** with the `/qa` skill:

```
claude
> /qa
```

The `/qa` skill loads the latest JUnit XML, lists all failures, and investigates each one using live device queries, network intent, and RFC context.

## Knowledge Base

Protocol documentation lives in `docs/` as Markdown files. Each file is tagged with `vendor`, `topic`, and `protocol` metadata during ingestion.

To update after editing docs:
```bash
make ingest
```

**RAG optimizations in place:**
- `protocol` metadata field - filters search by protocol (ospf, bgp, eigrp), eliminating cross-protocol noise
- Contextual chunk headers - source and protocol prepended to each chunk for better embedding quality
- Compound filtering - combine vendor, topic, and protocol filters in a single query

See [**OPTIMIZATIONS.md**](metadata/scalability/OPTIMIZATIONS.md) for the full optimization roadmap.

## Project Structure

```
YANA/
├── server/
│   └── MCPServer.py              # FastMCP server (8 tools)
├── tools/                        # MCP tool implementations
│   ├── ospf.py                   # get_ospf
│   ├── operational.py            # get_interfaces, traceroute
│   ├── routing.py                # get_routing
│   ├── intent.py                 # query_intent
│   ├── status.py                 # get_status
│   ├── inventory_tool.py         # list_devices
│   └── rag.py                    # search_knowledge_base
├── core/                         # Infrastructure
│   ├── inventory.py              # Device dict (from data/NETWORK.json)
│   └── settings.py               # Credentials + SSH config
├── data/                         # Data files
│   ├── NETWORK.json              # Device inventory
│   ├── INTENT.json               # Network design intent
│   └── chroma/                   # ChromaDB vector store (generated)
├── transport/                    # SSH execution (Scrapli)
│   ├── __init__.py               # Command dispatcher + semaphore
│   └── ssh.py                    # Multi-vendor SSH executor
├── platforms/                    # Vendor CLI command mapping
│   ├── platform_map.py           # OSPF, routing, interface + traceroute commands (6 vendors)
│   └── definitions/              # Custom Scrapli definitions
├── input_models/
│   └── models.py                 # Pydantic validation (OspfQuery, KBQuery, etc.)
├── docs/                         # Knowledge base (RFCs + vendor guides)
├── results/                      # QA test results (JUnit XML)
├── skills/
│   ├── ospf/                     # OSPF diagnostic decision tree
│   └── routing/                  # Routing diagnostic decision tree
├── testing/
│   ├── automated/                # Unit + integration tests
│   ├── live/                     # Live lab tests
│   └── run_tests.sh              # Test runner (--live for lab tests)
├── metadata/
│   ├── guardrails/               # Security controls documentation
│   ├── scalability/              # RAG optimization roadmap
│   ├── topology/                 # Test network diagram
│   └── workflow/                 # RAG pipeline walkthrough
├── ingest.py                     # RAG ingestion pipeline
├── Makefile                      # Setup automation (make setup)
├── CLAUDE.md                     # Investigation workflow + tool reference
├── TOPOLOGY.yml                  # Containerlab topology definition
├── .env.example                  # Credential template
├── CHANGELOG.md                  # Version history
├── LICENSE
├── requirements.txt
└── README.md
```

## Planned Upgrades

- EIGRP and BGP support (tools, skills, knowledge base docs)

## Disclaimer

You are responsible for defining your own network inventory and design intent, building your test environment, and meeting the necessary prerequisites (Python 3.11+, Claude CLI/API, network device access).

## License

Licensed under [**GNUv3.0**](LICENSE).

## Hi
Wanna say hello? Send me a DM at [**LinkedIn**](https://www.linkedin.com/in/tmihaicatalin/)

