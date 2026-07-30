# Contributing

## How this repo works

`spec.yaml` is the source of truth. **Do not edit files in `vp-out/` or `qs-out/` directly** — those are generated outputs. Changes to them will be overwritten on the next `quickpat compose` run.

### Making changes

1. Edit `spec.yaml` (blocks, secrets, wiring, custom components)
2. Edit charts in `charts/` or `image-builder-charts/` for custom component changes
3. Regenerate outputs:
   ```bash
   quickpat compose spec.yaml
   quickpat compose spec.yaml --format qs
   ```
4. Commit everything: `spec.yaml`, `charts/`, and the regenerated `vp-out/` and `qs-out/`

### Adding a new block type

If `spec.yaml` needs a block type that doesn't exist in quickpat yet, see:
[docs/adding-block-types.md](https://github.com/atyronesmith/quickpat/blob/main/docs/adding-block-types.md)

### Adding a new inference provider

1. Add a secret entry to the `secrets:` section of `spec.yaml`
2. Add the corresponding entry to `charts/pattern-secrets/templates/externalsecret.yaml`
3. Add a row to the `values-secret.yaml.template`
4. Regenerate

### CI

CI runs on every push. It installs quickpat, runs `quickpat compose`, validates the generated VP with `quickpat validate`, lints all charts with Helm, and validates manifests with kubeconform. See `.github/workflows/compose.yml`.
