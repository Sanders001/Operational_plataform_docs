# Sanders Studio — Arquitetura de Plataforma (DDD em Escala)

O objetivo final desta arquitetura não é apenas organizar arquivos, mas preparar o front-end para se tornar uma **Plataforma de Automação Corporativa**, hospedando múltiplos estúdios de nível de produto (Workflow e Chatbot) sem colisão, sem acoplamento e sem degradação de performance.

## Proposed Changes

### 1. Macro-Arquitetura (Os 3 Pilares)

A aplicação deixa de ter um domínio único e passa a ser regida por uma divisão superior de "Bounded Contexts":

```text
src/
├── layouts/       (UI Global da IDE: AppLayout, Sidebar, StatusBar)
├── config/        (Arquivos de menu e chaves base)
├── modules/
│   ├── shared-kernel/    (Base compartilhada)
│   ├── workflow-studio/  (Plataforma de Runtime)
│   └── chatbot-studio/   (Plataforma de AI)
└── App.tsx        (Ponto de entrada puro)
```

### 2. O Shared Kernel (Coração da Plataforma)
Tudo que é usado por ambos os Studios viverá aqui, matando a duplicação:
- **`modules/shared-kernel/`**: `components/` (Selectores, botões de IDE), `hooks/` (Websockets, Auth), `events/` (Global IDE events), `services/` (Métricas genéricas), `models/`.

### 3. O Workflow Studio (Runtime Platform)
Com toda a dignidade arquitetural discutida anteriormente:
- `providers/WorkflowProvider.tsx` (Envelopando ReactFlow e ErrorBoundary).
- `features/editor/`, `features/trace/`, `features/deadletter/`, `features/choreography/`, `features/observability/`.
- Fluxo estrito: `API → Service → Adapter → Store (Zustand) → Selector Hook → UI`.
- `engines/` e `events/` fortemente tipados (`ADD_NODE`, etc) com micro-stores.

### 4. O Chatbot Studio (AI Platform)
Aplicaremos a mesmíssima estrutura profissional ao Chatbot Studio, que crescerá exponencialmente devido à IA:

```text
modules/chatbot-studio/
    ├── providers/
    │    └── ChatbotProvider.tsx
    ├── features/
    │    ├── playground/    (Chat UI, Mock user, Conversas Multi-Session)
    │    ├── knowledge/     (FAISS, embeddings, RAG configs)
    │    ├── intents/       (Intent Editor, Confidence, Mapping)
    │    ├── training/      (Histórico de learning, Datasets, Jobs)
    │    ├── observability/ (Latência, Fallbacks, Cache hit, Tokens)
    │    └── runtime/       (Workers, Dispatch queues)
    ├── services/
    ├── adapters/
    ├── dto/
    ├── models/
    ├── stores/            (Micro-stores com seletores hook)
    ├── reducers/
    ├── engines/           (intentEngine, routingEngine, contextEngine, learningEngine)
    ├── events/            (Event-driven Catalog)
    └── constants/
```

#### Eventos e Motores de IA
Como o backend inteiro é orientado a evento, o front-end de Chatbot não pode ser procedural.
- **`events/chatbot.events.ts`**:
```typescript
type ChatbotEvent =
 | { type: "MESSAGE_RECEIVED", payload: MessagePayload }
 | { type: "INTENT_RESOLVED", payload: IntentData }
 | { type: "FALLBACK_TRIGGERED", context: FallbackContext }
 | { type: "LEARNING_COMPLETED", payload: JobStats }
 | { type: "BOT_SWITCHED", botId: string }
```
- **`engines/`**: Aqui vivem as lógicas que já saíram da alçada visual (`contextEngine.ts` para agrupar as entidades RAG do usuário, `routingEngine.ts` para guiar a lógica de fallbacks antes do render).

### 5. O Padrão Ouro de Performance e Engenharia
- **Fluxo Uni-Direcional**: Nenhuma das partes UI acessará APIs diretamente. Tudo passa pelo encanamento de `Adapters` e morre na Store.
- **Error Boundaries Isolados**: Um erro no `learningEngine` não derrubará o `playground`.
- **Zustand Micro-Stores e Selectors**: Hooks pré-definidos bloqueando re-renders catastróficos no Canvas do Workflow ou no feed do Chatbot.

## User Review Required
> [!IMPORTANT]
> - Este é o estado da arte do frontend. A transição começará pela **infraestrutura global** (`shared-kernel`, Layouts, Zustand setup), depois focaremos no **Workflow Studio** e depois no **Chatbot Studio**, para garantir estabilidade contínua na IDE.
> - `zustand` será instalado antes da montagem dos stores.

## Verification Plan
1. Analisar as dependências do `package.json` para adicionar `zustand`.
2. Refatorar a raiz (Criar `layouts/`, `modules/shared-kernel/`, limpar o `App.tsx`).
3. Estruturar a árvore de diretórios do **Workflow Studio** e migrar os componentes como acordado.
4. Estruturar a árvore de diretórios do **Chatbot Studio** e planejar a migração de suas lógicas atuais para as `features` isoladas.
5. Em todas as etapas, forçar a ausência de `any`, isolar tipagens via `dto`/`adapters` e centralizar os Providers.
6. Rodar os builds locais garantindo que a separação de domínios não produziu loops de estado nem reduziu o framerate da IDE gráfica.
