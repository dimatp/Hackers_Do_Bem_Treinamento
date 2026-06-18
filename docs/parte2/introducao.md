# Parte 2 — Análise Forense com Autopsy

## O que é o Autopsy?

O **Autopsy** é uma plataforma open-source de análise forense digital, desenvolvida pela Basis Technology. Ele funciona como uma interface gráfica sobre o **The Sleuth Kit (TSK)** — uma coleção de ferramentas de linha de comando para análise de sistemas de arquivos.

O Autopsy automatiza grande parte do trabalho de extração de artefatos: ao invés de inspecionar manualmente o registro do Windows ou parsear arquivos de histórico de browser, os módulos de ingestão fazem isso automaticamente e organizam os resultados em categorias navegáveis.

---

## O que é uma imagem forense?

Uma **imagem forense** é uma cópia bit-a-bit de um dispositivo de armazenamento — cada setor do disco original é copiado exatamente, incluindo espaço não alocado, arquivos deletados e metadados do sistema de arquivos. Diferente de uma cópia convencional, a imagem forense:

- Preserva arquivos deletados (ainda presentes no espaço não alocado)
- Preserva metadados de timestamps (MAC times)
- Inclui um hash de verificação para garantir integridade
- Pode ser analisada sem risco de alterar a evidência original

O formato `.E01` (EnCase) utilizado neste caso inclui o hash embutido e suporta compressão, sendo um dos formatos mais aceitos em processos judiciais.

---

## Artefatos do Windows analisados

| Artefato | Localização | O que revela |
|---|---|---|
| Registro do Windows | `Windows\System32\config\` | SO, proprietário, conta, desligamento |
| Programas instalados | `SOFTWARE\...\Uninstall` | Softwares instalados |
| Run Programs | `NTUSER.DAT\...\UserAssist` | Programas executados e frequência |
| Web History | Arquivos de cache do browser | Sites visitados |
| Shell Bags | `NTUSER.DAT\...\BagMRU` | Pastas abertas no Explorer |
| Recycle Bin | `RECYCLER\` | Arquivos deletados via lixeira |
| Recent Documents | `NTUSER.DAT\...\RecentDocs` | Documentos acessados recentemente |
| USB Devices | `SYSTEM\...\Enum\USBSTOR` | Dispositivos USB conectados |

---

## Questões respondidas nesta parte

| Questão | O que responde | Artefato principal |
|---|---|---|
| [Q6](q6-hash-so.md) | Integridade da imagem e SO | Hash MD5 + Registro |
| [Q7](q7-perfil.md) | Proprietário, conta, desligamento, logon | Registro do Windows |
| [Q8](q8-programas-lixeira.md) | Programas suspeitos e lixeira | Installed Programs + Recycle Bin |
| [Q9](q9-timeline.md) | Timeline das evidências | Autopsy Timeline |
| [Q10](q10-laudo.md) | Laudo Pericial completo | — |
