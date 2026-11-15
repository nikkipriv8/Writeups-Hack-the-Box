# 🦖 Writeups — Hack The Box

Repositório pessoal com meus writeups de máquinas do [Hack The Box](https://www.hackthebox.com/).
O objetivo é documentar o raciocínio por trás de cada máquina — recon, enumeração, exploração e escalação de privilégio — de forma clara e reproduzível.

> ⚠️ Todos os writeups são de máquinas **retired** (aposentadas). Nenhuma flag é publicada, apenas a metodologia.

---

## 📂 Máquinas

| Máquina | SO | Dificuldade | Tópicos principais |
|---------|-----|-------------|--------------------|
| [Bashed](./Bashed) | Linux | Easy | Web shell exposto · sudo misconfig · cron como root |
| [Cap](./Cap) | Linux | Easy | IDOR (download de PCAP) · credenciais em tráfego · Linux capabilities |
| [Lame](./Lame) | Linux | Easy | Samba `username map script` (CVE-2007-2447) → root |
| [Mirai](./Mirai) | Linux (Raspberry Pi) | Easy | Credenciais padrão · sudo · recuperação de dados em dispositivo |
| [Toolbox](./Toolbox) | Windows | Easy | SQLi (PostgreSQL) · RCE em container Docker · acesso ao host |
| [Writeup](./Writeup) | Linux (Devuan) | Easy | SQLi (CVE-2019-9053) · Password Cracking · PATH Hijacking |

---

## 🛠️ Ferramentas que aparecem por aqui

`nmap` · `searchsploit` · `hashcat` · `john` · `pspy` · `ffuf` · `burp suite` · `smbclient` · `netexec`

---

## 👤 Sobre mim

Estudante de Defesa Cibernética (FIAP), focado em **SOC / Blue Team**, estudando o lado ofensivo para defender melhor.

- 🔗 LinkedIn: [kayke-matos](https://www.linkedin.com/in/kayke-matos)
- 🎯 HackTheBox: [perfil](https://profile.hackthebox.com/profile/019ce967-318a-7077-bdbe-756e6047ae28)
