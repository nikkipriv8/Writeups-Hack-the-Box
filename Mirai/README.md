# Mirai — Hack The Box

| | |
|---|---|
| **Máquina** | Mirai |
| **SO** | Linux (Raspberry Pi) |
| **Dificuldade** | Easy |
| **Tópicos** | Credenciais padrão (pi:raspberry) · sudo · recuperação de dados em dispositivo |

---

## Resumo

Um **Raspberry Pi** rodando Pi-hole com as **credenciais SSH padrão** (`pi:raspberry`). O acesso root é trivial via `sudo`; o verdadeiro desafio é **recuperar a `root.txt`** que foi "apagada" de um pendrive — um exercício de forense.

---

## 1. Reconhecimento

```bash
nmap -sC -sV -p- -oA nmap/mirai 10.10.10.X
```

- **22** — OpenSSH
- **53** — dnsmasq
- **80 / 32400** — Pi-hole / Plex

> 📌 Pi-hole é a pista: Pi-hole roda tipicamente em **Raspberry Pi**, que tem credenciais de fábrica conhecidas.

---

## 2. Foothold — credenciais padrão

```bash
ssh pi@10.10.10.X      # senha: raspberry
cat /home/pi/user.txt
```

---

## 3. Escalação de privilégio + recuperação da flag

```bash
sudo -l       # (ALL : ALL) NOPASSWD
sudo su       # root imediato
```

A `root.txt`, porém, traz uma mensagem: a flag foi **apagada** de um pendrive montado em `/media/usbstick`. Dados deletados continuam no dispositivo bruto até serem sobrescritos:

```bash
strings /dev/sdb        # revela o conteúdo residual / backup da flag
```

---

## Lições aprendidas

- **Credenciais padrão** são porta escancarada — trocar sempre em qualquer device/serviço.
- Dados "apagados" permanecem no **dispositivo físico** até sobrescritos → base de recuperação forense.

### Perspectiva Blue Team / DFIR

- Política de troca de credenciais de fábrica (IoT/embarcados são alvos clássicos — o próprio nome "Mirai" referencia a botnet de IoT).
- Entender **recuperação de dados deletados** (`strings`, `foremost`, `photorec`) é skill direto de DFIR — útil pra CJCA.
- Para apagar de verdade: *wipe* seguro, não só `rm`.
