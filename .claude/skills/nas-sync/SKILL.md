---
name: nas-sync
description: Envia arquivos locais para uma pasta específica do Synology NAS. Use quando o usuário quiser sincronizar ou fazer upload de arquivos para o NAS.
allowed-tools: Bash,Read,AskUserQuestion
---

# /nas-sync - Enviar Arquivos para o Synology NAS

Faz upload de arquivos locais para uma pasta do Synology NAS via FileStation API.

## Argumento

O usuário fornece o caminho de uma pasta local (ex: `./canal-x` ou `/Users/felinto/Downloads/videos`).

## Fluxo

### 1. Verificar pasta local

Liste os arquivos que serão enviados:
```bash
ls -lh {PASTA_LOCAL}
```

Mostre ao usuário: "Encontrei **{N} arquivos** ({tamanho total}) em `{PASTA_LOCAL}`. Qual pasta no NAS deseja enviar? (padrão: `/volume1/video/youtube`)"

### 2. Ler credenciais

### 3. Autenticar

```bash
curl -s "{BASE_URL}?api=SYNO.API.Auth&version=6&method=login&account={USER}&passwd={PASS}&session=FileStation&format=sid"
```

Extraia o `sid` da resposta. Se der erro 106/107, re-autentique.

### 5. Criar pasta no NAS (se necessário)

```bash
curl -s "{BASE_URL}?api=SYNO.FileStation.CreateFolder&version=2&method=create&folder_path={PAI}&name={NOME_PASTA}&_sid={SID}"
```

### 6. Upload dos arquivos

Para cada arquivo na pasta local:
```bash
curl -s -X POST \
  -F "api=SYNO.FileStation.Upload" \
  -F "version=2" \
  -F "method=upload" \
  -F "path={DESTINO_NAS}" \
  -F "overwrite=true" \
  -F "create_parent=true" \
  -F "file=@{ARQUIVO_LOCAL}" \
  -F "_sid={SID}" \
  "{BASE_URL}"
```

Faça uploads em sequência (não paralelo) para evitar sobrecarga do NAS.

### 7. Logout

```bash
curl -s "{BASE_URL}?api=SYNO.API.Auth&version=6&method=logout&session=FileStation&_sid={SID}"
```

### 8. Resumo

Informe:
- Quantos arquivos foram enviados
- Qualquer erro encontrado
- Caminho completo no NAS

## Regras

- Sempre faça logout ao finalizar.
- Em caso de erro 106/107, re-autentique automaticamente.
- Para muitas pastas/arquivos, pergunte ao usuário antes de começar.
