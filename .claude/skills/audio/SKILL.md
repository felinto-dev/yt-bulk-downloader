---
name: audio
description: Extrai apenas o áudio de vídeos do YouTube em MP3 com metadata. Use quando o usuário quiser baixar só o áudio de um vídeo, playlist ou canal.
allowed-tools: Bash,Read,AskUserQuestion,Agent
---

# /audio - Extrair Áudio do YouTube

Extrai apenas o áudio de vídeos do YouTube com metadata e thumbnail embutidos.

## Argumento

O usuário fornece uma URL de vídeo, playlist ou canal do YouTube.

## Fluxo

### 1. Perguntar opções (se não especificado)

Pergunte ao usuário (use AskUserQuestion):
- **Enviar para o NAS?**: Sim/Não (padrão: Não)
- **Pasta no NAS** (se sim): Qual pasta destino (padrão: `/volume1/music/youtube`)

**Formato:** sempre MP3. Só pergunte sobre formato se o usuário explicitamente solicitar outro formato (ex: M4A).

### 2. Verificar yt-dlp e ffmpeg

```bash
which yt-dlp || echo "NOT_INSTALLED"
which ffmpeg || echo "NOT_INSTALLED"
```

Instale se necessário (`pip3 install yt-dlp`, `brew install ffmpeg`).

### 3. Informar

Antes de baixar, mostre o título do vídeo:
```bash
yt-dlp --print "%(title)s — %(duration_string)s" "URL"
```

Confirme com o usuário.

### 4. Baixar áudio

**Formato MP3:**
```bash
yt-dlp \
  --download-archive baixados.txt \
  -x --audio-format mp3 \
  --audio-quality 0 \
  --embed-thumbnail --add-metadata \
  -o "%(title)s.%(ext)s" \
  "URL"
```

**Formato M4A:**
```bash
yt-dlp \
  --download-archive baixados.txt \
  -x --audio-format m4a \
  --audio-quality 0 \
  --embed-thumbnail --add-metadata \
  -o "%(title)s.%(ext)s" \
  "URL"
```

> `--audio-quality 0` = melhor qualidade disponível.

### 5. Enviar para o NAS (se solicitado)

1. Autenticar, criar pasta se necessário, upload, logout.

### 6. Resumo final

Informe o que foi baixado e o destino final.

## Regras

- Use `--download-archive baixados.txt` para playlists/canais.
- Delegue para subagentes se o conteúdo for extenso.
