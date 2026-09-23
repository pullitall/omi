# omi-cli aģentiem

> Praktisks ceļvedis vidēm, kuras vada LLM modeļi (Claude Code, Cursor, jūsu pielāgotie boti).

## Kāpēc CLI ir ērts aģentiem

* **Stabils JSON līgums.** Karodziņš `--json` izvada derīgu JSON dokumentu uz stdout un *tikai* JSON dokumentu — bez progresa ziņojumiem, bez ielādes animācijām. Kļūdas tiek nosūtītas uz stderr formātā `{"error": "...", "detail": "..."}`.
* **Stabili izejas kodi (exit codes).** `0` kārtībā / `1` lietošanas kļūda / `2` autentifikācijas kļūda / `3` servera kļūda / `4` pārsniegts pieprasījumu limits / `5` nav atrasts. Aģenti var pieņemt lēmumus uz šo kodu pamata, neparsējot dabiskās valodas ziņojumus.
* **Nav interaktīvu uzvedņu bezgalvas režīmā (headless).** Destruktīvām komandām padodiet `--yes` (vai `-y`); padodiet `--api-key` vai iestatiet vides mainīgo `OMI_API_KEY`, lai izlaistu interaktīvo pieteikšanos.
* **Iecietīga atkārtošanas uzvedība.** Kļūdu kodi `429` un `5xx` tiek automātiski mēģināti vēlreiz ar eksponenciālu aizturi (backoff) pirms kļūdas parādīšanas.

## Autentifikācija (vienreizēja, veic cilvēks)

Lietotājs iegūst izstrādātāja API atslēgu no Omi tīmekļa lietotnes
(`https://app.omi.me` → Developer → API Keys) un palaiž vienu no šīm komandām:

```bash
omi auth login                          # interaktīva ielīmēšana; atslēga netiek saglabāta čaulas vēsturē
# vai
export OMI_API_KEY=omi_dev_...          # pagaidu, piemērots konteineriem
```

## Piecas darbības, ko aģenti veic visbiežāk

### 1. Atmiņu lasīšana (memories)

```bash
omi memory list --json --limit 50 | jq '.[] | {id, content, category}'
```

### 2. Atmiņas izveide

```bash
omi memory create --json "User prefers dark mode" --category lifestyle
```

### 3. Sarunu lasīšana

```bash
omi conversation list --json --limit 5 \
  | jq '.[] | {id, title: .structured.title, started_at}'
```

### 4. Atvērto uzdevumu lasīšana (action items)

```bash
omi action-item list --json --open
```

### 5. Uzdevuma atzīmēšana kā pabeigtu

```bash
omi action-item complete --json a1b2c3d4
```

## Vietējais darbvirsmas API (Local Desktop API)

Kad Omi Desktop iespējo savu vietējo API, aģenti var vaicāt ierīces ekrāna vēsturi, kopsavilkumus, SQL un uzdevumus, neizmantojot mākoņa API:

```bash
omi local configure --url http://127.0.0.1:47778 --token ...
# vai pagaidu sesijām:
export OMI_LOCAL_API_URL=http://127.0.0.1:47778
export OMI_LOCAL_TOKEN=...

omi --json local status
omi --json local tools
omi --json local call search_screen_history --args-json '{"query":"pricing page","days":7}'
omi --json local search-screen "pricing page" --days 7 --app Safari
omi --json local screenshot 123 --output /tmp/omi-shot.jpg
omi --json local sql "SELECT COUNT(*) AS screenshots FROM screenshots"
omi --json local task search "taxes" --include-completed
```

Pabeidziet vai dzēsiet uzdevumus tikai tad, kad lietotājs to nepārprotami pieprasa:

Pabeidziet vai dzēsiet uzdevumus tikai tad, kad lietotājs to skaidri pieprasa:

```bash
omi --json local task complete task_123
omi --json local task delete task_123 --yes
```

Komanda `omi local screenshot SCREENSHOT_ID --output PATH` ieraksta ekrānuzņēmumu diskā un skriptiem joprojām izvada JSON standarta izvadē (stdout). Ekrānuzņēmuma ID parasti iegūst no `local search-screen` vai SQL vaicājuma tabulā `screenshots`. Ja Desktop atgriež strukturētu kļūmi, piemēram, `screenshot_pending`, `screenshot_file_missing` vai `screenshot_chunk_corrupted`, JSON režīms saglabā laukus `reason`, `hint` un `screenshot_id` standarta kļūdu izvadē (stderr), lai aģenti varētu mēģināt vēlreiz ar vecāku ID vai ziņot par precīzu šķērsli. Pirms nodošanas redzes rīkiem pārbaudiet veiksmīgu izvadi ar `file PATH`.

## Praktisks piemērs: Python aģenta cikls

```python
import json
import subprocess
from typing import Any

def omi(*args: str) -> Any:
    """Invoke the omi CLI in JSON mode, raising on non-success exit codes."""
    result = subprocess.run(
        ["omi", "--json", *args],
        capture_output=True,
        text=True,
        check=False,
    )
    if result.returncode != 0:
        # The CLI prints structured errors to stderr in JSON mode:
        # {"error": "...", "detail": "..."}
        try:
            err = json.loads(result.stderr)
        except json.JSONDecodeError:
            err = {"error": result.stderr.strip()}
        raise RuntimeError(f"omi exited {result.returncode}: {err}")
    return json.loads(result.stdout) if result.stdout.strip() else None

# Read all open action items and mark anything older than 30 days complete.
from datetime import datetime, timedelta, timezone

cutoff = datetime.now(timezone.utc) - timedelta(days=30)
items = omi("action-item", "list", "--open")
for item in items or []:
    created = datetime.fromisoformat(item["created_at"].replace("Z", "+00:00"))
    if created < cutoff:
        omi("action-item", "complete", item["id"])
```

## Ātruma ierobežojumu apstrāde (Handling rate limits)

Atmiņas: 120/stundā. Sarunas: 25/stundā. Masveida izveide: 15/stundā.

```python
result = subprocess.run(["omi", "--json", "memory", "create", text], capture_output=True, text=True)
if result.returncode == 4:                             # rate limited
    err = json.loads(result.stderr)
    # err["detail"] looks like: "Retry in 12s. ..."
    time.sleep(parse_retry_window(err["detail"]) or 60)
```

## Noderīgi padomi (Tips)

* Izmantojiet `--profile <nosaukums>`, ja jūsu aģents pārvalda vairākus Omi kontus. Katram profilam ir savi akreditācijas dati un API bāze.
* Izmantojiet `--api-base http://localhost:8080` vietējai aizmugursistēmas testēšanai.
* Izmantojiet `OMI_LOCAL_API_URL` un `OMI_LOCAL_TOKEN`, lai aizstātu profila lokālos Desktop API iestatījumus vienai palaišanas reizei.
* Atkļūdošanai izmantojiet `--verbose` — tas reģistrē `METHOD path → status (Ns)` stderr, neietekmējot stdout, tādējādi JSON režīms paliek derīgs.
* Satura novirzīšanai sarunā caur konveijeru (pipe) izmantojiet `--text -`:
  ```bash
  cat meeting_notes.md | omi conversation create --text - --text-source other_text
  ```
