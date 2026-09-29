---
titulo: "Framework Reference — Ruby on Rails"
fonte_primaria: "Rails Guides (guides.rubyonrails.org)"
fontes_complementares:
  - "Shopify Engineering Blog"
  - "Sustainable Web Development with Ruby on Rails (David Copeland)"
versao_framework: "7.x"
---

# Ruby on Rails — Reference para Arquiteto Bob

## Deteccao

Sinais de que o projeto usa Ruby on Rails:

| Artefato | Caminho tipico | Observacao |
|---|---|---|
| Gemfile | `./Gemfile` | Contem `gem 'rails'` |
| Script Rails | `./bin/rails` | CLI principal do framework |
| Rotas | `./config/routes.rb` | Arquivo de roteamento centralizado |
| Configuracao | `./config/application.rb` | Classe `Application < Rails::Application` |
| Schema do banco | `./db/schema.rb` ou `./db/structure.sql` | Gerado por migrations |
| Diretorio app/ | `./app/models/`, `./app/controllers/` | Estrutura MVC canonica |

**Heuristica rapida**: a presenca de `Gemfile` com `gem 'rails'` + diretorio `app/` com subpastas `models/`, `controllers/` e `views/` confirma Rails com alta confianca.

---

## Filosofia e Principios Core

O Rails Guides (Getting Started) define dois principios filosoficos centrais:

1. **Don't Repeat Yourself (DRY)**: "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system." O framework reduz duplicacao sistematicamente — migrations como unica fonte da verdade do schema, validations no model em vez de no banco e na UI, etc.
   *Fonte: guides.rubyonrails.org/getting_started.html*

2. **Convention over Configuration**: O framework e deliberadamente opinativo, fazendo premissas sobre a melhor abordagem para o desenvolvimento. Em vez de exigir configuracao extensiva, Rails estabelece defaults sensiveis que desenvolvedores devem adotar.
   *Fonte: guides.rubyonrails.org/getting_started.html*

3. **MVC (Model-View-Controller)**: O Rails organiza codigo em tres conceitos:
   - **Model**: gerencia os dados da aplicacao, tipicamente tabelas do banco
   - **View**: renderiza respostas em formatos como HTML, JSON, XML
   - **Controller**: trata interacoes do usuario e a logica de cada request
   *Fonte: guides.rubyonrails.org/getting_started.html*

4. **RESTful by default**: rotas sao mapeadas a recursos REST com verbos HTTP semanticos. Um unico `resources :photos` gera 7 rotas RESTful.
   *Fonte: guides.rubyonrails.org/routing.html*

---

## Estrutura Canonica

Estrutura de diretorios padrao de uma aplicacao Rails (conforme Rails Guides — Getting Started):

```
app/
  assets/          # Stylesheets, JavaScript, imagens
  channels/        # Action Cable channels (WebSocket)
  controllers/     # Controllers (logica de request/response)
  helpers/         # View helpers (metodos auxiliares de views)
  jobs/            # Active Job classes (processamento assincrono)
  mailers/         # Action Mailer classes (envio de emails)
  models/          # Active Record models (camada de dados)
  views/           # Templates ERB/Haml/Slim
    layouts/       # Layouts compartilhados
bin/               # Scripts utilitarios (rails, rake, setup)
config/
  environments/    # Configuracao por ambiente (development, test, production)
  initializers/    # Codigo executado no boot
  locales/         # Arquivos I18n
  routes.rb        # Definicao de rotas
  database.yml     # Configuracao de banco de dados
  application.rb   # Configuracao principal da aplicacao
db/
  migrate/         # Migration files (versionamento do schema)
  schema.rb        # Schema atual do banco (gerado automaticamente)
  seeds.rb         # Dados iniciais
lib/               # Codigo de biblioteca/extensoes
  tasks/           # Rake tasks customizadas
log/               # Arquivos de log
public/            # Arquivos estaticos servidos diretamente
test/ (ou spec/)   # Testes (MiniTest ou RSpec)
tmp/               # Arquivos temporarios
vendor/            # Dependencias de terceiros
```

*Fonte: guides.rubyonrails.org/getting_started.html*

---

## Patterns Idiomaticos

### Active Record (vs Repository Pattern de Evans)

Active Record e simultaneamente um ORM e um design pattern descrito por Martin Fowler: um objeto que encapsula uma linha do banco de dados, carrega acesso aos dados e adiciona logica de dominio. No Rails, models herdam de `ApplicationRecord` e seguem convencoes estritas de nomenclatura:

- Classes usam UpperCamelCase; tabelas sao plurais em snake_case
- `Book` mapeia para `books`; `LineItem` mapeia para `line_items`
- Chave primaria: `id` (auto-incremento por padrao)
- Chaves estrangeiras: `singularized_table_name_id` (ex: `author_id`)

*Fonte: guides.rubyonrails.org/active_record_basics.html*

**Quando Active Record e suficiente**: para a maioria dos CRUDs, o pattern Active Record cobre modelo de dominio + persistencia de forma pragmatica. O ganho de produtividade compensa a violacao do principio de responsabilidade unica (SRP).

**Quando extrair logica para fora do model**: quando o model acumula callbacks complexos, regras de negocio com multiplos atores, ou orquestracao entre multiplos models — sinais de Fat Model. Nesse caso, Service Objects ou plain Ruby objects sao mais apropriados.

**Exemplo funcional — Model com validations e associations:**

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :books, dependent: :destroy

  validates :name, presence: true
  validates :email, uniqueness: true
end

# app/models/book.rb
class Book < ApplicationRecord
  belongs_to :author

  validates :title, presence: true
  validates :title, length: { minimum: 2, maximum: 255 }
  validates :isbn, uniqueness: true, format: {
    with: /\A[0-9\-]+\z/,
    message: "only allows numbers and hyphens"
  }
end
```

**CRUD — exemplos oficiais:**

```ruby
# Create
book = Book.create(title: "The Lord of the Rings", author: author)

# Read
books = Book.all
book  = Book.find(42)
book  = Book.find_by(title: "Metaprogramming Ruby 2")
Book.where(author: author).order(created_at: :desc)

# Update
book.update(title: "The Fellowship of the Ring")

# Delete
book.destroy
Book.destroy_by(author: author)
```

*Fonte: guides.rubyonrails.org/active_record_basics.html*

**Migrations — versionamento de schema:**

```ruby
class CreateBooks < ActiveRecord::Migration[7.1]
  def change
    create_table :books do |t|
      t.string :title
      t.string :isbn
      t.references :author, null: false, foreign_key: true
      t.timestamps
    end
  end
end
```

*Fonte: guides.rubyonrails.org/active_record_basics.html*

---

### Service Objects

Service Objects sao plain Ruby objects (POROs) que encapsulam logica de negocio complexa que nao pertence nem ao model nem ao controller. Nao ha geracao automatica no Rails — e um pattern da comunidade.

**Quando usar:**
- Logica que envolve multiplos models
- Orquestracao de operacoes (criar registro + enviar email + notificar)
- Regra de negocio que nao se encaixa em nenhum model especifico
- Integracao com servicos externos

**Exemplo funcional:**

```ruby
# app/services/register_author.rb
class RegisterAuthor
  def initialize(params)
    @params = params
  end

  def call
    author = Author.new(@params)

    ActiveRecord::Base.transaction do
      author.save!
      WelcomeMailer.with(author: author).welcome_email.deliver_later
    end

    author
  rescue ActiveRecord::RecordInvalid => e
    OpenStruct.new(errors: e.record.errors)
  end
end

# Uso no controller:
# result = RegisterAuthor.new(author_params).call
```

**Convencao comum**: interface `#call`, um unico ponto de entrada, nomeado como verbo + substantivo.

*Fonte: pattern documentado em "Sustainable Web Development with Ruby on Rails" (David Copeland). O uso de `ActiveRecord::Base.transaction` e `.deliver_later` segue os Rails Guides.*

---

### Concerns (Mixins)

Concerns sao modulos Ruby que usam `ActiveSupport::Concern` para adicionar comportamento compartilhado a models ou controllers. Vivem em `app/models/concerns/` e `app/controllers/concerns/`.

**Exemplo funcional — Concern de model:**

```ruby
# app/models/concerns/archivable.rb
module Archivable
  extend ActiveSupport::Concern

  included do
    scope :archived, -> { where.not(archived_at: nil) }
    scope :active, -> { where(archived_at: nil) }
  end

  def archive!
    update!(archived_at: Time.current)
  end

  def archived?
    archived_at.present?
  end
end

# app/models/book.rb
class Book < ApplicationRecord
  include Archivable

  belongs_to :author
  validates :title, presence: true
end
```

**Uso idiomatico:**
- Behavior reutilizavel entre models nao relacionados (Taggable, Searchable, Archivable)
- Extrair grupos coesos de scopes, validations e callbacks

**Anti-patterns com Concerns:**
- Usar Concerns para esconder Fat Models (mover codigo sem reduzir complexidade)
- Criar concerns com dependencias implicitas entre si
- Criar concerns usados por um unico model (nao ha reutilizacao)

*Fonte: ActiveSupport::Concern e parte do Rails framework (api.rubyonrails.org). O pattern de organizacao em `app/models/concerns/` e convencao do Rails.*

---

### Controllers

O Rails Guides define: "Rails favors making the resource in the controller's name plural. For example, `ClientsController` is preferred over `ClientController`."

*Fonte: guides.rubyonrails.org/action_controller_overview.html*

**Strong Parameters — protecao contra mass assignment:**

```ruby
class BooksController < ApplicationController
  before_action :set_book, only: [:show, :edit, :update, :destroy]

  def index
    @books = Book.all
  end

  def show
  end

  def create
    @book = Book.new(book_params)

    if @book.save
      redirect_to @book, notice: "Book was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @book.update(book_params)
      redirect_to @book, notice: "Book was successfully updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @book.destroy
    redirect_to books_url, notice: "Book was successfully destroyed.", status: :see_other
  end

  private

  def set_book
    @book = Book.find(params[:id])
  end

  def book_params
    params.expect(book: [:title, :isbn, :author_id])
  end
end
```

**Filtros (Callbacks de controller):**

```ruby
class ApplicationController < ActionController::Base
  before_action :require_login

  private

  def require_login
    unless logged_in?
      flash[:error] = "You must be logged in"
      redirect_to new_login_url
    end
  end
end

class SessionsController < ApplicationController
  skip_before_action :require_login, only: [:new, :create]
end
```

*Fonte: guides.rubyonrails.org/action_controller_overview.html*

**Rotas RESTful — o que `resources` gera:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :books
end
```

| HTTP Verb | Path | Action | Uso |
|---|---|---|---|
| GET | /books | books#index | Listar todos |
| GET | /books/new | books#new | Form de criacao |
| POST | /books | books#create | Criar |
| GET | /books/:id | books#show | Exibir um |
| GET | /books/:id/edit | books#edit | Form de edicao |
| PATCH/PUT | /books/:id | books#update | Atualizar |
| DELETE | /books/:id | books#destroy | Remover |

**Rotas aninhadas e namespaces:**

```ruby
# Rotas aninhadas
resources :authors do
  resources :books
end
# Gera: /authors/:author_id/books/:id

# Namespace para admin
namespace :admin do
  resources :books
end
# Gera: /admin/books com Admin::BooksController
```

*Fonte: guides.rubyonrails.org/routing.html*

---

### Background Jobs (Active Job + Sidekiq)

O Rails Guides define Active Job como "a framework designed for declaring background jobs and executing them on a queuing backend." Ele abstrai o backend de fila (Sidekiq, Resque, Delayed Job) com uma interface unificada.

*Fonte: guides.rubyonrails.org/active_job_basics.html*

**Configuracao do adapter:**

```ruby
# config/application.rb
config.active_job.queue_adapter = :sidekiq
```

**Exemplo funcional — Job com retry e error handling:**

```ruby
# app/jobs/guests_cleanup_job.rb
class GuestsCleanupJob < ApplicationJob
  queue_as :default

  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  discard_on ActiveJob::DeserializationError

  def perform(guest)
    guest.cleanup!
  end
end

# Enfileirar para execucao assincrona
GuestsCleanupJob.perform_later(guest)

# Agendar para o futuro
GuestsCleanupJob.set(wait_until: Date.tomorrow.noon).perform_later(guest)
GuestsCleanupJob.set(wait: 1.week).perform_later(guest)

# Enfileirar em lote
cleanup_jobs = Guest.all.map { |guest| GuestsCleanupJob.new(guest) }
ActiveJob.perform_all_later(cleanup_jobs)
```

**Callbacks disponiveis:** `before_enqueue`, `around_enqueue`, `after_enqueue`, `before_perform`, `around_perform`, `after_perform`, `after_discard`.

```ruby
class GuestsCleanupJob < ApplicationJob
  around_perform :measure_execution_time

  def perform(guest)
    guest.cleanup!
  end

  private

  def measure_execution_time
    start_time = Time.now
    yield
    duration = Time.now - start_time
    Rails.logger.info "GuestsCleanupJob took #{duration} seconds"
  end
end
```

*Fonte: guides.rubyonrails.org/active_job_basics.html*

---

## Modularizacao e Boundaries

### Rails Engines

O Rails Guides define: "Engines can be considered miniature applications that provide functionality to their host applications."

*Fonte: guides.rubyonrails.org/engines.html*

Engines sao o mecanismo nativo do Rails para modularizacao. Com `isolate_namespace`, cada engine tem models, controllers, views, rotas e migrations isolados — funcionando como um Bounded Context (na terminologia DDD).

**Gerando um engine mountable:**

```bash
rails plugin new blorgh --mountable
```

**Estrutura gerada:**

```
blorgh/
  app/
    controllers/blorgh/application_controller.rb
    models/blorgh/
    views/layouts/blorgh/application.html.erb
  config/
    routes.rb
  db/
    migrate/
  lib/
    blorgh.rb
    blorgh/engine.rb
  blorgh.gemspec
```

**Isolamento de namespace:**

```ruby
# lib/blorgh/engine.rb
module Blorgh
  class Engine < ::Rails::Engine
    isolate_namespace Blorgh
  end
end
```

Efeitos do `isolate_namespace`:
- Models ficam `Blorgh::Article` (nao `Article`)
- Tabelas usam prefixo `blorgh_articles`
- Controllers ficam `Blorgh::ArticlesController`
- Views ficam em `app/views/blorgh/`
- Rotas sao isoladas dentro do escopo da engine

**Montagem na aplicacao host:**

```ruby
# config/routes.rb (aplicacao host)
mount Blorgh::Engine, at: "/blog"
```

**Rotas dentro da engine:**

```ruby
# config/routes.rb (dentro da engine)
Blorgh::Engine.routes.draw do
  resources :articles do
    resources :comments
  end
  root to: "articles#index"
end
```

*Fonte: guides.rubyonrails.org/engines.html*

---

### Packwerk (Shopify)

Packwerk e uma ferramenta open-source criada pela Shopify para enforcar boundaries em monolitos Rails modulares sem precisar extrair engines ou gems. Ele analisa dependencias entre "packages" (diretorios) e detecta violacoes de encapsulamento em tempo de CI.

**Conceito**: cada diretorio sob `app/` (ou `components/`, `packs/`) pode ser declarado como um package com seu proprio `package.yml`, definindo quais dependencias sao permitidas e quais constantes sao publicas.

**Estrutura tipica:**

```
packs/
  billing/
    app/
      models/billing/invoice.rb
      services/billing/charge_customer.rb
    package.yml           # enforce_dependencies: true, enforce_privacy: true
  shipping/
    app/
      models/shipping/order.rb
    package.yml
```

**Uso**: `bin/packwerk check` no CI rejeita PRs que introduzam dependencias nao declaradas entre packages.

*Fonte: github.com/Shopify/packwerk, Shopify Engineering Blog*

---

## Anti-Patterns Rails

### Fat Model

**Problema**: acumular toda a logica de negocio no model, resultando em classes com centenas de metodos, callbacks encadeados e responsabilidades multiplas.

```ruby
# RUIM — Fat Model
class Order < ApplicationRecord
  has_many :items
  belongs_to :user

  after_create :send_confirmation_email
  after_create :notify_warehouse
  after_create :charge_credit_card
  after_create :update_inventory
  after_create :generate_invoice

  def total
    items.sum(&:price)
  end

  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end

  def charge_credit_card
    PaymentGateway.charge(user.credit_card, total)
  end

  # ... dezenas de metodos a mais
end
```

```ruby
# BOM — Model magro + Service Object para orquestracao
class Order < ApplicationRecord
  has_many :items
  belongs_to :user

  validates :status, presence: true

  def total
    items.sum(&:price)
  end
end

class PlaceOrder
  def initialize(order)
    @order = order
  end

  def call
    ActiveRecord::Base.transaction do
      @order.save!
      ChargePaymentJob.perform_later(@order)
      OrderMailer.confirmation(@order).deliver_later
      UpdateInventoryJob.perform_later(@order)
    end
  end
end
```

---

### God Controller

**Problema**: controller com logica de negocio, queries complexas e manipulacao direta de dados.

```ruby
# RUIM — God Controller
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    @order.status = "pending"
    @order.total = params[:items].sum { |i| Product.find(i[:id]).price * i[:qty] }
    @order.tax = @order.total * 0.15
    @order.save!
    UserMailer.order_confirmation(@order).deliver_now
    redirect_to @order
  end
end
```

```ruby
# BOM — Controller magro delega para service
class OrdersController < ApplicationController
  def create
    result = PlaceOrder.new(order_params).call

    if result.persisted?
      redirect_to result, notice: "Order placed successfully."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def order_params
    params.expect(order: [:user_id, items: [:product_id, :quantity]])
  end
end
```

---

### Callback Hell

**Problema**: callbacks encadeados (`after_create`, `after_save`, `before_validation`) que criam dependencias ocultas e tornam o fluxo imprevisivel.

**Regra pratica**: callbacks sao aceitaveis para manter invariantes do proprio model (ex: normalizar dados antes de salvar). Para efeitos colaterais (enviar email, notificar sistema externo), prefira Service Objects ou Jobs explicitos.

---

### N+1 Queries

**Problema**: carregar uma colecao e depois acessar associacoes lazily, gerando uma query por registro.

```ruby
# RUIM — N+1
books = Book.all
books.each do |book|
  puts book.author.name  # 1 query por book
end

# BOM — Eager loading
books = Book.includes(:author).all
books.each do |book|
  puts book.author.name  # 0 queries adicionais
end
```

Outras opcoes: `Book.eager_load(:author)` (LEFT OUTER JOIN) e `Book.preload(:author)` (query separada).

---

## Reconciliacao com Livros de Arquitetura

| Conceito (livro) | Equivalente Rails | Nota |
|---|---|---|
| Repository (Evans, DDD) | Active Record | AR unifica repositorio + entidade. Em dominios complexos, extrair um objeto Query ou Repository explicitamente |
| Entity (Evans, DDD) | Active Record Model | O model Rails carrega identidade (id) + comportamento, mas tambem persistencia |
| Value Object (Evans, DDD) | PORO ou `composed_of` | Rails nao tem suporte first-class; usar classes Ruby puras (Money, Address) |
| Aggregate Root (Evans, DDD) | Model pai com `has_many dependent: :destroy` | Convencao Rails: destruicao em cascata via `dependent:` |
| Use Case (Martin, Clean Architecture) | Service Object | Classe PORO com metodo `#call` que orquestra a operacao |
| Controller (Martin, Clean Architecture) | Rails Controller | No Rails, o controller ja e o adapter HTTP — nao a camada de use case |
| Presenter / ViewModel | View Helper ou Decorator (Draper) | Rails usa helpers; para complexidade maior, Draper ou ViewComponent |
| Bounded Context (Evans, DDD) | Rails Engine com `isolate_namespace` | Engines sao o mecanismo nativo; Packwerk e alternativa sem overhead de gem |
| Anti-Corruption Layer (Evans, DDD) | Wrapper/Adapter em `lib/` ou `app/services/` | Isolar integracoes externas atras de interface Ruby propria |
| Ports & Adapters (Hexagonal) | Nao idiomatico | Possivel com injecao de dependencia manual, mas vai contra Convention over Configuration |
| CQRS (separar read/write) | Scopes (read) + Service Objects (write) | Separacao natural: scopes no model para queries, services para commands |

---

## Checklist Arquitetural Rails

- [ ] **Gemfile.lock commitado** — garante reproducibilidade de dependencias
- [ ] **Schema versionado** — `db/schema.rb` ou `db/structure.sql` no repositorio
- [ ] **Migrations reversiveis** — toda migration com `change` reversivel ou `up`/`down` explicitos
- [ ] **Strong Parameters em todos os controllers** — nenhum `params.permit!` em producao
- [ ] **N+1 sob controle** — usar `includes`, `eager_load` ou `preload`; considerar gem Bullet em development
- [ ] **Models sem callbacks de efeito colateral** — email, notificacao, API externa via Service/Job
- [ ] **Jobs idem-potentes** — Active Job pode re-executar; garantir que re-execucao e segura
- [ ] **Testes de model, controller e integracao** — cobertura minima das regras de negocio
- [ ] **Environments separados** — `database.yml` com configuracao distinta para development, test, production
- [ ] **Secrets via credentials** — `rails credentials:edit`; nada sensivel em ENV sem criptografia
- [ ] **Rotas RESTful** — evitar rotas customizadas quando `resources` cobre o caso
- [ ] **CI verde** — rubocop (ou standard), testes, e bundler-audit no pipeline

---

## Quando Sair do Rails Way

O Rails Way (Convention over Configuration, Active Record, RESTful resources) funciona excepcionalmente bem para a maioria das aplicacoes web. Porem, ha sinais de que a complexidade ultrapassou o que as convencoes conseguem resolver:

1. **Models com mais de ~300 linhas** apos extrair concerns — indica dominio complexo que precisa de Service Objects, Form Objects ou domain layer explicita.

2. **Callbacks formando cadeias de 3+ niveis** — sinal de que o fluxo precisa ser explicito em um Service Object, nao implicito em callbacks.

3. **Controllers precisando de dados de 4+ models nao relacionados** — indica que a view precisa de um Presenter/ViewModel ou que o dominio precisa de reestruturacao.

4. **Testes de model quebram por efeitos colaterais** — callbacks com side effects dificultam teste unitario; extrair para Jobs ou Services.

5. **Multiplas equipes editando os mesmos models** — sinal classico para considerar Engines ou Packwerk para enforcar boundaries entre dominios.

6. **Regras de negocio que nao mapeiam para CRUD** — processos de negocio complexos (workflow de aprovacao, maquinas de estado multi-step) beneficiam-se de Service Objects ou gems como `aasm`/`statesman`.

7. **Performance exige queries nao-AR** — quando Active Record gera SQL sub-otimo para queries analiticas, usar `find_by_sql` ou Arel e aceitavel; para volumes massivos, considerar read replicas ou CQRS.

**Principio guia**: saia do Rails Way incrementalmente. Comece extraindo Service Objects, depois Form Objects, depois considere Engines. Nao adote Clean Architecture/Hexagonal completa de uma vez — o overhead nao compensa ate que a complexidade justifique.

*Fontes: "Sustainable Web Development with Ruby on Rails" (David Copeland) defende essa abordagem incremental; Rails Guides documentam os mecanismos (Engines, Active Job, Active Record) que suportam cada nivel de extracao.*
