# Q6 — Integridade da imagem e Sistema Operacional

## Q6a — A imagem está íntegra?

### Por que verificar o hash?

A verificação de integridade é o **primeiro ato obrigatório** de qualquer análise forense. Antes de extrair um único artefato, o perito precisa confirmar que a imagem recebida é idêntica à imagem coletada na cena — sem alterações, corrupções ou adulterações.

Isso é feito comparando o hash MD5 fornecido junto com a evidência contra o hash calculado localmente.

### Comando

```powershell
Get-FileHash "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Algorithm MD5 | Select-Object Hash, Path

# Alternativa via certutil
certutil -hashfile "C:\curso\estudodecaso\4Dell Latitude CPi.E01" MD5
```

### Resultado

```
Hash esperado:    AEE4FCD9301C03B3B054623CA261959A
Hash calculado:   AEE4FCD9301C03B3B054623CA261959A
Status:           ✅ ÍNTEGRA
```

!!! danger "Hash divergente"
    Se os hashes não coincidirem, a análise não deve prosseguir. Uma divergência indica que a imagem foi alterada após a coleta — o que pode invalidar toda a cadeia de custódia. O fato deve ser registrado formalmente e reportado à autoridade requisitante.

---

## Q6b — Qual o Sistema Operacional?

### Onde encontrar no Autopsy

```
Autopsy → Results → Operating System Information

Ou manualmente:
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SOFTWARE
  → Microsoft → Windows NT → CurrentVersion
  Campos: ProductName | CurrentVersion | CurrentBuildNumber
```

### Resultado esperado

**Windows 98** (ou Windows 98 Second Edition)

Consistente com o equipamento: um **Dell Latitude CPi** lançado originalmente em 1998, em uso no início dos anos 2000.
