# 🐧 Roteiro Operacional – Parte 1: Análise de PCAP com Zeek
**Caso:** Good Money Financial | **VM:** forenseLinux | **IP:** 192.168.98.10

---

## ✅ Checklist do Ambiente

Execute estes comandos logo ao acessar a VM, **antes de qualquer análise**.

```bash
# 1. Verificar versão do Zeek (caminho padrão de instalação compilada)
/opt/zeek/bin/zeek --version
```

> **Saída esperada:**
> ```
> Zeek version 4.x.x
> ```
> ⚠️ **Problema comum:** comando não encontrado. Causas e soluções:
> ```bash
> # a) Binário instalado em caminho diferente — localizar
> find / -name "zeek" -type f 2>/dev/null
>
> # b) Versão antiga da VM pode ter o nome 'bro' (antigo nome do projeto)
> which bro && bro --version
>
> # c) Adicionar ao PATH da sessão atual (evita digitar o caminho completo)
> export PATH=$PATH:/opt/zeek/bin
> echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc
> ```

```bash
# 2. Verificar zeek-cut disponível (deve estar junto com o zeek)
/opt/zeek/bin/zeek-cut --help | head -5
```

> **Saída esperada:** linha de uso do comando zeek-cut.
> ⚠️ Se não encontrar: `find /opt/zeek -name "zeek-cut"` — o binário fica no mesmo `bin/`.
> Se a VM tiver pacote Debian/RPM, pode estar em `/usr/bin/zeek-cut`.

```bash
# 3. Verificar existência e tamanho do arquivo de evidência
ls -lh /opt/estudodecaso/
```

> **Saída esperada:**
> ```
> -rw-r--r-- 1 root root  12M mai 10 2018  2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
> ```
> ⚠️ Se o diretório estiver vazio ou o arquivo ausente, não prossiga — acione o responsável pela VM.

```bash
# 4. Espaço em disco disponível — Zeek gera ~10% do tamanho do PCAP em logs
df -h ~/ && df -h /opt/estudodecaso/
```

> **Mínimo necessário:** 200 MB livres no diretório de trabalho.
> ⚠️ Disco cheio interrompe o Zeek silenciosamente no meio da análise, gerando logs incompletos.

```bash
# 5. RAM disponível (PCAPs com tráfego intenso consomem >500 MB)
free -h
```

```bash
# 6. Confirmar usuário atual
whoami && id
```

> ⚠️ Idealmente análise forense roda como usuário dedicado, não root.
> O `unzip` precisará de permissão de leitura em `/opt/estudodecaso/` — verifique com `ls -la`.

---

## ⚠️ Boas Práticas Forenses — Não Pule Esta Etapa

```
┌──────────────────────────────────────────────────────────────────────┐
│ NUNCA processe direto em /opt/estudodecaso/ — trabalhe em cópia.     │
│ DOCUMENTE timestamp de cada comando (use script ou tee).             │
│ NÃO altere permissões ou atributos dos arquivos extraídos.           │
│ REGISTRE os hashes ANTES e DEPOIS de cada operação.                  │
│ Qualquer anomalia inesperada: anote antes de tentar corrigir.        │
└──────────────────────────────────────────────────────────────────────┘
```

```bash
# Criar diretório de trabalho isolado com timestamp
mkdir -p ~/analise_caso && cd ~/analise_caso

# Iniciar gravação de sessão — tudo que aparecer no terminal vai para arquivo
script -a ~/analise_caso/sessao_$(date +%Y%m%d_%H%M%S).log

# Hashes do ZIP original — REGISTRAR no log de evidências
echo "[$(date)] Hash do arquivo original:"
md5sum /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
sha256sum /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
```

> 📝 Cole os valores no Log de Evidências no final deste roteiro.
> O `script` cria um arquivo `.log` com **tudo** que aparece no terminal, incluindo saídas de erro.
> Para encerrar a gravação: `exit`.

---

## 🔓 Extração do PCAP em `~/analise_caso/`

```bash
cd ~/analise_caso/

unzip -P infected \
  /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip \
  -d ~/analise_caso/
```

> **Saída esperada:**
> ```
> Archive:  /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
>   inflating: 2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap
> ```
> ⚠️ **Problema: `unsupported compression method 99`**
> O InfoZip nativo do Linux não suporta AES-256 (método 99). Solução:
> ```bash
> sudo apt-get install p7zip-full -y
> 7z x -Pinfected /opt/estudodecaso/*.zip -o~/analise_caso/
> ```

```bash
# Verificar extração e registrar hash do PCAP limpo
ls -lh ~/analise_caso/*.pcap
echo "[$(date)] Hash do PCAP extraído:"
md5sum ~/analise_caso/*.pcap
sha256sum ~/analise_caso/*.pcap
```

> 📝 Registre estes hashes. Se durante a análise surgir alguma dúvida sobre integridade, você consegue
> reprovar ou confirmar que o PCAP não foi modificado após a extração.

---

## 🦓 Processamento com Zeek

```bash
cd ~/analise_caso/

# Executar Zeek sobre o PCAP
/opt/zeek/bin/zeek -r *.pcap
```

> **Saída esperada:** nenhuma saída no terminal é o comportamento normal — Zeek processa silenciosamente.
> Se houver avisos como `weird` ou `truncated`, são informativos; não invalidam a análise.
>
> ⚠️ **Problema: `error in /opt/zeek/share/zeek/base/...`**
> Zeek pode reclamar de scripts faltando. Solução: rodar sem scripts locais:
> ```bash
> /opt/zeek/bin/zeek -r *.pcap --no-checksums
> # ou desativar scripts que causem erro pontualmente:
> /opt/zeek/bin/zeek -r *.pcap local 2>&1 | tee zeek_run.log
> ```
>
> ⚠️ **Problema: PCAP com nanosegundos (formato `LINKTYPE_...`)**
> ```bash
> # Verificar tipo de captura
> capinfos *.pcap | grep "File type"
> # Se reportar nanoseconds, converter antes:
> editcap -F pcap *.pcap pcap_convertido.pcap
> /opt/zeek/bin/zeek -r pcap_convertido.pcap
> ```

```bash
# Verificar logs gerados e tamanhos
ls -lhS *.log
```

> **Saída esperada (logs típicos de infecção via HTTP):**
> ```
> -rw-r--r-- 1 user user 2.1M  conn.log       ← todas as conexões
> -rw-r--r-- 1 user user 890K  http.log        ← requisições web
> -rw-r--r-- 1 user user 340K  dns.log         ← consultas DNS
> -rw-r--r-- 1 user user 120K  files.log       ← arquivos transferidos
> -rw-r--r-- 1 user user  45K  ssl.log         ← conexões TLS
> -rw-r--r-- 1 user user  12K  dhcp.log        ← negociações DHCP
> -rw-r--r-- 1 user user   8K  notice.log      ← alertas
> -rw-r--r-- 1 user user   4K  weird.log       ← anomalias
> ```
> 💡 Ausência de `dhcp.log` não significa falha — significa que não havia tráfego DHCP capturado.
> Ausência de `notice.log` significa que nenhum script de detecção disparou alerta.

```bash
# Ver os campos disponíveis em cada log (linha #fields)
grep "^#fields" dhcp.log
grep "^#fields" http.log
grep "^#fields" conn.log
grep "^#fields" files.log
```

> 💡 O `zeek-cut` será usado em todas as seções Q1 a Q4 deste roteiro para extrair colunas
> específicas de cada log. Confirme os campos disponíveis aqui antes de começar — eles variam
> entre versões do Zeek, e um nome de campo errado retorna colunas em branco silenciosamente, sem erro.

---

## 🔍 Q1 — Endereço MAC do cliente em 172.17.1.129

O `dhcp.log` registra a negociação DHCP completa: MAC, IP e hostname autodeclarado pelo cliente.
Antes de rodar o `zeek-cut`, confirme o nome exato do campo de IP nesta versão do Zeek —
em versões 4+ o campo é `client_addr`; em versões anteriores pode ser `assigned_addr`.

```bash
# Verificar campos disponíveis no dhcp.log
grep "^#fields" dhcp.log

# Extrair MAC — campo client_addr confirmado neste ambiente
/opt/zeek/bin/zeek-cut client_addr mac < dhcp.log   | grep "172.17.1.129" | sort -u
```

> **Saída obtida:**
> ```
> 172.17.1.129    00:1e:67:4a:d7:5c
> ```
> O prefixo OUI `00:1e:67` pertence à **Intel** — NIC física ou virtualização não-VMware.
> O `sort -u` elimina entradas duplicadas geradas pelas renovações de lease DHCP.

> 📸 **Print aqui** — Q1 respondido.

---

## 🔍 Q2 — Hostname do cliente em 172.17.1.129

O hostname autodeclarado pelo Windows está no campo `host_name` do mesmo `dhcp.log`.
Como Q1 e Q2 compartilham a mesma fonte, ambos podem ser extraídos com um único comando.

```bash
# MAC + Hostname em uma linha — saída limpa para o relatório
/opt/zeek/bin/zeek-cut client_addr mac host_name < dhcp.log   | grep "172.17.1.129" | sort -u
```

> **Saída obtida:**
> ```
> 172.17.1.129    00:1e:67:4a:d7:5c    Nalyvaiko-PC
> ```
> O hostname `Nalyvaiko-PC` sugere máquina de uso pessoal em ambiente corporativo — padrão de
> nomenclatura de usuário, não de ativo gerenciado por TI. Relevante para o relatório: indica
> possível BYOD ou estação sem gestão centralizada, o que explica a ausência de controles como
> bloqueio de macros Office.
> O domínio corporativo `kyivartworks.com` aparece na terceira linha do dhcp.log — registrar
> como contexto do ambiente para o Relatório CTI.

> 📸 **Print aqui** — Q1 e Q2 respondidos com este único comando.

---

## 🔍 Q3 — URL que retornou um documento Microsoft Word

O vetor de entrada mais comum em infecções via phishing de 2018 era um link para um `.doc` ou `.docx`
com macros VBA maliciosas. O campo `resp_mime_types` no `http.log` registra o Content-Type que o
servidor declarou na resposta — é onde vamos buscar.

```bash
# Listar todos os MIME types de respostas HTTP (panorama geral)
/opt/zeek/bin/zeek-cut resp_mime_types < http.log | grep -v "^#\|-" | sort | uniq -c | sort -rn
```

> **Saída obtida:**
> ```
>   3  text/plain
>   1  text/html
>   1  application/msword           ← documento Word legado (.doc)
> ```
> 💡 PCAP enxuto — poucos requests HTTP, o que é comum em capturas focadas no incidente.
> A ausência de `application/x-dosexec` indica que nenhum PE/EXE foi baixado diretamente via HTTP
> visível neste PCAP: o payload de segundo estágio pode ter chegado por outro protocolo,
> estar cifrado em TLS, ou ter sido entregue pelo próprio macro sem nova requisição HTTP.
> O `application/msword` confirma o vetor de entrada — prosseguir para extrair a URL.

```bash
# Extrair URL completa que entregou o Word doc
/opt/zeek/bin/zeek-cut host uri resp_mime_types < http.log \
  | grep -i "msword\|openxmlformats\|vnd.ms-word" \
  | awk '{print "http://" $1 $2, "\t[" $3 "]"}'
```

> **Saída obtida:**
> ```
> http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/    [application/msword]
> ```
> 💡 **Análise da URL:**
> - `ifcingenieria.cl` é um site legítimo de engenharia chileno — provavelmente **comprometido**
>   e usado como hospedeiro do malware, técnica comum para evadir reputação de domínio em proxies.
> - O caminho `/QpX8It/` tem aparência de identificador aleatório — típico de painéis de C2 ou
>   droppers que geram URLs únicas por campanha para dificultar takedown e correlação.
> - `Firmenkunden` é alemão para "clientes corporativos" — isca de engenharia social direcionada
>   a ambientes de negócios, reforça o perfil de ataque contra Good Money Financial.
> - A URL não termina em `.doc` — o arquivo é servido dinamicamente pelo servidor, o que impede
>   detecção simples por extensão em proxies sem inspeção de conteúdo.
>
> 📝 **Q3 — Resposta:** `http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/`

```bash
# Versão expandida — inclui IP de destino, status HTTP e tamanho do arquivo
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h host uri status_code response_body_len resp_mime_types < http.log \
  | grep -i "msword\|openxmlformats"
```

> ⚠️ **Problema: `resp_mime_types` vazio (`-`) mesmo o arquivo sendo um Word doc**
> Isso ocorre quando o servidor enviou o arquivo sem declarar Content-Type, ou declarou
> `application/octet-stream`. Alternativa: buscar pela extensão na URI ou pelo conteúdo nos files.log:
> ```bash
> # Buscar por extensão na URI
> /opt/zeek/bin/zeek-cut host uri < http.log | grep -i "\.doc\|\.docx\|\.dot\b"
>
> # Buscar por mime_type no files.log (Zeek identifica o tipo real pelo magic bytes)
> /opt/zeek/bin/zeek-cut tx_hosts rx_hosts mime_type filename < files.log \
>   | grep -i "msword\|openxml\|word"
> ```
> 💡 A diferença entre o MIME declarado pelo servidor e o MIME identificado pelo Zeek via magic bytes
> pode indicar **tentativa de ofuscação** — vale registrar no relatório se houver divergência.

> 📸 **Print aqui** — mostrar a URL completa reconstruída e o MIME type.

---

## 🔍 Q4 — Tipo de Infecção

> ✅ **Seção encerrada — análise concluída com base nos logs coletados.**
> Resultado: **Emotet** — Trojan bancário com módulo de spam e capacidade de propagação via AD.

---

### Passo 1 — Hashes dos arquivos transferidos

```bash
/opt/zeek/bin/zeek -r *.pcap \
  /opt/zeek/share/zeek/policy/frameworks/files/hash-all-files.zeek

/opt/zeek/bin/zeek-cut mime_type filename md5 sha1 < files.log \
  | grep -v "^#" | sort -u
```

> **Saída obtida:**
> ```
> application/msword    2018_11Details_zur_Transaktion.doc    -    -
> application/x-dosexec  6169583.exe                         -    -
> text/ini   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini   -   -
> ```
> 💡 Hashes retornaram `-` por truncamento do PCAP — arquivo capturado parcialmente.
> Os **nomes de arquivo** são IoCs suficientes para documentação.
>
> **Achados críticos:**
> - `2018_11Details_zur_Transaktion.doc` — alemão: *"Detalhes da Transação Nov 2018"*. Isca
>   financeira direcionada a ambiente corporativo, coerente com o alvo Good Money Financial.
> - `6169583.exe` — payload de segundo estágio com nome numérico aleatório, padrão de dropper
>   automatizado do Emotet.
> - `gpt.ini` via SMB — arquivo de Group Policy do AD `kyivartworks.com`, acesso ao controlador
>   de domínio para reconhecimento e possível propagação lateral.

---

### Passo 2 — Análise do executável (pe.log)

```bash
/opt/zeek/bin/zeek-cut ts machine compile_ts os subsystem is_exe < pe.log | grep -v "^#"
```

> **Saída obtida:**
> ```
> 1542056529.402998    I386    1542051170.000000    Windows 2000    WINDOWS_GUI    T
> ```
> 💡 **Análise:**
> - Arquitetura **I386** (32-bit) — compatibilidade máxima com parque Windows da época.
> - `compile_ts: 1542051170` = **12/11/2018 ~19:32 UTC** — compilado ~1h30 antes da infecção.
>   Binários Emotet eram compilados e redistribuídos no mesmo dia para evadir assinaturas AV.
> - Subsistema `WINDOWS_GUI` — PE com janela, não console — típico de banker trojan.

---

### Passo 3 — Módulo de spam (smtp.log)

```bash
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h mailfrom rcptto < smtp.log | grep -v "^#"
```

> **Saída obtida:**
> ```
> 1542061821   172.17.1.129   108.177.98.108   -   -
> 1542061823   172.17.1.129   108.177.98.108   -   -
> 1542061868   172.17.1.129   108.177.98.108   -   -
> 1542061901   172.17.1.129   185.93.245.68    -   -
> 1542061911   172.17.1.129   108.177.98.108   -   -
> 1542061942   172.17.1.129   41.204.202.10    -   -
> 1542062189   172.17.1.129   108.177.98.108   -   -
> 1542063260   172.17.1.129   108.177.98.108   -   -
> 1542063500   172.17.1.129   108.177.98.108   -   -
> 1542063746   172.17.1.129   108.177.98.108   -   -
> 1542063861   172.17.1.129   123.125.50.11    -   -
> ```
> 💡 A vítima realizou **11 conexões SMTP** para 4 servidores distintos ~1h após a infecção.
> `108.177.98.108` = Google/Gmail (7x). `mailfrom/rcptto` vazios indicam tentativas de relay
> ou autenticação falha. Este é o **módulo de spam do Emotet** em operação.

---

### Passo 4 — Reconhecimento de Active Directory (smb_files.log)

```bash
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h action name < smb_files.log | grep -v "^#"
```

> **Saída obtida:**
> ```
> 1542056481   172.17.1.129   172.17.1.2   SMB::FILE_OPEN   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
> 1542056492   172.17.1.129   172.17.1.2   SMB::FILE_OPEN   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
> 1542062242   172.17.1.129   172.17.1.2   SMB::FILE_OPEN   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
> 1542062853   172.17.1.129   172.17.1.2   SMB::FILE_OPEN   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
> ```
> 💡 `172.17.1.2` é o **controlador de domínio** de `kyivartworks.com`. O acesso ao `gpt.ini`
> (Group Policy Object) pode ser comportamento legítimo do Windows, mas a repetição em 4
> momentos distintos — incluindo após a ativação do módulo spam — indica reconhecimento
> sistemático da infraestrutura AD para possível propagação lateral.

---

### Passo 5 — Infraestrutura TLS (x509.log)

```bash
/opt/zeek/bin/zeek-cut id certificate.subject certificate.issuer < x509.log | grep -v "^#"
```

> **Saída obtida — servidores de email identificados:**
> ```
> CN=smtp.gmail.com          → Google Gmail
> CN=msmtp.bigticket.ae      → servidor email (UAE)
> CN=*.qiye.163.com           → NetEase China (163.com)
> CN=*.cpt2.host-h.net        → hosting genérico
> CN=cloud.q1whosting.com     → hosting genérico
> ```
> 💡 Diversidade de servidores SMTP em múltiplos países é característica do Emotet:
> o módulo de spam testa diferentes relays para maximizar entregabilidade das campanhas.

---

### 🧩 Linha do Tempo Confirmada

```
12/11/2018
~19:32 UTC  →  6169583.exe compilado (pe.log compile_ts = 1542051170)
~21:01 UTC  →  Doc Word baixado: ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/
~21:01 UTC  →  SMB FILE_OPEN no GPO do AD kyivartworks.com (172.17.1.2) — 1ª vez
~21:01 UTC  →  SMB FILE_OPEN no GPO — 2ª vez (11s depois)
~22:17 UTC  →  SMB FILE_OPEN — 3ª vez (pós-ativação do módulo spam)
~22:27 UTC  →  SMB FILE_OPEN — 4ª vez
~22:30 UTC  →  Módulo spam ativo: 11 conexões SMTP para Gmail, 163.com, bigticket.ae, etc.
```

> ✅ **Q4 — Resposta definitiva: Emotet**
> Trojan bancário distribuído via macro VBA em documento Office com tema financeiro em alemão,
> dropper com PE compilado no mesmo dia, módulo de spam ativo e reconhecimento de AD.
> Família dominante neste padrão exato em novembro de 2018.

## 📊 IoCs Confirmados para o Relatório CTI (Q5)

> ✅ Coletados e confirmados durante a análise. Exportar para anexo do relatório:
> ```bash
> # Salvar IoCs em arquivo para anexo
> cat > ~/analise_caso/iocs_emotet.txt << 'EOF'
> === CASO: Good Money Financial | Emotet | 12/11/2018 ===
>
> ATIVO COMPROMETIDO
> IP:        172.17.1.129
> MAC:       00:1e:67:4a:d7:5c
> Hostname:  Nalyvaiko-PC
> Domínio:   kyivartworks.com
>
> VETOR DE ENTRADA
> URL:       http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/
> Arquivo:   2018_11Details_zur_Transaktion.doc
> MIME:      application/msword
> Site:      ifcingenieria.cl (site legítimo comprometido, Chile)
>
> PAYLOAD
> Arquivo:   6169583.exe
> Tipo:      PE32 I386 WINDOWS_GUI
> Compilado: 12/11/2018 ~19:32 UTC
>
> C2 / SPAM (smtp.log)
> 108.177.98.108   (Google/Gmail  — 7 conexões SMTP)
> 185.93.245.68    (1 conexão SMTP)
> 41.204.202.10    (1 conexão SMTP)
> 123.125.50.11    (1 conexão SMTP)
>
> INFRAESTRUTURA TLS CONTATADA (x509.log)
> smtp.gmail.com
> msmtp.bigticket.ae
> *.qiye.163.com
> *.cpt2.host-h.net
> cloud.q1whosting.com
>
> RECONHECIMENTO AD (smb_files.log)
> DC:    172.17.1.2
> GPO:   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
> Acessos: 4x entre 21:01 e 22:27 UTC
> EOF
> ```

---

## 📑 Estrutura do Relatório Operacional de CTI (Q5)

> Seção seguinte — redigir com base nos IoCs confirmados acima.

```
1. RESUMO EXECUTIVO
   - Data da análise: 12/06/2026
   - PCAP analisado: 2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap
   - Zeek versão 6.0.4 | Ambiente: forenseLinux (192.168.98.10)
   - Ativo comprometido: Nalyvaiko-PC / 172.17.1.129 / 00:1e:67:4a:d7:5c
   - Tipo de infecção: Emotet — Trojan bancário com módulo de spam e reconhecimento AD
   - Impacto: comprometimento de estação em domínio corporativo kyivartworks.com;
     módulo de spam ativo; acesso ao controlador de domínio (172.17.1.2)

2. INDICADORES DE COMPROMETIMENTO (IoCs)
   | Tipo         | Valor                                        | Fonte         | Confiança |
   |--------------|----------------------------------------------|---------------|-----------|
   | IP vítima    | 172.17.1.129                                 | dhcp.log      | Alta      |
   | MAC          | 00:1e:67:4a:d7:5c                            | dhcp.log      | Alta      |
   | Hostname     | Nalyvaiko-PC                                 | dhcp.log      | Alta      |
   | Domínio AD   | kyivartworks.com                             | dhcp/smb      | Alta      |
   | URL maliciosa| http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/ | http.log  | Alta      |
   | Doc lure     | 2018_11Details_zur_Transaktion.doc           | files.log     | Alta      |
   | Payload      | 6169583.exe (PE32 I386, comp. 12/11/2018)    | files.log     | Alta      |
   | IP C2/spam   | 108.177.98.108                               | smtp.log      | Alta      |
   | IP C2/spam   | 185.93.245.68                                | smtp.log      | Alta      |
   | IP C2/spam   | 41.204.202.10                                | smtp.log      | Alta      |
   | IP C2/spam   | 123.125.50.11                                | smtp.log      | Alta      |
   | DC alvo      | 172.17.1.2 (GPO kyivartworks.com)            | smb_files.log | Alta      |
   | TLS C2       | smtp.gmail.com / msmtp.bigticket.ae          | x509.log      | Média     |
   | TLS C2       | *.qiye.163.com / cloud.q1whosting.com        | x509.log      | Média     |

3. LINHA DO TEMPO DO INCIDENTE
   ~19:32 UTC  Payload 6169583.exe compilado
   ~21:01 UTC  Download de 2018_11Details_zur_Transaktion.doc via HTTP
   ~21:01 UTC  Acesso SMB ao GPO do AD (172.17.1.2) — 1ª e 2ª vez
   ~22:17 UTC  Acesso SMB ao GPO — 3ª vez
   ~22:27 UTC  Acesso SMB ao GPO — 4ª vez
   ~22:30 UTC  Módulo spam ativo: 11 conexões SMTP para 4 servidores externos

4. RECOMENDAÇÕES IMEDIATAS
   - Isolar 172.17.1.129 imediatamente (preservar imagem de memória antes de desligar)
   - Bloquear IPs C2 no firewall: 108.177.98.108 / 185.93.245.68 / 41.204.202.10 / 123.125.50.11
   - Bloquear domínio ifcingenieria.cl no proxy corporativo
   - Verificar demais hosts da rede 172.17.0.0/16 com os mesmos IoCs (Emotet se propaga)
   - Auditar o controlador de domínio 172.17.1.2 — verificar GPOs e contas comprometidas
   - Resetar credenciais de todas as contas que fizeram logon em Nalyvaiko-PC
   - Desabilitar execução de macros Office via GPO em todo o domínio kyivartworks.com
   - Verificar se emails saíram via módulo spam — notificar destinatários se confirmado

5. REFERÊNCIAS
   - PCAP: 2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap
   - Ferramenta: Zeek 6.0.4
   - Script adicional: hash-all-files.zeek
   - Análise realizada em: 12/06/2026
   - Analista: [preencher]
```

---

## 🗒️ Log de Evidências

| # | Timestamp | Ação | Resultado | Arquivo |
|---|-----------|------|-----------|---------|
| 1 | 11/06/2026 | Hash ZIP original | (registrar) | sessao_20260611.log |
| 2 | 11/06/2026 | Hash PCAP extraído | (registrar) | sessao_20260611.log |
| 3 | 11/06/2026 | Zeek processado | Zeek 6.0.4, logs gerados | — |
| 4 | 11/06/2026 | **Q1** — MAC | `00:1e:67:4a:d7:5c` | print_q1.png |
| 5 | 11/06/2026 | **Q2** — Hostname | `Nalyvaiko-PC` | print_q2.png |
| 6 | 11/06/2026 | **Q3** — URL Word doc | `http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/` | print_q3.png |
| 7 | 12/06/2026 | **Q4** — Tipo de infecção | **Emotet** | print_q4.png |
| 8 | 12/06/2026 | IoCs exportados | 4 IPs C2, 1 URL, 2 arquivos, 1 DC | iocs_emotet.txt |
| 9 | — | **Q5** — Relatório CTI | A redigir | relatorio_cti.docx |

---

*Roteiro gerado em 11/06/2026 — Caso Good Money Financial*
