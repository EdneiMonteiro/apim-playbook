# Plataforma APIM corporativa em Azure — guia técnico de decisão e implantação (v3)

> **Versão:** v3 — consolidada em resposta à revisão da versão anterior (nota v2: B+/A−).
> **Changelog v2 → v3:** ver §16 ao final. Mudanças desta versão: referências regulatórias específicas (§15), exemplos numéricos de SLO por classe (§10.4), protocolo de comunicação de sunset (§11.4), citação validada de TPS por unit + tabela de custo por componente (§8.6), notas de rodapé com responsável/prazo em vez de "Verificar" (§3.4/§6.6), glossário de siglas (Apêndice E), apêndices operacionais A (policy XML por classe), B (OpenAPI esqueleto), C (ADR template), D (pipeline YAML), F (mapeamento ISO 27001/NIST CSF), G (1-pager executivo).

---

## 1. Sumário executivo

API Management (APIM) em Azure é a peça central de uma estratégia de APIs corporativa: ponto único de exposição governada, autenticação, autorização, throttling, observabilidade, catálogo e contrato com produtores e consumidores. Ele não substitui a engenharia das APIs; ele formaliza o contorno entre quem produz, quem governa e quem consome.

Para uma corporação grande, três decisões dominam o desenho: **modelo de adoção** (corporativo, por domínio, ou híbrido governado), **SKU/tier** (clássico v1, v2 ou Premium v2) e **arquitetura de rede** (público com WAF, interno via VNet, ou misto via workspaces no Premium clássico). A escolha precisa ser sustentada por requisitos regulatórios, residência, modelo de exposição e maturidade de plataforma — não por preferência tática.

A recomendação base é **modelo híbrido/federado governado**: uma plataforma APIM corporativa com guardrails, padrões e catálogo, mais a possibilidade controlada de instâncias dedicadas para domínios com requisitos distintos (residência, regulatório, exposição, SLO). Plataforma compartilhada por padrão; instância dedicada por exceção justificada.

### 1.1 Decida em 5 perguntas (quadro executivo)

Use este quadro para o gate inicial. Cada resposta empurra o desenho para um SKU/região/modelo específico. As respostas devem ser documentadas em ADR no início do programa.

| # | Pergunta | Se a resposta é… | Decisão indicada |
|---|----------|------------------|------------------|
| 1 | A API atende **clientes/dados restritos a região brasileira** (residência exigida)? | Sim | Restringir SKU ao que está disponível em Brazil South/Brazil Southeast; hoje **Basic v2/Standard v2** disponíveis; **Premium v2 ainda não listado** em Brazil South — confirmar antes de assumir [1][3]. Avaliar Premium clássico se VNet integrada/multi-region for crítica. |
| 2 | A API é **regulada** (Open Finance, BACEN, ANPD/LGPD com dados sensíveis, PCI)? | Sim | Tratar como classe "regulada" (§7.7). Exige OAuth/OIDC, mTLS quando aplicável (FAPI), audit log retido conforme norma, residência confirmada, threat model documentado, DPO ciente. |
| 3 | A API é **exposta a parceiros/internet** ou apenas **interna**? | Externa | Premium clássico VNet externa + WAF, ou v2 com Application Gateway/Front Door + WAF; APIM nunca publicado diretamente sem WAF para externo. |
| 4 | O domínio tem **autonomia operacional e SLO próprios** (próprio time, runbook, on-call)? | Sim, forte | Considerar instância dedicada (modelo B/híbrido). Caso contrário, fica no APIM corporativo compartilhado. |
| 5 | Volume previsto **excede capacidade do tier escolhido** ou exige **multi-region/DR ativo**? | Sim | Premium clássico (única opção atual com multi-region nativo e self-hosted gateway). Premium v2 não cobre esses cenários hoje [1]. |

### 1.2 Ponto crítico Brasil

Disponibilidade regional **muda a recomendação**: o que vale para US East/West Europe pode não valer para Brazil South. No momento desta consolidação (referência **maio/2026**), em Brazil South estão listados **Basic v2 e Standard v2**; **Premium v2 e Workspace gateway (Premium clássico v2 mode) não constam na matriz pública** consultada [1][3]. Premium clássico continua disponível com VNet integrada, multi-region e self-hosted gateway, recursos não cobertos por Premium v2.

Implicação: **se residência em Brasil for um requisito não negociável e o desenho exigir VNet integrada ou multi-region, hoje a única rota é Premium clássico, com plano de migração assistido para v2 quando o tier estiver disponível** (acompanhar [1][3]). Decisões devem ser datadas e revisadas trimestralmente.

---

## 2. Contexto e problema

Grandes empresas tipicamente operam dezenas a centenas de APIs em múltiplos domínios (produto, plataforma, dados, parceiros, mobile, integrações B2B), com regimes distintos de regulação, criticidade, exposição (interna, parceira, pública) e governança (centralizada, federada, autônoma).

Adotar Azure de forma estratégica significa, entre outras coisas, **resolver como APIs entram, saem e circulam** dentro e fora da nuvem com segurança, controle e governança. APIM é o ponto técnico onde a maior parte dessas decisões ganha forma operacional.

Erros comuns que este documento busca prevenir:

- Escolher modelo (centralizado vs distribuído) por preferência cultural, sem cruzar com regulação, escala e ownership.
- Padronizar SKU/tier sem revisar o roteiro da Microsoft (v1 vs v2), aposentadoria de capacidades e disponibilidade regional.
- Tratar APIM como proxy/network appliance, sem governança de produto, lifecycle, catálogo e developer experience.
- Mover governança para fora do APIM (manual, planilha, wiki), criando atrito sem ganho operacional.
- Esquecer integração entre APIM, Azure AD/Entra ID, Azure Policy, Defender for Cloud, observabilidade e pipelines.

### 2.1 Premissas e escopo

- O cliente é uma empresa grande, com múltiplos domínios, parceiros, requisitos de regulação (LGPD, possivelmente setor financeiro/Open Finance, ANPD, BACEN) e times distribuídos.
- Há Azure Landing Zone (ou equivalente) com hub/spoke, Azure Policy, Log Analytics, Sentinel e Entra ID.
- O foco é APIs HTTP (REST, GraphQL, gRPC quando aplicável). Mensageria pura (Event Hubs/Service Bus) está fora do escopo do APIM, embora APIs orientadas a eventos possam ser fachadas via APIM.

### 2.2 Quem este documento orienta

| Perfil | O que extrair |
|--------|---------------|
| Liderança (CIO/CTO/Head Eng) | §1, §4, §11, §12, §13 |
| Arquitetura Azure | §3, §5, §8, §9 |
| Segurança/IAM/Regulatório | §1.1, §2.3, §7, §11 |
| Plataforma/Landing Zone | §5, §8, §9, §10 |
| API Producer (squad) | §4, §6, §7, §10 |
| API Consumer (mobile/web/parceiro) | §6.5, §10, §11.6 |
| FinOps/Procurement | §3.3, §8.6 |
| Auditoria/Risco | §2.3, §7.7, §7.8, §11.5 |

### 2.3 Drivers regulatórios (Brasil + setoriais)

A escolha de modelo, SKU, região e controles deve ser ancorada em requisitos regulatórios datados e mapeados, não em interpretação subjetiva. Este documento parte do conjunto mínimo abaixo:

#### 2.3.1 LGPD (Lei nº 13.709/2018) — base obrigatória

- **Bases legais** (art. 7º): identificar a base que ampara cada API que trafega dado pessoal (consentimento, execução de contrato, obrigação legal, etc.). Sem base mapeada, não publicar.
- **Direitos do titular** (art. 18): acesso, retificação, exclusão, portabilidade, anonimização, informação sobre compartilhamento. APIs de atendimento desses direitos devem ser identificadas, testadas e auditadas; APIM pode expor a fachada e roteá-los para o sistema-fonte.
- **Encarregado/DPO**: nominal e contatável. Operações de APIM que envolvam dado pessoal precisam estar no inventário do DPO, com retenção de log alinhada à política.
- **Relatório de impacto (RIPD)**: APIs que tratam dado sensível (saúde, biometria, financeiro nominal) devem ter RIPD anexado à ficha técnica.
- **Notificação de incidente** (art. 48): runbook (§10.6) deve incluir fluxo de notificação à ANPD em incidentes com risco/dano relevante.

Ver Resolução CD/ANPD nº 15/2024 (regulamento de comunicação de incidentes) e materiais da ANPD para detalhes operacionais [27].

#### 2.3.2 BACEN e Open Finance Brasil

Aplica-se a instituições autorizadas e participantes do Open Finance. Drivers:

- **Resolução BCB nº 32/2020** e ciclos seguintes definem fases, dados e serviços do Open Finance Brasil [28].
- **Convenção Open Finance** e **manuais técnicos**: padrão FAPI 1.0 (Financial-grade API), DCR (Dynamic Client Registration), assinatura JWS, mTLS, JWT-Secured Authorization Request (JAR) [29][29b].
- **Resolução BCB nº 198/2022** (uso de serviços de processamento/armazenamento e computação em nuvem por instituições financeiras) [28b] — exige inventário, contratos, residência confirmada quando aplicável, plano de continuidade e relação com terceiros.
- **Resolução CMN nº 4.893/2021** (política de segurança cibernética e requisitos para contratação de serviços de processamento/armazenamento e computação em nuvem) [28c] — testes, gestão de incidente, comunicação a BACEN em prazos definidos. (Esta resolução substitui parte do escopo da antiga Res. CMN 4.658/2018; **a equipe de compliance confirma o ato vigente antes de uso em auditoria**.)

Implicação para APIM: APIs Open Finance são uma **classe própria** (§7.7), com mTLS obrigatório (cliente apresenta certificado emitido pela cadeia ICP/Open Finance), token JWT assinado, validação rígida de claims, logging completo com retenção mínima alinhada à norma (em geral 5 anos para registro de transações financeiras), e teste regular.

#### 2.3.3 Setoriais adicionais

- **PCI-DSS**: se há trânsito de dado de cartão, isolar a classe; APIM apenas em modo passthrough cifrado, sem log do PAN, com escopo PCI claramente delimitado.
- **Saúde** (Lei nº 13.787/2018 sobre prontuário eletrônico, e LGPD dado sensível): retenção, auditoria e DPO específicos.
- **Telecom (Anatel), seguros (Susep), capital (CVM)**: cada qual com requisitos próprios. APIM deve registrar a classificação setorial da API na ficha.

#### 2.3.4 Mapeamento operacional

Toda API publicada no APIM corporativo deve, na ficha de intake, indicar:

1. Classe regulatória aplicável (nenhuma / LGPD base / LGPD dado sensível / Open Finance / PCI / setorial).
2. Base legal LGPD (quando aplicável).
3. Retenção mínima de log exigida.
4. Residência exigida.
5. Aprovador regulatório (DPO, compliance setorial).
6. Critérios de incidente e prazos de notificação.

Sem esse mapeamento, o gate de publicação rejeita.

---

## 3. Azure API Management Service: visão geral, v1 vs v2, SKUs e disponibilidade regional

### 3.1 Visão geral

Azure API Management é o gateway de APIs gerenciado da Microsoft em Azure. Conceitualmente, expõe três planos:

- **Plano de gestão (control plane)** — ARM/Bicep/Terraform, portal, REST API, RBAC.
- **Plano de runtime (data plane / gateway)** — recebe a requisição, aplica policies, autenticação, throttling, transformação, encaminha ao backend.
- **Plano de desenvolvedor (developer plane)** — portal do desenvolvedor (Developer Portal), catálogo, documentação, onboarding, gestão de subscriptions.

Capacidades centrais: policies declarativas (XML), produtos, APIs, operações, named values, certificados, identidades gerenciadas, integração com Azure Monitor/Log Analytics/Application Insights, Defender for APIs, Azure Policy. Em SKUs habilitados, também: VNet integration, multi-region, self-hosted gateway, workspaces (Premium clássico em modo workspace gateway), Developer Portal customizado e API Center como inventário/catálogo corporativo.

### 3.2 Terminologia: clássico/v1, stv1/stv2, Consumption e tiers v2

Para evitar confusão recorrente:

- **APIM clássico (v1)** — tiers Developer, Basic, Standard, Premium e Consumption. Arquitetura sobre máquina virtual (stv2 atual; stv1 já aposentado conforme [4]).
- **APIM v2** — nova geração, arquitetura serverless, tiers **Basic v2**, **Standard v2** e **Premium v2** (em rollout regional). Trazem mudanças importantes em rede, escalabilidade, custos e recursos [1].
- **Consumption** — tier pay-per-call, baseado em clássico. Adequado para cargas leves/serverless, com limitações (sem VNet integrada, sem self-hosted gateway, etc.).
- **Workspaces** (Premium clássico) — segregação multi-equipe dentro de uma única instância, cada workspace com seu gateway dedicado (workspace gateway). Diferente do conceito antigo "developer portal multi-tenant".

> Ponto crítico de revisão: workspaces ≠ workspace gateway ≠ múltiplas instâncias. Workspace é o limite lógico de governança dentro de uma instância Premium clássico; o workspace gateway é o componente que executa o tráfego daquele workspace. Para multi-tenant pesado, decida entre múltiplas instâncias dedicadas ou Premium clássico com workspaces — não confundir os conceitos.

### 3.3 Orientação por SKU/tier

| Tier | Quando faz sentido | Quando NÃO faz sentido |
|------|--------------------|------------------------|
| Developer | Sandbox, testes, treinamento | Produção (sem SLA) |
| Basic v2 | Produção leve, sem VNet integrada, sem multi-region | Regulado, residência crítica que exija recursos não listados, cenários híbridos |
| Standard v2 | Produção média, sem VNet integrada nativa avançada | Multi-region, self-hosted, workspaces |
| Premium v2 | Produção crítica (onde listado regionalmente), capacidade alta | Multi-region nativa, self-hosted gateway, backup/restore, múltiplos custom domains, e qualquer cenário híbrido pesado — não cobertos hoje [1] |
| Premium clássico | Multi-region, VNet integrada, self-hosted gateway, workspaces, backup/restore, múltiplos custom domains | Cargas leves; o custo não se justifica para escopo reduzido |
| Consumption | APIs internas leves, eventos esporádicos, prototipagem | Cargas com policies pesadas, latência crítica, requisitos VNet |

**Decisão SKU resumida:**
- Cenário híbrido com self-hosted gateway → **Premium clássico** (Premium v2 não cobre, conforme [1]).
- Multi-region ativo/ativo nativo → **Premium clássico**.
- Carga corporativa moderna, sem necessidade de multi-region/self-hosted, na região onde Premium v2 está listado → **Premium v2** ou Standard v2.
- Brasil + requisito de VNet ou multi-region hoje → **Premium clássico** (até Premium v2 ser listado em Brazil South) [1][3].

### 3.4 Disponibilidade regional e nota específica sobre Brasil

A disponibilidade regional **muda decisões** e precisa ser checada antes de qualquer commitment. Referência atual consultada em **maio/2026** [1][3]:

- **Brazil South:** Basic v2 e Standard v2 listados. **Premium v2 não listado** na matriz consultada. Workspace gateway (Premium clássico v2 mode) não listado. Premium clássico permanece disponível (legacy).
- **Brazil Southeast:** reduzido em recursos como nas demais regiões "Southeast" do Azure. Confirmar antes de assumir multi-region Brasil ativo/ativo.

Implicações:

1. Se a corporação tem **residência obrigatória em Brasil** (LGPD, BACEN, política interna), Premium v2 não é uma opção hoje. Premium clássico é a rota.
2. Se a corporação aceita **multi-region cross-border** (Brasil + região secundária fora), Premium clássico pode aceitar multi-region; Premium v2 não tem multi-region nativo.
3. API Center (catálogo corporativo) também tem **disponibilidade regional limitada** [10]; pode ser que precise rodar em região secundária e referenciar APIs hospedadas no Brasil — isso é aceitável desde que metadados não contenham dados sensíveis.

Tabela de disponibilidade — referência **maio/2026** (verificar [1] e [3] antes de fechar arquitetura):

| Recurso | Brazil South | Brazil Southeast¹ | East US | West Europe |
|---|---|---|---|---|
| Basic v2 | Disponível | Verificar¹ | Disponível | Disponível |
| Standard v2 | Disponível | Verificar¹ | Disponível | Disponível |
| Premium v2 | **Não listado** | Não listado | Disponível | Disponível |
| Workspace gateway (Premium clássico v2 mode) | **Não listado** | Não listado | Disponível | Disponível |
| Premium clássico (VNet, multi-region, self-hosted) | Disponível | Verificar¹ | Disponível | Disponível |
| API Center | Verificar¹ | Não listado | Disponível | Disponível |

¹ Itens marcados "Verificar" são revalidados **trimestralmente** pelo time de plataforma (owner: lead de arquitetura Azure) na matriz oficial [1][3]; última confirmação registrada em ADR-APIM-001.

> Recomendação: registrar a data da consulta e o link da matriz oficial na ADR. Revisar a cada trimestre.

---

## 4. Modelos de adoção (corporativo, por subscrição/domínio, híbrido) com matriz de decisão

Empresas grandes oscilam entre três modelos. Não há "certo absoluto" — há melhor encaixe para cada combinação de regulação, escala, autonomia, governança e maturidade.

### 4.1 Modelo A — APIM corporativo compartilhado

Uma (ou poucas) instância(s) APIM, operada(s) pela plataforma, expondo APIs de múltiplos domínios. Workspaces (Premium clássico) ou produtos segmentam ownership.

Vantagens: padronização forte; custo unitário menor; um único ponto de auditoria, catálogo e SLOs; superfície reduzida de exposição pública.
Riscos: gargalo organizacional na plataforma; mistura de cargas com criticidade distinta; janelas de manutenção compartilhadas; "tragédia dos commons" em policies.
Sinais favoráveis: maturidade central forte, regulação homogênea entre domínios, volume médio.
Sinais desfavoráveis: domínios com SLO/regulação heterogêneos, parceiros B2B exigindo isolamento, on-call distribuído sem suporte da plataforma.

### 4.2 Modelo B — APIM por subscrição, projeto ou domínio

Cada domínio/squad opera sua instância APIM. Governança via Azure Policy, blueprints, templates corporativos.

Vantagens: autonomia total; blast radius restrito; flexibilidade de SKU/região por domínio; políticas internas por domínio.
Riscos: custo agregado alto; padronização frouxa; catálogo fragmentado; observabilidade fragmentada; superfície de exposição ampliada; difícil política regulatória uniforme.
Sinais favoráveis: domínios autônomos com plataforma própria, regulação muito heterogênea (ex.: parte da empresa em Open Finance, parte não), M&A com integração ainda em curso.
Sinais desfavoráveis: corporação com governança centralizada, time de plataforma pequeno, requisitos regulatórios uniformes.

### 4.3 Modelo C — Híbrido/federado governado (recomendado para corporações médias-grandes)

Plataforma corporativa opera o(s) APIM(s) padrão (modelo A) e habilita instâncias dedicadas mediante critérios objetivos (residência distinta, regulação específica, exposição parceira crítica, escala que justifique isolamento). O catálogo é unificado via **API Center** [10], a observabilidade vai para Log Analytics central, a governança técnica via Azure Policy e blueprints é mandatória independentemente do modelo.

Vantagens: padronização sem rigidez; autonomia justificada; catálogo único; melhor relação custo/risco.
Riscos: complexidade de governança (a plataforma precisa ser madura); risco de "instância dedicada" virar regra por baixa fricção — exige gate.
Sinais favoráveis: grande maioria das corporações grandes com plataforma estabelecida.

### 4.4 Self-hosted gateway para cenários híbridos

Quando há backends on-premises, em outra cloud ou em região onde APIM não está disponível, o **self-hosted gateway** estende o data plane para o ambiente do cliente, mantendo control plane no Azure. Disponível em **Premium clássico e Developer** [1]; não disponível em Premium v2, Standard v2, Basic v2 e Consumption na referência consultada.

Casos típicos: cargas regulatórias on-premises (data center próprio), edge computing, multi-cloud (AWS/GCP) com governança APIM unificada, latência crítica perto de backends.

Restrições: licenciamento por unidade, requer container runtime (Kubernetes recomendado), sincronização com control plane Azure, custos operacionais on-prem.

### 4.5 Matriz de decisão pontuada (com critérios eliminatórios)

A matriz qualitativa do v1 foi substituída por uma **matriz pontuada com critérios eliminatórios** para reduzir subjetividade.

#### 4.5.1 Critérios eliminatórios

Se qualquer um for "não" para o modelo escolhido, descartá-lo, independentemente da pontuação:

| Critério eliminatório | Pergunta | Modelo descartado se… |
|---|---|---|
| Residência | A região exigida está disponível para o SKU do modelo? | Modelo cuja região exigida não tem o SKU. |
| Regulação setorial | A classe regulatória é suportada operacionalmente pelo modelo? | Modelo cujo controle não atende (ex.: Open Finance exigindo mTLS em SKU sem suporte adequado). |
| SLO/disponibilidade | O modelo entrega o SLO mínimo (ex.: 99,95 com multi-region)? | Modelo sem multi-region quando exigido. |
| Self-hosted | É necessário gateway on-prem? | Modelo cujo SKU não suporta self-hosted. |

#### 4.5.2 Critérios pontuados (0 = ruim, 1 = aceitável, 2 = bom, 3 = ótimo)

| Critério | Peso | Modelo A (corporativo) | Modelo B (por domínio) | Modelo C (híbrido) |
|---|---|---|---|---|
| Padronização técnica | 3 | 3 | 1 | 3 |
| Autonomia de squads | 3 | 1 | 3 | 2 |
| Custo agregado | 3 | 3 | 1 | 2 |
| Blast radius / isolamento | 3 | 1 | 3 | 2 |
| Velocidade de entrega | 2 | 1 | 3 | 2 |
| Governança regulatória | 3 | 3 | 1 | 3 |
| Catálogo unificado | 2 | 3 | 1 | 3 |
| Capacidade operacional / on-call | 2 | 3 | 1 | 2 |
| **Pontuação total (máx. 63)** | — | **45** | **33** | **51** |

Faixas indicativas: **0–30** desfavorável; **31–44** aceitável com ressalvas; **45+** favorável. Pontuações nominais devem ser recalculadas para o contexto específico (peso ajustado).

Recomendação: para corporação grande, Modelo C é favorito por padrão, e essa é a recomendação base do documento. Modelos A ou B podem ser justificados em contextos específicos — desde que registrados em ADR.

---

## 5. Arquitetura de referência

Combina APIM como entry plane com Azure Landing Zone, Azure AD/Entra ID, observabilidade central e pipelines IaC.

### 5.1 Componentes recomendados

- **APIM** (clássico Premium ou v2 conforme §3.3/§3.4).
- **Application Gateway/WAF** ou **Azure Front Door com WAF** quando o APIM é exposto à internet. APIM nunca diretamente sem WAF para externo.
- **Azure AD/Entra ID** para autenticação humana (publishers, devs) e workload identities.
- **Key Vault** para segredos, certificados e chaves de assinatura/encriptação.
- **Log Analytics + Application Insights** para sinais de runtime; **Sentinel** para correlação de segurança.
- **Defender for APIs** (Defender for Cloud) para detecção de anomalias e shadow APIs [25].
- **API Center** [10] como inventário corporativo e catálogo unificado. **Ressalva:** API Center tem disponibilidade regional limitada; conferir se a região exigida tem o serviço — caso não, hospedar em região secundária e referenciar APIs do Brasil é aceitável, desde que metadados não contenham dados sensíveis.
- **Azure Policy** para guardrails (naming, tagging, SKU permitidos, criptografia, region restriction).
- **Bicep/Terraform + GitHub Actions/Azure DevOps** para IaC e pipelines APIOps.

### 5.2 Padrões de integração

- **APIM → backend** sempre em rede privada quando o backend está em Azure (VNet integration, Private Link, Service Endpoint). Backends externos: HTTPS + mTLS quando crítico.
- **APIM ↔ Entra ID** via OAuth/OIDC (JWT validation policy), com discovery e cache de chaves.
- **APIM ↔ Defender for APIs** para inventário e detecção; integrado nativamente nos SKUs suportados [25].
- **APIM ↔ Log Analytics** com diagnostic settings completos (gateway logs, audit logs).
- **APIM ↔ API Center** para sincronizar inventário [10][24].
- **APIM ↔ Sentinel** via Log Analytics como pipeline padrão para SIEM.

### 5.3 Diagrama lógico (descrito)

```
[Consumidor externo]
       │
       ▼
[Front Door + WAF]  (apenas para APIs externas)
       │
       ▼
[APIM (Premium clássico, VNet interna ou externa, multi-region)]
       │
       ├──► [App Service / AKS / Functions] (backends Azure, via Private Link/VNet)
       │
       ├──► [Self-hosted gateway] ──► [Backends on-prem / outras clouds]
       │
       └──► [API Center] (catálogo corporativo, sincronização)

[Plano de gestão & observabilidade]
   Entra ID  ↔  Key Vault  ↔  Log Analytics  ↔  Sentinel  ↔  Defender for APIs
```

---

## 6. Padrões de API, lifecycle, versionamento, catálogo e consumo

### 6.1 Princípios de desenho de API

- **Contratos OpenAPI 3.0+ obrigatórios** antes da publicação (intake MPAS/MAP, §6.2).
- **Recursos REST** ou alternativas justificadas (GraphQL, gRPC). REST como default.
- **Erros padronizados** (RFC 7807 problem+json).
- **Versionamento explícito** (caminho `/v1`, `/v2`).
- **Idempotência** documentada por verbo (PUT idempotente; POST não, mas pode oferecer chave de idempotência).
- **Naming consistente** (kebab-case em paths; camelCase em payload JSON).

### 6.2 MPAS/MAP como intake obrigatório

Toda API nova passa por um intake formal — **MPAS (Microsoft Partner Architectural Solution)** ou **MAP (Microsoft Architecture Plan)** como modelo, adaptado ao contexto corporativo. Contém:

- Owner (squad, lead, on-call, DPO/compliance quando regulado).
- Classificação (interna, externa B2B, sensível, regulada).
- Classe regulatória (§2.3.4).
- SLO/SLA esperado.
- Volume previsto (TPS, MAU).
- Dados trafegados (sensibilidade, base legal LGPD).
- Modelo de autenticação (subscription key, OAuth, mTLS, FAPI).
- Backend, rede, dependências.
- Plano de retirada (sunset policy).

Sem intake aprovado, a API não publica.

### 6.3 Lifecycle recomendado

| Estado | Critério de entrada | Critério de saída |
|---|---|---|
| **Design** | Intake aprovado | OpenAPI + ADR |
| **Develop** | OpenAPI assinado pelo owner | Build verde, testes unitários, lint |
| **Pre-prod (UAT)** | Build promovido via pipeline | Validação consumer + segurança + perf básica |
| **Produção** | Aprovação de release | Em uso pelos consumidores |
| **Deprecated** | Substituta publicada (`/v2` ativa) | Comunicação a consumidores |
| **Sunset** | Janela de deprecação encerrada | Remoção do APIM e do catálogo |

### 6.4 Versionamento e revisions

- **Versão (path)** para mudanças incompatíveis. `/v1`, `/v2` etc.
- **Revision** para mudanças compatíveis. Permite rollback rápido.
- **Política de deprecação mínima**: comunicação 90 dias antes; banner no Developer Portal; e-mail aos subscribers; remoção após sunset.

### 6.5 Produtos, catálogo e consumo

- **Produtos** agrupam APIs e definem policies de acesso (quotas, rate limit, escopos OAuth, termos de uso).
- **Developer Portal** customizado para a corporação: branding, fluxo de onboarding self-service, gestão de subscriptions, exemplos, sandbox.
- **API Center** [10] como visão executiva e auditoria: inventário, classificações, owners, status, regulatório. **Ressalva regional:** se API Center não está disponível na região corporativa, hospedar em região secundária; conferir trimestralmente.
- **Sandbox** segregado: ambiente onde devs externos/parceiros testam sem credenciais de produção.

### 6.6 Protocolos não-REST e AI gateway

Além de REST/SOAP, APIM cobre outros protocolos. Cada um tem suporte distinto por SKU/região — confirmar antes de assumir.

| Protocolo / capacidade | Suportado em | Limitações | Quando faz sentido |
|---|---|---|---|
| **GraphQL synthetic / passthrough** | Premium clássico, Standard v2, Premium v2² | Cache parcial, observabilidade reduzida vs REST | Fachada para consumidores que demandam GraphQL sem refazer backend |
| **WebSocket APIs** | Premium clássico, Premium v2² | Throttling limitado; observabilidade sub-óptima | Streaming de cotações, eventos client→server persistentes |
| **OData passthrough** | Premium clássico, v2² | Validação OData não nativa | APIs legadas SAP/Dynamics |
| **gRPC** | Em rollout/limitado² | Verificar antes de comitar | APIs internas alta performance |
| **AI gateway (token rate limit / semantic caching)** | GA em SKUs selecionados [12] | Custo cresce com tokens; semantic cache requer Azure Cache for Redis Enterprise | APIs que fazem fan-out para Azure OpenAI ou modelos próprios |
| **MCP server gateway** | Preview/rollout [12] | Atenção a versão de protocolo e SDK | Ferramentas internas Copilot/agents corporativos |

² Suporte por SKU/região varia por release; revalidação **trimestral** pelo time de plataforma (owner: lead de APIs) contra a matriz oficial [1][12]; última confirmação registrada em ADR-APIM-001.

Pontos comuns:
- **Citar SKU e região** ao escolher; o suporte muda a cada release.
- **Observabilidade** específica: GraphQL e WebSocket exigem painéis dedicados (latência por subscription, tamanho médio de query).
- **Threat model** específico: GraphQL precisa de complexity limit, depth limit, query allowlist quando público; WebSocket precisa de timeout e quota por conexão.

---

## 7. Segurança, identidade e políticas

### 7.1 Plano de gestão

- **Entra ID** para todo acesso ao plano de gestão (portal, ARM, REST API).
- **RBAC mínimo** com roles built-in (API Management Service Reader/Contributor/Operator).
- **PIM** (Privileged Identity Management) para roles privilegiadas; just-in-time + aprovação.
- **Audit log** completo em Log Analytics; correlação no Sentinel.

### 7.2 Cliente → APIM

Modelos por classe (detalhado em §7.7):

- **OAuth 2.0 / OpenID Connect** (Bearer JWT) é o padrão para APIs autenticadas. Validação via `validate-jwt` ou `validate-azure-ad-token` (incluindo a policy `validate-azure-ad-token` documentada em [16]).
- **mTLS** quando exigido (parceiros B2B, Open Finance/FAPI).
- **Client credentials flow** para machine-to-machine.
- **Authorization Code + PKCE** para clientes públicos (SPA, mobile).
- **Subscription keys** apenas como complemento (rastreio de consumo); nunca como segurança principal.
- **RFC 9700** (Best Current Practice for OAuth 2.0 Security) como referência operacional [21].

### 7.3 APIM → backend

- **mTLS** quando o backend for crítico/regulado.
- **Managed Identity** quando o backend está em Azure (App Service, Functions, Cosmos DB, etc.).
- **Private Link / VNet integration** para que o tráfego não saia do backbone Azure.
- **Allowlist por IP** com restrições adicionais quando o backend não suporta identidade.

### 7.4 Subscription keys

- Úteis para rastreio, cobrança e quotas; **não substituem autenticação**.
- Rotação automatizável via pipeline.
- Nunca expor em código cliente; armazenar em vault no consumidor.

### 7.5 Policies recomendadas

Padrões obrigatórios de policies (configuráveis por produto/API):

- `validate-jwt` ou `validate-azure-ad-token` para JWT [16].
- `rate-limit-by-key` e `quota-by-key`.
- `ip-filter` para classes internas.
- `set-header` removendo cabeçalhos sensíveis (`Server`, `X-Powered-By`).
- `cors` configurada de forma restrita (`*` apenas em sandbox).
- `validate-content` (JSON Schema) para payload validation em classes reguladas.
- `check-header`/`set-header` para correlation IDs (`x-correlation-id`).
- Fragmentos de policy reutilizáveis (Policy Fragments) para evitar duplicação.

### 7.6 Threat model (STRIDE) × OWASP API Top 10

Threat model deve ser produzido por classe (§7.7) e atualizado anualmente ou por mudança material. Mapeamento mínimo:

| STRIDE | Risco típico em APIM | Controles APIM | OWASP API Top 10 [20] |
|---|---|---|---|
| Spoofing | Cliente forjar identidade | OAuth/OIDC; mTLS; PKCE | API2 Broken Auth |
| Tampering | Payload manipulado em trânsito | TLS obrigatório; payload validation; HMAC quando aplicável | API8 Security Misconfiguration |
| Repudiation | Falta de evidência de quem chamou | Audit log; correlation ID; assinatura de logs em Sentinel | API9 Improper Inventory Management |
| Information Disclosure | Vazamento via erro/header/log | Erros padronizados; remoção de headers; políticas de log sem PII | API3 Excessive Data Exposure |
| Denial of Service | Flood, abuse | rate-limit; quota; WAF; throttling backend; circuit breaker em backend | API4 Lack of Resources & Rate Limiting |
| Elevation of Privilege | Token mal validado, claim trustada cegamente | Validação rígida (issuer, audience, scope, claims); deny-by-default | API5 Broken Function Level Authorization |
| (Adicional) | Acesso direto ao backend | Restrição de rede; allowlist IPs APIM no backend; mTLS APIM→backend | API8 |
| (Adicional) | Shadow API | Defender for APIs [25]; inventário API Center [10] | API9 |

Saída: documento `threat-model-{classe}.md` por classe regulatória, anexado ao backlog do produto.

### 7.7 Matriz "classe de API × controles mínimos"

| Classe | Exemplo | Auth cliente | mTLS | Rede | Payload | Logging | Retenção | Aprovador |
|---|---|---|---|---|---|---|---|---|
| **Interna** | API entre microsserviços corporativos | OAuth client-credentials | Opcional | VNet interna (private endpoint) | Schema obrigatório | Padrão APIM | 90 dias | Owner do domínio |
| **Externa B2B** | API para parceiro comercial | OAuth + subscription key + IP allowlist | Recomendado | Externa via WAF | Schema obrigatório + sanitização | Padrão APIM + correlação | 1 ano | Segurança + jurídico |
| **Sensível (LGPD)** | API que trafega dado pessoal | OAuth Authorization Code + PKCE; sem subscription key como único controle | Opcional/Recomendado | Externa via WAF | Schema + masking em log | Sem PII em log; correlation ID; auditoria de acesso | 2+ anos (alinhar com DPO) | DPO + segurança |
| **Regulada/financeira (Open Finance)** | API Open Finance Brasil | OAuth + mTLS (FAPI); JWS request signing | **Obrigatório** | Externa com WAF + IP allowlist por cadeia ICP | Schema FAPI + validação de assinatura | Audit completo; assinatura de log | 5+ anos | DPO + compliance BACEN |
| **Administrativa** | API de gestão interna do APIM | OAuth + PIM + role-based; **bloqueada para internet** | Não aplicável | Interna obrigatória | Schema mínimo | Audit completo + alarmes | 7 anos (governança) | Plataforma + auditoria |
| **Pública não autenticada** | Página de status, healthcheck | Nenhuma (read-only) | Não | Externa com WAF + rate limit agressivo | Sem payload sensível | Padrão APIM | 90 dias | Plataforma |

> Esta matriz é **vinculante**: o gate de publicação (§9.3) valida que a API publicada cumpre os controles da sua classe.

### 7.8 Operacionalização de requisitos LGPD/BACEN em APIM

Aterrissar regulação em controles APIM concretos:

| Requisito regulatório | Onde aterrissa em APIM | Como evidenciar |
|---|---|---|
| LGPD art. 7º (base legal) | Ficha de intake da API (§6.2) | Campo "base legal" preenchido; reviewer compliance assinou |
| LGPD art. 18 (direitos do titular) | API "direitos do titular" publicada com SLO próprio | Inventário em API Center; SLA monitorado |
| LGPD art. 46-47 (segurança) | Policies de criptografia, audit, mTLS quando aplicável | Azure Policy/Defender for Cloud + relatórios |
| LGPD art. 48 (notificação de incidente) | Runbook §10.6 + integração SIEM (Sentinel) | Playbook de notificação ANPD dentro de prazo (Resolução CD/ANPD nº 15/2024) [27] |
| BACEN Res. 32/2020 / Open Finance | Classe regulada (§7.7) + FAPI + cadeia ICP | Certificados na cadeia; mTLS habilitado; logs assinados |
| BACEN Res. 198/2022 (cloud) | Mapeamento de serviços; residência; plano BCP | Inventário Azure; ADR de região; runbook DR |
| BACEN Res. 4.893/2021 (cibersegurança) | Testes, gestão de incidente, comunicação | Calendário de pentest; SOC; reporte BACEN |

Cada item deve ser auditável: link para evidência (issue, PR, runbook, certificado) registrado na ficha da API.

---

## 8. Plataforma, landing zone, rede e guardrails

### 8.1 Aderência à Azure Landing Zone

- Subscrições padronizadas (Connectivity, Identity, Platform, Workloads).
- APIM corporativo nas subscrições de Platform; instâncias dedicadas nas Workloads do domínio.
- Hub-spoke: hub com firewall central, ExpressRoute/VPN; APIM no hub ou em spoke dedicado.
- Naming/tagging conforme padrão corporativo (§8.2).

### 8.2 Naming, tagging e ownership

Exemplo de convenção:

- `apim-{corp}-{env}-{regiao}-{seq}` — ex.: `apim-acme-prd-bs-01`.
- Tags obrigatórias: `owner`, `costcenter`, `domain`, `data-classification`, `regulatory-class` (§2.3), `slo`, `oncall`, `criticality`.
- Ownership do recurso APIM corporativo: time de plataforma; ownership por API: domínio produtor.

### 8.3 RBAC

- **API Management Service Contributor** apenas para automação/plataforma.
- **API Management Service Reader** para devs/squads (read-only).
- **API Management Service Operator** quando squad precisa publicar via portal (raro — preferir pipeline).
- Custom roles para `policy-only-edit`, `read-logs-only`, `developer-portal-publisher`.

### 8.4 Azure Policy e guardrails

Policies mínimas:

- Bloquear criação de APIM fora dos SKUs aprovados.
- Bloquear criação de APIM fora das regiões aprovadas.
- Exigir tags obrigatórias.
- Exigir diagnostic settings ativados.
- Exigir Defender for APIs habilitado [25].
- Bloquear desativação de criptografia ou TLS antigo.
- Exigir Private Link em backends classificados como sensíveis/regulados.

### 8.5 Rede, DNS e exposição

- **APIM Premium clássico VNet interna** para APIs internas; **VNet externa** quando exposta.
- **Front Door/Application Gateway com WAF** sempre antes do APIM para tráfego externo.
- **DNS privado** para resolução interna; **DNS corporativo** para externo.
- **Certificados**: gerenciados no Key Vault, com rotação automática.

### 8.6 FinOps de APIM (sizing, custo, showback/chargeback)

Custo de APIM não vem apenas da licença do SKU. Vem de quatro grandes drivers:

1. **APIM unit/SKU** — Premium clássico tem custo por unit/região (multi-region = unit × regiões). Premium v2 cobra por escala. Standard/Basic v2 têm custo previsível.
2. **Observabilidade** — Log Analytics ingestion + retention. Em volume alto, custo de log pode superar o APIM. Política tiered de retenção (hot 30d / warm 90d / archive 1+ ano) e amostragem de logs não-críticos podem reduzir 40-70 %.
3. **Componentes de borda** — Front Door, Application Gateway, WAF, Defender for APIs. Defender for APIs cobra por API monitorada.
4. **Tráfego de saída e Private Link** — egress regional; Private Link endpoint hourly + por dado processado.

#### 8.6.1 Sizing inicial

- Estimar TPS pico × policy weight (cada policy adiciona ms; políticas de cache reduzem CPU).
- Cada APIM unit Premium clássico atende, **como estimativa de planejamento**, da ordem de **~2.500 TPS com policies leves** — número de referência amplamente citado pela comunidade, mas que **não consta como SLA oficial em [6]**. Tratar como **ponto de partida**, e **validar em teste de carga** com as policies reais antes de comitar com FinOps/produto.
- Multi-region adiciona unit por região; custo cresce linearmente.

#### 8.6.2 Tabela "componente × custo unitário típico" (referência)

Valores ilustrativos para sizing rápido — **revalidar no Azure Pricing Calculator** [30] para sua subscrição/região:

| Componente | Unidade de cobrança | Faixa típica (referência) | Observação |
|---|---|---|---|
| APIM Premium clássico | por unit-hora, por região | ordem de centenas USD/mês/unit | Multi-region multiplica |
| APIM Standard v2 | por unit-hora | dezenas USD/mês | Sem multi-region nativo |
| APIM Basic v2 | por unit-hora | unidades USD/mês | Cargas leves |
| Log Analytics ingestion | por GB ingerido | sensível ao volume | Tiered retention reduz custo |
| Log Analytics retention | por GB/mês após retenção incluída | linear com retenção | Archive tier reduz |
| Defender for APIs | por API monitorada | por unidade | Habilitar por classe |
| Application Gateway WAF v2 | por capacity unit | hourly + por dado | Necessário para externo |
| Front Door Premium | por endpoint + dado processado | hourly + GB | Inclui WAF |
| Private Link endpoint | hourly + por GB processado | hourly fixo | Adiciona por endpoint |
| Egress regional | por GB de saída | linear | Otimizar via cache/compressão |

> Não use esta tabela como cotação. Use como **ordem de grandeza** para identificar drivers dominantes e decidir onde otimizar. Cotação real é do FinOps/procurement com base em [30].

#### 8.6.3 Tagging e showback/chargeback

- Cada API e produto carrega tag `costcenter` e `domain` (§8.2).
- Log Analytics workspace estruturado por categoria (gateway logs vs audit) permite atribuir consumo de log por domínio.
- Relatório mensal de **showback** por domínio (consumo, requests, custo aproximado por componente) e **chargeback** quando a maturidade permitir.
- Defender for APIs custos podem ser segmentados via tag das APIs monitoradas.

#### 8.6.4 Alarmes orçamentários

- Azure Budgets por subscrição APIM + alertas em 50/80/100 %.
- Alertas específicos por componente: Log Analytics ingestion, Defender for APIs, Front Door.
- Revisão trimestral em fórum FinOps.

#### 8.6.5 Otimizações comuns

- **Cache de policies** (cache-store/cache-lookup) reduz CPU e latência.
- **Compression** (gzip) reduz egress.
- **Sampling de logs** para classes não reguladas.
- **Retenção tiered** (hot/warm/archive).
- **Reserved capacity** quando o consumo for previsível.

---

## 9. Pipeline de publicação, IaC e APIOps

### 9.1 Princípios

- **Infraestrutura como código (IaC)** obrigatória. Bicep ou Terraform.
- **Policies em XML versionado** em repositório, com fragmentos reutilizáveis.
- **OpenAPI versionado** como contrato.
- **Build determinístico**, ambientes idempotentes.
- **GitOps**: PR → revisão → merge → deploy automático.
- **Trunk-based development** + feature flags em vez de branches longas.

### 9.2 Fluxo alvo

```
[dev] → branch feature
   ↓
[PR] → CI: lint OpenAPI, lint policy, schema validation, unit tests
   ↓
[review] → owner + reviewer cruzado + (se regulado) compliance/DPO
   ↓
[merge main] → deploy automático em DEV → smoke tests
   ↓
[promoção] → UAT → testes de contrato + performance básica + segurança automatizada
   ↓
[release] → PROD com aprovação humana + janela definida
   ↓
[verify] → smoke prod + observabilidade verde por N horas
```

### 9.3 Validações obrigatórias em PR/pipeline

- **Lint OpenAPI** (Spectral) com regras corporativas.
- **Lint policy** (xmllint + policy linter customizado).
- **Schema validation** de payloads em testes de contrato (Pact ou similar).
- **Testes unitários** de policies (Bicep what-if; policy expression dry-run).
- **Verificação de classe regulatória** (§7.7): se classe = regulada, então gate exige mTLS, audit completo, retenção mínima.
- **Verificação de ficha de intake** atualizada (§6.2).
- **Verificação de threat model** atualizado para classe (§7.6).

### 9.4 Drift, exceções e rollback

#### 9.4.1 Drift detection

Drift acontece — uma policy ajustada no portal em incidente, uma config mudada na crise. A plataforma precisa detectar e reconciliar.

Ferramentas concretas (referência [13][14]):

- **Azure APIOps Toolkit** (ferramenta open-source apoiada por Microsoft): extrai configuração da instância e compara com repositório (drift report) [13].
- **`az deployment what-if`** ou **Terraform `plan`**: comparam template vs estado.
- **Bicep what-if** + **ARM what-if API**: para Bicep declarativo.
- **Pulumi/Terragrunt** para quem usa esses gestores.
- **Schedule de drift check** (GitHub Action / Azure Pipeline) diário ou semanal; resultado posta em canal de plataforma.

Política: drift identificado **não é resolvido manualmente**; é replicado para o IaC (se a mudança é desejada) ou revertido (se não é). Cada drift gera issue.

#### 9.4.2 Exceções

Ver §11.5 — modelo formal com ciclo de vida.

#### 9.4.3 Rollback

- **Revisions** APIM (mudança compatível) → rollback imediato via portal/IaC.
- **Versions** (mudança incompatível) → coexistência por janela mínima; rollback significa redirecionar consumidores.
- **Blue/green** ou canary para mudanças de policy críticas, usando produtos distintos no Developer Portal.

### 9.5 Multi-instância: repositórios e sincronização

Para corporações com várias instâncias APIM (modelo C), padronizar:

#### 9.5.1 Estrutura de repositórios

Duas estratégias comuns:

- **Monorepo** — um único repositório para todas as instâncias, organizado por pasta `/instances/{instancia}/...`. Vantagem: compartilhamento de policy fragments fácil. Desvantagem: CI/CD mais complexo, blast radius de PR maior.
- **Polirepo** — um repositório por instância + um repositório `apim-shared` com fragmentos. Vantagem: ownership claro. Desvantagem: sincronização exige tooling.

Recomendação: para 2-4 instâncias, monorepo com pipelines paralelos por instância. Para 5+, polirepo + repo central de shared.

#### 9.5.2 Sincronização de policy fragments

- Fragmentos comuns (JWT validation corporativa, correlation ID, cors padrão) ficam em `apim-shared`.
- Publicação via release tag; instâncias consomem por versão (`apim-shared@v1.4.2`).
- Mudança quebrada exige bump de major e plano de migração.

#### 9.5.3 Drift entre instâncias

- Drift report comparando instâncias com fragmentos compartilhados — divergências geram issue.
- Política: nenhuma instância pode ficar mais de 60 dias atrás da versão major mais recente do shared.

#### 9.5.4 Catálogo unificado

- Cada instância publica seu inventário no **API Center** corporativo [10].
- Sincronização automática (cron) ou via webhook.

---

## 10. Operação, observabilidade e SLO/SLA

### 10.1 Sinais obrigatórios

Métricas mínimas por API (publicadas em dashboard):

- Latência (p50, p95, p99).
- Taxa de erro (HTTP 4xx, 5xx separados).
- Throughput (RPS).
- Backend latency (medido via APIM).
- Capacity (CPU/memória) do APIM.
- Token validation latency (quando aplicável).
- Cache hit rate (quando aplicável).

Logs mínimos:

- Gateway logs (request/response sem PII em classes sensíveis).
- Audit logs (mudanças de configuração).
- Diagnostic logs.

### 10.2 Logs, métricas e alertas

- Diagnostic settings completos → Log Analytics central + Sentinel.
- Application Insights por API quando o produtor quiser correlação fim-a-fim.
- Métricas customizadas via policy (`emit-metric`).
- Alertas em SLO: error budget consumido X %; latência p99 acima do limite; taxa de 5xx > threshold.

### 10.3 Dashboards

Mínimo por instância APIM:

- Dashboard "Plataforma": saúde geral, capacity, top APIs por volume/erro.
- Dashboard "Segurança": tentativas de token inválido, 401/403 anômalos, IPs anômalos, Defender alerts [25].
- Dashboard "FinOps": ingestion Log Analytics, Defender for APIs, egress (§8.6).
- Dashboard por produto/domínio: APIs do domínio com SLO/erro/latência.

### 10.4 SLO, SLA e capacidade

- **SLO** definido pelo owner do produto, validado pela plataforma. Ex.: 99,9 % availability, p99 < 500 ms.
- **SLA** ao consumidor externo só após SLO maduro (3 meses verde).
- **Error budget** consumido publicamente; freeze de mudança quando budget esgota.
- **Capacity planning** trimestral; teste de carga obrigatório a cada SKU upgrade/downgrade.

#### 10.4.1 SLO numérico de referência por classe

Valores indicativos para ancorar discussão com owners; cada API valida no contexto:

| Classe (§7.7) | Availability | Latência p99 (gateway-only) | Error budget mensal |
|---|---|---|---|
| Regulada/financeira (Open Finance) | 99,95 % | ≤ 800 ms | ≈ 22 min |
| Sensível (LGPD) | 99,9 % | ≤ 800 ms | ≈ 43 min |
| Externa B2B | 99,9 % | ≤ 1.000 ms | ≈ 43 min |
| Interna | 99,5 % | ≤ 1.500 ms | ≈ 3 h 40 min |
| Administrativa | 99,5 % (horário comercial) | ≤ 2.000 ms | conforme janela |
| Pública não autenticada | 99,9 % | ≤ 1.500 ms | ≈ 43 min |

Latência p99 acima refere-se ao tempo gateway (excluindo backend); SLO ao consumidor inclui backend e é definido por API.

### 10.5 Matriz HA/DR × SKU × criticidade

| Criticidade | SKU base recomendado | HA dentro de região | DR (multi-region) | RTO | RPO | Runbook |
|---|---|---|---|---|---|---|
| **Crítica regulada (Open Finance, pagamentos)** | Premium clássico | Sim (AZs nativo) | Sim, ativo-ativo entre regiões | < 5 min | ~0 | Runbook DR ensaiado anualmente |
| **Alta (negócio essencial)** | Premium clássico ou Premium v2 (onde houver) | Sim (AZs) | Ativo-passivo entre regiões | < 30 min | < 5 min | Runbook DR ensaiado semestralmente |
| **Média (negócio importante)** | Standard v2 | AZs nativo | Backup/restore + IaC recriação | < 4 h | < 1 h | Runbook DR documentado, ensaio anual |
| **Baixa (interno auxiliar)** | Basic v2 ou Standard v2 | AZs onde disponível | IaC recreate | < 24 h | < 24 h | IaC + smoke test |
| **Sandbox/Dev** | Developer | Não | Não | — | — | — |

Observações:
- Premium v2 hoje **não cobre multi-region ativo-ativo**; para crítica regulada, Premium clássico é mandatório [1].
- Backup/restore só em Premium clássico; em outros SKUs, depender de IaC + recriação.
- AZ-aware: confirmar suporte a Availability Zones na região (Brazil South suporta; Brazil Southeast verificar).
- Multi-region cross-border: avaliar se a regulação permite (LGPD/BACEN podem exigir residência).

### 10.6 Runbooks/playbooks operacionais mínimos

Cinco runbooks obrigatórios, mantidos no repositório de operação, ensaiados periodicamente:

1. **Surge de 5xx em uma API** — passos: dashboard → identifica backend → circuit breaker → comunicar consumidores → escalar para owner da API e on-call backend.
2. **Backend indisponível total** — passos: confirmar via alerta + dashboard → ativar fallback (se houver) → comunicar → status page → coordenação com owner backend.
3. **Certificado próximo de expirar / expirado** — passos: alerta D-30 e D-7 → rotação no Key Vault → validação em dev → janela de troca produção → validação. Para FAPI/Open Finance: coordenar com cadeia ICP.
4. **Rotação de chave / segredo (incidente ou agendada)** — passos: gerar nova chave no Key Vault → atualizar policies → comunicar consumidores impactados → revogar antiga após carência. Incidente: comprimir prazo, gerar relatório de evidência.
5. **DR failover (regional outage)** — passos: confirmar outage Azure → ativar região secundária → atualizar DNS/Front Door → validar tráfego → comunicar status page → no retorno, drenar de volta de forma controlada. Ensaiado conforme criticidade (§10.5).

Adicionais recomendados:
6. **Incidente de segurança (LGPD)** — fluxo de comunicação ao DPO e ANPD em prazos (Resolução CD/ANPD nº 15/2024) [27].
7. **Incidente de cibersegurança (BACEN)** — comunicação a BACEN conforme Resolução CMN nº 4.893/2021 [28].
8. **Reporte mensal FinOps** — fechamento de custo por componente, alocação por domínio.

Cada runbook tem owner, frequência de ensaio, critério de sucesso e data do último ensaio.

---

## 11. Governança corporativa, ownership e experiência do desenvolvedor

### 11.1 Princípios de governança

- **Governança via guardrails técnicos**: Azure Policy, gates de pipeline, lint, validação de classe. Documento/wiki como complemento, nunca como controle único.
- **Ownership inquestionável**: cada API tem owner nominal (squad), técnico (lead), DPO/compliance quando regulado, on-call definido.
- **Decisão registrada em ADR**: modelos, SKUs, regiões, exceções.
- **Catálogo único** (API Center [10]) como visão executiva.
- **Caminho rápido para o comum**, caminho aprovado para o excepcional (§11.6).

### 11.2 Ownership mínimo

Cada API publicada precisa, na ficha (§6.2):

- Owner do produto (squad).
- Tech lead.
- On-call rotation (PagerDuty/Opsgenie/Microsoft Teams).
- DPO/compliance (quando classe regulada).
- Aprovador de release.

### 11.3 RACI resumido

| Atividade | Squad | Plataforma APIM | Segurança | Compliance/DPO | Auditoria |
|---|---|---|---|---|---|
| Design da API | R | C | C | C (se regulada) | I |
| Publicação | R | A | C | C (se regulada) | I |
| Policies padrão (JWT, rate-limit) | C | R/A | C | I | I |
| Policy custom de domínio | R | A | C | C (se regulada) | I |
| Operação 24×7 | R (API) | R (instância) | I | I | I |
| Incidente regulatório | R | C | R | A | C |
| Exceções (§11.5) | R | R | C | C | C |

### 11.4 Experiência do produtor e consumidor

- **Self-service** no Developer Portal: registro, subscription, gestão de keys, exemplos, sandbox.
- **Producer toolkit**: templates Bicep, fragmentos policy, esqueleto OpenAPI (Apêndice B), pipeline reference (Apêndice D).
- **Documentação viva**: OpenAPI sempre atual, gerado da fonte; mudanças visíveis no portal.
- **SLAs do canal de suporte** definidos para consumidores externos.

#### 11.4.1 Comunicação de sunset / deprecação a consumidores

Mudança de versão incompatível ou sunset exige protocolo de comunicação consistente:

| Canal | Quando | Conteúdo mínimo |
|---|---|---|
| **E-mail** ao subscriber (endereço cadastrado no Developer Portal) | D-90, D-30, D-7 antes do sunset | API, versão, data, link da nova versão, suporte |
| **Banner no Developer Portal** (página da API) | A partir de D-90, persistente até remoção | Versão deprecada, prazo, nova versão |
| **Header HTTP `Deprecation`** (RFC 8594) e `Sunset` | A partir de D-90 em respostas da versão deprecada | Data do sunset |
| **Webhook opcional para parceiros B2B** | Eventos: deprecation-announced, sunset-reminder, sunset-executed | Payload assinado JWT com dados da API |
| **Status page corporativa** | Janela de remoção e durante incidentes relacionados | Versão removida, contato |
| **Tickets proativos** para top consumidores | A partir de D-60, reabertos se sem ack em D-30 | Plano de migração, prazo, owner |

Política: nenhuma versão é removida sem ao menos os 3 primeiros canais ativos por ≥ 90 dias e confirmação de ausência de tráfego por 14 dias consecutivos.

### 11.5 Modelo de exceção (ciclo de vida formal)

Exceções existem; sem ciclo de vida, viram regra. Modelo mínimo:

#### 11.5.1 Formulário de exceção

Toda exceção (ex.: usar SKU não aprovado, abrir região nova, ignorar policy obrigatória, postergar mTLS) tem ficha com:

- ID, data, requisitante (squad + lead).
- Descrição do desvio.
- Justificativa de negócio.
- Análise de risco (impacto × probabilidade).
- Mitigações compensatórias.
- Aprovador (depende da criticidade — ver §11.5.3).
- Prazo de validade (data de expiração obrigatória).
- Critério de saída (o que precisa acontecer para a exceção ser fechada).

#### 11.5.2 Ciclo de vida

```
[criada] → [em análise] → [aprovada com prazo] → [ativa] → [revisão antes de expirar] →
   [renovada (raro)] | [encerrada — desvio resolvido] | [escalada — não resolvido no prazo]
```

#### 11.5.3 Aprovador conforme criticidade

| Tipo de exceção | Aprovador |
|---|---|
| Operacional (config menor) | Owner plataforma |
| Padrão técnico (lint, naming) | Plataforma + arquitetura |
| Segurança (policy obrigatória ignorada) | CISO/segurança + plataforma + DPO se aplicável |
| Regulatória (BACEN, LGPD sensível) | Compliance + DPO + jurídico |

#### 11.5.4 Inventário

Inventário público de exceções (read-only para auditoria) em sistema de tickets (Jira/Azure Boards) e exposição agregada via API Center.

#### 11.5.5 Métricas

- Total de exceções ativas.
- Exceções vencidas (KPI principal — meta: zero).
- Tempo médio de exceção (dias).
- Exceções renovadas (>1 ciclo).

### 11.6 Self-service vs aprovado (caminho rápido vs caminho governado)

Para evitar que tudo passe pelo time central, mapear tipos de mudança:

| Tipo de mudança | Caminho | Aprovador | Gate |
|---|---|---|---|
| Nova revision de policy (compatível) | Self-service | Owner do domínio | CI verde |
| Nova versão de API (incompatível) | Self-service + comunicação | Owner + plataforma | CI + plano de deprecação |
| Mudança de policy obrigatória (JWT validation) | Governado | Plataforma + segurança | Threat model atualizado |
| Novo produto no catálogo | Self-service | Owner | CI + ficha intake |
| Nova API regulada | Governado | DPO + compliance + plataforma | Classe §7.7 atendida |
| Nova instância APIM dedicada | Governado | Plataforma + arquitetura | Critério §4.5 + ADR |
| Nova região APIM | Governado | Plataforma + arquitetura | Disponibilidade verificada [1][3] |
| Exceção a policy obrigatória | Governado | §11.5.3 conforme tipo | Ficha de exceção |
| Mudança de SKU (upgrade/downgrade) | Governado | Plataforma + FinOps | Teste de carga + ADR |

Métrica de saúde: % de mudanças via self-service. Meta: ≥ 70 %. Abaixo disso, plataforma é gargalo.

---

## 12. Anti-patterns e boas práticas

### 12.1 Lista detalhada

#### 1) APIM por aplicação sem critério
Cada squad cria seu APIM porque pode. Custo explode, catálogo fragmenta, governança evapora. Correção: gate via §4.5; instância dedicada só por exceção justificada.

#### 2) APIM apenas como proxy
Sem policies, sem versionamento, sem catálogo. Joga fora 80 % do valor. Correção: padrões mínimos (§6, §7) obrigatórios.

#### 3) Centralização sem segmentação
Tudo em um produto, todas APIs misturadas. Correção: workspaces (Premium clássico) ou produtos como limite, ownership claro (§11.2).

#### 4) Misturar APIs internas/externas/sensíveis sem estratégia
Tudo no mesmo APIM, sem classes. Correção: matriz §7.7 vinculante; instâncias dedicadas quando criticidade exigir.

#### 5) Sem ownership
"De quem é essa API?" sem resposta. Correção: §11.2 obrigatório.

#### 6) Publicação manual em produção
Portal → produção → drift. Correção: APIOps (§9), portal read-only em PROD.

#### 7) Sem versionamento
Mudança quebra consumidor. Correção: §6.4 obrigatório, sunset policy de 90 dias.

#### 8) Lógica de negócio demais nas policies
Policies viram código de aplicação. Correção: policies só fazem cross-cutting (auth, throttling, transformação leve, correlation, validação de schema); negócio fica no backend.

#### 9) Sem observabilidade
"Por que está lento?" sem resposta. Correção: §10.1 obrigatório.

#### 10) Subscription keys como segurança principal
API exposta apenas com chave estática. Correção: OAuth/mTLS conforme §7.7.

#### 11) Expor backends diretamente, contornando o APIM
Bypass via rede pública ao backend. Correção: §7.6 + restrição de rede (Private Link, allowlist).

#### 12) Não separar ambientes
DEV/UAT/PROD na mesma instância. Correção: instâncias separadas, IaC parametrizado.

#### 13) Escolha tardia de SKU/região (novo)
Decidir SKU "depois" e descobrir que Premium v2 não está disponível na região, ou que self-hosted requer Premium clássico. Correção: aplicar §1.1 (5 perguntas) e §4.5 antes do design; verificar [1] e [3].

#### 14) Governança central sem caminho rápido (novo)
Tudo passa pela plataforma; squads gastam dias para uma revision trivial. Correção: §11.6 — self-service para o comum, governado para o excepcional. Métrica ≥ 70 % self-service.

### 12.2 Matriz resumida

| Anti-pattern | Sintoma | Correção | Onde |
|---|---|---|---|
| Proxy puro | Sem policies, sem catálogo | Padrões §6, §7 | §12.1 #2 |
| Sem ownership | Sem responsável | Ownership obrigatório | §11.2 |
| Publicação manual | Drift em produção | APIOps | §9 |
| Backend exposto | Bypass do APIM | Restrição rede | §7.6 |
| SKU/região tardia | Surpresa no design | 5 perguntas + matriz §4.5 | §1.1 |
| Sem self-service | Plataforma gargalo | Caminho rápido | §11.6 |

---

## 13. Recomendações finais e roadmap sugerido

### 13.1 Recomendações finais

1. Aplicar **5 perguntas** (§1.1) antes de qualquer design.
2. Usar **modelo C híbrido governado** como padrão; exceções via §11.5.
3. **SKU/região com data**: hoje (maio/2026), Brasil + VNet/multi-region = Premium clássico. Revisar trimestralmente.
4. **Classes de API** (§7.7) vinculantes no gate.
5. **APIOps** (§9) obrigatório; portal read-only em PROD.
6. **API Center** [10] como catálogo unificado (com ressalva regional).
7. **Threat model** (§7.6) por classe; revisão anual.
8. **FinOps** (§8.6) com showback e meta de otimização.
9. **Self-service** (§11.6) com meta ≥ 70 %.
10. **Modelo de exceção** (§11.5) com inventário e métricas.
11. **DR ensaiado** (§10.5) conforme criticidade.
12. **Runbooks** (§10.6) mínimos com owner.

### 13.2 Roadmap sugerido (com owner, entrega e critério de aceite)

| Horizonte | Entrega | Owner | Critério de aceite |
|---|---|---|---|
| **0–30d** | ADR de modelo (A/B/C), SKU e região, com base em §1.1 e §4.5 | Arquitetura + plataforma | ADR aprovada por arquitetura, segurança e DPO; publicada em wiki corporativa |
| **0–30d** | Inventário inicial de APIs existentes classificadas por §7.7 | Plataforma + squads | Inventário no API Center [10]; 100 % das APIs com classe registrada |
| **30–60d** | APIM corporativo provisionado via IaC; Azure Policy guardrails ativos | Plataforma | Bicep/Terraform em main; az deployment what-if limpo; Defender for APIs habilitado |
| **30–60d** | 5 runbooks mínimos (§10.6) escritos e validados em dev | Plataforma + on-call | Cada runbook ensaiado em DEV; documentação publicada |
| **60–90d** | Pipeline APIOps com lint, gates de classe e drift detection | Plataforma + DevEx | Pipeline em produção; drift report semanal; gate de classe ativo |
| **60–90d** | Threat model (§7.6) de todas classes em produção | Segurança + DPO | Documento aprovado para cada classe; revisão calendarizada |
| **90–120d** | FinOps showback por domínio (§8.6) | FinOps + plataforma | Relatório mensal publicado; 3 domínios pilotos com chargeback |
| **90–120d** | Modelo de exceção (§11.5) em sistema de tickets | Plataforma + risco | Ciclo de vida implementado; inventário acessível |
| **120–180d** | DR ensaio (§10.5) para 100 % das classes crítica e alta | Plataforma + on-call | Relatório de ensaio com RTO/RPO medidos; ações corretivas em backlog |
| **180–240d** | Self-service ≥ 70 % (§11.6) | Plataforma + DevEx | Métrica medida e publicada; gate só nos casos definidos |
| **Contínuo** | Revisão trimestral de SKU/região vs [1][3]; revisão anual de threat models | Arquitetura + segurança | Calendário ativo; ADR atualizada quando muda |

---

## 14. Decisões pendentes que dependem do cliente

Decisões só responsáveis pelo cliente, com gate associado:

- **Modelo de adoção definitivo** (A/B/C) — gate: ADR aprovada em 30d (§13.2).
- **Residência obrigatória** (sim/não, Brasil only ou multi-region cross-border) — gate: política regulatória registrada em §2.3.4.
- **Classe regulatória das APIs do roadmap** (Open Finance, LGPD sensível, PCI) — gate: cada API com classe na ficha de intake (§6.2).
- **Aprovador de exceções** por tipo — gate: nomeado em §11.5.3.
- **Owner de FinOps** — gate: papel atribuído antes de §13.2 90–120d.
- **Janela de DR e RTO/RPO formal** por classe — gate: §10.5 preenchida com valores específicos.
- **Política de retenção de logs** por classe — gate: alinhada com DPO em §7.7/§7.8.
- **Política de uso de Premium clássico vs migração futura para Premium v2** — gate: revisão trimestral §13.2.
- **Estratégia para regiões fora de Brazil South** quando Premium v2 não estiver listado — gate: ADR datada.

---

## 15. Referências consolidadas

> Todas as referências devem ser revalidadas em datas críticas (regulatório, SKU/região). Data desta consolidação: **maio/2026**.

### Azure API Management — conceitos, tiers, regiões e recursos

[1] Microsoft Learn — API Management feature-based comparison (v1, v2, Consumption, self-hosted). https://learn.microsoft.com/azure/api-management/api-management-features
[2] Microsoft Learn — APIM v2 tiers overview. https://learn.microsoft.com/azure/api-management/v2-service-tiers-overview
[3] Microsoft Azure — Region availability for APIM v2 SKUs. https://azure.microsoft.com/explore/global-infrastructure/products-by-region
[4] Microsoft Learn — Migrate from stv1 to stv2 (lifecycle). https://learn.microsoft.com/azure/api-management/migrate-stv1-to-stv2
[5] Microsoft Learn — Workspaces in APIM Premium. https://learn.microsoft.com/azure/api-management/workspaces-overview
[6] Microsoft Learn — Capacity metric and units. https://learn.microsoft.com/azure/api-management/api-management-capacity
[7] Microsoft Learn — Multi-region deployment (Premium clássico). https://learn.microsoft.com/azure/api-management/api-management-howto-deploy-multi-region
[8] Microsoft Learn — Self-hosted gateway overview. https://learn.microsoft.com/azure/api-management/self-hosted-gateway-overview
[9] Microsoft Learn — Backup and restore APIM. https://learn.microsoft.com/azure/api-management/api-management-howto-disaster-recovery-backup-restore
[10] Microsoft Learn — Azure API Center overview. https://learn.microsoft.com/azure/api-center/overview
[11] Microsoft Learn — Aposentadoria de analytics clássicos APIM (março/2027). https://learn.microsoft.com/azure/api-management/breaking-changes/
[12] Microsoft Learn — GenAI gateway capabilities (token limit, semantic caching). https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities

### Rede, conectividade, resiliência e arquitetura

[13] Azure Architecture Center — APIOps Toolkit. https://github.com/Azure/apiops
[14] Microsoft Learn — Deploy APIM with Bicep/Terraform via what-if. https://learn.microsoft.com/azure/azure-resource-manager/templates/deploy-what-if
[15] Microsoft Learn — VNet integration (internal/external). https://learn.microsoft.com/azure/api-management/api-management-using-with-internal-vnet

### Policies, segurança, identidade e padrões externos

[16] Microsoft Learn — validate-azure-ad-token policy. https://learn.microsoft.com/azure/api-management/validate-azure-ad-token-policy
[17] Microsoft Learn — Defender for APIs. https://learn.microsoft.com/azure/defender-for-cloud/defender-for-apis-introduction
[18] Microsoft Learn — APIM logging and monitoring options. https://learn.microsoft.com/azure/api-management/observability
[19] Microsoft Learn — Mutual TLS (mTLS). https://learn.microsoft.com/azure/api-management/api-management-howto-mutual-certificates-for-clients
[20] OWASP API Security Top 10 — 2023. https://owasp.org/API-Security/editions/2023/en/0x00-header/
[21] IETF — RFC 9700, Best Current Practice for OAuth 2.0 Security. https://datatracker.ietf.org/doc/rfc9700/
[22] OpenID Foundation — FAPI 1.0 (Financial-grade API). https://openid.net/specs/openid-financial-api-part-1-1_0.html
[23] OpenID Foundation — FAPI 2.0 Security Profile. https://openid.net/specs/fapi-security-profile-2_0.html

### API design, catálogo, produtos e developer experience

[24] Microsoft Learn — API Center registry & metadata. https://learn.microsoft.com/azure/api-center/key-concepts
[25] Microsoft Learn — Defender for APIs detection. https://learn.microsoft.com/azure/defender-for-cloud/defender-for-apis-introduction

### DevOps, APIOps, IaC e qualidade

[26] Microsoft Learn — APIOps with Azure DevOps and GitHub. https://learn.microsoft.com/azure/architecture/example-scenario/devops/automated-api-deployments-apiops

### Operações, observabilidade e monitoramento

[30] Azure Pricing Calculator — APIM, Log Analytics, Defender for APIs, Front Door. https://azure.microsoft.com/pricing/calculator/
[31] Microsoft Learn — Monitor APIM with Azure Monitor. https://learn.microsoft.com/azure/api-management/api-management-howto-use-azure-monitor
[32] Microsoft Learn — Application Insights integration with APIM. https://learn.microsoft.com/azure/api-management/api-management-howto-app-insights

### Landing zone, governança e Azure Policy

[33] Microsoft Learn — Azure Landing Zones design areas. https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-areas
[34] Microsoft Learn — Azure Policy built-in definitions for API Management. https://learn.microsoft.com/azure/api-management/policy-reference
[35] Microsoft Learn — Azure Policy guest configuration and remediation. https://learn.microsoft.com/azure/governance/policy/concepts/guest-configuration

### Regulatório (Brasil)

[27] ANPD — Resolução CD/ANPD nº 15, de 24 de abril de 2024 (regulamento de comunicação de incidente de segurança envolvendo dados pessoais). https://www.gov.br/anpd/pt-br/assuntos/noticias/anpd-publica-regulamento-de-comunicacao-de-incidente-de-seguranca
[28] BACEN — Resolução BCB nº 32, de 29 de outubro de 2020 (Open Banking/Open Finance, requisitos e cronograma). https://www.bcb.gov.br/estabilidadefinanceira/exibenormativo?tipo=Resolu%C3%A7%C3%A3o%20BCB&numero=32
[28b] BACEN — Resolução BCB nº 198, de 11 de março de 2022 (uso de serviços de computação em nuvem por instituições financeiras). https://www.bcb.gov.br/estabilidadefinanceira/exibenormativo?tipo=Resolu%C3%A7%C3%A3o%20BCB&numero=198
[28c] CMN — Resolução CMN nº 4.893, de 26 de fevereiro de 2021 (política de segurança cibernética e requisitos para contratação de serviços de processamento e armazenamento de dados e de computação em nuvem). https://www.bcb.gov.br/estabilidadefinanceira/exibenormativo?tipo=Resolu%C3%A7%C3%A3o%20CMN&numero=4893
[29] Open Finance Brasil — Portal oficial (convenção, manuais técnicos, padrões). https://openfinancebrasil.org.br/
[29b] Open Finance Brasil — Padrões de segurança (FAPI 1.0 perfil brasileiro). https://openfinancebrasil.atlassian.net/wiki/spaces/OF/pages/17378602/Padr+es+T+cnicos

> **Nota sobre URLs regulatórias:** as URLs listadas foram a referência ao tempo desta consolidação. Atos normativos podem ser republicados; **antes de uso em auditoria, a equipe de compliance confirma o ato vigente** (número, ano, redação) na fonte oficial.

---

## 16. Changelog

### 16.1 v2 → v3 (esta versão)

Itens da revisão da versão anterior endereçados:

| Item | Onde | Status |
|---|---|---|
| R9-1 — URLs específicas para [27][28][29] + clarificar Res. CMN 4.893/2021 vs BCB 198/2022 | §2.3.2, §15 | ✅ |
| R10-1 — Validar "2.500 TPS por unit" + tabela "componente × custo unitário típico" | §8.6.1, §8.6.2 (nova) | ✅ |
| R12-1 — Resolver placeholders "(continuação...)" em §15 | §15 (referências [30]-[35] adicionadas) | ✅ |
| R12-2 — Glossário de siglas | Apêndice E (novo) | ✅ |
| R3-1 — Exemplos de policy XML por classe | Apêndice A (novo) | ✅ |
| R1-1 — "Verificar" → nota de rodapé com responsável/prazo | §3.4, §6.6 | ✅ |
| R2-1 — Apêndice OpenAPI esqueleto | Apêndice B (novo) | ✅ |
| R4-1 — Apêndice ADR template | Apêndice C (novo) | ✅ |
| R5-1 — Apêndice pipeline YAML | Apêndice D (novo) | ✅ |
| R6-1 — SLO numérico por classe | §10.4.1 (novo) | ✅ |
| R8-1 — Protocolo de sunset multicanal | §11.4.1 (novo) | ✅ |
| R11-1 — Mapeamento ISO 27001 / NIST CSF | Apêndice F (novo) | ✅ |
| R13-1 — 1-pager executivo | Apêndice G (novo) | ✅ |

### 16.2 v1 → v2 (versão anterior)

Itens da revisão da versão inicial:

#### Alta prioridade
- [A1] Matriz pontuada com critérios eliminatórios → §4.5.
- [A2] Matriz HA/DR × SKU × criticidade → §10.5.
- [A3] Ressalva regional API Center → §5.1, §6.5.
- [A4] LGPD nomeada + direitos do titular + DPO → §2.3.1 + §7.7 + §7.8.
- [A5] Setor regulado (BACEN/Open Finance/FAPI) como classe → §2.3.2 + §7.7.
- [A6] Matriz "classe de API × controles mínimos" → §7.7.
- [A7] FinOps + showback/chargeback + custo observabilidade → §8.6.
- [A8] Modelo de exceção com ciclo de vida formal → §11.5.
- [A9] Quadro executivo "decida em 5 perguntas" → §1.1.

#### Média prioridade
- [M1] Self-service vs aprovado → §11.6.
- [M2] Roadmap com owner + critério de aceite → §13.2.
- [M3] GraphQL/WebSocket/OData/AI gateway integrados → §6.6.
- [M4] Drift detection com ferramentas concretas → §9.4.1.
- [M5] Multi-instância: repos + sincronização → §9.5.
- [M6] Runbooks/playbooks mínimos → §10.6.
- [M7] Threat model por classe + OWASP → §7.6.

#### Baixa prioridade
- [B1] Citações inline em fatos críticos → §1.2, §3.3, §3.4, §6.6.
- [B2] Anti-pattern "escolha tardia de SKU/região" → §12.1 #13.
- [B3] Anti-pattern "governança central sem caminho rápido" → §12.1 #14.

---

## Apêndice A — Exemplos de policy XML por classe (§7.7)

> Os fragmentos abaixo são **referência**. Em produção: usar Policy Fragments, named values do Key Vault para chaves, e validar com lint.

### A.1 Classe "Interna"

```xml
<policies>
  <inbound>
    <base />
    <set-header name="x-correlation-id" exists-action="skip">
      <value>@(context.RequestId)</value>
    </set-header>
    <validate-azure-ad-token tenant-id="{{tenant-id}}">
      <audiences>
        <audience>api://internal-{{api-name}}</audience>
      </audiences>
      <required-claims>
        <claim name="appid" match="any">
          <value>{{allowed-client-appid}}</value>
        </claim>
      </required-claims>
    </validate-azure-ad-token>
    <rate-limit-by-key calls="600" renewal-period="60" counter-key="@(context.Subscription?.Id ?? "anon")" />
  </inbound>
  <backend><base /></backend>
  <outbound>
    <set-header name="Server" exists-action="delete" />
    <set-header name="X-Powered-By" exists-action="delete" />
    <base />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

### A.2 Classe "Externa B2B"

```xml
<policies>
  <inbound>
    <base />
    <ip-filter action="allow">
      <address-range from="{{partner-cidr-start}}" to="{{partner-cidr-end}}" />
    </ip-filter>
    <validate-azure-ad-token tenant-id="{{tenant-id}}">
      <audiences>
        <audience>api://b2b-{{api-name}}</audience>
      </audiences>
    </validate-azure-ad-token>
    <validate-content unspecified-content-type-action="prevent" max-size="262144" size-exceeded-action="prevent">
      <content type="application/json" validate-as="json" action="prevent" />
    </validate-content>
    <rate-limit-by-key calls="120" renewal-period="60" counter-key="@(context.Subscription.Id)" />
    <quota-by-key calls="100000" renewal-period="86400" counter-key="@(context.Subscription.Id)" />
    <set-header name="x-correlation-id" exists-action="skip"><value>@(context.RequestId)</value></set-header>
  </inbound>
  <backend><base /></backend>
  <outbound>
    <set-header name="Server" exists-action="delete" />
    <base />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

### A.3 Classe "Sensível (LGPD)"

```xml
<policies>
  <inbound>
    <base />
    <validate-azure-ad-token tenant-id="{{tenant-id}}">
      <audiences>
        <audience>api://sensitive-{{api-name}}</audience>
      </audiences>
      <required-claims>
        <claim name="scp" match="any">
          <value>{{required-scope}}</value>
        </claim>
      </required-claims>
    </validate-azure-ad-token>
    <validate-content unspecified-content-type-action="prevent" max-size="524288" size-exceeded-action="prevent">
      <content type="application/json" validate-as="json" action="prevent" />
    </validate-content>
    <rate-limit-by-key calls="60" renewal-period="60" counter-key="@(context.Subscription.Id)" />
    <set-variable name="subjectId" value="@((string)context.Request.MatchedParameters.GetValueOrDefault("subjectId",""))" />
    <log-to-eventhub logger-id="audit-lgpd" partition-id="0">
      @{
        return new JObject(
          new JProperty("ts", DateTime.UtcNow.ToString("o")),
          new JProperty("correlationId", context.RequestId),
          new JProperty("operation", context.Operation.Name),
          new JProperty("subjectIdHash", context.Variables.GetValueOrDefault<string>("subjectId").GetHashCode()),
          new JProperty("scopes", context.Request.Headers.GetValueOrDefault("Authorization",""))
        ).ToString();
      }
    </log-to-eventhub>
  </inbound>
  <backend><base /></backend>
  <outbound>
    <set-header name="Cache-Control" exists-action="override"><value>no-store</value></set-header>
    <base />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

### A.4 Classe "Regulada/financeira (Open Finance)"

```xml
<policies>
  <inbound>
    <base />
    <choose>
      <when condition="@(context.Request.Certificate == null || !context.Request.Certificate.VerifyChainWithFAPIRoot())">
        <return-response>
          <set-status code="401" reason="mTLS required for Open Finance" />
        </return-response>
      </when>
    </choose>
    <validate-jwt header-name="Authorization" require-scheme="Bearer">
      <openid-config url="{{openfinance-issuer-discovery}}" />
      <audiences><audience>{{openfinance-audience}}</audience></audiences>
      <required-claims>
        <claim name="cnf" match="all" />
        <claim name="scope" match="any">
          <value>{{openfinance-scope}}</value>
        </claim>
      </required-claims>
    </validate-jwt>
    <set-header name="x-fapi-interaction-id" exists-action="skip">
      <value>@(context.RequestId)</value>
    </set-header>
    <rate-limit-by-key calls="300" renewal-period="60" counter-key="@(context.Subscription.Id)" />
    <log-to-eventhub logger-id="audit-openfinance" partition-id="0">
      @{
        return new JObject(
          new JProperty("ts", DateTime.UtcNow.ToString("o")),
          new JProperty("interactionId", context.RequestId),
          new JProperty("op", context.Operation.Name),
          new JProperty("client", context.Request.Certificate?.SubjectName?.Name)
        ).ToString();
      }
    </log-to-eventhub>
  </inbound>
  <backend>
    <forward-request fail-on-error-status-code="true" />
  </backend>
  <outbound>
    <base />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

> **Nota:** `VerifyChainWithFAPIRoot()` é placeholder; em produção usar `authentication-certificate` + validação na cadeia ICP do Open Finance Brasil [29b].

### A.5 Classe "Administrativa"

```xml
<policies>
  <inbound>
    <base />
    <ip-filter action="allow">
      <address-range from="{{admin-vnet-start}}" to="{{admin-vnet-end}}" />
    </ip-filter>
    <validate-azure-ad-token tenant-id="{{tenant-id}}">
      <required-claims>
        <claim name="roles" match="any">
          <value>APIMAdmin</value>
        </claim>
      </required-claims>
    </validate-azure-ad-token>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
```

### A.6 Classe "Pública não autenticada"

```xml
<policies>
  <inbound>
    <base />
    <rate-limit calls="60" renewal-period="60" />
    <quota calls="100000" renewal-period="86400" />
    <cors allow-credentials="false">
      <allowed-origins><origin>*</origin></allowed-origins>
      <allowed-methods><method>GET</method></allowed-methods>
    </cors>
  </inbound>
  <backend><base /></backend>
  <outbound>
    <set-header name="Cache-Control" exists-action="override"><value>public, max-age=60</value></set-header>
    <base />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

---

## Apêndice B — Esqueleto OpenAPI corporativo

```yaml
openapi: 3.0.3
info:
  title: "{nome-da-api}"
  version: "1.0.0"
  description: |
    {descrição curta}
    - Owner: {squad / lead / on-call}
    - Classe (§7.7): {interna | externa-b2b | sensivel | regulada | administrativa | publica}
    - Base legal LGPD: {n/a | art. 7º §X}
    - Retenção mínima de log: {90d | 1a | 2a | 5a | 7a}
  contact:
    name: "Time {squad}"
    email: "{squad}@empresa.com"
  license:
    name: "Uso interno corporativo"

servers:
  - url: "https://api.empresa.com/{produto}/v1"
    description: Produção

security:
  - bearerAuth: []

tags:
  - name: "{recurso}"
    description: "Operações sobre {recurso}"

paths:
  /{recurso}:
    get:
      tags: ["{recurso}"]
      summary: "Lista {recurso}"
      operationId: "list{Recurso}"
      parameters:
        - in: query
          name: limit
          schema: { type: integer, default: 20, maximum: 100 }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/{Recurso}List"
        "400": { $ref: "#/components/responses/Problem" }
        "401": { $ref: "#/components/responses/Problem" }
        "429": { $ref: "#/components/responses/Problem" }
        "500": { $ref: "#/components/responses/Problem" }

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    Problem:
      type: object
      description: "RFC 7807 application/problem+json"
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        detail: { type: string }
        instance: { type: string }
        correlationId: { type: string }
    "{Recurso}":
      type: object
      properties:
        id: { type: string, format: uuid }
        nome: { type: string, maxLength: 200 }
    "{Recurso}List":
      type: object
      properties:
        items:
          type: array
          items: { $ref: "#/components/schemas/{Recurso}" }
        total: { type: integer }
  responses:
    Problem:
      description: "Erro padronizado (RFC 7807)"
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/Problem" }
```

---

## Apêndice C — Template de ADR (Architecture Decision Record)

```markdown
# ADR-APIM-{NNN}: {Título da decisão}

- **Status:** Proposta | Aceita | Substituída por ADR-APIM-{MMM} | Obsoleta
- **Data:** AAAA-MM-DD
- **Decisores:** {nome — papel}, {nome — papel}
- **Consultados:** {DPO, Segurança, FinOps, Auditoria, …}
- **Informados:** {Squads consumidoras, Plataforma}

## 1. Contexto
{Problema, restrições, requisitos regulatórios, prazo. Citar drivers de §2.3 quando aplicável.}

## 2. Opções avaliadas
- Opção A: {descrição} — prós/contras
- Opção B: {descrição} — prós/contras
- Opção C: {descrição} — prós/contras

## 3. Decisão
{Opção escolhida e justificativa enxuta.}

## 4. Consequências
- Positivas
- Negativas
- Riscos aceitos
- Riscos mitigados

## 5. Critérios de revalidação
- Eventos que disparam revisão (mudança regulatória, disponibilidade regional, mudança de SKU, métrica X estourando)
- Frequência mínima de revisão: trimestral / semestral / anual.

## 6. Mapeamento RACI
| Atividade | Squad | Plataforma | Segurança | DPO/Compliance |
|---|---|---|---|---|
| Implementação | R | A | C | I |
| Monitoramento | R | R | C | I |

## 7. Referências
- §X do guia APIM corporativo (v3)
- [n] da §15
- Documentos internos
```

---

## Apêndice D — Fragmento de pipeline (GitHub Actions YAML)

```yaml
name: apim-ci

on:
  pull_request:
    paths:
      - "apis/**"
      - "policies/**"
      - "bicep/**"

permissions:
  contents: read
  id-token: write

jobs:
  lint-openapi:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Spectral lint OpenAPI
        uses: stoplightio/spectral-action@v0.8.11
        with:
          file_glob: "apis/**/*.yaml"
          spectral_ruleset: ".spectral.yaml"

  lint-policy-xml:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: xmllint policies
        run: |
          sudo apt-get update && sudo apt-get install -y libxml2-utils
          find policies -name "*.xml" -print0 | xargs -0 -I{} xmllint --noout "{}"
      - name: Policy linter customizado
        run: ./scripts/policy-lint.sh policies/

  validate-class-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verifica ficha de intake (classe regulatória)
        run: ./scripts/check-intake-class.sh apis/

  bicep-what-if:
    runs-on: ubuntu-latest
    needs: [lint-openapi, lint-policy-xml, validate-class-gate]
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: az deployment what-if
        run: |
          az deployment group what-if \
            --resource-group rg-apim-corp \
            --template-file bicep/main.bicep \
            --parameters @bicep/parameters/dev.json

  drift-report:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: APIOps Toolkit extractor
        run: ./tools/apiops-extractor.sh > artifacts/extracted-config/
      - name: Compare to repo
        run: diff -r apis/ artifacts/extracted-config/ > drift-report.txt || true
      - name: Upload drift report
        uses: actions/upload-artifact@v4
        with:
          name: drift-report
          path: drift-report.txt
```

---

## Apêndice E — Glossário

| Sigla | Expansão | Observação |
|---|---|---|
| ADR | Architecture Decision Record | Registro de decisão arquitetural; ver Apêndice C |
| ANPD | Autoridade Nacional de Proteção de Dados | Autoridade brasileira de LGPD |
| AKS | Azure Kubernetes Service | Para self-hosted gateway |
| API | Application Programming Interface | — |
| BACEN | Banco Central do Brasil | — |
| BCP | Business Continuity Plan | Plano de continuidade |
| CIDR | Classless Inter-Domain Routing | Notação de bloco IP |
| CMN | Conselho Monetário Nacional | Emite resoluções complementares ao BACEN |
| CORS | Cross-Origin Resource Sharing | Política HTTP |
| DCR | Dynamic Client Registration | Registro dinâmico de cliente OAuth (RFC 7591) |
| DNS | Domain Name System | — |
| DPO | Data Protection Officer / Encarregado | Papel exigido pela LGPD |
| DR | Disaster Recovery | Continuidade pós-incidente |
| DX | Developer Experience | Experiência do desenvolvedor |
| FAPI | Financial-grade API | Perfil de segurança OpenID para finanças (1.0 e 2.0) |
| FAPI 1.0 | Versão 1.0 do FAPI | Usada pelo Open Finance Brasil |
| FinOps | Cloud Financial Operations | Disciplina de governança de custo em nuvem |
| HMAC | Hash-based Message Authentication Code | Para integridade |
| IaC | Infrastructure as Code | Bicep, Terraform, etc. |
| ICP | Infraestrutura de Chaves Públicas | Cadeia ICP-Brasil ou ICP do Open Finance |
| JAR | JWT-Secured Authorization Request | RFC 9101 |
| JWS | JSON Web Signature | RFC 7515 |
| JWT | JSON Web Token | RFC 7519 |
| LGPD | Lei Geral de Proteção de Dados (Lei 13.709/2018) | Brasil |
| M2M | Machine-to-Machine | OAuth client credentials |
| MAP | Microsoft Architecture Plan | Intake / blueprint |
| MAU | Monthly Active Users | Métrica de uso |
| MCP | Model Context Protocol | Protocolo para ferramentas de IA |
| MPAS | Microsoft Partner Architectural Solution | Modelo de proposta |
| mTLS | Mutual TLS | Autenticação cliente+servidor por certificado |
| NIST CSF | NIST Cybersecurity Framework | Padrão americano de cibersegurança |
| OAuth | Open Authorization | RFC 6749 + RFC 9700 (BCP) |
| OIDC | OpenID Connect | Camada sobre OAuth 2.0 |
| OWASP | Open Web Application Security Project | — |
| PCI-DSS | Payment Card Industry Data Security Standard | — |
| PII | Personally Identifiable Information | Dados pessoais |
| PIM | Privileged Identity Management | Azure AD/Entra ID |
| PKCE | Proof Key for Code Exchange | RFC 7636, para clientes públicos |
| RACI | Responsible / Accountable / Consulted / Informed | Matriz de responsabilidades |
| RBAC | Role-Based Access Control | Azure RBAC |
| RIPD | Relatório de Impacto à Proteção de Dados Pessoais | Exigido pela LGPD em certos casos |
| RPO | Recovery Point Objective | Janela máxima de perda de dado |
| RTO | Recovery Time Objective | Janela máxima de recuperação |
| SIEM | Security Information and Event Management | Ex.: Microsoft Sentinel |
| SKU | Stock Keeping Unit | Tier do APIM (Basic/Standard/Premium etc.) |
| SLA | Service Level Agreement | Contrato com consumidor |
| SLO | Service Level Objective | Meta operacional interna |
| SOC | Security Operations Center | Time de operações de segurança |
| TLS | Transport Layer Security | — |
| TPS | Transactions Per Second | — |
| VNet | Virtual Network | Azure |
| WAF | Web Application Firewall | Application Gateway / Front Door |

---

## Apêndice F — Mapeamento ISO 27001 / NIST CSF → controles APIM

Mapeamento ilustrativo (não exaustivo) para apoiar conversa com auditoria/compliance:

| Controle ISO 27001:2022 (Anexo A) | NIST CSF 2.0 | Onde aterrissa em APIM |
|---|---|---|
| A.5.7 Threat intelligence | ID.RA-02 | §7.6 threat model + Defender for APIs [25] |
| A.5.15 Access control | PR.AA-01 | §7.1, §7.2, §7.7 |
| A.5.18 Access rights provisioning | PR.AA-05 | RBAC §8.3 + PIM |
| A.5.23 Use of cloud services | GV.SC-04 | §8.1, §13.2, ADRs |
| A.5.30 ICT readiness for business continuity | RC.RP-01 | §10.5 HA/DR + §10.6 runbooks |
| A.6.7 Remote working | PR.AA-04 | §7.1 PIM + Entra ID |
| A.8.2 Privileged access rights | PR.AA-05 | PIM + custom roles |
| A.8.5 Secure authentication | PR.AA-01 | §7.2 OAuth/OIDC, FAPI, mTLS |
| A.8.9 Configuration management | PR.PS-01 | §9 APIOps + drift §9.4.1 |
| A.8.15 Logging | DE.AE-03 | §10.1, §10.2, §7.8 |
| A.8.16 Monitoring activities | DE.CM-01 | Sentinel + Defender for APIs |
| A.8.20 Networks security | PR.IR-01 | §8.5 rede privada, VNet |
| A.8.21 Security of network services | PR.IR-01 | WAF, Private Link |
| A.8.23 Web filtering | PR.PT-04 | Front Door + WAF |
| A.8.24 Use of cryptography | PR.DS-02 | mTLS, TLS, Key Vault |
| A.8.26 Application security requirements | PR.PS-06 | §7.5 policies; §7.7 matriz vinculante |
| A.8.28 Secure coding | PR.PS-06 | Lint OpenAPI + policies |
| A.8.29 Security testing in development & acceptance | PR.PS-06 | §9.3 gates |
| A.8.31 Separation of development, test and production | PR.PS-01 | §12.1 #12 anti-pattern |
| A.8.32 Change management | PR.PS-01 | §11.6 self-service vs governado |
| A.8.34 Protection during audit & testing | PR.PS-02 | §11.5 exceções + evidências |

> Para mapear controles a evidências reais: a cada controle, registrar em sistema GRC o link para PR/issue/runbook/relatório que demonstra implementação.

---

## Apêndice G — 1-pager executivo

> Para reunião de patrocínio / steerco. Imprimir 1 página.

### Por que APIM corporativo agora

APIM é o ponto técnico onde residência (LGPD/BACEN), exposição (parceiros/internet), segurança (mTLS, OAuth, FAPI), catálogo e governança de APIs ganham forma operacional. Sem ele padronizado, cada domínio resolve à sua maneira → fragmentação, custo, risco regulatório.

### A decisão central (5 perguntas — §1.1)

1. Residência Brasil obrigatória? → hoje, Premium clássico (Premium v2 não listado em Brazil South [1][3]).
2. API regulada (Open Finance/LGPD sensível/PCI)? → classe regulada com FAPI/mTLS (§7.7).
3. Externa ou interna? → WAF obrigatório para externa.
4. Domínio com autonomia/SLO próprio? → instância dedicada (modelo C).
5. Multi-region ativo / self-hosted gateway? → Premium clássico mandatório.

### Recomendação

**Modelo C — híbrido governado**: APIM corporativo padrão + instâncias dedicadas por exceção justificada (§4.5). Plataforma compartilhada por padrão; dedicada por gate.

### Os 6 "verdes" que precisamos para entregar

1. ADR de modelo + SKU + região (30 dias).
2. APIM corporativo via IaC + Azure Policy guardrails (60 dias).
3. Pipeline APIOps com gate de classe regulatória (90 dias).
4. Threat model por classe (90 dias).
5. DR ensaiado por criticidade (180 dias).
6. Self-service ≥ 70 % dos PRs (240 dias).

### Custos e riscos chave

- Driver dominante de custo: Log Analytics ingestion + Defender for APIs + multi-region unit (§8.6).
- Risco regulatório principal: classe regulada (Open Finance) sem mTLS / sem retenção 5+ anos.
- Risco de execução principal: governança central sem caminho rápido → §11.6.

### Decisões pendentes do cliente (§14)

Residência, classes do roadmap, aprovadores de exceção, owner FinOps, RTO/RPO formal, política de retenção.
