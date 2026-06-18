# 🪟 Roteiro Operacional – Parte 2: Análise Forense com Autopsy
**Caso:** Good Money Financial / Greg Schardt | **VM:** forenseWin | **IP:** 192.168.98.30
**Acesso:** RDP → usuário `Aluno` | senha `RnpEsr123@`

---

## ✅ Checklist do Ambiente

> ✅ **Seção concluída — ambiente verificado em 13/06/2026.**

```powershell
# 1. Verificar versão do Autopsy
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" |
  Where-Object DisplayName -like "*Autopsy*" |
  Select-Object DisplayName, DisplayVersion
```
> **Resultado:** `Autopsy 4.21.0` ✅

```powershell
# 2. Verificar Java instalado (Autopsy exige JRE 64-bit)
java -version
```
> **Resultado:** `java version "1.8.0_391"` — Java 8 64-bit ✅

```powershell
# 3. Verificar espaço em disco
Get-PSDrive C | Select-Object Used, Free
```
> **Resultado:** ~4,6 GB livres ✅ (suficiente — evitar salvar outros arquivos grandes durante a análise)

```powershell
# 4. Verificar segmentos da imagem
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, Length, LastWriteTime
```
> **Resultado:**
> ```
> Name                      Length       LastWriteTime
> ----                      ------       -------------
> 4Dell Latitude CPi.E01    671094597    3/28/2024 1:08:30 AM   ✅ presente
> 4Dell Latitude CPi.E02    —            —                      ⬇️ aguardando download
> ```

> 💡 **Ferramentas adicionais disponíveis no desktop da VM** (relevantes para Q8a):
> `FTK Imager` | `Nmap / Zenmap GUI` | `NetworkMiner` | `DB Browser (SQLite / SQLCipher)` | `AccessData`

---

## ⚠️ Boas Práticas Forenses

```powershell
# 1. NUNCA trabalhe sobre a imagem original
#    ⚠️ Executar SOMENTE APÓS o download do E02 — marcar os dois segmentos como read-only
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Name IsReadOnly -Value $true
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E02" -Name IsReadOnly -Value $true
# Verificar atributo — ambos devem retornar IsReadOnly: True
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, IsReadOnly
```
> **Status:** E01 marcado como read-only ✅ | E02 aguardando download para ser marcado

```powershell
# 2. REGISTRE hash ANTES — referência para comparação futura
#    Criar diretório do case primeiro
New-Item -ItemType Directory -Path "C:\curso\case_schardt" -Force
$hashAntes = Get-FileHash "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Algorithm MD5
$hashAntes | Export-Csv "C:\curso\case_schardt\hash_evidencia.csv" -NoTypeInformation
Write-Host "Hash E01: $($hashAntes.Hash)"
```
> 💡 O CSV gerado em `hash_evidencia.csv` documenta a integridade para o Laudo Pericial (Q10).

```powershell
# 3. Iniciar log de sessão — equivalente ao script do Linux
Start-Transcript -Path "C:\curso\case_schardt\sessao_$(Get-Date -Format 'yyyyMMdd_HHmmss').log" -Append
# Para encerrar: Stop-Transcript
```

```powershell
# 4. NÃO execute binários encontrados na imagem — verificar política vigente
Get-ExecutionPolicy -List
```

---

## 📥 Download do Segundo Segmento da Imagem

A imagem está no formato EnCase dividida em dois segmentos (E01/E02).
Ambos precisam estar no mesmo diretório para o Autopsy processar corretamente.
**E01 já presente** (671 MB). Baixar o E02:

```powershell
# Opção 1 — Invoke-WebRequest (PowerShell nativo)
Invoke-WebRequest `
  -Uri "https://cfreds-archive.nist.gov/images/4Dell%20Latitude%20CPi.E02" `
  -OutFile "C:\curso\estudodecaso\4Dell Latitude CPi.E02" `
  -UseBasicParsing
```

> ⚠️ **Download lento ou travado:** cancelar com `Ctrl+C` e usar curl:
> ```powershell
> curl.exe -L `
>   -o "C:\curso\estudodecaso\4Dell Latitude CPi.E02" `
>   "https://cfreds-archive.nist.gov/images/4Dell%20Latitude%20CPi.E02"
> ```
>
> ⚠️ **Erro de certificado SSL:**
> ```powershell
> Invoke-WebRequest -Uri $url -OutFile $dest -SkipCertificateCheck
> ```

```powershell
# Após download: confirmar ambos os segmentos e marcar E02 como read-only
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, Length
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E02" -Name IsReadOnly -Value $true
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, IsReadOnly
```

> **Saída obtida:**
> ```
> Mode    LastWriteTime       Length      Name
> ----    -------------       ------      ----
> -ar---  6/14/2026 2:49 AM   671094597   4Dell Latitude CPi.E01   ✅ read-only
> -ar---  6/14/2026 2:51 AM   419384951   4Dell Latitude CPi.E02   ✅ read-only
> ```
> Ambos os segmentos presentes e protegidos contra escrita.

---

## 🔍 Q6a — Integridade da Imagem (Hash MD5)

O hash esperado fornecido no roteiro é: `aee4fcd9301c03b3b054623ca261959a`

O formato EnCase (E01) armazena o hash **internamente** no arquivo. O Autopsy verifica
automaticamente a integridade ao carregar a imagem. Mas realize a verificação manual também:

```powershell
# Verificar hash MD5 do primeiro segmento (E01 contém o hash da imagem completa)
Get-FileHash "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Algorithm MD5 |
  Select-Object Hash, Path
```

> **Saída obtida:**
> ```
> Algorithm  Hash                              Path
> ---------  ----                              ----
> MD5        943243E71EDA7481FEE7B83F06698993  C:\curso\estudodecasoDell Latitude CPi.E01
> ```
>
> ⚠️ **Este hash não coincide com `aee4fcd9301c03b3b054623ca261959a` — isso é ESPERADO.**
> O `Get-FileHash` calculou apenas o segmento E01 isolado. O hash do roteiro refere-se
> à **imagem completa** (E01+E02 combinados), e o formato EnCase armazena esse hash
> internamente no cabeçalho do E01.
>
> **A verificação correta é feita pelo FTK Imager**, que lê o hash interno do EnCase:
> ```
> FTK Imager → File → Add Evidence Item → Image File
>   → selecionar 4Dell Latitude CPi.E01
>   → aba Image Information ou Properties
>   → campo "Computed Hash" e "Stored Hash" devem coincidir
> ```
> Se `Computed Hash == Stored Hash` no FTK Imager → **imagem íntegra** ✅
> O Autopsy também verifica a integridade interna ao carregar a imagem (mensagem
> "No errors reported" no módulo Recent Activity confirma isso).
>
> **Resultado obtido (14/06/2026 03:25):**
> ```
> Name:                    4Dell Latitude CPi.E01
> Sector count:            9.514.260
> MD5 Computed hash:       aee4fcd9301c03b3b054623ca261959a
> MD5 Stored hash:         aee4fcd9301c03b3b054623ca261959a
> Verify result:           Match ✅
> SHA1 Computed hash:      da2fe30fe21711edf42310873af475859a6...
> Bad sectors:             No bad sectors found ✅
> ```
> 💡 O `Get-FileHash` do PowerShell havia retornado `943243E71EDA...` porque calculou
> apenas o arquivo E01 como dado bruto, sem interpretar o formato EnCase.
> O FTK Imager lê a imagem completa (E01+E02) e compara com o hash armazenado
> internamente pela ferramenta de aquisição original — este é o método correto.
>
> ✅ **Q6a — Integridade confirmada.** Cadeia de custódia preservada.
> 📸 Print salvo: `Q6a_-_Hash_do_Disco.png`

---

## 🖥️ Configuração do Case no Autopsy

### Criar novo case

```
Autopsy → New Case
  Case Name:    GoodMoneyFinancial_Schardt
  Base Directory: C:\curso\case_schardt\
  Case Number:  [preencher]
  Examiner:     [preencher]
→ Next → Finish
```

### Adicionar a imagem como Data Source

```
Add Data Source → Image File
  Path: C:\curso\estudodecaso\4Dell Latitude CPi.E01
  (Autopsy localiza o E02 automaticamente no mesmo diretório)
  
Timezone: UTC ou conforme fuso identificado
```

> ⚠️ **Problema: Autopsy não encontra o E02**
> Verificar se ambos os arquivos têm exatamente o mesmo nome base
> (`4Dell Latitude CPi.E01` e `4Dell Latitude CPi.E02`) no mesmo diretório.
> Nomes divergentes (maiúsculas, espaços) quebram a detecção automática.

### Módulos de Ingestão — selecionar todos relevantes

```
✅ Hash Lookup               → verifica hashes contra bases de malware/NSRL
✅ File Type Identification  → identifica tipos reais de arquivo (magic bytes)
✅ Recent Activity           → histórico browser, downloads, execuções recentes
✅ Keyword Search            → busca por termos relevantes ao caso
✅ Windows Registry          → extrai artefatos de registro automaticamente
✅ Extension Mismatch        → arquivos com extensão diferente do tipo real
✅ Encryption Detection      → identifica arquivos cifrados ou protegidos
```

> 💡 A ingestão pode levar de 10 a 30 minutos dependendo do tamanho da imagem.
> Não feche o Autopsy durante o processo — acompanhe o progresso na barra inferior.
> Enquanto a ingestão roda em background, é possível navegar pelos arquivos.
>
> **Status em 14/06/2026 03:50 — ingestão a 62%**, já encontrado:
>
> | Categoria | Quantidade | Relevância |
> |-----------|-----------|------------|
> | Installed Programs | 32 | Q8a |
> | Run Programs | 81 | Q8a / Q9 |
> | Shell Bags | 51 | Q9 timeline |
> | Recent Documents | 8 | Q9 / Q10 |
> | Web History | 887 | Q9 |
> | Web Bookmarks | 6 | Q9 |
> | Web Cookies | 24 | Q9 |
> | Web Search | 4 | Q9 |
> | USB Device Attached | 1 | Q10 — achado relevante |
> | OS Accounts | — | Q7 |
> | Operating System Information | 1 | Q6b |
>
> ⚠️ Aguardar 100% antes de navegar — módulos ainda em execução podem atualizar os resultados.

---

## 🔍 Q6b — Sistema Operacional Utilizado

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SOFTWARE

Navegar até:
  SOFTWARE → Microsoft → Windows NT → CurrentVersion
  
Campos relevantes:
  ProductName        → nome do SO
  CurrentVersion     → versão
  CurrentBuildNumber → build
  RegisteredOwner    → proprietário (também responde Q7a)
  RegisteredOrganization
  InstallDate        → data de instalação
```

> **Resultado obtido (14/06/2026 04:08) — via Autopsy → Data Artifacts → Operating System Information:**
> ```
> Source:        4Dell Latitude CPi.E01
> Program Name:  Microsoft Windows XP
> Architecture:  x86 (32-bit)
> Computer Name: N-1A9ODN6ZXi4LQ
> Product ID:    55274-640-0147306-23684
> Path:          C:\WINDOWS
> Temp Dir:      %SystemRoot%\TEMP
> Owner:         Greg Schardt              ← também responde Q7a
> Organization:  N/A
> ```
> 💡 Windows XP x86 — não Windows 98 como a era do hardware poderia sugerir.
> O Dell Latitude CPi foi atualizado com XP, prática comum em empresas no início dos anos 2000.
>
> ✅ **Q6b — SO: Microsoft Windows XP (x86)**
> 📸 Print salvo: `Q6b_-_Sistema_operacional.png`

---

## 🔍 Q7 — Proprietário, Nome da Conta, Último Desligamento e Último Logon

### Q7a — Proprietário Registrado

```
Autopsy → Data Artifacts → Operating System Information
  Coluna: Owner
```

> ✅ **Resultado:** `Greg Schardt`
> Confirmado via Operating System Information — mesma tela de Q6b.

### Q7b — Nome do Computador

```
Autopsy → Data Artifacts → Operating System Information
  Coluna: Name
```

> ✅ **Resultado:** `N-1A9ODN6ZXi4LQ`
> Confirmado via Operating System Information — mesma tela de Q6b.
> Para confirmar via registro (opcional):
> ```
> Autopsy → Data Sources → [imagem] → vol2 → Windows\System32\config\SYSTEM
>   SYSTEM → ControlSet001 → Control → ComputerName → ComputerName
> ```

### Q7c — Último Desligamento

```
Autopsy → Data Sources → img_4Dell Latitude CPi.E01
  → vol_vol2 → WINDOWS → system32 → config → [SYSTEM hive]
  → aba Application (painel inferior) → Watchdog → Windows → ShutdownTime

Caminho no registro: SYSTEM\ControlSet001\Control\Windows
Valor: ShutdownTime | Tipo: REG_BIN
```

> **Resultado obtido (14/06/2026 04:29):**
> ```
> Valor bruto (little-endian): C4 FC 00 07 4D 8C C4 01
> FILETIME (64-bit LE):        0x01C48C4D0700FCC4
> Conversão:
>   (0x01C48C4D0700FCC4 - 116.444.736.000.000.000) ÷ 10.000.000
>   = 1.093.621.573 (Unix timestamp)
>   = 2004-08-27 15:46:13 UTC
>
> Modification Time da chave Windows: 2004-08-27 15:46:33 UTC
> (SO registrou o shutdown → ~20s depois o kernel flushou o registro no disco)
> ```
>
> ✅ **Q7c — Último desligamento: 2004-08-27 15:46:13 UTC**
> 📸 Print salvo: Q7c_-_ShutdownTime.png

### Q7d — Conta de Usuário / Último Logon

```
Autopsy → OS Accounts
  → Clicar em "Mr. Evil"
  → Aba "OS Account" no painel inferior
```

> ✅ **Resultado obtido (14/06/2026 04:12):**
> ```
> Login Name:    Mr. Evil
> SID:           S-1-5-21-2000478354-688789844-1708537768-1003
> Tipo:          Usuário criado manualmente
> Criação:       2004-08-19 23:03:54 UTC
> Host:          4Dell Latitude CPi.E01
> Scope:         Local
> ```
> 🔴 **Achado crítico:** o proprietário registrado é `Greg Schardt` mas o único usuário
> criado manualmente é **`Mr. Evil`** (SID ...1003). Esta é a conta operacional do suspeito.
> O nome da conta é evidência relevante — registrar no Laudo Pericial (Q10).
>
> Para obter o timestamp do último logon, clicar na conta `Mr. Evil` e verificar
> a aba `OS Account` no painel inferior do Autopsy.

> 📸 Print salvo: `Q7c_-_sera.png` (OS Accounts — renomear para `Q7d_-_OS_Accounts.png`)

---

## 🔍 Q8a — Programas Instalados com Potencial Malicioso

```
Autopsy → Data Artifacts → Installed Programs (32)
```

> **Resultado obtido (14/06/2026 04:33) — lista completa de 32 programas:**

**🔴 Ferramentas de Ataque / Reconhecimento — instalação entre 19-27/08/2004:**

| Programa | Data Instalação (UTC) | Classificação |
|----------|----------------------|---------------|
| Cain & Abel v2.5 beta45 | 2004-08-20 15:05:58 | 🔴 CRÍTICO — password cracker + ARP poisoner + sniffer |
| 123 Write All Stored Passwords | 2004-08-20 15:13:08 | 🔴 CRÍTICO — extrai senhas armazenadas no Windows |
| Network Stumbler 0.4.0 | 2004-08-27 15:12:15 | 🔴 Scanner WiFi / wardriving |
| Look@LAN 2.50 Build 29 | 2004-08-25 15:56:11 | 🔴 Scanner de rede — mapeia hosts ativos |
| Ethereal 0.10.6 | 2004-08-27 15:29:19 | 🔴 Analisador de protocolo / packet sniffer |
| WinPcap 3.01 alpha | 2004-08-27 15:15:19 | 🔴 Biblioteca de captura de pacotes (dependência Ethereal/Cain) |
| Anonymizer Bar 2.0 | 2004-08-20 15:05:09 | 🟡 Ferramenta de anonimização online |

**🟡 Dual-use / Comunicação:**

| Programa | Data Instalação (UTC) | Observação |
|----------|----------------------|-----------|
| mIRC | 2004-08-20 15:10:04 | Cliente IRC — usado historicamente como canal C2 |
| CuteFTP | 2004-08-20 15:09:02 | Cliente FTP — potencial exfiltração de dados |
| Forté Agent | 2004-08-20 15:08:19 | Leitor de newsgroups |
| NetMeeting | 2004-08-19 22:31:52 | Colaboração/acesso remoto |

**🟢 Legítimos / Componentes Windows (instalados 2004-08-19):**
Powertoys XP, CuteHTML, Faber Toys, WebFldrs XP, NetShow Player 2.0, MPlayer2,
IE40/IE4Data/IE5BAKEX/IEData, DirectAnimation, DirectDrawEx, OutlookExpress,
AddressBook, ICW, PCHealth, Fontcore, SchedulingAgent, Connection Manager,
MobileOptionPack, Branding.

> 🔴 **Análise crítica:** a instalação sequencial de Ethereal + WinPcap + Cain & Abel +
> Network Stumbler + Look@LAN + 123WritePasswords entre 19–27/08/2004 configura um
> **toolkit ofensivo completo** — captura de tráfego, reconhecimento de redes WiFi e
> cabeadas, e extração de credenciais. Nenhum uso profissional legítimo justifica essa
> combinação num computador pessoal.
>
> ✅ **Q8a concluída.**
> 📸 Prints salvos: `Q8aa.png` (programas 1–21) | `Q8ab.png` (programas 22–32)

## 🔍 Q8b — Arquivos na Lixeira

```
Autopsy → Data Sources → img_4Dell Latitude CPi.E01
  → vol_vol2 → RECYCLER → S-1-5-21-2000478354-688789844-1708537768-1003
```

> **Resultado obtido (14/06/2026 04:44) — 8 itens, SID = Mr. Evil:**

| Arquivo | Tamanho | Criado (UTC) | Modificado (UTC) |
|---------|---------|--------------|-----------------|
| Dc1.exe | 2.160.043 bytes (~2,1 MB) | 2004-08-27 15:51:24 | 2004-08-27 15:51:23 |
| Dc2.exe | 1.324.940 bytes (~1,3 MB) | 2004-08-27 15:11:07 | 2004-08-27 15:11:07 |
| Dc3.exe | 442.417 bytes (~432 KB)   | 2004-08-27 15:14:20 | 2004-08-27 15:14:20 |
| Dc4.exe | 8.460.502 bytes (~8,1 MB) | 2004-08-27 15:24:24 | 2004-08-27 15:24:24 |
| desktop.ini | 65 bytes | 2004-08-25 16:18:25 | 2004-08-25 16:12:30 |
| INFO2 | 3.220 bytes | 2004-08-25 16:18:25 | 2004-08-27 15:46:17 |

> 🔴 **Achados críticos:**
>
> **1. Padrão de limpeza de evidências:** todos os EXEs foram deletados em 2004-08-27
> entre 15:11 e 15:51 UTC — o shutdown ocorreu às 15:46 UTC. O usuário deletou
> ferramentas minutos antes de desligar o computador.
>
> **2. Nomes originais no INFO2:** o arquivo INFO2 contém os nomes reais dos arquivos
> antes de serem renomeados pelo Windows para Dc*.exe. Verificar:
> ```
> Clicar em INFO2 → aba Text no painel inferior
> ```
> Os nomes originais são evidência direta do que foi deletado.
>
> **3. Dc4.exe com 8,46 MB** — tamanho incomum, possivelmente toolkit completo ou
> arquivo auto-extraível.
>
> **4. Arquivos recuperáveis** — lixeira não foi esvaziada. Exportar via:
> ```
> Botão direito em cada Dc*.exe → Extract File(s) → salvar em C:\curso\case_schardt\lixeira> ```
>
> **Nomes originais confirmados via INFO2 (14/06/2026 14:15):**
>
> | Dc*.exe | Nome Original | Ferramenta | Tamanho |
> |---------|--------------|-----------|---------|
> | Dc1.exe | lalsetup250.exe | Look@LAN 2.50 installer | 2.160.043 bytes |
> | Dc2.exe | netstumblerInstaller_0_4_0.exe | Network Stumbler 0.4.0 installer | 1.324.940 bytes |
> | Dc3.exe | WinPcap_3_01_a.exe | WinPcap 3.01a installer | 442.417 bytes |
> | Dc4.exe | ethereal-setup-0.10.6.exe | Ethereal 0.10.6 installer | 8.460.502 bytes |
>
> **Caminho original de todos:** `C:\Documents and Settings\Mr. Evil\Desktop\`
>
> 🔴 **Conclusão forense:**
> - Instaladores estavam no **Desktop de Mr. Evil** — instalação deliberada e intencional
> - Deletados em 2004-08-27 entre 15:11–15:51 UTC — mesmo dia do shutdown (15:46 UTC)
> - Padrão: instalou → usou → tentou apagar rastros antes de desligar
> - Lixeira **não foi esvaziada** — arquivos ainda recuperáveis como evidência material
> - Correlação direta com Q8a: estes 4 instaladores correspondem a 4 dos 7 programas
>   ofensivos encontrados em Installed Programs
>
> ✅ **Q8b concluída.**
> 📸 Prints salvos: `Q8b.png` + `Q8ba.png` (INFO2 com nomes originais)

## 🔍 Q9 — Timeline das Evidências

```
Autopsy → Tools → Timeline → List View
Filtros aplicados:
  ✅ File Created + File Modified (File Accessed desmarcado)
  ✅ Executables
  Start: Aug 25, 2004  |  End: Aug 27, 2004
→ 20 eventos relevantes
```

> **Resultado obtido (14/06/2026 14:50) — 20 eventos:**

**Legenda:** `_B_` = Criado | `__M` = Modificado | `__BM` = Criado+Modificado

### 2004-08-25 — Início da limpeza

| Hora (Local) | Tipo | Arquivo | Significado |
|---|---|---|---|
| 15:51:23 | `__M` | RECYCLER/.../Dc1.exe | lalsetup250.exe (Look@LAN) movido à lixeira |
| 15:51:24 | `_B_` | RECYCLER/.../Dc1.exe | Entrada na lixeira criada |
| 15:55:27 | `__M` | WINDOWS/iun6002.exe | InstallShield uninstaller — Look@LAN desinstalado |
| 15:56:09 | `_B_` | WINDOWS/iun6002.exe | Arquivo de desinstalação criado |

### 2004-08-26 — Driver de rede

| Hora (Local) | Tipo | Arquivo | Significado |
|---|---|---|---|
| 15:07:39–41 | `_B_` | system32/spool/drivers/PCL5ERES/PCLXL/UNIDRV/UNIRES.DLL | Drivers — Look@LAN mapeando impressoras de rede |

### 2004-08-27 — Sequência sistemática de cobertura de rastros

| Hora (Local) | Tipo | Arquivo | Significado |
|---|---|---|---|
| 15:11:07 | `__BM` | RECYCLER/.../Dc2.exe | netstumblerInstaller movido à lixeira |
| **15:11:50** | `_B_` | **IE5 cache/ethereal-setup-0.10.6[1].exe** | **🔴 Ethereal baixado via Internet Explorer** |
| 15:12:16 | `__BM` | Program Files/Network Stumbler/uninst.exe | Network Stumbler desinstalado |
| 15:14:20 | `__BM` | RECYCLER/.../Dc3.exe | WinPcap_3_01_a.exe movido à lixeira |
| 15:15:17 | `_B_` | Program Files/WinPcap/Uninstall.exe | WinPcap desinstalado |
| 15:16:49 | `__M` | IE5 cache/ethereal-setup-0.10.6[1].exe | Ethereal ainda em cache IE |
| 15:24:24 | `__BM` | RECYCLER/.../Dc4.exe | ethereal-setup movido à lixeira |
| 15:29:19 | `_B_` | Program Files/Ethereal/uninstall.exe | Ethereal desinstalado |
| 15:29:31 | `__M` | Program Files/Ethereal/uninstall.exe | — |
| 15:31:30 | `_B_` | system32/drivers/wlluc48.sys (x2) | Driver WiFi Lucent/Agere — usado pelo Network Stumbler |

> 🔴 **Achados críticos:**
>
> **1. Ethereal baixado via IE:** caminho `Content.IE5/PN0J7OQM/ethereal-setup-0.10.6[1].exe`
> é o cache do Internet Explorer. O sufixo `[1]` indica cópia anterior existente.
> Prova acesso intencional à internet para obtenção de ferramenta de ataque.
>
> **2. Sequência deliberada de limpeza em 2004-08-27:**
> deletar instalador → desinstalar ferramenta → repetir. Processo sistemático de
> remoção de evidências executado antes do shutdown às 15:46 UTC.
>
> **3. Cain & Abel e 123WritePasswords ausentes da limpeza** — não aparecem na timeline
> de desinstalação. Estavam possivelmente ainda instalados ao desligar a máquina.
>
> ✅ **Q9 concluída.**
> 📸 Print salvo: `Q9a.png` | CSV exportado: `timeline_schardt.csv`

## 📑 Estrutura do Laudo Pericial (Q10)

```
1. INTRODUÇÃO
   - Identificação do perito e autoridade requisitante (DCC)
   - Número do caso / processo
   - Suspeito: Greg Schardt
   - Contexto: incidente Good Money Financial → investigação policial →
     busca e apreensão → imagem do notebook apreendido
   - Objetivo: correlacionar evidências do notebook ao incidente de 12/11/2018

2. MÉTODO
   Equipamento periciado:
     Tipo:          Notebook Dell Latitude CPi
     SO:            [preencher com Q6b]
     Identificação: imagem E01/E02
     Hash MD5:      aee4fcd9301c03b3b054623ca261959a
     Origem:        apreendido na residência de Greg Schardt pela DCC
   
   Procedimentos:
     - Recebimento e verificação de integridade (hash MD5)
     - Criação de case no Autopsy [versão]
     - Ingestão com módulos: [listar os ativados]
     - Análise de registro Windows, sistema de arquivos, lixeira e timeline

3. COLETA DE EVIDÊNCIAS
   - Imagem de disco (E01+E02) com hash verificado
   - Registros Windows extraídos via Autopsy
   - Timeline gerada e exportada

4. ANÁLISE E RESULTADOS
   Q6: Integridade confirmada. SO: [resultado]
   Q7: Proprietário: [resultado] | Conta: [resultado] |
       Desligamento: [resultado] | Último logon: [resultado]
   Q8a: Programas suspeitos: [listar com caminhos]
   Q8b: Arquivos na lixeira: [listar com metadados]
   Q9: Timeline — eventos relevantes correlacionados ao incidente de 12/11/2018

5. DISCUSSÃO E CONCLUSÕES
   - Correlação entre evidências do notebook e o incidente na Good Money Financial
   - Achados que vinculam Greg Schardt ao crime investigado
   - Limitações da análise (ex: arquivos cifrados, dados apagados não recuperáveis)

6. ANEXOS
   - Print da verificação de hash (Q6a)
   - Prints de cada achado de registro (Q7)
   - Lista completa de programas (Q8a)
   - Lista completa da lixeira (Q8b)
   - Arquivo CSV da timeline completa (Q9)
   - Capturas de tela relevantes do Autopsy
```

---

## 🗒️ Log de Evidências — Preencher Durante a Análise

| # | Quando | Ação | Resultado | Arquivo |
|---|--------|------|-----------|---------|
| 1 | 14/06/2026 02:51 | Download E02 concluído | 419.384.951 bytes | — |
| 2 | 14/06/2026 02:49 | Read-only ambos segmentos | E01 ✅ E02 ✅ | — |
| 3 | 14/06/2026 | Hash E01 (Get-FileHash) | `943243E71EDA7481FEE7B83F06698993` | print_hash_ps.png |
| 4 | 14/06/2026 03:25 | **Q6a** — Integridade via FTK Imager | MD5 Match ✅ `aee4fcd9...` / Sem bad sectors | Q6a_-_Hash_do_Disco.png |
| 5 | 14/06/2026 03:50 | Case Autopsy criado | GoodMoneyFinancial_Schardt | — |
| 6 | 14/06/2026 03:50 | Ingestão em andamento | 62% — aguardando 100% | — |
| 5 | | **Q6a** — Integridade | | print_q6a.png |
| 6 | | **Q6b** — SO | | print_q6b.png |
| 7 | 14/06/2026 04:08 | **Q7a** — Proprietário | Greg Schardt | Q6b_-_Sistema_operacional.png |
| 8 | 14/06/2026 04:08 | **Q7b** — Nome do computador | N-1A9ODN6ZXi4LQ | Q6b_-_Sistema_operacional.png |
| 9 | 14/06/2026 04:29 | **Q7c** — Último desligamento | 2004-08-27 15:46:13 UTC | Q7c_-_ShutdownTime.png |
| 10 | 14/06/2026 04:12 | **Q7d** — Conta usuário | Mr. Evil (SID ...1003, criado 2004-08-19) | Q7d_-_OS_Accounts.png |
| 11 | 14/06/2026 04:33 | **Q8a** — Programas suspeitos | 7 ferramentas ofensivas identificadas | Q8aa.png + Q8ab.png |
| 12 | 14/06/2026 14:15 | **Q8b** — Lixeira (Mr. Evil) | 4 instaladores de ferramentas ofensivas — Desktop de Mr. Evil | Q8b.png + Q8ba.png |
| 13 | 14/06/2026 14:50 | **Q9** — Timeline (20 eventos) | Sequência de limpeza 25–27/08/2004 confirmada | Q9a.png |
| 14 | | **Q10** — Laudo redigido | | laudo_pericial.docx |

---

*Roteiro gerado em 12/06/2026 — Caso Good Money Financial / Parte 2 — Análise Forense*
