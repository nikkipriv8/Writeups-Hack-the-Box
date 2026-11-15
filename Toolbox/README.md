# Toolbox — Hack The Box

| | |
|---|---|
| **Máquina** | Toolbox |
| **SO** | Windows |
| **Dificuldade** | Easy |
| **Tópicos** | SQL Injection (PostgreSQL) · RCE em container Docker · acesso ao host via boot2docker |

---

## Resumo

Um portal web com **SQL Injection** (backend PostgreSQL) permite **RCE dentro de um container Docker** (boot2docker). A partir do container, chega-se ao **host Windows** usando as credenciais padrão do boot2docker, lendo a flag de root.

> ⚠️ Toolbox tem variações no caminho do Docker — **confere e ajusta** conforme o que você fez na sua resolução.

---

## 1. Reconhecimento

```bash
nmap -sC -sV -p- -oA nmap/toolbox 10.10.10.X
```

- **21** — FTP (login anônimo)
- **443** — HTTPS (site "MegaLogistics")
- **445 / 5985** — SMB / WinRM
- porta extra de **SSH do boot2docker**

FTP anônimo revela arquivos; o site em `https://10.10.10.X/` tem um painel de **login do admin**.

---

## 2. Foothold — SQL Injection (PostgreSQL)

O login do admin é vulnerável a **SQLi**. Com `sqlmap` dá pra obter execução de comando:

```bash
sqlmap -r login.req --batch --os-shell
# backend: PostgreSQL -> os-shell via stacked queries / COPY ... TO PROGRAM
```

A shell obtida está **dentro de um container Docker** (boot2docker / TinyCore Linux). A `user.txt` é obtida aqui.

---

## 3. Escalação / acesso ao host Windows

O **boot2docker** expõe SSH com credenciais padrão **`docker:tcuser`**:

```bash
ssh docker@10.10.10.X -p <porta_boot2docker>   # senha: tcuser
```

O disco `C:` do host Windows fica **montado** dentro do boot2docker. Navegando até o diretório do host:

```bash
cat /c/Users/Administrator/Desktop/root.txt
```

---

## Lições aprendidas

- **SQL Injection** continua no topo do OWASP — usar *prepared statements* / consultas parametrizadas.
- **Credenciais padrão do Docker Toolbox** (`docker:tcuser`) dão acesso à VM que hospeda os containers.
- Isolamento de container mal feito (disco do host **montado** no container) quebra a barreira container↔host.

### Perspectiva Blue Team / detecção

- WAF e detecção de padrões de SQLi; monitorar erros/tempos anômalos no banco.
- Trocar credenciais padrão de Docker/boot2docker; restringir SSH.
- Revisar **montagens de volume** Docker — nunca montar o disco do host inteiro num container exposto.
