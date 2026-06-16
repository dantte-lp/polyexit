---
name: Bug report
about: Report a reproducible problem with playbook, role, or template
labels: bug
---

## Summary

<!-- One sentence. -->

## Expected behavior

## Actual behavior

## Reproduction

1. Inventory state: (paste relevant host_vars or anonymized snippet)
2. Command: `ansible-playbook playbooks/site.yml ...`
3. Output: (relevant lines, anonymize IPs/hostnames if sensitive)

## Environment

- Controller: `ansible --version`
- Target OS: `cat /etc/os-release | head -2`
- Kernel: `uname -r`
- Repository commit: `git rev-parse HEAD`

## Anything else?
