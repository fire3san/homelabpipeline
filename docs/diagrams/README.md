# Diagrams

This directory contains the rendered architecture diagrams for the homelab pipeline.

| File | Description |
|------|-------------|
| `architecture.mmd` | Mermaid source — Architecture at a glance |
| `architecture.svg` | Rendered SVG — Architecture at a glance |
| `architecture.png` | Rendered PNG — Architecture at a glance |
| `topology.mmd` | Mermaid source — Home-network topology |
| `topology.svg` | Rendered SVG — Home-network topology |
| `topology.png` | Rendered PNG — Home-network topology |

## Regenerating images

After editing a `.mmd` file, re-render the SVG and PNG:

```bash
npm install -g @mermaid-js/mermaid-cli

# Architecture diagram
mmdc -i architecture.mmd -o architecture.svg -b white --scale 2
mmdc -i architecture.mmd -o architecture.png -b white --scale 3

# Topology diagram
mmdc -i topology.mmd -o topology.svg -b white --scale 2
mmdc -i topology.mmd -o topology.png -b white --scale 3
```

If running as root, pass a Puppeteer config to allow `--no-sandbox`:

```bash
echo '{"args":["--no-sandbox","--disable-setuid-sandbox"]}' > /tmp/puppeteer-config.json
mmdc -i architecture.mmd -o architecture.svg -b white --scale 2 -p /tmp/puppeteer-config.json
```