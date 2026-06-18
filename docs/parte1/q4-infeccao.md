# Q4 — Tipo de infecção

## A pergunta

> Que tipo de infecção ocorreu neste pcap?

## Resposta

**Emotet** — Trojan bancário com módulo de spam e capacidade de reconhecimento de Active Directory.

---

## Como chegamos a essa conclusão

A identificação não parte de um único indicador, mas da **convergência de múltiplos comportamentos** observados nos logs. Cada passo abaixo adiciona uma camada de evidência.

---

### Passo 1 — Arquivos transferidos (`files.log`)

```bash
/opt/zeek/bin/zeek -r *.pcap \
  /opt/zeek/share/zeek/policy/frameworks/files/hash-all-files.zeek

/opt/zeek/bin/zeek-cut mime_type filename md5 sha1 < files.log \
  | grep -v "^#" | sort -u
```

```
application/msword      2018_11Details_zur_Transaktion.doc   -   -
application/x-dosexec   6169583.exe                          -   -
text/ini                kyivartworks.com\Policies\{31B2F340...}\gpt.ini   -   -
```

Três arquivos notáveis:

- **`.doc` em alemão** com temática financeira — isca de engenharia social
- **`6169583.exe`** — nome puramente numérico, padrão de payload gerado automaticamente
- **`gpt.ini` via SMB** — acesso ao Group Policy Object do Active Directory

Os hashes aparecem como `-` porque o PCAP foi truncado — os arquivos foram identificados pelos magic bytes, mas não capturados integralmente.

---

### Passo 2 — Análise do executável (`pe.log`)

```bash
/opt/zeek/bin/zeek-cut ts machine compile_ts os subsystem is_exe < pe.log \
  | grep -v "^#"
```

```
1542056529.402998   I386   1542051170.000000   Windows 2000   WINDOWS_GUI   T
```

- `compile_ts: 1542051170` = **12/11/2018 ~19:32 UTC**
- Download ocorreu em ~21:01 UTC → **gap de ~1h30 entre compilação e infecção**
- Arquitetura I386 (32-bit) para compatibilidade máxima
- Subsistema `WINDOWS_GUI` — executa sem janela de console visível

!!! info "Por que o gap de 1h30 importa?"
    O Emotet compila binários sob demanda, pouco antes de cada campanha. Isso serve a dois propósitos: o hash nunca foi visto antes nos bancos de assinaturas de antivírus, e o binário pode ser descartado rapidamente se a infraestrutura for comprometida. Um executável compilado no mesmo dia da infecção, com gap de horas, é uma assinatura comportamental característica do Emotet.

---

### Passo 3 — Módulo de spam (`smtp.log`)

```bash
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h mailfrom rcptto < smtp.log \
  | grep -v "^#"
```

**11 conexões SMTP** originadas de `172.17.1.129` para 4 servidores externos, iniciadas ~1h30 após a infecção:

| IP destino | Conexões | Contexto |
|---|---|---|
| `108.177.98.108` | 7x | Google/Gmail — relay legítimo abusado |
| `185.93.245.68` | 1x | — |
| `41.204.202.10` | 1x | — |
| `123.125.50.11` | 1x | Rede 163.com (China) |

Estações de trabalho não devem iniciar conexões SMTP diretamente. Esse comportamento é característico do **módulo de spam do Emotet**.

---

### Passo 4 — Reconhecimento de AD (`smb_files.log`)

```bash
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h action name < smb_files.log \
  | grep -v "^#"
```

```
21:01 UTC   172.17.1.129 → 172.17.1.2   SMB::FILE_OPEN   kyivartworks.com\Policies\{31B2F340...}\gpt.ini
21:01 UTC   172.17.1.129 → 172.17.1.2   SMB::FILE_OPEN   (11s depois)
22:17 UTC   172.17.1.129 → 172.17.1.2   SMB::FILE_OPEN
22:27 UTC   172.17.1.129 → 172.17.1.2   SMB::FILE_OPEN
```

4 acessos ao GPO padrão do domínio `kyivartworks.com` no controlador de domínio `172.17.1.2`. O GUID `{31B2F340-016D-11D2-945F-00C04FB984F9}` identifica o **Default Domain Policy** — presente em todo ambiente AD com configuração padrão.

---

### Passo 5 — Infraestrutura TLS (`x509.log`)

```bash
/opt/zeek/bin/zeek-cut id certificate.subject < x509.log | grep -v "^#"
```

Servidores contatados via TLS: `smtp.gmail.com`, `msmtp.bigticket.ae`, `*.qiye.163.com`, `cloud.q1whosting.com`

A diversidade geográfica dos servidores SMTP (EUA, Emirados Árabes, China) é característica da infraestrutura distribuída do Emotet — projetada para resistir a bloqueios por geolocalização.

---

## Linha do tempo consolidada

| Timestamp (UTC) | Evento |
|---|---|
| 12/11/2018 19:32 | `6169583.exe` compilado |
| 12/11/2018 21:01 | Download do `.doc` via `ifcingenieria.cl` |
| 12/11/2018 21:01 | Macro executa → payload instalado → 1º acesso SMB ao GPO |
| 12/11/2018 21:01 | 2º acesso SMB ao GPO (11s depois) |
| 12/11/2018 22:17 | 3º acesso SMB ao GPO |
| 12/11/2018 22:27 | 4º acesso SMB ao GPO |
| 12/11/2018 22:30 | Módulo de spam ativo — 11 conexões SMTP |
