# O que é o Emotet

## Origem e história

O Emotet surgiu em 2014 como um **trojan bancário** simples, projetado para roubar credenciais de acesso a serviços financeiros. Ao longo dos anos seguintes, evoluiu para se tornar uma das ameaças mais sofisticadas e persistentes já documentadas — tanto que o Departamento de Segurança Interna dos EUA (DHS) o descreveu como "o malware mais custoso e destrutivo que afeta governos estaduais, locais, tribais e territoriais".

Em janeiro de 2021, uma operação coordenada entre autoridades de 8 países desmantelou a infraestrutura do Emotet. Em novembro do mesmo ano, o malware ressurgiu. Em 2022, uma nova operação coordenada pela Europol voltou a derrubá-lo.

!!! info "Grupo operador"
    O Emotet é operado pelo grupo **Mealybug** (também referenciado como TA542 ou Gold Crestwood). Referência: [MITRE ATT&CK — G0046](https://attack.mitre.org/groups/G0046/).

---

## Como o Emotet funciona

O Emotet opera em **três fases principais**:

### 1. Entrega (Delivery)
O vetor de entrada preferencial é o **phishing por e-mail com documento Office anexado ou linkado**. O documento contém uma macro VBA que, ao ser executada pelo usuário, inicia a cadeia de infecção.

O Emotet é conhecido por usar técnicas de **thread hijacking** — ele sequestra conversas de e-mail legítimas e responde a elas com o documento malicioso, tornando a isca altamente convincente.

No caso analisado, o documento foi servido a partir de um **site legítimo comprometido** (`ifcingenieria.cl`), o que reduz a eficácia de filtros baseados em reputação de domínio.

### 2. Instalação e Persistência
Após a execução da macro, o Emotet baixa e instala seu payload principal — no caso analisado, `6169583.exe`. O binário é tipicamente compilado poucas horas antes da infecção (no caso: ~1h30 de intervalo entre `compile_ts` e o download), tornando as assinaturas de antivírus ineficazes.

A persistência é mantida via **chaves de registro** (`HKCU\Software\Microsoft\Windows\CurrentVersion\Run`).

### 3. Atividade pós-infecção
Uma vez instalado, o Emotet pode:

- **Exfiltrar credenciais** armazenadas no navegador e no sistema
- **Ativar módulo de spam** — transforma o host infectado em um nó distribuidor de phishing
- **Realizar reconhecimento de rede** — acessa o Active Directory via SMB para mapear a infraestrutura
- **Baixar payloads secundários** — historicamente distribuiu Trickbot, QakBot e Ryuk ransomware

---

## Por que o compile timestamp importa

```
pe.log → compile_ts: 1542051170 → 12/11/2018 ~19:32 UTC
http.log → download do .exe: 12/11/2018 ~21:01 UTC
Intervalo: ~1h30
```

Esse padrão é uma **assinatura comportamental do Emotet**: os binários são compilados sob demanda, pouco antes de cada campanha. Isso serve a dois propósitos:

1. **Evasão de antivírus** — hash nunca visto antes nos bancos de assinaturas
2. **Descartabilidade** — se o C2 for derrubado, um novo binário é gerado e distribuído rapidamente

Um binário compilado no mesmo dia da infecção, com intervalo de horas, é um forte indicador de Emotet mesmo sem análise de código.

---

## Módulo de spam — o que os logs revelam

O `smtp.log` registrou **11 conexões SMTP** saindo do host infectado para 4 servidores distintos, aproximadamente 1h30 após a infecção inicial:

| Servidor | Conexões | Observação |
|---|---|---|
| `108.177.98.108` | 7x | IP do Google/Gmail — relay legítimo abusado |
| `185.93.245.68` | 1x | — |
| `41.204.202.10` | 1x | — |
| `123.125.50.11` | 1x | IP da rede 163.com (China) |

!!! warning "Por que usar Gmail como relay?"
    O Emotet usa servidores de e-mail legítimos (Gmail, Yahoo, Outlook) como relay para aumentar a taxa de entrega — e-mails originados desses servidores raramente são bloqueados por filtros de spam corporativos. O tráfego TLS para `smtp.gmail.com` em uma estação de trabalho comum é, em si, um indicador suspeito.

---

## Reconhecimento de Active Directory

O acesso ao GPO (`Group Policy Object`) do domínio via SMB é uma das técnicas mais documentadas do Emotet para **movimentação lateral**:

```
smb_files.log → 4x SMB::FILE_OPEN
Destino: 172.17.1.2 (Controlador de Domínio)
Arquivo: kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
```

O GPO `{31B2F340-016D-11D2-945F-00C04FB984F9}` é o **Default Domain Policy** — presente em todo ambiente Active Directory com configuração padrão. Acessá-lo permite ao malware mapear a estrutura do domínio e identificar alvos para propagação.
