# NeuroInsight · Sistema da clínica — instalação e operação

Versão: veja o arquivo `VERSION`. Servidor alvo: **Debian 12 com Docker**, publicado em
**https://clinica.optec.eng.br** pelo Nginx que já existe no servidor.

```
Internet ──HTTPS──> Nginx do servidor (443, certificado Let's Encrypt)
                        └──> 127.0.0.1:8081 ──> container app (FastAPI + React)
                                                   └──> container db (PostgreSQL 16, sem porta exposta)
```

## 1. Primeira instalação

### 1.1 DNS
Crie um registro **A** `clinica.optec.eng.br` apontando para o IP do servidor (hoje `45.224.43.72`).
Confira com `getent hosts clinica.optec.eng.br`.

### 1.2 Docker (se ainda não tiver)
```bash
curl -fsSL https://get.docker.com | sh
```

### 1.3 Copiar o pacote (no Windows, PowerShell)
```powershell
scp optec-clinica-v0.2.0.tar.gz root@45.224.43.72:/opt/
```

### 1.4 Extrair e instalar (no servidor)
```bash
cd /opt && tar xzf optec-clinica-v0.2.0.tar.gz && cd optec-clinica
```
```bash
sudo sh scripts/install.sh
```
O script gera o `.env` com senha do banco e **chave de cifragem** aleatórias, compila a imagem e sobe tudo.

> **Copie o arquivo `/opt/optec-clinica/.env` para um local seguro fora do servidor.**
> Sem a `ENCRYPTION_KEY`, prontuários, CPFs e segredos de 2FA ficam ilegíveis, inclusive nos backups.

### 1.5 Nginx + HTTPS
```bash
cp deploy/nginx-clinica.optec.eng.br.conf /etc/nginx/sites-available/clinica.optec.eng.br
ln -s /etc/nginx/sites-available/clinica.optec.eng.br /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```
```bash
certbot --nginx -d clinica.optec.eng.br
```
(Responda "redirect" para forçar HTTPS.) O cookie de sessão só funciona em HTTPS.

### 1.6 Criar o administrador
```bash
docker compose exec app python -m app.cli create-admin --email seu@email.com.br --name "Seu Nome"
```
Aparece uma senha provisória. No primeiro acesso o sistema obriga a trocar a senha e ativar o 2FA
(Google Authenticator, Microsoft Authenticator etc.).

### 1.7 (Opcional) Dados fictícios para apresentação
```bash
docker compose exec app python -m app.cli seed-demo
```
Cria 4 profissionais `@clinica-demo.com.br`, 12 pacientes fictícios, reservas e consultas.
Para uma instalação de produção limpa, **não rode** este comando.

## 2. Uso diário (resumo)

| Perfil | O que vê |
|---|---|
| **Administrador** | Tudo da gestão: salas, equipe, financeiro da clínica, configurações e auditoria. **Não lê prontuário** de outros profissionais. |
| **Recepção** | Agenda de todos, cadastro de pacientes e reservas. Sem prontuário, sem financeiro. |
| **Profissional** | Apenas os **próprios** pacientes, prontuários, agenda e financeiro. Reserva salas se a configuração permitir. |

- **Equipe → Adicionar**: cria o acesso e mostra uma senha provisória (só aparece uma vez). A foto pode ser enviada
  pelo administrador ou pelo próprio profissional em *Meu perfil*.
- **Configurações**: quantidade de salas, nomes, valores por hora, horário de funcionamento, dias da semana,
  duração e valor padrão da consulta, regras de reserva, mensagem do WhatsApp, logo e regras de segurança.
- **Celular**: abra o endereço no navegador e use "Adicionar à tela inicial" para ficar como aplicativo.

## 3. Atualizar para uma nova versão

> Da v0.1 para a v0.2 o banco recebe a migração `0002` automaticamente quando o container sobe.
> Registros de prontuário já existentes passam a ser **privados** (como eram na v0.1). Para compartilhar com a
> equipe, marque no paciente o consentimento "prontuário compartilhado" e registre as novas evoluções como "equipe".


```powershell
scp optec-clinica-vX.Y.Z.tar.gz root@45.224.43.72:/opt/
```
```bash
cd /opt/optec-clinica && sh scripts/backup.sh
```
```bash
cd /opt && tar xzf optec-clinica-vX.Y.Z.tar.gz
```
```bash
cd /opt/optec-clinica && docker compose up -d --build
```
O `.env` e os dados (volumes `db-data` e `uploads`) não são tocados pelo pacote. As migrações do banco rodam
sozinhas quando o container sobe.

## 4. Backup e restauração

Backup manual (banco + fotos/logo, guarda 30 dias em `backups/`):
```bash
cd /opt/optec-clinica && sh scripts/backup.sh
```
Backup diário às 2h (cron do root):
```bash
(crontab -l 2>/dev/null; echo "0 2 * * * cd /opt/optec-clinica && sh scripts/backup.sh >/dev/null 2>&1") | crontab -
```
Leve os arquivos de `backups/` para fora do servidor com frequência (Google Drive, outro servidor).

Restaurar um backup do banco:
```bash
cd /opt/optec-clinica && docker compose exec -T db pg_restore -U clinica -d clinica --clean --if-exists < backups/db-AAAAMMDD-HHMMSS.dump
```
Restaurar fotos/logo:
```bash
cd /opt/optec-clinica && docker compose exec -T app tar -C /data -xzf - < backups/uploads-AAAAMMDD-HHMMSS.tar.gz
```

## 5. Comandos úteis

| Situação | Comando |
|---|---|
| Ver se está no ar | `docker compose ps` e `curl -s http://127.0.0.1:8081/api/health` |
| Logs da aplicação | `docker compose logs -f app` |
| Usuário perdeu o celular do 2FA | Administrador: *Equipe → membro → Remover 2FA*, ou `docker compose exec app python -m app.cli reset-mfa --email x@y` |
| Usuário esqueceu a senha | Administrador: *Equipe → membro → Gerar nova senha* |
| Reiniciar | `docker compose restart app` |
| Rodar os testes (só em laboratório) | precisa de um banco **separado** terminado em `_test` (`TEST_DATABASE_URL`); os testes apagam esse banco |

## 6. Segurança implementada

- **Senhas** com Argon2id; mínimo configurável (padrão 10 caracteres, letras e números); senha provisória obrigatoriamente trocada.
- **2FA (TOTP)** obrigatório por padrão para todos os perfis.
- **Bloqueio** após tentativas erradas (padrão 5 tentativas, 15 minutos); mensagem igual para e-mail inexistente.
- **Sessão** em cookie `HttpOnly`, `Secure`, `SameSite=Strict`, guardada no banco (pode ser encerrada); expira por
  inatividade (padrão 30 min) e no máximo em 12 h. Trocar a senha encerra as outras sessões.
- **CSRF**: toda alteração exige o cabeçalho `X-Requested-With` além do cookie `SameSite=Strict`.
- **Cifragem em repouso** (Fernet/AES) de prontuário, observações do paciente, CPF e segredos de 2FA.
- **Isolamento**: profissional só acessa os próprios pacientes; prontuário só para o profissional responsável.
- **Prontuário imutável**: registros não são editados nem apagados; correções viram adendos com data.
- **Auditoria**: login, falhas, abertura de paciente e de prontuário, exportações e alterações ficam registrados com IP.
- **Uploads**: fotos e logo só imagens até 5 MB, regravadas em WebP (sem EXIF); anexos do prontuário (PDF/JPG/PNG/WebP
  até 15 MB) verificados pelo conteúdo e **cifrados no disco**; tudo só com login.
- **Link de confirmação** (`/c/…`): token aleatório de 24 bytes, válido até o dia da consulta, mostra só primeiro
  nome, data, horário, profissional e endereço (nada clínico).
- **Prontuário compartilhado** só com consentimento do paciente; registros privados ficam só com o autor.
- **Cabeçalhos**: CSP restritiva, `X-Frame-Options: DENY`, `nosniff`, HSTS (Nginx), `noindex`.
- Banco e aplicação **sem portas públicas**; só o Nginx fala com a aplicação.

## 7. LGPD — o que o sistema cobre e o que depende da clínica

O sistema ajuda com: consentimento registrado (atendimento e WhatsApp, com data), exportação dos dados do paciente
(*Paciente → Exportar*, portabilidade — art. 18), trilha de auditoria (art. 37), cifragem e controle de acesso (art. 46),
prontuário que não pode ser apagado (guarda exigida pelos conselhos).

Fica com a clínica: nomear o encarregado (DPO), política de privacidade e termos assinados, contrato com os
profissionais definindo controlador/operador, plano de resposta a incidentes (comunicação à ANPD em 3 dias úteis —
Resolução CD/ANPD 15/2024), backups fora do servidor e validação jurídica dos prazos de guarda de cada conselho
(CFP, COFFITO, CFN, CFFa, CFM).
