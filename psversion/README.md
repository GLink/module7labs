# AKS Resource Management & Autoscaling - Hands-on Lab (PowerShell Version)

This folder contains the **PowerShell-friendly** version of the AKS Resource Management & Autoscaling labs. All bash commands have been converted to PowerShell syntax.

## 📚 Chapters

| # | Chapter | Link |
|---|---------|------|
| 0 | Setup and Prerequisites | [README](./chapter-0-setup/README.md) |
| 1 | Resource Requests and Limits | [README](./chapter-1-requests-limits/README.md) |
| 2 | LimitRange and ResourceQuota | [README](./chapter-2-limitrange-quota/README.md) |
| 3 | Horizontal Pod Autoscaler (HPA) | [README](./chapter-3-hpa/README.md) |
| 4 | KEDA - Event-Driven Autoscaling | [README](./chapter-4-keda/README.md) |
| 5 | Pod Disruption Budgets | [README](./chapter-5-pdb/README.md) |

## 🔧 Requirements

- **PowerShell 7+** (Windows PowerShell or PowerShell Core)
- **Azure CLI** (`az`)
- **kubectl**
- **Helm** (v3.x)

## 🚀 Getting Started

Start with **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)** and complete each chapter in sequence.

## 📝 Key Differences from Bash Version

| Bash | PowerShell |
|------|-----------|
| `export VAR="value"` | `$VAR = "value"` |
| `source ./file.sh` | `. ./file.ps1` |
| `\` line continuation | `` ` `` backtick continuation |
| `sleep 30` | `Start-Sleep -Seconds 30` |
| `echo "text"` | `Write-Host "text"` |
| `cmd \| grep pattern` | `cmd \| Select-String pattern` |
| `$(command)` subshell | `$(command)` or `$var = command` |
| Heredoc `<<EOF` | Here-string `@" ... "@` |
| `for i in {1..10}` | `1..10 \| ForEach-Object` |
| `watch command` | `kubectl ... --watch` or loop |

> **Note**: `kubectl`, `az`, and `helm` commands work identically in both shells. Commands that run inside containers (`kubectl exec -- bash -c "..."`) remain as Linux/bash since they execute in the container, not on your host.
