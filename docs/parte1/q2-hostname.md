# Q2 — Hostname do host comprometido

## A pergunta

> Qual é o nome do host do cliente Windows em 172.17.1.129?

---

## Comando

```bash
/opt/zeek/bin/zeek-cut client_addr mac host_name < dhcp.log \
  | grep "172.17.1.129" | sort -u
```

## Resultado

```
172.17.1.129    00:1e:67:4a:d7:5c    Nalyvaiko-PC
```

---

## Interpretação

O hostname `Nalyvaiko-PC` segue o padrão `<sobrenome>-PC` — convenção comum em ambientes onde o usuário nomeou a própria máquina, ou onde o departamento de TI adota o nome do colaborador como identificador.

Esse padrão é distinto do que normalmente se vê em ambientes corporativos maduros, onde hostnames costumam seguir um esquema padronizado (ex: `WKS-00342`, `CORP-LPT-147`). A ausência desse padrão pode indicar uma empresa com controle de TI menos centralizado — o que é relevante para o contexto do incidente.

O domínio corporativo `kyivartworks.com` foi identificado no mesmo `dhcp.log`, confirmando que a máquina pertence ao ambiente de Active Directory da empresa.

!!! info "Hostname como IoC"
    O hostname é um IoC de média-alta utilidade: ele não é globalmente único como um IP público ou hash de arquivo, mas dentro de uma rede corporativa permite identificar rapidamente o ativo nos logs de outros sistemas (SIEM, EDR, firewall).

---

## IoC registrado

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| Hostname | `Nalyvaiko-PC` | `dhcp.log` | Alta |
| Domínio AD | `kyivartworks.com` | `dhcp.log` / `smb_files.log` | Alta |
