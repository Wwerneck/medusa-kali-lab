# Laboratório de Auditoria de Credenciais com Kali Linux e Medusa

![Status](https://img.shields.io/badge/status-laboratório%20educacional-blue)
![Uso](https://img.shields.io/badge/uso-autorizado%20e%20isolado-red)
![Kali](https://img.shields.io/badge/Kali%20Linux-supported-557C94)
![Medusa](https://img.shields.io/badge/Medusa-2.3-orange)

Projeto prático desenvolvido para estudar, em ambiente controlado, como tentativas automatizadas de autenticação podem afetar serviços FTP, SMB e aplicações web vulneráveis. O laboratório usa Kali Linux como máquina de auditoria e Metasploitable 2/DVWA como alvos intencionalmente vulneráveis.

> **Aviso legal e ético:** execute os procedimentos somente em máquinas próprias, autorizadas e isoladas. Nunca aponte o Medusa para IPs públicos, sistemas de terceiros, redes corporativas ou contas reais. O objetivo é aprender a identificar e corrigir riscos, não obter acesso indevido.

## Sumário

- [Objetivos](#objetivos)
- [Arquitetura do laboratório](#arquitetura-do-laboratório)
- [Pré-requisitos](#pré-requisitos)
- [Configuração das VMs](#configuração-das-vms)
- [Preparação do Kali](#preparação-do-kali)
- [Cenário 1 — FTP](#cenário-1--ftp)
- [Cenário 2 — formulário web/DVWA](#cenário-2--formulário-webdvwa)
- [Cenário 3 — SMB e password spraying](#cenário-3--smb-e-password-spraying)
- [Registro de evidências](#registro-de-evidências)
- [Mitigações](#mitigações)
- [Resultados e reflexão](#resultados-e-reflexão)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Referências](#referências)

## Objetivos

1. Entender o risco de senhas fracas e reutilizadas.
2. Identificar serviços expostos em uma VM de laboratório.
3. Usar o Medusa de forma consciente e limitada.
4. Comparar força bruta direcionada com password spraying.
5. Documentar evidências sem publicar credenciais reais.
6. Relacionar cada achado a uma medida de prevenção.

## Arquitetura do laboratório

![Arquitetura do laboratório](images/arquitetura-laboratorio.svg)

| Componente | Função | Exemplo de IP |
|---|---|---|
| Kali Linux | Máquina de auditoria | `192.168.56.10` |
| Metasploitable 2 | Serviços vulneráveis, incluindo FTP/SMB | `192.168.56.20` |
| DVWA | Aplicação web deliberadamente vulnerável | `192.168.56.30` ou hospedada no Kali |

Use uma rede **Host-only** ou **Internal Network** no VirtualBox. Não use modo bridge para este laboratório. Os endereços são exemplos: substitua pelos endereços realmente atribuídos às suas VMs.

## Pré-requisitos

- VirtualBox atualizado.
- Kali Linux instalado.
- Metasploitable 2 obtido de fonte confiável.
- DVWA instalado localmente ou em uma VM própria.
- Pelo menos 4 GB de RAM disponíveis para as VMs.
- Conhecimentos básicos de Linux, TCP/IP e Git.
- Autorização explícita para todos os testes.

## Configuração das VMs

1. Crie uma rede Host-only no VirtualBox, por exemplo `vboxnet0`.
2. Configure as duas VMs com um adaptador conectado a essa rede.
3. Desative adaptadores extras durante os testes, especialmente bridge.
4. Inicialize as VMs e descubra os IPs dentro de cada sistema:

```bash
ip -br addr
```

5. A partir do Kali, teste apenas a conectividade do laboratório:

```bash
ping -c 3 192.168.56.20
```

6. Faça uma enumeração limitada para confirmar os serviços previstos:

```bash
nmap -Pn -sV --reason -p 21,80,139,445 192.168.56.20
```

Registre o resultado, o horário e o IP em `evidence/scan-example.md`. Não publique endereços ou credenciais de redes reais.

## Preparação do Kali

Verifique a instalação do Medusa:

```bash
sudo apt update
sudo apt install -y medusa nmap smbclient
medusa -V
medusa -d
```

O Medusa é um auditor modular e paralelo de logins. A opção `-M` seleciona o módulo; `-h` define o alvo; `-u`/`-U` definem usuários; `-p`/`-P` definem senhas; `-O` grava um log; `-f` interrompe após encontrar uma combinação válida no host.

Crie wordlists pequenas e específicas para o laboratório. Exemplos deste repositório são fictícios e não devem ser reutilizados em ambientes reais:

```bash
mkdir -p ~/medusa-kali-lab/{wordlists,evidence}
cp wordlists/* ~/medusa-kali-lab/wordlists/
```

## Cenário 1 — FTP

### Hipótese

Um serviço FTP de laboratório aceita autenticação com credenciais fracas. O objetivo é demonstrar por que FTP sem proteção criptográfica, senhas previsíveis e ausência de bloqueio são riscos.

### Verificação manual

Antes de automatizar qualquer tentativa, confirme que o serviço previsto está ativo:

```bash
nc -nv 192.168.56.20 21
```

Encerre a conexão sem testar credenciais reais. Em seguida, execute um teste mínimo, com usuário e senha de laboratório:

```bash
medusa -h 192.168.56.20 -u labuser -p LabOnly-2026! -M ftp -v 4 -O evidence/ftp-single.log
```

### Teste controlado com wordlist

```bash
medusa -h 192.168.56.20 \
  -U wordlists/users-lab.txt \
  -P wordlists/passwords-lab.txt \
  -M ftp \
  -t 2 \
  -f \
  -v 4 \
  -O evidence/ftp-medusa.log
```

O parâmetro `-t 2` mantém o teste pequeno e reduz impacto no serviço. A opção `-f` encerra o host depois de um resultado positivo. Em um relatório real, registre quantidade de tentativas, duração, mensagem do serviço e se houve bloqueio.

### Mitigações para FTP

- Preferir SFTP ou FTPS.
- Desabilitar login anônimo quando não for necessário.
- Usar senhas fortes e únicas.
- Aplicar limitação de tentativas e alertas.
- Restringir o serviço por firewall e rede de gerenciamento.
- Monitorar logs de autenticação e transferências.

## Cenário 2 — formulário web/DVWA

O DVWA é uma aplicação criada para treinamento. Configure o nível de segurança conforme o roteiro da aula e use somente uma conta fictícia.

> O Medusa não substitui a compreensão do protocolo. Formulários web podem exigir parâmetros, cookies, tokens CSRF, mensagens de falha e tratamento de sessão específicos. Antes de usar qualquer módulo, confirme a versão da aplicação e leia sua documentação.

### Procedimento de documentação

1. Acesse o DVWA pelo navegador dentro da rede isolada.
2. Crie uma conta de laboratório, sem dados pessoais.
3. Registre a URL privada, o método HTTP e o campo que representa usuário/senha.
4. Capture uma evidência sem exibir senha válida.
5. Execute no máximo algumas tentativas fictícias e documente o comportamento.
6. Observe se existe bloqueio, atraso, CAPTCHA, token ou mensagem genérica.

Exemplo de checklist:

```text
Alvo: http://192.168.56.30/dvwa/
Conta: labuser (fictícia)
Nível DVWA: baixo / médio / alto
Controle anti-automação observado: __________________
Quantidade máxima planejada: 10 tentativas
Resultado: __________________
```

Não publique um comando copiável apontando para sites reais. O valor pedagógico está em entender a requisição, as defesas e a evidência, não em automatizar ataques fora do laboratório.

### Mitigações para aplicações web

- Hash de senha com Argon2id ou bcrypt e parâmetros atualizados.
- MFA para contas sensíveis.
- Rate limiting por conta, IP e dispositivo.
- Backoff progressivo e bloqueio temporário.
- CAPTCHA adaptativo após comportamento suspeito.
- Tokens CSRF e gerenciamento correto de sessão.
- Mensagens de erro genéricas, sem revelar qual campo falhou.
- Logs, alertas e correlação de tentativas distribuídas.

## Cenário 3 — SMB e password spraying

Password spraying usa uma senha de laboratório contra vários usuários, em vez de testar muitas senhas contra uma única conta. O risco é contornar políticas que bloqueiam uma conta após repetidas falhas.

### Enumeração autorizada

Em Metasploitable 2, confirme o serviço SMB e documente apenas contas fictícias ou já previstas no exercício:

```bash
nmap -Pn -p 139,445 --script smb-os-discovery 192.168.56.20
```

Não enumere usuários de terceiros. O arquivo `wordlists/users-lab.txt` contém nomes exclusivamente didáticos.

### Teste de uma senha controlada

```bash
medusa -h 192.168.56.20 \
  -U wordlists/users-lab.txt \
  -p LabOnly-2026! \
  -M smbnt \
  -t 1 \
  -v 4 \
  -O evidence/smb-spray.log
```

O módulo pode variar conforme a versão do Medusa e do serviço SMB. Consulte `medusa -q -M smbnt` antes do teste. Se o módulo não for compatível, registre a incompatibilidade e use a verificação manual com `smbclient` em vez de insistir:

```bash
smbclient -L //192.168.56.20 -U labuser
```

### Mitigações para SMB

- Desabilitar SMBv1 e protocolos legados.
- Exigir SMB signing quando aplicável.
- Restringir 139/445 a redes necessárias.
- Usar políticas de senha e MFA onde suportado.
- Aplicar bloqueio inteligente sem facilitar negação de serviço.
- Remover contas inativas e privilégios excessivos.
- Monitorar falhas de autenticação e padrões de spraying.

## Registro de evidências

Use o modelo em `evidence/report-template.md` para cada cenário. Uma boa evidência contém:

| Campo | Exemplo |
|---|---|
| Data/hora | 2026-10-28 20:00 BRT |
| Alvo | VM privada `192.168.56.20` |
| Serviço | FTP/SMB/DVWA |
| Comando ou ação | Sem incluir segredos reais |
| Quantidade de tentativas | Número exato |
| Resultado | Sucesso, falha, bloqueio ou incompatibilidade |
| Impacto | O que seria possível fazer no laboratório |
| Mitigação | Controle técnico e operacional |

Antes de publicar:

- Remova senhas, tokens, cookies e chaves.
- Desfoque nomes reais e endereços públicos.
- Não inclua dumps de banco ou dados pessoais.
- Não publique logs que permitam acesso ao alvo.
- Prefira resultados agregados e prints com credenciais ocultas.

## Mitigações gerais

1. **Identidade:** senhas únicas, MFA, cofre de credenciais e menor privilégio.
2. **Aplicação:** rate limiting, CAPTCHA adaptativo, sessão segura e validação de entrada.
3. **Rede:** segmentação, firewall, VPN para administração e serviços mínimos.
4. **Monitoramento:** logs centralizados, alertas de falhas repetidas e revisão periódica.
5. **Processo:** autorização escrita, escopo, janela de teste, plano de parada e relatório.
6. **Resposta:** revogar credenciais expostas, preservar evidências e investigar origem.

## Resultados e reflexão

O aprendizado principal foi perceber que uma ferramenta de auditoria não é uma solução de segurança por si só. O resultado depende da qualidade do escopo, da rede isolada, do limite de tentativas, da interpretação dos logs e da capacidade de transformar o achado em controle preventivo.

Perguntas para completar após a execução:

- Qual serviço aceitou a combinação de laboratório?
- Quantas tentativas foram necessárias?
- Houve limitação, atraso ou bloqueio?
- O que mudou quando a senha foi fortalecida?
- Quais logs registraram o comportamento?
- Qual mitigação teria maior prioridade e por quê?

## Estrutura do repositório

```text
.
├── README.md
├── LICENSE
├── wordlists/
│   ├── users-lab.txt
│   └── passwords-lab.txt
├── evidence/
│   └── report-template.md
└── images/
    ├── arquitetura-laboratorio.svg
    └── fluxo-auditoria.svg
```

## Referências

- [Medusa no Kali Linux](https://www.kali.org/tools/medusa/)
- [Documentação upstream do Medusa](https://jmk-foofus.github.io/medusa/medusa.html)
- [Kali Linux — site oficial](https://www.kali.org/)
- [DVWA — Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [GitHub — About README files](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Credential Stuffing Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html)

## Licença

Este material é disponibilizado para fins educacionais sob a licença MIT. O uso das técnicas permanece condicionado à autorização do proprietário do ambiente.
