# Q1 — Endereço MAC do host comprometido

## A pergunta

> Qual é o endereço MAC do cliente Windows em 172.17.1.129?

---

## Por que o dhcp.log?

Quando um dispositivo entra na rede, ele solicita um endereço IP ao servidor DHCP. Nessa negociação, o servidor registra tanto o IP atribuído quanto o **MAC address** do dispositivo — que é o identificador físico único da placa de rede.

O Zeek captura esse diálogo e o registra no `dhcp.log`, tornando-o a fonte mais confiável para correlacionar IP ↔ MAC ↔ hostname em um único log.

---

## Comando

```bash
# Confirmar os campos disponíveis no log desta versão do Zeek
grep "^#fields" dhcp.log

# Extrair IP e MAC, filtrar pelo IP de interesse
/opt/zeek/bin/zeek-cut client_addr mac < dhcp.log \
  | grep "172.17.1.129" | sort -u
```

## Resultado

```
172.17.1.129    00:1e:67:4a:d7:5c
```

---

## Interpretação

O MAC address `00:1e:67:4a:d7:5c` identifica unicamente a interface de rede do host comprometido.

**Prefixo OUI `00:1e:67` → Intel Corporation**

O OUI (*Organizationally Unique Identifier*) são os três primeiros octetos do MAC address, registrados pelo IEEE para cada fabricante. O prefixo `00:1e:67` pertence à Intel — indicando que a NIC (*Network Interface Card*) é fabricada pela Intel, o que é consistente com um notebook ou desktop corporativo comum da época.

!!! tip "Por que o MAC importa para a investigação?"
    O endereço IP pode mudar a cada reconexão à rede (DHCP dinâmico). O MAC address, por ser fixo na placa de rede, permite rastrear o mesmo dispositivo mesmo que ele mude de IP ao longo do tempo. Em logs de rede, o par MAC + IP é a âncora de identidade do dispositivo.

---

## IoC registrado

| Tipo | Valor | Fonte | Confiança |
|---|---|---|---|
| MAC address | `00:1e:67:4a:d7:5c` | `dhcp.log` | Alta |
| IP interno | `172.17.1.129` | `dhcp.log` | Alta |
