# Spectra/Spectrana Hermes Skill

This project tracks the Hermes skill source for `spectra-data-analysis`.

## Contents

- `SKILL.md` — primary Hermes skill definition.
- `templates/` — reusable prompts/templates referenced by the skill.

## Local Hermes source

The live local copy currently used by Hermes is:

```text
~/.hermes/skills/data-science/spectra-data-analysis/
```

This project folder is the git-tracked source mirror so updates can be committed locally.

## Sync workflow

After editing the tracked copy here, sync it back into Hermes with:

```bash
rsync -av --delete   /Users/nuelo/hermes-projects/spectrana-hermes-skill/   /Users/nuelo/.hermes/skills/data-science/spectra-data-analysis/   --exclude .git --exclude README.md --exclude .gitignore
```

After editing the live Hermes copy directly, refresh this tracked project with:

```bash
rsync -av --delete   /Users/nuelo/.hermes/skills/data-science/spectra-data-analysis/   /Users/nuelo/hermes-projects/spectrana-hermes-skill/   --exclude .git
```

## Validate

Basic frontmatter validation:

```bash
python3 - <<'PY'
from pathlib import Path
import re
content = Path('SKILL.md').read_text()
assert content.startswith('---')
match = re.search(r'\n---\s*\n', content[3:])
assert match, 'missing closing frontmatter delimiter'
frontmatter = content[3:match.start()+3]
assert re.search(r'^name:\s*spectra-data-analysis\s*$', frontmatter, re.M)
description = re.search(r'^description:\s*(.*)$', frontmatter, re.M)
assert description and description.group(1).strip()
assert len(description.group(1).strip().strip('"\'')) <= 1024
assert len(content) <= 100_000
print('SKILL.md validation OK')
PY
```


