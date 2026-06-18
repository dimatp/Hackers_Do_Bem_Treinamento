# Q10 — Laudo Pericial

## O que é um Laudo Pericial?

O **Laudo Pericial** é o documento técnico-jurídico produzido pelo perito forense para apresentar as conclusões da análise ao solicitante — neste caso, a Delegacia de Crimes Cibernéticos. Diferente de um relatório técnico interno, o laudo tem valor probatório em processo judicial e deve seguir padrões de clareza, objetividade e fundamentação.

---

## Estrutura do Laudo

### 1. Identificação

``` text
LAUDO PERICIAL Nº: [XXX/2026/DCC]
Processo:         [número do processo]
Data de Autuação: [data]
Solicitante:      Delegacia de Crimes Cibernéticos
Perito:           [nome] — Matrícula [número]
```

---

### 2. Introdução

Deve conter:

- **Objetivo do laudo** — o que se pretende determinar com a análise
- **Contexto da investigação** — como o equipamento chegou à custódia pericial
- **Suspeito identificado** — Greg Schardt, identificado pela operadora de internet a partir do IP e porta utilizados durante o incidente na Good Money Financial
- **Cadeia de custódia** — data e local da apreensão, responsável pelo transporte, condições de recebimento

---

### 3. Método

#### Equipamento periciado

| Campo | Valor |
|---|---|
| Tipo | Notebook |
| Marca/Modelo | Dell Latitude CPi |
| Sistema Operacional | Windows 98 |
| Formato da imagem | EnCase (.E01 + .E02) |
| Hash MD5 | `aee4fcd9301c03b3b054623ca261959a` |
| Verificação | ✅ Hash conferido — imagem íntegra |

#### Ferramentas utilizadas

| Ferramenta | Versão | Uso |
|---|---|---|
| Autopsy | 4.21.0 | Análise forense da imagem |
| Java | 1.8.0_391 (64-bit) | Requisito do Autopsy |
| PowerShell | — | Verificação de hash e logs de sessão |

#### Procedimentos adotados

1. Verificação de integridade da imagem (hash MD5)
2. Marcação dos arquivos de evidência como somente leitura
3. Registro de log de sessão (PowerShell Transcript)
4. Criação do case no Autopsy com todos os módulos de ingestão ativos
5. Análise dos artefatos por categoria (registro, programas, histórico, timeline)
6. Documentação de cada achado com captura de tela

---

### 4. Coleta de evidências

Fontes de evidências analisadas:

- **Registro do Windows** — sistema operacional, proprietário, contas, desligamento
- **Programas instalados** — via registro e sistema de arquivos
- **Run Programs** — histórico de execução de programas (UserAssist)
- **Web History** — histórico de navegação (887 itens)
- **Shell Bags** — pastas abertas no Explorer (51 entradas)
- **Recycle Bin** — arquivos deletados via lixeira
- **Recent Documents** — documentos acessados recentemente (8 itens)
- **USB Devices** — dispositivos USB conectados (1 dispositivo)
- **Timeline** — consolidação cronológica de todos os artefatos

---

### 5. Análise e resultados

Esta seção consolida os resultados de Q6 a Q9. Para cada questão, inclua:

- O artefato analisado e onde foi encontrado
- O resultado obtido
- A captura de tela correspondente (referência ao Anexo)
- A interpretação forense do achado

**Exemplo de entrada:**

> **5.1 Integridade da imagem (Q6a)**
>
> O hash MD5 calculado sobre o arquivo `4Dell Latitude CPi.E01` resultou em `AEE4FCD9301C03B3B054623CA261959A`, idêntico ao hash fornecido junto com a evidência. A imagem está íntegra e a análise pode prosseguir. *(ver Anexo A — print_q6a.png)*

---

### 6. Limitações da análise

!!! warning "Transparência metodológica"
    Todo laudo pericial deve documentar suas limitações. Omiti-las não fortalece o documento — pelo contrário, enfraquece sua credibilidade perante um perito contrário ou juiz técnico.

- **Dados voláteis não disponíveis** — a análise foi conduzida sobre imagem de disco. Memória RAM, processos em execução e conexões de rede ativas no momento da apreensão não estão disponíveis.
- **Timestamps sem fuso horário confirmado** — o Windows 98 não registra fuso horário de forma padronizada. Os timestamps foram registrados conforme exibidos pelo Autopsy; o fuso da máquina não pôde ser confirmado independentemente.
- **Web History parcialmente analisada** — 887 itens de histórico de navegação foram identificados. A análise cobriu os itens mais relevantes para o caso; a totalidade está disponível no arquivo `web_history.csv` (Anexo X).
- **Independência dos casos** — os dois cenários analisados (Emotet 2018 e NIST CFREDS 2004) são casos independentes utilizados para fins didáticos. Não há relação temporal ou técnica direta entre os ativos comprometidos das duas partes.

---

### 7. Discussão e conclusões

Esta seção deve:

- Correlacionar as evidências encontradas com o crime investigado
- Apontar quais achados vinculam Greg Schardt ao incidente na Good Money Financial
- Destacar a significância dos programas ofensivos identificados
- Apresentar as conclusões de forma clara e objetiva, sem especulação além do que as evidências suportam

---

### 8. Anexos

| Anexo | Conteúdo |
|---|---|
| A | Hash MD5 verificado (Q6a) |
| B | Prints do registro — SO e proprietário (Q6b, Q7a) |
| C | Prints de conta, desligamento e logon (Q7b, Q7c, Q7d) |
| D | Lista de programas suspeitos com timestamps de execução (Q8a) |
| E | Lista completa da lixeira (Q8b) |
| F | Timeline exportada — `timeline_schardt.csv` (Q9) |
| G | Histórico de navegação — `web_history.csv` |
| H | Programas executados — `run_programs.csv` |
| I | Shell Bags — `shell_bags.csv` |
| J | Log de sessão PowerShell |
