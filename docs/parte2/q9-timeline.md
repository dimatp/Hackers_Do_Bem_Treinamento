# Q9 — Timeline das evidências

## O que é uma timeline forense?

A **timeline forense** é a reconstrução cronológica de todos os eventos registrados no sistema — criação, modificação e acesso a arquivos (MAC times), entradas de registro, histórico de navegação, execução de programas, logons e desligamentos.

Ela é o principal instrumento de **correlação de evidências**: permite identificar padrões, sequências de ações e janelas temporais relevantes para o caso.

---

## Gerando a timeline no Autopsy

```
Autopsy → Tools → Timeline → Generate Timeline
  → Selecionar intervalo relevante (todos os períodos disponíveis para primeira análise)
  → Visualização: List View (para exportação) ou Graph View (para análise visual)

Exportar:
  Timeline → Export → CSV
  Salvar: C:\curso\case_schardt\timeline_schardt.csv
```

---

## O que procurar na timeline

### Período de preparação
Eventos nos dias/semanas anteriores ao incidente:

- Downloads de ferramentas (`web_history.csv` + `run_programs.csv`)
- Instalação de software suspeito
- Acessos a sites de hacking ou repositórios de ferramentas

### Período do incidente
Eventos correlacionados ao crime investigado:

- Execução das ferramentas identificadas em Q8a
- Acessos a arquivos relacionados à Good Money Financial
- Conexões de rede (se disponíveis nos logs)

### Período pós-incidente
Eventos após o crime:

- Deleção de arquivos (correlacionar com lixeira de Q8b)
- Limpeza de histórico
- Último desligamento (Q7c)

---

## Analisando o CSV exportado

```powershell
# Importar e filtrar no PowerShell
$timeline = Import-Csv "C:\curso\case_schardt\timeline_schardt.csv"

# Filtrar por data específica
$timeline | Where-Object { $_.Date -like "*2004-08-27*" } |
  Select-Object Date, Source, Description |
  Format-Table -AutoSize

# Buscar por termos suspeitos
$timeline | Where-Object { $_.Description -match "hack|crack|scan|sniff|tool" } |
  Select-Object Date, Source, Description
```

!!! tip "Fuso horário"
    O Autopsy pode exibir timestamps em horário local sem especificar o fuso. Verifique em `Tools → Options → Display → Timezone`. Documente a configuração no laudo para garantir a reprodutibilidade da análise.
