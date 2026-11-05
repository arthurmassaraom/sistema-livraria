# Biblioteca

Aplicação web multipágina para um acervo comunitário: catálogo com capas, compra e aluguel de livros, carrinho, checkout Pix/cartão em demonstração ou via gateway real, retirada/entrega, avaliações verificadas, busca regional por CEP e gestão administrativa. Construída com Flask, Jinja2, SQLAlchemy e MySQL ou SQLite.

> O projeto está pronto para instalação e customização; cobrança real só fica habilitada depois de configurar suas próprias credenciais do Mercado Pago, webhook HTTPS e dados legais da entidade. Revise as políticas de operação e privacidade antes de receber clientes.

## Requisitos

- Python 3.10 ou superior.
- MySQL 8+ para produção; SQLite local é usado automaticamente se `DATABASE_URL`/`MYSQL_DATABASE` não forem configurados.
- Conta Mercado Pago Developers para pagamentos reais ou credenciais de teste para homologação.

## Instalação local no Windows

Extraia o ZIP, abra PowerShell na pasta `sistema_biblioteca` e rode:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
notepad .env
```

Para usar MySQL, preencha em `.env` `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_USER`, `MYSQL_PASSWORD` e `MYSQL_DATABASE=biblioteca_db`. Importe o esquema pelo MySQL Workbench ou pelo Prompt de Comando:

```cmd
mysql -u root -p < database/schema_mysql.sql
```

O script cria as tabelas sem conta administrativa, credenciais padrão ou unidades parceiras fictícias. Na primeira inicialização, `seed.py` adiciona planos informativos e 16 registros de catálogo com capas locais. Revise disponibilidade, preço e estoque para refletir seu acervo real antes de habilitar cobranças. Após criar o administrador, cadastre somente pontos de retirada autorizados em **Unidades parceiras**; até lá, a retirada fica desabilitada e o checkout aceita entrega em domicílio.

Inicie em desenvolvimento:

```powershell
py app.py
```

Acesse `http://127.0.0.1:5000`.

## Linux/macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Edite .env, importe o database/schema_mysql.sql se estiver usando MySQL.
python app.py
```

Para servir via WSGI (inclusive no Windows), depois de configurar HTTPS/reverse proxy, use `waitress-serve --listen=127.0.0.1:8000 app:app`. O servidor Flask embutido é apenas para desenvolvimento.

Sem MySQL configurado, a aplicação cria `instance/biblioteca.db`; colunas novas ausentes de bancos SQLite legados são adicionadas automaticamente. Uma migração única desativa a antiga conta administrativa de demonstração com senha conhecida; ela não é repetida nas inicializações seguintes. A chave de sessão é lida de `SECRET_KEY` ou gerada e persistida localmente em `instance/.flask_secret_key`. Em produção, defina uma chave forte e privada no ambiente; nunca a publique no GitHub.

## Criar a conta administradora

Não há administrador, e-mail ou senha padrão no código ou no schema. Com o ambiente ativo, crie a conta pelo comando interativo:

```bash
python -m flask --app app create-admin
```

O comando pede nome, e-mail e senha, grava somente o hash da senha e recusa e-mails já cadastrados. O sistema também reconhece contas administrativas legadas com papel `administrador` ou `bibliotecário` (sem diferenciar maiúsculas/minúsculas e acentos). Uma conta com papel `cliente` não ganha privilégios automaticamente. A área administrativa tem navegação horizontal própria e nela você gerencia catálogo, unidades parceiras, pedidos e curadoria. Se precisar reaproveitar uma conta existente (inclusive a conta demo antiga, que foi rebaixada por segurança), execute `python -m flask --app app promote-admin --email seu@email.com`. O comando exige uma senha administrativa nova, digitada de forma oculta; não reative a senha demo conhecida.

## Pagamentos: demonstração e modo real

Por padrão, `PAYMENT_MODE=demo`. Esse modo é uma demonstração claramente identificada na interface: gera um QR e um código aleatórios com prefixo não pagável; o Pix de demonstração é aceito automaticamente após cinco segundos e o cartão de demonstração é aceito ao clicar em “Finalizar pagamento”. Em ambos os casos o cliente é levado ao seu perfil, onde o pedido aparece como aquisição, e o estoque é debitado como em um pedido real. **Não há cobrança real e não é Pix válido.** Os números digitados nos campos de cartão de demonstração não são enviados nem armazenados. Não use dados reais nesses campos.

Para ativar o Mercado Pago real, configure `PAYMENT_MODE=mercadopago`, `MP_PUBLIC_KEY` e `MP_ACCESS_TOKEN` no `.env`. O modo real só é habilitado quando essa opção e as credenciais necessárias estão presentes. Em homologação, use as credenciais e cartões de teste da sua conta. Antes de produção:

1. Configure credenciais corretas e mantenha `MP_ACCESS_TOKEN` somente no servidor.
2. Configure um endpoint HTTPS público para `https://seu-dominio/webhooks/mercadopago` e informe o segredo de assinatura da integração em `MP_WEBHOOK_SECRET`.
3. Configure `MP_NOTIFICATION_URL` com a URL HTTPS completa do webhook e habilite notificações de pagamento no painel do Mercado Pago.
4. Configure `SESSION_COOKIE_SECURE=true` sob HTTPS e use um proxy/servidor WSGI de produção, nunca o servidor de desenvolvimento do Flask.
5. Teste pagamentos aprovados, pendentes, recusados, cancelados, notificações repetidas, estornos e perda temporária de rede com as credenciais de teste.

No modo real, QR Code e código Pix são emitidos pela API do Mercado Pago. Para cartão, o Card Payment Brick envia um token; a aplicação não salva número, validade ou CVV. Estoque é reservado de forma atômica e a confirmação depende da API/webhook. Não apresente a demonstração como se fosse uma cobrança real.

Frete domiciliar é atualmente fixo em R$ 12,90 e retirada em biblioteca é gratuita. Ajuste essa regra em `routes/carrinho_routes.py` e `routes/pagamentos_routes.py` conforme sua operação real.

## Configuração institucional e termos

Preencha `BUSINESS_CNPJ`, `BUSINESS_ADDRESS`, `BUSINESS_PHONE`, `BUSINESS_WHATSAPP` e `BUSINESS_EMAIL` com dados oficiais no `.env`. O WhatsApp deve ser apenas o número internacional com DDI e DDD, sem `+` ou caracteres, por exemplo `5511999999999`.

Os termos ficam em `/termos` e são versionados pela variável opcional `TERMS_VERSION`. O texto incluído é uma base informativa: substitua-o/revise-o com orientação jurídica, informe a entidade responsável, política de entrega/devolução/estorno e publique política de privacidade compatível com a LGPD antes de operar. Não invente CNPJ, endereços ou contatos.

## MySQL: atualização de bancos existentes

`database/schema_mysql.sql` é para uma instalação nova. `db.create_all()` cria tabelas ausentes, mas não altera tabelas existentes. Faça backup antes de qualquer migração.

- Bancos criados a partir do schema da versão anterior com checkout: execute **uma única vez** `database/upgrade_mysql_project.sql`. Ele rebaixa a conta demo antiga conhecida e remove locais ilustrativos sem pedidos históricos.
- Bancos ainda anteriores ao checkout devem primeiro receber a migração legada `database/upgrade_mysql_checkout.sql`, depois `database/upgrade_mysql_project.sql`, conferindo os campos existentes para evitar duplicidade.
- Não execute scripts de migração em banco novo criado pelo `database/schema_mysql.sql`.

O driver usa URL SQLAlchemy com PyMySQL. Se preferir `DATABASE_URL`, ela tem prioridade sobre `MYSQL_*`; faça URL-encoding de caracteres especiais no usuário/senha da URL.

## Funcionalidades

- Catálogo de 16 títulos com capas locais, busca/filtros, sinopses e detalhes bibliográficos.
- Compra e aluguel por 7, 14 ou 30 dias, carrinho por sessão e controle transacional de estoque.
- Checkout com validação de nome, CPF, telefone, e-mail, CEP/endereço ou biblioteca parceira, opção explícita de endereço sem complemento e aceite obrigatório dos termos.
- Modo de demonstração Pix/cartão claramente identificado, separado do Mercado Pago real; integração real possui consulta de status e webhook validado por HMAC-SHA256.
- Avaliações criadas por usuários com pedido pago, uma avaliação por usuário/obra e média calculada em tempo real; o catálogo não contém notas inventadas.
- Busca ViaCEP pelo servidor, unidades autorizadas cadastradas no admin, resultados agrupados por proximidade e interface mobile responsiva.
- Área de cliente com pedidos e área admin privada para acervo, unidades parceiras, entregas e curadoria.
- Hash de senha, proteção CSRF para POST, cookies HttpOnly/SameSite, validação de papel e cabeçalhos HTTP de proteção.

## Imagens e fontes

As capas incluídas em `static/images/books/` foram localizadas em fontes editoriais/catálogos públicos. A edição visual pode diferir do exemplar físico; consulte `docs/capas-fontes.md` para os links e a identificação de cada arquivo. As capas são protegidas por direitos dos titulares: confirme permissões/licenças antes de redistribuição comercial ou publicação em produção, ou substitua-as por imagens licenciadas pela sua organização. Referências técnicas de pagamentos estão em `docs/referencias-integracao.md`.

## Testes e verificações

```bash
python -m unittest discover -s tests -v
python -m compileall -q .
```

## Segurança e repositório

O `.gitignore` exclui `.env`, chaves geradas, bancos locais, caches e ambientes virtuais. Antes de subir seu repositório, confira `git status` e garanta que não incluiu credenciais, dumps SQL de produção, CPF/endereço/telefone reais ou logs. Não reutilize credenciais de teste em produção. O aplicativo protege rotas e não renderiza CPF completo em páginas públicas; dados de checkout necessários ao atendimento ficam no banco, então limite acesso, habilite backup criptografado e defina retenção/descarte conforme sua política.
