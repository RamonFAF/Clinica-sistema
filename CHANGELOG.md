# Histórico de versões

## v0.3.0 — financeiro completo

**Paciente: consulta individual ou pacote**
- Aba **Financeiro** na ficha do paciente (substitui "Pacotes"): plano (consulta individual × pacote), valor da sessão
  próprio do paciente (preço social/combinado), em aberto, crédito, pacotes, movimentos com recibo e estorno.
- **Pacote livre**: o profissional escolhe a quantidade de sessões e o valor; mostra valor por sessão e desconto em
  relação à avulsa, parcelas combinadas, entrada no ato, validade, "falta consome sessão ou não", observações.
- Pacote: barra de uso, realizadas × restantes, pago × saldo, alerta de renovação e de vencimento, **encerrar** com
  devolução sugerida (pago − sessões realizadas), recibo com todos os pagamentos.
- **Pagamentos parciais** em consultas e pacotes; pagamento a mais vira **crédito do paciente**; adiantamento;
  **usar crédito** para quitar consultas; estorno.
- **Abonar falta** (zera o valor da consulta), consulta sem cobrança (cortesia).
- Taxa da maquininha por forma de pagamento: o profissional vê bruto, taxas e líquido.
- Quem recebeu: **profissional** ou **clínica/recepção** (padrão configurável).

**Profissional**
- Financeiro com Resumo (recebido líquido, produzido, a receber, pacotes a realizar, aluguel, resultado), acerto com a
  clínica, pacotes para renovar, Recebimentos, Atendimentos (receber saldo), Pacotes, A receber, Minhas contas e
  Outras receitas.

**Clínica**
- **Contas a pagar**: categoria, fornecedor, nº do documento, vencimento, repetição mensal (até 36 meses), painel de
  vencidas / hoje / 7 dias / 30 dias / pagas no mês, filtros e busca, lista agrupada por vencimento,
  **informar pagamento** (data, valor pago com juros ou desconto, forma) e **anexar boleto, nota e comprovante**
  (guardados criptografados).
- **Receitas avulsas** (sublocação, cursos, parcerias) com o mesmo controle.
- **Aluguéis**: pagamento parcial, saldo, situação parcial/atrasada, multa e juros sugeridos com um clique,
  modelo **fixo + %** além de por hora, mensal e percentual.
- **Repasses**: dinheiro de pacientes recebido pela recepção abate o aluguel (compensação) e o restante é repassado
  ao profissional, com histórico.
- Visão geral com **competência × caixa** e **fluxo de caixa dos próximos 30 dias** (a receber × a pagar).
- Configurações → Financeiro: taxas de cartão/Pix, multa e juros, alerta de renovação, categorias de despesas e
  receitas.
- CSV da clínica e do profissional com os novos campos.

## v0.2.3 — manual e instalação
- **Manual do usuário em PDF** (`docs/Manual-do-Usuario-NeuroInsight.pdf`): administrador, recepção, profissional,
  o que o paciente vê, boas práticas de LGPD e perguntas frequentes, com telas reais.
- Consulta presencial sem sala: a janela mostra as **salas livres no horário** e um botão “Reservar e salvar”
  (reserva a sala com a mesma repetição semanal e grava a consulta).
- `scripts/diagnostico.sh`: confere `.env`, containers, aplicação, Nginx, DNS, HTTPS, certificado e backup.
- `scripts/update.sh`: recusa rodar sem `.env` e espera a aplicação responder.
- INSTALL.md reescrito na ordem que funciona no servidor, com tabela de problemas (PATH do `su`, “Arquivo existe”,
  404 do Nginx, senha do banco divergente, certificado, login em http).
- Lembretes: “Confirmada” (pela clínica) separado de “Confirmou pelo link” (pelo paciente); salas sem recursos
  não mostram mais “—”.

## v0.2.2 — instalação
- `install.sh` não cria `.env` novo se já existir banco de instalação anterior (evita senha do banco divergente e
  perda da chave de cifragem). INSTALL: como corrigir `password authentication failed`, nunca apagar a pasta.

## v0.2.1 — instalação
- Novo `scripts/setup-nginx.sh`: publica com HTTPS em um comando (corrige PATH do `su` sem hífen, `ln -sfn` sem erro de
  "Arquivo existe", confere aplicação e DNS, preserva o arquivo depois do certbot, instala certbot se faltar).
- INSTALL.md: IP atual do servidor, como entrar como root (`su -`), tabela de erros comuns do Nginx/certbot.

## v0.2.0 — fase 2

**Prontuário multiprofissional (modelo NeuroInsight)**
- Prontuário único com **equipe de cuidado** (responsável + profissionais que atendem). Agendar um paciente com outro
  profissional o inclui automaticamente na equipe.
- Visibilidade por registro: **equipe** (exige o consentimento "prontuário compartilhado") ou **privado** (só o autor).
  Registros da v0.1 viraram privados na migração.
- **Modelos estruturados**: anamnese geral, avaliação psicológica (PHQ-9, GAD-7, risco), avaliação nutricional,
  terapias corporais (EVA, contraindicações), avaliação fisioterapêutica (COFFITO 414/2012), PTI por área,
  evolução SOAP, sessão de psicoterapia (nasce privada — CFP 01/2009), evolução de fisioterapia, alta/encaminhamento.
- Alertas automáticos (ideação/plano suicida, contraindicações).
- **Anexos** (PDF/imagens até 15 MB) cifrados no disco.
- **PDF do prontuário** no layout da clínica; identificação completa do paciente (sexo, estado civil, RG, profissão,
  endereço, convênio, responsável) e os 4 consentimentos do termo.

**Financeiro**
- Clínica: visão geral (faturado, recebido, despesas, resultado, 6 meses), despesas por categoria e recorrentes,
  vencimento e atraso de faturas, descontos/acréscimos, detalhe das reservas, recibo de locação, exportação CSV.
- Profissional: pacotes de sessões, despesas próprias, a receber por paciente, recebido por forma de pagamento,
  gráfico de 6 meses, recibo por consulta/pacote/mês, exportação CSV.

**Comunicação sem API de WhatsApp**
- Link de confirmação por consulta (`/c/…`): o paciente confirma ou pede remarcação sem login; a agenda atualiza.
- Tela Comunicação: lembretes do dia seguinte com envio em um toque, pedidos de remarcação, aniversariantes e cobranças.
- Mensagens editáveis em Configurações (lembrete, cobrança, aniversário).

**Outros**
- Lista de espera, declaração de comparecimento em PDF, CPF/CNPJ do profissional nos recibos, logo oficial.
- Testes recusam qualquer banco que não termine em `_test` (proteção contra apagar produção).

## v0.1.0 — fase 1
Salas, reservas, equipe com foto, pacientes, prontuário cifrado, agenda, financeiro básico, 2FA, auditoria.
  
