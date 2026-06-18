# 🎯 Roteiro de Melhorias — Estudo de Caso Good Money Financial
**Implementação direta das sugestões de avaliação | Prioridade alta → baixa**

---

## BLOCO 1 — Autopsy (forenseWin) — Artefatos não explorados
*Executar antes de qualquer edição no documento*

### 1.1 Web History — 887 itens
```
Autopsy → Data Artifacts → Web History (887)
→ Clicar em "Save Table as CSV" → salvar web_history.csv

Filtros de interesse no Autopsy:
  Colunas: Date Accessed | URL | Title | Browser
  Ordenar por Date (mais recentes primeiro)
  Buscar termos: hack | crack | password | tool | exploit | scan | sniff
```

> **O que procurar:**
> - Visitas a sites de hacking: `neworder.box.sk`, `netcat.be`, `packetstormsecurity.com`
> - Downloads de ferramentas (URLs com .exe, .zip)
> - Sites de anonimização / proxies
> - Qualquer acesso relacionado à Good Money Financial ou alvos financeiros

**Print necessário:** tela com Web History filtrada mostrando sites suspeitos.

---

### 1.2 Run Programs — 81 entradas
```
Autopsy → Data Artifacts → Run Programs (81)
→ Clicar em "Save Table as CSV" → salvar run_programs.csv

Colunas relevantes: Program Name | Run Count | Last Run | Arguments
```

> **O que procurar:**
> - Execuções de Cain & Abel, Ethereal, NetStumbler, Look@LAN com timestamps
> - Frequência de uso — quantas vezes cada ferramenta rodou
> - Argumentos de linha de comando — revelam alvos ou configurações

**Por que importa:** Installed Programs prova instalação. Run Programs prova **uso efetivo** — diferença crítica para o laudo.

**Print necessário:** tela mostrando execuções das 7 ferramentas ofensivas com timestamps.

---

### 1.3 USB Device Attached — 1 dispositivo
```
Autopsy → Data Artifacts → USB Device Attached (1)
→ Anotar: fabricante, produto, número de série, primeira e última conexão
```

> **O que procurar:**
> - Data de conexão vs. datas das atividades suspeitas — havia pendrive durante o uso das ferramentas?
> - Número de série — pode ser rastreado como evidência adicional
> - Se coincide com o dia 2004-08-27 (dia da limpeza), indica possível exfiltração

**Print necessário:** tela USB Device Attached com todos os campos visíveis.

---

### 1.4 Recent Documents — 8 itens
```
Autopsy → Data Artifacts → Recent Documents (8)
→ Anotar todos os nomes de arquivo e timestamps
```

> **O que procurar:**
> - Documentos acessados próximo ao shutdown (15:46 UTC em 27/08/2004)
> - Qualquer arquivo relacionado às ferramentas ou às vítimas

---

### 1.5 Shell Bags — 51 entradas
```
Autopsy → Data Artifacts → Shell Bags (51)
→ Salvar como CSV → shell_bags.csv

Colunas: Folder Path | Last Write
```

> Shell Bags registram pastas que o Windows Explorer abriu — mesmo após deleção dos arquivos.
> Revelar pastas de ferramentas acessadas confirma uso ativo, não apenas instalação.

---

## BLOCO 2 — Adições ao Relatório CTI (Q5)

### 2.1 Tabela MITRE ATT&CK — inserir após seção de IoCs

```
TÉCNICAS MITRE ATT&CK — Emotet (novembro 2018)

| ID         | Tática          | Técnica                              | Evidência                              |
|------------|-----------------|--------------------------------------|----------------------------------------|
| T1566.001  | Initial Access  | Phishing: Spearphishing Link         | URL ifcingenieria.cl → http.log        |
| T1204.002  | Execution       | User Execution: Malicious File       | Abertura do .doc com macro             |
| T1059.005  | Execution       | Command & Scripting: Visual Basic    | Macro VBA no Word                      |
| T1547.001  | Persistence     | Registry Run Keys / Startup Folder   | Comportamento padrão Emotet            |
| T1071.001  | C2              | Application Layer Protocol: Web      | Conexões SMTP smtp.log                 |
| T1021.002  | Lateral Movement| Remote Services: SMB/Admin Shares    | SMB FILE_OPEN no GPO — smb_files.log   |
| T1078      | Defense Evasion | Valid Accounts                       | Emotet exfiltra credenciais locais     |
| T1070.004  | Defense Evasion | Indicator Removal: File Deletion     | Sequência limpeza 2004-08-27           |
| T1568      | C2              | Dynamic Resolution                   | Múltiplos IPs/domínios SMTP            |

Referência: https://attack.mitre.org/groups/G0046/ (Mealybug — operador Emotet)
```

---

### 2.2 Kill Chain — inserir após tabela MITRE

```
LOCKHEED MARTIN CYBER KILL CHAIN — Caso Emotet GMF

Reconnaissance    →  Alvo: kyivartworks.com / Good Money Financial
Weaponization     →  Macro VBA em .doc + 6169583.exe compilado mesmo dia
Delivery          →  HTTP GET ifcingenieria.cl (site legítimo comprometido)
Exploitation      →  Usuário abre .doc → macro executa
Installation      →  6169583.exe instalado + persistência no registro
Command & Control →  11 conexões SMTP para Gmail, 163.com, bigticket.ae
Actions on Obj.   →  Spam + Reconhecimento AD + Potencial propagação lateral
```

---

## BLOCO 3 — Adições ao Laudo Pericial (Q10)

### 3.1 Referências legais — inserir na seção 1 (Introdução)

```
ARCABOUÇO LEGAL APLICÁVEL

Lei nº 12.737/2012 (Lei Carolina Dieckmann)
  Art. 154-A — Invasão de dispositivo informático alheio
  Pena: detenção de 3 meses a 1 ano + multa

Lei nº 14.155/2021
  Agrava penas do art. 154-A quando o crime é praticado contra
  instituições financeiras ou causa prejuízo econômico à vítima
  Pena: reclusão de 1 a 4 anos + multa

Decreto-Lei nº 2.848/1940 — Código Penal
  Art. 153 — Divulgação de segredo
  Art. 299 — Falsidade ideológica (se aplicável)

NORMAS TÉCNICAS DE REFERÊNCIA
  ABNT NBR ISO/IEC 27037:2013 — Diretrizes para identificação, coleta,
    aquisição e preservação de evidência digital
  ABNT NBR ISO/IEC 27042:2015 — Diretrizes para análise e interpretação
    de evidência digital
  RFC 3227 — Guidelines for Evidence Collection and Archiving
  NIST SP 800-86 — Guide to Integrating Forensic Techniques into Incident Response
```

---

### 3.2 Seção de limitações — inserir antes de Conclusões

```
LIMITAÇÕES DA ANÁLISE

1. PCAP truncado (Parte 1): hashes MD5/SHA1 dos arquivos transferidos estão
   indisponíveis pois os arquivos não foram completamente capturados na evidência
   de rede. Os nomes e tipos MIME foram identificados, mas os arquivos não podem
   ser submetidos a verificação em bases de inteligência de ameaças.

2. Timestamps em horário local (Parte 2): o Autopsy exibiu os eventos da
   Timeline em horário local sem especificação do fuso horário da máquina
   periciada. Os eventos foram registrados no computador que pode não ter
   estado com o relógio corretamente configurado em 2004.

3. Dados voláteis não coletados: a análise foi realizada sobre imagem de disco.
   Memória RAM, processos em execução e conexões de rede ativas no momento da
   apreensão não estão disponíveis.

4. Web History não integralmente analisada: 887 itens de histórico de navegação
   foram identificados pelo Autopsy. A análise cobriu os itens mais relevantes;
   a totalidade dos registros está disponível no arquivo web_history.csv (Anexo X).

5. Independência dos casos: os dois cenários analisados (Emotet 2018 e
   NIST CFREDS 2004) são casos independentes utilizados para fins didáticos.
   Não há relação direta entre os ativos comprometidos das duas partes.
```

---

### 3.3 Número de laudo e cabeçalho formal — inserir na capa

```
LAUDO PERICIAL Nº: [XXX/2026/DCC]
Processo:         [número do processo]
Data de Autuação: 14 de junho de 2026
Prazo:            [data limite]
Solicitante:      Delegacia de Crimes Cibernéticos
Perito:           [nome] — Matrícula [número]
```

---

### 3.4 Hashes das evidências digitais — inserir na seção 3

```
INTEGRIDADE DAS EVIDÊNCIAS DIGITAIS

Arquivo             | MD5                              | Verificado em
--------------------|----------------------------------|------------------
Q1-Q2_Mac_Host.png  | [calcular]                      | 14/06/2026
Q3_URL.png          | [calcular]                      | 14/06/2026
Q4_infeccao.png     | [calcular]                      | 14/06/2026
Q6a_Hash.png        | [calcular]                      | 14/06/2026
... (todos os prints)

Calcular via PowerShell:
Get-ChildItem *.png | ForEach-Object {
  $h = Get-FileHash $_.FullName -Algorithm MD5
  "$($_.Name) | $($h.Hash)"
}
```

---

## BLOCO 4 — Correções de Apresentação no PDF

### 4.1 Problema: Página 2 em branco (índice vazio)
**Causa:** o índice foi gerado como texto simples sem flowables de conteúdo real.
**Correção no script Python:**
```python
# Substituir a seção de índice por uma Table com o conteúdo
# O índice atual gera parágrafos mas sem conteúdo visível no PDF
# Adicionar Spacer(1, 0.5*cm) após o título do índice e garantir
# que os parágrafos de itens não são filtrados pelo renderizador
```

### 4.2 Problema: Timestamps sem especificação de fuso
**Correção:** substituir "Horário (Local)" por "Horário (Local — UTC−03:00 estimado)" ou
confirmar no Autopsy: `Tools → Options → Display → Timezone`.

### 4.3 Problema: Screenshots com texto ilegível
**Correção:** aumentar `max_h` de 7cm para 9cm nas imagens menores e usar
`hAlign='CENTER'` com margem reduzida. Alternativamente, exportar os screenshots
em resolução maior (200 DPI mínimo) antes de inserir.

### 4.4 Glossário — adicionar como última seção antes dos Anexos
```
GLOSSÁRIO DE ACRÔNIMOS

AD    Active Directory
C2    Command and Control
CTI   Cyber Threat Intelligence
DHCP  Dynamic Host Configuration Protocol
DLL   Dynamic-Link Library
GPO   Group Policy Object
IoC   Indicator of Compromise
MAC   Media Access Control
MIME  Multipurpose Internet Mail Extensions
PCAP  Packet Capture
PE    Portable Executable
SMB   Server Message Block
SMTP  Simple Mail Transfer Protocol
TTP   Tactic, Technique and Procedure
VBA   Visual Basic for Applications
```

---

## BLOCO 5 — Checklist de Execução

Execute nesta ordem para máximo aproveitamento das melhorias:

```
FORENSEWIN — Autopsy (fazer primeiro — dados ainda disponíveis na VM)
[ ] 1.1  Exportar Web History (887) → web_history.csv + print de sites suspeitos
[ ] 1.2  Exportar Run Programs (81) → run_programs.csv + print de execuções
[ ] 1.3  Print do USB Device Attached com todos os campos
[ ] 1.4  Anotar Recent Documents (8 itens)
[ ] 1.5  Exportar Shell Bags → shell_bags.csv

DOCUMENTO — Edições no PDF/Relatório
[ ] 2.1  Inserir tabela MITRE ATT&CK no Q5
[ ] 2.2  Inserir Kill Chain no Q5
[ ] 3.1  Inserir referências legais no Q10 seção 1
[ ] 3.2  Inserir seção de Limitações no Q10
[ ] 3.3  Preencher número de laudo e campo de analistas
[ ] 3.4  Calcular e registrar hashes de todos os prints
[ ] 4.1  Corrigir índice (página 2)
[ ] 4.2  Corrigir label de timestamps
[ ] 4.3  Aumentar tamanho das imagens
[ ] 4.4  Inserir glossário

REGENERAR PDF FINAL com todas as melhorias incorporadas
```

---

## Impacto Esperado por Bloco

| Bloco | Esforço | Impacto na Nota |
|-------|---------|----------------|
| 1 — Web History + Run Programs | 1h Autopsy | +1,0 ponto |
| 2 — MITRE ATT&CK + Kill Chain | 30min | +0,5 ponto |
| 3 — Legal + Limitações + Hashes | 45min | +0,5 ponto |
| 4 — Correções de apresentação | 30min | +0,3 ponto |

**Nota estimada após melhorias: 9,0–9,5 / 10**

---

*Roteiro de melhorias — 14/06/2026 — Good Money Financial*
