# 🏛️ Plano de Consolidação Arquitetural — V1 para Venda de Serviços

> **Documento**: Plano de Implementação — Nível 3 (Consolidação)
> **Objetivo**: Preparar a plataforma para operar como serviço multi-cliente (SaaS) com segurança, isolamento e governança de acesso.
> **Ordem de execução**: RBAC → Multi-Tenant → Segurança de Rotas → Governança LGPD → Infraestrutura NGINX/Cloudflare

---

## 📊 Diagnóstico Atual

### Estado da Autenticação
- ✅ JWT com Argon2 (via `python-jose` + `passlib`)
- ✅ `get_current_user` e `get_current_active_superuser` como dependencies
- ✅ Suporte a API Keys para integrações M2M
- ⚠️ Coluna `role` existe na tabela `users` mas é um `String(50)` sem validação — não há tabela de permissões
- ❌ Sem sistema RBAC granular — apenas binário `is_superuser` ou `role="user"`

### Estado do Multi-Tenant
- ⚠️ Campo `empresa_id` existe em `Bot` e `ChatSession` mas **não há tabela de empresas/tenants**
- ⚠️ `ExecutionContext` tem `tenant_id` mas nunca é populado automaticamente
- ❌ Nenhum filtro automático por tenant nas queries
- ❌ Sem associação `User ↔ Tenant`

### Estado da Segurança de Rotas
- ✅ Idempotência implementada em 10 rotas críticas via Redis
- ✅ Alfândega (CustomsMiddleware) para integrações externas
- ✅ CorrelationMiddleware com `X-Correlation-Id` e `X-Session-Id`
- ⚠️ CORS com `allow_methods=["*"]` e `allow_headers=["*"]`
- ❌ **14+ rotas sem autenticação** — Rules, Events, Actions, Pedidos, Campanhas abertas
- ❌ Sem rate limiting global por IP/user
- ❌ Sem CSP, X-Frame-Options, X-Content-Type-Options

### Rotas Sem Autenticação (GAP Crítico)

| Módulo | Rotas Desprotegidas | Risco |
|--------|---------------------|-------|
| `platform/rules` | GET `/`, POST `/`, POST `/{id}/execute`, POST `/execute-all` | **CRÍTICO** — execução de regras sem auth |
| `platform/events` | GET `/`, POST `/trigger`, GET `/exceptions` | **CRÍTICO** — trigger de eventos manual sem auth |
| `platform/actions` | GET `""`, POST `""` | **ALTO** — cadastro de ações sem auth |
| `erp/pedidos` | GET `/`, GET `/{id}`, POST `/`, DELETE `/{id}` | **ALTO** — CRUD de pedidos sem auth |
| `crm/campanhas` | GET `/`, GET `/{id}`, POST `/`, PATCH `/{id}`, DELETE `/{id}` | **ALTO** — CRUD de campanhas sem auth |

---

## 🔑 FASE 1 — RBAC (Role-Based Access Control)

> **Princípio**: Consolidar o campo `role` da tabela `users` com um sistema de permissões granulares baseado em roles pré-definidas, verificáveis em cada rota via dependency injection.

### 1.1 Modelo de Dados

#### [NEW] `app/modules/platform/auth/permissions.py`

Módulo central de definição de roles e permissões usando Enums Python (sem tabela extra no banco):

```python
# Roles hierárquicas (ordenadas por privilégio)
class Role(str, Enum):
    VIEWER = "viewer"         # Somente leitura
    OPERATOR = "operator"     # Operações do dia-a-dia (atendente)
    MANAGER = "manager"       # Gestão do domínio (gerente de loja)
    ADMIN = "admin"           # Administração completa da instância
    SUPERADMIN = "superadmin" # Platform Owner (nós, provedores do SaaS)

# Permissões atômicas agrupadas por domínio
class Permission(str, Enum):
    # ERP
    ERP_PRODUTOS_READ = "erp.produtos.read"
    ERP_PRODUTOS_WRITE = "erp.produtos.write"
    ERP_VENDAS_READ = "erp.vendas.read"
    ERP_VENDAS_WRITE = "erp.vendas.write"
    ERP_ESTOQUE_READ = "erp.estoque.read"
    ERP_ESTOQUE_WRITE = "erp.estoque.write"
    ERP_PEDIDOS_READ = "erp.pedidos.read"
    ERP_PEDIDOS_WRITE = "erp.pedidos.write"
    ERP_CLIENTES_READ = "erp.clientes.read"
    ERP_CLIENTES_WRITE = "erp.clientes.write"
    ERP_FORNECEDORES_READ = "erp.fornecedores.read"
    ERP_FORNECEDORES_WRITE = "erp.fornecedores.write"
    ERP_PAGAMENTOS_READ = "erp.pagamentos.read"
    ERP_PAGAMENTOS_WRITE = "erp.pagamentos.write"
    ERP_RELATORIOS_READ = "erp.relatorios.read"
    
    # CRM
    CRM_LEADS_READ = "crm.leads.read"
    CRM_LEADS_WRITE = "crm.leads.write"
    CRM_CAMPANHAS_READ = "crm.campanhas.read"
    CRM_CAMPANHAS_WRITE = "crm.campanhas.write"
    CRM_COMUNICACAO_READ = "crm.comunicacao.read"
    CRM_COMUNICACAO_WRITE = "crm.comunicacao.write"
    CRM_CHATBOT_READ = "crm.chatbot.read"
    CRM_CHATBOT_WRITE = "crm.chatbot.write"
    CRM_DESK_READ = "crm.desk.read"
    CRM_DESK_WRITE = "crm.desk.write"
    
    # Platform
    PLATFORM_RULES_READ = "platform.rules.read"
    PLATFORM_RULES_WRITE = "platform.rules.write"
    PLATFORM_EVENTS_READ = "platform.events.read"
    PLATFORM_EVENTS_WRITE = "platform.events.write"
    PLATFORM_ACTIONS_READ = "platform.actions.read"
    PLATFORM_ACTIONS_WRITE = "platform.actions.write"
    PLATFORM_INTEGRATIONS_MANAGE = "platform.integrations.manage"
    PLATFORM_USERS_MANAGE = "platform.users.manage"

# Mapeamento Role → Permissões (hierárquico)
ROLE_PERMISSIONS: dict[Role, set[Permission]] = {
    Role.VIEWER: { ... somente .read ... },
    Role.OPERATOR: { ...read + write operacional... },
    Role.MANAGER: { ...tudo do operator + relatorios + campanhas... },
    Role.ADMIN: { ...tudo exceto platform.users.manage com superadmin... },
    Role.SUPERADMIN: { ...ALL... },
}
```

**Decisão de design**: Manter permissões em código (não em tabela) porque:
1. Evita N+1 queries em cada request para buscar permissões
2. Permissões são definidas pelo provedor (nós), não pelo cliente
3. Cache em memória — zero latência
4. Migração Alembic desnecessária para adicionar permissão

### 1.2 Dependency de Verificação

#### [MODIFY] `app/modules/platform/auth/dependencies.py`

Adicionar uma factory de dependency parametrizável:

```python
def require_permission(*permissions: Permission):
    """
    Dependency factory que verifica se o usuário tem TODAS as permissões listadas.
    Uso: Depends(require_permission(Permission.ERP_VENDAS_WRITE))
    """
    async def _checker(current_user: User = Depends(get_current_user)) -> User:
        user_role = Role(current_user.role)
        user_permissions = ROLE_PERMISSIONS.get(user_role, set())
        
        missing = set(permissions) - user_permissions
        if missing:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permissões insuficientes. Requer: {[p.value for p in missing]}"
            )
        return current_user
    return _checker


def require_role(minimum_role: Role):
    """
    Dependency que verifica role mínima (hierárquica).
    Uso: Depends(require_role(Role.MANAGER))
    """
    ...
```

### 1.3 Migration do Campo `role`

#### [MODIFY] Alembic migration

- Alterar coluna `role` de `String(50)` para `String(50)` com CHECK constraint no enum `Role`
- Migrar registros existentes: `"user"` → `"operator"`, manter `is_superuser=True` → `"superadmin"`
- Depreciar campo `is_superuser` (manter por retrocompatibilidade, derivar de `role`)

### 1.4 Aplicação nas Rotas

Adicionar `Depends(require_permission(...))` em **todas** as rotas atualmente desprotegidas:

| Arquivo | Ação |
|---------|------|
| `platform/rules/routes.py` | Adicionar `require_permission(PLATFORM_RULES_READ/WRITE)` |
| `platform/events/routes.py` | Adicionar `require_permission(PLATFORM_EVENTS_READ/WRITE)` |
| `platform/actions/routes.py` | Adicionar `require_permission(PLATFORM_ACTIONS_READ/WRITE)` |
| `erp/pedidos/routes.py` | Adicionar `require_permission(ERP_PEDIDOS_READ/WRITE)` |
| `crm/campanhas/routes.py` | Adicionar `require_permission(CRM_CAMPANHAS_READ/WRITE)` |
| Todas as demais rotas com `get_current_user` | Substituir por `require_permission(...)` adequada |

### 1.5 Atualizar JWT Payload

Incluir `role` no claim do token JWT para evitar query extra:

```python
# security.py → create_access_token
data={"sub": str(user.id), "email": user.email, "role": user.role}
```

### 1.6 Endpoint de Gestão de Usuários

#### [NEW] Rotas administrativas em `auth/routes.py`

```
GET    /api/v1/auth/users          → Lista usuários (ADMIN+)
PATCH  /api/v1/auth/users/{id}     → Atualiza role/status (ADMIN+)
DELETE /api/v1/auth/users/{id}     → Soft-delete (SUPERADMIN)
```

---

## 🏢 FASE 2 — Multi-Tenant

> **Princípio**: Adicionar uma camada de isolamento por `tenant_id` sobre **todos** os registros do banco, garantindo que um cliente nunca acesse dados de outro.

### 2.1 Modelo de Dados

#### [NEW] `app/modules/platform/tenant/models.py`

```python
class Tenant(Base):
    __tablename__ = "tenants"
    
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False)
    slug = Column(String(100), unique=True, nullable=False, index=True)  # subdomínio
    is_active = Column(Boolean, default=True)
    plan = Column(String(50), default="free")  # free | starter | pro | enterprise
    settings = Column(JSONB, default={})  # limites, features flags
    
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    deleted_at = Column(DateTime(timezone=True), nullable=True)
```

#### [MODIFY] `app/modules/platform/auth/models.py` — `User`

```python
class User(Base):
    ...
    tenant_id = Column(UUID, ForeignKey("tenants.id"), nullable=False, index=True)
    ...
```

### 2.2 Mixin de Tenant para Models

#### [NEW] `app/core/mixins/tenant.py`

```python
class TenantMixin:
    """Mixin que adiciona tenant_id a qualquer model."""
    
    @declared_attr
    def tenant_id(cls):
        return Column(
            UUID(as_uuid=True), 
            ForeignKey("tenants.id"), 
            nullable=False, 
            index=True
        )
```

**Models que receberão o mixin** (Alembic migration):

| Domínio | Models |
|---------|--------|
| ERP | `Produto`, `Venda`, `ItemVenda`, `Pedido`, `Cliente`, `EnderecoCliente`, `Fornecedor`, `MovimentacaoEstoque`, `TransacaoPix` |
| CRM | `Lead`, `LeadHistory`, `Campanha`, `Comunicacao`, `Bot`, `ChatSession` |
| Platform | `Event`, `Integration`, `IntegrationAudit`, `PlatformWorkflow`, `Rule` |
| Cognitive | `ResponseTemplate`, `IntentLog`, `LearnedIntent` |

### 2.3 Contexto de Tenant (Context Variable)

#### [NEW] `app/core/tenant_context.py`

```python
from contextvars import ContextVar

_tenant_id: ContextVar[str | None] = ContextVar("tenant_id", default=None)

def get_current_tenant_id() -> str | None:
    return _tenant_id.get()

def set_current_tenant_id(tid: str) -> None:
    _tenant_id.set(tid)
```

### 2.4 Middleware de Tenant

#### [NEW] `app/core/middleware/tenant.py`

```python
class TenantMiddleware(BaseHTTPMiddleware):
    """
    Extrai tenant_id do JWT (claim) e seta no ContextVar.
    Rotas públicas (login, register, health) fazem bypass.
    """
    async def dispatch(self, request, call_next):
        # Extrai do token JWT que já foi decodificado
        # Seta via set_current_tenant_id()
        ...
```

### 2.5 Filtro Automático no Repository Base

#### [NEW] `app/core/repository.py` — Base Repository com filtro por tenant

```python
class TenantAwareRepository:
    """
    Base repository que injeta automaticamente WHERE tenant_id = :tid
    em todas as queries e seta tenant_id no INSERT.
    """
    def __init__(self, db: AsyncSession, model: type):
        self.db = db
        self.model = model
    
    def _base_query(self):
        tid = get_current_tenant_id()
        if tid is None:
            raise HTTPException(403, "Tenant não identificado")
        return select(self.model).where(self.model.tenant_id == tid)
    
    async def create(self, instance):
        instance.tenant_id = get_current_tenant_id()
        self.db.add(instance)
        ...
```

### 2.6 Onboarding de Tenant

#### [NEW] `app/modules/platform/tenant/service.py`

```
POST /api/v1/tenants                    → Criar tenant (SUPERADMIN)
GET  /api/v1/tenants                    → Listar tenants (SUPERADMIN)
POST /api/v1/tenants/{id}/invite-user   → Convidar usuário ao tenant (ADMIN+)
```

### 2.7 Migration Alembic

Uma migration única que:
1. Cria tabela `tenants`
2. Adiciona `tenant_id` em todas as tabelas listadas
3. Cria tenant default para dados existentes (Este será o primeiro tenant, pertencente ao provedor da plataforma)
4. Popula `tenant_id` nos registros existentes com o tenant default
5. Aplica constraint `NOT NULL` após população
6. Cria índices compostos `(tenant_id, id)` para performance

---

## 🔒 FASE 3 — Segurança de Rotas e Hardening

> **Princípio**: Garantir validação de identidade e idempotência em **100% das rotas**, adicionar headers de segurança e preparar para proxy reverso.

### 3.1 Auditoria Completa de Rotas

Vistoriar cada rota e garantir:

| Verificação | Ação |
|-------------|------|
| **Auth** | Toda rota tem `Depends(require_permission(...))` — nenhuma rota aberta exceto `/health`, `/auth/login`, `/auth/register` (temporariamente invite-only, futuramente redirecionará para compra de plano), `/auth/password-reset/*` |
| **Idempotência** | Todo endpoint mutante (POST/PUT/PATCH/DELETE) tem `@idempotence(...)` |
| **Rate Limiting** | Rate limit por `(tenant_id + user_id)` via Redis em rotas sensíveis |
| **Input Validation** | Todos os payloads passam por Pydantic com `Field(max_length=...)` |

### 3.2 Security Headers Middleware

#### [NEW] `app/core/middleware/security_headers.py`

```python
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
        return response
```

### 3.3 CORS Hardening

#### [MODIFY] `app/main.py`

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,  # ← lista explícita, sem "*"
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Correlation-Id", "X-Session-Id"],
)
```

### 3.4 Rate Limiter Global

#### [NEW] `app/core/middleware/rate_limiter.py`

Rate limiter baseado em sliding window via Redis:

```
- Rotas de auth: 10 req/min por IP
- Rotas de API: 100 req/min por (tenant_id + user_id)
- Webhook inbound: 300 req/min por API Key
```

### 3.5 Refresh Token

#### [MODIFY] Fluxo de autenticação

Implementar refresh token com rotação:
- Access token: 15 min (reduzir de 60 min atual)
- Refresh token: 7 dias (cookie HttpOnly, Secure, SameSite=Lax)
- Endpoint `POST /auth/refresh` para renovação

---

## 🛡️ FASE 4 — Governança de Dados e LGPD

> **Princípio**: Garantir que a plataforma esteja em conformidade com as leis de proteção de dados, gerenciando permissões de uso, anonimização e o direito ao esquecimento.

### 4.1 Validação de Consentimento

#### [NEW] `app/modules/platform/privacy/models.py`

Tabela para registrar os opt-ins e permissões do usuário/lead:
```python
class DataConsent(Base):
    __tablename__ = "data_consents"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=True)
    lead_id = Column(Integer, ForeignKey("leads.id"), nullable=True)
    
    marketing_opt_in = Column(Boolean, default=False)
    analytics_opt_in = Column(Boolean, default=False)
    data_sharing_opt_in = Column(Boolean, default=False)
    
    ip_address = Column(String(50), nullable=True)
    user_agent = Column(String, nullable=True)
    
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### 4.2 Anonimização de PII (Personally Identifiable Information)

#### [NEW] `app/core/privacy/anonymizer.py`

Utilitário centralizado para mascarar dados sensíveis:
- Mascaramento de e-mails (`e***@dominio.com`)
- Mascaramento de CPFs/CNPJs (`***.123.456-**`)
- Mascaramento de telefones (`+55 11 9****-1234`)

A ser aplicado em logs, painéis de auditoria visuais e exportações de relatórios.

### 4.3 Direito ao Esquecimento (Data Erasure)

#### [NEW] Event Listener para Remoção de Dados

Usar o barramento de eventos (Metrô) para orquestrar o esquecimento. Quando acionado o evento `PRIVACY.DATA_ERASURE_REQUESTED`:
1. **Soft Delete / Hard Delete**: Apagar referências diretas (Lead, User).
2. **Data Masking**: Em registros inapagáveis (ex: Faturas financeiras por exigência fiscal), substituir os dados sensíveis por `[ANONIMIZADO]` ou usar o utilitário do passo 4.2, mantendo os valores monetários intactos para os relatórios.
3. Gravar na Auditoria (Jornal Oficial) a execução da solicitação.

---

## 🌐 FASE 5 — Infraestrutura NGINX + Cloudflare Tunnel

> **Princípio**: Adicionar NGINX como proxy reverso na frente do Uvicorn para SSL termination, IP whitelisting, e preparar Cloudflare Tunnel para exposição segura.

### 4.1 NGINX como Proxy Reverso

#### [NEW] `docker/nginx/nginx.conf`

```nginx
upstream api_backend {
    server api:8000;
}

server {
    listen 443 ssl;
    server_name _;
    
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # IP Whitelist (Cloudflare ranges + rede interna)
    # include /etc/nginx/conf.d/cloudflare-ips.conf;
    # deny all;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=30r/s;
    
    # Security headers (backup do middleware)
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
    
    # Proxy para FastAPI
    location /api/ {
        limit_req zone=api_limit burst=50 nodelay;
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Health check sem rate limit
    location /health {
        proxy_pass http://api_backend;
    }
}

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

### 4.2 Docker Compose — Serviço NGINX

#### [MODIFY] `docker-compose.yml`

```yaml
  nginx:
    image: nginx:1.27-alpine
    container_name: crm_nginx
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - api
    networks:
      - crm_net
```

Remover `ports: - "8000:8000"` do serviço `api` (acesso somente via NGINX):
```yaml
  api:
    ...
    # ports:    ← removido, acesso interno apenas
    expose:
      - "8000"
```

### 4.3 Cloudflare Tunnel

O tunnel será criado via portal da Cloudflare para simplificar a gestão. Apenas o token do tunnel será passado via variável de ambiente. O foco imediato é garantir que o NGINX esteja 100% funcional para rotear corretamente quando o tunnel for ativado no portal.

#### [NEW] Serviço no Docker Compose

```yaml
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: crm_cloudflared
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    command: tunnel run
    depends_on:
      - nginx
    networks:
      - crm_net
```

### 4.4 IP Whitelisting (Cloudflare)

#### [NEW] `docker/nginx/conf.d/cloudflare-ips.conf`

Script de atualização periódica dos IPs do Cloudflare:
```bash
# Busca ranges IPv4/IPv6 oficiais do Cloudflare
# Gera arquivo .conf com `allow X.X.X.X/XX;` para cada range
# + allow da rede Docker interna (172.16.0.0/12)
# deny all;
```

### 4.5 Trusted Proxy no FastAPI

#### [MODIFY] `app/main.py`

```python
# Configurar trusted proxies para X-Forwarded-For
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware
app.add_middleware(ProxyHeadersMiddleware, trusted_hosts=["nginx", "127.0.0.1"])
```

---

## 📋 Ordem de Execução (Cronograma)

```
FASE 1 — RBAC
├── 1.1 Criar permissions.py (enum de roles/permissions)
├── 1.2 Criar dependency require_permission()
├── 1.3 Migration: constraint no campo role + depreciar is_superuser
├── 1.4 Aplicar auth em TODAS as rotas desprotegidas
├── 1.5 Incluir role no JWT payload
├── 1.6 Endpoints de gestão de usuários
└── ✅ CHECKPOINT: Testar todas as rotas com diferentes roles

FASE 2 — Multi-Tenant
├── 2.1 Criar model Tenant
├── 2.2 Adicionar tenant_id em User
├── 2.3 Criar TenantMixin
├── 2.4 Migration: adicionar tenant_id em todas as tabelas
├── 2.5 Criar TenantMiddleware (extract do JWT)
├── 2.6 Criar TenantAwareRepository (filtro automático)
├── 2.7 Refatorar repositories existentes para herdar do TenantAwareRepository
├── 2.8 Endpoints de onboarding de tenant
└── ✅ CHECKPOINT: Testar isolamento entre tenants

FASE 3 — Segurança de Rotas
├── 3.1 Auditoria completa de rotas (checklist)
├── 3.2 SecurityHeadersMiddleware
├── 3.3 CORS hardening (substituir wildcards)
├── 3.4 Rate Limiter global via Redis
├── 3.5 Refresh token com rotação
├── 3.6 Idempotência em rotas mutantes faltantes
└── ✅ CHECKPOINT: Pen-test básico (checklist OWASP Top 10)

FASE 4 — Governança de Dados e LGPD
├── 4.1 Criar model DataConsent
├── 4.2 Utilitário de anonimização (anonymizer.py)
├── 4.3 Listener de evento PRIVACY.DATA_ERASURE_REQUESTED
├── 4.4 Rotina de mascaramento e soft delete
└── ✅ CHECKPOINT: Simular evento de esquecimento e validar banco

FASE 5 — NGINX + Cloudflare
├── 5.1 nginx.conf + Dockerfile
├── 5.2 Atualizar docker-compose.yml
├── 5.3 Configurar serviço do Cloudflared no compose
├── 5.4 IP Whitelisting (Cloudflare ranges)
├── 5.5 ProxyHeaders no FastAPI
└── ✅ CHECKPOINT: Deploy em ambiente staging
```

---

## 🔍 Verificação e Testes

### Testes Automatizados
- **RBAC**: Testes unitários para cada role × permissão (matrix test)
- **Multi-Tenant**: Testes de isolamento — usuário de Tenant A nunca vê dados do Tenant B
- **Idempotência**: Teste de replay — mesma request 2x retorna resultado cacheado
- **Rate Limit**: Teste de burst — 101ª request retorna 429

### Testes Manuais
- Criar 2 tenants, 2 usuários (um em cada tenant), verificar isolamento via Postman
- Acessar rota sem token → 401
- Acessar rota com token mas sem permissão → 403
- Acessar via IP fora da whitelist → NGINX 403

### Segurança
- Executar `security scanner` nas rotas modificadas
- Validar headers de segurança via [securityheaders.com](https://securityheaders.com)
- TODO(security): Implementar MFA para roles ADMIN+ em fase futura
- TODO(security): Implementar detecção de senhas vazadas em fase futura

### 🧪 Security First Guardrail (Automations)
Testes de conformidade arquitetural e hooks para rodar no CI e garantir que as novas regras nunca sejam violadas:
- **Auth Guard**: Rota exposta no FastAPI sem `Depends(require_permission(...))` (ou sem declaração explícita de rota pública) → **Falha no CI**
- **Tenant Guard**: Model de dados criado sem a coluna `tenant_id` (ou sem uso do `TenantMixin`) → **Bloqueio de merge**
- **Idempotency Guard**: Endpoint mutável (POST, PUT, PATCH, DELETE) declarado sem o decorator `@idempotence` → **Warning automático**

---

## ⚠️ Riscos e Mitigações

| Risco | Mitigação |
|-------|-----------|
| Migration `tenant_id NOT NULL` quebra dados existentes | Criar tenant default, popular antes de aplicar NOT NULL |
| Performance de filtro por tenant em queries complexas | Índices compostos `(tenant_id, ...)` em todas as tabelas |
| Token JWT cresce com claims extras | Manter claims mínimos: `sub`, `email`, `role`, `tenant_id` |
| NGINX como SPOF | Docker healthcheck + restart policy `unless-stopped` |
| Cloudflare Tunnel instável | Fallback via IP público + firewall rules manuais |

---

## 📁 Arquivos Novos (Resumo)

```
app/modules/platform/auth/permissions.py        ← Enums RBAC
app/modules/platform/tenant/                    ← Módulo Tenant (models, service, routes, schemas)
app/core/mixins/tenant.py                       ← TenantMixin para models
app/core/tenant_context.py                      ← ContextVar de tenant
app/core/middleware/tenant.py                    ← TenantMiddleware
app/core/middleware/security_headers.py          ← Security Headers
app/core/middleware/rate_limiter.py              ← Rate Limiter global
app/core/repository.py                          ← TenantAwareRepository base
docker/nginx/nginx.conf                         ← NGINX config
docker/nginx/conf.d/cloudflare-ips.conf         ← IP Whitelist
docker/cloudflare/config.yml                    ← Tunnel config
migrations/versions/xxxx_rbac_consolidation.py  ← Migration RBAC
migrations/versions/xxxx_multi_tenant.py        ← Migration Multi-Tenant
```

## 📁 Arquivos Modificados (Resumo)

```
app/modules/platform/auth/dependencies.py  ← require_permission(), require_role()
app/modules/platform/auth/models.py        ← +tenant_id no User
app/modules/platform/auth/schemas.py       ← +role enum, +tenant_id
app/core/security.py                       ← JWT claims com role/tenant_id
app/main.py                                ← +SecurityHeaders, +TenantMiddleware, CORS hardening
docker-compose.yml                         ← +nginx, +cloudflared, api expose only

# Todas as rotas de cada módulo → adicionar require_permission()
app/modules/platform/rules/routes.py
app/modules/platform/events/routes.py
app/modules/platform/actions/routes.py
app/modules/erp/pedidos/routes.py
app/modules/erp/pagamentos/routes.py
app/modules/erp/produtos/routes.py
app/modules/erp/vendas/routes.py
app/modules/erp/clientes/routes.py
app/modules/erp/estoque/routes.py
app/modules/erp/fornecedores/routes.py
app/modules/erp/relatorios/routes.py
app/modules/crm/campanhas/routes.py
app/modules/crm/leads/routes.py
app/modules/crm/comunicacao/routes.py
app/modules/crm/chatbot/api/studio.py
app/modules/crm/chatbot/api/session_routes.py
app/modules/crm/chatbot/api/metrics_routes.py
```
