---
name: channel
description: Baixa todos os vídeos de um canal do YouTube com --download-archive para evitar duplicatas. Opcionalmente envia para o Synology NAS. Use quando o usuário fornecer uma URL de canal do YouTube ou pedir para baixar todos os vídeos de um canal.
allowed-tools: Bash,Read,AskUserQuestion,Agent
---

# /channel - Baixar Canal do YouTube

Baixa todos os vídeos de um canal do YouTube, evitando duplicatas com `--download-archive`.

## Argumento

O usuário fornece uma URL de canal do YouTube (ex: `https://www.youtube.com/@canal/videos` ou `https://www.youtube.com/c/Canal/videos`).

## Fluxo

### 1. Perguntar opções (se não especificado)

Pergunte ao usuário (use AskUserQuestion):
- **Qualidade do vídeo**: 1080p, 720p, 480p, melhor disponível (padrão: melhor disponível)
- **Enviar para o NAS?**: Sim/Não (padrão: Não)
- **Pasta no NAS** (se sim): Qual pasta destino (padrão: `/volume1/video/youtube`)

### 2. Verificar yt-dlp

```bash
which yt-dlp || echo "NOT_INSTALLED"
```

Se não instalado, instale com `pip3 install yt-dlp`.

### 3. Verificar ffmpeg (para merge de vídeo+áudio)

```bash
which ffmpeg || echo "NOT_INSTALLED"
```

Se não instalado, instale com `brew install ffmpeg`.

### 4. Informar o canal

Antes de baixar, mostre ao usuário:
```bash
yt-dlp --print "%(channel)s — %(channel_id)s" "CHANNEL_URL"
```

Confirme: "Vou baixar todos os vídeos do canal **{nome}** na qualidade **{qualidade}**. Arquivo de controle: `baixados.txt`. Podem haver muitos vídeos. Continuar?"

### 5. Baixar

Mapeamento de qualidade:
- "melhor disponível": `-f "bestvideo+bestaudio/best"`
- "1080p": `-f "bestvideo[height<=1080]+bestaudio/best[height<=1080]"`
- "720p": `-f "bestvideo[height<=720]+bestaudio/best[height<=720]"`
- "480p": `-f "bestvideo[height<=480]+bestaudio/best[height<=480]"`

Comando de download:
```bash
yt-dlp \
  --download-archive baixados.txt \
  -f "{FORMAT}" \
  -o "%(channel)s/%(upload_date)s - %(title)s.%(ext)s" \
  --embed-thumbnail --add-metadata \
  --concurrent-downloads 3 \
  "CHANNEL_URL"
```

### 6. Enviar para o NAS (se solicitado)

1. Autenticar no FileStation:
```bash
curl -s "{BASE_URL}?api=SYNO.API.Auth&version=6&method=login&account={USER}&passwd={PASS}&session=FileStation&format=sid"
```

2. Criar pasta no NAS se não existir (use `SYNO.FileStation.CreateFolder`).

3. Upload dos arquivos usando `curl -F "file=@{ARQUIVO}"` com `SYNO.FileStation.Upload`.

4. Logout da sessão.

### 7. Resumo final

Informe ao usuário:
- Quantos vídeos foram baixados
- Qualquer erro encontrado
- Se foi enviado para o NAS, confirme o destino

## Regras

- Use `--download-archive baixados.txt` sempre para evitar duplicatas.
- Delegue processamento de conteúdo (metadados grandes) para subagentes para preservar contexto.
- Em caso de interrupção, o comando pode ser re-executado e vai continuar de onde parou graças ao archive.
