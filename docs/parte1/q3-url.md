# Q3 — URL que entregou o documento Word

## A pergunta

> Qual URL no pcap retornou um documento Microsoft Word?

---

## Estratégia de análise

O `http.log` do Zeek registra todas as requisições HTTP, incluindo o campo `resp_mime_types` — o tipo MIME declarado pelo servidor na resposta. Para identificar a entrega de um documento Word, buscamos pelo MIME type `application/msword` (formato `.doc` legado) ou `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (formato `.docx`).

---

## Comando

```bash
# Panorama dos MIME types presentes no tráfego
/opt/zeek/bin/zeek-cut resp_mime_types < http.log \
  | grep -v "^#\|-" | sort | uniq -c | sort -rn

# Extrair a URL que entregou o documento Word
/opt/zeek/bin/zeek-cut host uri resp_mime_types < http.log \
  | grep -i "msword\|openxmlformats\|vnd.ms-word" \
  | awk '{print "http://" $1 $2, "\t[" $3 "]"}'
```

!!! tip "Se `resp_mime_types` retornar vazio (`-`)"
    ```bash
    # Alternativa 1: buscar por extensão na URI
    /opt/zeek/bin/zeek-cut host uri < http.log \
      | grep -i "\.doc\|\.docx\|\.dot\b"

    # Alternativa 2: via files.log (identificação por magic bytes — mais confiável)
    /opt/zeek/bin/zeek-cut tx_hosts rx_hosts mime_type filename < files.log \
      | grep -i "msword\|openxml\|word"
    ```

## Resultado

```
http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/    [application/msword]
```

---

## Interpretação

Cada elemento da URL revela informações sobre a campanha:

**`ifcingenieria.cl`** — domínio de uma empresa de engenharia chilena legítima. O site foi **comprometido** e utilizado como hospedeiro do malware. Usar sites legítimos como infraestrutura de entrega é uma técnica deliberada: filtros de reputação de domínio raramente bloqueiam sites corporativos ativos.

**`/QpX8It/`** — segmento com aparência de ID aleatório. Esse padrão é característico de painéis C2 que geram caminhos únicos por campanha ou por vítima, dificultando o bloqueio por regras simples de URL.

**`/BIZ/Firmenkunden/`** — *Firmenkunden* significa "clientes corporativos" em alemão. Combinado com o nome do documento (`2018_11Details_zur_Transaktion.doc` — "Detalhes da Transação — Novembro 2018"), a isca é claramente direcionada a um contexto financeiro corporativo de língua alemã ou com operações na Europa.

**Ausência de extensão `.doc` na URL** — o arquivo é servido dinamicamente. Filtros que bloqueiam URLs terminadas em `.exe` ou `.doc` não teriam efeito aqui.

!!! warning "Site legítimo comprometido — implicações para resposta"
    Bloquear o domínio `ifcingenieria.cl` no proxy corporativo é necessário para conter a ameaça imediata. Porém, como se trata de um site legítimo comprometido (não de infraestrutura criminosa dedicada), vale também considerar notificar o proprietário do domínio — uma prática de boa-fé na comunidade de segurança.

---

## IoC registrado

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| URL maliciosa | `http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/` | `http.log` | Alta |
| Site hospedeiro | `ifcingenieria.cl` (comprometido) | `http.log` | Alta |
| Documento lure | `2018_11Details_zur_Transaktion.doc` | `files.log` | Alta |
| MIME type | `application/msword` | `http.log` | Alta |
