# 🛡️ TryHackMe — Writeups & Estudos

Repositório com meus **writeups** e anotações da plataforma [TryHackMe](https://tryhackme.com), enquanto percorro uma trilha de estudos focada em **operações de segurança (SOC / Blue Team)** e fundamentos de pentest.

O objetivo não é só registrar as respostas — é documentar a **metodologia**: por que cada passo foi dado, quais ferramentas usei e o que aprendi. As flags são omitidas de propósito (a TryHackMe desencoraja expor respostas de salas ativas).

---

## 📚 Writeups

| # | Sala | Dificuldade | SO | Foco | Writeup |
|---|------|-------------|-----|------|---------|
| 1 | OpenVPN | Easy | — | Setup de conexão VPN | — |
| 2 | Vulnversity | Easy | Linux | Recon, upload malicioso, privesc SUID | [Ver](./vulnversity/) |
| 3 | Blue | Easy | Windows | EternalBlue (MS17-010), Metasploit | [Ver](./blue/) |
| 4 | Kenobi | Easy | Linux | Samba, ProFTPD, NFS, PATH hijacking | [Ver](./kenobi/) |

---

## 🧭 Metodologia

Sigo sempre a mesma sequência em cada sala — vira hábito e acelera o aprendizado:

1. **Conectar** — VPN ou AttackBox, anotar o IP alvo.
2. **Enumerar portas** — descobrir portas abertas e versões dos serviços (`nmap`).
3. **Enumerar a web** — código-fonte, diretórios, parâmetros (`gobuster`, `ffuf`, Burp).
4. **Enumerar serviços** — FTP, SMB, NFS, SSH por acesso anônimo e versões vulneráveis.
5. **Obter acesso** — explorar a falha encontrada e conseguir um shell.
6. **Escalar privilégios** — `sudo -l`, binários SUID, cron, credenciais salvas.
7. **Capturar as flags** — ler os arquivos indicados.
8. **Documentar** — registrar a falha, como detectei, como explorei e como corrigir.

---

## 🧰 Ferramentas usadas até aqui

`nmap` · `gobuster` · `Burp Suite` · `netcat` · `smbclient` · `searchsploit` · `Metasploit` · `hashcat` / CrackStation · `mount` (NFS) · GTFOBins

---

## ⚠️ Aviso

Todo o conteúdo aqui foi praticado em **ambiente autorizado** (laboratórios da TryHackMe). As técnicas são documentadas para fins **educacionais**. Nunca use em sistemas sem permissão explícita.

---

*Em constante atualização conforme avanço na trilha. 🚀*
