---
titulo: "Framework Reference — Django"
fonte_primaria: "Django Documentation (docs.djangoproject.com)"
fontes_complementares:
  - "Django REST Framework docs"
  - "Two Scoops of Django (Greenfeld & Greenfeld)"
  - "Django Design Philosophies"
versao_framework: "5.x"
---

# Django — Reference para Arquiteto Bob

## Deteccao

O projeto usa Django quando pelo menos um destes indicadores esta presente:

- Arquivo `manage.py` na raiz do projeto
- Arquivo `settings.py` (ou modulo `settings/`) com `INSTALLED_APPS`, `MIDDLEWARE`, `DATABASES`
- `django` listado em `requirements.txt`, `Pipfile`, `pyproject.toml` ou `setup.cfg`
- Arquivo `wsgi.py` ou `asgi.py` com `get_wsgi_application()` / `get_asgi_application()`
- Diretorio `migrations/` dentro de apps com arquivos numerados (`0001_initial.py`)
- Presenca de `urls.py` com `urlpatterns`

Se `djangorestframework` aparece nas dependencias, o projeto tambem usa DRF para APIs REST.

---

## Filosofia e Principios Core

A documentacao oficial do Django (docs.djangoproject.com/en/5.1/misc/design-philosophies/) define os seguintes principios:

### Loose Coupling

> "A fundamental goal of Django's stack is loose coupling and tight cohesion. The various layers of the framework shouldn't 'know' about each other unless absolutely necessary."

O template system nao conhece web requests; a camada de banco nao sabe nada sobre exibicao de dados; o sistema de views nao se importa com qual template engine e usada.

### DRY — Don't Repeat Yourself

> "Every distinct concept and/or piece of data should live in one, and only one, place. Redundancy is bad. Normalization is good."

O framework deduz o maximo possivel a partir do minimo de informacao (ex: campos do model geram automaticamente formularios, admin, migrations).

### Explicit is Better Than Implicit

> "Django shouldn't do too much 'magic.' Magic shouldn't happen unless there's a really good reason for it."

Principio core do Python (PEP 20) adotado pelo Django. O desenvolvedor deve chamar `save()` explicitamente — nao ha gravacao silenciosa em background.

### Less Code / Quick Development

> "Django apps should use as little code as possible; they should lack boilerplate."
> "Django should allow for incredibly quick web development."

O framework aproveita as capacidades dinamicas do Python (introspeccao, metaclasses) para reduzir boilerplate.

### Consistency

> "The framework should be consistent at all levels."

De estilo de codigo a experiencia de uso, a consistencia e um valor central.

### MTV — Model-Template-View

Django adota o pattern **Model-Template-View**, variacao do MVC classico:

| Componente | Responsabilidade | Equivalente MVC |
|---|---|---|
| **Model** | Dados + logica de dominio (Active Record) | Model |
| **Template** | Apresentacao (somente logica de apresentacao) | View |
| **View** | Recebe request, coordena logica, retorna response | Controller |

A filosofia oficial de Models diz: "Models should encapsulate every aspect of an 'object,' following Martin Fowler's Active Record design pattern."

**Fonte:** docs.djangoproject.com/en/5.1/misc/design-philosophies/

---

## Estrutura Canonica

### Projeto Django padrao

```
meu_projeto/
    manage.py
    meu_projeto/              # pacote de configuracao
        __init__.py
        settings.py           # (ou settings/ com base.py, local.py, production.py)
        urls.py               # URLconf raiz
        wsgi.py
        asgi.py
    app_pedidos/              # app de dominio
        __init__.py
        apps.py
        models.py
        views.py
        urls.py
        serializers.py        # se usa DRF
        admin.py
        forms.py
        signals.py
        services.py           # service layer (pattern recomendado para logica complexa)
        managers.py           # custom managers/querysets
        tests/
            __init__.py
            test_models.py
            test_views.py
            test_services.py
        migrations/
            __init__.py
            0001_initial.py
        templates/
            app_pedidos/
        static/
            app_pedidos/
    app_clientes/             # outra app de dominio
        ...
```

### Projeto com apps organizadas em pacote

Para projetos maiores, models podem ser divididos em modulos (conforme docs.djangoproject.com/en/5.1/topics/db/models/):

```
app_pedidos/
    models/
        __init__.py           # from .pedido import Pedido
        pedido.py             # class Pedido(models.Model)
        item_pedido.py        # class ItemPedido(models.Model)
```

---

## Patterns Idiomaticos

### Models e ORM (vs Domain Model de Evans)

Django usa o pattern **Active Record** (Fowler, PoEAA): o model combina dados, persistencia e logica de dominio em uma unica classe. Isso difere do **Domain Model puro** de Evans (DDD), onde entidades nao conhecem o mecanismo de persistencia.

**Reconciliacao com DDD**: em Django, o Manager atua como um **Repository simplificado**, encapsulando queries complexas. Logica de dominio rica pode viver nos methods do model, enquanto orquestracao complexa vai para uma **service layer**.

**Exemplo funcional — Model com Manager customizado:**

```python
# app_pedidos/managers.py
from django.db import models


class PedidoQuerySet(models.QuerySet):
    def pendentes(self):
        return self.filter(status="pendente")

    def do_cliente(self, cliente_id):
        return self.filter(cliente_id=cliente_id)

    def valor_total_acima_de(self, valor):
        return self.filter(valor_total__gte=valor)


class PedidoManager(models.Manager):
    def get_queryset(self):
        return PedidoQuerySet(self.model, using=self._db)

    def pendentes(self):
        return self.get_queryset().pendentes()
```

```python
# app_pedidos/models.py
from django.db import models
from .managers import PedidoManager


class Pedido(models.Model):
    class Status(models.TextChoices):
        PENDENTE = "pendente", "Pendente"
        APROVADO = "aprovado", "Aprovado"
        CANCELADO = "cancelado", "Cancelado"

    cliente = models.ForeignKey(
        "app_clientes.Cliente",
        on_delete=models.PROTECT,
        related_name="pedidos",
    )
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDENTE,
    )
    valor_total = models.DecimalField(max_digits=10, decimal_places=2)
    criado_em = models.DateTimeField(auto_now_add=True)
    atualizado_em = models.DateTimeField(auto_now=True)

    objects = PedidoManager()

    class Meta:
        ordering = ["-criado_em"]
        verbose_name = "pedido"
        verbose_name_plural = "pedidos"

    def __str__(self):
        return f"Pedido #{self.pk} - {self.get_status_display()}"

    def aprovar(self):
        """Logica de dominio no model — adequado para regras simples."""
        if self.status != self.Status.PENDENTE:
            raise ValueError("Apenas pedidos pendentes podem ser aprovados.")
        self.status = self.Status.APROVADO
        self.save(update_fields=["status", "atualizado_em"])

    def cancelar(self):
        if self.status == self.Status.CANCELADO:
            raise ValueError("Pedido ja esta cancelado.")
        self.status = self.Status.CANCELADO
        self.save(update_fields=["status", "atualizado_em"])
```

Uso:
```python
# Equivalente funcional a um Repository
Pedido.objects.pendentes()
Pedido.objects.pendentes().do_cliente(cliente_id=42)
Pedido.objects.pendentes().valor_total_acima_de(1000)
```

**Fonte:** docs.djangoproject.com/en/5.1/topics/db/models/ (secoes "Managers", "Custom managers", "Model methods")

---

### Views e ViewSets

Django oferece dois estilos de views. A filosofia oficial diz:

> "Writing a view should be as simple as writing a Python function. Developers shouldn't have to instantiate a class when a function will do."

**Function-Based View (FBV) — simples e explicita:**

```python
# app_pedidos/views.py
from django.http import JsonResponse
from django.shortcuts import get_object_or_404
from .models import Pedido


def pedido_detail(request, pk):
    pedido = get_object_or_404(Pedido, pk=pk)
    return JsonResponse({
        "id": pedido.pk,
        "status": pedido.get_status_display(),
        "valor_total": str(pedido.valor_total),
    })
```

**Class-Based View (CBV) — reutilizavel via heranca:**

```python
# app_pedidos/views.py
from django.views.generic import ListView


class PedidoPendenteListView(ListView):
    model = Pedido
    template_name = "app_pedidos/pendentes.html"
    context_object_name = "pedidos"

    def get_queryset(self):
        return Pedido.objects.pendentes()
```

**Quando usar cada um:**

| Situacao | Recomendacao |
|---|---|
| Logica simples, poucos HTTP methods | FBV |
| CRUD padrao com templates | CBV genericas (ListView, DetailView, CreateView) |
| API REST com DRF | ViewSets (ver secao DRF) |
| Logica complexa com muitos branches | FBV ou service layer + CBV |

**Fonte:** docs.djangoproject.com/en/5.1/topics/class-based-views/

---

### Django Apps como Bounded Contexts

Uma **Django app** e a unidade natural para representar um **Bounded Context** (Evans, DDD). Cada app deve:

1. Ter um dominio bem definido (ex: `pedidos`, `clientes`, `estoque`)
2. Encapsular seus models, views, serializers e services
3. Comunicar-se com outras apps via **interfaces explicitas** (imports de models ou services), nao via acesso direto a tabelas internas

**Exemplo de estrutura orientada a dominios:**

```
projeto/
    app_clientes/         # Bounded Context: Clientes
        models.py         # Cliente, Endereco
        services.py       # ClienteService
    app_pedidos/          # Bounded Context: Pedidos
        models.py         # Pedido, ItemPedido
        services.py       # PedidoService (usa app_clientes.models.Cliente via FK)
    app_estoque/          # Bounded Context: Estoque
        models.py         # Produto, MovimentacaoEstoque
        services.py       # EstoqueService
    app_financeiro/       # Bounded Context: Financeiro
        models.py         # Pagamento, NotaFiscal
        services.py       # PagamentoService
```

**Regras de dependencia entre apps:**

- Dependencias devem ser **unidirecionais** (evitar ciclos)
- Usar `ForeignKey` com string reference (`"app_clientes.Cliente"`) para evitar imports circulares
- Logica que cruza dominios pertence a uma **service layer**, nao ao model
- Se duas apps precisam de dependencia bidirecional, considere extrair um terceiro app ou usar signals

**Fonte:** "Two Scoops of Django" (Greenfeld & Greenfeld), Cap. 4 — "Fundamentals of Django App Design"

---

### Signals vs Chamadas Explicitas

A documentacao oficial de signals (docs.djangoproject.com/en/5.1/topics/signals/) inclui um aviso explicito:

> "Signals give the appearance of loose coupling but can lead to hard-to-understand, difficult-to-debug code. Direct function calls are preferred when possible."

**Quando usar signals:**

- Reagir a eventos de **apps de terceiros** que voce nao controla (ex: `post_save` do auth.User)
- Notificacao **cross-cutting** onde o emissor nao deve conhecer os receptores
- Invalidacao de cache em multiplas apps

**Quando evitar signals:**

- Comunicacao entre apps **que voce controla** — use chamada explicita de service
- Logica de negocio critica — signals escondem fluxo e dificultam debugging
- Efeitos colaterais que devem participar da mesma transacao

**Exemplo funcional — signal bem aplicado (reagir a model de terceiro):**

```python
# app_clientes/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User


@receiver(post_save, sender=User)
def criar_perfil_cliente(sender, instance, created, **kwargs):
    """Cria perfil de cliente quando um User do Django e criado."""
    if created:
        from .models import PerfilCliente
        PerfilCliente.objects.create(user=instance)
```

```python
# app_clientes/apps.py
from django.apps import AppConfig


class AppClientesConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "app_clientes"

    def ready(self):
        from . import signals  # noqa: F401 — conecta os receivers
```

**Alternativa explicita (preferivel quando voce controla ambos os lados):**

```python
# app_pedidos/services.py
from app_estoque.services import EstoqueService


class PedidoService:
    @staticmethod
    def confirmar_pedido(pedido):
        """Chamada explicita — fluxo visivel e debugavel."""
        pedido.aprovar()
        for item in pedido.itens.select_related("produto"):
            EstoqueService.reservar(produto=item.produto, quantidade=item.quantidade)
```

**Fonte:** docs.djangoproject.com/en/5.1/topics/signals/

---

### Middleware Pipeline

Middleware e o mecanismo de hooks para cross-cutting concerns em Django. O pipeline funciona como uma "cebola" (onion model): request flui de cima para baixo; response flui de baixo para cima (ordem reversa).

**Fonte:** docs.djangoproject.com/en/5.1/topics/http/middleware/

**Exemplo funcional — middleware de auditoria de tempo de resposta:**

```python
# comum/middleware.py
import logging
import time

logger = logging.getLogger(__name__)


class RequestTimingMiddleware:
    """Loga o tempo de processamento de cada request."""

    def __init__(self, get_response):
        self.get_response = get_response
        # Inicializacao unica — executada quando o servidor sobe.

    def __call__(self, request):
        inicio = time.monotonic()

        response = self.get_response(request)

        duracao_ms = (time.monotonic() - inicio) * 1000
        logger.info(
            "method=%s path=%s status=%s duration=%.2fms",
            request.method,
            request.path,
            response.status_code,
            duracao_ms,
        )
        return response
```

Registro em `settings.py`:

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "comum.middleware.RequestTimingMiddleware",  # apos autenticacao
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

**Ordem importa:** `AuthenticationMiddleware` precisa rodar depois de `SessionMiddleware`. Middleware customizado de auditoria geralmente vem apos autenticacao para ter acesso a `request.user`.

---

### Services / Use Cases

Django nao prescreve um service layer, mas para **logica que nao pertence nem ao model nem a view**, a comunidade Django consolidou o pattern de **services**: funcoes ou classes em `services.py` que orquestram multiplos models e side effects.

Este pattern alinha-se com o **Use Case** de Clean Architecture (Martin) e o **Application Service** de DDD (Evans).

**Quando usar service layer:**

- Logica envolve multiplos models ou apps
- Ha side effects alem do banco (envio de email, chamada a API externa)
- A regra de negocio e complexa demais para viver no `save()` do model
- Voce precisa de uma unidade testavel independente do HTTP

**Exemplo funcional:**

```python
# app_pedidos/services.py
from django.db import transaction
from .models import Pedido, ItemPedido


class CriarPedidoService:
    """
    Use Case: criar pedido com itens.
    Orquestra validacao, persistencia e side effects.
    """

    @staticmethod
    @transaction.atomic
    def executar(cliente, itens_data):
        """
        Args:
            cliente: instancia de Cliente
            itens_data: lista de dicts com {produto_id, quantidade, preco_unitario}

        Returns:
            Pedido criado

        Raises:
            ValueError: se a lista de itens estiver vazia
        """
        if not itens_data:
            raise ValueError("Pedido deve conter ao menos um item.")

        valor_total = sum(
            item["quantidade"] * item["preco_unitario"] for item in itens_data
        )

        pedido = Pedido.objects.create(
            cliente=cliente,
            valor_total=valor_total,
        )

        itens = [
            ItemPedido(
                pedido=pedido,
                produto_id=item["produto_id"],
                quantidade=item["quantidade"],
                preco_unitario=item["preco_unitario"],
            )
            for item in itens_data
        ]
        ItemPedido.objects.bulk_create(itens)

        return pedido
```

Uso na view:

```python
# app_pedidos/views.py
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .services import CriarPedidoService
from .serializers import PedidoSerializer, CriarPedidoInputSerializer


@api_view(["POST"])
def criar_pedido(request):
    input_serializer = CriarPedidoInputSerializer(data=request.data)
    input_serializer.is_valid(raise_exception=True)

    pedido = CriarPedidoService.executar(
        cliente=request.user.perfil_cliente,
        itens_data=input_serializer.validated_data["itens"],
    )

    return Response(
        PedidoSerializer(pedido).data,
        status=status.HTTP_201_CREATED,
    )
```

**Fonte:** "Two Scoops of Django" (Greenfeld & Greenfeld), Cap. 8 — "Function- and Class-Based Views"; HackSoftware Django Styleguide (github.com/HackSoftware/Django-Styleguide)

---

## API Design com Django REST Framework

DRF (django-rest-framework.org) e o toolkit padrao para construir APIs REST em Django. Seus componentes principais:

### Serializers

Convertem instancias de models em representacoes JSON e vice-versa. Funcionam de forma analoga a `forms.Form` do Django.

**Fonte:** django-rest-framework.org/api-guide/serializers/

```python
# app_pedidos/serializers.py
from rest_framework import serializers
from .models import Pedido, ItemPedido


class ItemPedidoSerializer(serializers.ModelSerializer):
    class Meta:
        model = ItemPedido
        fields = ["id", "produto", "quantidade", "preco_unitario"]


class PedidoSerializer(serializers.ModelSerializer):
    itens = ItemPedidoSerializer(many=True, read_only=True)
    status_display = serializers.CharField(
        source="get_status_display", read_only=True
    )

    class Meta:
        model = Pedido
        fields = [
            "id",
            "cliente",
            "status",
            "status_display",
            "valor_total",
            "criado_em",
            "itens",
        ]
        read_only_fields = ["valor_total", "criado_em"]


class CriarPedidoItemInputSerializer(serializers.Serializer):
    produto_id = serializers.IntegerField()
    quantidade = serializers.IntegerField(min_value=1)
    preco_unitario = serializers.DecimalField(max_digits=10, decimal_places=2)


class CriarPedidoInputSerializer(serializers.Serializer):
    itens = CriarPedidoItemInputSerializer(many=True, allow_empty=False)
```

**Validacao no serializer:**

```python
class PedidoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Pedido
        fields = "__all__"

    def validate_valor_total(self, value):
        if value <= 0:
            raise serializers.ValidationError("Valor total deve ser positivo.")
        return value

    def validate(self, data):
        """Validacao cross-field."""
        if data.get("status") == "aprovado" and data.get("valor_total", 0) == 0:
            raise serializers.ValidationError(
                "Pedido aprovado deve ter valor total maior que zero."
            )
        return data
```

### ViewSets e Routers

ViewSets agrupam logica relacionada (list, create, retrieve, update, destroy) em uma unica classe. Routers geram URL patterns automaticamente.

**Fonte:** django-rest-framework.org/api-guide/viewsets/

```python
# app_pedidos/viewsets.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from .models import Pedido
from .serializers import PedidoSerializer


class PedidoViewSet(viewsets.ModelViewSet):
    """
    CRUD completo de pedidos + acoes customizadas.
    """
    queryset = Pedido.objects.all()
    serializer_class = PedidoSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        """Filtra pedidos pelo cliente autenticado."""
        return Pedido.objects.filter(
            cliente=self.request.user.perfil_cliente
        ).select_related("cliente")

    @action(detail=True, methods=["post"])
    def aprovar(self, request, pk=None):
        """POST /api/pedidos/{id}/aprovar/"""
        pedido = self.get_object()
        try:
            pedido.aprovar()
        except ValueError as e:
            return Response(
                {"detail": str(e)},
                status=status.HTTP_400_BAD_REQUEST,
            )
        return Response(PedidoSerializer(pedido).data)

    @action(detail=False)
    def pendentes(self, request):
        """GET /api/pedidos/pendentes/"""
        qs = self.get_queryset().pendentes()
        page = self.paginate_queryset(qs)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        serializer = self.get_serializer(qs, many=True)
        return Response(serializer.data)
```

```python
# app_pedidos/urls.py
from rest_framework.routers import DefaultRouter
from .viewsets import PedidoViewSet

router = DefaultRouter()
router.register(r"pedidos", PedidoViewSet, basename="pedido")

urlpatterns = router.urls
```

```python
# meu_projeto/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("app_pedidos.urls")),
]
```

**Permissoes condicionais por action:**

```python
def get_permissions(self):
    if self.action == "list":
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdminUser]
    return [permission() for permission in permission_classes]
```

**Fonte:** django-rest-framework.org/api-guide/viewsets/ (secao "Introspecting ViewSet actions")

---

## Anti-Patterns Django

### 1. Fat Views

**Problema:** toda logica de negocio vive dentro da view, tornando-a impossivel de testar sem HTTP.

```python
# RUIM — fat view
@api_view(["POST"])
def criar_pedido(request):
    itens = request.data.get("itens", [])
    if not itens:
        return Response({"error": "Sem itens"}, status=400)
    valor_total = sum(i["quantidade"] * i["preco_unitario"] for i in itens)
    pedido = Pedido.objects.create(
        cliente=request.user.perfil_cliente,
        valor_total=valor_total,
    )
    for item in itens:
        ItemPedido.objects.create(pedido=pedido, **item)
    # Enviar email inline...
    send_mail("Pedido criado", "...", "no-reply@ex.com", [request.user.email])
    return Response({"id": pedido.pk}, status=201)
```

```python
# BOM — view magra delega para service
@api_view(["POST"])
def criar_pedido(request):
    serializer = CriarPedidoInputSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    pedido = CriarPedidoService.executar(
        cliente=request.user.perfil_cliente,
        itens_data=serializer.validated_data["itens"],
    )
    return Response(PedidoSerializer(pedido).data, status=201)
```

### 2. Signal Spaghetti

**Problema:** cadeia de signals onde `post_save` de A dispara signal em B que dispara signal em C. Fluxo invisivel, debugging impossivel.

```python
# RUIM — signal cascata
@receiver(post_save, sender=Pedido)
def ao_salvar_pedido(sender, instance, **kwargs):
    atualizar_estoque(instance)  # que por sua vez dispara outro signal...

@receiver(post_save, sender=Estoque)
def ao_atualizar_estoque(sender, instance, **kwargs):
    notificar_fornecedor(instance)  # e outro...
```

```python
# BOM — service com fluxo explicito
class ConfirmarPedidoService:
    @staticmethod
    @transaction.atomic
    def executar(pedido):
        pedido.aprovar()
        EstoqueService.reservar_itens(pedido)
        NotificacaoService.notificar_fornecedor(pedido)
```

### 3. Circular Imports entre Apps

**Problema:** `app_pedidos.models` importa `app_clientes.models` que importa `app_pedidos.models`.

```python
# RUIM — import circular
from app_clientes.models import Cliente  # no topo de app_pedidos/models.py

# BOM — usar string reference no ForeignKey
cliente = models.ForeignKey("app_clientes.Cliente", on_delete=models.PROTECT)
```

### 4. N+1 Queries com ORM

**Problema:** iterar sobre queryset acessando relacoes sem `select_related` ou `prefetch_related`.

```python
# RUIM — N+1 (1 query para pedidos + N queries para cliente)
pedidos = Pedido.objects.all()
for pedido in pedidos:
    print(pedido.cliente.nome)  # query por iteracao!

# BOM — 2 queries no total
pedidos = Pedido.objects.select_related("cliente").all()
for pedido in pedidos:
    print(pedido.cliente.nome)  # sem query adicional

# Para relacoes many-to-many ou reverse FK:
pedidos = Pedido.objects.prefetch_related("itens").all()
```

A filosofia oficial diz:

> "It [the ORM] should execute SQL statements as few times as possible, and it should optimize statements internally."

O `select_related()` e descrito como "optional performance optimization" — o desenvolvedor deve optar explicitamente.

**Fonte:** docs.djangoproject.com/en/5.1/misc/design-philosophies/ (secao "SQL efficiency")

---

## Reconciliacao com Livros de Arquitetura

| Conceito (livro) | Equivalente Django | Nota |
|---|---|---|
| **Repository** (Evans, DDD) | Django ORM Manager / QuerySet customizado | Manager encapsula acesso a dados; QuerySets sao chainable e lazy |
| **Entity** (Evans, DDD) | Django Model com `pk` | Model com identidade via primary key; lifecycle gerenciado pelo ORM |
| **Value Object** (Evans, DDD) | `dataclass` ou `NamedTuple` separado | Django nao tem suporte nativo; usar Python puro para VOs |
| **Aggregate** (Evans, DDD) | Model raiz + related models | Usar `select_related`/`prefetch_related` para carregar o aggregate |
| **Use Case** (Martin, Clean Arch) | Service layer (`services.py`) | Funcao/classe que orquestra models e side effects |
| **Controller** (MVC) | Django View / DRF ViewSet | Recebe request, delega para service/model, retorna response |
| **Active Record** (Fowler, PoEAA) | Django Model | Pattern explicitamente adotado pela filosofia do Django |
| **Data Mapper** (Fowler, PoEAA) | Nao ha equivalente nativo | Django usa Active Record; para Data Mapper puro, considerar SQLAlchemy |
| **Domain Event** (Evans, DDD) | Django Signal (com ressalvas) | Signals funcionam como eventos, mas sem garantia transacional |
| **Anti-Corruption Layer** (Evans) | App wrapper / Adapter pattern | App dedicada que traduz entre sistema legado e dominio Django |
| **Specification** (Evans, DDD) | QuerySet filters / Q objects | `Q(status="ativo") & Q(valor__gte=100)` compoe specs |
| **Unit of Work** (Fowler, PoEAA) | `transaction.atomic()` | Garante atomicidade de operacoes no service layer |

---

## Checklist Arquitetural Django

- [ ] **Models representam o dominio?** Nomes refletem a linguagem ubiqua do negocio, nao termos tecnicos
- [ ] **Apps tem coesao interna?** Cada app cobre um unico bounded context / dominio
- [ ] **Dependencias entre apps sao unidirecionais?** Sem ciclos de importacao
- [ ] **Views sao magras?** Delegam para services ou model methods; nao contem logica de negocio
- [ ] **Queries sao eficientes?** `select_related` e `prefetch_related` usados em relacoes acessadas em loops
- [ ] **Signals sao justificados?** Usados apenas quando emissor nao pode/deve conhecer o receptor
- [ ] **Service layer existe para logica complexa?** Operacoes multi-model e side effects vivem em `services.py`
- [ ] **Serializers validam input?** Validacao field-level e object-level implementada
- [ ] **Permissions sao granulares?** DRF `permission_classes` definidas por ViewSet/action
- [ ] **Middleware e cross-cutting?** Apenas concerns que afetam TODAS as requests (logging, timing, headers)
- [ ] **Settings separados por ambiente?** `base.py`, `local.py`, `production.py` em vez de `if DEBUG`
- [ ] **Migrations estao consistentes?** `makemigrations --check` nao gera novas migrations
- [ ] **`transaction.atomic()` protege operacoes compostas?** Especialmente em services

---

## Quando Sair do Django Way

Django e opinativo e seu "happy path" (Active Record + MTV + DRF) funciona bem para a maioria dos projetos. Porem, alguns sinais indicam que patterns adicionais sao necessarios:

### Sinais de complexidade crescente

1. **Models com mais de 500 linhas** — logica de dominio precisa ser extraida para services ou domain objects separados
2. **`save()` override com muitos branches** — considerar services explicitos em vez de logica no model lifecycle
3. **Signal chains com mais de 2 niveis** — substituir por orquestracao explicita em service layer
4. **Circular imports frequentes entre apps** — reavaliar limites de bounded contexts; possivelmente extrair app intermediaria
5. **Queries ORM extremamente complexas** — considerar `raw()` SQL ou views materializadas no banco; a filosofia Django diz: "The framework should make it easy to write custom SQL"
6. **Necessidade de CQRS** — separar models/querysets de leitura dos de escrita; DRF permite serializers diferentes por action
7. **Multiplos consumers do mesmo dominio** (API REST + GraphQL + CLI + worker Celery) — service layer forte se torna obrigatorio para evitar duplicacao
8. **Regras de dominio que nao encaixam em Active Record** — considerar um domain layer com objetos Python puros (dataclasses) e usar o ORM apenas como mecanismo de persistencia (aproximacao ao Data Mapper)

### O que NAO justifica sair do Django Way

- "Quero seguir Clean Architecture ao pe da letra" — o custo de abstrair o ORM raramente compensa em projetos Django
- "Models nao deviam saber sobre banco" — Active Record e uma decisao de design deliberada do Django, nao um acidente
- "Preciso de Repository pattern" — Managers ja cumprem esse papel; criar repositorios adicionais adiciona indirection sem beneficio

**Regra de ouro:** adicione camadas somente quando a dor de manutencao justificar o custo de abstraccao. O "Django Way" escala bem ate projetos de porte medio-grande. Alem disso, evolua incrementalmente — nao reescreva tudo de uma vez.

**Fontes:**
- docs.djangoproject.com/en/5.1/misc/design-philosophies/
- "Two Scoops of Django" (Greenfeld & Greenfeld), Cap. 29 — "What About Those Random Utilities?"
- HackSoftware Django Styleguide (github.com/HackSoftware/Django-Styleguide)