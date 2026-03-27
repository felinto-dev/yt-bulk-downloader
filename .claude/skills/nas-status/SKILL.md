---
name: nas-status
description: Mostra o status do Synology NAS: tarefas ativas do DownloadStation, espaço livre nos volumes e informações do sistema. Use quando o usuário quiser verificar o estado do NAS.
allowed-tools: Bash,Read
---

# /nas-status - Status do Synology NAS

Mostra informações de status do Synology NAS: downloads ativos, espaço em disco e info do sistema.

## Fluxo

### 1. Ler credenciais

### 2. Autenticar (sessão DownloadStation)

```bash
curl -s "{BASE_URL}?api=SYNO.API.Auth&version=6&method=login&account={USER}&passwd={PASS}&session=DownloadStation&format=sid"
```

### 4. Listar tarefas do DownloadStation

```bash
curl -s "{BASE_URL}?api=SYNO.DownloadStation.Task&version=1&method=list&additional=transfer,destination&_sid={SID}"
```

Parseie e mostre:
- Nome de cada tarefa
- Status (downloading, seeding, finished, paused, error)
- Progresso (%)
- Velocidade de download
- Destino

### 5. Informações do sistema

Autentique na sessão `FileStation` e consulte:

```bash
curl -s "{BASE_URL}?api=SYNO.DSM.Info&version=2&method=getinfo&_sid={SID}"
```

```bash
curl -s "{BASE_URL}?api=SYNO.Storage.CGI.Storage&version=1&method=load_info&_sid={SID}"
```

Mostre:
- Modelo do NAS
- Versão do DSM
- Uptime
- Temperatura (se disponível)
- Status de cada volume (tamanho total, usado, livre) — com barra visual

### 6. Logout

Faça logout de todas as sessões abertas.

### 7. Formatação da saída

Apresente de forma clara e resumida, por exemplo:

```
## Synology NAS — DS920+

**DSM**: 7.2.2 (uptime: 45 dias)
**Temp**: 42°C

### Volumes
| Volume | Usado | Livre | Uso |
|--------|-------|-------|-----|
| volume1 | 1.2 TB | 800 GB | 60% ████████░░░░ |

### DownloadStation (2 tarefas)
| Arquivo | Status | Progresso | Velocidade |
|---------|--------|-----------|------------|
| video.mp4 | Baixando | 67% | 12.5 MB/s |
| podcast.mp3 | Finalizado | 100% | — |
```

## Regras

- Sempre faça logout ao finalizar.
- Se o NAS não responder, informe o erro claramente.
