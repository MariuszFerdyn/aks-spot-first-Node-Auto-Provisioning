# Copilot Repository Instructions

- After implementing any change, update README.md so documentation stays accurate.
- Validate all YAML and JSON files before finalizing changes.
- Do not propose or create any GitHub Actions workflows for this repository.
- Use these commands for validation:

```bash
# JSON
python -m json.tool azuredeploy.json >/dev/null

# YAML (syntax check)
ruby -e 'require "psych"; ARGV.each { |f| Psych.parse_stream(File.read(f)) }' \
  01-nodepool-spot.yaml 02-nodepool-ondemand.yaml 03-deployment-app.yaml
```
