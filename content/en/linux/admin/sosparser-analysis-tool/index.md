---
title: Streamlining Linux Diagnostics with SOSParser
description: A comprehensive guide to using SOSParser, an automated analysis tool for Linux sosreport and supportconfig diagnostic files.
date: 2026-01-06
type: docs
author: Samuel Matildes
tags: [linux, diagnostics, sosreport, supportconfig, troubleshooting, system-analysis, automation]
---

<p>
  <a href="https://github.com/samatild/SOSParser" target="_blank" rel="noopener">
    <i class="fab fa-github" aria-hidden="true"></i> GitHub
  </a>
  &nbsp;•&nbsp;
  <a href="https://hub.docker.com/r/samuelmatildes/sosparser" target="_blank" rel="noopener">
    <i class="fab fa-docker" aria-hidden="true"></i> Docker Hub
  </a>
  &nbsp;•&nbsp;
</p>

<i class="fas fa-search" aria-hidden="true"></i> Parse, analyze, and understand Linux diagnostic reports with automated intelligence.

## What is SOSParser?

[`SOSParser`](https://github.com/samatild/SOSParser) is a powerful web application designed to automatically parse and analyze Linux `sosreport` and `supportconfig` diagnostic files, converting them into comprehensive, interactive HTML reports. Created to streamline the often tedious process of manually reviewing system diagnostic data, SOSParser transforms raw diagnostic archives into structured, searchable insights that accelerate troubleshooting and system analysis.

Whether you're a system administrator, DevOps engineer, or support technician dealing with complex Linux environments, SOSParser provides an automated approach to understanding what's happening inside your systems.

<img src="./img/sosparser.gif" alt="SOSParser screenshot: Animated workflow of analysis and reporting" style="max-width: 100%; border-radius:8px; box-shadow:0 2px 12px rgba(0,0,0,0.07); margin: 1.5rem 0;">



## The Problem SOSParser Solves

When Linux systems encounter issues, the standard diagnostic approach involves generating comprehensive reports using tools like:

- **`sosreport`** - A utility that collects detailed system information from Red Hat-based distributions
- **`supportconfig`** - SUSE's equivalent diagnostic collection tool

These reports contain thousands of files with critical system information, but analyzing them manually is:

- **Time-consuming**: Hours of sifting through logs, configurations, and system data
- **Error-prone**: Easy to miss important correlations between different system components
- **Inconsistent**: Different analysts may interpret the same data differently
- **Repetitive**: Common patterns and issues require rediscovery each time

SOSParser addresses these challenges by providing automated, intelligent analysis that surfaces key insights immediately.

## How SOSParser Works

### Input Processing

SOSParser accepts standard diagnostic archives in various compressed formats:
- `.tar.xz` (most common)
- `.tar.gz`
- `.tar.bz2`
- `.tar`

### Analysis Pipeline

Once uploaded, SOSParser processes the diagnostic data through multiple analysis modules:

1. **Data Extraction**: Automatically unpacks and organizes the diagnostic archive
2. **Content Parsing**: Extracts and structures data from hundreds of system files
3. **Correlation Analysis**: Identifies relationships between different system components
4. **Insight Generation**: Applies heuristics and rules to identify potential issues
5. **Report Generation**: Creates an interactive HTML report with visualizations and recommendations

## What SOSParser Analyzes

### System Information
- **Hardware Details**: CPU architecture, memory configuration, disk layout
- **OS Information**: Distribution, version, kernel details
- **System Resources**: Current utilization, capacity planning insights

### System Configuration
- **Boot Configuration**: GRUB settings, init systems, startup services
- **Authentication**: PAM configuration, user management, security policies
- **Services**: Systemd units, cron jobs, running processes
- **Security**: SELinux/AppArmor status, firewall rules, package integrity

### Filesystem Analysis
- **Mount Points**: Filesystem types, mount options, capacity usage
- **LVM Configuration**: Volume groups, logical volumes, physical volumes
- **Disk Usage**: Largest directories, file ownership patterns, permission issues
- **Filesystem Health**: Journal status, inode usage, fragmentation indicators

### Network Analysis
- **Interface Configuration**: IP addresses, subnet masks, gateway settings
- **Routing Tables**: Static and dynamic routes, network connectivity
- **DNS Configuration**: Resolvers, search domains, DNS query patterns
- **Firewall Rules**: iptables/nftables configuration, active rulesets
- **Network Services**: Listening ports, connection states, network statistics

### Log Analysis
- **System Logs**: `/var/log/messages`, `/var/log/syslog`, journald entries
- **Kernel Logs**: `dmesg` output, kernel ring buffer analysis
- **Authentication Logs**: Login attempts, sudo usage, security events
- **Service Logs**: Application-specific log analysis and error pattern detection
- **Security Events**: Failed access attempts, intrusion indicators

### Cloud Services Integration
- **AWS**: EC2 instance metadata, IAM roles, VPC configuration
- **Azure**: VM extensions, resource groups, networking setup
- **GCP**: Compute Engine metadata, service accounts, network configuration
- **Oracle Cloud**: Instance details, VNICs, storage configuration

## Getting Started with SOSParser

### Docker Deployment (Recommended)

The easiest way to run SOSParser is using Docker:

```bash
# Pull the official image
docker pull samuelmatildes/sosparser:latest

# Run the container
docker run -d -p 8000:8000 --name sosparser samuelmatildes/sosparser:latest
```

Then open `http://localhost:8000` in your browser.

### Persisting Data

For production use, mount volumes to persist uploads and generated reports:

```bash
# Using bind mounts
docker run -d -p 8000:8000 --name sosparser \
  -v $(pwd)/data/uploads:/app/webapp/uploads \
  -v $(pwd)/data/outputs:/app/webapp/outputs \
  samuelmatildes/sosparser:latest

# Using named volumes
docker run -d -p 8000:8000 --name sosparser \
  -v sosparser_uploads:/app/webapp/uploads \
  -v sosparser_outputs:/app/webapp/outputs \
  samuelmatildes/sosparser:latest
```

### Local Development

To build and run locally:

```bash
git clone https://github.com/samatild/SOSParser.git
cd SOSParser
docker build -t sosparser:local .
docker run -d -p 8000:8000 sosparser:local
```

## Using SOSParser

### Web Interface Workflow

1. **Upload**: Select your `sosreport` or `supportconfig` file (supports multiple formats)
2. **Analyze**: Click "Analyze Report" to start automated processing
3. **Review**: Browse the generated interactive HTML report
4. **Export**: Download reports for sharing or archival

### Report Features

The generated reports include:

- **Interactive Navigation**: Collapsible sections, searchable content
- **Visual Indicators**: Color-coded severity levels for issues
- **Cross-References**: Links between related system components
- **Recommendations**: Actionable suggestions based on findings
- **Export Options**: PDF generation, data extraction

### Common Use Cases

#### Incident Response
- Rapid triage of production system issues
- Correlation of symptoms across multiple subsystems
- Identification of root cause patterns

#### Capacity Planning
- Resource utilization analysis
- Performance bottleneck identification
- Growth trend assessment

#### Security Audits
- Configuration compliance checking
- Vulnerability assessment
- Access pattern analysis

#### Change Validation
- Pre/post-change comparison
- Configuration drift detection
- Impact assessment

## Advanced Features and Roadmap

### Currently Available
- Multi-format diagnostic file support
- Cloud platform detection and analysis
- Comprehensive system health scoring
- Interactive HTML report generation

### Planned Enhancements
- **Advanced Disk Diagnostics**: SMART data analysis, ATA command integration
- **Application Server Analysis**: Apache/Nginx configuration parsing, database connectivity
- **Container Orchestration**: Kubernetes pod analysis, Docker container inspection
- **Backup System Integration**: Backup status validation, recovery testing
- **Monitoring Integration**: Prometheus metrics correlation, alerting rule validation
- **Machine Learning**: Anomaly detection, predictive issue identification

## Performance and Scalability

SOSParser is designed to handle large diagnostic reports efficiently:

- **Processing Speed**: Most reports analyzed in under 2 minutes
- **Memory Usage**: Optimized for systems with 2GB+ RAM
- **Storage**: Reports typically 10-20% of original archive size
- **Concurrency**: Supports multiple simultaneous analyses

## Security Considerations

- **Local Processing**: All analysis occurs locally - no data sent to external services
- **Container Isolation**: Docker deployment provides additional security boundaries
- **Data Privacy**: Sensitive information remains within your infrastructure
- **Audit Trail**: Processing logs available for compliance requirements

## Integration and Automation

### API Access
SOSParser provides REST API endpoints for integration with existing workflows:

```bash
# Upload and analyze via API
curl -X POST -F "file=@sosreport.tar.xz" http://localhost:8000/api/analyze
```

### CI/CD Integration
- Automated analysis of system snapshots
- Regression testing for configuration changes
- Compliance validation pipelines

### Monitoring Integration
- Alert generation based on analysis results
- Dashboard integration for system health overview
- Trend analysis across multiple systems

## Contributing and Community

SOSParser is an open-source project that welcomes contributions:

- **Bug Reports**: Use GitHub Issues for problems or feature requests
- **Code Contributions**: Pull requests are reviewed and merged regularly
- **Documentation**: Help improve guides and examples
- **Testing**: Report compatibility with different Linux distributions

Join the community on [Telegram](https://t.me/sosparser) for updates, discussions, and support.

## Troubleshooting SOSParser

### Common Issues

**Upload Failures**
- Check file size limits (typically 500MB max)
- Verify archive integrity before upload
- Ensure proper file permissions

**Analysis Errors**
- Confirm the diagnostic file was generated correctly
- Check for corrupted archives
- Review Docker logs for processing errors

**Performance Issues**
- Allocate sufficient CPU and memory resources
- Process large reports during off-peak hours
- Consider horizontal scaling for high-volume environments

## Conclusion

SOSParser represents a significant advancement in Linux system diagnostics, transforming the traditionally manual and time-intensive process of analyzing `sosreport` and `supportconfig` files into an automated, intelligent workflow. By providing comprehensive analysis, actionable insights, and interactive reports, it empowers system administrators and support teams to resolve issues faster and maintain healthier Linux environments.

Whether you're managing a single server or overseeing enterprise-scale deployments, SOSParser provides the tools needed to understand your systems at a deeper level, identify potential issues before they become critical, and maintain optimal system health.

---

**Learn More**
- Project Repository: [`github.com/samatild/SOSParser`](https://github.com/samatild/SOSParser)
- Docker Hub: [`hub.docker.com/r/samuelmatildes/sosparser`](https://hub.docker.com/r/samuelmatildes/sosparser)
- Issue Tracker: [GitHub Issues](https://github.com/samatild/SOSParser/issues)