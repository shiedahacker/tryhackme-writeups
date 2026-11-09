# TryHackMe: Vulnversity

> **Plataforma:** TryHackMe · **Dificuldade:** Fácil · **Sistema-alvo:** Linux (servidor web)
> **Temas:** reconhecimento, enumeração de diretórios web, upload de arquivo malicioso, escalação de privilégio
> **Ferramentas:** Nmap, Gobuster, Burp Suite (proxy e Intruder), Netcat, GTFOBins

As flags da sala foram omitidas de propósito, seguindo as boas práticas da comunidade do TryHackMe. O objetivo deste write-up é mostrar o raciocínio e, principalmente, como um analista de SOC detectaria e evitaria esse ataque.

## Visão geral

A sala Vulnversity simula um servidor web com várias falhas encadeadas: um diretório escondido, um formulário de upload com validação fraca e uma configuração insegura de permissões no Linux. Juntas, elas permitem sair de um acesso anônimo pela web até o controle total do servidor como **root**.

## Metodologia do ataque

### 1. Reconhecimento

Comecei com uma varredura de portas e serviços:

```bash
nmap -sV <IP_DO_ALVO>
```

A varredura mostrou os serviços em execução e um **servidor web rodando em uma porta não padrão**, que virou o foco da investigação.

### 2. Enumeração de diretórios

Usei o Gobuster para descobrir diretórios que não aparecem nos links do site:

```bash
gobuster dir -u http://<IP_DO_ALVO>:<PORTA> -w <WORDLIST>
```

O Gobuster testa milhares de nomes comuns de diretórios a partir de uma lista de palavras (wordlist). Assim encontrei o **diretório com a página de upload de arquivos**, que não estava acessível pela navegação normal.

### 3. Acesso inicial: contornando o filtro de upload

O formulário bloqueava arquivos `.php`, mas o filtro olhava **apenas a extensão** do arquivo. Configurei o **Burp Suite como proxy** para interceptar a requisição de upload e, com o **Intruder**, testei uma lista de extensões que o servidor também executa como PHP. A extensão **`.phtml`** passou pelo filtro.

Em seguida:

1. Preparei uma **reverse shell em PHP** apontando para o meu IP da VPN e salvei o arquivo com a extensão `.phtml`.
2. Deixei o **Netcat** escutando na minha máquina: `nc -lvnp <PORTA>`.
3. Enviei o arquivo e acessei a URL dele no servidor, o que fez o servidor se conectar de volta para mim.

Com isso, ganhei uma shell no servidor como o usuário **`www-data`**, o usuário do serviço web.

### 4. Escalação de privilégio

Procurei arquivos com o bit **SUID** ativado, que rodam com as permissões do dono do arquivo (muitas vezes o root), e não com as de quem os executa:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Na lista apareceu o **`/bin/systemctl` com SUID**, uma configuração errada: o `systemctl` gerencia os serviços do sistema, e com SUID qualquer usuário consegue criar e iniciar serviços como root. Seguindo a técnica documentada no **GTFOBins** para o `systemctl`, criei um serviço que executou meus comandos com privilégios de administrador e **virei root**.

## Visão do defensor (Blue Team)

### Como detectar

| Fase do ataque | O que procurar | Fonte |
|---|---|---|
| Reconhecimento | Um único IP testando várias portas em pouco tempo | Logs de firewall, IDS/IPS |
| Enumeração | Rajada de requisições com muitos erros 404 vindas do mesmo IP, às vezes com o User-Agent da ferramenta | Logs de acesso do servidor web (Apache/Nginx), WAF |
| Upload malicioso | Requisições POST para a página de upload seguidas de acesso direto ao arquivo enviado | Logs do servidor web |
| Reverse shell | O processo do servidor web abrindo uma conexão de saída para um IP desconhecido | Logs de firewall, EDR, `auditd` |
| Escalação | O usuário `www-data` executando o `systemctl` e criando serviços novos | `auditd`, logs do sistema (`journalctl`) |

Um bom indicador para um alerta no SIEM: **o usuário do servidor web nunca deveria abrir shells nem criar serviços no sistema.**

### Como prevenir

- **Validar o tipo real do arquivo**, pelo conteúdo, e não só pela extensão. Um `.phtml` é tão executável quanto um `.php`.
- **Usar uma lista branca de extensões**, aceitando apenas as necessárias (como `.jpg` e `.pdf`), em vez de tentar bloquear as perigosas uma a uma.
- **Guardar os arquivos enviados fora da pasta pública do site** e sem permissão de execução.
- **Nunca dar SUID a binários como o `systemctl`.** Remover o SUID de tudo que não precisa dele e auditar periodicamente com o mesmo comando `find` usado no ataque.
- **Filtrar o tráfego de saída**, porque um servidor web raramente precisa iniciar conexões para IPs aleatórios.
- **Usar um WAF e limitar a taxa de requisições**, para dificultar a enumeração automatizada.

## Mapeamento MITRE ATT&CK

| Tática | Técnica | Na sala |
|---|---|---|
| Reconhecimento | T1595.003: Active Scanning: Wordlist Scanning | Gobuster |
| Descoberta | T1046: Network Service Discovery | Varredura com o Nmap |
| Acesso inicial | T1190: Exploit Public-Facing Application | Contorno do filtro de upload |
| Persistência | T1505.003: Server Software Component: Web Shell | Arquivo `.phtml` enviado |
| Execução | T1059.004: Command and Scripting Interpreter: Unix Shell | Reverse shell |
| Escalação de privilégio | T1548.001: Abuse Elevation Control Mechanism: Setuid and Setgid | `/bin/systemctl` com SUID |

## O que aprendi

- **Ferramentas:** Nmap, Gobuster, Burp Suite (proxy e Intruder), reverse shell em PHP com Netcat e escalação de privilégio via SUID.
- Um filtro que olha só a extensão do arquivo é fácil de contornar. Validação de upload precisa ser feita com lista branca e checagem do tipo real do arquivo.
- O mesmo comando que o atacante usa para achar binários SUID serve para o defensor auditar o sistema.
- Falhas pequenas se encadeiam: um diretório escondido, um filtro fraco e uma permissão errada viraram um comprometimento total do servidor.
