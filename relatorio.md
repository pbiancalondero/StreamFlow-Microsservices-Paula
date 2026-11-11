# StreamFlow


**Nome:**  Paula Bianca Londero. 
---

### Subindo os serviços

```bash
sudo docker compose up --build
```

<img width="905" height="288" alt="Captura de tela 2026-11-10 211737" src="https://github.com/user-attachments/assets/f09fde10-d2d6-4f5b-b57b-08d4fcf798af" />

### Bug encontrado no gateway — e corrigido

Ao consumir o health check do gateway logo após subir, o endpoint retornou erro: 503

```
GET http://localhost:8080/health
```

A causa foi identificada diretamente no código do gateway:

```js
const allUp = Object.values(results).every(r => r.status === 'up'); //status foi trocado para 'ok'
```

Os serviços internos respondem com `"status": "ok"`, mas o gateway comparava com `"up"` — comparação que nunca seria verdadeira.

**Observação:** esse bug revela um problema de observabilidade real — a equipe poderia estar monitorando um endpoint que sempre reporta degradação sem perceber, criando uma falsa sensação de problema permanente.

### Autenticação

Com os serviços rodando, o login funcionou corretamente:

```
POST http://localhost:8080/auth/login
```

```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_1",
    "name": "Ana Silva",
    "role": "admin"
  }
}
```

<img width="1348" height="389" alt="Captura de tela 2026-11-10 211739" src="https://github.com/user-attachments/assets/cd712a2d-1857-4d7c-a6b0-673345b66645" />


## Cenários de Teste

### Cenário 1 — Latência da Cadeia Síncrona

**Objetivo:** medir o custo real da cadeia síncrona no fluxo de Play e comparar com uma rota simples.

**Chamada complexa — POST /streaming/play:**

```
POST http://localhost:8080/api/streaming/play
Body: { "movieId": 2 }
```

Resposta:

```json
{
  "sessionId": 6,
  "movie": "Interestelar",
  "status": "playing",
  "message": "Reprodução iniciada."
}
```

<img width="1145" height="248" alt="Captura de tela 2026-11-10 211797" src="https://github.com/user-attachments/assets/6bda006f-7803-4b9b-ab34-577ebb050590" />


**Chamada simples — GET /catalog:**

```
GET http://localhost:8080/api/catalog
```
<img width="1301" height="370" alt="Captura de tela 2026-11-10 211789" src="https://github.com/user-attachments/assets/5db9584c-0a3c-4b3d-a8bb-c4cb3458588b" />

**Análise:** a diferença de tempo entre as duas chamadas é o custo direto da cadeia síncrona. O Play encadeia três chamadas bloqueantes antes de responder — catalog (verificação de licença), recommendation (registro de histórico) e notification (envio de alerta). O usuário espera o resultado de todas elas antes de ver qualquer resposta. Das três, apenas a verificação de licença precisa ser síncrona. As outras duas poderiam ser assíncronas, eliminando a maior parte dessa latência.

---

### Cenário 2 — Notification como Ponto de Falha

**Objetivo:** demonstrar o que acontece quando o notification-service fica indisponível.

Primeiro, o serviço foi parado:

```bash
docker compose stop notification-service
```

Em seguida, uma nova requisição de Play foi feita normalmente.

<img width="512" height="218" alt="Captura de tela 2026-11-10 231151" src="https://github.com/user-attachments/assets/329b8fb7-16d1-42c5-b41e-230687b6930a" />

```
streaming-service-1 | {"level":30,...,"msg":"Iniciando fluxo de play..."}
streaming-service-1 | {"level":40,...,"err":{"type":"TypeError",
  "message":"fetch failed: getaddrinfo EAI_AGAIN notification-service"},
  "msg":"Falha ao enviar notificação"}
streaming-service-1 | {"level":30,...,"res":{"statusCode":201},
  "responseTime":21.53,"msg":"request completed"}
```

**Análise:** o streaming-service registrou `level:40` (warning) pela falha na notificação, mas ainda assim retornou `statusCode: 201` ao usuário. 

O problema se torna crítico quando a falha ocorre em um serviço que *é* essencial — como o catalog para verificação de licença. Nesse caso, sem timeout configurado, o streaming-service ficaria bloqueado aguardando uma resposta que nunca chega, travando a thread e potencialmente derrubando o serviço inteiro por esgotamento de recursos.

---

### Cenário 3 — Banco Compartilhado entre Billing e Analytics

**Objetivo:** provar que o analytics-service está acoplado diretamente ao banco do billing-service, violando o princípio de banco isolado por serviço.

O billing foi parado:

```bash
docker compose stop billing-service
```

Em seguida, o analytics foi consultado normalmente:

```
GET http://localhost:8080/api/analytics/report
```

<img width="1246" height="518" alt="Captura de tela 2026-11-10 211790" src="https://github.com/user-attachments/assets/472199be-8b1c-4c1a-b097-427d851b093f" />


**Análise:** o analytics continuou respondendo com dados completos de assinaturas e faturas mesmo com o billing completamente fora do ar. Isso só é possível porque o analytics não consome a *API* do billing — ele lê diretamente o arquivo `shared_billing.db` via volume Docker compartilhado.

Essa é a definição de acoplamento pelo banco: dois serviços que teoricamente são independentes na camada de aplicação, mas que compartilham o mesmo armazenamento físico. As consequências práticas são:

- O billing não pode evoluir seu schema sem quebrar o analytics silenciosamente
- Não é possível fazer deploy independente dos dois serviços com segurança
- O analytics não tem domínio de dados próprio — ele é um parasita do billing

---

### Cenário 4 — Health Check Agregado

**Objetivo:** verificar como o gateway reporta o estado da plataforma com serviços indisponíveis.

Como os cenários anteriores já haviam derrubado billing e notification, o health check foi consultado nesse estado:

```
GET http://localhost:8080/health
```

<img width="1170" height="445" alt="Captura de tela 2026-11-10 211770" src="https://github.com/user-attachments/assets/b4d537c6-c5d9-450b-8da8-93331231c28e" />

**Análise:** com serviços down, o gateway retornou `HTTP 503` com `"status": "degraded"` e listou individualmente quais serviços estavam fora. Isso é a funcionalidade correta — o problema é que esse foi o *único* mecanismo de observabilidade que funcionou durante os testes. Não há correlation IDs propagados, não há tracing distribuído, não há métricas de latência. O health check informa *se* os serviços estão de pé, mas não consegue responder *por que* um incidente aconteceu, *quanto* durou ou *quantos usuários* foram afetados.

---

## Seção 1 — Custo Operacional Oculto

O gerente está certo. O custo não subiu por crescimento de usuários — subiu por overhead operacional que não entrega valor proporcional.

**O inventário atual:**

| Serviço | Banco | Justificativa como microsserviço |
|---|---|---|
| gateway |  |  Ponto de entrada único, essencial |
| auth-service | auth.db |  Domínio de identidade, sensível |
| catalog-service | catalog.db |  Conteúdo evolui independente |
| streaming-service | streaming.db |  Núcleo do negócio |
| recommendation-service | recommendation.db |  Poderia ser módulo do streaming |
| notification-service |  |  Stateless, sem domínio próprio |
| billing-service | shared_billing.db |  Domínio financeiro |
| analytics-service | shared_billing.db |  Banco compartilhado, pipeline interno |

Cada serviço carrega: 1 Dockerfile, 1 pipeline de CI/CD, 1 configuração de rede, variáveis de ambiente, endpoint `/health` a monitorar. Com 8 componentes, isso é ~40 variáveis de ambiente e 8 pipelines para uma equipe de 15 pessoas.

**Candidatos à consolidação:**

O `notification-service` é stateless (sem banco), não tem domínio de negócio próprio e é chamado de forma síncrona — exatamente o oposto do que justifica um microsserviço. Deveria ser uma chamada assíncrona (fila/evento) disparada internamente pelo streaming-service.

O `analytics-service` compartilha banco com o billing e opera um pipeline sequencial interno. Não tem API consumida por outros serviços, não precisa escalar de forma independente e não pode ser deployado sem coordenação com o billing. Deve ser consolidado como módulo interno do billing-service.

**Ação:** consolidar de 8 para 5–6 componentes. Estimativa de redução de 25–30% no custo de infraestrutura.

---

## Seção 2 — Latência de Rede

O gerente exagerou no número — são 3 hops síncronos, não 12 — mas o problema é real e foi medido. O fluxo de `POST /streaming/play` encadeia:

| Etapa | Chamada | Justifica bloqueio? | Latência medida |
|---|---|---|---|
| 1 | Catalog — verificação de licença |  Sim | ~10ms |
| 2 | Recommendation — registrar histórico |  Não | ~10ms |
| 3 | Notification — enviar alerta |  Não | 100–300ms (simulado) |

O resultado medido em ambiente local: **~300ms no Play vs ~30ms no Catalog** — 10x mais lento, com 270ms sendo overhead puro de chamadas que não precisam ser síncronas.

**Proposta:** manter a verificação de licença síncrona (é uma restrição de negócio). Converter o registro de histórico para evento assíncrono (fire-and-forget com BullMQ/Redis). Eliminar o notification-service como serviço independente e tratar notificações como evento assíncrono. Resultado esperado: latência do Play cai para ~30–40ms.

---

## Seção 3 — Observabilidade Frágil

O gerente está completamente certo, e os testes confirmaram. Durante toda a execução dos cenários, o único mecanismo de correlação disponível foi olhar log por log em cada serviço manualmente.

O bug encontrado no health check (comparação `'up'` vs `'ok'`) é um sintoma direto disso: um sistema com observabilidade saudável teria alertas que teriam detectado que o health check sempre retornava `degraded`, mesmo sem falhas reais.

**O que falta:**

| Capacidade | Status |
|---|---|
| Correlation ID propagado entre serviços |  Ausente |
| Tracing distribuído |  Ausente (saiu com Rafael e Camila) |
| Métricas de latência por serviço |  Ausente |
| Alertas automáticos de erro |  Ausente |
| Cálculo de usuários afetados por incidente |  Ausente |
| Health check funcional |  Presente, mas tinha bug |

**Proposta em etapas:**

1. **Correlation ID** — implementar no gateway um header `x-correlation-id` (UUID v4) propagado para todos os serviços downstream. Custo: ~2h. Resolve o rastreamento básico entre serviços.
2. **Logging padronizado** — todos os serviços já usam Pino/JSON. Padronizar campos obrigatórios: `correlation_id`, `service`, `duration_ms`, `user_id`.
3. **OpenTelemetry + Jaeger** — instrumentar com `@opentelemetry/sdk-node` e exportar traces para Jaeger (open source, deploy via Docker). Permite visualizar o caminho completo de qualquer requisição.
4. **Prometheus + Grafana** — expor métricas via `fastify-metrics` e criar dashboards com latência P95, taxa de erros e uptime por serviço.

---

## Seção 4 — Consistência Eventual

O gerente está certo. A ausência de propagação de eventos entre serviços cria janelas de inconsistência que afetam diretamente a experiência do usuário.

**O cenário crítico:** quando um filme é removido do `catalog-service`, o evento não é propagado para nenhum outro serviço. O `recommendation-service` continua armazenando aquele `movie_id` em `view_history` e `user_preferences`. O resultado é o usuário clicar numa recomendação e receber erro 404 ou conteúdo sem licença ativa.

**Níveis de consistência necessários por domínio:**

| Operação | Consistência | Justificativa |
|---|---|---|
| Verificação de licença no Play | Forte | Questão legal/contratual |
| Autenticação | Forte | Segurança |
| Remoção de catálogo → recommendations | Eventual (< 30s) | Delay curto é aceitável |
| Registro de histórico após Play | Eventual (segundos) | Usuário não percebe |
| Notificações | Eventual (minutos) | Efeito colateral tolerante |
| Relatórios de analytics | Eventual (horas) | Batch, não real-time |

**Proposta:** implementar eventos de domínio publicados pelo `catalog-service` em remoções. O `recommendation-service` escuta esses eventos e invalida os registros afetados. Tecnologia sugerida: Redis Pub/Sub — já comum em stacks Node.js e de baixo custo operacional.

---

## Seção 5 — Lei de Conway

A Lei de Conway diz que a arquitetura de um sistema tende a espelhar a estrutura de comunicação da organização que o produz. Com 15 pessoas e 8 componentes, a razão é de 1,9 pessoas por componente — abaixo do mínimo recomendado para manter conhecimento distribuído.

A saída de Rafael e Camila não foi só perda de mão de obra: foi perda do conhecimento sobre as camadas mais críticas e menos documentadas da infraestrutura (tracing e service mesh). Três incidentes sem diagnóstico em duas semanas confirmam que o conhecimento estava concentrado, não distribuído.

**Reorganização proposta:**

| Time | Pessoas | Responsabilidade |
|---|---|---|
| Core (Plataforma) | 5 | Gateway + Auth + Streaming |
| Conteúdo | 5 | Catalog + Recommendation |
| Negócio | 5 | Billing + Analytics (consolidado) |

Três times, cada um com domínio claro e autonomia real. A consolidação dos serviços (Seção 1) é pré-condição para essa reorganização funcionar — não faz sentido um time de 5 pessoas manter 3 serviços com os antipatterns atuais.

**Ações complementares:** documentar Architecture Decision Records (ADRs) antes de qualquer migração, criar runbooks de resposta a incidentes por serviço, e implementar pair programming obrigatório para features críticas de infraestrutura.

---
