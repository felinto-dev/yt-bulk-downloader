---
name: transcript
description: Baixa a transcrição/legendas de um vídeo do YouTube e salva como .txt limpo. Use quando o usuário quiser a transcrição, legendas ou captions de um vídeo.
allowed-tools: Bash,Read,AskUserQuestion,Agent
---

# /transcript - Baixar Transcrição do YouTube

Baixa a transcrição de um vídeo do YouTube e converte para texto limpo.

## Argumento

O usuário fornece uma URL de vídeo do YouTube.

## Fluxo

### 1. Verificar yt-dlp

```bash
which yt-dlp || echo "NOT_INSTALLED"
```

Instale se necessário.

### 2. Listar legendas disponíveis

```bash
yt-dlp --list-subs "URL"
```

Verifique se há legendas manuais ou auto-geradas. Use essa informação para determinar:
- Quais idiomas estão disponíveis (o código varia: `pt`, `pt-BR`, `pt-PT`, `pt-orig`, `en`, etc.)
- Se há legendas manuais ou apenas auto-geradas
- **NUNCA** assuma o código de idioma. Sempre verifique com `--list-subs` antes de usar `--sub-lang`.

### 3. Baixar legenda

Prioridade: legendas manuais > auto-geradas.

**Tentar legenda manual primeiro:**
```bash
yt-dlp --write-sub --skip-download --sub-lang pt,en --output "transcript_temp" "URL"
```

**Se não houver manual, tentar auto-gerada:**
```bash
yt-dlp --write-auto-sub --skip-download --sub-langs "pt.*" --output "transcript_temp" "URL"
```

### 4. Converter VTT para texto limpo

**IMPORTANTE:** O YouTube repete cada linha de legenda dentro do mesmo cue (VTT). A conversão DEVE:
1. Remover linhas em branco
2. Deduplicar linhas consecutivas idênticas (não usar `seen` global — apenas consecutivas)
3. Adicionar quebras de parágrafo nos finais de sentença (`.`, `!`, `?`)

Obtenha o título do vídeo para o nome do arquivo:
```bash
yt-dlp --print "%(title)s" "URL" | tr '/' '_' | tr ':' '-' | tr '?' '' | tr '"' ''
```

Converta o VTT para texto limpo com este script Python:
```python
import re, sys

vtt_file = sys.argv[1]
out_file = sys.argv[2]

with open(vtt_file, 'r', encoding='utf-8', errors='replace') as f:
    lines = f.readlines()

# 1) Remove blank lines, timestamps, headers e tags HTML
cleaned = []
for line in lines:
    stripped = line.strip()
    if not stripped:
        continue
    if stripped.startswith('WEBVTT') or stripped.startswith('Kind:') or stripped.startswith('Language:'):
        continue
    if '-->' in stripped or re.match(r'^\d+$', stripped):
        continue
    text = re.sub(r'<[^>]*>', '', stripped)
    text = text.replace('&amp;', '&').replace('&gt;', '>').replace('&lt;', '<')
    if text:
        cleaned.append(text)

# 2) Deduplica linhas consecutivas
deduped = []
prev = None
for line in cleaned:
    if line != prev:
        deduped.append(line)
        prev = line

# 3) Adiciona quebras de parágrafo nos finais de sentença
result = []
for i, line in enumerate(deduped):
    if i > 0:
        prev_line = deduped[i - 1]
        if prev_line and prev_line[-1] in '.!?':
            result.append('')
    result.append(line)

with open(out_file, 'w', encoding='utf-8') as f:
    f.write('\n'.join(result) + '\n')
```

Uso:
```bash
python3 script.py "transcript_temp.pt.vtt" "{TITULO}.txt"
```

Remova o `.vtt` após a conversão.

### 5. Formatar com Gemini CLI

Use o Gemini CLI com o modelo flash para revisar e melhorar a transcrição:

```bash
gemini "Revise e formate o arquivo {TITULO}.txt que é uma transcrição de vídeo do YouTube. Siga estas instruções: 1) Corrija erros de ortografia e gramática. 2) Corrija nomes próprios e termos técnicos quando óbvio. 3) Melhore a pontuação e formatação. 4) Adicione parágrafos onde fizer sentido para organizar o conteúdo. 5) NÃO remova nem altere nenhum conteúdo relevante — o objetivo é limpar e formatar, não reescrever. 6) Mantenha o idioma original da transcrição. Grave o resultado de volta no arquivo {TITULO}.txt sobrescrevendo o conteúdo anterior." -m gemini-2.5-flash --yolo -o text
```

**Atenção**: Delegue esta etapa para um subagente para não poluir o contexto principal com o conteúdo da transcrição.

### 6. Resumo

Informe ao usuário:
- Arquivo salvo: `{TITULO}.txt`
- Fonte da legenda: manual ou auto-gerada
- Idioma

## Regras

- **NUNCA** despeje transcrições brutas no contexto. Salve em arquivo e retorne apenas resumo.
- Sempre dedupe as legendas auto-geradas (YouTube repete linhas com timestamps progressivos).
- Se não houver legendas disponíveis, pergunte ao usuário se quer usar Whisper (requer confirmação por ser pesado).
