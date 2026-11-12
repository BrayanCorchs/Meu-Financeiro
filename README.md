# Meu-Financeiro[Uploading README.md…]()
# Meu Financeiro — versão com back-end em Python

Controle financeiro pessoal: entradas, saídas, contas fixas e parceladas, bancos (saldo e limite) e meta para guardar dinheiro.

- **frontend/** — o app (HTML, CSS e JavaScript puro). É o que roda no celular.
- **backend/** — a API em Python (Flask) com banco de dados SQLite, login e senha.

## Como funciona

O app continua fazendo as contas na tela. O back-end guarda os dados de cada usuário, para que apareçam em qualquer aparelho depois do login.

| Parte | Tecnologia |
|---|---|
| Telas | HTML + CSS + JavaScript |
| API | Python 3 + Flask |
| Banco de dados | SQLite (4 tabelas: usuarios, sessoes, bancos, lancamentos) |
| Login | senha com PBKDF2-SHA256 e token de sessão |
| Cálculos CLT | `backend/clt.py`: INSS 2026, IRRF com a redução da Lei 15.270/2025, 13º e férias |

## Rodar no computador

```bash
cd backend
pip install -r requirements.txt
python teste_api.py     # roda os testes
python app.py           # sobe a API em http://localhost:5000
```

Depois abra `frontend/index.html` em um editor e, perto do início do `<script>`, troque:

```js
var API_URL="";
```

por:

```js
var API_URL="http://localhost:5000";
```

Abra o `frontend/index.html` no navegador. Vai aparecer a tela de login.

## Rotas da API

Todas as rotas, menos `/`, `/cadastro` e `/login`, pedem o cabeçalho `Authorization: Bearer <token>`.
Valores em dinheiro são sempre em **centavos** (R$ 150,00 = `15000`).

| Método | Rota | O que faz |
|---|---|---|
| GET | `/` | Diz se a API está no ar |
| POST | `/cadastro` | Cria a conta `{email, senha}` e devolve o token |
| POST | `/login` | Entra e devolve o token |
| POST | `/sair` | Encerra a sessão |
| GET | `/eu` | Dados do usuário logado |
| GET / PUT | `/sincronizar` | Baixa ou envia tudo de uma vez (é o que o app usa) |
| GET | `/lancamentos` | Lista todos; com `?mes=2026-11` lista só os do mês, com o valor da parcela |
| POST | `/lancamentos` | Cria um lançamento |
| PUT / DELETE | `/lancamentos/<id>` | Altera ou apaga |
| GET / POST | `/bancos` | Lista ou cria banco |
| PUT / DELETE | `/bancos/<id>` | Altera ou apaga |
| PUT | `/meta` | Define quanto guardar por mês `{meta}` |
| GET | `/resumo?mes=2026-11` | Entrou, saiu, guardado, sobra, livre para gastar, falta pagar, ranking por categoria |
| GET | `/clt` | 13º e férias líquidos (INSS e IR de 2026). Aceita na URL: `bruto`, `extras`, `dependentes`, `meses`, `dias`, `vender` |

Exemplo de lançamento:

```json
{"tipo": "saida", "desc": "Celular", "valor": 240000, "data": "2026-11-10",
 "cat": "Compras", "forma": "Crédito", "parcelas": 10}
```

## Publicar

**Back-end** (exemplo com Render, serviço do tipo Web Service ligado ao repositório do GitHub):

- Root Directory: `backend`
- Build Command: `pip install -r requirements.txt`
- Start Command: `gunicorn app:app`
- Variável de ambiente `ORIGENS_PERMITIDAS`: o endereço do seu app, por exemplo `https://meu-financeiro.vercel.app`

**Front-end** (Vercel): coloque em `API_URL` o endereço que o serviço do back-end te deu e publique a pasta `frontend` (Root Directory: `frontend`).

### Atenção ao banco de dados

O SQLite é um arquivo (`financeiro.db`). Em hospedagens gratuitas sem disco persistente, esse arquivo é apagado quando o servidor reinicia, e os dados somem. Para uso de verdade, use uma destas opções:

- um disco persistente no serviço, apontando a variável `BANCO_DE_DADOS` para ele (ex.: `/var/data/financeiro.db`);
- uma hospedagem que mantém os arquivos, como a PythonAnywhere;
- trocar o SQLite por PostgreSQL.

## 13º e férias

O cálculo segue a CLT com as tabelas de 2026, igual no app e no `clt.py`:

- **INSS** progressivo: 7,5% até R$ 1.621,00; 9% até R$ 2.902,84; 12% até R$ 4.354,27; 14% até o teto de R$ 8.475,55.
- **IRRF**: base = valor bruto − INSS − R$ 189,59 por dependente (ou desconto simplificado de R$ 607,20, se for melhor). Depois da tabela, entra a redução da Lei 15.270/2025: imposto zerado até R$ 5.000 e redução de 978,62 − 0,133145 × rendimento até R$ 7.350.
- **13º**: 1ª parcela é metade do bruto, sem descontos (até 30/11). INSS e IR saem da 2ª parcela (até 20/12), calculados separados do salário.
- **Férias**: INSS e IR sobre férias + 1/3. A venda de até 1/3 dos dias (abono) não tem INSS nem IR.

Quando as tabelas mudarem (normalmente em janeiro), atualize as faixas no topo do `clt.py` e no objeto `CLT` do `frontend/index.html`.

## Limites desta versão

- Não tem "esqueci minha senha".
- Não tem limite de tentativas de login.
- O app envia todos os dados a cada alteração (`PUT /sincronizar`). Funciona bem para uso pessoal; para muitos usuários, o ideal é o app chamar as rotas de cada lançamento.
