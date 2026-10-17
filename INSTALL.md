# NeuroInsight · Sistema da clínica — instalação e operação

Versão: veja o arquivo `VERSION`. Manual de uso (administrador, profissional e recepção):
**`docs/Manual-do-Usuario-NeuroInsight.pdf`**.

Servidor de produção: **Debian 12 com Docker**, máquina `debian12`, IP público `45.224.41.120`,
publicado em **https://clinica.optec.eng.br** pelo Nginx do próprio servidor.

```
Internet ──HTTPS──> Nginx do servidor (443, certificado Let's Encrypt)
                        └──> 127.0.0.1:8081 ──> container app (FastAPI + React)
                                                   └──> container db (PostgreSQL 16, sem porta exposta)
```

> **Três regras que evitam quase todos os problemas**
> 1. Entre como root com **`su -`** (com hífen) ou `sudo -i`. Com `su` sem hífen, `nginx` e `certbot` dão
>    "comando não encontrado". Se já estiver numa sessão assim: `export PATH=$PATH:/usr/sbin:/sbin`.
> 2. **Nunca apague a pasta `/opt/optec-clinica`.** Atualizações são extraídas **por cima**. O arquivo `.env`
>    guarda a senha do banco e a chave que cifra os prontuários.
> 3. **Guarde uma cópia do `.env` fora do servidor** (pendrive, cofre de senhas). Sem ele, o banco não abre e os
>    dados cifrados não podem ser lidos — nem pelos backups.

---

## 1. Primeira instalação

### 1.1 Conferir o DNS (no servidor)
O registro **A** `clinica.optec.eng.br` precisa apontar para o IP deste servidor.
```bash
getent hosts clinica.optec.eng.br; hostname -I
```
O IP do primeiro comando precisa aparecer na lista do segundo. Se não aparecer, você está na máquina errada ou o
DNS ainda não propagou.

### 1.2 Docker (se ainda não tiver)
```bash
curl -fsSL https://get.docker.com | sh
```

### 1.3 Copiar o pacote (no Windows, PowerShell)
```powershell
scp optec-clinica-v0.3.0.tar.gz root@45.224.41.120:/opt/
```

### 1.4 Extrair e subir a aplicação (no servidor)
```bash
cd /opt && tar xzf optec-clinica-v0.3.0.tar.gz && cd optec-clinica
```
```bash
sh scripts/install.sh
```
O script gera o `.env` (senha do banco e chave de cifragem aleatórias), compila a imagem e sobe tudo.
Se já existir um banco de uma instalação anterior e o `.env` tiver sumido, ele **para e avisa** em vez de criar um
`.env` novo (isso deixaria o banco inacessível).

**Agora copie o `/opt/optec-clinica/.env` para fora do servidor** (por exemplo, no Windows:
`scp root@45.224.41.120:/opt/optec-clinica/.env .`).

### 1.5 Publicar com HTTPS (obrigatório: o login só funciona em HTTPS)
```bash
cd /opt/optec-clinica && sh scripts/setup-nginx.sh clinica.optec.eng.br optec.fibra@gmail.com
```
O script corrige o PATH, instala Nginx e certbot se faltarem, confere a aplicação e o DNS, grava a configuração,
ativa o site (sem o erro "Arquivo existe"), detecta Nginx que só carrega `conf.d` e nomes duplicados, recarrega o
Nginx, gera o certificado e força HTTPS. Pode ser rodado de novo quantas vezes precisar.
No final deve aparecer `HTTP/1.1 200 OK` (ou `HTTP/2 200`).

### 1.6 Criar o administrador
```bash
cd /opt/optec-clinica && docker compose exec app python -m app.cli create-admin --email seu@email.com.br --name "Seu Nome"
```
Aparece uma **senha provisória**. Acesse **https://clinica.optec.eng.br**: o sistema pede para trocar a senha e
ativar a verificação em duas etapas (Google Authenticator, Microsoft Authenticator ou similar).

### 1.7 Agendar o backup diário
```bash
(crontab -l 2>/dev/null; echo "0 2 * * * cd /opt/optec-clinica && sh scripts/backup.sh >/dev/null 2>&1") | crontab -
```

### 1.8 (Opcional, só para apresentação) Dados fictícios
```bash
docker compose exec app python -m app.cli seed-demo
```
Cria profissionais `@clinica-demo.com.br`, pacientes fictícios, reservas, consultas, despesas e prontuários de
exemplo. **Não rode em produção.**

### 1.9 Conferir tudo de uma vez
```bash
cd /opt/optec-clinica && sh scripts/diagnostico.sh
```
Mostra `.env`, containers, aplicação, Nginx, DNS, HTTPS, certificado e último backup, com `[OK]` ou `[ERRO]` e
o que fazer. **Se algo der errado, mande a saída deste comando para o suporte.**

---

## 2. Atualizar para uma nova versão

No Windows (PowerShell):
```powershell
scp optec-clinica-vX.Y.Z.tar.gz root@45.224.41.120:/opt/
```
No servidor (extrair **por cima**, sem apagar a pasta):
```bash
cd /opt && tar xzf optec-clinica-vX.Y.Z.tar.gz
```
```bash
cd /opt/optec-clinica && sh scripts/update.sh
```
O `update.sh` confere o `.env`, faz backup, recompila, sobe e espera a aplicação responder.
As migrações do banco rodam sozinhas. **Não é preciso refazer Nginx/certbot** numa atualização.

> Da v0.1 para a v0.2 os registros de prontuário já existentes passaram a ser **privados** (como eram na v0.1).
> Para compartilhar com a equipe, marque no paciente o consentimento "prontuário compartilhado".

> Da v0.2 para a v0.3 a migração converte o financeiro antigo: consultas e pacotes marcados como "pago" viram
> lançamentos de pagamento (com a data e a forma registradas) e despesas pagas ganham data e valor pago.
> Nada é apagado. Depois de atualizar, confira em **Configurações → Financeiro** as taxas da maquininha, multa/juros
> e quem fica com o dinheiro quando a recepção recebe.

---

## 3. Problemas comuns

| Sintoma | Causa | Solução |
|---|---|---|
| `nginx: comando não encontrado` ou `certbot: comando não encontrado` | Root aberto com `su` sem hífen | `export PATH=$PATH:/usr/sbin:/sbin` ou entrar com `su -` |
| `ln: ... Arquivo existe` | Link do site já criado antes | Inofensivo. Use `sh scripts/setup-nginx.sh` (usa `ln -sfn`) |
| `HTTP/1.1 404 Not Found` em `http://clinica.../api/health` | O Nginx não usa o arquivo do sistema (link antigo, `nginx.conf` sem `sites-enabled` ou nome duplicado) | `sh scripts/setup-nginx.sh clinica.optec.eng.br seu@email` — detecta e corrige |
| `setup-nginx.sh`: "A aplicação não respondeu" / `curl: (56)` | Container `app` parado ou reiniciando | `sh scripts/diagnostico.sh` e veja a linha de erro do log |
| App em `Restarting` com `password authentication failed for user "clinica"` | O `.env` foi recriado (pasta apagada/substituída) e o banco ficou com a senha antiga | **Melhor:** restaurar o `.env` antigo e `docker compose restart app`. **Sem o `.env` antigo:** ver 3.1 |
| Certbot: `Timeout` ou `Connection refused` | Portas 80/443 fechadas no firewall ou roteador | Liberar 80 e 443 TCP para o servidor (`ufw allow 80,443/tcp` se usar ufw) |
| Certbot: `NXDOMAIN` ou IP errado | DNS não aponta para o servidor | Corrigir o registro A e esperar propagar |
| Página abre, mas o login volta para a tela inicial | Acesso por `http://` (o cookie de sessão exige HTTPS) | Usar `https://`; rodar o passo 1.5 |
| `502 Bad Gateway` | Aplicação parada | `docker compose ps` e `docker compose logs --tail 50 app` |
| Usuário perdeu o celular do 2FA | — | Administrador: *Equipe → membro → Remover 2FA*, ou `docker compose exec app python -m app.cli reset-mfa --email x@y` |
| Usuário esqueceu a senha | — | Administrador: *Equipe → membro → Gerar nova senha* |
| Usuário bloqueado por senha errada | 5 tentativas (configurável) | Esperar 15 min ou *Gerar nova senha* (também desbloqueia) |

### 3.1 Banco com senha diferente do `.env` (sem o `.env` antigo)
Faz o banco aceitar a senha do `.env` atual, **sem perder dados**:
```bash
cd /opt/optec-clinica && DBP=$(grep '^DB_PASSWORD=' .env | cut -d= -f2) && docker compose exec -T db psql -U clinica -d clinica -c "ALTER USER clinica PASSWORD '$DBP';"
```
```bash
docker compose restart app && sleep 15 && curl -s http://127.0.0.1:8081/api/health
```
Atenção: com um `.env` novo, a **chave de cifragem também mudou**. Prontuários, CPFs, endereços e segredos de 2FA
gravados com a chave antiga ficam ilegíveis ("[não foi possível decifrar]"). Os usuários precisarão configurar o
2FA de novo (`reset-mfa`). Por isso a cópia do `.env` fora do servidor é essencial.

---

## 4. Backup e restauração

Backup manual (banco + fotos, logo e anexos; mantém 30 dias em `backups/`):
```bash
cd /opt/optec-clinica && sh scripts/backup.sh
```
Leve os arquivos de `backups/` **e o `.env`** para fora do servidor com frequência. O backup sem o `.env` não
serve para ler os dados cifrados.

Restaurar o banco:
```bash
cd /opt/optec-clinica && docker compose exec -T db pg_restore -U clinica -d clinica --clean --if-exists < backups/db-AAAAMMDD-HHMMSS.dump
```
Restaurar fotos, logo e anexos:
```bash
cd /opt/optec-clinica && docker compose exec -T app tar -C /data -xzf - < backups/uploads-AAAAMMDD-HHMMSS.tar.gz
```

---

## 5. Comandos úteis

| Situação | Comando |
|---|---|
| Diagnóstico completo | `sh scripts/diagnostico.sh` |
| Ver se está no ar | `docker compose ps` e `curl -s http://127.0.0.1:8081/api/health` |
| Logs da aplicação | `docker compose logs -f app` |
| Reiniciar a aplicação | `docker compose restart app` |
| Testar o Nginx | `/usr/sbin/nginx -t && systemctl reload nginx` |
| Renovação do certificado | automática (timer do certbot); teste com `certbot renew --dry-run` |
| Rodar os testes automatizados | só em laboratório, com um banco **separado** terminado em `_test` (`TEST_DATABASE_URL`). Os testes apagam esse banco e se recusam a rodar em qualquer outro |

---

## 6. Segurança implementada

- **Senhas** com Argon2id; mínimo configurável (padrão 10 caracteres, letras e números); senha provisória
  obrigatoriamente trocada no primeiro acesso.
- **2FA (TOTP)** obrigatório por padrão para todos os perfis.
- **Bloqueio** após tentativas erradas (padrão 5, por 15 minutos); mesma mensagem para e-mail inexistente.
- **Sessão** em cookie `HttpOnly`, `Secure`, `SameSite=Strict`, guardada no banco (pode ser encerrada); expira por
  inatividade (padrão 30 min) e no máximo em 12 h. Trocar a senha encerra as outras sessões.
- **CSRF**: toda alteração exige o cabeçalho `X-Requested-With` além do cookie `SameSite=Strict`.
- **Cifragem em repouso** (Fernet/AES) de prontuário, alertas clínicos, CPF, RG, endereço, nomes de anexos e
  segredos de 2FA; **anexos cifrados no disco**.
- **Acesso por equipe de cuidado**: cada profissional vê só os pacientes que atende. Registros "equipe" aparecem
  para a equipe **só com o consentimento** de prontuário compartilhado; registros "privados" ficam só com o autor.
  Administração e recepção **não leem prontuário**.
- **Prontuário imutável**: registros não são editados nem apagados; correções viram adendos com data.
- **Auditoria**: login, falhas, abertura de paciente, prontuário e anexos, exportações, PDFs e alterações, com IP.
- **Uploads**: fotos e logo só imagens até 5 MB, regravadas em WebP (sem EXIF); anexos PDF/JPG/PNG/WebP até
  15 MB verificados pelo conteúdo.
- **Link de confirmação** (`/c/…`): token aleatório de 24 bytes, válido até o dia da consulta, mostra só primeiro
  nome, data, horário, profissional e endereço (nada clínico).
- **Cabeçalhos**: CSP restritiva, `X-Frame-Options: DENY`, `nosniff`, HSTS (Nginx), `noindex`.
- Banco e aplicação **sem portas públicas**; só o Nginx fala com a aplicação.

---

## 7. LGPD — o que o sistema cobre e o que depende da clínica

O sistema ajuda com: consentimentos registrados com data (termo, prontuário eletrônico, compartilhamento na equipe
e WhatsApp), exportação dos dados do paciente (*Paciente → Exportar* e *PDF*; portabilidade e acesso — art. 18),
trilha de auditoria (art. 37), cifragem e controle de acesso (art. 46) e prontuário que não pode ser apagado
(guarda exigida pelos conselhos).

Fica com a clínica: nomear o encarregado (DPO), política de privacidade e termos assinados, contrato com os
profissionais definindo controlador/operador, plano de resposta a incidentes (comunicação à ANPD em 3 dias úteis —
Resolução CD/ANPD 15/2024), backups e cópia do `.env` fora do servidor e validação jurídica dos prazos de guarda de
cada conselho (CFP, COFFITO, CFN, CFFa, CFM).
