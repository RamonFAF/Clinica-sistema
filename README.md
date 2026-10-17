# NeuroInsight · Sistema da clínica

Gestão de clínica multiprofissional: salas, reservas, equipe com foto, pacientes, prontuário multiprofissional
com modelos estruturados e anexos cifrados, agenda com link de confirmação, financeiro da clínica (despesas, faturas,
resultado) e de cada profissional (pacotes, recibos), comunicação por WhatsApp sem API, lista de espera, 2FA e auditoria.

- **Histórico de versões:** [CHANGELOG.md](CHANGELOG.md)

- **Instalação e operação:** [INSTALL.md](INSTALL.md)
- **Stack:** FastAPI + SQLAlchemy + Alembic · PostgreSQL 16 · React 18 + Vite + TypeScript + Tailwind + TanStack Query · Docker Compose
- **Telas:** `docs/telas/`

## Desenvolvimento

```bash
# back-end (precisa de um PostgreSQL local)
cd backend && pip install -r requirements.txt
export DATABASE_URL=postgresql+psycopg://clinica:clinica@localhost:5432/clinica
export ENCRYPTION_KEY=$(python -c "from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())")
export COOKIE_SECURE=false UPLOAD_DIR=/tmp/uploads
alembic upgrade head && uvicorn app.main:app --reload

# front-end (proxy /api -> 127.0.0.1:8000)
cd frontend && npm install && npm run dev

# testes: usam SOMENTE um banco terminado em _test (padrão clinica_test local) e o apagam
cd backend && TEST_DATABASE_URL=postgresql+psycopg://clinica:clinica@localhost:5432/clinica_test pytest -q
```
