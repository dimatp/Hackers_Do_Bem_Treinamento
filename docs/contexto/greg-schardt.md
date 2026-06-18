# Caso Greg Schardt — NIST CFREDS

## O dataset NIST CFREDS

A imagem de disco utilizada na Parte 2 é parte do projeto **Computer Forensics Reference Data Sets (CFREDS)**, mantido pelo [National Institute of Standards and Technology (NIST)](https://www.nist.gov/). O CFREDS foi criado para fornecer evidências digitais padronizadas para treinamento e validação de ferramentas forenses.

O caso específico utilizado aqui é o **"Hacking Case"** do CFREDS — um dos mais conhecidos datasets forenses públicos, usado em cursos e certificações de forense digital ao redor do mundo.

!!! info "Dataset público"
    A imagem está disponível em: `https://cfreds-archive.nist.gov/images/`
    Arquivos: `4Dell Latitude CPi.E01` e `4Dell Latitude CPi.E02`
    Hash MD5 esperado: `aee4fcd9301c03b3b054623ca261959a`

---

## O contexto narrativo

Após a análise de rede da Parte 1, a Good Money Financial reportou o incidente à **Delegacia de Crimes Cibernéticos (DCC)**. De posse do IP e porta utilizados durante o ataque, a delegacia acionou a operadora de internet e identificou o titular da conexão: **Greg Schardt**.

Uma operação de busca e apreensão foi realizada na residência do suspeito. Entre os itens apreendidos, um **notebook Dell Latitude CPi** — o equipamento que possivelmente foi utilizado para cometer os crimes.

O notebook foi devidamente apreendido, documentado e encaminhado para perícia. Uma imagem forense do disco foi gerada com hash MD5 para garantir a integridade da cadeia de custódia.

---

## O equipamento periciado

| Campo | Valor |
|---|---|
| **Equipamento** | Notebook Dell Latitude CPi |
| **Sistema Operacional** | Windows 98 |
| **Período de uso** | ~2004 |
| **Imagem** | 4Dell Latitude CPi.E01 + E02 (formato EnCase) |
| **Hash MD5** | `aee4fcd9301c03b3b054623ca261959a` |

O formato `.E01` é o formato de imagem forense do **EnCase** — um dos softwares forenses mais utilizados em investigações criminais. Diferente de uma cópia simples de disco, o E01 inclui metadados da coleta, hash de verificação embutido e compressão opcional, garantindo a integridade da evidência.

---

## Por que Windows 98 em 2004?

O gap entre o lançamento do Windows 98 (1998) e a data do incidente (2004) é propositalmente realista. Em ambientes domésticos e de pequenas empresas no início dos anos 2000, era comum encontrar máquinas rodando sistemas operacionais desatualizados — sem patches de segurança, sem antivírus atualizado, e sem as proteções que hoje consideramos básicas.

Esse contexto é importante para a análise forense: artefatos do Windows 98 diferem dos sistemas modernos em estrutura de registro, localização de arquivos de usuário e formato de metadados.

---

## Cadeia de custódia

!!! warning "Conceito fundamental"
    A **cadeia de custódia** é o registro documentado de cada pessoa que teve acesso à evidência, desde a coleta até a apresentação em juízo. Qualquer ruptura na cadeia de custódia pode tornar a evidência inadmissível.

    No contexto deste estudo de caso, a verificação do hash MD5 da imagem (`aee4fcd9301c03b3b054623ca261959a`) é o primeiro ato do perito — e o mais crítico. Um hash divergente invalida toda a análise subsequente.
