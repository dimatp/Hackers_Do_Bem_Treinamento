# Q5 — Relatório CTI

## O que é um Relatório de CTI?

**Cyber Threat Intelligence (CTI)** é o processo de coletar, analisar e comunicar informações sobre ameaças cibernéticas de forma que permitam tomada de decisão. Um Relatório de CTI traduz dados técnicos brutos (logs, hashes, IPs) em inteligência acionável — respondendo não só *o que aconteceu*, mas *quem fez*, *como fez* e *o que fazer agora*.

---

## Resumo do incidente

Em 12 de novembro de 2018, o host **Nalyvaiko-PC** (`172.17.1.129`), pertencente ao domínio `kyivartworks.com`, foi comprometido por meio de um documento Microsoft Word com macro VBA maliciosa. O arquivo foi entregue via site legítimo comprometido (`ifcingenieria.cl`).

Após a abertura do documento, a macro executou o payload `6169583.exe` — um dropper da família **Emotet**, compilado ~1h30 antes da infecção. O malware ativou um módulo de spam (11 conexões SMTP para 4 servidores externos) e realizou reconhecimento da infraestrutura de Active Directory, acessando o controlador de domínio `172.17.1.2` via SMB.

---

## Indicadores de Comprometimento (IoCs)

### Ativo comprometido

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| IP interno | `172.17.1.129` | `dhcp.log` | Alta |
| MAC address | `00:1e:67:4a:d7:5c` | `dhcp.log` | Alta |
| Hostname | `Nalyvaiko-PC` | `dhcp.log` | Alta |
| Domínio AD | `kyivartworks.com` | `dhcp.log` / `smb_files.log` | Alta |
| Controlador DC | `172.17.1.2` | `smb_files.log` | Alta |

### Vetor de entrada

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| URL maliciosa | `http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/` | `http.log` | Alta |
| Site hospedeiro | `ifcingenieria.cl` (comprometido) | `http.log` | Alta |
| Documento lure | `2018_11Details_zur_Transaktion.doc` | `files.log` | Alta |
| MIME type | `application/msword` | `http.log` | Alta |

### Payload

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| Executável | `6169583.exe` | `files.log` | Alta |
| Arquitetura | `PE32 I386 / WINDOWS_GUI` | `pe.log` | Alta |
| Compile timestamp | `12/11/2018 ~19:32 UTC` | `pe.log` | Alta |
| Hash MD5/SHA1 | Indisponível (PCAP truncado) | `files.log` | N/A |

### Infraestrutura C2 / Spam

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| IP C2/spam | `108.177.98.108` (Google/Gmail — 7x) | `smtp.log` | Alta |
| IP C2/spam | `185.93.245.68` | `smtp.log` | Alta |
| IP C2/spam | `41.204.202.10` | `smtp.log` | Alta |
| IP C2/spam | `123.125.50.11` | `smtp.log` | Alta |
| TLS C2 | `smtp.gmail.com` / `msmtp.bigticket.ae` | `x509.log` | Média |
| TLS C2 | `*.qiye.163.com` / `cloud.q1whosting.com` | `x509.log` | Média |

---

## Mapeamento MITRE ATT&CK

| ID | Tática | Técnica | Evidência |
|---|---|---|---|
| T1566.001 | Initial Access | Phishing: Spearphishing Link | URL `ifcingenieria.cl` → `http.log` |
| T1204.002 | Execution | User Execution: Malicious File | Abertura do `.doc` com macro |
| T1059.005 | Execution | Command & Scripting: Visual Basic | Macro VBA no Word |
| T1547.001 | Persistence | Registry Run Keys / Startup Folder | Comportamento padrão Emotet |
| T1071.001 | C2 | Application Layer Protocol: Web | Conexões SMTP — `smtp.log` |
| T1021.002 | Lateral Movement | Remote Services: SMB/Admin Shares | SMB FILE_OPEN no GPO — `smb_files.log` |
| T1078 | Defense Evasion | Valid Accounts | Emotet exfiltra credenciais locais |
| T1568 | C2 | Dynamic Resolution | Múltiplos IPs/domínios SMTP |

Referência: [MITRE ATT&CK — G0046 (Mealybug)](https://attack.mitre.org/groups/G0046/)

---

## Cyber Kill Chain

```
Reconnaissance    →  Alvo: kyivartworks.com / Good Money Financial
Weaponization     →  Macro VBA em .doc + 6169583.exe compilado no mesmo dia
Delivery          →  HTTP GET ifcingenieria.cl (site legítimo comprometido)
Exploitation      →  Usuário abre .doc → macro executa
Installation      →  6169583.exe instalado + persistência no registro
Command & Control →  11 conexões SMTP para Gmail, 163.com, bigticket.ae
Actions on Obj.   →  Spam + Reconhecimento AD + Propagação lateral potencial
```

---

## Recomendações

### Ações imediatas (0–24h)

- [ ] **Isolar** o host `172.17.1.129` da rede — realizar RAM dump antes do desligamento
- [ ] **Bloquear** os IPs C2 no firewall: `108.177.98.108`, `185.93.245.68`, `41.204.202.10`, `123.125.50.11`
- [ ] **Bloquear** o domínio `ifcingenieria.cl` no proxy corporativo e DNS filtering
- [ ] **Auditar** o controlador de domínio `172.17.1.2` — verificar GPOs modificados e novas contas desde 21:01 UTC
- [ ] **Resetar credenciais** de todas as contas com logon em `Nalyvaiko-PC`

### Contenção e erradicação (24–72h)

- [ ] **Varredura** de todos os hosts da rede `172.17.0.0/16` pelos IoCs confirmados
- [ ] **Revogar e regenerar** o GPO `{31B2F340-016D-11D2-945F-00C04FB984F9}` após auditoria
- [ ] **Verificar** se mensagens de spam foram efetivamente enviadas — notificar destinatários se confirmado
- [ ] **Reimagem** do host `Nalyvaiko-PC` após coleta forense completa

### Melhorias estruturais (médio prazo)

- [ ] **Desabilitar macros Office** para todos os usuários via GPO — permitir apenas macros assinadas digitalmente
- [ ] **Implementar inspeção de conteúdo** no proxy para identificar binários Office com macros
- [ ] **Implantar EDR** nos endpoints para detecção de comportamento pós-infecção
- [ ] **Revisar segmentação de rede** — estação comprometida acessou diretamente o DC via SMB sem restrições
- [ ] **Monitorar tráfego SMTP** de saída de estações de trabalho — conexões SMTP a partir de endpoints devem ser alertadas
