# omi-cli para Agentes de IA

> Guia prático para ambientes orientados a LLM (Claude Code, Cursor, bots customizados).

## Por que a CLI é Amigável para Agentes

* **Contrato JSON Estável.** A flag `--json` emite JSON válido para stdout e *apenas* JSON — sem logs de status ou spinners. Erros saem em stderr no formato `{"error": "...", "detail": "..."}`.
* **Códigos de Saída Estáveis.** `0` ok / `1` erro de uso / `2` erro de autenticação / `3` erro de servidor / `4` limite de taxa excedido / `5` não encontrado. Agentes podem ramificar logicamente por código de saída sem regex em mensagens em linguagem natural.
* **Sem Prompts Interativos em Modo Headless.** Passe `--yes` (ou `-y`) para comandos destrutivos; passe `--api-key` ou defina `OMI_API_KEY` para evitar login interativo via navegador.
* **Comportamento de Retry Resiliente.** Respostas `429` e `5xx` realizam tentativas automáticas com backoff exponencial antes de propagar falhas.

## Autenticação (Etapa Única Humana)

O usuário obtém uma chave de API de desenvolvedor no app web Omi
(`https://app.omi.me` → Developer → API Keys) e executa:

```bash
omi auth login                          # colar interativo; a chave não vaza no histórico da shell
# ou
export OMI_API_KEY=omi_dev_...          # efêmero, ideal para contêineres e CI/CD
```

## As Cinco Ações Mais Comuns de Agentes

### 1. Ler Memórias

```bash
omi memory list --json --limit 50 | jq '.[] | {id, content, category}'
```

### 2. Criar uma Memória

```bash
omi memory create --json "User prefers dark mode" --category lifestyle
```

### 3. Ler Conversas

```bash
omi conversation list --json --limit 5 \
  | jq '.[] | {id, title: .structured.title, started_at}'
```

### 4. Ler Itens de Ação Pendentes

```bash
omi action-item list --json --open
```

### 5. Concluir um Item de Ação

```bash
omi action-item complete --json a1b2c3d4
```

## API Local de Desktop (Local Desktop API)

Quando o Omi Desktop expõe sua API local, os agentes podem consultar o histórico de tela do dispositivo, resumos, SQL e tarefas sem usar a API em nuvem:

```bash
omi local configure --url http://127.0.0.1:47778 --token ...
# ou em sessões efêmeras:
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

Conclua ou exclua tarefas somente quando solicitado explicitamente pelo usuário:

Apenas conclua ou exclua tarefas quando o usuário solicitar expressamente:

```bash
omi --json local task complete task_123
omi --json local task delete task_123 --yes
```

`omi local screenshot SCREENSHOT_ID --output PATH` grava a captura de tela no disco e ainda imprime JSON no stdout para scripts. O ID da captura de tela geralmente vem de `local search-screen` ou de consulta SQL na tabela `screenshots`. Se o Desktop retornar uma falha estruturada como `screenshot_pending`, `screenshot_file_missing` ou `screenshot_chunk_corrupted`, o modo JSON preserva os campos `reason`, `hint` e `screenshot_id` no stderr para que os agentes possam tentar novamente com um ID mais antigo ou relatar o bloqueio exato. Valide as saídas bem-sucedidas com `file PATH` antes de passá-las para ferramentas de visão.

## Exemplo Prático: Loop de Agente Python

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

## Gerenciamento de Limites de Taxa (Handling rate limits)

Memórias: 120/hora. Conversas: 25/hora. Criações em lote: 15/hora.

```python
result = subprocess.run(["omi", "--json", "memory", "create", text], capture_output=True, text=True)
if result.returncode == 4:                             # rate limited
    err = json.loads(result.stderr)
    # err["detail"] looks like: "Retry in 12s. ..."
    time.sleep(parse_retry_window(err["detail"]) or 60)
```

## Dicas Úteis (Tips)

* Use `--profile <nome>` se o seu agente gerencia várias contas Omi. Cada perfil tem sua própria credencial e base de API.
* Use `--api-base http://localhost:8080` para testes locais com o backend.
* Use `OMI_LOCAL_API_URL` e `OMI_LOCAL_TOKEN` para substituir as configurações de Desktop API local do perfil em uma única execução.
* Use `--verbose` para depuração — registra `METHOD path → status (Ns)` no stderr sem afetar o stdout, mantendo o modo JSON válido.
* Para enviar conteúdo para uma conversa por pipe, use `--text -`:
  ```bash
  cat meeting_notes.md | omi conversation create --text - --text-source other_text
  ```
