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

Verifique se há legendas manuais ou auto-geradas.

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

Obtenha o título do vídeo para o nome do arquivo:
```bash
yt-dlp --print "%(title)s" "URL" | tr '/' '_' | tr ':' '-' | tr '?' '' | tr '"' ''
```

Converta o VTT para texto deduplicado:
```python
import re
seen = set()
with open('transcript_temp.pt.vtt', 'r') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith('WEBVTT') and not line.startswith('Kind:') and not line.startswith('Language:') and '-->' not in line:
            clean = re.sub('<[^>]*>', '', line)
            clean = clean.replace('&amp;', '&').replace('&gt;', '>').replace('&lt;', '<')
            if clean and clean not in seen:
                print(clean)
                seen.add(clean)
```

Salve como `{TITULO}.txt` e remova o `.vtt`.

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
