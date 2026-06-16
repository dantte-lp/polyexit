## What this changes

<!-- One sentence. Imperative mood. -->

## Why

<!-- Link to issue / discussion. If none, briefly justify. -->

## Validation

<!-- Tick what applies. -->

- [ ] `ansible-playbook --syntax-check playbooks/site.yml` passes
- [ ] `ansible-lint` passes
- [ ] Dry-run on at least one host: `ansible-playbook ... --check --diff`
- [ ] Idempotent: second apply reports `changed=0`
- [ ] If host_vars / role changed → docs updated (EN + RU)
- [ ] If mermaid diagram changed → validated locally with `mmdc`

## Notes for the reviewer

<!-- Anything that's not obvious from the diff. -->
