

# CONTEXTUALIZANDO O PROJETO

Hoje o projeto está tomando uma proporção maior do que a organização atual permite gerenciar. Temos 19 módulos registrados, mas que de forma indireta formam alguns produtos do sistema e este o ponto que quero trabalhar: organizar os módulos em domínios explícitos.

## OBJETIVOS
Fazer uma separação clara e objetiva dos módulos em domínios, para facilitar a manutenção, escalabilidade, teste e organização do código. Além de facilitar a gestão do domínio para o usuário e futuramento a atuação dos bots autômatos dentro dos domínios.

## QUAIS SÃO OS DOMÍNIOS?
Hoje conseguimos traçar alguns cortes neste projeto que está em nivel de plataformma em serviços e domínios. Considero serviços todas as partes do sistema em que podem ser separados em containers, ou seja: 
    -integração do whatsapp, 
    -chatbot studio,
    -api-backend
No momento são estes, e considero domínios todas as partes do sistema que podem ser agrupados de forma que cada grupo consiga exercer controle e autônomia sobre seus próprios recursos, hoje, todos os módulos tem a sua importância e idepotencia dentro do projeto, mas organizando melhor teríamos:
    - ERP: (Financeiro, Estoque, Pagamentos, Fornecedores, Produtos, Clientes, Vendas)
    - CRM: (leads, comunicacao,chatbot, campanhas)
    - Plataforma: (rules, integrations, notifications, events, congnitive,actions)
Assim, fica delimitado dentro dos domínios, as suas funcionalidades domínios, facilitando o entendimento via conceito quanto pela organização dos modulos.

## ESTRUTURA DO SISTEMA
Estamos caminhando para uma estrutura de alto padrão de complexidade e como eu estou trabalhando sozinho, fica mais facil para mim separando a complexidade maior em mini complexidades de forma que cada uma não atravesse a outra. Ou seja, se temos uma parte que cuidad do estoque, tudo relacionado a estocagem, ou seja, registro de movimentações, controle de estoque, cadastro de produtos, controle de fornecedores, etc, estejamm tudo dentro de deste guarda chuva, mas cruzando esta necessidade com a realidade do projeto, vemos que temos módulos ultra especialistas e que um módulo é dependente do outro e isto está certo, mas quero colocar um guarda chuva maior para organizar os módulos em seus domínios, facilitando o entendimento do sistema, tempo de manutenção e divisão de contexto.
então teríamos uma estrutura assim:

PLATAFORMA:
|- INFRAESTRUTURA
|   |- BANCO DE DADOS
|   |   |- POSTGRES
|   |- CACHE
|   |   |- REDIS
|   |- AUTENTICAÇÃO
|   |   |- JWT
|   |   |- OTP
|   |   |- MFA
|   |   |- RBAC
|   |- CONFIGURAÇÃO DE AMBIENTES
|   |   |- ENV
|   |   |- secrets
|   |   |- CONFIG
|   |   |- Plataform Rules
|- SERVIÇOS
|   |   |- EVENTOS
|   |   |- WORKERS
|   |   |- GOVERNANÇA
|   |   |- MONITORAMENTO
|   |   |- NOTIFICAÇÕES
|   |   |- INTEGRAÇÕES
|   |   |   |- COMUNICACAO
|   |   |   |   |- Integration (whatsapp)
|   |   |   |   |- Integration (telegram)
|   |   |   |   |- Integration (facebook)
|   |   |   |   |- Integration (instagram)
|   |   |   |   |- Integration (Email)
|   |   |   |- BANCOS
|   |   |   |   |- Inter
|   |   |   |   |- Mercado Pago
|   |   |   |- Transportadoras
|   |   |   |   |- Correios
|   |   |   |   |- Jadlog
|- CORE
|   |- CAPACIDADE DE DECISÃO (RULE ENGINE)
|   |- CAPACIDADE COGNITIVA (CONGITIVE ENGINE)
|   |   |- NLP
|   |   |- LLM
|   |   |- Speech-To-Text
|   |   |- Text-To-Speech
|   |   |- Computer Vision
|   |   |- Orchestration
|   |   |- HYDRATE
|   |- CAPACIDADE EXECUTORA (ACTION ENGINE)
|- AMBIENTES
|   |- COMMUNICATIONS
|   |- Chatbot-Studio (container)
|   |   |- DOMAIN
|   |   |   |- Understanding
|   |   |   |   |- intents
|   |   |   |   |- contexts
|   |   |   |   |- entities
|   |   |   |   |- flows
|   |   |   |- Capabilities
|   |   |   |   |- actions
|   |   |   |   |- verifications
|   |   |   |   |- validations
|   |   |   |- Channels
|   |   |   |- Orchestration
|   |   |   |   |-pipeline
|   |   |   |   |-States
|   |   |   |   |-strategy
|   |   |   |- bots
|   |- BACKEND (container)
|   |   |- DOMAIN
|   |   |   |- ERP
|   |   |   |   |- Financeiro
|   |   |   |   |   |- AdapterBanco - pagamentos (integracao_banco)
|   |   |   |   |   |- Contas a pagar
|   |   |   |   |   |- Contas a receber
|   |   |   |   |   |- Regras do financeiro
|   |   |   |   |- Estoque
|   |   |   |   |   |- Produtos
|   |   |   |   |   |- Auditoria (todas as movimentacoes de estoque)
|   |   |   |   |   |- Regras do estoque
|   |   |   |   |- Logistica
|   |   |   |   |   |- Rastreio
|   |   |   |   |   |- Rotas
|   |   |   |   |   |- Entrega
|   |   |   |   |   |- Devolucoes
|   |   |   |   |   |- Regras da logistica
|   |   |   |   |- Vendas
|   |   |   |   |   |- Pedidos
|   |   |   |   |   |- Faturas
|   |   |   |   |   |- Clientes
|   |   |   |   |   |- Regras das vendas
|   |   |   |   |- Relacionamentos
|   |   |   |   |   |- Fornecedores
|   |   |   |   |   |- Clientes
|   |   |   |   |   |- Transportadores
|   |   |   |   |   |- Regras de relacionamento
|   |   |   |- CRM
|   |   |   |   |- Leads
|   |   |   |   |   |- Contatos
|   |   |   |   |   |- Funil de vendas
|   |   |   |   |   |- Score
|   |   |   |   |   |- Regras dos leads
|   |   |   |   |- Chatbot
|   |   |   |   |   |- Bots
|   |   |   |   |   |- Regras dos chatbots
|   |   |   |   |- Campanhas
|   |   |   |   |   |- Campanhas
|   |   |   |   |   |- Regras das campanhas
|   |   |   |   |- Atendimento
|   |   |   |   |   |- Atendimentos
|   |   |   |   |   |- Regras de atendimento
|   |- FRONTEND


## A ANALOGIA GUIA DA ARQUITETURA

## O País

O país representa a **Plataforma inteira**.

Ela fornece estrutura, leis fundamentais, infraestrutura e serviços compartilhados para que todos os estados possam operar.

```text
PAÍS
    = Plataforma
```

Responsabilidades:

```text
- Infraestrutura
- Segurança
- Comunicação
- Governança
- Execução
- Inteligência
- Auditoria
- Integrações
```

Ela não administra cidades diretamente.

Ela cria as condições para que funcionem.

---

# Constituição do país

As regras pétreas representam princípios absolutos definidos pelo dono.

São regras que não podem ser violadas:

```text
- nunca vender sem autorização
- nunca executar ação sem permissão
- nunca ignorar autenticação
- nunca quebrar políticas financeiras
- nunca agir fora do escopo
```

Na arquitetura:

```text
Constituição
    = Regras pétreas
```

---

# Dono do país

O dono representa a autoridade máxima.

```text
DONO
```

Função:

```text
define visão
define princípios
define limites
define políticas
```

Não executa operações diárias.

Não resolve estoque.

Não responde atendimento.

Define:

```text
"como o país funciona"
```

---

# Diplomata Executivo

O controller virou uma peça muito elegante.

Não é ditador.

Não manda em estados.

Não cria leis.

Ele atua somente quando:

```text
- existe ambiguidade
- existe conflito
- faltou contexto
- surgiu exceção
```

```text
CONTROLLER
    = Diplomata Executivo
```

Exemplo:

```text
Financeiro:
    recusou

CRM:
    cliente VIP

Logística:
    aprovou
```

Diplomata:

```text
Existe política para isso?
```

Se existir:

```text
segue
```

Se não:

```text
encaminha
```

---

# Estados

Estados representam domínios.

```text
ERP
CRM
CHATBOT
LOGÍSTICA
```

Cada estado possui:

```text
- autonomia
- regras locais
- responsabilidades próprias
```

Princípio:

```text
Nenhum estado manda no outro
```

---

# Cidades

As cidades representam módulos.

Exemplo:

Estado ERP:

```text
Financeiro
Estoque
Vendas
Relacionamentos
Logística
```

Estado CRM:

```text
Leads
Campanhas
Atendimento
Chatbot
```

Cada cidade resolve seus próprios problemas.

---

# Bairros

Representam funcionalidades internas.

Exemplo:

```text
Estoque
    Produtos
    Auditoria
    Movimentações
```

ou:

```text
Leads
    Score
    Funil
    Contatos
```

---

# Habitantes

Os habitantes representam:

```text
Usuários
Bots
Agentes
Workers
```

---

# Agentes públicos

Os agentes não governam.

Eles representam.

Exemplo:

```text
Agente Financeiro
Agente Estoque
Agente CRM
```

Eles são:

```text
porta-vozes do estado
```

Internamente usam:

```text
regras
contexto
permissões
memória
```

Eles não inventam política.

---

# Leis locais

Cada cidade possui leis.

Na plataforma:

```text
RULE ENGINE
```

Função:

```text
SE condição
ENTÃO ação
```

Exemplo:

```text
SE estoque <= 0
ENTÃO bloquear venda
```

---

As regras existem em níveis:

```text
Módulo
↓

Domínio
↓

Plataforma
```

---

# Ônibus

O ônibus representa o Rule Engine em execução.

Fluxos previsíveis.

Rotas conhecidas.

Paradas definidas.

Funciona internamente ao estado.

```text
Pedido

↓

valida estoque

↓

valida financeiro

↓

aprova
```

---

# Metrô

Metrô representa eventos.

Liga estados diferentes.

Assíncrono.

Rápido.

Não dá ordens.

Comunica fatos.

```text
PedidoPago
```

Estados escutam:

```text
Estoque
Financeiro
CRM
Logística
```

---

# Fatos e não ordens

Errado:

```text
EstoqueExecuteReserva
```

Certo:

```text
PedidoPago
```

Evento informa:

```text
algo aconteceu
```

Quem decide agir:

```text
o estado
```

---

# Polícia e órgãos executores

Representam:

```text
ACTION ENGINE
```

Executam:

```text
enviar email
emitir cobrança
gerar pedido
chamar APIs
```

Não tomam decisões.

Só executam.

---

# Inteligência nacional

Representa:

```text
COGNITIVE ENGINE
```

Composto por:

```text
NLP
LLM
Speech-to-text
Text-to-speech
Vision
Hydrate
Orchestration
```

Função:

```text
entender
interpretar
traduzir
contextualizar
```

Sem regras:

```text
vira caos
```

---

# Serviços públicos

Infraestrutura compartilhada:

```text
Banco
Cache
Autenticação
Monitoramento
Eventos
Notificações
Integrações
```

Equivalente:

```text
energia
água
rodovias
polícia
correios
```

Estados usam.

Não precisam reconstruir.

---

# Embaixadas e relações exteriores

Integrações:

```text
WhatsApp
Telegram
Correios
Inter
Mercado Pago
```

São conexões com outros países.

---

# Jornal Oficial

Representa:

```text
Auditoria
Logs
Event Store
```

Registra:

```text
o que aconteceu
quem fez
quando fez
por que fez
```

---

# Casos diplomáticos especiais

Quando um estado recusa:

```text
Financeiro:
    recusado
```

Mas o diplomata permite:

```text
Controller:
    aprovado
```

Vai para:

```text
Registro Nacional de Exceções
```

Exemplo:

```text
Domínio:
Financeiro

Recusa:
margem insuficiente

Controller:
autorizado

Motivo:
cliente VIP

Regra:
EXC-004
```

---

Isso permite:

```text
auditoria total
aprendizado
melhoria de regras
descoberta de padrões
```

---

# Princípios constitucionais do país

```text
1 — cada cidade possui suas regras

2 — cada estado governa seus recursos

3 — nenhum estado governa outro

4 — estados se comunicam por fatos

5 — regras locais têm prioridade

6 — conflitos são diplomáticos

7 — exceções são auditadas

8 — agentes representam regras

9 — inteligência não governa

10 — o dono define princípios
```

