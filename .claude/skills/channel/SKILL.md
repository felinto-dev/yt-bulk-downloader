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

### 1. Investigar estado do projeto

Antes de qualquer pergunta, inspecione o estado atual:
- Liste pastas e arquivos existentes para entender a organização.
- Verifique se já existe uma pasta para o canal em questão.
- Identifique padrões de nomenclatura e qualidade usados anteriormente.
- Verifique se existem arquivos `baixados-*.txt` e quantos vídeos já foram baixados.

### 2. Obter informações do canal

**IMPORTANTE:** NUNCA use `--flat-playlist` para obter metadados do canal (`%(channel)s`, `%(channel_id)s`). O `--flat-playlist` pula o download de metadados e esses campos retornam "NA".

Fluxo correto em **dois passos**:

**Passo A** — Obter o ID de um vídeo do canal (com `--flat-playlist`, que é rápido):
```bash
yt-dlp --flat-playlist --print "%(id)s" --playlist-items 1 "CHANNEL_URL"
```

**Passo B** — Usar esse vídeo para obter os metadados completos (sem `--flat-playlist`):
```bash
yt-dlp --print "%(channel)s | %(channel_id)s" "https://www.youtube.com/watch?v={VIDEO_ID}"
```

**Passo C** — Derivar a URL de uploads (mais rápida que a URL do canal):
```bash
# Troque o segundo caractere do channel ID de 'C' para 'U'
# Ex: UCPKZrWKhJILytzjACeUfveg -> UUPKZrWKhJILytzjACeUfveg
UPLOADS_URL="https://www.youtube.com/playlist?list=UU{resto_do_id}"
```

**Passo D** — Contar total de vídeos na playlist de uploads:
```bash
yt-dlp --flat-playlist --print "%(id)s" "$UPLOADS_URL" | wc -l
```

### 3. Perguntar opções (se não especificado)

Use `AskUserQuestion` com sugestões baseadas na investigação do passo 1:
- **Tipo de mídia**: Vídeo (MP4), Apenas áudio (MP3/M4A), Apenas transcrição (.txt), Áudio + transcrição, Vídeo + transcrição
- **Qualidade** (se aplicável): 1080p, 720p, 480p, melhor disponível. Sugira a mesma qualidade usada anteriormente se houver padrão.
- **Enviar para o NAS?**: Sim/Não
- **Pasta no NAS** (se sim): Sugira o mesmo destino usado anteriormente se houver padrão.

### 4. Verificar yt-dlp e ffmpeg

```bash
which yt-dlp || echo "NOT_INSTALLED"
which ffmpeg || echo "NOT_INSTALLED"
```

Se não instalado, instale com `pip3 install yt-dlp` ou `brew install ffmpeg`.

### 5. Confirmar o plano

Antes de executar, repita ao usuário: canal, quantidade de vídeos, tipo de mídia, qualidade, pasta destino, NAS sim/não. Peça confirmação.

### 6. Baixar

**Se o tipo escolhido for "Apenas transcrição" ou envolver transcrição:**
- **SEMPRE** delegue para a skill `/transcript`. A skill `/transcript` é a fonte da verdade para tudo relacionado a transcrição — nunca implemente lógica de transcrição nesta skill.
- Liste todos os vídeos da playlist de uploads com `--flat-playlist --print "%(id)s"`.
- Para cada vídeo, delegue a um subagente que invoque a skill `/transcript` com `https://www.youtube.com/watch?v={VIDEO_ID}`, passando a pasta destino e outras opções relevantes.
- Use `--download-archive` para rastrear quais vídeos já tiveram transcrição baixada.

**Se o tipo escolhido for vídeo ou áudio:**

Mapeamento de qualidade (vídeo):
- "melhor disponível": `-f "bestvideo+bestaudio/best"`
- "1080p": `-f "bestvideo[height<=1080]+bestaudio/best[height<=1080]"`
- "720p": `-f "bestvideo[height<=720]+bestaudio/best[height<=720]"`
- "480p": `-f "bestvideo[height<=480]+bestaudio/best[height<=480]"`

Mapeamento de qualidade (áudio):
- "mp3": `-x --audio-format mp3`
- "m4a": `-x --audio-format m4a`

**Use sempre a URL de uploads** obtida no passo 2C.

Nome do arquivo de archive: `baixados-{qualidade}.txt` (ex: `baixados-480p.txt`, `baixados-mp3.txt`, `baixados-720p.txt`).
Coloque o archive **dentro da pasta do canal**, nunca na raiz do projeto.

Formato de saída: sempre use `%(title)s.%(ext)s` (apenas título + extensão).

Comando de download (vídeo):
```bash
yt-dlp \
  --download-archive "{CANAL_DIR}/baixados-{QUALIDADE}.txt" \
  -f "{FORMAT}" \
  --merge-output-format mp4 \
  -o "{CANAL_DIR}/%(title)s.%(ext)s" \
  --embed-thumbnail --add-metadata \
  "$UPLOADS_URL"
```

Comando de download (áudio):
```bash
yt-dlp \
  --download-archive "{CANAL_DIR}/baixados-mp3.txt" \
  -x --audio-format mp3 \
  -o "{CANAL_DIR}/%(title)s.%(ext)s" \
  --embed-thumbnail --add-metadata \
  "$UPLOADS_URL"
```

### 7. Enviar para o NAS (se solicitado)

1. Autenticar no FileStation:
```bash
curl -s "{BASE_URL}?api=SYNO.API.Auth&version=6&method=login&account={USER}&passwd={PASS}&session=FileStation&format=sid"
```

2. Criar pasta no NAS se não existir (use `SYNO.FileStation.CreateFolder`).

3. Upload dos arquivos usando `curl -F "file=@{ARQUIVO}"` com `SYNO.FileStation.Upload`.

4. Logout da sessão.

### 8. Resumo final

Informe ao usuário:
- Quantos vídeos foram baixados
- Qualquer erro encontrado
- Se foi enviado para o NAS, confirme o destino

## Regras

- **NUNCA** use `--flat-playlist` para obter metadados (canal, uploader, etc.). Use apenas para listar IDs e títulos rapidamente.
- Use `--download-archive` sempre para evitar duplicatas. Nome do arquivo: `baixados-{qualidade}.txt`, dentro da pasta do canal.
- Use a URL de uploads (`UU...`) sempre que possível — é mais rápida e encontra mais vídeos.
- Saída sempre em MP4 (use `--merge-output-format mp4`).
- Nome do arquivo: apenas `%(title)s.%(ext)s`.
- **NUNCA** use `--concurrent-downloads` (flag não existe no yt-dlp atual).
- Delegue processamento de conteúdo (metadados grandes) para subagentes para preservar contexto.
- Em caso de interrupção, o comando pode ser re-executado e vai continuar de onde parou graças ao archive.
