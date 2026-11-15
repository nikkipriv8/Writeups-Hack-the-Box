# Bashed — Hack The Box

| | |
|---|---|
| **Máquina** | Bashed |
| **SO** | Linux |
| **Dificuldade** | Easy |
| **Tópicos** | Web shell exposto · sudo misconfig · cron executando script gravável como root |

---

## Resumo

Um **web shell PHP** (phpbash) deixado pelo desenvolvedor está acessível na aplicação → shell como `www-data`. A escalação passa por um `sudo` mal configurado (para o usuário `scriptmanager`) e um **cron do root** executando um script que você pode editar.

---

## 1. Reconhecimento

```bash
nmap -sC -sV -oA nmap/bashed 10.10.10.X      # 80/tcp Apache
feroxbuster -u http://10.10.10.X --rate-limit 10 -t 10
```

Descoberta importante: diretório **`/dev/`** contendo **`phpbash.php`** — um web shell interativo (o próprio dev comenta no site que desenvolveu o phpbash "localmente nesta máquina").

---

## 2. Foothold

Acessando `http://10.10.10.X/dev/phpbash.php` você já tem **execução de comando como `www-data`**. (Opcional: estabilizar com um reverse shell.)

```bash
cat /home/arrexel/user.txt
```

---

## 3. Escalação de privilégio

```bash
sudo -l
# (scriptmanager) NOPASSWD: ALL  -> www-data pode virar scriptmanager sem senha
sudo -u scriptmanager bash
```

No diretório `/scripts`:
- `test.py` — pertence ao `scriptmanager` (você **escreve**)
- `test.txt` — pertence ao **root**, atualizado a cada minuto

Ou seja: um **cron do root executa `test.py`**. Basta reescrever o `test.py` com um payload (reverse shell ou SUID bash) e esperar o cron rodar como root.

```bash
cat /root/root.txt
```

---

## Lições aprendidas

- **Ferramenta de dev (web shell) esquecida em produção** = RCE de graça. Nunca deixar artefatos de teste expostos.
- Cadeia **`sudo` → arquivo gravável executado por cron root** é um dos privescs mais clássicos do Linux.

### Perspectiva Blue Team / detecção

- Monitorar criação de arquivos `.php` suspeitos em diretórios web (File Integrity Monitoring).
- Auditar `sudoers` e `crontab`; alertar quando um script executado pelo root é **gravável** por usuário comum.
- Revisão de código / varredura antes de deploy para remover web shells e backdoors de teste.
