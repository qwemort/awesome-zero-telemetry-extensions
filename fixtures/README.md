# Test fixtures

Open each file in your browser **from a web server** (extensions often don't run on `file://` URLs by default):

```bash
python3 -m http.server 8000
# then open http://localhost:8000/bigint.json
```

| File | What it tests | Pass |
|---|---|---|
| `bigint.json` | Integers above 2^53 | Every number shown exactly as in the file |
| `xss.json` | HTML / script in keys and values | Shown as text, no alert, `javascript:` not clickable |
| `big.json` (generate below) | ~20 MB, 120,000 records | Opens in a few seconds, tab stays responsive, search works |

Generate `big.json`:

```bash
python3 - <<'PY'
import json, random
rows = [{"id": i, "guid": "%032x" % random.getrandbits(128), "name": f"Name {i}",
         "value": random.random(), "tags": [f"t{i % 7}", "x"],
         "obj": {"a": i, "b": {"c": f"deep {i}"}}} for i in range(120000)]
json.dump(rows, open("big.json", "w"))
PY
```

Record the browser, version and extension version with your result.
