# MITRE ATT&CK — Emotet

## O que é o MITRE ATT&CK?

O **MITRE ATT&CK** (*Adversarial Tactics, Techniques, and Common Knowledge*) é uma base de conhecimento mantida pela organização sem fins lucrativos MITRE, que documenta as táticas e técnicas utilizadas por adversários reais em ataques cibernéticos. É hoje o framework de referência mais adotado pela indústria de segurança para descrever comportamento de ameaças.

A estrutura é organizada em **táticas** (o *porquê* — o objetivo do atacante) e **técnicas** (o *como* — o método utilizado).

---

## Mapeamento do caso Emotet

| ID | Tática | Técnica | Evidência no caso |
|---|---|---|---|
| [T1566.001](https://attack.mitre.org/techniques/T1566/001/) | Initial Access | Phishing: Spearphishing Link | URL `ifcingenieria.cl` → `http.log` |
| [T1204.002](https://attack.mitre.org/techniques/T1204/002/) | Execution | User Execution: Malicious File | Abertura do `.doc` com macro |
| [T1059.005](https://attack.mitre.org/techniques/T1059/005/) | Execution | Command & Scripting: Visual Basic | Macro VBA no Word |
| [T1547.001](https://attack.mitre.org/techniques/T1547/001/) | Persistence | Registry Run Keys / Startup Folder | Comportamento padrão Emotet |
| [T1071.001](https://attack.mitre.org/techniques/T1071/001/) | Command & Control | Application Layer Protocol: Web | Conexões SMTP — `smtp.log` |
| [T1021.002](https://attack.mitre.org/techniques/T1021/002/) | Lateral Movement | Remote Services: SMB/Admin Shares | SMB FILE_OPEN — `smb_files.log` |
| [T1078](https://attack.mitre.org/techniques/T1078/) | Defense Evasion | Valid Accounts | Emotet exfiltra credenciais locais |
| [T1568](https://attack.mitre.org/techniques/T1568/) | Command & Control | Dynamic Resolution | Múltiplos IPs/domínios SMTP |

---

## Grupo operador — G0046 Mealybug

O Emotet é atribuído ao grupo **Mealybug** (também conhecido como TA542 e Gold Crestwood). O grupo opera desde pelo menos 2014 e é responsável por distribuir o Emotet como *Malware-as-a-Service* para outros grupos criminosos.

Referência completa: [attack.mitre.org/groups/G0046](https://attack.mitre.org/groups/G0046/)
