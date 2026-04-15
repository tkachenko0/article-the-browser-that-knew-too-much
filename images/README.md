# Diagram generation

## Regenerate everything

```bash
cd images
for f in csrf-flow pkce-flow bff-architecture bff-sequence; do
  npx -y -p @mermaid-js/mermaid-cli mmdc \
    -i "$f.mmd" \
    -o "$f.png" \
    -p puppeteer-config.json \
    -b white \
    -w 1600
done
```
