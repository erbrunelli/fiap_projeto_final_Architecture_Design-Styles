# AntecipaJá — Antecipação de Recebíveis para Prestadores de Serviço Autônomos

**Disciplina:** IT Architecture Design & Styles — MBA Arquitetura de Soluções, FIAP
**Professor:** Leonardo Pinho
**Tema:** Antecipação de Recebíveis
**Integrantes:**  Catharina Moral Hermacula RM367112 · Daniele Aguiar Ramalho RM364853 · Erika Regina Brunelli RM369277 · Nauana Kelly Lima Nonato RM367585 · Shirlei Alexandrino dos Anjos RM369103

---

## 1. Story Telling

### 1.1 O problema

Motoristas de aplicativo, entregadores, prestadores de serviço via plataformas (GetNinjas, faxineiras e diaristas cadastradas, profissionais liberais que emitem nota avulsa) vivem uma contradição: **eles já trabalharam e já têm o direito de receber, mas o dinheiro não está na conta.** As plataformas repassam em D+7, D+15, D+30. Nesse intervalo, o prestador precisa pagar combustível, aluguel do equipamento, insumos, mercado — e não tem histórico bancário robusto nem CNPJ formal na maioria das vezes, o que o exclui do crédito tradicional bancário.

O **AntecipaJá** é uma plataforma que se integra às plataformas parceiras (via API/convênio), identifica os valores já "ganhos mas não recebidos" por um prestador, calcula um risco com base em dados transacionais (não em score bancário tradicional) e antecipa o valor, mediante uma taxa de deságio, via Pix, em minutos.

### 1.2 O que esperamos aprender com esse projeto

- Como desenhar uma arquitetura de solução fintech que integra múltiplos sistemas externos (as plataformas parceiras).
- Como aplicar o C4 Model em um cenário real de mercado, com granularidade crescente (Contexto → Container → Componente).
- Como equilibrar velocidade de decisão de crédito com gestão de risco e fraude.
- Como uma decisão de arquitetura (ex: onde fica a regra de score) se conecta a uma decisão de negócio (ex: apetite de risco).

### 1.3 Perguntas que precisamos responder

1. Como validar que o recebível a antecipar é real e ainda não foi antecipado em outro lugar (evitar duplicidade/fraude)?
2. Como integrar com plataformas parceiras que têm APIs, formatos e SLAs diferentes, sem acoplar o núcleo do produto a cada uma delas?
3. Qual o modelo de precificação de risco (taxa de deságio) por perfil de prestador e por parceiro?
4. Como garantir compliance regulatório — quem, juridicamente, está fazendo a operação de crédito (o AntecipaJá precisa de uma SCD/parceria com instituição financeira licenciada)?
5. Como desenhar um onboarding rápido para autônomos que, em geral, não têm histórico bancário robusto?

### 1.4 Principais riscos

| Risco | Descrição |
|---|---|
| **Crédito/inadimplência** | O recebível antecipado não se concretiza (parceiro não repassa, prestador é desligado da plataforma antes do repasse). |
| **Fraude** | Recebíveis falsos, duplicados ou manipulados para simular volume maior do que o real. |
| **Regulatório** | Operar antecipação de recebível é operação de crédito — exige estrutura regulada (SCD) ou parceria com instituição financeira autorizada pelo Bacen. |
| **Dependência de integração** | O produto depende de APIs de terceiros (plataformas parceiras) que podem mudar contrato, cair ou revogar acesso. |
| **Concentração** | Poucos parceiros grandes concentrando a maior parte do volume antecipado. |

### 1.5 Plano para aprender o que precisamos

- Rodar um piloto (MVP) com 1–2 plataformas parceiras antes de escalar.
- Construir o motor de score com dados alternativos (histórico de prestação, avaliações, recorrência) + consulta a bureau de crédito (Serasa/SPC) como camada complementar.
- Envolver jurídico/compliance desde o desenho da arquitetura, não depois — decidir logo se o AntecipaJá será uma SCD própria ou operará via parceria com instituição financeira licenciada (fronting).
- Medir a taxa de fraude e inadimplência do piloto antes de abrir novos parceiros.

### 1.6 Plano para reduzir riscos

- Arquitetura de integração com **circuit breaker** e fallback para indisponibilidade de parceiros (não travar o produto inteiro se um parceiro cair).
- Diversificação gradual de parceiros — nenhum parceiro deve concentrar mais que um limite definido do volume total antecipado.
- Auditoria e rastreabilidade de ponta a ponta (todo recebível antecipado tem trilha completa, exigência regulatória).
- Limites de exposição por prestador e por parceiro, com trava automática no motor de risco.

### 1.7 Quem são as partes interessadas (stakeholders)

| Stakeholder | O que espera ganhar |
|---|---|
| Plataformas parceiras (apps de serviço/marketplaces) | Retenção e satisfação dos prestadores, diferencial competitivo, sem assumir o risco de crédito. |
| Instituição financeira parceira / fundo de recebíveis | Retorno financeiro sobre o capital que financia as antecipações. |
| Time interno (produto, risco, compliance, engenharia) | Produto sustentável, saudável e dentro do apetite de risco definido. |
| Reguladores (Bacen) | Operação de crédito transparente, auditável e dentro do arcabouço regulatório. |

### 1.8 Quem são os usuários

Prestadores de serviço autônomos que já emitiram uma nota ou já concluíram um serviço através de uma plataforma parceira, e têm um valor a receber ainda não repassado.

### 1.9 O que eles estão tentando realizar

Transformar um valor que já é seu por direito (mas ainda não está na conta) em dinheiro disponível **hoje**, para cobrir despesas do dia a dia, sem burocracia de banco tradicional e sem esperar o prazo de repasse da plataforma.

### 1.10 Qual o pior cenário possível

Uma onda de fraude coordenada (recebíveis falsificados em massa) gera um prejuízo financeiro relevante **e**, simultaneamente, chama atenção do regulador para uma operação de crédito não devidamente licenciada — resultando em prejuízo financeiro somado a encerramento forçado da operação.

---

## 2. Arquitetura Freeform (versão inicial)

> Diagrama: [`diagramas/01-arquitetura-freeform.drawio`](diagramas/imagens/01-arquitetura-freeform.jpg)

### 2.1 Componentes

| Componente | Descrição |
|---|---|
| **App/Portal do Prestador** | Interface web/mobile onde o prestador acompanha o valor disponível para antecipação, simula a taxa e confirma a antecipação. |
| **API Gateway** | Porta de entrada única, aplica autenticação, rate limiting e roteamento para os serviços internos. |
| **Serviço de Integração com Parceiros** | Camada de conectores (um adaptador por parceiro) que traduz cada API de plataforma parceira para um modelo de dados único e interno de "recebível". |
| **Serviço de Onboarding/KYC** | Cadastro e validação de identidade do prestador (documento, selfie, dados bancários). |
| **Motor de Score e Risco** | Calcula o risco do recebível e do prestador combinando dados internos (histórico de antecipações, recorrência) com dados externos (bureau de crédito). |
| **Serviço de Antecipação** | Orquestra a jornada: valida o recebível, aplica a taxa de deságio calculada pelo motor de risco, gera o contrato digital. |
| **Serviço de Pagamentos** | Executa o pagamento via Pix ao prestador e depois concilia com o repasse futuro do parceiro. |
| **Serviço de Compliance/Antifraude** | Monitora padrões suspeitos (recebíveis duplicados, velocidade atípica de solicitações) e bloqueia transações de risco. |
| **Base de Dados de Recebíveis** | Armazena o ciclo de vida de cada recebível: identificado → antecipado → pago pelo parceiro → conciliado. |
| **Serviço de Notificações** | Avisa o prestador sobre disponibilidade de antecipação, status do pagamento etc. (push, SMS, WhatsApp). |
| **BackOffice / Dashboard Administrativo** | Painel interno para o time de risco e operações acompanhar volume, inadimplência e performance por parceiro. |
| **Integração — Bureau de Crédito** | Serviço externo (Serasa/SPC) consultado pelo Motor de Score. |
| **Integração — Instituição Financeira Parceira** | Fonte de funding (capital) para as antecipações e/ou estrutura regulatória (fronting). |
| **Integração — Plataforma Parceira** | Sistema externo (ex: app de entrega, marketplace de serviços) de onde vêm os dados do recebível. |

### 2.2 Requisitos considerados importantes

1. **Alta disponibilidade (99,9%)** — o prestador depende do serviço para o fluxo de caixa do dia a dia; indisponibilidade tem impacto direto na vida financeira dele.
2. **Segurança de dados financeiros e pessoais (LGPD)** — dados bancários, documentos e histórico financeiro exigem criptografia em trânsito e em repouso, e controle de acesso rígido.
3. **Baixa latência na decisão de crédito** — a decisão de antecipar (ou não) precisa acontecer em segundos/poucos minutos; é o principal diferencial de valor do produto frente ao crédito tradicional.
4. **Escalabilidade horizontal para múltiplos parceiros simultâneos** — cada novo parceiro integrado não pode degradar a performance dos demais.
5. **Auditabilidade e rastreabilidade completa** — exigência regulatória: cada decisão de crédito e cada transação precisam ter trilha completa e reconstituível.
6. **Resiliência a falhas de integrações externas** — a indisponibilidade de um parceiro ou do bureau de crédito não pode derrubar o sistema inteiro (circuit breaker, filas, retries).

### 2.3 Reflexão sobre o diagrama

- **Sobre o que o diagrama ajuda a pensar:** evidencia que o produto é, no fundo, um orquestrador de risco entre três partes (prestador, parceiro, financiador) — e que a maior complexidade não está na antecipação em si, mas na camada de integração com sistemas externos heterogêneos.
- **Padrões essenciais no diagrama:** Gateway/API Gateway, Adapter (um conector por parceiro), orquestração de serviço (Serviço de Antecipação), circuit breaker implícito na integração externa.
- **Padrões ocultos:** padrão de conciliação financeira (saga/eventual consistency entre "pago ao prestador" e "recebido do parceiro"); padrão de auditoria (event sourcing implícito na trilha de recebíveis).
- **Metamodelo:** Sistema de software composto por serviços de domínio (negócio de antecipação), serviços de suporte (onboarding, notificação, backoffice) e adaptadores para sistemas externos — típico de uma arquitetura de plataforma fintech com integrações B2B2C.
- **Pode ser discernido no diagrama único?** Em parte — dá para ver os blocos principais e as integrações externas, mas não dá para ver o fluxo temporal (o que acontece primeiro) nem o nível de acoplamento interno entre os serviços; por isso os diagramas C4 complementam.
- **O diagrama está completo?** Não — está completo o suficiente para uma primeira leitura de negócio e arquitetura, mas falta detalhar a camada de dados (quais bancos, qual estratégia de consistência) e a camada de segurança (onde fica o cofre de dados sensíveis).
- **Poderia ser simplificado e ainda ser eficaz?** Sim — para uma audiência de negócio, os conectores por parceiro poderiam ser agrupados em um único bloco "Integrações Externas", sem perder o entendimento essencial.

### 2.4 Discussões e decisões da equipe

- **Discussão importante:** se o Motor de Score deveria ser um serviço único e centralizado ou um serviço por segmento de prestador (ex: motoristas vs. prestadores de serviço doméstico). Optamos por um serviço único com regras parametrizáveis por segmento, para não fragmentar o conhecimento de risco.
- **Decisão difícil:** se a validação antifraude deveria bloquear automaticamente (mais seguro, pior experiência) ou apenas sinalizar para revisão manual (melhor experiência, mais risco). Decidimos por bloqueio automático apenas para os casos de risco alto, com revisão manual para os casos intermediários.
- **Decisão tomada sob incerteza:** decidimos desenhar a arquitetura assumindo que o AntecipaJá **não** será uma SCD própria no curto prazo, e sim vai operar via parceria com instituição financeira licenciada — decisão de negócio que ainda depende de validação jurídica, mas que impacta diretamente o desenho dos serviços de Pagamentos e Compliance.
- **Ponto de decisão sem retorno:** a escolha do modelo de dados do "recebível" (contrato interno único, independente do formato de cada parceiro) foi um ponto sem volta — uma vez que os conectores dos parceiros são construídos sobre esse contrato, mudar o modelo depois exigiria reescrever todos os adaptadores.

---

## 3. C4 Model

> Diagramas completos em [`diagramas/editaveis`](diagramas/editaveis), formato `.drawio` (editável) e exportados como `.jpg` para a entrega [`diagramas/imagens`](diagramas/imagens).

### 3.1 Nível 1 — Contexto

Mostra o AntecipaJá e as pessoas/sistemas que interagem com ele, sem detalhe interno.

- **Pessoa:** Prestador de Serviço Autônomo
- **Sistema em foco:** AntecipaJá
- **Sistemas externos:** Plataforma Parceira, Bureau de Crédito, Instituição Financeira Parceira, Sistema de Pagamentos (Pix/PSP)

Arquivo: [`diagramas\imagens\02-c4-nivel1-contexto.jpg`](diagramas\imagens\02-c4-nivel1-contexto.jpg)

### 3.2 Nível 2 — Container

Detalha os containers (aplicações, serviços, bancos de dados) que compõem o AntecipaJá e como se comunicam.

- App/Portal do Prestador (Web/Mobile)
- API Gateway
- Serviço de Onboarding/KYC
- Serviço de Integração com Parceiros
- Motor de Score e Risco
- Serviço de Antecipação
- Serviço de Pagamentos
- Serviço de Compliance/Antifraude
- Banco de Dados de Recebíveis (PostgreSQL)
- Message Broker (Kafka) — para eventos entre serviços
- BackOffice Administrativo

Arquivo: [`diagramas/imagens/03-c4-nivel2-container.jpg`](diagramas/imagens/03-c4-nivel2-container.jpg)


### 3.3 Nível 3 — Componente

Detalha os componentes internos do **Motor de Score e Risco** (container escolhido para aprofundar, por concentrar a maior complexidade de negócio):

- Controller de Score (recebe a solicitação de avaliação)
- Motor de Regras de Risco (aplica as políticas por segmento)
- Cliente de Bureau de Crédito (integração externa)
- Serviço de Cálculo de Deságio (define a taxa)
- Repositório de Histórico de Score

Arquivo: [`diagramas/imagens/04-c4-nivel3-componente.jpg`](diagramas/imagens/04-c4-nivel3-componente.jpg)

### 3.4 Nível 4 — Código (opcional)

Não desenvolvido nesta entrega — nível opcional conforme orientação da disciplina.

---

## 4. Checklist de validação C4

Validação sugerida via [c4model.com/review](https://c4model.com/review/): Realizado

---

## 5. Vídeo de apresentação

Vídeo completo em:

---

