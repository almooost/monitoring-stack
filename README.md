# Distributed Customer Monitoring Stack

## Overview

A production-ready monitoring infrastructure for distributed customer environments using Prometheus, Alertmanager, and Grafana. Designed for multi-tenant deployments with centralized alert forwarding via vmagent.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                    CUSTOMER SITE (Firewalled)                                                   │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                   │
│  ┌────────────────────────────────┐  ┌────────────────────────────┐  ┌─────────────────────┐   │
│  │   Prometheus                   │  │   Exporters                │  │  Alertmanager       │   │
│  │  (30-60 day retention)         │  │  (SNMP v3/Blackbox/Node)   │  │  (Local Aggregation)│   │
│  └────────────────────────────────┘  └────────────────────────────┘  └─────────────────────┘   │
│         │                                    │                             │                     │
│         └────────────────────────────────────┼─────────────────────────────┘                     │
│                                              │                                                   │
│  ┌────────────────────────────────┐         │         ┌────────────────────────────┐            │
│  │   Grafana                      │◄────────┼────────►│   vmagent                  │            │
│  │   (Local Visualization)        │         │         │  (Webhook for alerts)      │            │
│  └────────────────────────────────┘         │         └────────────────────────────┘            │
│                                              │                                                   │
│  DOCKER STACK (Debian 13)                   │                                                   │
│  - Ansible-managed configuration            │                                                   │
│  - Target definitions per device type       │                                                   │
│                                              │                                                   │
└──────────────────────────────────────────────┼───────────────────────────────────────────────────┘
                                               │
                        HTTPS + vmauth authentication
                                               │
                                               ▼
                          Central VictoriaMetrics Server
                          (Receives alerts + heartbeat)
                                               │
                                               ▼
                               GoAlert Integration
                            (Alert escalation & on-call)
```

## Key Features

✅ **Local Metrics**: Prometheus stores 30-60 days of metrics locally  
✅ **Centralized Alerts**: Only alerts + heartbeat sent to central server (HTTPS)  
✅ **Secure**: SNMP v3, TLS encryption, vmauth authentication  
✅ **Multi-Device Support**: Proxmox, ESXi, HP/Cisco Switches, Fortigate/SonicWall Firewalls, Linux/Windows VMs  
✅ **Infrastructure as Code**: Ansible playbooks for reproducible deployments  
✅ **Template Engine**: Prometheus configs generated from variables  
✅ **Flexible Targets**: Separate files per device type for easy management  
✅ **Scalable**: Designed for 10+ customer sites with ~1,000 metrics each  

## Deployment Targets

| Device Type | Exporter | Protocol | Port | SNMP Version |
|---|---|---|---|---|
| Proxmox | SNMP/REST API | SNMP/HTTP | 161/8006 | v3 |
| VMware ESXi | SNMP/API | SNMP/HTTPS | 161/443 | v3 |
| Physical Sensors | SNMP | SNMP | 161 | v3 |
| HP Switches | SNMP Exporter | SNMP | 161 | v3 |
| Cisco Switches | SNMP Exporter | SNMP | 161 | v3 |
| Fortigate Firewall | SNMP Exporter | SNMP | 161 | v3 |
| SonicWall Firewall | SNMP Exporter | SNMP | 161 | v3 |
| Linux VMs | Node Exporter | HTTP | 9100 | N/A |
| Windows VMs | WMI Exporter | HTTP | 9182 | N/A |

## Scale Planning

**Per Customer Site:**
- 1 Physical Hypervisor (Proxmox/ESXi): ~200 metrics
- 5 Virtual Machines: ~500 metrics
- 2-3 Network Devices: ~300 metrics
- Sensors & Other: ~50 metrics
- **Total: ~1,050 metrics/series per site**

**Total (10 customers):**
- Central server receives: ~10,500 metrics (alerts + heartbeats only)
- Each customer site storage: 1-2TB (30-60 day retention)

## Quick Start

### Prerequisites

- Debian 13 Linux server
- Docker and Docker Compose installed
- Ansible 2.10+
- Python 3.8+

### 1. Clone Repository

```bash
git clone https://github.com/almooost/monitoring-stack.git
cd monitoring-stack
```

### 2. Configure Inventory

```bash
cp ansible/inventory/examples/customer_example.yml ansible/inventory/customers/customer_a.yml
# Edit with your customer details
```

### 3. Deploy Monitoring Stack

```bash
cd ansible
ansible-playbook playbooks/deploy_monitoring_stack.yml \
  -i inventory/customers/customer_a.yml
```

### 4. Verify Deployment

```bash
# Check containers
docker ps

# Prometheus UI
curl http://localhost:9090

# Grafana UI  
curl http://localhost:3000
```

## Directory Structure

```
monitoring-stack/
├── README.md
├── LICENSE
├── CHANGELOG.md
├── docs/
│   ├── architecture.md
│   ├── deployment_guide.md
│   ├── snmp_setup.md
│   ├── device_specifics.md
│   └── troubleshooting.md
│
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── examples/
│   │   │   └── customer_example.yml
│   │   ├── group_vars/
│   │   │   └── all/
│   │   │       ├── docker_config.yml
│   │   │       ├── monitoring_defaults.yml
│   │   │       └── network_config.yml
│   │   └── customers/
│   │       └── .gitkeep
│   │
│   ├── roles/
│   │   ├── docker_setup/
│   │   ├── prometheus/
│   │   ├── snmp_exporter/
│   │   ├── blackbox_exporter/
│   │   ├── node_exporter/
│   │   ├── alertmanager/
│   │   ├── vmagent/
│   │   └── grafana/
│   │
│   ├── playbooks/
│   │   ├── deploy_monitoring_stack.yml
│   │   ├── update_prometheus_config.yml
│   │   ├── update_targets.yml
│   │   ├── deploy_alerts.yml
│   │   └── health_check.yml
│   │
│   └── templates/
│       ├── prometheus.yml.j2
│       ├── alertmanager.yml.j2
│       ├── snmp_exporter_config.yml.j2
│       ├── blackbox_exporter_config.yml.j2
│       ├── vmagent_config.yml.j2
│       ├── alert_rules.yml.j2
│       └── docker-compose.yml.j2
│
├── docker/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── snmp_exporter/
│   │   └── snmp.yml
│   ├── blackbox_exporter/
│   │   └── blackbox.yml
│   ├── alertmanager/
│   │   └── alertmanager.yml
│   ├── vmagent/
│   │   └── vmagent_config.yml
│   └── grafana/
│       ├── provisioning/
│       │   ├── dashboards/
│       │   └── datasources/
│       └── grafana.ini
│
└── examples/
    ├── alert_rules_proxmox.yml
    ├── alert_rules_esxi.yml
    ├── alert_rules_network.yml
    ├── dashboard_proxmox.json
    ├── dashboard_esxi.json
    ├── dashboard_network.json
    └── snmp_discovery.sh
```

## Documentation

- **[Architecture](docs/architecture.md)** - Detailed system design
- **[Deployment Guide](docs/deployment_guide.md)** - Step-by-step setup
- **[SNMP Setup](docs/snmp_setup.md)** - SNMP v3 configuration
- **[Device Specifics](docs/device_specifics.md)** - Per-device OID references
- **[Troubleshooting](docs/troubleshooting.md)** - Common issues and solutions

## Security Considerations

🔒 **SNMP v3** - Authentication and encryption enabled  
🔒 **TLS/HTTPS** - All central communication encrypted  
🔒 **vmauth** - Token-based authentication for central server  
🔒 **Firewall** - Customers behind firewalls, pull-based metrics only  
🔒 **Credentials** - Use Ansible vault for sensitive data  

## Monitoring Stack Components

### Customer Site Containers

| Container | Version | Role | Port |
|---|---|---|---|
| prometheus | latest | Metrics collection | 9090 |
| snmp_exporter | latest | SNMP metrics | 9116 |
| blackbox_exporter | latest | Endpoint checks | 9115 |
| node_exporter | latest | Host metrics | 9100 |
| alertmanager | latest | Alert aggregation | 9093 |
| vmagent | latest | Alert/heartbeat forwarding | 8429 |
| grafana | latest | Visualization | 3000 |

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Test on multiple customer site configurations
4. Submit a pull request with documentation

## License

MIT License - See [LICENSE](LICENSE) file

## Support

For issues, questions, or contributions:

- 📝 [GitHub Issues](https://github.com/almooost/monitoring-stack/issues)
- 📚 [Documentation](docs/)
- 💬 [Discussions](https://github.com/almooost/monitoring-stack/discussions)

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.
