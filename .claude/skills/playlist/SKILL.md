---
name: playlist
description: Baixa vídeos de uma playlist do YouTube com opção de qualidade e envio para o NAS. Use quando o usuário fornecer uma URL de playlist ou pedir para baixar uma playlist.
allowed-tools: Bash,Read,AskUserQuestion,Agent
---

# /playlist - Baixar Playlist do YouTube

Baixa vídeos de uma playlist do YouTube com controle de qualidade e opção de envio ao NAS.

## Argumento

O usuário fornece uma URL de playlist do YouTube (ex: `https://www.youtube.com/playlist?list=PLxxxxx`).

## Fluxo

### 1. Perguntar opções (se não especificado)

Pergunte ao usuário (use AskUserQuestion):
- **Qualidade do vídeo**: 1080p, 720p, 480p, melhor disponível (padrão: melhor disponível)
- **Enviar para o NAS?**: Sim/Não (padrão: Não)
- **Pasta no NAS** (se sim): Qual pasta destino (padrão: `/volume1/video/youtube`)
- **Faixa de vídeos** (opcional): Todos, primeiros N, ou intervalo (padrão: todos)

### 2. Verificar yt-dlp

```bash
which yt-dlp || echo "NOT_INSTALLED"
```

Se não instalado, instale com `pip3 install yt-dlp`.

### 3. Verificar ffmpeg

```bash
which ffmpeg || echo "NOT_INSTALLED"
```

Se não instalado, instale com `brew install ffmpeg`.

### 4. Informar a playlist

Antes de baixar, mostre ao usuário:
```bash
yt-dlp --print "Playlist: %(playlist_title)s (%(playlist_count)s vídeos)" "PLAYLIST_URL" --flat-playlist
```

Confirme: "Vou baixar **{N} vídeos** da playlist **{nome}** na qualidade **{qualidade}**. Continuar?"

### 5. Baixar

**Se o tipo escolhido for "Apenas transcrição" ou envolver transcrição:**
- Liste todos os vídeos da playlist com `--flat-playlist --print "%(id)s"`.
- Para cada vídeo, delegue a um subagente que invoque a skill `/transcript` com `https://www.youtube.com/watch?v={VIDEO_ID}`, passando a pasta destino e outras opções relevantes.
- Use `--download-archive` para rastrear quais vídeos já tiveram transcrição baixada.
- Não baixe manualmente nem converta VTT — deixe a skill `/transcript` cuidar da deduplicação e formatação.

**Se o tipo escolhido for vídeo ou áudio:**

Mapeamento de qualidade:
- "melhor disponível": `-f "bestvideo+bestaudio/best"`
- "1080p": `-f "bestvideo[height<=1080]+bestaudio/best[height<=1080]"`
- "720p": `-f "bestvideo[height<=720]+bestaudio/best[height<=720]"`
- "480p": `-f "bestvideo[height<=480]+bestaudio/best[height<=480]"`

Se o usuário especificou faixa (ex: primeiros 10):
```bash
yt-dlp \
  --download-archive baixados.txt \
  -f "{FORMAT}" \
  -I 1:{N} \
  -o "%(playlist_title)s/%(playlist_index)03d - %(title)s.%(ext)s" \
  --embed-thumbnail --add-metadata \
  --concurrent-downloads 3 \
  "PLAYLIST_URL"
```

Sem faixa (todos):
```bash
yt-dlp \
  --download-archive baixados.txt \
  -f "{FORMAT}" \
  -o "%(playlist_title)s/%(playlist_index)03d - %(title)s.%(ext)s" \
  --embed-thumbnail --add-metadata \
  --concurrent-downloads 3 \
  "PLAYLIST_URL"
```

### 6. Enviar para o NAS (se solicitado)

1. Autenticar no FileStation:
```bash
curl -s "{BASE_URL}?api=SYNO.API.Auth&version=6&method=login&account={USER}&passwd={PASS}&session=FileStation&format=sid"
```

2. Criar pasta no NAS se não existir.

3. Upload dos arquivos usando `SYNO.FileStation.Upload`.

4. Logout da sessão.

### 7. Resumo final

Informe ao usuário:
- Quantos vídeos foram baixados
- Qualquer erro encontrado
- Se foi enviado para o NAS, confirme o destino

## Regras

- Use `--download-archive baixados.txt` sempre para evitar duplicatas.
- Delegue processamento de conteúdo para subagentes.
