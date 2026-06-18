# Plataforma APIM corporativa em Azure — guia técnico de decisão e implantação

---

## 1. Sumário executivo

API Management (APIM) em Azure é a peça central de uma estratégia de APIs corporativa: ponto único de exposição governada, autenticação, autorização, throttling, observabilidade, catálogo e contrato com produtores e consumidores. Ele não substitui a engenharia das APIs; ele formaliza o contorno entre quem produz, quem governa e quem consome.

Para uma corporação grande, três decisões dominam o desenho: **modelo de adoção** (corporativo, por domínio, ou híbrido governado), **SKU/tier** (clássico v1, v2 ou Premium v2) e **arquitetura de rede/isolamento** (público com WAF, interno via VNet, workspaces com isolamento lógico e/ou runtime dedicado por workspace gateway). A escolha precisa ser sustentada por requisitos regulatórios, residência, modelo de exposição e maturidade de plataforma — não por preferência tática.

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

Disponibilidade regional **muda a recomendação**: o que vale para US East/West Europe pode não valer para Brazil South. No momento desta consolidação (referência **maio/2026**, revalidada contra docs Microsoft de junho/2026), em Brazil South estão listados **Basic v2 e Standard v2**; **Premium v2 não consta na matriz pública** consultada [1][3]. Workspaces não são exclusivos do Premium clássico: existem em **Basic v2, Standard v2, Premium e Premium v2**; já **workspace gateway dedicado** é um runtime separado, com disponibilidade regional/custo próprios, e deve ser confirmado antes de ser assumido [1][5]. Premium clássico continua relevante para VNet injection, multi-region e self-hosted gateway, recursos não cobertos por Premium v2.

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
- Esquecer integração entre APIM, Microsoft Entra ID, Azure Policy, Defender for Cloud, observabilidade e pipelines.

### 2.1 Premissas e escopo

- O cliente é uma empresa grande, com múltiplos domínios, parceiros, requisitos de regulação (LGPD, possivelmente setor financeiro/Open Finance, ANPD, BACEN) e times distribuídos.
- Há Azure Landing Zone (ou equivalente) com hub/spoke, Azure Policy, Log Analytics, Sentinel e Microsoft Entra ID.
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

Capacidades centrais: policies declarativas (XML), produtos, APIs, operações, named values, certificados, identidades gerenciadas, integração com Azure Monitor/Log Analytics/Application Insights, Defender for APIs, Azure Policy. Em SKUs habilitados, também: VNet integration, multi-region, self-hosted gateway, **workspaces** (isolamento lógico em Basic v2, Standard v2, Premium e Premium v2) e **workspace gateways** (runtime dedicado/isolado quando disponível), Developer Portal customizado e API Center como inventário/catálogo corporativo [1][5].

### 3.2 Terminologia: clássico/v1, stv1/stv2, Consumption e tiers v2

Para evitar confusão recorrente:

- **APIM clássico (v1)** — tiers Developer, Basic, Standard, Premium e Consumption. Arquitetura sobre máquina virtual (stv2 atual; stv1 já aposentado conforme [4]).
- **APIM v2** — nova geração, arquitetura serverless, tiers **Basic v2**, **Standard v2** e **Premium v2** (em rollout regional). Trazem mudanças importantes em rede, escalabilidade, custos e recursos [1].
- **Consumption** — tier pay-per-call, baseado em clássico. Adequado para cargas leves/serverless, com limitações (sem VNet integrada, sem self-hosted gateway, etc.).
- **Workspaces** — limite **lógico** de governança/federação dentro de uma instância APIM. Na documentação atual, workspaces se aplicam a **Basic v2, Standard v2, Premium clássico e Premium v2** [1][5]. Um workspace contém APIs, produtos, subscriptions, named values e recursos relacionados, com RBAC escopado ao workspace.
- **Gateway associado ao workspace** — limite de **runtime**. Um workspace pode usar o **default managed gateway** da instância (menor custo, menor isolamento) ou um **workspace gateway** separado (recurso dedicado, escala/hostname/rede próprios, custo e disponibilidade regional próprios) quando suportado [5].

> Ponto crítico de revisão: **workspace ≠ workspace gateway ≠ múltiplas instâncias**. Workspace resolve ownership, RBAC e catálogo federado; workspace gateway resolve isolamento de runtime; múltiplas instâncias resolvem isolamento de blast radius, região, SKU, quota e operação. Não justificar Premium clássico apenas porque “precisa de workspace”; justificar Premium clássico quando o requisito for multi-region nativo, self-hosted gateway, backup/restore ou VNet/maturidade operacional específica [1][5].

### 3.3 Orientação por SKU/tier

| Tier | Quando faz sentido | Quando NÃO faz sentido |
|------|--------------------|------------------------|
| Developer | Sandbox, testes, treinamento, self-hosted gateway apenas para avaliação | Produção (sem SLA) |
| Basic v2 | Produção leve, workspaces lógicos, custo previsível | Regulado crítico, multi-region, self-hosted gateway, VNet injection |
| Standard v2 | Produção média, workspaces lógicos, outbound VNet integration quando disponível, autoscale | Multi-region nativo, self-hosted gateway, backup/restore |
| Premium v2 | Produção crítica (onde listado regionalmente), capacidade alta, VNet injection, múltiplos custom domains no gateway, workspaces | Multi-region nativa, self-hosted gateway, backup/restore — não cobertos hoje [1][2] |
| Premium clássico | Multi-region, VNet integrada, self-hosted gateway, backup/restore, múltiplos custom domains no gateway, workspaces | Cargas leves; o custo não se justifica para escopo reduzido |
| Consumption | APIs internas leves, eventos esporádicos, prototipagem | Cargas com policies pesadas, latência crítica, requisitos VNet, WebSocket |

**Decisão SKU resumida:**
- Cenário híbrido com self-hosted gateway → **Premium clássico** em produção (Developer só avaliação), porque self-hosted gateway se aplica a Developer/Premium e Premium v2 não cobre [1][8].
- Multi-region ativo/ativo nativo → **Premium clássico**.
- Federação por times sem runtime dedicado → **workspaces com default managed gateway** em Basic v2/Standard v2/Premium/Premium v2, validando região e limites [1][5].
- Federação por times com runtime isolado → **workspace gateway** quando listado para o SKU/região; tratar como recurso/custo independente e validar em ADR [5].
- Carga corporativa moderna, sem necessidade de multi-region/self-hosted/backup-restore, na região onde Premium v2 está listado → **Premium v2** ou Standard v2.
- Brasil + requisito de VNet injection hoje → **Premium clássico** até Premium v2 estar listado em Brazil South; Brasil + multi-region nativo → **Premium clássico** [1][3].

### 3.4 Disponibilidade regional e nota específica sobre Brasil

A disponibilidade regional **muda decisões** e precisa ser checada antes de qualquer commitment. Referência atual consultada em **maio/2026** e revalidada contra docs Microsoft publicadas em **junho/2026** [1][3][5]:

- **Brazil South:** Basic v2 e Standard v2 listados. **Premium v2 não listado** na matriz consultada. Workspaces são recurso de Basic v2/Standard v2/Premium/Premium v2, mas **workspace gateway dedicado** deve ser confirmado na região antes de entrar no desenho.
- **Brazil Southeast:** reduzido em recursos como nas demais regiões "Southeast" do Azure. Confirmar antes de assumir multi-region Brasil ativo/ativo.

Implicações:

1. Se a corporação tem **residência obrigatória em Brasil** (LGPD, BACEN, política interna) e precisa de Premium v2, a decisão não pode assumir disponibilidade sem evidência pública ou confirmação Microsoft. Premium clássico é a rota conservadora para VNet injection/multi-region/self-hosted.
2. Se a corporação aceita **multi-region cross-border** (Brasil + região secundária fora), Premium clássico pode aceitar multi-region; Premium v2 não tem multi-region nativo.
3. API Center (catálogo corporativo) também tem **disponibilidade regional limitada** [10]; pode ser que precise rodar em região secundária e referenciar APIs hospedadas no Brasil — isso é aceitável desde que metadados não contenham dados sensíveis.
4. Workspaces não resolvem residência por si só: eles isolam ownership e, quando associados a workspace gateway dedicado, podem isolar runtime. Região, rede e dados continuam sendo decisão de instância/gateway.

Tabela de disponibilidade — referência **maio/2026** (verificar [1], [3] e [5] antes de fechar arquitetura):

| Recurso | Brazil South | Brazil Southeast¹ | East US | West Europe |
|---|---|---|---|---|
| Basic v2 | Disponível | Verificar¹ | Disponível | Disponível |
| Standard v2 | Disponível | Verificar¹ | Disponível | Disponível |
| Premium v2 | **Não listado** | Não listado | Disponível | Disponível |
| Workspaces (isolamento lógico) | Disponível quando o SKU suporte estiver disponível | Verificar¹ | Disponível nos SKUs suportados | Disponível nos SKUs suportados |
| Workspace gateway dedicado | Verificar¹; não assumir sem evidência regional | Verificar¹ | Disponível conforme SKU/região | Disponível conforme SKU/região |
| Premium clássico (VNet, multi-region, self-hosted) | Disponível | Verificar¹ | Disponível | Disponível |
| API Center | Verificar¹ | Não listado | Disponível | Disponível |

¹ Itens marcados "Verificar" são revalidados **trimestralmente** pelo time de plataforma (owner: lead de arquitetura Azure) na matriz oficial [1][3][5]; última confirmação registrada em ADR-APIM-001.

> Recomendação: registrar a data da consulta e o link da matriz oficial na ADR. Revisar a cada trimestre.

---

## 4. Modelos de adoção (corporativo, por subscrição/domínio, híbrido) com matriz de decisão

Empresas grandes oscilam entre três modelos. Não há "certo absoluto" — há melhor encaixe para cada combinação de regulação, escala, autonomia, governança e maturidade.

### 4.1 Modelo A — APIM corporativo compartilhado

Uma (ou poucas) instância(s) APIM, operada(s) pela plataforma, expondo APIs de múltiplos domínios. **Workspaces** (Basic v2, Standard v2, Premium clássico ou Premium v2) segmentam ownership/RBAC/catálogo; **produtos** segmentam consumo/comercialização; **workspace gateways** segmentam runtime quando o isolamento dedicado é exigido e disponível [1][5].

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

Combina APIM como entry plane com Azure Landing Zone, Microsoft Entra ID, observabilidade central e pipelines IaC.

### 5.1 Componentes recomendados

- **APIM** (clássico Premium ou v2 conforme §3.3/§3.4).
- **Application Gateway/WAF** ou **Azure Front Door com WAF** quando o APIM é exposto à internet. APIM nunca diretamente sem WAF para externo.
  - **Use Azure Front Door + WAF** quando o requisito é entrada global, multi-região, failover global, CDN/cache/edge, aceleração de tráfego e proteção no perímetro global [33].
  - **Use Application Gateway WAF v2** quando o requisito é entrada regional, integração forte com VNet, APIM interno, roteamento L7 por path/host dentro da landing zone, Private Link/backend privado e inspeção regional [15][33].
  - **Use ambos** quando há perímetro global + inspeção/regra regional: Front Door global → Application Gateway WAF regional → APIM interno. Evite dupla terminação TLS sem justificativa e documente headers, probes e health checks.
- **Microsoft Entra ID** para autenticação humana (publishers, devs) e workload identities.
- **Key Vault** para segredos, certificados e chaves de assinatura/encriptação.
- **Log Analytics + Application Insights** para sinais de runtime; **Sentinel** para correlação de segurança.
- **Defender for APIs** (Defender for Cloud) para detecção de anomalias e shadow APIs [25].
- **API Center** [10] como inventário corporativo e catálogo unificado. **Ressalva:** API Center tem disponibilidade regional limitada; conferir se a região exigida tem o serviço — caso não, hospedar em região secundária e referenciar APIs do Brasil é aceitável, desde que metadados não contenham dados sensíveis.
- **Azure Policy** para guardrails (naming, tagging, SKU permitidos, criptografia, region restriction).
- **Bicep/Terraform + GitHub Actions/Azure DevOps** para IaC e pipelines APIOps.

### 5.2 Padrões de integração

- **APIM → backend** sempre em rede privada quando o backend está em Azure (VNet integration, Private Link, Service Endpoint). Backends externos: HTTPS + mTLS quando crítico.
- **APIM ↔ Microsoft Entra ID** via OAuth/OIDC (JWT validation policy), com discovery e cache de chaves.
- **APIM ↔ Defender for APIs** para inventário e detecção; integrado nativamente nos SKUs suportados [25].
- **APIM ↔ Log Analytics** com diagnostic settings completos (gateway logs, audit logs).
- **APIM ↔ API Center** para sincronizar inventário [10][24].
- **APIM ↔ Sentinel** via Log Analytics como pipeline padrão para SIEM.

### 5.3 Diagrama lógico (descrito)

```
[Consumidor externo]
       │
       ▼
[Front Door + WAF]  (global/multi-region/CDN/edge)
       │
       ▼
[Application Gateway WAF] (opcional: regional/VNet/L7)
       │
       ▼
[APIM managed gateway]
       │
       ├──► [App Service / AKS / Functions] (backends Azure, via Private Link/VNet)
       └──► [API Center] (catálogo corporativo, sincronização)

[Consumidor on-prem / filial / outra cloud]
       │
       ▼
[Self-hosted gateway] (container federado ao control plane APIM)
       │
       └──► [Backends on-prem / outras clouds]

[Plano de gestão & observabilidade]
   Microsoft Entra ID  ↔  Key Vault  ↔  Log Analytics  ↔  Sentinel  ↔  Defender for APIs
          └── configura/observa APIM managed gateway e self-hosted gateways
```

**Correção arquitetural importante:** self-hosted gateway **não é backend downstream** do gateway gerenciado. Ele é um runtime alternativo do data plane, implantado perto dos backends híbridos/multicloud, associado ao mesmo control plane do APIM para configuração, policies, catálogo e telemetria [8]. O tráfego do consumidor deve entrar no gateway adequado ao cenário; não se deve desenhar “managed gateway → self-hosted gateway” como caminho padrão.

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

- **Versão** para mudanças incompatíveis. O default corporativo é **path versioning** (`/v1`, `/v2`) por legibilidade, roteamento simples, cache/WAF previsíveis e menor ambiguidade operacional.
- **Revision** para mudanças compatíveis dentro da mesma versão. Permite rollback rápido sem mudar contrato público.
- **Política de deprecação mínima**: comunicação 90 dias antes; banner no Developer Portal; e-mail aos subscribers; remoção após sunset.

Trade-offs obrigatórios em ADR quando o padrão `/v{n}` não for usado:

| Estratégia | Vantagens | Riscos / custo operacional | Quando aceitar |
|---|---|---|---|
| **Path** (`/v1/pagamentos`) | Claro para consumidores; fácil roteamento no APIM/WAF; bom para documentação, cache, analytics e coexistência | “Polui” URL; pode incentivar versionamento excessivo | **Default** para APIs públicas, B2B, mobile e reguladas |
| **Query** (`?api-version=1`) | Útil quando path é imposto por legado; fácil testar side-by-side | Cache/proxy podem ignorar query; consumidores esquecem parâmetro; menos explícito em logs | APIs internas/legadas com contrato já estabilizado e gateway impondo default/validação |
| **Header** (`api-version: 1`) | Preserva URL; permite negociação avançada | Menos visível; difícil para browsers/CDN/WAF; maior chance de erro em suporte | APIs internas entre serviços maduros, com SDK controlado e observabilidade preparada |

Regra: mudança incompatível exige nova **versão**; mudança compatível usa **revision**. Não usar versionamento por query/header para esconder breaking change.

### 6.5 Produtos, catálogo e consumo

- **Produtos** agrupam APIs e definem policies de acesso (quotas, rate limit, escopos OAuth, termos de uso).
- **Anti-pattern proibido:** produto genérico **"All APIs"**, **"Unlimited"** ou equivalente em produção. Ele elimina ownership, blast radius, chargeback e governança de quota. Exceção só em sandbox temporário com data de expiração.
- **Developer Portal** customizado para a corporação: branding, fluxo de onboarding self-service, gestão de subscriptions, exemplos, sandbox.
- **API Center** [10] como visão executiva e auditoria: inventário, classificações, owners, status, regulatório. **Ressalva regional:** se API Center não está disponível na região corporativa, hospedar em região secundária; conferir trimestralmente.
- **Sandbox** segregado: ambiente onde devs externos/parceiros testam sem credenciais de produção.

Metadados mínimos por **API** (registrados no OpenAPI/API Center/intake):

| Campo | Obrigatório | Uso |
|---|---|---|
| `owner` / `apiProductOwner` | Sim | Responsável técnico e decisor de roadmap |
| `supportContact` / `oncall` | Sim | Incidente e comunicação com consumidores |
| `slaTier` e SLO | Sim | Gate de arquitetura, alertas e contrato de consumo |
| `dataClassification` / `regulatoryClass` | Sim | LGPD/BACEN/PCI, retenção e aprovação |
| `lifecycleState` | Sim | Design, active, deprecated, sunset |
| `costAllocation` / `costcenter` | Sim | Showback/chargeback |
| `mpasMapId` | Sim | Rastreabilidade com intake/ADR |

Metadados mínimos por **produto**: owner do produto, APIs incluídas, consumidores-alvo, termos de uso, quotas/rate limits, SLA/SLO, suporte, classificação máxima permitida, custo/showback, data de revisão.

Quota não é só técnica; é por **jornada/consumidor**. Ex.: “consulta saldo parceiro A” pode ter 100 rpm e 1M/mês, enquanto “inicia pagamento parceiro A” tem 20 rpm, quota diária menor, mTLS obrigatório e aprovação manual. O gate de produto rejeita publicação sem quota/rate-limit explícitos por consumidor, produto ou claim.

### 6.6 Protocolos não-REST e AI gateway

Além de REST/SOAP, APIM cobre outros protocolos. Cada um tem suporte distinto por SKU/região — confirmar antes de assumir.

| Protocolo / capacidade | Suportado em | Limitações | Quando faz sentido |
|---|---|---|---|
| **GraphQL pass-through** | **Todos os tiers APIM** [1] | Encaminha queries/mutations/subscriptions para backend GraphQL existente; usar `validate-graphql-request` para max-size, max-depth e autorização por paths [18][31] | Fachada para backend GraphQL existente, com governança APIM |
| **GraphQL synthetic** | Todos os tiers APIM, com ressalvas [1] | Synthetic GraphQL não é suportado em workspaces; synthetic subscriptions estão em preview e não disponíveis no Consumption; resolvers HTTP/Cosmos DB/Azure SQL exigem governança própria | Criar schema GraphQL sobre APIs legadas sem reescrever backend |
| **WebSocket APIs** | **Developer, Basic, Basic v2, Standard, Standard v2, Premium, Premium v2**; não Consumption [1] | Pass-through; policies só no `onHandshake`; conexões 1:1 não são distribuídas entre múltiplos backends; CORS/validate-content e várias policies não se aplicam ao handshake | Streaming client↔server, cotações, notificações persistentes |
| **OData passthrough** | Premium clássico, v2² | Validação OData não nativa | APIs legadas SAP/Dynamics |
| **gRPC** | Em rollout/limitado² | Verificar antes de comitar | APIs internas alta performance |
| **AI gateway (token rate limit / semantic caching)** | GA em SKUs selecionados [12] | Custo cresce com tokens; semantic cache requer Azure Cache for Redis Enterprise | APIs que fazem fan-out para Azure OpenAI ou modelos próprios |
| **MCP server gateway** | Preview/rollout [12] | Atenção a versão de protocolo e SDK | Ferramentas internas Copilot/agents corporativos |

² Suporte por SKU/região varia por release; revalidação **trimestral** pelo time de plataforma (owner: lead de APIs) contra a matriz oficial [1][12]; última confirmação registrada em ADR-APIM-001.

Controles específicos:
- **GraphQL público/B2B**: `validate-graphql-request` com `max-size`, `max-depth`, bloqueio de introspection (`/__*`) quando aplicável, allowlist/denylist de paths sensíveis, quota por consumer e logs de query hash — não logar query completa com PII [18][31].
- **GraphQL pass-through vs synthetic**: pass-through preserva backend e reduz transformação; synthetic aumenta acoplamento ao APIM e exige testes de resolvers, cache e autorização de campo.
- **Subscriptions GraphQL/WebSocket**: definir timeout, quota por conexão, quota por mensagem/evento, limite de conexões por consumer e runbook de desconexão.
- **Query allowlist**: para APIs públicas/reguladas, preferir persisted queries/allowlist por hash quando o conjunto de queries é conhecido.

Decisão APIM vs integração assíncrona:

| Necessidade | Use APIM quando… | Use Service Bus / Event Grid / Event Hubs / iPaaS quando… |
|---|---|---|
| Request/response síncrono | Consumidor espera resposta imediata, contrato HTTP, auth/throttling/catálogo | Não se aplica |
| Comando assíncrono | Precisa só aceitar comando HTTP e retornar `202 Accepted` com correlation ID | Precisa fila, retry, DLQ, ordenação, backpressure e consumidor desacoplado (Service Bus) |
| Evento de domínio | APIM pode publicar fachada de consulta/status | Precisa fan-out reativo, roteamento por evento e assinatura de múltiplos consumidores (Event Grid) |
| Telemetria/stream | APIM só protege endpoint de ingestão leve | Alto volume, particionamento, retenção curta e processamento streaming (Event Hubs) |
| Integração SaaS/B2B | APIM expõe contrato externo governado | Transformações/processos longos/conectores gerenciados são centrais (iPaaS/Logic Apps) |

Regra: APIM governa **contratos síncronos** e fachadas HTTP. Não usar APIM como fila, barramento, orquestrador de processo longo ou broker de eventos.

---

## 7. Segurança, identidade e políticas

### 7.1 Plano de gestão

- **Microsoft Entra ID** para todo acesso ao plano de gestão (portal, ARM, REST API).
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

APIM faz **autorização grossa** no perímetro: valida identidade, emissor, audience, escopos/roles/claims, produto/subscription, quota e políticas de acesso por API/operação. Isso reduz superfície e bloqueia chamadas obviamente inválidas, mas **não substitui autorização de negócio no backend**.

- **BOLA (Broken Object Level Authorization / OWASP API1)**: checar se o sujeito pode acessar aquele objeto específico (`accountId`, `customerId`, `consentId`) é responsabilidade do backend/sistema de domínio.
- **BFLA (Broken Function Level Authorization / OWASP API5)**: APIM pode bloquear funções por scope/role, mas a decisão final de função/ação do domínio deve estar no backend.
- **Autorização por campo/propriedade (OWASP API3)**: masking, allowlist de campos e validação de propriedade sensível pertencem ao backend; APIM pode complementar com schema validation e remoção de headers, não decidir regra de negócio fina.

Controles de comunicação APIM → backend:

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

| STRIDE | Risco típico em APIM | Controles APIM | OWASP API Top 10 2023 [20] |
|---|---|---|---|
| Spoofing | Cliente forjar identidade | OAuth/OIDC; mTLS; PKCE; validação de issuer/audience | API2 Broken Authentication |
| Tampering | Payload manipulado em trânsito ou mass assignment | TLS obrigatório; payload validation; HMAC/JWS quando aplicável; schema deny-by-default | API3 Broken Object Property Level Authorization; API8 Security Misconfiguration |
| Repudiation | Falta de evidência de quem chamou | Audit log; correlation ID; assinatura de logs em Sentinel | API9 Improper Inventory Management |
| Information Disclosure | Vazamento via erro/header/log/campo não autorizado | Erros padronizados; remoção de headers; políticas de log sem PII; validação de resposta quando aplicável | API3 Broken Object Property Level Authorization |
| Denial of Service | Flood, abuse, query GraphQL profunda | rate-limit; quota; WAF; `validate-graphql-request`; throttling backend; circuit breaker em backend | API4 Unrestricted Resource Consumption |
| Elevation of Privilege | Token mal validado, claim trustada cegamente, função admin exposta | Validação rígida (issuer, audience, scope, claims); deny-by-default; produtos separados por função | API5 Broken Function Level Authorization |
| (Adicional) | Acesso direto ao backend | Restrição de rede; allowlist IPs APIM no backend; mTLS APIM→backend | API8 Security Misconfiguration |
| (Adicional) | Shadow API | Defender for APIs [25]; inventário API Center [10] | API9 Improper Inventory Management |
| (Adicional) | Consumo inseguro de API externa | Timeout, validação de schema, allowlist de hosts, retry com limite | API10 Unsafe Consumption of APIs |

> Correção OWASP 2023: **API3 é "Broken Object Property Level Authorization"**. O rótulo usado no Top 10 2019 para exposição excessiva de dados foi consolidado no risco de autorização por propriedade/objeto em 2023. APIM ajuda na borda; autorização de objeto/campo permanece no backend (§7.3).

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
- **APIM não fica na Connectivity subscription por padrão.** Connectivity hospeda hub, Firewall/VWAN, DNS resolver, ExpressRoute/VPN e componentes compartilhados de rede. APIM é workload/plataforma de aplicação e deve nascer por **subscription vending** em uma subscrição dedicada de Platform-APIM ou na Workload subscription do domínio, com policies/tags/RBAC herdados [33].
- APIM corporativo nas subscrições de Platform; instâncias dedicadas nas Workloads do domínio. Exceção para colocar APIM no hub/Connectivity exige ADR aprovada por plataforma, segurança e rede, com justificativa de risco/blast radius.
- Hub-spoke: hub com firewall central, ExpressRoute/VPN e DNS híbrido; APIM em spoke dedicado ou workload spoke conforme modelo de adoção.
- Naming/tagging conforme padrão corporativo (§8.2).

### 8.2 Naming, tagging e ownership

Exemplo de convenção:

- `apim-{corp}-{env}-{regiao}-{seq}` — ex.: `apim-acme-prd-bs-01`.
- Tags obrigatórias em recursos APIM, workspace gateway, WAF, Log Analytics/Event Hub e recursos correlatos: `owner`, `costcenter`, `costAllocation`, `domain`, `apiProductOwner`, `data-classification`, `regulatory-class` (§2.3), `slaTier`, `slo`, `exposure` (`internal|partner|public|admin`), `retentionClass`, `oncall`, `criticality`, `mpasMapId`, `env`.
- Tags obrigatórias no nível de API/produto (API Center/OpenAPI/intake): `owner`, `apiProductOwner`, `supportContact`, `costAllocation`, `slaTier`, `exposure`, `retentionClass`, `regulatoryClass`, `mpasMapId`.
- Ownership do recurso APIM corporativo: time de plataforma; ownership por API/produto: domínio produtor. O owner de produto aprova quota, consumer onboarding, deprecação e custo alocado.

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

Matriz de responsabilidade dos guardrails:

| Camada | O que controla bem | O que NÃO controla | Exemplo de gate |
|---|---|---|---|
| **Azure Policy** | Recurso Azure: SKU/região, tags, TLS mínimo, diagnostic settings, public network access, Private Endpoint, Defender, identidade gerenciada | Semântica de OpenAPI, quota por produto, `<base/>` em policy XML se não houver definição específica, regra de negócio | Deny APIM fora de regiões permitidas; DeployIfNotExists para diagnostic settings |
| **Pipeline/IaC** | Contrato e configuração declarativa: OpenAPI, products, version sets, policies, fragments, backends, loggers, RBAC, budgets, alertas, breaking changes | Mudança emergencial feita no portal após deploy; comportamento runtime real | openapi-diff, secret scan, Bicep what-if/Terraform plan, policy lint |
| **Runtime APIM/WAF** | Tráfego real: OAuth/mTLS, CORS, rate limit/quota, schema validation, WAF, logs, correlation ID | Provar ownership, corrigir contrato ruim, autorizar objeto/campo de domínio | Bloquear token inválido, limitar flood, rejeitar GraphQL depth excessivo |

Rollout de Azure Policy:

1. **Audit** por 2–4 semanas para medir impacto e falsos positivos.
2. **Modify** quando o valor é seguro de corrigir automaticamente (tags padrão, diagnostic settings) e há managed identity de remediation.
3. **DeployIfNotExists** para recursos auxiliares obrigatórios (diagnostic settings, Defender, private DNS link quando padronizado).
4. **Deny** para riscos de alta confiança (região proibida, SKU não aprovado em produção, public network access proibido para classe interna/regulada, TLS antigo).

Toda mudança Audit→Deny/Modify/DeployIfNotExists exige changelog, owner, janela de comunicação e exceção formal (§11.5).

### 8.5 Rede, DNS e exposição

- **APIM Premium clássico VNet interna** para APIs internas; **VNet externa** quando exposta e justificada.
- **Front Door/Application Gateway com WAF** sempre antes do APIM para tráfego externo.
  - **Front Door + WAF**: default para entrada global, multi-region, CDN/cache/edge, aceleração, failover global e proteção no perímetro Microsoft [33].
  - **Application Gateway WAF v2**: default para entrada regional, APIM interno, VNet/L7, roteamento por path/host, Private Link/backend privado e integração com NSG/UDR [15][33].
  - **Front Door + App Gateway**: usar quando precisa edge global e perímetro regional privado. Documentar TLS end-to-end, headers (`X-Forwarded-*`), probes e origem privada.
- **DNS privado** para resolução interna; **DNS corporativo** para externo.
- **Certificados**: gerenciados no Key Vault, com rotação automática.

Matriz por classe de exposição:

| Classe | Padrão de entrada | DNS | NSG/UDR/Firewall/VWAN | Public network access |
|---|---|---|---|---|
| **Interna** | APIM internal VNet ou Private Endpoint inbound quando suportado; sem publicação internet | Private DNS zone `privatelink.azure-api.net` quando Private Endpoint; zona privada para domínios internos (`api.int.empresa.com`) linkada aos spokes | NSG permite apenas origem corporativa/hub; UDR força egress por Azure Firewall/VWAN quando inspeção é exigida; rotas simétricas testadas | Desabilitado quando o SKU/recurso suportar; se não suportar, compensar com WAF/IP allowlist e ADR |
| **Externa B2B** | Front Door WAF ou App Gateway WAF conforme global/regional; APIM não exposto direto | DNS público (`api.empresa.com`) com CNAME para Front Door/AppGW; DNS privado para backends | Allowlist de parceiros no WAF/APIM quando aplicável; egress backend via Private Link/VNet | Permitido somente no perímetro WAF; backend privado |
| **Regulada/Open Finance** | WAF + mTLS obrigatório no APIM/gateway; preferir AppGW regional se houver restrição de residência/VNet | DNS público com certificados EV/OV conforme política; Private DNS para backends e Key Vault | Azure Firewall/VWAN registra egress; NSG/UDR deny-by-default; CRL/OCSP/issuer endpoints alcançáveis para validação de certificado | APIM público só via WAF e mTLS; backends sem acesso público |
| **Administrativa** | Sem internet; acesso via bastion/VPN/ExpressRoute/Privileged Access Workstation | DNS privado apenas | NSG restritivo, UDR para firewall, PIM/JIT para gestão | Desabilitado/bloqueado |
| **Pública não autenticada** | Front Door WAF com rate limit agressivo e cache quando possível | DNS público; origin protegido | Origin só aceita Front Door/AppGW; logs completos de WAF | Permitido apenas no edge; APIM/backends protegidos |

DNS híbrido obrigatório:
- Resolver do hub encaminha zonas privadas Azure para spokes e, quando necessário, para DNS on-prem via Azure DNS Private Resolver.
- On-prem deve resolver FQDNs privados de APIM/backends sem hairpin para internet.
- Validar split-horizon (`api.empresa.com` externo vs interno) em runbook antes do go-live.

### 8.6 FinOps de APIM (sizing, custo, showback/chargeback)

Custo de APIM não vem apenas da licença do SKU. Vem de quatro grandes drivers:

1. **APIM unit/SKU** — Premium clássico tem custo por unit/região (multi-region = unit × regiões). Tiers v2 têm custo por unit/escala, chamadas incluídas/excedentes e, quando aplicável, secondary units.
2. **Observabilidade** — Log Analytics ingestion + retention. Em volume alto, custo de log pode superar o APIM. Política tiered de retenção (hot 30d / warm 90d / archive 1+ ano), table plans (Analytics/Basic/Auxiliary) e amostragem de logs não-críticos podem reduzir 40-70 % [18][31].
3. **Componentes de borda e segurança** — Front Door, Application Gateway, WAF, Defender for APIs. Defender for APIs cobra por API monitorada.
4. **Tráfego de saída e Private Link** — egress regional; Private Link endpoint hourly + por dado processado; Event Hub/SIEM quando logs são enviados para auditoria em tempo real.

#### 8.6.1 Sizing inicial

- Não use número fixo de TPS por unit como premissa de sizing ou orçamento. Throughput publicado ou citado é informativo, não é SLA e não deve ser usado para capacity/budget planning.
- Dimensione a partir do perfil real esperado: pico de requisições, concorrência, tamanho de payload, latência do backend, reutilização de conexões, TLS, número e custo das policies, cache, transformação, validação JWT/mTLS e logs.
- **Teste de carga representativo é obrigatório** antes de comprometer FinOps/produto. O teste deve refletir payloads, policies, backends, observabilidade e p95/p99 esperados em produção.
- Para operação e autoscale, use métricas sustentadas de capacidade/CPU/memória, latência, erros e saturação por período prolongado; não tome decisão por spike curto.
- Multi-region, unidades adicionais, calls v2, workspace gateways/packs e self-hosted gateways aumentam custo conforme meter aplicável por SKU/região. Revalidar preços no Azure Pricing Calculator [30].

#### 8.6.2 Tabela "componente × custo unitário típico" (referência)

Valores de lista consultados na Azure Retail Prices API para `API Management` em `brazilsouth`, acesso em **2026-06-18**. Cálculo mensal usa **730 h/mês**. Preços em USD, sem desconto, MACC, EA, impostos ou câmbio; **revalidar no Azure Pricing Calculator [30]** e na página de preços antes de aprovar orçamento.

| Componente / meter | Região | Unidade de cobrança | Preço lista | ≈ USD/mês | Driver / observação |
|---|---:|---:|---:|---:|---|
| APIM Basic v2 Unit | Brazil South | unit-hora | US$0.20548/h | US$150 | Custo base por unit |
| APIM Basic v2 Calls | Brazil South | 10K chamadas | tier incluído; depois ~US$0.03/10K | variável | Meter de chamadas v2; depende do volume excedente |
| APIM Standard v2 Unit | Brazil South | unit-hora | US$0.95890/h | US$700 | Custo base por unit |
| APIM Standard v2 Secondary Unit | Brazil South | unit-hora | US$0.68493/h | US$500 | Secondary units/escala adicional conforme billing meter |
| APIM Standard v2 Calls | Brazil South | 10K chamadas | tier incluído; depois ~US$0.025/10K | variável | Meter de chamadas v2; depende do volume excedente |
| APIM Premium clássico Unit | Brazil South | unit-hora, por região | US$3.82900/h | US$2.795 | Multi-region multiplica por unit/região |
| APIM Self-hosted Gateway (Developer/Premium) | Brazil South | gateway-hora/unidade licenciada | variável | variável | Runtime alternativo federado ao APIM; produção em Premium, Developer só avaliação [1][8] |
| APIM Premium v2 Unit | Brazil South | unit-hora | US$3.83562/h | US$2.800 | Custo base por unit; validar disponibilidade regional |
| APIM Premium v2 Secondary Unit | Brazil South | unit-hora | US$1.91781/h | US$1.400 | Secondary units/escala adicional conforme billing meter |
| APIM Workspace Gateway / Workspace Pack | região do gateway | gateway/pack-hora | variável | variável | Runtime dedicado por workspace; custo separado do workspace lógico [5] |
| Log Analytics ingestion | região do workspace | por GB ingerido | variável | variável | Volume de logs; planos Analytics/Basic/Auxiliary; transformação/sampling [18][31] |
| Log Analytics retention/archive | região do workspace | GB/mês após retenção incluída | variável | variável | Dias de retenção hot/warm/archive e volume retido |
| Event Hub para SIEM/auditoria | região do Event Hub | throughput/processing units + retenção + egress | variável | variável | `log-to-eventhub`, captura, consumidores SIEM e retenção [18][31] |
| Microsoft Sentinel | região do workspace | GB analisado/retido | variável | variável | Correlação de segurança; custo adicional ao Log Analytics |
| Defender for APIs | região/plano Defender | por API/overage conforme plano | variável | variável | Quantidade de APIs monitoradas e overage |
| Application Gateway WAF v2 | região do gateway | gateway-hora + capacity unit + tráfego | variável | variável | Capacidade, autoscale, WAF, regras e outbound data transfer |
| Front Door Premium | global | base fee + requests + data transfer | variável | variável | Requests, GB edge-cliente, GB edge-origem; WAF/Private Link incluídos no Premium |
| Private Link endpoint | região do endpoint | endpoint-hora + GB processado | variável | variável | Número de endpoints e dados processados |
| Egress regional | região de saída | GB de saída | variável | variável | Brazil South em zona de cobrança de bandwidth; otimizar com cache/compressão |

> Não use esta tabela como cotação. Use como baseline FinOps para identificar drivers dominantes. Cotação real deve ser revalidada por FinOps/procurement no Azure Pricing Calculator [30] e na subscrição do cliente.

#### 8.6.3 Tagging e showback/chargeback

- Cada API e produto carrega tags `costcenter`, `costAllocation`, `domain`, `apiProductOwner`, `slaTier`, `exposure`, `retentionClass` e `mpasMapId` (§8.2).
- Log Analytics workspace estruturado por categoria (gateway logs vs audit) permite atribuir consumo de log por domínio/API/produto.
- Relatório mensal de **showback** por domínio (consumo, requests, custo aproximado por componente) e **chargeback** quando a maturidade permitir.
- Defender for APIs custos podem ser segmentados por quantidade de APIs monitoradas e tag do produto/API.

Fórmula objetiva de rateio mensal por API/produto:

```text
custo_api =
  (requests_api / requests_total_produto) * custo_variavel_calls_v2
+ (gb_logs_api / gb_logs_total) * (custo_log_analytics_ingestao + custo_retencao + custo_eventhub_siem)
+ (apis_monitoradas_api / apis_monitoradas_total) * custo_defender_apis
+ (peso_sla_api / soma_pesos_sla) * custo_compartilhado_apim_waf
```

Pesos sugeridos de SLA para rateio compartilhado: Bronze=1, Silver=2, Gold=4, Regulado=6. O owner pode substituir por alocação direta quando há instância/workspace gateway dedicado.

#### 8.6.4 Alarmes orçamentários

- Azure Budgets por subscrição APIM + alertas em 50/80/100 %.
- Alertas específicos por componente: Log Analytics ingestion, Defender for APIs, Front Door, Application Gateway, Event Hub/Sentinel.
- Revisão trimestral em fórum FinOps.

#### 8.6.5 Otimizações comuns

- **Cache de policies** (cache-store/cache-lookup) reduz CPU e latência.
- **Compression** (gzip) reduz egress.
- **Sampling de logs** para classes não reguladas.
- **Retenção tiered** (hot/warm/archive) e table plans adequados.
- **Reserved capacity/commitment tiers** quando o consumo for previsível.

#### 8.6.6 Estratégia de ambientes não-produtivos

- DEV/Sandbox usa SKU menor e sem WAF global salvo teste específico; UAT replica controles de segurança, mas com escala reduzida.
- Desligamento/agendamento quando o SKU/ambiente permitir; quando APIM não puder ser desligado, reduzir scale units, workspace gateways e retenção de logs fora do horário útil.
- Dados sintéticos/mascarados obrigatórios em DEV/UAT; sem PII real salvo exceção DPO com retenção curta.
- Logs de não-prod em table plan/retention de menor custo (ex.: 7–30 dias), sem envio a SIEM salvo teste de segurança.
- Testes de carga usam janela controlada, budget temporário e limpeza automática de recursos.

---

## 9. Pipeline de publicação, IaC e APIOps

### 9.1 Princípios

- **Infraestrutura como código (IaC)** obrigatória. Bicep ou Terraform.
- **Tudo como código**: instâncias APIM, workspaces/workspace gateways, APIs, operations, schemas, **version sets**, **products**, groups/subscriptions, policy fragments, policies XML, backends, certificates references, loggers, diagnostics, alerts/action groups, RBAC, budgets, named values e Key Vault references.
- **Named values sem secrets em repositório**: valores sensíveis ficam no Key Vault; o IaC referencia Key Vault/managed identity. Secret literal em YAML/Bicep/policy/OpenAPI bloqueia PR.
- **Policies em XML versionado** em repositório, com fragmentos reutilizáveis e `<base/>` obrigatório quando herda controles globais/workspace/produto.
- **OpenAPI versionado** como contrato; GraphQL schema versionado quando aplicável.
- **Build determinístico**, ambientes idempotentes.
- **GitOps**: PR → revisão → merge → deploy automático.
- **Trunk-based development** + feature flags em vez de branches longas.
- **Não-prod econômico**: DEV/UAT com SKU/scale/retention reduzidos, dados sintéticos, teardown/scale-down quando possível e budgets próprios (§8.6.6).

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
- **openapi-diff / breaking change detection**: breaking change exige nova versão major, plano de migração e comunicação de deprecação (§6.4).
- **Lint policy** (xmllint + policy linter customizado), incluindo `<base/>` obrigatório em policies que herdam escopo global/workspace/produto.
- **Schema validation** de payloads em testes de contrato (Pact ou similar).
- **Testes unitários** de policies (Bicep what-if; policy expression dry-run).
- **Verificação SKU/região**: região do APIM, workspace gateway, WAF, Log Analytics e Key Vault precisa estar permitida para classe/regulação (§3.4, §8.4).
- **Secrets scan** em OpenAPI, policies, named values, YAML, Bicep/Terraform e exemplos; secrets só via Key Vault reference/managed identity.
- **CORS gate**: `*` apenas em sandbox; produção exige origins explícitos, métodos/headers mínimos e sem credential wildcard.
- **Products/quotas gate**: toda API publicada em produto explícito, sem “All APIs/Unlimited”, com quota/rate-limit por consumidor/jornada (§6.5).
- **Diagnostics/logging gate**: diagnostic settings, logger, correlation ID, mascaramento de PII e retenção conforme classe (§7.7/§8.6).
- **Verificação de classe regulatória** (§7.7): se classe = regulada, então gate exige mTLS, audit completo, retenção mínima e evidência de cadeia/certificado.
- **Verificação de ficha de intake** atualizada (§6.2).
- **Verificação de threat model** atualizado para classe (§7.6).
- **Testes pós-deploy obrigatórios**: smoke, contract test, auth negative test, quota/rate-limit, CORS, health probe WAF/APIM/backend e consulta de logs no workspace.

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
- **Infraestrutura** (rede, SKU, hostname, certificado, DNS, Private Endpoint, WAF, workspace gateway): rollback precisa de plano antes do merge. O PR deve incluir `what-if/plan`, passos de reversão, janela, owner, risco residual e evidência esperada.
- **Teste de rollback obrigatório** em UAT para mudanças de infra: validar resolução DNS, certificado/cadeia, health probes, mTLS, Private Endpoint, rota NSG/UDR/Firewall, WAF policy e smoke test do consumidor.
- **Evidência pós-rollback**: anexar ao incidente/PR logs de deploy, `az deployment operation`, diff de DNS/WAF/APIM, métricas p95/5xx e query KQL mostrando tráfego saudável por pelo menos N minutos.

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

Métricas mínimas por API (publicadas em dashboard/workbook):

- Latência gateway-only e end-to-end (p50, p95, p99).
- Taxa de erro separando **gateway** e **backend**: HTTP 4xx, 5xx, `GatewayResponseCodeCategory`, `BackendResponseCodeCategory`.
- Throughput/RPS por API, operação, produto, subscription e região.
- Backend latency (`BackendDuration`) e overall gateway duration (`Duration`).
- Token validation latency e falhas de validação (OAuth/OIDC/FAPI) quando aplicável.
- Cache hit/miss e uso de quota/rate-limit quando aplicável.

Sinais de capacidade por modelo de execução — **não misturar no mesmo alerta**:

| Modelo APIM | Métrica oficial | Uso recomendado |
|---|---|---|
| **Tiers clássicos** (Developer/Basic/Standard/Premium) | `Capacity` / **Capacity** | Tendência de saturação da instância clássica; média para escala e investigação de picos persistentes [6][36]. |
| **Tiers v2** (Basic v2/Standard v2/Premium v2) | `CpuPercent_Gateway` / **CPU Percentage of Gateway** e `MemoryPercent_Gateway` / **Memory Percentage of Gateway** | Escala/troubleshooting por CPU e memória do gateway v2; substituir o raciocínio antigo de Capacity [6][36]. |
| **Workspace gateway** (`Microsoft.ApiManagement/gateways`) | `CpuPercent_Gateway` / **CPU Utilization (%)** e `MemoryPercent_Gateway` / **Memory Utilization (%)** | Capacidade por gateway de workspace; acompanhar isolamento de runtime e noisy neighbor [6][36]. |

Sinais adicionais obrigatórios:

- **Rede/conectividade**: `NetworkConnectivity` (**Network Connectivity Status of Resources**) por `ResourceType`; health de Private Endpoint/Private DNS, WAF/AppGW/Front Door, Key Vault, Event Hub e backends críticos [36].
- **Mudança/drift**: `AzureActivity` para `Microsoft.ApiManagement/*/write`, `delete`, `regenerateKey`, alteração de named values, diagnostics, logger, custom domains, certificates e rede.
- **Segurança**: 401/403 anômalos, 429, spikes por IP/subscription/client-id, falhas mTLS/FAPI, Defender for APIs, alterações de RBAC/PIM e tentativa de acesso administrativo fora de janela [17][25].
- **Entrega de telemetria**: Event Hub dropped/rejected/throttled/timed-out events quando `log-to-eventhub` ou diagnostic settings usam Event Hubs [36].

Logs mínimos:

- Gateway logs sem corpo/header sensível por padrão; nunca registrar `Authorization`, refresh token, secret, chave de subscription ou payload pessoal sem base legal explícita.
- Audit/activity logs de plano de controle e Developer Portal.
- Diagnostic settings versionados em IaC; mudança manual vira incidente de governança.

### 10.2 Logs, métricas e alertas

#### 10.2.1 Categorias e tabelas oficiais

Diagnostic settings completos → Log Analytics central, Microsoft Sentinel quando aplicável, Storage WORM para auditoria crítica e Event Hub apenas quando houver consumidor operacional/SIEM externo. Preferir **resource-specific tables**; `AzureDiagnostics` fica como compatibilidade/legado ou workspace logs onde o recurso ainda não expõe tabela dedicada [31][36][37].

| Origem | Categoria oficial | Tabela Log Analytics recomendada | Uso |
|---|---|---|---|
| APIM gateway | `GatewayLogs` | `ApiManagementGatewayLogs` | Requisições HTTP, latência, códigos, erro, cache, API/operação/produto/subscription. |
| WebSocket | `WebSocketConnectionLogs` | `ApiManagementWebSocketConnectionLogs` | Handshake, abertura/fechamento e falhas de conexões WebSocket. |
| Developer Portal | `DeveloperPortalAuditLogs` | `APIMDevPortalAuditDiagnosticLog` | Login, navegação, teste de API e ações no portal do desenvolvedor. |
| GenAI gateway | `GatewayLlmLogs` | `ApiManagementGatewayLlmLog` | APIs LLM: tokens/modelo/prompt/response somente se permitido por classe e DPO. |
| MCP gateway | `GatewayMCPLogs` | `ApiManagementGatewayMCPLog` | Uso de servidores MCP publicados via gateway. |
| Plano de controle Azure | Activity Log | `AzureActivity` | PUT/PATCH/DELETE, RBAC, diagnostics, custom domains, certificados, SKU, rede. |
| Métricas exportadas | `AllMetrics` | `AzureMetrics` | `Requests`, `Duration`, `BackendDuration`, `Capacity`, CPU/memória gateway, Event Hub metrics. |
| Legado/compatibilidade | categorias antigas/serviços sem tabela dedicada | `AzureDiagnostics` | Usar só quando resource-specific não estiver disponível. |

Application Insights por API é opcional e deve ser do tipo workspace-based, com connection string/Managed Identity sempre que possível, para correlação fim-a-fim do produtor [32]. Métricas customizadas via policy (`emit-metric`) só para sinais agregados; não usar para PII.

#### 10.2.2 Alertas mínimos de produção

| Alerta | Fonte/KQL ou métrica | Severidade inicial | Ação obrigatória |
|---|---|---|---|
| 401/403 anômalo | `ApiManagementGatewayLogs` por `ResponseCode in (401,403)` e baseline por API/client-id/IP | Sev3; Sev2 se API regulada | Verificar token issuer/audience/scope, mTLS, client bloqueado, possível credential stuffing. |
| 429/rate limit | `ApiManagementGatewayLogs` `ResponseCode == 429` + dimensão subscription/API | Sev3 | Validar quota contratada, abuso, retry/backoff do consumidor, necessidade de escala. |
| 5xx gateway/backend | Métrica `Requests` por `GatewayResponseCodeCategory`/`BackendResponseCodeCategory` = `5xx` | Sev2 | Acionar owner API/backend; usar runbook 10.6 #1/#2. |
| Conectividade de backend | `BackendDuration`, `LastErrorReason`, `NetworkConnectivity`, health probe sintético | Sev2; Sev1 se crítica | Validar DNS, Private Endpoint, firewall, certificado TLS backend, SNAT/port exhaustion. |
| Capacidade clássica | `Capacity` média acima do limiar aprovado por 15 min; `Max` apenas investigação em Premium | Sev3/Sev2 | Escalar unidade/SKU, reduzir policy pesada, revisar TLS/new connections [6]. |
| Capacidade v2/workspace | `CpuPercent_Gateway` ou `MemoryPercent_Gateway` acima do limiar aprovado por 15 min | Sev3/Sev2 | Escalar gateway/units, revisar policy/cache/backend lento [6]. |
| Certificado D-30/D-7 | Key Vault certificate expiring ou inventário de custom domains/certificados | Sev3 em D-30; Sev2 em D-7 | Rotação via runbook 10.6 #3; evidência de validação em DEV/PROD. |
| Ausência de logs | Heartbeat/KQL: sem `ApiManagementGatewayLogs` para API com tráfego esperado por 15–30 min | Sev2 | Verificar diagnostic setting, sampling, workspace, ingestion, mudança manual. |
| Mudança manual | `AzureActivity` para write/delete fora do pipeline/service principal esperado | Sev2; Sev1 se PROD crítica | Abrir incidente de drift; bloquear portal write; reconciliar IaC/APIOps. |
| Evento de segurança | Defender for APIs/Sentinel analytics rule | Conforme regra SOC | Triage SOC, preservar evidência, acionar DPO se dado pessoal. |

#### 10.2.3 Gate de residência regional da observabilidade

Antes de habilitar APIM, workspace gateway, API Center, Application Insights, Event Hub, Sentinel ou destino de diagnostic settings, a ficha da API deve executar o gate abaixo e anexar a decisão ao ADR (§3.4 e Apêndice C):

| Recurso de telemetria/catálogo | Decisão exigida | Evidência mínima |
|---|---|---|
| **Log Analytics workspace** | Região por classe de dado; separar workspaces se residência, ownership, retenção ou Sentinel exigirem fronteira própria | ADR com região, classe (§7.7), owner, retenção por tabela; justificativa se fora de Brasil [38][40]. |
| **Application Insights workspace-based** | Mesma geografia/região da aplicação/APIM quando houver requisito de residência; revisar coleta de IP/URLs/custom telemetry | ADR + aprovação DPO/data owner; processor para mascarar PII quando necessário [32][43]. |
| **Event Hub para logs/auditoria** | Namespace na mesma região do recurso regional quando usado por diagnostic settings; avaliar geo-DR sabendo que metadata DR não replica eventos | ADR + owner do consumidor do stream + plano de retenção/capture [37][44]. |
| **Microsoft Sentinel** | Sentinel herda o Log Analytics workspace; workspace dedicado quando SOC/compliance exigir segregação | Aprovação SOC/DPO; retenção mínima 90 dias para Sentinel e política de archive se necessário [40][42]. |
| **Azure API Center** | Região do API Center aprovada para metadados de catálogo; sem cross-region failover automático no serviço | ADR de catálogo; confirmar que metadados e anexos não carregam segredos/PII indevidos [10][41]. |
| **Storage de arquivo/auditoria** | Mesmo país/geografia quando exigido; WORM bloqueado para auditoria crítica | Política imutável locked ou legal hold; chave, RBAC e lifecycle definidos [37][39]. |

Regra de aprovação: API **regulada**, **LGPD sensível**, **Open Finance**, **PCI** ou dado com restrição contratual só passa se DPO/jurídico, data owner e plataforma registrarem a região/destino em ADR. Sem ADR aprovado, o pipeline bloqueia diagnostics e publicação.

### 10.3 Dashboards

Dependência operacional: **Azure Monitor Workbooks** e dashboard Azure Monitor-based do APIM, não analytics clássico. O analytics clássico/built-in dos tiers clássicos será aposentado em **15/03/2027**; até lá, manter paralelo apenas para comparação e migrar consultas, alertas e reports para Log Analytics/Workbooks [11][31][53].

Mínimo por instância APIM:

- Dashboard/Workbook "Plataforma": saúde geral, `Requests`, latência, erros, capacidade correta por modelo (Capacity clássico vs CPU/memória v2/workspace), top APIs por volume/erro.
- Dashboard "Segurança": tentativas de token inválido, 401/403 anômalos, IPs anômalos, Defender alerts [25].
- Dashboard "FinOps": ingestion Log Analytics, Defender for APIs, egress (§8.6).
- Dashboard por produto/domínio: APIs do domínio com SLO/erro/latência, consumo por consumer owner e quota.
- Dashboard "Mudança e drift": AzureActivity, PR/deploy correlacionado, mudanças manuais, owner, rollback/evidência.
- Dashboard "Residência e retenção": workspace/destino por região, tabelas com retenção, WORM/Storage, exceções vencidas.

Plano de migração do analytics clássico:

| Data limite | Entrega | Owner |
|---|---|---|
| D+30 após aprovação do guia | Inventário de usuários/reports que ainda usam Analytics (classic) | Plataforma + DevEx |
| D+60 | Workbooks equivalentes publicados e validados com owners | Plataforma + Observabilidade |
| D+90 | Alertas migrados para métricas/KQL oficiais; classic só read-only | Plataforma + SOC |
| Até 2027-01-15 | Comunicação final e treinamento; dashboard clássico removido de rotinas | DevEx |
| Até 2027-03-15 | Zero dependência operacional do analytics clássico | Plataforma |

### 10.4 SLO, SLA e capacidade

- **SLO** definido pelo owner do produto, validado pela plataforma. Ex.: 99,9 % availability, p99 < 500 ms.
- **SLA** ao consumidor externo só após SLO maduro (3 meses verde).
- **Error budget** consumido publicamente; freeze de mudança quando budget esgota.
- **Capacity planning** trimestral; teste de carga obrigatório a cada SKU upgrade/downgrade.
- Alerta de escala deve usar o sinal correto do runtime: Capacity para clássico; CPU/memória para v2 e workspace gateway (§10.1).

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
| **Crítica regulada (Open Finance, pagamentos)** | Premium clássico | Sim — AZs (zone-redundant) | Sim, ativo-ativo entre regiões (nativo) | < 5 min | ~0 | Runbook DR ensaiado anualmente |
| **Alta (negócio essencial)** | Premium clássico ou Premium v2 (onde houver) | Sim — AZs (Premium / Premium v2) | Premium clássico: ativo-passivo entre regiões. Premium v2: **sem multi-region nativo** — usar IaC + 2ª instância | < 30 min | < 5 min | Runbook DR ensaiado semestralmente |
| **Média (negócio importante)** | Standard v2 | **Não suporta AZ** (redundância por múltiplas unidades, sem zona) | **Sem backup/restore nativo**; DR por IaC + recriação | < 4 h | < 1 h | Runbook DR documentado, ensaio anual |
| **Baixa (interno auxiliar)** | Basic v2 ou Standard v2 | **Não suporta AZ** | DR por IaC recreate | < 24 h | < 24 h | IaC + smoke test |
| **Sandbox/Dev** | Developer | Não | Não | — | — | — |

Observações:
- **Availability Zones: apenas Premium clássico e Premium v2.** Basic v2 e Standard v2 **não** suportam AZ [1]. Confirmar disponibilidade de zonas na região (Brazil South suporta para os tiers elegíveis; Brazil Southeast verificar).
- Premium v2 hoje **não cobre multi-region ativo-ativo**; para crítica regulada, Premium clássico é mandatório [1].
- **Backup/restore: disponível nos tiers clássicos** (Developer/Basic/Standard/Premium); **não disponível nos tiers v2** — nestes, DR depende de IaC + recriação.
- Multi-region cross-border: avaliar se a regulação permite (LGPD/BACEN podem exigir residência); se logs/telemetria cruzam região, aplicar gate §10.2.3.

### 10.6 Runbooks/playbooks operacionais mínimos

Cinco runbooks obrigatórios, mantidos no repositório de operação, ensaiados periodicamente:

1. **Surge de 5xx em uma API** — passos: dashboard → identifica backend → circuit breaker → comunicar consumidores → escalar para owner da API e on-call backend.
2. **Backend indisponível total** — passos: confirmar via alerta + dashboard → ativar fallback (se houver) → comunicar → status page → coordenação com owner backend.
3. **Certificado próximo de expirar / expirado** — passos: alerta D-30 e D-7 → rotação no Key Vault → validação em dev → janela de troca produção → validação. Para FAPI/Open Finance: coordenar com cadeia ICP.
4. **Rotação de chave / segredo (incidente ou agendada)** — passos: gerar nova chave no Key Vault → atualizar policies → comunicar consumidores impactados → revogar antiga após carência. Incidente: comprimir prazo, gerar relatório de evidência.
5. **DR failover (regional outage)** — passos: confirmar outage Azure → ativar região secundária → atualizar DNS/Front Door → validar tráfego → comunicar status page → no retorno, drenar de volta de forma controlada. Ensaiado conforme criticidade (§10.5) e alinhado com evidência de rollback/teste definida em §9.

Adicionais recomendados:
6. **Incidente de segurança (LGPD)** — fluxo de comunicação ao DPO e ANPD em prazos (Resolução CD/ANPD nº 15/2024) [27].
7. **Incidente de cibersegurança (BACEN)** — comunicação a BACEN conforme Resolução CMN nº 4.893/2021 [28c].
8. **Reporte mensal FinOps** — fechamento de custo por componente, alocação por domínio.
9. **Falha/ausência de telemetria** — diagnostic settings, ingestion, Event Hub, workspace, sampling e RBAC; classificar como incidente se API crítica ficar sem logs.
10. **Drift/mudança manual em PROD** — congelar mudanças, exportar AzureActivity, reconciliar IaC, abrir RCA e evidência para auditoria.

Cada runbook tem owner, frequência de ensaio, critério de sucesso e data do último ensaio.

### 10.7 Retenção, finalidade e imutabilidade de auditoria

Retenção não é "guardar tudo para sempre". Cada tabela/destino deve ter finalidade, base legal/contratual, owner e prazo. Referência inicial, ajustável pelo DPO/jurídico:

| Dado/log | Finalidade | Base/driver | Retenção mínima sugerida | Imutabilidade |
|---|---|---|---|---|
| `ApiManagementGatewayLogs` operacional | Troubleshooting, SLO, capacity, suporte | Interesse legítimo/contrato/SLO | 30–90 dias hot; até 180 dias para APIs externas | Não, salvo incidente. |
| Gateway logs de API regulada | Evidência de transação, disputa, auditoria regulatória | BACEN/Open Finance/contrato | 1–5 anos conforme norma/contrato | Sim para extrato/auditoria crítica em Storage WORM. |
| `AzureActivity` e mudanças APIM/IaC | Auditoria de change, SoD, investigação | Controles internos, ISO/NIST/COBIT/COSO | 1–7 anos conforme GRC | Sim para PROD crítica e trilha de mudança. |
| Developer Portal audit | Acesso de consumidor, solicitação/teste de API | Contrato, segurança, suporte | 180 dias–2 anos | Opcional; obrigatório em APIs reguladas. |
| Application Insights traces/dependencies | Diagnóstico da aplicação produtora | Operação/SLO | 30–90 dias; maior só com justificativa | Não por padrão; evitar PII. |
| Sentinel/security alerts | Detecção/resposta a incidentes | Segurança/cyber/regulatório | ≥ 90 dias hot; archive conforme SOC | WORM quando evidência legal/regulatória. |
| Exports de auditoria e evidências de RCA | Prova de controle, investigação, litígio | Auditoria/jurídico | Conforme política GRC/litigation hold | Sim: policy locked ou legal hold. |

Para auditoria crítica, enviar cópia para Azure Storage com **immutability policy/WORM** (time-based retention locked ou legal hold). A política precisa ser testada antes de bloquear, pois depois de locked não pode ser reduzida/removida dentro do período [39]. Log Analytics mantém analytics/long-term retention até 12 anos, mas WORM é a camada de não alteração quando a evidência precisa ser não apagável/não modificável [38][39].

---

## 11. Governança corporativa, ownership e experiência do desenvolvedor

### 11.1 Princípios de governança

- **Governança via guardrails técnicos**: Azure Policy, gates de pipeline, lint, validação de classe. Documento/wiki como complemento, nunca como controle único.
- **Ownership inquestionável**: cada API tem owner nominal (squad), técnico (lead), DPO/compliance quando regulado, on-call definido.
- **Decisão registrada em ADR**: modelos, SKUs, regiões, exceções.
- **Catálogo único** (API Center [10]) como visão executiva.
- **Caminho rápido para o comum**, caminho aprovado para o excepcional (§11.6).

### 11.2 Ownership mínimo

Cada API publicada precisa, na ficha (§6.2), com owner nominal e grupo substituto:

- **API/Product owner** (squad produtor): accountable por valor, SLO, backlog e comunicação de mudança.
- **Tech lead produtor**: accountable técnico pela implementação e compatibilidade OpenAPI/policies da API.
- **Consumer owner**: responsável pelo sistema consumidor principal/parceiro; recebe mudanças, acompanha quota e incidentes de consumo.
- **Data owner**: accountable por classificação, finalidade, base legal, retenção, mascaramento e autorização de uso de dados.
- **Platform owner APIM**: accountable pela instância, guardrails, runtime, diagnostics, rede da plataforma e pipeline comum.
- **Security owner**: accountable pelos controles de autenticação, mTLS/FAPI, threat model e resposta de segurança.
- **DPO/compliance/jurídico** quando classe regulada, LGPD sensível, Open Finance, PCI ou transferência regional.
- **On-call rotation** (PagerDuty/Opsgenie/Microsoft Teams) do produtor e da plataforma.
- **Aprovador de release** distinto do autor quando PROD crítica (§11.6).

Regra: nenhuma API entra em PROD com owner genérico ("time X") sem pessoa ou papel operacional escalável. Substitutos são permitidos; accountable continua único por decisão.

### 11.3 RACI resumido

RACI usa **um único Accountable (A)** por atividade. Quando houver comitê, ele atua como Consulted; uma pessoa/papel decide e assina.

| Atividade | API/Product owner | Tech lead produtor | Consumer owner | Data owner | Plataforma APIM | Segurança | Compliance/DPO | Auditoria |
|---|---|---|---|---|---|---|---|---|
| Design da API e contrato OpenAPI | A | R | C | C | C | C | C se regulada | I |
| Classificação da API e dados (§7.7/§10.7) | C | C | I | A/R | C | C | C | I |
| Publicação em DEV/UAT | A | R | I | C | R | C | I | I |
| Publicação em PROD não crítica | A | R | I | C | R | C | C se regulada | I |
| Publicação em PROD crítica/regulada | A | R | C | C | R | C | C | I |
| Policies padrão (JWT, rate-limit, headers) | C | C | I | I | A/R | C | I | I |
| Policy custom de domínio | A | R | I | C | C | C | C se regulada | I |
| Operação 24×7 da API | A | R | I | I | R pela instância | C | I | I |
| Operação 24×7 da plataforma | I | C | I | I | A/R | C | I | I |
| Incidente de backend/API | A | R | C | C se dado afetado | C | C | C se regulado | I |
| Incidente de segurança | C | R | I | C | C | A/R | C se dado pessoal | I |
| Incidente regulatório/LGPD | C | R | I | C | C | C | A/R | C |
| Residência regional de telemetria (§10.2.3) | C | C | I | R | R | C | A | I |
| Retenção/WORM de auditoria (§10.7) | C | C | I | A/R | R | C | C | C |
| Exceções (§11.5) | R | R | I | C se dado | C | C | A conforme tipo | C |
| Aprovação de mudança PROD crítica (SoD) | C | R (autor) | I | C | A ou delegado não autor | C | C se regulada | I |

### 11.4 Experiência do produtor e consumidor

- **Self-service** no Developer Portal: registro, subscription, gestão de keys, exemplos, sandbox.
- **Producer toolkit**: templates Bicep, fragmentos policy, esqueleto OpenAPI (Apêndice B), pipeline reference (Apêndice D).
- **Documentação viva**: OpenAPI sempre atual, gerado da fonte; mudanças visíveis no portal.
- **SLAs do canal de suporte** definidos para consumidores externos.

#### 11.4.1 Jornada do consumidor end-to-end

| Etapa | Experiência esperada | Evidência/controle |
|---|---|---|
| 1. Descobrir API | Consumidor encontra API no API Center/Developer Portal por domínio, classe, versão, ambiente, owner e status de suporte | API registrada com metadados obrigatórios, OpenAPI válido, exemplos e changelog. |
| 2. Entender contrato | Lê autenticação, scopes, quota, payloads, erros padrão, SLO, política de uso e dados tratados | OpenAPI + guia rápido; data owner e finalidade visíveis quando há dado pessoal. |
| 3. Solicitar acesso | Abre solicitação de subscription/app registration com justificativa, ambiente, owner consumidor e volumes esperados | Ticket/workflow com consumer owner, API/product owner e data owner quando necessário. |
| 4. Aprovar | Aprovação por owner correto conforme classe: self-service para baixo risco; governado para regulada/sensível | RACI §11.3, registro de aprovação, prazo/SLA de aprovação medido. |
| 5. Testar em sandbox | Recebe endpoint sandbox, credenciais/claims de teste, collections e exemplos; erros são diagnósticos sem vazar PII | Sandbox isolado, synthetic tests, dados fictícios, suporte de onboarding. |
| 6. Promover para produção | Passa por checklist de segurança, quota, contato on-call, fallback e janela de go-live | Gate CI/CD, owner consumidor, plano de rollback e status page. |
| 7. Acompanhar quota/consumo | Consumidor vê consumo, 429, latência, erros e budget; recebe alertas proativos antes de quota estourar | Workbook/portal por subscription; relatório mensal de top consumers. |
| 8. Suporte e mudança | Canais claros para incidente, dúvida, aumento de quota, deprecação e breaking change | SLA de suporte, comunicação D-90/D-30/D-7, headers `Deprecation`/`Sunset`. |

#### 11.4.2 Comunicação de sunset / deprecação a consumidores

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
- Aprovador único (depende da criticidade — ver §11.5.3).
- Prazo de validade (data de expiração obrigatória).
- Critério de saída (o que precisa acontecer para a exceção ser fechada).
- Data owner/consumer owner impactado quando a exceção afeta dado ou consumo.

#### 11.5.2 Ciclo de vida

```
[criada] → [em análise] → [aprovada com prazo] → [ativa] → [revisão antes de expirar] →
   [renovada (raro)] | [encerrada — desvio resolvido] | [escalada — não resolvido no prazo]
```

#### 11.5.3 Aprovador conforme criticidade

| Tipo de exceção | Aprovador único (A) | Consultados obrigatórios |
|---|---|---|
| Operacional (config menor) | Owner plataforma | API owner |
| Padrão técnico (lint, naming) | Arquitetura de plataforma | Tech lead produtor |
| Segurança (policy obrigatória ignorada) | CISO/segurança | Plataforma + DPO se aplicável |
| Dados/residência/retenção | DPO/compliance | Data owner + jurídico + plataforma |
| Regulatória (BACEN, LGPD sensível, Open Finance) | Compliance executivo/DPO designado | Jurídico + segurança + plataforma + API owner |

#### 11.5.4 Inventário

Inventário público de exceções (read-only para auditoria) em sistema de tickets (Jira/Azure Boards) e exposição agregada via API Center.

#### 11.5.5 Métricas

- Total de exceções ativas.
- Exceções vencidas (KPI principal — meta: zero).
- Tempo médio de exceção (dias).
- Exceções renovadas (>1 ciclo).
- Backlog de exceções por classe e por owner.

### 11.6 Self-service vs aprovado (caminho rápido vs caminho governado)

Para evitar que tudo passe pelo time central, mapear tipos de mudança:

| Tipo de mudança | Caminho | Aprovador único | Gate |
|---|---|---|---|
| Nova revision de policy compatível em DEV/UAT | Self-service | API/Product owner | CI verde |
| Nova revision compatível em PROD não crítica | Self-service controlado | API/Product owner | CI + CODEOWNERS aplicável |
| Nova versão de API (incompatível) | Self-service + comunicação | API/Product owner | CI + plano de deprecação |
| Mudança de policy obrigatória (JWT validation) | Governado | Segurança | Threat model atualizado |
| Novo produto no catálogo | Self-service | API/Product owner | CI + ficha intake |
| Nova API regulada | Governado | DPO/compliance | Classe §7.7 atendida + ADR |
| Nova instância APIM dedicada | Governado | Arquitetura de plataforma | Critério §4.5 + ADR |
| Nova região APIM/telemetria | Governado | DPO/compliance | Gate §10.2.3 + disponibilidade verificada [1][3] |
| Exceção a policy obrigatória | Governado | §11.5.3 conforme tipo | Ficha de exceção |
| Mudança de SKU (upgrade/downgrade) | Governado | Plataforma | Teste de carga + ADR |
| Mudança PROD crítica/regulada | Governado + SoD | Plataforma ou API owner **não autor** | CODEOWNERS + branch protection + environment approval sem self-review [50][51][52] |

#### 11.6.1 Segregação de funções (SoD) para PROD crítica

Para APIs críticas/reguladas, o autor do PR/deploy **não pode** ser o aprovador final. Implementação mínima:

- `CODEOWNERS` com owner por área: `/apis/{dominio}/`, `/policies/`, `/bicep/`, `/workbooks/`.
- Branch protection/ruleset em `main` e branches de release: PR obrigatório, status checks obrigatórios, review de code owner, bloquear force-push/delete e aplicar a admins quando possível [51].
- GitHub Environments para `prod`: required reviewers, prevent self-review, deployment branches restritas e segredos só liberados após aprovação [52].
- Service principal/Managed Identity de deploy sem permissão interativa humana; portal APIM read-only em PROD.
- Evidência: PR, reviewer, check run, deployment approval, `AzureActivity` e hash do artefato publicado anexados ao change.

#### 11.6.2 Métricas DX e governança

| Métrica | Definição | Meta inicial |
|---|---|---|
| Lead time de mudança | Tempo de PR aberto até produção | P50 ≤ 2 dias para self-service; P90 ≤ 10 dias governado |
| Tempo de onboarding consumidor | Solicitação aprovada até primeiro 2xx em sandbox/prod | Sandbox ≤ 3 dias; PROD ≤ 10 dias |
| Deploy failure rate | Deploys com rollback/fix-forward ou incidente em 24 h | < 10 %; crítica < 5 % |
| Satisfação/NPS do consumidor | Survey trimestral por consumidor/parceiro | NPS ≥ 40 ou tendência positiva |
| Backlog de exceções | Exceções ativas/vencidas por owner e classe | Vencidas = 0; redução mensal |
| % mudanças self-service | Mudanças sem comitê manual fora dos gates definidos | ≥ 70 % |
| Tempo de aprovação governada | SLA de Segurança/DPO/Plataforma para decisões | P90 ≤ 5 dias úteis |

Métrica de saúde: % de mudanças via self-service. Meta: ≥ 70 %. Abaixo disso, plataforma é gargalo ou os gates estão mal calibrados.

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

Decisões do cliente priorizadas por impacto na implantação. Cada item precisa de owner sugerido, prazo e ADR/ticket de evidência.

| Prioridade | Decisão pendente | Owner sugerido | Gate / critério de aceite |
|---|---|---|---|
| **Bloqueia implantação** | Modelo de adoção definitivo (A/B/C) | Head de Arquitetura + Platform owner APIM | ADR aprovada em 30d (§13.2). |
| **Bloqueia implantação** | Residência obrigatória: Brasil only, geografia específica ou multi-region cross-border | DPO/Compliance + Jurídico | Política regulatória registrada em §2.3.4/§3.4 e gate §10.2.3. |
| **Bloqueia implantação** | Classe regulatória das APIs do roadmap (Open Finance, LGPD sensível, PCI) | Data owner de cada domínio + API/Product owner | Cada API com classe na ficha de intake (§6.2) e matriz §7.7. |
| **Bloqueia implantação** | Região/destino de Log Analytics, Application Insights, Event Hub, Sentinel e API Center | DPO/Compliance + Platform owner APIM | ADR regional por classe; sem ADR, pipeline bloqueia diagnostics/publicação (§10.2.3). |
| **Bloqueia implantação** | Aprovador único de exceções por tipo | CISO/DPO/Compliance executivo | Nomes/papéis cadastrados em §11.5.3 e ferramenta de tickets. |
| **Bloqueia implantação** | Política de retenção de logs e WORM para auditoria crítica | Data owner + DPO + Auditoria | Tabela §10.7 aprovada; Storage WORM/long-term retention provisionados. |
| **Em paralelo** | Owner de FinOps | FinOps lead + Platform owner APIM | Papel atribuído antes de §13.2 90–120d; relatório mensal definido. |
| **Em paralelo** | Janela de DR e RTO/RPO formal por classe | BCP/DR owner + Platform owner APIM | §10.5 preenchida com valores específicos; ensaio calendarizado. |
| **Em paralelo** | SoD em PROD crítica (CODEOWNERS/branch protection/environment approval) | DevEx lead + Segurança | Controles §11.6.1 ativos antes do primeiro go-live crítico. |
| **Pode esperar** | Política de uso de Premium clássico vs migração futura para Premium v2 | Arquitetura de plataforma | Revisão trimestral §13.2; gatilho quando Premium v2 cobrir requisitos/região. |
| **Pode esperar** | Estratégia para regiões fora de Brazil South quando Premium v2 não estiver listado | Arquitetura + DPO | ADR datada; revalidar disponibilidade [1][3]. |
| **Pode esperar** | Modelo de chargeback/showback por domínio | FinOps lead | Piloto após telemetria estável; integrar workbook FinOps. |

---

## 15. Referências consolidadas

> Formato: **instituição — título — URL — data de acesso**. Todas as referências devem ser revalidadas em datas críticas (regulatório, SKU/região). Data desta consolidação: **18 jun. 2026**.

### Nota contratual e responsabilidade compartilhada

O desenho APIM deve ser lido junto com os **Online Services Terms (OST)** e o **Microsoft Products and Services Data Protection Addendum (DPA)** aplicáveis ao contrato do cliente. Esses documentos não substituem a decisão arquitetural: no modelo de responsabilidade compartilhada, Microsoft opera a infraestrutura/plataforma do serviço em nuvem, enquanto o cliente continua responsável por dados, identidades, configurações, acesso, classificação, retenção, chaves, policies e conformidade do uso da API [45][46][47].

### Azure API Management — conceitos, tiers, regiões e recursos

[1] Microsoft Learn — API Management feature-based comparison (v1, v2, Consumption, self-hosted) — https://learn.microsoft.com/en-us/azure/api-management/api-management-features — Acesso em: 18 jun. 2026.

[2] Microsoft Learn — APIM v2 tiers overview — https://learn.microsoft.com/en-us/azure/api-management/v2-service-tiers-overview — Acesso em: 18 jun. 2026.

[3] Microsoft Azure — Products available by region / API Management — https://azure.microsoft.com/explore/global-infrastructure/products-by-region — Acesso em: 18 jun. 2026.

[4] Microsoft Learn — Migrate from stv1 to stv2 — https://learn.microsoft.com/en-us/azure/api-management/migrate-stv1-to-stv2 — Acesso em: 18 jun. 2026.

[5] Microsoft Learn — Workspaces in Azure API Management — https://learn.microsoft.com/en-us/azure/api-management/workspaces-overview — Acesso em: 18 jun. 2026.

[6] Microsoft Learn — Capacity Metrics in Azure API Management — https://learn.microsoft.com/en-us/azure/api-management/api-management-capacity — Acesso em: 18 jun. 2026.

[7] Microsoft Learn — Deploy an Azure API Management service instance to multiple Azure regions — https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-deploy-multi-region — Acesso em: 18 jun. 2026.

[8] Microsoft Learn — Self-hosted gateway overview — https://learn.microsoft.com/en-us/azure/api-management/self-hosted-gateway-overview — Acesso em: 18 jun. 2026.

[9] Microsoft Learn — Backup and restore an Azure API Management service instance — https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-disaster-recovery-backup-restore — Acesso em: 18 jun. 2026.

[10] Microsoft Learn — Azure API Center overview — https://learn.microsoft.com/en-us/azure/api-center/overview — Acesso em: 18 jun. 2026.

[11] Microsoft Learn — Azure API Management analytics dashboard retirement (March 2027) — https://learn.microsoft.com/en-us/azure/api-management/breaking-changes/analytics-dashboard-retirement-march-2027 — Acesso em: 18 jun. 2026.

[12] Microsoft Learn — GenAI gateway capabilities in Azure API Management — https://learn.microsoft.com/en-us/azure/api-management/genai-gateway-capabilities — Acesso em: 18 jun. 2026.

### Rede, conectividade, resiliência e arquitetura

[13] Azure GitHub — Azure/apiops toolkit — https://github.com/Azure/apiops — Acesso em: 18 jun. 2026.

[14] Microsoft Learn — ARM/Bicep deployment what-if operation — https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-what-if — Acesso em: 18 jun. 2026.

[15] Microsoft Learn — Use API Management with virtual networks — https://learn.microsoft.com/en-us/azure/api-management/api-management-using-with-internal-vnet — Acesso em: 18 jun. 2026.

### Policies, segurança, identidade e padrões externos

[16] Microsoft Learn — validate-azure-ad-token policy — https://learn.microsoft.com/en-us/azure/api-management/validate-azure-ad-token-policy — Acesso em: 18 jun. 2026.

[17] Microsoft Learn — Microsoft Defender for APIs — https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-apis-introduction — Acesso em: 18 jun. 2026.

[18] Microsoft Learn — Observability in Azure API Management — https://learn.microsoft.com/en-us/azure/api-management/observability — Acesso em: 18 jun. 2026.

[19] Microsoft Learn — Configure mutual certificate authentication in Azure API Management — https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-mutual-certificates-for-clients — Acesso em: 18 jun. 2026.

[20] OWASP Foundation — OWASP API Security Top 10 2023 — https://owasp.org/API-Security/editions/2023/en/0x00-header/ — Acesso em: 18 jun. 2026.

[21] IETF — RFC 9700: Best Current Practice for OAuth 2.0 Security — https://datatracker.ietf.org/doc/rfc9700/ — Acesso em: 18 jun. 2026.

[22] OpenID Foundation — Financial-grade API (FAPI) 1.0 Part 1 — https://openid.net/specs/openid-financial-api-part-1-1_0.html — Acesso em: 18 jun. 2026.

[23] OpenID Foundation — FAPI 2.0 Security Profile — https://openid.net/specs/fapi-security-profile-2_0.html — Acesso em: 18 jun. 2026.

### API design, catálogo, produtos e developer experience

[24] Microsoft Learn — Azure API Center key concepts / registry and metadata — https://learn.microsoft.com/en-us/azure/api-center/key-concepts — Acesso em: 18 jun. 2026.

[25] Microsoft Learn — Defender for APIs detections — https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-apis-introduction — Acesso em: 18 jun. 2026.

### DevOps, APIOps, IaC e qualidade

[26] Microsoft Learn — Automated API deployments with APIOps — https://learn.microsoft.com/en-us/azure/architecture/example-scenario/devops/automated-api-deployments-apiops — Acesso em: 18 jun. 2026.

### Regulatório (Brasil)

[27] ANPD — Resolução CD/ANPD nº 15/2024 e comunicação de incidente de segurança — https://www.gov.br/anpd/pt-br/assuntos/noticias/anpd-publica-regulamento-de-comunicacao-de-incidente-de-seguranca — Acesso em: 18 jun. 2026.

[28] BACEN — Resolução BCB nº 32/2020 (Open Banking/Open Finance) — https://www.bcb.gov.br/estabilidadefinanceira/exibenormativo?tipo=Resolu%C3%A7%C3%A3o%20BCB&numero=32 — Acesso em: 18 jun. 2026.

[28b] BACEN — Resolução BCB nº 198/2022 (serviços de processamento/armazenamento e computação em nuvem) — https://www.bcb.gov.br/estabilidadefinanceira/exibenormativo?tipo=Resolu%C3%A7%C3%A3o%20BCB&numero=198 — Acesso em: 18 jun. 2026.

[28c] CMN — Resolução CMN nº 4.893/2021 (política de segurança cibernética e contratação de nuvem) — https://www.bcb.gov.br/estabilidadefinanceira/exibenormativo?tipo=Resolu%C3%A7%C3%A3o%20CMN&numero=4893 — Acesso em: 18 jun. 2026.

[29] Open Finance Brasil — Portal oficial — https://openfinancebrasil.org.br/ — Acesso em: 18 jun. 2026.

[29b] Open Finance Brasil — Padrões Técnicos / Segurança — https://openfinancebrasil.atlassian.net/wiki/spaces/OF/pages/17378602/Padr+es+T+cnicos — Acesso em: 18 jun. 2026.

> **Nota sobre URLs regulatórias:** as URLs listadas foram a referência ao tempo desta consolidação. Atos normativos podem ser republicados; **antes de uso em auditoria, a equipe de compliance confirma o ato vigente** (número, ano, redação) na fonte oficial.

### Operações, observabilidade, retenção e monitoramento

[30] Microsoft Azure — Pricing Calculator (APIM, Log Analytics, Defender for APIs, Front Door) — https://azure.microsoft.com/pricing/calculator/ — Acesso em: 18 jun. 2026.

[31] Microsoft Learn — Tutorial: Monitor APIs in Azure API Management with Azure Monitor — https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor — Acesso em: 18 jun. 2026.

[32] Microsoft Learn — Integrate Azure API Management with Azure Application Insights — https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights — Acesso em: 18 jun. 2026.

[36] Microsoft Learn — Monitoring data reference for Azure API Management — https://learn.microsoft.com/en-us/azure/api-management/monitor-api-management-reference — Acesso em: 18 jun. 2026.

[37] Microsoft Learn — Diagnostic settings in Azure Monitor — https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings — Acesso em: 18 jun. 2026.

[38] Microsoft Learn — Manage data retention in a Log Analytics workspace — https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure — Acesso em: 18 jun. 2026.

[39] Microsoft Learn — Immutable storage for blob data — https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview — Acesso em: 18 jun. 2026.

[40] Microsoft Learn — Design a Log Analytics workspace architecture — https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design — Acesso em: 18 jun. 2026.

[41] Microsoft Learn — Reliability in Azure API Center — https://learn.microsoft.com/en-us/azure/reliability/reliability-api-center — Acesso em: 18 jun. 2026.

[42] Microsoft Learn — Onboard to Microsoft Sentinel — https://learn.microsoft.com/en-us/azure/sentinel/quickstart-onboard — Acesso em: 18 jun. 2026.

[43] Microsoft Learn — Application Insights FAQ: data collection, retention, storage, and privacy — https://learn.microsoft.com/en-us/azure/azure-monitor/app/application-insights-faq#how-does-application-insights-handle-data-collection--retention--storage--and-privacy — Acesso em: 18 jun. 2026.

[44] Microsoft Learn — Geo-disaster recovery for Azure Event Hubs — https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-geo-dr — Acesso em: 18 jun. 2026.

[53] Microsoft Learn — Azure Workbooks overview — https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview — Acesso em: 18 jun. 2026.

### Landing zone, governança e Azure Policy

[33] Microsoft Learn — Azure landing zones design areas — https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas — Acesso em: 18 jun. 2026.

[34] Microsoft Learn — API Management policy reference — https://learn.microsoft.com/en-us/azure/api-management/policy-reference — Acesso em: 18 jun. 2026.

[35] Microsoft Learn — Azure Policy guest configuration and remediation — https://learn.microsoft.com/en-us/azure/governance/policy/concepts/guest-configuration — Acesso em: 18 jun. 2026.

### Contratos, responsabilidade compartilhada e frameworks de controle

[45] Microsoft Learn — Shared responsibility in the cloud — https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility — Acesso em: 18 jun. 2026.

[46] Microsoft Licensing — Online Services Terms (OST) — https://www.microsoft.com/licensing/docs/view/Online-Services-Terms — Acesso em: 18 jun. 2026.

[47] Microsoft Licensing — Microsoft Products and Services Data Protection Addendum (DPA) — https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA — Acesso em: 18 jun. 2026.

[48] ISACA — COBIT 2019 framework resources — https://www.isaca.org/resources/cobit — Acesso em: 18 jun. 2026.

[49] COSO — Internal Control — Integrated Framework guidance — https://www.coso.org/guidance-on-ic — Acesso em: 18 jun. 2026.

[50] GitHub Docs — About code owners — https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners — Acesso em: 18 jun. 2026.

[51] GitHub Docs — About protected branches — https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches — Acesso em: 18 jun. 2026.

[52] GitHub Docs — Deployments and environments — https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments — Acesso em: 18 jun. 2026.


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
    <!-- NUNCA logar o header Authorization / bearer token. Registrar apenas claims ja validadas pelo validate-azure-ad-token acima (scope, client_id). -->
    <set-variable name="authScope" value="@{
        var jwt = context.Request.Headers.GetValueOrDefault("Authorization","").Replace("Bearer ","").Trim().AsJwt();
        return jwt?.Claims.GetValueOrDefault("scp","") ?? "";
    }" />
    <set-variable name="authClientId" value="@{
        var jwt = context.Request.Headers.GetValueOrDefault("Authorization","").Replace("Bearer ","").Trim().AsJwt();
        return jwt?.Claims.GetValueOrDefault("appid","") ?? "";
    }" />
    <log-to-eventhub logger-id="audit-lgpd" partition-id="0">
      @{
        return new JObject(
          new JProperty("ts", DateTime.UtcNow.ToString("o")),
          new JProperty("correlationId", context.RequestId),
          new JProperty("operation", context.Operation.Name),
          new JProperty("subjectIdHash", context.Variables.GetValueOrDefault<string>("subjectId").GetHashCode()),
          new JProperty("scope", context.Variables.GetValueOrDefault<string>("authScope","")),
          new JProperty("clientId", context.Variables.GetValueOrDefault<string>("authClientId",""))
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

> **Exemplo vs produção:** o XML abaixo é exemplo de padrão verificável para mTLS + sender-constrained token. Em produção, antes do deploy: (1) habilitar negociação de certificado de cliente no gateway/custom domain; (2) cadastrar no APIM as CAs raiz/intermediárias ICP-Brasil/Open Finance aprovadas; (3) validar CRL/OCSP/revogação e rotação; (4) testar com certificado válido, expirado, revogado e de emissor não autorizado; (5) confirmar com compliance o perfil FAPI/Open Finance vigente [19][29b][19].

```xml
<policies>
  <inbound>
    <base />

    <!-- mTLS real: falha se o cliente não apresentar certificado confiável, vigente, não revogado e emitido pela cadeia aprovada. -->
    <validate-client-certificate
        validate-revocation="true"
        validate-trust="true"
        validate-not-before="true"
        validate-not-after="true"
        ignore-error="false">
      <identities>
        <!-- IDs de certificados/CA carregados em APIM. Exemplos: substituir pelos IDs reais da cadeia ICP-Brasil/Open Finance aprovada. -->
        <identity issuer-certificate-id="{{openfinance-icp-br-root-ca-id}}" />
        <identity issuer-certificate-id="{{openfinance-directory-root-ca-id}}" />
      </identities>
    </validate-client-certificate>

    <set-variable name="clientCertX5tS256" value="@{
        var cert = context.Request.Certificate;
        if (cert == null) { return &quot;&quot;; }
        using (var sha256 = System.Security.Cryptography.SHA256.Create()) {
          var hash = sha256.ComputeHash(cert.RawData);
          return Convert.ToBase64String(hash).TrimEnd('=').Replace('+','-').Replace('/','_');
        }
    }" />

    <validate-jwt header-name="Authorization" require-scheme="Bearer" failed-validation-httpcode="401" failed-validation-error-message="Invalid Open Finance token">
      <openid-config url="{{openfinance-issuer-discovery}}" />
      <audiences><audience>{{openfinance-audience}}</audience></audiences>
      <required-claims>
        <claim name="scope" match="any">
          <value>{{openfinance-scope}}</value>
        </claim>
        <!-- Sender-constrained token: cnf precisa existir; comparação x5t#S256 ocorre no choose abaixo. -->
        <claim name="cnf" match="all" />
      </required-claims>
    </validate-jwt>

    <set-variable name="tokenCnfX5tS256" value="@{
        var jwt = context.Request.Headers.GetValueOrDefault(&quot;Authorization&quot;,&quot;&quot;).Replace(&quot;Bearer &quot;,&quot;&quot;).Trim().AsJwt();
        var cnf = jwt?.Claims.GetValueOrDefault(&quot;cnf&quot;,&quot;&quot;) ?? &quot;&quot;;
        if (String.IsNullOrEmpty(cnf)) { return &quot;&quot;; }
        return JObject.Parse(cnf).Value<string>(&quot;x5t#S256&quot;) ?? &quot;&quot;;
    }" />

    <choose>
      <when condition="@((string)context.Variables[&quot;clientCertX5tS256&quot;] != (string)context.Variables[&quot;tokenCnfX5tS256&quot;])">
        <return-response>
          <set-status code="401" reason="Sender constrained token mismatch" />
        </return-response>
      </when>
    </choose>

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
          new JProperty("clientCertSubject", context.Request.Certificate?.SubjectName?.Name),
          new JProperty("clientCertThumbprint", context.Request.Certificate?.Thumbprint)
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

> Produção não deve depender de helper fictício de validação de cadeia. A cadeia é validada por `validate-client-certificate` contra CAs configuradas no APIM; o token é sender-constrained pela claim `cnf` (`x5t#S256`) vinculada ao certificado apresentado; e o backend continua responsável por consentimento, autorização de objeto e regra FAPI/Open Finance fina.

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
      - ".github/workflows/apim-ci.yml"
  push:
    branches: ["main"]
    paths:
      - "apis/**"
      - "policies/**"
      - "bicep/**"
  schedule:
    # Drift semanal fora do horário comercial. Ajustar fuso/janela do cliente.
    - cron: "17 3 * * 1"
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

concurrency:
  group: apim-${{ github.ref }}
  cancel-in-progress: false

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
      - name: Breaking change gate
        run: ./scripts/openapi-breaking-change.sh apis/

  lint-policy-xml:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: xmllint policies
        run: |
          sudo apt-get update && sudo apt-get install -y libxml2-utils
          find policies -name "*.xml" -print0 | xargs -0 -I{} xmllint --noout "{}"
      - name: Policy linter customizado
        run: ./scripts/policy-lint.sh policies/ --require-base

  validate-class-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verifica ficha de intake (classe regulatória)
        run: ./scripts/check-intake-class.sh apis/
      - name: Secrets, CORS, products, quotas e diagnostics
        run: ./scripts/check-apim-release-gates.sh apis/ policies/ bicep/

  bicep-what-if:
    runs-on: ubuntu-latest
    needs: [lint-openapi, lint-policy-xml, validate-class-gate]
    if: github.event_name != 'schedule'
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

  deploy-prod:
    runs-on: ubuntu-latest
    needs: [bicep-what-if]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.empresa.com
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy APIM IaC
        run: ./scripts/deploy-apim.sh prod
      - name: Post-deploy smoke/contract/auth/quota tests
        run: ./scripts/post-deploy-tests.sh prod
      - name: Upload production evidence
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: prod-deploy-evidence
          path: artifacts/prod/**

  drift-report:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: APIOps Toolkit extractor
        run: |
          mkdir -p artifacts/extracted-config
          ./tools/apiops-extractor.sh --output artifacts/extracted-config 2>&1 | tee artifacts/apiops-extractor.log
      - name: Compare to repo
        run: |
          mkdir -p artifacts
          diff -ru apis/ artifacts/extracted-config/ > artifacts/drift-report.diff || true
          if [ -s artifacts/drift-report.diff ]; then
            echo "DRIFT_DETECTED=true" >> "$GITHUB_ENV"
          else
            echo "DRIFT_DETECTED=false" >> "$GITHUB_ENV"
            echo "No drift detected" > artifacts/drift-report.diff
          fi
      - name: Upload drift report
        uses: actions/upload-artifact@v4
        with:
          name: drift-report
          path: |
            artifacts/drift-report.diff
            artifacts/apiops-extractor.log
      - name: Fail when drift exists
        if: env.DRIFT_DETECTED == 'true'
        run: |
          echo "Drift detected. Open an issue and reconcile via IaC, not manually."
          exit 1
```

> Para produção, configurar o **GitHub Environment `production`** com reviewers obrigatórios (plataforma + owner + segurança/compliance quando regulado), secrets segregados, branch protection e retenção dos artifacts de evidência. O YAML referencia o environment; a aprovação é aplicada na configuração do repositório.

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
| COBIT | Control Objectives for Information and Related Technologies | Framework ISACA para governança de informação e tecnologia |
| COSO | Committee of Sponsoring Organizations of the Treadway Commission | Framework de controles internos e gestão de risco |
| CORS | Cross-Origin Resource Sharing | Política HTTP |
| DCR | Dynamic Client Registration | Registro dinâmico de cliente OAuth (RFC 7591) |
| DNS | Domain Name System | — |
| DPA | Data Protection Addendum | Aditivo contratual Microsoft para proteção de dados |
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
| OST | Online Services Terms | Termos contratuais Microsoft para serviços online |
| OWASP | Open Web Application Security Project | — |
| PCI-DSS | Payment Card Industry Data Security Standard | — |
| PII | Personally Identifiable Information | Dados pessoais |
| PIM | Privileged Identity Management | Microsoft Entra ID |
| PKCE | Proof Key for Code Exchange | RFC 7636, para clientes públicos |
| RACI | Responsible / Accountable / Consulted / Informed | Matriz de responsabilidades; A deve ser único |
| RBAC | Role-Based Access Control | Azure RBAC |
| RIPD | Relatório de Impacto à Proteção de Dados Pessoais | Exigido pela LGPD em certos casos |
| RPO | Recovery Point Objective | Janela máxima de perda de dado |
| RTO | Recovery Time Objective | Janela máxima de recuperação |
| SIEM | Security Information and Event Management | Ex.: Microsoft Sentinel |
| SKU | Stock Keeping Unit | Tier do APIM (Basic/Standard/Premium etc.) |
| SLA | Service Level Agreement | Contrato com consumidor |
| SLO | Service Level Objective | Meta operacional interna |
| SOC | Security Operations Center | Time de operações de segurança |
| SoD | Segregation of Duties | Autor, aprovador e operador separados em mudança crítica |
| TLS | Transport Layer Security | — |
| TPS | Transactions Per Second | — |
| VNet | Virtual Network | Azure |
| WAF | Web Application Firewall | Application Gateway / Front Door |
| WORM | Write Once, Read Many | Imutabilidade: dado gravado não pode ser alterado/apagado no período |

---

## Apêndice F — Mapeamento ISO 27001 / NIST CSF / COBIT 2019 / COSO → controles APIM

Mapeamento ilustrativo (não exaustivo) para apoiar conversa com auditoria/compliance. COBIT/COSO são usados como linguagem de governança/controle; o controle real é a evidência técnica (PR, policy, AzureActivity, workbook, runbook, aprovação) [48][49].

| Controle ISO 27001:2022 (Anexo A) | NIST CSF 2.0 | COBIT 2019 | COSO | Onde aterrissa em APIM |
|---|---|---|---|---|
| A.5.7 Threat intelligence | ID.RA-02 | APO12 / DSS05 | Risk Assessment | §7.6 threat model + Defender for APIs [25] |
| A.5.15 Access control | PR.AA-01 | DSS05 / APO13 | Control Activities | §7.1, §7.2, §7.7 |
| A.5.18 Access rights provisioning | PR.AA-05 | DSS06 / DSS05 | Control Activities | RBAC §8.3 + PIM |
| A.5.23 Use of cloud services | GV.SC-04 | APO10 / APO13 | Control Environment | §8.1, §13.2, ADRs, OST/DPA §15 |
| A.5.30 ICT readiness for business continuity | RC.RP-01 | DSS04 | Risk Assessment | §10.5 HA/DR + §10.6 runbooks |
| A.6.7 Remote working | PR.AA-04 | DSS05 | Control Activities | §7.1 PIM + Microsoft Entra ID |
| A.8.2 Privileged access rights | PR.AA-05 | DSS05 | Control Activities | PIM + custom roles + SoD |
| A.8.5 Secure authentication | PR.AA-01 | DSS05 | Control Activities | §7.2 OAuth/OIDC, FAPI, mTLS |
| A.8.9 Configuration management | PR.PS-01 | BAI10 | Control Activities | §9 APIOps + drift §9.4.1 + AzureActivity §10.2 |
| A.8.15 Logging | DE.AE-03 | DSS01 / MEA03 | Information & Communication | §10.1, §10.2, §7.8 |
| A.8.16 Monitoring activities | DE.CM-01 | DSS01 / DSS05 | Monitoring Activities | Sentinel + Defender for APIs + Workbooks |
| A.8.20 Networks security | PR.IR-01 | DSS05 | Control Activities | §8.5 rede privada, VNet |
| A.8.21 Security of network services | PR.IR-01 | DSS05 | Control Activities | WAF, Private Link |
| A.8.23 Web filtering | PR.PT-04 | DSS05 | Control Activities | Front Door + WAF |
| A.8.24 Use of cryptography | PR.DS-02 | DSS05 | Control Activities | mTLS, TLS, Key Vault |
| A.8.26 Application security requirements | PR.PS-06 | BAI03 / APO13 | Control Activities | §7.5 policies; §7.7 matriz vinculante |
| A.8.28 Secure coding | PR.PS-06 | BAI03 | Control Activities | Lint OpenAPI + policies |
| A.8.29 Security testing in development & acceptance | PR.PS-06 | BAI07 | Control Activities / Monitoring | §9.3 gates |
| A.8.31 Separation of development, test and production | PR.PS-01 | BAI07 / DSS06 | Control Activities | §12.1 #12 anti-pattern; ambientes separados |
| A.8.32 Change management | PR.PS-01 | BAI06 | Control Activities / Information & Communication | §11.6 self-service vs governado; PR + branch protection |
| A.8.34 Protection during audit & testing | PR.PS-02 | MEA03 | Monitoring Activities | §11.5 exceções + evidências |
| SoD autor ≠ aprovador em PROD crítica | GV.RR-02 / PR.PS-01 | DSS06.03 / BAI06 | Control Activities | §11.6.1 CODEOWNERS, branch protection, environment approval |
| Change evidenciado e rastreável | PR.PS-01 / GV.OV-03 | BAI06 / MEA02 | Information & Communication | PR, check run, approval, AzureActivity, artefato publicado |
| Evidência imutável de auditoria | PR.DS-01 / GV.OV-03 | DSS06 / MEA03 | Monitoring Activities | §10.7 Storage WORM, legal hold, retenção por finalidade |

> Para mapear controles a evidências reais: a cada controle, registrar em sistema GRC o link para PR/issue/runbook/relatório que demonstra implementação. Se a evidência não é reproduzível, o controle não conta.

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
3. Pipeline APIOps com gate de classe regulatória e SoD para PROD crítica (90 dias).
4. Threat model por classe (90 dias).
5. DR ensaiado por criticidade (180 dias).
6. Self-service ≥ 70 % dos PRs, com DX medido (240 dias).

### Custos e riscos chave

- Driver dominante de custo: Log Analytics ingestion + Microsoft Sentinel/Defender for APIs + multi-region unit (§8.6).
- Risco regulatório principal: classe regulada (Open Finance) sem mTLS, sem decisão de residência da telemetria ou sem retenção/WORM para evidência crítica.
- Risco de execução principal: governança central sem caminho rápido → §11.6.
- Risco de auditoria: autor aprovando própria mudança em PROD crítica; mitigar com CODEOWNERS, branch protection e environment approval (§11.6.1).

### Decisões pendentes do cliente (§14)

Bloqueiam implantação: modelo A/B/C, residência, classes do roadmap, regiões/destinos de telemetria/catálogo, aprovadores de exceção e retenção/WORM. Em paralelo: FinOps, DR/RTO/RPO e SoD. Pode esperar: estratégia Premium v2 futura e chargeback.
