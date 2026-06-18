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

> **Saída esperada após download:**
> ```
> Name                      Length
> ----                      ------
> 4Dell Latitude CPi.E01    671094597   ← já presente
> 4Dell Latitude CPi.E02    XXXXXXXXX   ← recém baixado
> ```

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

> **Saída esperada:**
> ```
> Hash                             Path
> ----                             ----
> AEE4FCD9301C03B3B054623CA261959A C:\curso\estudodecaso\4Dell Latitude CPi.E01
> ```
> 💡 PowerShell retorna o hash em maiúsculas — comparar ignorando case.
> Se o hash bater com `aee4fcd9301c03b3b054623ca261959a`, a imagem está íntegra.
>
> ⚠️ **Hash divergente:** a imagem foi corrompida ou modificada após a coleta —
> isso invalida a cadeia de custódia. Não prosseguir com a análise; registrar a divergência
> e solicitar nova cópia da evidência.
>
> ⚠️ **Alternativa via certutil (cmd):**
> ```cmd
> certutil -hashfile "C:\curso\estudodecaso\4Dell Latitude CPi.E01" MD5
> ```

> 📸 **Print aqui** — mostrar hash calculado vs hash esperado. Q6a respondida.

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

> **Saída esperada (Dell Latitude CPi, fim dos anos 1990):**
> ```
> ProductName:  Windows 98  (ou Windows 98 Second Edition)
> CurrentVersion: 4.10
> ```
> 💡 O Dell Latitude CPi era um notebook corporativo lançado em 1998 — a época corresponde
> ao Windows 98/98SE como SO predominante.
>
> ⚠️ **Alternativa via Autopsy Results:**
> ```
> Results → Operating System Information
> ```
> O módulo Recent Activity extrai automaticamente essas informações para a aba Results.

> 📸 **Print aqui** — Q6b respondida.

---

## 🔍 Q7 — Proprietário, Nome da Conta, Último Desligamento e Último Logon

### Q7a — Proprietário Registrado

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SOFTWARE
  SOFTWARE → Microsoft → Windows NT → CurrentVersion
  Valor: RegisteredOwner
```

> **Saída esperada:** `Greg Schardt` (ou nome associado ao suspeito)

### Q7b — Nome da Conta do Computador

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SYSTEM
  SYSTEM → ControlSet001 → Control → ComputerName → ComputerName
  Valor: ComputerName
```

> 💡 Em Windows 98 o nome do computador também aparece em:
> ```
> HKLM\System\CurrentControlSet\Control\ComputerName\ComputerName
> ```
> ⚠️ Em algumas imagens o hive SYSTEM se chama `system` (minúsculas) — verificar no
> explorador de arquivos do Autopsy antes de navegar.

### Q7c — Último Desligamento

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SYSTEM
  SYSTEM → ControlSet001 → Control → Windows
  Valor: ShutdownTime  (formato FILETIME — Autopsy converte automaticamente)
```

> 💡 O campo ShutdownTime é um valor binário FILETIME (100-nanosecond intervals desde
> 01/01/1601). O Autopsy exibe o valor convertido para data/hora legível.
>
> ⚠️ **Alternativa via Results:**
> ```
> Results → Operating System User Account  (contém metadados de contas)
> Results → Recent Activity → Shutdown Time
> ```

### Q7d — Último Usuário a Fazer Logon

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SOFTWARE
  SOFTWARE → Microsoft → Windows NT → CurrentVersion → Winlogon
  Valor: DefaultUserName  (último usuário a fazer logon com auto-login)
  Valor: LastLoggedOnUser (disponível em algumas versões)
```

> 💡 Em Windows 98, o histórico de logon é mais limitado que no NT/2000/XP.
> O campo `DefaultUserName` no Winlogon registra o último usuário que efetuou login
> com credenciais salvas.
>
> ⚠️ **Alternativa:** verificar perfis em `C:\Windows\Profiles\` ou `C:\Users\`
> — a data de modificação da pasta do perfil indica o último uso.
> ```
> Autopsy → Data Sources → [imagem] → vol → Windows → Profiles
> ```

> 📸 **Print aqui** para cada item de Q7 — mostrar caminho no registro + valor.

---

## 🔍 Q8a — Programas Instalados com Potencial Malicioso

### Via Results (ingestão automática)

```
Autopsy → Results → Installed Programs
```

> 💡 Anotar todos os programas e classificar:
> - **Suspeitos:** network scanners, password crackers, sniffers, exploits, keyloggers
> - **Legítimos mas abusáveis:** ferramentas de administração remota, VPNs, compressores
> - **Normais:** Office, browsers, drivers

### Via Registro (manual)

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SOFTWARE
  SOFTWARE → Microsoft → Windows → CurrentVersion → Uninstall
  (listar todas as subchaves — cada uma é um programa instalado)
  
SOFTWARE → Microsoft → Windows → CurrentVersion → App Paths
  (programas registrados com atalho de execução)
```

### Via sistema de arquivos (programas sem entrada no registro)

```
Autopsy → Data Sources → [imagem] → vol → Program Files
  (listar diretórios — cada pasta é um programa instalado)
  
Autopsy → Data Sources → [imagem] → vol → Windows
  (verificar executáveis soltos no diretório Windows)
```

> ⚠️ **Programas portáteis (sem instalação)** não aparecem no registro —
> a varredura do sistema de arquivos é obrigatória para achados completos.
> Use a busca por extensão no Autopsy:
> ```
> Autopsy → Views → File Types → Executables
> ```
> Listar todos os `.exe` e `.dll` fora dos diretórios padrão do SO.

> 📸 **Print aqui** — lista de programas suspeitos com caminho completo.

---

## 🔍 Q8b — Arquivos na Lixeira

```
Autopsy → Results → Recycle Bin
```

> 💡 O Autopsy lista automaticamente os arquivos encontrados na lixeira com:
> - Nome original do arquivo
> - Caminho original antes da exclusão
> - Data/hora de exclusão
> - Tamanho
> - Status (se foi realmente excluído ou permanece recuperável)
>
> **Verificação manual no sistema de arquivos:**
> ```
> Autopsy → Data Sources → [imagem] → vol → RECYCLER  (Windows 98/2000/XP)
>   ou
> Autopsy → Data Sources → [imagem] → vol → $Recycle.Bin  (Windows Vista+)
> ```
>
> 💡 Arquivos na lixeira do Windows 98 ficam em `C:\RECYCLED` ou `C:\RECYCLER`.
> O arquivo `INFO2` dentro da pasta contém os metadados (nome original, data, tamanho).
> Autopsy parseia o INFO2 automaticamente se o módulo de recycle bin estiver ativo.
>
> ⚠️ **"Realmente excluídos"** significa verificar se o arquivo ainda está recuperável:
> - Se aparece na lixeira do Autopsy → **não** foi permanentemente excluído (ainda na lixeira)
> - Se aparece como `unallocated space` → foi excluído da lixeira (mas pode ser recuperável)
> - Verificar status na coluna `Known Status` do Autopsy

> 📸 **Print aqui** — lista de arquivos na lixeira com metadados.

---

## 🔍 Q9 — Timeline das Evidências

### Gerar Timeline no Autopsy

```
Autopsy → Tools → Timeline
  → Generate Timeline  (se ainda não foi gerada)
  → Selecionar intervalo de datas relevante
  → Visualizar em modo List View para exportação
```

> 💡 O Autopsy gera a timeline consolidando timestamps de:
> - Criação, modificação e acesso de arquivos (MAC times)
> - Entradas de registro (last written time)
> - Histórico de browser
> - Arquivos recentes (Recent Documents)
> - Eventos de logon
> - Prefetch (execuções de programas)

### Exportar Timeline

```
Autopsy → Timeline → Export  →  CSV
Salvar em: C:\curso\case_schardt\timeline_schardt.csv
```

> 💡 O CSV exportado pode ser aberto no Excel para filtrar por intervalo de datas
> e identificar atividade suspeita. Filtrar pela data aproximada do incidente
> (12/11/2018) e pelos dias anteriores para identificar preparação do ataque.
>
> ⚠️ **Problema: Timeline não gera ou fica em loop**
> Verificar se a ingestão foi concluída completamente antes de gerar a timeline.
> O progresso aparece na barra inferior do Autopsy — aguardar 100% em todos os módulos.

> 📸 **Print aqui** — tela do Autopsy com Timeline, evidências ordenadas por data.

---

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
| 1 | | Download E02 concluído | Tamanho: | — |
| 2 | | Hash MD5 verificado | ✅ / ❌ bate com `aee4fcd9...` | print_hash.png |
| 3 | | Case Autopsy criado | Diretório: C:\curso\case_schardt | — |
| 4 | | Ingestão concluída | Módulos: | — |
| 5 | | **Q6a** — Integridade | | print_q6a.png |
| 6 | | **Q6b** — SO | | print_q6b.png |
| 7 | | **Q7a** — Proprietário | | print_q7a.png |
| 8 | | **Q7b** — Nome conta | | print_q7b.png |
| 9 | | **Q7c** — Desligamento | | print_q7c.png |
| 10 | | **Q7d** — Último logon | | print_q7d.png |
| 11 | | **Q8a** — Prog. suspeitos | | print_q8a.png |
| 12 | | **Q8b** — Lixeira | | print_q8b.png |
| 13 | | **Q9** — Timeline exportada | | timeline_schardt.csv |
| 14 | | **Q10** — Laudo redigido | | laudo_pericial.docx |

---

*Roteiro gerado em 12/06/2026 — Caso Good Money Financial / Parte 2 — Análise Forense*
