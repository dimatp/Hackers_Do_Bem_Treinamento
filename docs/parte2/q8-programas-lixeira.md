# Q8 — Programas suspeitos e Lixeira

## Q8a — Programas com potencial malicioso

### Onde encontrar no Autopsy

```
Autopsy → Results → Installed Programs

Ou pelo registro:
SOFTWARE → Microsoft → Windows → CurrentVersion → Uninstall
SOFTWARE → Microsoft → Windows → CurrentVersion → App Paths
```

### Classificação dos programas encontrados

Ao analisar os programas instalados, classifique-os em três categorias:

| Categoria | Critério | Exemplos esperados no caso |
|---|---|---|
| **Suspeitos** | Ferramentas ofensivas ou de ataque direto | Cain & Abel, NetStumbler, Ethereal |
| **Abusáveis** | Ferramentas legítimas com potencial de uso indevido | VNC, putty, WinPcap |
| **Normais** | Software de uso geral sem relevância para o caso | Office, WinZip, browsers |

!!! warning "Installed Programs vs Run Programs"
    `Installed Programs` prova que o software estava no disco.
    `Run Programs` (Data Artifacts) prova que o software foi **efetivamente executado**.

    Para o laudo, a distinção é crítica: ter uma ferramenta instalada pode ser coincidência; tê-la executado múltiplas vezes, com timestamps, é evidência de uso deliberado.

    ```
    Autopsy → Data Artifacts → Run Programs
    Colunas relevantes: Program Name | Run Count | Last Run | Arguments
    ```

---

## Q8b — Arquivos na lixeira

### Onde encontrar no Autopsy

```
Autopsy → Results → Recycle Bin

Verificação manual:
  Autopsy → Data Sources → [imagem] → vol → RECYCLER
  (arquivo INFO2 contém: nome original, data de deleção, tamanho)
```

### Interpretação

| Localização do arquivo | Significado |
|---|---|
| Na lixeira (`RECYCLER`) | Deletado pelo usuário, mas **recuperável** — não foi esvaziado |
| Em espaço não alocado | Deletado permanentemente — pode ser recuperável com ferramentas forenses |
| Ausente | Pode ter sido sobrescrito ou nunca existiu nesta imagem |

!!! tip "Valor forense da lixeira"
    Um arquivo na lixeira que nunca foi esvaziado pode revelar intenção de deletar evidências — o usuário quis remover o arquivo do acesso imediato, mas não completou o processo. Arquivos deletados há pouco tempo antes do desligamento (correlacionar com `ShutdownTime`) são particularmente relevantes.
