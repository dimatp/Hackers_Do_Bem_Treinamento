# Configuração do Ambiente de Lab

## Visão geral

O estudo de caso utiliza duas VMs distintas, cada uma com sua ferramenta e evidência:

| VM | IP | SO | Ferramenta | Caso |
|---|---|---|---|---|
| `forenseLinux` | 192.168.98.10 | Linux (Ubuntu) | Zeek 6.0.4 | Parte 1 — Emotet |
| `forenseWin` | 192.168.98.30 | Windows | Autopsy 4.21.0 | Parte 2 — Greg Schardt |

O acesso às VMs é feito via **RDP** a partir de uma máquina *landing*.

---

## Parte 1 — forenseLinux (Zeek)

### Pré-requisitos

| Componente | Versão | Observação |
|---|---|---|
| Zeek | 6.0.4 | Instalado em `/opt/zeek/bin/` |
| zeek-cut | — | Incluído na instalação do Zeek |
| p7zip-full | qualquer | Necessário se `unzip` não suportar AES-256 |

### Checklist do ambiente

```bash
# 1. Verificar versão do Zeek
/opt/zeek/bin/zeek --version

# Se o comando não for encontrado:
find / -name "zeek" -type f 2>/dev/null
export PATH=$PATH:/opt/zeek/bin

# 2. Verificar zeek-cut
/opt/zeek/bin/zeek-cut --help | head -5

# 3. Verificar evidência
ls -lh /opt/estudodecaso/

# 4. Espaço em disco (Zeek gera ~10% do PCAP em logs)
df -h ~/

# 5. Usuário atual
whoami && id
```

!!! warning "Problema comum — AES-256"
    Se o `unzip` retornar `unsupported compression method 99`, instale o `p7zip`:
    ```bash
    sudo apt-get install p7zip-full -y
    7z x -Pinfected /opt/estudodecaso/*.zip -o~/analise_caso/
    ```

### Boas práticas forenses

```bash
# NUNCA processe direto em /opt/estudodecaso/ — trabalhe em cópia
mkdir -p ~/analise_caso && cd ~/analise_caso

# Iniciar gravação de sessão (registra todos os comandos e saídas)
script -a ~/analise_caso/sessao_$(date +%Y%m%d_%H%M%S).log

# Hashes do ZIP original (registrar ANTES de qualquer operação)
md5sum /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
sha256sum /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
```

!!! tip "Por que gravar a sessão?"
    O `script` registra todos os comandos executados e suas saídas em um arquivo de log. Em uma investigação real, esse log é parte da documentação forense — prova que os procedimentos foram executados corretamente e na ordem registrada.

### Extração e processamento do PCAP

```bash
cd ~/analise_caso/

# Extrair o PCAP
unzip -P infected \
  /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip \
  -d ~/analise_caso/

# Registrar hash do PCAP extraído
md5sum ~/analise_caso/*.pcap

# Processar com Zeek (sem saída = comportamento normal)
/opt/zeek/bin/zeek -r *.pcap

# Verificar logs gerados
ls -lhS *.log
```

### Logs gerados pelo Zeek

```bash
# Confirmar campos disponíveis antes de usar zeek-cut
grep "^#fields" dhcp.log
grep "^#fields" http.log
grep "^#fields" files.log
```

| Log | Conteúdo |
|---|---|
| `conn.log` | Todas as conexões de rede |
| `http.log` | Requisições HTTP |
| `dns.log` | Consultas DNS |
| `files.log` | Arquivos transferidos |
| `dhcp.log` | Negociações DHCP (IP, MAC, hostname) |
| `smtp.log` | Conexões de e-mail |
| `pe.log` | Análise de executáveis PE |
| `smb_files.log` | Acessos a arquivos via SMB |
| `x509.log` | Certificados TLS |

---

## Parte 2 — forenseWin (Autopsy)

### Pré-requisitos

| Componente | Versão | Observação |
|---|---|---|
| Autopsy | 4.21.0 | Windows 64-bit |
| Java | 1.8.0_391 (64-bit) | Requisito do Autopsy |

### Download do segundo segmento

A imagem forense está dividida em dois arquivos (E01 + E02). O E01 já está na VM, o E02 precisa ser baixado:

```powershell
# Opção 1 — PowerShell
Invoke-WebRequest `
  -Uri "https://cfreds-archive.nist.gov/images/4Dell%20Latitude%20CPi.E02" `
  -OutFile "C:\curso\estudodecaso\4Dell Latitude CPi.E02" `
  -UseBasicParsing

# Confirmar que os dois arquivos estão no mesmo diretório
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, Length
```

### Boas práticas forenses

```powershell
# Marcar imagens como somente leitura
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Name IsReadOnly -Value $true
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E02" -Name IsReadOnly -Value $true

# Registrar hash ANTES da análise
$hash = Get-FileHash "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Algorithm MD5
$hash | Export-Csv "C:\curso\case_schardt\hash_evidencia.csv" -NoTypeInformation

# Iniciar log de sessão
Start-Transcript -Path "C:\curso\case_schardt\sessao_$(Get-Date -Format 'yyyyMMdd_HHmmss').log" -Append
```

### Configuração do case no Autopsy

```
Autopsy → New Case
  Case Name:      GoodMoneyFinancial_Schardt
  Base Directory: C:\curso\case_schardt\

Add Data Source → Image File
  Path: C:\curso\estudodecaso\4Dell Latitude CPi.E01
  (Autopsy localiza o E02 automaticamente no mesmo diretório)

Módulos de ingestão recomendados:
  ✅ Hash Lookup
  ✅ File Type Identification
  ✅ Recent Activity
  ✅ Keyword Search
  ✅ Windows Registry
  ✅ Extension Mismatch
  ✅ Encryption Detection
```

!!! info "Tempo de ingestão"
    A ingestão pode levar entre 10 e 30 minutos dependendo da máquina. Aguarde 100% em todos os módulos antes de gerar a timeline ou consultar artefatos.
