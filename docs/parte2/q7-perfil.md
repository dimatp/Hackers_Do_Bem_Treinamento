# Q7 — Perfil do suspeito

## Q7a — Proprietário registrado

```
Autopsy → Data Sources → SOFTWARE → Microsoft → Windows NT → CurrentVersion
  Valor: RegisteredOwner
```

**Resultado: Greg Schardt**

O nome registrado no SO confirma a posse do equipamento e é consistente com as informações obtidas pela operadora de internet.

---

## Q7b — Nome da conta do computador

```
Autopsy → Data Sources → SYSTEM → ControlSet001 → Control → ComputerName → ComputerName
  Valor: ComputerName
```

---

## Q7c — Último desligamento

```
Autopsy → Results → Recent Activity → Shutdown Time

Ou manualmente:
SYSTEM → ControlSet001 → Control → Windows
  Valor: ShutdownTime  (formato FILETIME — Autopsy converte automaticamente)
```

!!! tip "Por que o último desligamento importa?"
    O timestamp de desligamento delimita o escopo temporal da investigação. Se o desligamento ocorreu logo após atividades suspeitas, pode indicar uma tentativa deliberada de interromper processos em execução ou encerrar conexões de rede antes da apreensão.

---

## Q7d — Último usuário a fazer logon

```
SOFTWARE → Microsoft → Windows NT → CurrentVersion → Winlogon
  Valor: DefaultUserName  ou  LastLoggedOnUser
```

!!! info "Windows 98 e controle de acesso"
    O Windows 98 não tem um sistema robusto de controle de acesso multi-usuário como os sistemas NT. O campo `DefaultUserName` registra o último usuário que fez logon na tela de boas-vindas, mas o sistema pode ser iniciado sem autenticação.
