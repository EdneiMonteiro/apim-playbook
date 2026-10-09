# Azure API Management (APIM) corporativo: guia técnico de decisão e implantação

<!-- apim-navigation:start -->
## Seções do playbook

- [1. Sumário executivo](#1-sumário-executivo)
- [2. Contexto e problema](#2-contexto-e-problema)
- [3. Azure API Management: planos, tiers clássicos, Consumption, v2 e regiões](#3-azure-api-management-planos-tiers-clássicos-consumption-v2-e-regiões)
- [4. Modelos de adoção (corporativo, por subscrição/domínio, híbrido) com matriz de decisão](#4-modelos-de-adoção-corporativo-por-subscriçãodomínio-híbrido-com-matriz-de-decisão)
- [5. Arquitetura de referência](#5-arquitetura-de-referência)
- [6. Padrões de API, ciclo de vida, versionamento, catálogo e consumo](#6-padrões-de-api-ciclo-de-vida-versionamento-catálogo-e-consumo)
- [7. Segurança, identidade e políticas](#7-segurança-identidade-e-políticas)
- [8. Plataforma, landing zone, rede e guardrails](#8-plataforma-landing-zone-rede-e-guardrails)
- [9. Pipeline de publicação com infraestrutura como código e APIOps](#9-pipeline-de-publicação-com-infraestrutura-como-código-e-apiops)
- [10. Operação, observabilidade e objetivos e acordos de nível de serviço](#10-operação-observabilidade-e-objetivos-e-acordos-de-nível-de-serviço)
- [11. Governança corporativa, responsabilidades e experiência do desenvolvedor](#11-governança-corporativa-responsabilidades-e-experiência-do-desenvolvedor)
- [12. Práticas inadequadas e boas práticas](#12-práticas-inadequadas-e-boas-práticas)
- [13. Recomendações finais e plano de implantação sugerido](#13-recomendações-finais-e-plano-de-implantação-sugerido)
- [14. Decisões pendentes que dependem do cliente](#14-decisões-pendentes-que-dependem-do-cliente)
- [15. Referências consolidadas](#15-referências-consolidadas)
- [Apêndice A: exemplos de policies XML por perfil de consumidor (§7.7)](#apêndice-a-exemplos-de-policies-xml-por-perfil-de-consumidor-77)
- [Apêndice B: modelo OpenAPI para contrato HTTP/REST e metadados de governança](#apêndice-b-modelo-openapi-para-contrato-httprest-e-metadados-de-governança)
- [Apêndice C: modelo de registro de decisão arquitetural (ADR)](#apêndice-c-modelo-de-registro-de-decisão-arquitetural-adr)
- [Apêndice D: fragmento de promoção e verificação de drift (GitHub Actions)](#apêndice-d-fragmento-de-promoção-e-verificação-de-drift-github-actions)
- [Apêndice E: glossário](#apêndice-e-glossário)
- [Apêndice F: resultados NIST CSF 2.0, práticas APIM e evidências de controle](#apêndice-f-resultados-nist-csf-20-práticas-apim-e-evidências-de-controle)
- [Apêndice G: síntese executiva de adoção, controles e prontidão](#apêndice-g-síntese-executiva-de-adoção-controles-e-prontidão)
<!-- apim-navigation:end -->

---

## 1. Sumário executivo

Azure API Management (APIM) centraliza controles de exposição de interfaces de programação de aplicações (APIs): autenticação, autorização de perímetro, limitação de tráfego, observabilidade, catálogo e contratos com consumidores. Não substitui a engenharia das APIs nem a autorização de objetos e regras de negócio no backend.

O desenho separa **modelo de adoção** (compartilhado, por domínio ou híbrido), **tier/SKU** (plano de recursos e capacidade), **runtime** (gateway gerenciado, gateway de workspace ou self-hosted gateway) e **região/rede**. SKU significa Stock Keeping Unit e identifica a oferta comercial; não determina sozinho governança, isolamento ou residência. Workspaces delegam responsabilidades e acesso a recursos, mas não equivalem a instâncias independentes nem asseguram isolamento de execução em todos os tiers.

O **modelo híbrido/federado governado** é a hipótese inicial para uma empresa com plataforma central madura: compartilhar controles, padrões e catálogo e admitir instâncias dedicadas quando o limite necessário de falha, operação, região ou capacidade justificar a separação. Autonomia de equipe ou um objetivo de nível de serviço (SLO, Service Level Objective) próprio são fatores de avaliação, não condições suficientes para criar uma instância independente.

### 1.1 Cinco perguntas para a decisão (quadro executivo)

Documente requisitos, alternativas e escolha em um **Registro de Decisão Arquitetural (ADR, Architecture Decision Record)** antes de comprometer SKU e topologia. Neste quadro, **WAF** é o firewall de aplicações web; **VNet** é a rede virtual Azure; **TLS** (Transport Layer Security) protege a comunicação em trânsito, e **mTLS** acrescenta autenticação mútua por certificados. OAuth 2.0 é um framework de autorização; **OIDC** (OpenID Connect) acrescenta identidade em cenários de usuário; **FAPI** (Financial-grade API) define um perfil de segurança aplicável; **DPO** é o encarregado pelo tratamento de dados pessoais. **LGPD** é a Lei Geral de Proteção de Dados Pessoais; **PCI DSS** é o padrão de segurança de dados de cartões. O perfil FAPI precisa ser identificado, não presumido a partir desses regimes.

| # | Pergunta | Se a resposta é… | Decisão indicada |
|---|----------|------------------|------------------|
| 1 | Existe restrição aprovada de localização de dados, metadados ou processamento? | Sim | Selecionar região e recursos permitidos. Na consulta de 09/10/2026, Brazil South lista Basic v2/Standard v2, mas não Premium v2 nem workspace gateway [3]. Residência Brasil, por si só, não exige Premium clássico; Brazil Southeast e capacidade de contratação precisam de confirmação. |
| 2 | Quais dados, regimes setoriais e tipos de consumidor estão presentes? | Há requisitos adicionais | Compor exposição, dado pessoal comum/sensível legal/restrito corporativo, setor, consumidor e criticidade (§7.7). OAuth client credentials atende cenários máquina a máquina pertinentes; Authorization Code com PKCE atende acesso delegado por usuário. PKCE protege a troca do código de autorização. Aplicar FAPI/mTLS quando o perfil ou requisito identificado os exigir; LGPD/PCI não impõem FAPI genericamente. Retenção segue política aprovada por conjunto e finalidade [65][66]. |
| 3 | A API será acessível pela internet? | Sim | A política corporativa deste guia exige WAF no caminho de entrada e proteção contra acesso direto à origem. Escolher Front Door para entrada global ou Application Gateway para regional. Quando houver mTLS, explicitar terminação TLS, identidade do consumidor e confiança no trecho proxy–APIM (§8.5) [15][19][69]. |
| 4 | A operação exige separar equipe, manutenção, falhas ou SLO? | Separação demonstrada | Comparar delegação por workspace, runtime dedicado e instância independente. Instância dedicada é candidata quando o isolamento exigido e a capacidade de operar equipe, runbook e plantão a justificarem; autonomia isolada não basta. |
| 5 | O volume excede a capacidade testada, ou há necessidade explícita de multirregião nativa/self-hosted gateway? | Sim | Para volume, testar escala, policies, backend e alternativas de tier antes de escolher. Para multirregião nativa ou self-hosted gateway em produção, Premium clássico é a opção documentada; Developer serve apenas à avaliação. Recuperação de desastre também pode usar instâncias independentes, com provas de recuperação da solução [1][7][54]. |

### 1.2 Disponibilidade no Brasil

A consulta documental de **09/10/2026** lista Basic v2 e Standard v2 em Brazil South; não lista Premium v2 nem workspace gateway nessa região [3]. Workspaces lógicos existem em Basic v2, Standard v2, Premium clássico e Premium v2 [1][5]. O gateway padrão pode atender workspaces somente nos tiers v2; em Premium clássico o workspace exige workspace gateway, com região e cobrança próprias. Assim, workspaces de Premium clássico não são a alternativa de runtime compartilhado no Brasil apresentada pelos tiers v2.

**Residência Brasil não seleciona um tier sozinha.** Standard v2 oferece integração de saída à VNet e private endpoint de entrada; Premium v2 oferece VNet injection, mas depende de oferta regional. Premium clássico permanece a alternativa documentada quando são exigidos multirregião nativa, self-hosted gateway em produção ou injection em uma região brasileira onde Premium v2 não esteja listado [1][2][3][15]. Migração exige cobertura funcional, rede, continuidade, capacidade e custo validados, além de disponibilidade regional. Revisar a decisão trimestralmente e antes de contratação ou mudança material.

---

## 2. Contexto e problema

Grandes empresas tipicamente operam dezenas a centenas de APIs em múltiplos domínios (produto, plataforma, dados, parceiros, mobile, integrações entre empresas (B2B)), com regimes distintos de regulação, criticidade, exposição (interna, parceira, pública) e governança (centralizada, federada, autônoma).

A adoção de Azure exige definir como as APIs serão expostas e consumidas dentro e fora da nuvem, com segurança, controle e governança. O APIM implementa grande parte dos controles decorrentes dessas decisões.

Erros comuns que este documento busca prevenir:

- Escolher o modelo centralizado ou distribuído por preferência cultural, sem considerar regulação, escala e responsabilidades.
- Padronizar o tier comercial (SKU) sem distinguir modelo de adoção, geração do serviço, gateway, capacidades necessárias e disponibilidade regional.
- Tratar APIM como proxy ou equipamento de rede, sem governança de produto, ciclo de vida, catálogo e experiência do desenvolvedor.
- Substituir controles técnicos por aprovações manuais, ou dispensar registros de decisão e evidência porque existem policies no gateway.
- Esquecer integração entre APIM, Microsoft Entra ID, Azure Policy, Defender for Cloud, observabilidade e pipelines.

### 2.1 Premissas e escopo

- Este guia considera uma empresa de grande porte, com múltiplos domínios e equipes distribuídas. A aplicabilidade da Lei Geral de Proteção de Dados Pessoais (LGPD), de regras do Banco Central do Brasil (BCB), de Open Finance e do padrão de segurança de dados de cartões PCI DSS deve ser confirmada por atividade e entidade, não presumida pelo porte.
- Pressupõe-se uma Azure Landing Zone, base de governança e conectividade da nuvem, ou equivalente, com rede hub/spoke, Azure Policy, Log Analytics, Sentinel e Microsoft Entra ID. A existência e a configuração efetiva desses controles precisam ser verificadas antes do aceite.
- O foco é a publicação de APIs e transportes suportados pelo gateway escolhido: HTTP/REST, GraphQL, WebSocket e gRPC quando confirmado (§6.6). Cada protocolo exige contrato e testes apropriados (§9.3). Mensageria pura (Event Hubs/Service Bus) está fora do escopo do APIM; uma fachada HTTP não substitui os controles do sistema de mensageria.
- Exposição, dados pessoais/sensíveis conforme a LGPD, classificação corporativa de dados, setor, tipo de consumidor e criticidade são dimensões independentes da ficha de intake (§6.2). Os controles resultam da combinação aplicável; dados pessoais ou cartões não tornam toda API participante de um perfil Financial-grade API (FAPI).

### 2.2 Quem este documento orienta

| Perfil | Seções e decisões principais |
|--------|-------------------------------|
| Liderança de tecnologia e engenharia | §1: enquadramento; §4: adoção; §11–14: responsabilidades, implantação e decisões |
| Arquitetura Azure | §3: tiers; §5: arquitetura; §8: landing zone; §9: entrega |
| Segurança, gestão de identidade e regulatório | §1.1: requisitos; §2.3: normas; §7: controles; §11: autoridade e aprovação |
| Plataforma e landing zone | §5, §8–10: desenho, guardrails, entrega e operação |
| Produtor da API (squad) | §4, §6, §7, §9–11: adoção, contrato, segurança, publicação e operação |
| Consumidor da API (mobile, web ou parceiro) | §6.5: consumo; §10: operação; §11.4: acesso, suporte e migração |
| Gestão financeira da nuvem (FinOps) e compras | §3.3: escolha de tier; §8.6: custos e alocação |
| Auditoria e risco | §2.3: aplicabilidade; §7.7–7.8: controles; §11.5: exceções e evidência |

### 2.3 Requisitos regulatórios (Brasil e setores específicos)

A escolha de modelo, SKU, região e controles deve se basear em requisitos identificados por entidade, atividade e conjunto de dados, com versão e data da fonte. O inventário abaixo distingue obrigação aplicável, orientação da autoridade e política corporativa; nenhuma categoria dispensa a análise das demais.

#### 2.3.1 LGPD (Lei nº 13.709/2018): classificação e responsabilidade pelo tratamento

- **Classificação e bases legais:** identificar a finalidade de cada operação de tratamento e a hipótese e o inciso do art. 7º para dados pessoais comuns ou do art. 11 para dados pessoais sensíveis. O art. 5º, II define a categoria sensível, que inclui, por exemplo, saúde e biometria vinculada a pessoa natural. Dado financeiro nominal não é automaticamente sensível legal, embora possa revelar uma categoria sensível pelo contexto e exigir proteção elevada. A classificação corporativa de dado restrito é uma dimensão adicional. Execução contratual e interesse legítimo do art. 7º não se tornam, por analogia, hipóteses do art. 11. Sem mapeamento do tratamento aplicável, não publicar [66].
- **Direitos do titular** (art. 18): manter procedimentos de acesso, retificação, eliminação, portabilidade e informação sobre compartilhamento, respeitando as condições legais e as hipóteses de conservação do art. 16. Quando APIs forem usadas para atendê-los, identificar, testar e auditar a integração com o sistema-fonte; APIM pode expor a fachada, mas não executar sozinho a decisão jurídica ou a eliminação nos sistemas de domínio [66].
- **Encarregado/DPO** (Data Protection Officer): identificar o encarregado e seus meios de contato, conforme o enquadramento aplicável. O controlador mantém o inventário do tratamento e a responsabilidade pelas decisões; o encarregado orienta e apoia, inclusive quanto a logs, incidentes e transferências [66].
- **Relatório de Impacto à Proteção de Dados Pessoais (RIPD):** distinguir a determinação da Agência Nacional de Proteção de Dados (ANPD), nos termos do art. 38, da recomendação de elaboração para operações de alto risco. Uma exigência adicional da empresa deve ser identificada como política interna aprovada. Vincular a API à operação ou ao projeto de tratamento coberto; operações similares em natureza, finalidade e riscos podem compartilhar um relatório, conforme a orientação da ANPD [66][76].
- **Incidente de segurança** (art. 48 e Resolução CD/ANPD nº 15/2024): o controlador deve comunicar à ANPD e aos titulares quando houver potencial de risco ou dano relevante. O gatilho do art. 5º combina impacto significativo nos interesses e direitos com pelo menos um critério: dados sensíveis; de crianças, adolescentes ou idosos; financeiros; de autenticação; protegidos por sigilo; ou tratamento em larga escala [27]. O runbook (§10.6) precisa operacionalizar:
  1. **Relógio e responsáveis:** registrar quando o controlador soube que dados pessoais foram afetados. A comunicação à ANPD tem prazo de três dias úteis, ressalvado prazo previsto em legislação específica; a dos titulares, três dias úteis conforme o art. 9º. Não aguardar a conclusão da investigação para iniciar a contagem. O prazo da comunicação inicial à ANPD e da complementação, assim como o prazo do caput do art. 9º, conta em dobro somente para agentes efetivamente enquadrados na Resolução CD/ANPD nº 2/2022; não presumir a elegibilidade da empresa ou estender a exceção a todos os prazos do regulamento.
  2. **Comunicação à ANPD:** usar o formulário eletrônico disponibilizado pela autoridade, por meio do encarregado ou representante constituído, com prova do vínculo ou dos poderes de representação apresentada no prazo. Incluir as informações do art. 6º, §2º, inclusive identificação do controlador e operador, datas, dados e titulares afetados, riscos e medidas. Informações podem ser complementadas de forma fundamentada em vinte dias úteis a partir da comunicação, observado o regime de prazo em dobro quando efetivamente aplicável. Justificar eventual atraso; observar segredos comercial e industrial e requerer sigilo das informações protegidas na forma do art. 7º, sem suspender o dever de comunicar.
  3. **Comunicação aos titulares:** empregar linguagem simples e contato direto individualizado quando possível. Se isso for inviável ou não for possível identificar todos os afetados, divulgar de modo amplo e facilmente visível pelos meios disponíveis, no mesmo prazo e com o conteúdo exigido, mantendo a divulgação por pelo menos três meses. Juntar ao processo a declaração dos meios utilizados em até três dias úteis após o término do prazo de comunicação aos titulares, conforme o art. 9º, §4º.
  4. **Registro:** conservar o registro do incidente, inclusive do não comunicado, por pelo menos cinco anos a partir da data do registro, ou prazo maior decorrente de obrigação adicional. Registrar a avaliação de risco e os motivos da não comunicação. Para entidades do art. 23 da LGPD, observar a ressalva do art. 10, §2º, relativa às regras de guarda permanente e à tabela de temporalidade aplicável; ela não é uma dispensa geral de documentação.
- **Transferências internacionais:** avaliar dados pessoais comuns e sensíveis, incluindo metadados, logs e disponibilização de acesso no exterior. Comprovar a base dos arts. 7º ou 11 e o mecanismo válido do art. 33, operacionalizado pela Resolução CD/ANPD nº 19/2024. Se o mecanismo for contratual, identificar o instrumento e as cláusulas efetivamente aplicáveis; um documento genérico de proteção de dados ou a ausência de dado sensível não basta [66][67].

#### 2.3.2 Segurança cibernética, contratação de nuvem e Open Finance Brasil

O Banco Central do Brasil (BCB) e o Conselho Monetário Nacional (CMN) têm atos com alcances diferentes. Classificar a pessoa jurídica regulada e o serviço relevante antes de escolher o ato; participação no Open Finance é outra condição de aplicabilidade.

- **Resolução CMN nº 4.893/2021, versão consolidada 3:** abrange as instituições autorizadas a funcionar pelo BCB, com as exclusões de seu art. 1º: administradoras de consórcio, instituições de pagamento, corretoras e distribuidoras de títulos e valores mobiliários e corretoras de câmbio. Seu art. 27 revoga integralmente as Resoluções CMN nº 4.658/2018 e nº 4.752/2019 [28c].
- **Resolução BCB nº 85/2021, versão consolidada 4:** seu art. 1º, na redação da Resolução BCB nº 552/2026, abrange instituições de pagamento, corretoras e distribuidoras de títulos e valores mobiliários, corretoras de câmbio e sociedades prestadoras de serviços de ativos virtuais autorizadas a funcionar pelo BCB. A exclusão de uma entidade da CMN nº 4.893 não a inclui automaticamente na BCB nº 85; outras entidades exigem o ato de seu próprio enquadramento [28b].
- **Serviços relevantes de processamento, armazenamento e nuvem:** quando abrangidos por um desses atos, mapear os arts. 12 a 17: avaliação prévia do fornecedor, requisitos e cláusulas contratuais, segurança, acesso da instituição e do BCB, subcontratação, continuidade e saída. O art. 15 exige comunicar a contratação e alterações pertinentes em até dez dias. Para serviços no exterior, o art. 16 exige condições documentadas, países e regiões previamente definidos e continuidade; na ausência do convênio previsto, solicitar autorização ao BCB com antecedência mínima de sessenta dias. Isso não estabelece uma proibição genérica de serviços fora do Brasil. Testes, resposta a incidentes e guarda dos documentos enumerados nos arts. 22-A e 23 também integram o controle aplicável [28b][28c].
- **Open Finance:** a Resolução Conjunta nº 1/2020, versão 9, estabelece o regime de implementação; a Resolução BCB nº 32/2020, versão 6, dispõe sobre requisitos técnicos e procedimentos operacionais e remete aos manuais [28][78]. Registrar a versão efetivamente aplicável dos manuais, do perfil de segurança, dos certificados e das jornadas [29][29b]. Os fundamentos gerais do perfil Financial-grade API (FAPI) e do vínculo entre token e certificado estão em [79][64], mas não substituem a verificação do perfil brasileiro vigente.

Implicação para APIM: compor os controles de §7.7 com a exposição, os dados, o consumidor e a criticidade. Em fluxos que exigem token vinculado a certificado por TLS mútuo (mTLS), declarar o ponto de terminação TLS e verificar o certificado original do consumidor; o exemplo direto A.4 não é intercambiável com uma rota por proxy de camada 7. Consentimento e autorização de negócio permanecem no sistema de domínio. Reter os conjuntos enumerados na norma, com seus marcos (§10.7), em vez de presumir cinco anos para todo log do gateway. Este guia não atesta conformidade do ecossistema nem certificação FAPI.

#### 2.3.3 Setoriais adicionais

- **PCI DSS** (Payment Card Industry Data Security Standard): se houver dados de cartão, identificar o ambiente de dados do portador (CDE), fornecedores e componentes que podem acessá-los ou afetar sua segurança, com avaliação do assessor aplicável. Distinguir TLS por salto, com terminação e nova conexão nos proxies de camada 7, de cifragem do payload ou tokenização que mantenha o número do cartão (PAN) opaco ao APIM. Essa última opção limita inspeção e validação do conteúdo e exige gestão de chaves e desenho próprio. Cifragem, inclusive de payload, não exclui automaticamente o componente ou os dados do escopo PCI; o PCI SSC também não impõe cifragem de dado em memória não persistente como regra universal [81][82]. Não registrar PAN nem dados de autenticação de cartão em logs. A presença de TLS ou de uma policy não certifica o desenho.
- **Saúde** (Lei nº 13.787/2018 sobre prontuário eletrônico, e LGPD dado sensível): retenção, auditoria e DPO específicos.
- **Telecom (Anatel), seguros (Susep), capital (CVM)**: cada qual com requisitos próprios. APIM deve registrar a classificação setorial da API na ficha.

#### 2.3.4 Mapeamento operacional

Toda API publicada no APIM corporativo deve, na ficha de intake, indicar:

1. Dimensões compostas (§7.7): exposição, dados pessoais comuns/sensíveis legais/financeiros, classificação corporativa, regimes setoriais, tipo de consumidor e criticidade. Registrar também a pessoa jurídica e a atividade que fundamentam a incidência do regime.
2. Finalidades do tratamento e bases LGPD, com artigo, inciso e evidência: art. 7º ou, quando aplicável, art. 11. Vincular a avaliação de risco e o RIPD exigido ou recomendado, distinguindo política interna.
3. Conjuntos de registros, campos necessários, finalidade, obrigação ou política aprovada, responsável, prazo e evento inicial da retenção (§10.7). Separar evidência de negócio ou regulatória de telemetria de diagnóstico.
4. Destinos de armazenamento, processamento e acesso, restrições setoriais/contratuais e, quando houver transferência internacional de dados pessoais, mecanismo do art. 33/ANPD nº 19 e instrumento que o comprova. Ausência de dado sensível não basta para aprovar o destino.
5. Responsáveis pelas decisões do controlador e pelos controles; aprovações de segurança, jurídico/privacidade e compliance setorial conforme as dimensões aplicáveis. O apoio do encarregado não transfere a responsabilidade do controlador.
6. Gatilho, relógio, responsáveis/substitutos, canais, comunicação à ANPD e aos titulares, complementação e guarda do registro de incidente, com as ressalvas de §2.3.1 efetivamente aplicáveis.

Sem esse mapeamento e as evidências dos controles aplicáveis, o gate bloqueia a publicação. A ficha e as extensões do contrato documentam decisões; não substituem a implementação ou o teste das policies.

---

## 3. Azure API Management: planos, tiers clássicos, Consumption, v2 e regiões

### 3.1 Visão geral

Azure API Management oferece um gateway gerenciado e recursos de administração e consumo. Conceitualmente, separa três planos:

- **Plano de gestão (control plane)**: Azure Resource Manager (ARM), Bicep/Terraform, portal e API administrativa, protegidos por controle de acesso baseado em papéis (RBAC, Role-Based Access Control).
- **Plano de execução (data plane/gateway)**: recebe requisições, aplica policies, autenticação, limitação de tráfego e transformação e encaminha ao backend.
- **Plano de desenvolvedor**: Developer Portal, documentação, integração de consumidores e subscriptions de consumo APIM, que não são as subscrições Azure de faturamento. A disponibilidade do portal depende do tier [1].

Capacidades centrais incluem policies XML (Extensible Markup Language), produtos, APIs, operações, named values, certificados, identidades gerenciadas, Azure Monitor e Azure Policy. Rede, multirregião, self-hosted gateway, workspaces e gateway de workspace dependem de tier, tipo de execução e região [1][5][54]. Defender for APIs é uma integração com onboarding e cobertura específicos, não um controle presente em todo gateway (§5.1). API Center é um serviço separado de inventário e governança de projeto; não executa o tráfego das APIs [10].

### 3.2 Terminologia: clássico/v1, stv1/stv2, Consumption e tiers v2

Os termos abaixo separam oferta comercial, compute e execução. Na seleção de workspaces, considerar também que recursos de certificados de autoridades certificadoras (`CA certificates`) não são suportados no workspace; a configuração de confiança necessária a mTLS deve ser validada no gateway e no caminho escolhidos, sem presumir equivalência com recursos do serviço [5].

- **Tiers clássicos de compute dedicado**: Developer, Basic, Standard e Premium, na plataforma stv2. stv1 foi aposentada; essa identificação histórica de compute não inclui Consumption nem os tiers v2 [2][4].
- **Tiers v2**: Basic v2, Standard v2 e Premium v2, geração distinta com unidades de capacidade, escala e recursos próprios. O uso de infraestrutura gerenciada não significa cobrança ou escala iguais ao Consumption; Premium v2 usa um App Service Environment dedicado [2].
- **Consumption**: oferta serverless com escala automática e cobrança por chamadas. Não é uma instância de máquina virtual (VM) na plataforma stv2. Não oferece VNet injection, self-hosted gateway ou WebSocket, e o suporte de policies/logs precisa de avaliação específica [1][54].
- **Workspaces**: limite lógico de delegação e RBAC para APIs, produtos, subscriptions e recursos relacionados. Existem em Basic v2, Standard v2, Premium clássico e Premium v2. O uso do **gateway gerenciado padrão** pelo workspace é documentado somente nos tiers v2 [1][5].
- **Workspace gateway**: recurso de execução separado associado ao workspace. Em Premium clássico, é necessário para executar suas APIs; não se substitui pelo gateway padrão. Tem escala, custo e requisitos de rede próprios, mas não oferece custom hostnames nem private endpoints de entrada. A região e a subscrição devem ser as da região primária e subscrição do serviço APIM. Sua disponibilidade deve ser confirmada; não está listado em Brazil South na consulta de 09/10/2026 [3][5].

> **Produtos, workspaces, gateways e instâncias fornecem limites diferentes.** Produtos agrupam consumo e policies, mas não isolam execução. Workspaces delegam recursos e administração; compartilhar o gateway mantém capacidade e falhas de execução compartilhadas. Workspace gateway separa execução, não todo o plano de gestão. Instâncias independentes permitem escolhas distintas de região, tier e operação, sem eliminar dependências externas compartilhadas. Premium clássico deve ser justificado pelos recursos necessários: multirregião nativa, self-hosted gateway de produção, injection ou backup/restore elegível. Backup/restore não é suportado em uma instância com workspace gateways associados (§10.5) [1][5][9].

### 3.3 Orientação por SKU/tier

| Tier | Quando faz sentido | Quando NÃO faz sentido |
|------|--------------------|------------------------|
| Developer | Sandbox, testes, treinamento e self-hosted gateway de avaliação | Produção: não oferece SLA [1] |
| Basic v2 | Carga cuja capacidade e controles sejam comprovados por teste; workspaces no gateway padrão | Requisitos obrigatórios de injection, multirregião nativa, self-hosted gateway ou backup/restore [1][2] |
| Standard v2 | Gateway com integração de saída à VNet, private endpoint de entrada, autoscale e workspaces no gateway padrão | Multirregião nativa, self-hosted gateway ou backup/restore; redundância zonal requer fechar o conflito documental de §10.5 [1][2][15][56][57] |
| Premium v2 | Capacidade e isolamento de rede que justifiquem injection, onde houver oferta regional; workspaces e múltiplos domínios do gateway | Multirregião nativa, self-hosted gateway e backup/restore não disponíveis na consulta de 09/10/2026 [1][2] |
| Premium clássico | Multirregião nativa, injection, self-hosted gateway de produção e backup/restore quando elegível | Workspaces no gateway padrão não suportados; workspaces exigem gateway separado e região elegível. Não há backup/restore com workspace gateways associados [5][9] |
| Consumption | Carga HTTP esporádica cujo conjunto de policies, logs, rede e latência seja compatível | Necessidade de VNet injection, WebSocket, self-hosted gateway ou funcionalidades ausentes no gateway Consumption [1][54] |

**Seleção de tier e runtime, com data-base documental de 09/10/2026:**
- Self-hosted gateway necessário em produção: Premium clássico; Developer apenas para avaliação [1][8].
- Multirregião nativa necessária: Premium clássico. Duas instâncias independentes são outra arquitetura de recuperação, não o mesmo recurso nativo [1][7].
- Federação com gateway compartilhado: workspaces no gateway padrão de Basic v2, Standard v2 ou Premium v2, com região e limites confirmados. Premium clássico não oferece essa associação [5].
- Federação com execução separada: avaliar workspace gateway elegível ou instâncias independentes. Em Premium clássico o gateway de workspace é necessário; não está listado em Brazil South. APIs/produtos no nível do serviço são alternativa de segmentação de consumo, sem a mesma delegação ou isolamento [3][5].
- Carga sem necessidade dos recursos exclusivos acima: comparar Basic v2, Standard v2, Premium v2 ou outra oferta elegível, segundo rede, políticas, carga testada e custo; volume elevado não seleciona Premium automaticamente.
- Brasil com injection obrigatório: na ausência de Premium v2 listado na região exigida, avaliar Premium clássico. Brasil com rede privada de entrada/saída sem injection obrigatório pode usar Standard v2 conforme os requisitos de private endpoint e integração de saída. Residência Brasil isoladamente não exige Premium [2][3][15].

### 3.4 Disponibilidade regional e nota específica sobre Brasil

Disponibilidade documental e capacidade de contratação são verificações diferentes. A matriz de tiers v2/workspace gateways consultada em **09/10/2026**, com atualização indicada em 28/09/2026, contém limitações temporárias de criação em algumas regiões. Anexar à ADR a oferta, região, runtime e capacidade confirmados antes da contratação [3][5].

- **Brazil South:** Basic v2 e Standard v2 listados; Premium v2 e workspace gateway não listados. Federação lógica por workspaces nos tiers v2 não depende de afirmar a existência de workspace gateway no Brasil [3][5].
- **Brazil Southeast:** não aparece na matriz consultada de v2/workspace gateways. A oferta clássica, capacidade, permissões de acesso à região e possibilidade de uma topologia brasileira com duas regiões devem ser verificadas especificamente; não inferir restrições a partir do nome "Southeast".

Implicações:

1. Quando existir restrição brasileira legal, setorial, contratual ou corporativa efetivamente identificada, avaliar cada conjunto de dados, metadados e processamento. LGPD não determina residência Brasil universal, e dados financeiros não são automaticamente dados pessoais sensíveis legais. Selecionar um tier que atenda aos requisitos na região permitida, não um tier apenas pelo rótulo regulatório [66].
2. Multirregião com destino no exterior requer aprovação dos países/regiões, acessos, dados e salvaguardas aplicáveis. Premium clássico oferece o recurso nativo; Premium v2 não o oferece. Aceite interno não substitui o mecanismo de transferência internacional nem as condições setoriais/contratuais [7][66][67].
3. API Center não lista regiões brasileiras na consulta de 09/10/2026 [10]. Seu destino deve passar pelo gate de §10.2.3 para metadados, documentos, exemplos e acessos. Ausência de dados sensíveis não basta: dados pessoais comuns também podem exigir mecanismo de transferência, e dados não pessoais podem ter restrição contratual. Se não houver destino permitido, manter inventário em solução autorizada e fasear API Center mediante decisão registrada [41][66][67].
4. Workspaces delimitam administração e recursos, mas não definem residência. Gateway padrão, workspace gateway, plano de gestão, catálogo, backends e destinos de telemetria precisam de decisões de localização e acesso próprias. A execução dedicada não elimina o compartilhamento do serviço APIM [5].

Disponibilidade documental de **09/10/2026** para v2/workspace gateways [3] e API Center [10]. "Listado" não assegura capacidade comercial de criação; "não listado" não prova indisponibilidade permanente.

| Recurso | Brazil South | Brazil Southeast | East US | West Europe |
|---|---|---|---|---|
| Basic v2 | Listado | Não listado | Listado | Listado |
| Standard v2 | Listado | Não listado | Listado | Listado |
| Premium v2 | Não listado | Não listado | Não listado | Não listado |
| Workspaces no gateway padrão v2 | Recurso lógico em Basic v2/Standard v2 | Depende de oferta v2 confirmada | Recurso lógico nos tiers v2 listados | Recurso lógico nos tiers v2 listados |
| Workspace gateway separado | Não listado | Não listado | Não listado | Listado; validar oferta/capacidade |
| Premium clássico¹ | Disponível na referência histórica; revalidar | Verificação específica necessária | Disponível na referência histórica; revalidar | Disponível na referência histórica; revalidar |
| API Center | Não listado | Não listado | Listado | Listado |

¹ A linha clássica preserva a referência histórica de 18/06/2026, sem representar nova confirmação comercial. Sua oferta/capacidade e a topologia em Brazil Southeast devem ser confirmadas antes de contratação. A matriz [3] cobre v2/workspace gateways, não substitui a consulta de oferta clássica. Um workspace Premium clássico exige o gateway separado; não interpretar o recurso lógico como disponível no gateway padrão [5].

O líder de arquitetura Azure registra na ADR data, URL, tier, runtime, região, limitações de capacidade e confirmação comercial pertinente. Revisão trimestral e antes de contratação, alteração de região ou adoção de um novo recurso; não se presume que ADR-APIM-001 já contenha essas evidências.

---

## 4. Modelos de adoção (corporativo, por subscrição/domínio, híbrido) com matriz de decisão

Grandes empresas adotam três modelos principais. A adequação de cada um depende de regulação, escala, autonomia, governança e maturidade.

### 4.1 Modelo A: APIM corporativo compartilhado

Uma ou poucas instâncias APIM, operadas pela equipe de plataforma, expõem APIs de múltiplos domínios. **Produtos** segmentam consumo e policies, sem isolar execução; **workspaces** segmentam recursos e RBAC. Basic v2/Standard v2/Premium v2 podem associar workspaces ao gateway padrão. Premium clássico exige workspace gateway, que não está listado em Brazil South na consulta de 09/10/2026. Gateway dedicado separa execução, mas mantém dependências do serviço APIM; quando a separação exigida incluir região, plano de gestão, manutenção ou operação, avaliar instâncias independentes [1][3][5].

Vantagens: padronização forte; custo unitário menor; um único ponto de auditoria, catálogo e SLOs; superfície reduzida de exposição pública.
Riscos: gargalo organizacional na plataforma; mistura de cargas com criticidade distinta; janelas de manutenção compartilhadas; uso de policies sem considerar o impacto nos recursos compartilhados.
Sinais favoráveis: maturidade central forte, regulação homogênea entre domínios, volume médio.
Sinais desfavoráveis: domínios com SLO/regulação heterogêneos, parceiros B2B exigindo isolamento, on-call distribuído sem suporte da plataforma.

### 4.2 Modelo B: APIM por subscrição, projeto ou domínio

Cada domínio/squad opera sua instância APIM, dentro dos guardrails corporativos. Governança via Azure Policy e modelos IaC versionados; "modelos" não significa dependência do produto Azure Blueprints.

Vantagens: autonomia sobre a instância; menor alcance de falhas de execução entre domínios; flexibilidade de tier/região quando permitida; policies por domínio. Identidade, rede, backend ou telemetria compartilhados continuam podendo propagar incidentes.
Riscos: custo agregado alto; padronização inconsistente; catálogo e observabilidade fragmentados; superfície de exposição ampliada; dificuldade de aplicar uma política regulatória uniforme.
Sinais favoráveis: domínios autônomos com plataforma própria, regulação muito heterogênea (ex.: parte da empresa em Open Finance, parte não), Fusões e aquisições (M&A) com integração ainda em curso.
Sinais desfavoráveis: corporação com governança centralizada, equipe de plataforma pequena, requisitos regulatórios uniformes.

### 4.3 Modelo C: híbrido/federado governado, candidato para plataforma central madura

A equipe de plataforma opera instâncias compartilhadas (modelo A) e habilita instâncias dedicadas quando requisitos demonstrados de região, operação, alcance de falhas, rede ou capacidade justificarem o custo e a responsabilidade adicionais. A governança via Azure Policy e modelos IaC versionados vale para ambas. Catálogo e observabilidade têm gestão unificada, mas podem usar destinos segregados conforme localização, retenção e responsabilidades aprovadas (§10.2.3).

API Center é a opção de inventário corporativo quando existir uma região permitida e disponível. Em 09/10/2026 não lista região brasileira; metadados, anexos e acessos internacionais precisam de aprovação própria. Sem destino permitido, usar inventário autorizado e fasear o serviço, sem dispensar catálogo nem impor transferência de metadados [10][41][66][67].

Vantagens: padrões compartilhados com autonomia justificada e consolidação de inventário e operação. A relação custo/risco deve ser demonstrada para o contexto, não presumida pelo nome do modelo.
Riscos: governança mais complexa, capacidade central insuficiente e proliferação de instâncias sem critério. Cada criação passa pelo gate de requisitos e operação.
Sinais favoráveis: controles centrais maduros, caminho de autosserviço, limites de isolamento claros e equipes capazes de operar as exceções.

### 4.4 Self-hosted gateway para cenários híbridos

Quando backends estão no datacenter próprio, em outra nuvem ou próximos a uma operação com exigência de baixa latência, o **self-hosted gateway** leva a execução ao ambiente do cliente, mantendo a gestão no Azure. É suportado em Premium clássico para produção e em Developer para avaliação; não nos tiers v2 ou Consumption na consulta documental de 09/10/2026 [1][54].

Casos típicos: conectividade híbrida, processamento próximo ao backend e operação multicloud. A localização do runtime não prova residência de toda a solução: configuração, identidade, catálogo, sincronização e telemetria continuam sujeitos ao gate regional.

O cliente opera containers, disponibilidade, capacidade, atualização, sincronização e coleta de telemetria, com custos próprios além da oferta APIM/gateway contratada. Defender for APIs não cobre APIs expostas pelo self-hosted gateway; exigir logs e detecção alternativas integradas ao centro de operações de segurança (SOC), conforme §8.4 e §10.1 [18][54][58].

### 4.5 Matriz de decisão pontuada (com critérios eliminatórios)

A **matriz pontuada com critérios eliminatórios** permite comparar os modelos com critérios explícitos e reduzir a subjetividade.

#### 4.5.1 Critérios eliminatórios

Primeiro caracterize configurações factíveis para cada modelo: tier, região, tipo de gateway, rede, limites compartilhados, equipe e operação. Descarte uma **configuração** que não atenda a um requisito obrigatório aplicável; descarte um modelo somente se nenhuma configuração permitida atender. Recurso dispensável é "não aplicável", não reprovação. Se nenhuma alternativa atender, escalar a restrição antes de pontuar.

| Requisito eliminatório | Pergunta aplicada à configuração | Condição de descarte |
|---|---|---|
| Localização e oferta | Região, recursos, dados, metadados e acessos são permitidos, disponíveis e com capacidade confirmada? | Não existe configuração contratável que atenda à localização aprovada. |
| Controles aplicáveis | A composição dados/setor/exposição/consumidor e o perfil exigido podem ser atendidos e operados? | Falta controle obrigatório, incluindo cobertura ou alternativa de detecção aceita; habilitar um plano isoladamente não prova atendimento. |
| Continuidade e capacidade | O desenho e os ensaios atendem SLO, modos de falha, capacidade e objetivos de recuperação acordados? | Topologia ou operação não atende requisito necessário. Um percentual de SLO não implica, sozinho, multirregião nativa. |
| Gateway local | Quando self-hosted gateway for necessário, tier e operação o suportam? | Gateway necessário não suportado ou sem operação adequada. Se dispensável, o requisito não elimina a alternativa. |

#### 4.5.2 Comparação ponderada e calibração com o cliente

Escala local: 0 = ruim, 1 = aceitável, 2 = bom, 3 = ótimo. As notas e os pesos abaixo são **hipóteses ilustrativas**, não medições da organização nem propriedades universais dos modelos. A comparação só se aplica às configurações que passaram pelos requisitos eliminatórios.

Os oito critérios, na ordem das colunas, são padronização técnica, autonomia de squads, custo agregado, alcance de falhas/isolamento, velocidade de entrega, governança dos controles aplicáveis, catálogo unificado e capacidade operacional/plantão. Os pesos correspondentes são 3, 3, 3, 3, 2, 3, 2 e 2.

<!-- check: weighted weights=3,3,3,3,2,3,2,2 -->
| Modelo | Padronização | Autonomia | Custo | Isolamento | Entrega | Governança | Catálogo | Operação | Total |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| A: corporativo compartilhado | 3 | 1 | 3 | 1 | 1 | 3 | 3 | 3 | 47 |
| B: por domínio | 1 | 3 | 1 | 3 | 3 | 1 | 1 | 1 | 37 |
| C: híbrido governado | 3 | 2 | 2 | 2 | 2 | 3 | 3 | 2 | 50 |

Total = soma de peso × nota; máximo = 3 × (3+3+3+3+2+3+2+2) = **63**. Faixas ilustrativas para esse máximo: 0–30 desfavorável, 31–44 aceitável com ressalvas e 45–63 favorável. Mudança de pesos ou escala exige recalcular máximo e recalibrar faixas, sem conservar automaticamente o corte 45.

Arquitetura, FinOps, segurança e responsáveis pela operação registram na ADR evidência e motivo de cada nota/peso: custo estimado, limites compartilhados, maturidade, equipe e plantão. Revisar a correlação entre critérios para não contar duas vezes o mesmo benefício. A nota operacional de B pode aumentar se o domínio comprovar plataforma e plantão próprios.

**Sensibilidade:** a vantagem inicial C–A é de três pontos. Mantendo as notas e elevando de 3 para 5 o peso de custo e de 2 para 4 o de operação, A passa a 59, B a 41 e C a 58, com máximo 75. Esse cenário hipotético inverte A/C, não descreve o cliente. Comparar prioridades plausíveis e incerteza antes da escolha.

Modelo C permanece uma hipótese para plataforma central madura com exceções operáveis. Modelos A ou B podem ser preferíveis após calibração; registrar a decisão e as condições que a mudariam na ADR.

---

## 5. Arquitetura de referência

A arquitetura combina o gateway APIM com a landing zone Azure, identidade Microsoft Entra ID, observabilidade em destinos aprovados e pipelines de infraestrutura como código (IaC, Infrastructure as Code). Cada componente tem escopo e condições próprios; o gateway não substitui a engenharia do backend nem o gate regional.

### 5.1 Componentes recomendados

- **APIM** (clássico Premium ou v2 conforme §3.3/§3.4).
- **WAF na entrada externa**, por política corporativa deste guia; o backend/origem APIM deve impedir contorno do perímetro.
  - **Azure Front Door com WAF** para entrada global, distribuição de conteúdo (CDN, Content Delivery Network), cache de borda e roteamento entre origens saudáveis [69].
  - **Application Gateway WAF v2** para entrada regional e roteamento HTTP por host/caminho, com acesso ao APIM privado conforme as opções de rede do tier [15].
  - **Combinação de ambos** somente quando os controles globais e regionais a justificarem. Proxies de camada de aplicação (L7) terminam TLS e criam outros segmentos TLS; recriptografia não é passthrough do certificado consumidor. Em mTLS, escolher onde autenticar o cliente e verificar o vínculo entre token e certificado original, com origem restrita e metadados autenticados quando usados. Application Gateway não encaminha automaticamente o certificado original ao APIM [19][64].
- **Microsoft Entra ID** para autenticação humana (publicadores, desenvolvedores) e identidades de cargas de trabalho.
- **Key Vault** para segredos, certificados e chaves de assinatura/encriptação.
- **Log Analytics + Application Insights** para sinais de runtime; **Sentinel** para correlação de segurança.
- **Defender for APIs**, quando elegível, para análise e detecção sobre APIs REST efetivamente integradas ao plano. Habilitar o plano sem onboarding não comprova proteção; workspaces e self-hosted gateway não são cobertos, e a FAQ exclui Consumption. Manter inventário independente e controles alternativos para tipos/runtimes excluídos; não prometer descoberta universal de APIs fora do inventário. O serviço pode receber cópias parciais de corpos de requisição/resposta: avaliar finalidade, coleta, destino e acesso, mesmo que body logging esteja desligado no gateway [25][58][72].
- **API Center** como inventário e catálogo de projeto, fora do caminho de execução. Região, metadados, anexos e acessos passam pelo gate de §10.2.3, inclusive para dados pessoais comuns e restrições contratuais. Em 09/10/2026 não há região brasileira listada. Sem destino permitido e disponível, usar inventário autorizado e fasear API Center; não autorizar região externa apenas pela ausência de dados sensíveis [10][41][66][67].
- **Azure Policy** para controles de governança (nomenclatura, tags, SKUs permitidos, criptografia, restrição de região).
- **Bicep/Terraform + GitHub Actions/Azure DevOps** para IaC e pipelines APIOps.

### 5.2 Padrões de integração

- **APIM → backend**: a política corporativa prioriza conectividade privada para backends Azure quando suportada pelo tier e pelo backend. Injection, integração de saída e private endpoint são mecanismos diferentes; um endpoint privado de entrada não cria saída privada. Backends externos exigem HTTPS e controles definidos pelo risco/perfil, incluindo mTLS quando necessário [15].
- **APIM ↔ Microsoft Entra ID**: validar tokens OAuth 2.0 com emissor, audiência, aplicação e permissões esperadas. JSON Web Token (JWT) é um formato de token; OIDC acrescenta identidade em cenários de usuário. Importar o contrato não instala as policies de autenticação/autorização [16][55][65].
- **APIM ↔ Defender for APIs**: inventário e detecção somente no escopo suportado, com onboarding comprovado por API/gateway e revisão das exclusões. Telemetria alternativa permanece obrigatória para workspaces e self-hosted gateway [25][58].
- **APIM ↔ destinos de observabilidade** com categorias e coletores suportados e aprovados. Separar logs de requisição, atividade administrativa e evidência de segurança. Workspaces, Consumption e self-hosted têm fontes/limites específicos; não exigir a mesma tabela de gateway logs em todos os runtimes (§10.1) [18][31][36].
- **APIM ↔ API Center** para sincronizar inventário [10][24].
- **APIM ↔ Sentinel** via Log Analytics como pipeline padrão para gestão de informações e eventos de segurança (SIEM).

### 5.3 Diagrama lógico (descrito)

```
[Consumidor externo]
       │
       ▼
[WAF escolhido: Front Door global OU Application Gateway regional]
       │  (combinação de ambos somente quando justificada)
       ▼
[Gateway APIM escolhido, com origem protegida]
       │
       └──► [App Service / AKS / Functions: backends em rede aprovada]

[Consumidor do datacenter próprio / filial / outra nuvem]
       │
       ▼
[Self-hosted gateway: execução em containers operados pelo cliente]
       │
       └──► [Backends locais / outras nuvens]

[Plano de gestão]
   Entra ID / RBAC / Key Vault ──► configuração dos gateways
   Inventário autorizado / API Center ──► catálogo e sincronização configurada

[Observabilidade e segurança]
   Telemetria por runtime ──► destinos aprovados / SOC
   Defender for APIs ──► apenas APIs REST elegíveis com onboarding
```

**Legenda e limites:** WAF é o firewall de aplicações web; AKS é Azure Kubernetes Service; RBAC é controle de acesso baseado em papéis; SOC é o centro de operações de segurança. Rede virtual (VNet) e Private Link dependem do gateway/tier. API Center está fora do caminho da requisição. Os blocos de catálogo e telemetria representam integrações configuradas, não descoberta ou exportação automáticas universais.

O self-hosted gateway **não é um backend do gateway gerenciado**. É uma execução alternativa perto dos backends híbridos/multicloud, associada à gestão APIM [8][54]. O consumidor entra no gateway adequado ao cenário. Workspaces e self-hosted gateway não têm cobertura Defender; usam detecção e telemetria alternativas [58]. Cada proxy de entrada tem terminação TLS própria; em mTLS, a identidade do certificado consumidor precisa ser verificada no ponto escolhido, sem presumir sua passagem automática até APIM [19][64].

---

## 6. Padrões de API, ciclo de vida, versionamento, catálogo e consumo

### 6.1 Princípios de desenho de API

- **Contrato obrigatório por protocolo**, aprovado antes da publicação (§6.2/§6.6). Para HTTP/REST, adotar OpenAPI **3.0.3**. APIM importa OpenAPI 3.1 com compatibilidade de importação, não de todos os recursos; exceções exigem conferir o artefato importado, schemas, operações, testes e perda de campos. Definições `security` não instalam autenticação: policies e backend devem implementar os controles aprovados [55][62].
- **REST como padrão corporativo**; GraphQL, WebSocket, SOAP, OData ou gRPC exigem contrato adequado e prova de suporte do gateway/tier/região. O Apêndice B é um template HTTP/REST, não o contrato principal de todos os protocolos.
- **Erros padronizados** (RFC 7807 problem+json).
- **Versionamento explícito** (caminho `/v1`, `/v2`).
- **Idempotência** documentada por verbo (PUT idempotente; POST não, mas pode oferecer chave de idempotência).
- **Nomenclatura consistente** (kebab-case em paths; camelCase em payload JSON).

### 6.2 Avaliação inicial corporativa de arquitetura e publicação

Toda API nova passa por uma avaliação inicial formal (intake). Este guia usa os rótulos MPAS (Microsoft Partner Architectural Solution) e MAP (Microsoft Architecture Plan) para os modelos de avaliação citados; não os trata como norma, certificação ou obrigação imposta pela Microsoft. A organização formaliza nome, responsável, campos e aprovação do template efetivamente adotado. A avaliação contém:

- Responsável pela API/produto, equipe, suporte, plantão e responsáveis por dados e controles aplicáveis.
- Dimensões compostas: exposição (`internal|partner|public|admin`), dados pessoais comuns, dados sensíveis legais, classificação corporativa, setor/regimes, consumidor e criticidade. Dados financeiros não são automaticamente sensíveis pelo art. 5º, II, da LGPD [66].
- Finalidades e conjuntos de dados, hipótese legal pertinente (art. 7º ou art. 11 quando aplicável), restrições contratuais, política de retenção aprovada e avaliação de transferências/acessos internacionais [66][67].
- SLO e acordo de nível de serviço (SLA, Service Level Agreement), escopo de disponibilidade/latência e objetivos de recuperação por objeto (§10.4/§10.5).
- Pico, concorrência e perfil de carga: transações por segundo (TPS), requisições por segundo e, quando útil, usuários ativos mensais (MAU, Monthly Active Users), payloads e latência do backend.
- Tipo de consumidor e fluxo de identidade: client credentials para acesso máquina a máquina pertinente; Authorization Code com PKCE para acesso delegado por usuário. Subscription key identifica um contrato de consumo, não substitui identidade/autorização. FAPI/mTLS dependem do perfil exigido ou da política corporativa explicitamente aprovada [65].
- Protocolo, contrato principal, gateway/tier/região, rede, backends, dependências e cobertura de segurança/telemetria.
- Plano de deprecação, migração de consumidores e retirada (sunset), com prazo e responsáveis.

Sem intake aprovado, a API não pode ser publicada.

### 6.3 Ciclo de vida recomendado

| Estado | Critério de entrada | Critério de saída |
|---|---|---|
| **Design** | Intake aprovado | Contrato principal adequado ao protocolo (§6.6) + ADR; OpenAPI 3.0.3 para HTTP/REST |
| **Develop** | Contrato principal aprovado pelo responsável | Build, testes unitários, lint e verificações do protocolo/gateway concluídos; autenticação/autorização configuradas separadamente |
| **Pré-produção/homologação (UAT, User Acceptance Testing)** | Artefato promovido via pipeline | Validação do consumidor, segurança, compatibilidade e desempenho com evidências do contrato e gateway escolhidos |
| **Produção** | Aprovação de release | Em uso pelos consumidores |
| **Deprecated** | Substituta publicada (`/v2` ativa) | Comunicação a consumidores |
| **Sunset** | Janela de deprecação encerrada | Remoção do APIM e do catálogo |

### 6.4 Versionamento e revisions

- **Versão** para mudanças incompatíveis. O padrão corporativo é **path versioning** (`/v1`, `/v2`) por legibilidade, roteamento simples, cache/WAF previsíveis e menor ambiguidade operacional.
- **Revision** para mudanças compatíveis dentro da mesma versão. Permite rollback rápido sem mudar contrato público.
- **Política corporativa inicial de deprecação**: comunicar com pelo menos 90 dias de antecedência da retirada, salvo prazo normativo/contratual maior ou tratamento emergencial autorizado que não dispense obrigação externa. Registrar datas distintas de deprecação e sunset, avisos no portal e comunicação aos consumidores. `Deprecation` segue RFC 9745; `Sunset` segue RFC 8594 [70][71]. Uma major interna compatível não muda a versão pública; uma mudança de componente que quebre autenticação ou comportamento consumido exige coexistência/migração e observância desses compromissos antes da retirada, não atualização automática apenas pela idade da major (§9.5.3).

Vantagens e riscos que devem ser registrados em ADR quando o padrão `/v{n}` não for usado:

| Estratégia | Vantagens | Riscos / custo operacional | Quando aceitar |
|---|---|---|---|
| **Path** (`/v1/pagamentos`) | Claro para consumidores; fácil roteamento no APIM/WAF; adequado para documentação, cache, analytics e coexistência | Acrescenta a versão à URL; pode incentivar versionamento excessivo | **Padrão** para APIs públicas, B2B, mobile e reguladas |
| **Query** (`?api-version=1`) | Útil quando o path é imposto por legado; facilita testes em paralelo | Cache/proxy podem ignorar query; consumidores esquecem parâmetro; menos explícito em logs | APIs internas/legadas com contrato já estabilizado e gateway impondo valor padrão/validação |
| **Header** (`api-version: 1`) | Preserva URL; permite negociação avançada | Menos visível; difícil para browsers/CDN/WAF; maior chance de erro em suporte | APIs internas entre serviços maduros, com kit de desenvolvimento de software (SDK) controlado e observabilidade preparada |

Regra: mudança incompatível exige nova **versão**; mudança compatível usa **revision**. Não usar versionamento por query/header para ocultar uma mudança incompatível.

### 6.5 Produtos, catálogo e consumo

- **Produtos** agrupam APIs, termos de uso e policies de consumo, como quotas e limites de tráfego. Escopos OAuth exigem validação explícita nas policies/backend; produto ou `security` OpenAPI não a instala automaticamente. Produtos não são limites de isolamento de runtime [5][55].
- **Prática proibida:** produto genérico **"All APIs"**, **"Unlimited"** ou equivalente em produção. Ele impede delimitar responsabilidades, impacto de falhas, chargeback e governança de quota. Exceção só em sandbox temporário com data de expiração.
- **Developer Portal** personalizado para a corporação: identidade visual, fluxo de adesão por autoatendimento, gestão de subscriptions, exemplos, sandbox.
- **API Center** para inventário, responsáveis, ciclo de vida, contratos e classificações. Região, metadados, anexos e acessos passam por §10.2.3; falta de região corporativa disponível não autoriza transferência automática. Sem destino permitido, manter inventário autorizado e fasear API Center em ADR. Revisar oferta e aprovação trimestralmente e antes de mudança material [10][41][66][67].
- **Sandbox** segregado: ambiente onde devs externos/parceiros testam sem credenciais de produção.

Metadados mínimos por **API** (registrados no OpenAPI/API Center/intake):

| Campo | Obrigatório | Uso |
|---|---|---|
| `owner` / `apiProductOwner` | Sim | Responsável técnico e decisor de roadmap |
| `supportContact` / `oncall` | Sim | Incidente e comunicação com consumidores |
| `slaTier` e SLO | Sim | Gate de arquitetura, alertas e contrato de consumo |
| `dataClassification` / `personalDataCategory` / `regulatoryClass` | Sim | Classificação corporativa, dado pessoal comum/sensível legal e regimes aplicáveis como dimensões distintas; base/finalidade quando houver tratamento pessoal |
| `exposure` / `consumerType` / `criticality` | Sim | Exposição, consumidor máquina/usuário e criticidade; compor controles sem escolher uma classe única |
| `retentionPolicyId` / `dataResidencyDecisionId` | Sim | Política aprovada por conjunto/finalidade e decisão de destinos/acessos; fonte, responsável, prazo, marco inicial e versão |
| `lifecycleState` | Sim | Design, active, deprecated, sunset |
| `costAllocation` / `costcenter` | Sim | Showback/chargeback com pool e período definidos |
| `mpasMapId` | Sim | ID do intake corporativo adotado e vínculo à ADR; nome do campo legado, não certificação |

Metadados mínimos por **produto**: responsável, APIs incluídas, consumidores-alvo, termos de uso, quotas/limites, SLA/SLO, suporte, dimensões de dados/regimes permitidas, políticas de retenção/localização pertinentes, pool de custo e data de revisão. O catálogo não contém credenciais ou payloads pessoais reais como exemplos.

A quota considera **jornada e consumidor**, além de capacidade técnica. Como exemplo de parametrização, consulta de saldo pode ter 100 requisições por minuto e um milhão por mês; iniciação de pagamento pode ter 20 por minuto e quota diária menor. Esses valores não são padrão de mercado nem cotação de capacidade. mTLS e aprovação manual dependem do perfil/requisito aplicável. O gate exige limite explícito, escopo do contador e evidência de suporte por gateway; para consumo anônimo, não presumir que quota por subscription key controle chamadas sem chave.

### 6.6 Protocolos e gateway para inteligência artificial

REST (Representational State Transfer) é o padrão de desenho de recursos HTTP adotado; SOAP usa mensagens/contratos próprios. APIM também atende outros protocolos e chamadas de inteligência artificial (AI, Artificial Intelligence). Suporte e limitações dependem de tier, tipo de gateway e região, não apenas do nome comercial. Confirmar contrato, transporte e policies antes de escolher [54].

| Protocolo / capacidade | Suportado em | Limitações | Quando faz sentido |
|---|---|---|---|
| **GraphQL pass-through** | Todos os tiers; conferir gateway e transporte [54][73] | Encaminha operações ao backend existente. `validate-graphql-request` permite limites de tamanho/profundidade e regras por caminho de consulta; não substitui autorização de objeto no backend [74] | Fachada GraphQL com contrato, limites e controles testados |
| **GraphQL synthetic** | Todos os tiers, com as restrições documentadas [73] | Não suportado em workspaces; synthetic subscriptions estão em preview e não disponíveis em Consumption. Resolvers HTTP/Cosmos DB/Azure SQL exigem autorização, disponibilidade e testes próprios | Schema GraphQL sobre backends existentes quando o acoplamento e a operação forem aceitáveis |
| **WebSocket APIs** | **Developer, Basic, Basic v2, Standard, Standard v2, Premium, Premium v2**; não Consumption [1] | Pass-through; policies só no `onHandshake`; conexões 1:1 não são distribuídas entre múltiplos backends; CORS/validate-content e várias policies não se aplicam ao handshake | Streaming client↔server, cotações, notificações persistentes |
| **OData pass-through** | Gateways clássico, v2, Consumption, self-hosted e workspace na matriz consultada [54] | Encaminhamento HTTP não é validação nativa da semântica OData; testar queries, limites e autenticação | Fachada de APIs SAP/Dynamics ou outro backend OData |
| **gRPC pass-through** | Gateway clássico em preview e self-hosted na matriz consultada [54] | v2, Consumption e workspace não estão marcados nessa matriz; confirmar transporte, backend e condição de preview antes do compromisso | Integração RPC quando o gateway escolhido tiver suporte comprovado |
| **AI gateway: limites de tokens e cache semântico** | Conferir a policy e o tipo de gateway, não apenas o tier [12][61] | Cache semântico usa Azure Managed Redis ou outro cache externo compatível com RediSearch. `llm-semantic-cache-lookup` não suporta workspace gateway. Isolar cache por consumidor/dados e testar qualidade, atualização e rede; custos de modelo, cache e APIM são componentes distintos | Governança de chamadas a modelos de linguagem, quando as policies forem elegíveis |
| **MCP server gateway** | Conferir capacidade, estágio de disponibilidade e gateway na documentação vigente [12] | Model Context Protocol (MCP) integra ferramentas a agentes; validar versão de protocolo, SDK, autenticação e policies antes da publicação | Ferramentas corporativas expostas com contrato e controle de acesso |

**Contrato principal e aceite por protocolo:**

| Protocolo | Artefato principal / adicional | Verificações e evidência mínima |
|---|---|---|
| HTTP/REST | OpenAPI 3.0.3; 3.1 apenas como exceção testada | Lint, operações/schemas importados, compatibilidade, testes e policies de autenticação/autorização separados [55][62] |
| GraphQL | Schema SDL (Schema Definition Language), operações/resolvers; OpenAPI apenas para eventual fachada HTTP distinta | Schema, limites de consulta, autorização de campos no backend, resolvers, subscription/transporte e suporte do gateway |
| WebSocket | Contrato de handshake, mensagens/eventos e fechamento; schema de mensagem quando pertinente | Testes de conexão, autenticação no handshake, limites de conexões/mensagens e reconexão; não tratar OpenAPI como descrição completa do fluxo |
| SOAP | WSDL (Web Services Description Language) e schemas associados | Importação, binding, transformação se usada e testes de operações no gateway escolhido |
| OData | Modelo/metadados do serviço e contrato HTTP; OpenAPI adicional se houver fachada descrita | Queries/opções admitidas, autorização, paginação e custo no backend; pass-through não valida a semântica OData |
| gRPC | Arquivos `.proto`, serviços/métodos e versão de transporte | Evidência do tipo de gateway, preview quando aplicável, compatibilidade e testes completos de tráfego |
| AI/MCP | Contrato HTTP/REST ou protocolo de ferramenta, versão e schemas pertinentes | Policies por gateway, isolamento de dados/cache, identidade e testes de consumo |

O líder de APIs registra na ADR a evidência por tier, gateway e região, com data-base documental de 09/10/2026 para esta matriz. Revisão trimestral e antes de novo recurso ou contratação. Publicação depende das evidências, não de uma data de roadmap ou de importar um arquivo.

Controles específicos:
- **GraphQL público/B2B**: `validate-graphql-request` com `max-size`, `max-depth` e regras de autorização por caminho; bloqueio de introspection (`/__*`) quando aprovado para o cenário. Registrar limites e testes negativos; manter autorização de objetos/campos de domínio no backend. Logs de identificação/hash de consulta somente conforme finalidade e classificação, sem query pessoal completa nem credenciais [74].
- **GraphQL pass-through vs synthetic**: pass-through preserva backend e reduz transformação; synthetic aumenta acoplamento ao APIM e exige testes de resolvers, cache e autorização de campo.
- **Subscriptions GraphQL/WebSocket**: acordar timeout, limites de conexões e mensagens/eventos, reconexão e runbook de desconexão. Aplicar cada limite no componente que efetivamente o suporte; policies do handshake APIM não implementam automaticamente quota por mensagem no fluxo estabelecido. Testar o contrato de transporte e o backend/gateway escolhidos [54][73].
- **Query allowlist**: para APIs públicas/reguladas, preferir persisted queries/allowlist por hash quando o conjunto de queries é conhecido.

Decisão APIM vs integração assíncrona:

| Necessidade | Use APIM quando… | Use Service Bus / Event Grid / Event Hubs / plataforma de integração como serviço (iPaaS) quando… |
|---|---|---|
| Request/response síncrono | Consumidor espera resposta imediata, contrato HTTP, auth/throttling/catálogo | Não se aplica |
| Comando assíncrono | Fachada HTTP aceita o comando e retorna `202 Accepted` com identificador de correlação; o processamento ocorre fora do gateway | Fila, retentativa, fila de mensagens não processáveis (DLQ, dead-letter queue), ordenação e controle da pressão de entrada conforme a capacidade do consumidor (backpressure), com produtor/consumidor desacoplados: avaliar Service Bus |
| Evento de domínio | APIM pode publicar fachada de consulta/status | Precisa fan-out reativo, roteamento por evento e assinatura de múltiplos consumidores (Event Grid) |
| Telemetria/stream | APIM só protege endpoint de ingestão leve | Alto volume, particionamento, retenção curta e processamento streaming (Event Hubs) |
| Integração de software como serviço (SaaS) entre empresas (B2B) | APIM expõe contrato externo governado | Transformações/processos longos/conectores gerenciados são centrais (iPaaS/Logic Apps) |

Regra: APIM governa **contratos síncronos** e fachadas HTTP. Não usar APIM como fila, barramento, orquestrador de processo longo ou broker de eventos.

---

## 7. Segurança, identidade e políticas

### 7.1 Plano de gestão

- **Microsoft Entra ID** para acesso ao plano de gestão, incluindo portal e APIs do Azure Resource Manager (ARM). Identidades de carga de trabalho devem ter credenciais protegidas ou federação apropriada ao processo.
- **RBAC** (controle de acesso baseado em papéis) no menor escopo necessário. Selecionar role pelas ações permitidas: Service Reader lê; Service Operator administra o serviço, mas não publica entidades como APIs e policies; publicação exige permissão específica, conforme §8.3.
- **PIM** (Privileged Identity Management) para acesso humano privilegiado elegível, com ativação temporária e aprovação. Papéis de aplicação em um token de API não são papéis ARM nem demonstram ativação PIM.
- **Trilha de gestão** com eventos de mudança, responsável, aprovação e artefato implantado, correlacionada aos registros AzureActivity/Log Analytics e ao Microsoft Sentinel. A retenção e a integridade seguem §10.7; a correlação no Sentinel não produz automaticamente assinatura criptográfica de logs.

### 7.2 Cliente → APIM

Escolher o perfil de identidade pelo consumidor e pela jornada, compondo os requisitos de §7.7:

- **OAuth 2.0** governa a autorização e a emissão de tokens de acesso; **OpenID Connect (OIDC)** acrescenta identidade para login. A API valida o token de acesso destinado a ela, não usa um ID token de login como autorização. Usar `validate-jwt` para o emissor aprovado ou `validate-azure-ad-token` para Microsoft Entra ID, com emissor/tenant, audience (destinatário), aplicações clientes permitidas, prazo e permissões verificados [16][84].
- **Máquina a máquina (M2M):** usar client credentials com autenticação de cliente apropriada ao risco. No Entra ID, conferir permissões de aplicação em `roles`, concedidas exclusivamente às aplicações consumidoras desse perfil. Um token app-only pode ser emitido sem roles; isso não o autoriza. Uma classificação pessoal ou sensível do dado não transforma esse fluxo em Authorization Code [93].
- **Acesso delegado de usuário:** usar Authorization Code; clientes públicos, como aplicações de página única e mobile, devem usar **PKCE** (Proof Key for Code Exchange) para proteger a troca do código [21]. Conferir os escopos delegados. O gateway não realiza sozinho login, consentimento ou a troca do código.
- **TLS mútuo (mTLS)** e tokens vinculados ao certificado aplicam-se quando exigidos pelo perfil setorial, contrato ou decisão de risco. Declarar onde TLS termina e como o certificado do consumidor chega ao ponto de verificação. FAPI não decorre automaticamente de LGPD ou PCI DSS [79][64].
- **Subscription keys:** complemento de identificação de consumo/produto e quotas, nunca autenticação suficiente. APIs deliberadamente anônimas não dependem dessas chaves para limitar tráfego; ver A.6.
- **Versão do token Entra:** nos exemplos A.1–A.3 e A.5, fixar access token v2.0, audience igual ao client ID da API e aplicação cliente em `azp`. Em v1.0, revisar audience do recurso e `appid`; não trocar apenas o nome da claim. `scp` é uma lista de escopos delegados separada por espaços. `roles` representa papéis/permissões conforme a emissão: a presença da claim, isoladamente, não prova um fluxo M2M. Usar o objeto emitido por `output-token-variable-name` após validação, sem reanalisar o header para auditoria [16][85][87][93].
- **RFC 9700**, Best Current Practice for OAuth 2.0 Security, orienta os fluxos [21]. Aqui, Best Current Practice é a prática técnica da IETF; não significa plano de continuidade de negócios.

### 7.3 APIM → backend

APIM faz **autorização de escopo amplo** no perímetro: valida identidade, emissor, audience, escopos/roles/claims, produto/subscription, quota e políticas de acesso por API/operação. Isso reduz a superfície de exposição e bloqueia chamadas inválidas segundo esses controles, mas **não substitui a autorização de negócio no backend**.

- **BOLA (Broken Object Level Authorization / OWASP API1)**: checar se o sujeito pode acessar aquele objeto específico (`accountId`, `customerId`, `consentId`) é responsabilidade do backend/sistema de domínio.
- **BFLA (Broken Function Level Authorization / OWASP API5)**: APIM pode bloquear funções por scope/role, mas a decisão final de função/ação do domínio deve estar no backend.
- **Autorização por campo/propriedade (OWASP API3)**: mascaramento, allowlist de campos e validação de propriedade sensível pertencem ao backend; APIM pode complementar com validação de schema e remoção de headers, mas não decidir regras de negócio específicas.

Controles de comunicação APIM → backend:

- **mTLS** quando exigido pelo perfil, contrato ou avaliação de risco da comunicação com o backend, com cadeia, renovação e revogação verificadas nesse segmento.
- **Managed Identity** quando o serviço de destino aceitar autenticação Microsoft Entra e tiver audience e autorização configuradas para a identidade do APIM; presença em Azure, isoladamente, não configura essa autorização.
- **Private Link ou integração com rede virtual (VNet)** quando suportados pelo SKU/runtime e pelo destino. Comprovar DNS, rotas, regras de rede e bloqueio do acesso público de bypass; não inferir o caminho efetivo apenas pelo nome do recurso.
- **Allowlist por IP** como controle complementar. Identificar o endereço observado no salto APIM–backend; uma faixa de rede não substitui identidade nem autorização de objeto.

### 7.4 Subscription keys

- Úteis para rastreio, cobrança e quotas; **não substituem autenticação**.
- Rotação automatizável via pipeline.
- Não tratar aplicação pública de navegador/mobile como cofre de segredos. Quando uma chave for necessária, mantê-la em componente confidencial do consumidor, protegida em vault; não a registrar em logs ou incluí-la em código distribuído.

### 7.5 Policies recomendadas

A ficha de intake determina as policies aplicáveis por API/operação e o gateway em que serão executadas. O gate verifica essa composição, não instala a mesma autenticação em toda API:

- `validate-jwt` ou `validate-azure-ad-token` para fluxos com token JWT: fixar emissor/tenant, audience, clientes, permissões e versão; separar os valores de escopos quando a claim for multivalorada. Consumir o token validado pela variável de saída [16][84][87].
- `rate-limit-by-key` e `quota-by-key` onde suportadas, com chave que explicite API, finalidade do contador e consumidor. Rate limits são aproximados e independentes por gateway; quotas também têm ressalva de comportamento em reinícios. Não são prova de contabilização global exata. `rate-limit` e `quota` por subscription não limitam consumo realmente anônimo [88][89].
- `ip-filter` quando houver uma faixa de origem aprovada e identificada no salto observado pelo gateway. Atrás de proxy, não presumir que o endereço ou um header recebido seja a identidade original do consumidor.
- Remover headers de identificação técnica desnecessários, como `Server` e `X-Powered-By`; esses headers não são uma categoria equivalente a tokens ou segredos. Não registrar Authorization, cookies, subscription keys, segredos ou corpos com dados pessoais em diagnostics, traces ou eventos.
- `cors` para consumidores de navegador, com origins explícitos em produção e credenciais somente quando necessárias. `*` fica restrito ao sandbox pela política corporativa deste guia; CORS não autentica a API [91].
- `validate-content` quando houver corpo e schema contratados: cadastrar/verificar o schema por operação ou seu `schema-id`, tipos de conteúdo, tamanho e tratamento de falhas. O apêndice B não instala sozinho validação; contratos de outros protocolos exigem verificações próprias [90][55].
- Correlacionar requisições com identificador gerado ou validado; não confiar em um valor arbitrário do cliente como prova de identidade. Eventos de auditoria devem ter campos minimizados, finalidade e retenção aprovadas (§10.7).
- Reutilizar Policy Fragments e named values protegidos quando apropriado, verificando a ordem e as policies herdadas por `<base />`. Parse XML, schema APIM e testes de runtime são verificações distintas; os exemplos do apêndice A não constituem evidência de execução [85].

### 7.6 Modelo de ameaças por perfil composto e OWASP API Top 10

Produzir o modelo para a composição de exposição, dados, setor, consumidor e criticidade (§7.7); revisar anualmente ou por mudança material. STRIDE reúne as categorias Spoofing (falsificação de identidade), Tampering (alteração), Repudiation (repúdio), Information Disclosure (divulgação), Denial of Service (negação de serviço) e Elevation of Privilege (elevação de privilégio). A tabela relaciona cenários aos riscos do projeto OWASP de segurança de aplicações; não substitui o teste da autorização no backend.

Na tabela, TLS (Transport Layer Security) protege o transporte; HMAC é autenticação de mensagem baseada em hash com chave; JWS (JSON Web Signature) representa assinatura de conteúdo JSON; WAF é firewall de aplicações web. Validação de schema verifica o corpo contratado, mas não autoriza o acesso ao objeto. Cada mecanismo precisa de escopo e configuração próprios:

| STRIDE | Risco típico em APIM | Controles APIM | OWASP API Top 10 2023 [20] |
|---|---|---|---|
| Spoofing | Cliente forjar identidade | OAuth/OIDC; mTLS; PKCE; validação de issuer/audience | API2 Broken Authentication |
| Tampering | Payload manipulado em trânsito ou mass assignment | TLS obrigatório; payload validation; HMAC/JWS quando aplicável; schema deny-by-default | API3 Broken Object Property Level Authorization; API8 Security Misconfiguration |
| Repudiation | Falta de evidência correlacionável de chamada e mudança | Eventos minimizados; correlation ID; trilha de gestão; integridade/imutabilidade conforme §10.7. Sentinel correlaciona, não assina automaticamente os logs | API9 Improper Inventory Management (inventário e rastreabilidade; não equivalência de categorias) |
| Information Disclosure | Vazamento via erro/header/log/campo não autorizado | Erros padronizados; remoção de headers; políticas de log sem informação pessoal identificável (PII); validação de resposta quando aplicável | API3 Broken Object Property Level Authorization |
| Denial of Service | Sobrecarga, abuso e consulta GraphQL excessiva | Limites e quotas por chave pertinente onde suportados; proteção de borda conforme o caminho TLS aprovado; `validate-graphql-request` quando aplicável; limites e circuit breaker no backend | API4 Unrestricted Resource Consumption |
| Elevation of Privilege | Token validado incorretamente, claim aceita sem verificação, função administrativa exposta | Validação rígida (issuer, audience, scope, claims); deny-by-default; produtos separados por função | API5 Broken Function Level Authorization |
| (Adicional) | Acesso direto ao backend | Restrição de rede; allowlist IPs APIM no backend; mTLS APIM→backend | API8 Security Misconfiguration |
| (Adicional) | API fora do inventário ou sem controle conhecido | Inventário API Center [10] reconciliado com deployments e backends; Defender for APIs [25] somente para APIs REST onboarded e gateways suportados, sem presumir cobertura de workspaces/self-hosted ou descoberta universal | API9 Improper Inventory Management |
| (Adicional) | Consumo inseguro de API externa | Timeout, validação de schema, allowlist de hosts, retry com limite | API10 Unsafe Consumption of APIs |

> No OWASP API Top 10 2023, **API3 é "Broken Object Property Level Authorization"**. O rótulo usado no Top 10 2019 para exposição excessiva de dados foi consolidado no risco de autorização por propriedade/objeto em 2023. APIM aplica controles no perímetro; a autorização de objeto/campo permanece no backend (§7.3).

Saída: modelo de ameaças do produto ou cenário, com as dimensões de §7.7, fronteiras de confiança, caminho TLS, controles, riscos residuais e testes vinculados ao backlog. O arquivo pode usar o nome descritivo do produto; uma mudança de marcador regulatório não exige um documento artificialmente separado se o cenário já estiver coberto.

### 7.7 Composição de perfis de API e controles aplicáveis

A ficha registra cinco dimensões independentes: **exposição** (interna/externa), **dados** (categorias legais e classificação corporativa), **setor/regime**, **consumidor/jornada** (M2M, usuário delegado ou anônimo autorizado) e **criticidade**. Os marcadores se acumulam; não são classes mutuamente exclusivas. A criticidade acrescenta requisitos de disponibilidade, operação e autorização, sem alterar automaticamente a categoria legal do dado ou o fluxo OAuth.

Aplicar os requisitos de todas as dimensões pertinentes. O fluxo depende do consumidor; a rede, da exposição e do caminho aprovado; as permissões, da operação; retenção, dos conjuntos e finalidades. Uma API interna M2M com dado pessoal continua interna e M2M, acrescida dos controles de privacidade. Dados pessoais comuns, sensíveis legais, financeiros e restritos corporativos não têm o mesmo significado. LGPD e PCI DSS não tornam uma API participante de Open Finance nem impõem FAPI por si [66][79][81].

Os perfis abaixo são exemplos de composição. As colunas de mTLS, rede e retenção não constituem mínimos legais universais. TLS mútuo autentica ambas as pontas do segmento; schema descreve o corpo contratado; telemetria é evidência técnica minimizada, distinta do registro de negócio.

| Perfil composto (exemplo) | Consumidor e permissão | mTLS | Exposição/rede | Corpo contratado | Telemetria | Retenção | Aprovação adicional |
|---|---|---|---|---|---|---|---|
| **Interna M2M**, regime setorial identificado separadamente | Client credentials, aplicação permitida e application role; A.1 | Por contrato/perfil/risco | Caminho interno aprovado; conectividade privada conforme SKU/runtime | Schema por operação quando houver corpo | Correlação e metadados operacionais necessários | Política por conjunto/finalidade (§10.7) | Owner do domínio e segurança conforme risco |
| **Externa B2B M2M** | Client credentials e permissão de aplicação; subscription key apenas complementar; A.2 | Quando exigido pelo perfil ou contrato | Borda aprovada e origem restrita; IP é o observado no salto, não derivado da cadeia de certificados | Schema e limites contratados | Campos aprovados de chamada e consumidor | Obrigação contratual/normativa aplicável e política aprovada | Segurança e jurídico; compliance se houver regime |
| **Dados pessoais comuns ou sensíveis legais**, interna ou externa | M2M com `roles` ou usuário delegado com `scp`, conforme jornada; A.3 exemplifica somente a segunda | Decisão de risco ou requisito aplicável, não consequência automática da LGPD | Segue a exposição aprovada, sem forçar ingress externo | Schema quando aplicável; minimizar campos | Sem tokens, segredos, headers de credencial ou corpo pessoal; pseudônimos ainda exigem análise LGPD | Sem piso universal de dois anos; mapear art. 7º/11, finalidade e conjuntos | Controlador/owner, privacidade/encarregado, jurídico e segurança |
| **Participante de Open Finance**, jornada e dados definidos | Perfil brasileiro vigente; emissor, destinatário, cliente, permissões e consentimento pertinentes | Vínculo ao certificado original quando o perfil exigir RFC 8705; A.4 é exemplo direto | Ponto TLS e controles de borda aprovados. Proxy L7 exige outro mecanismo de confiança, comprovado | Contrato da jornada e assinaturas exigidas pelo perfil | Evidência de controle e negócio nos sistemas adequados, não cópia integral de payload | Conjuntos do art. 49 da Conjunta nº 1/2020 e demais obrigações (§10.7), não todo gateway log por cinco anos | Compliance do regime, segurança e privacidade |
| **Fachada administrativa de aplicação**, distinta de ARM | Cliente permitido e application role no exemplo M2M A.5; usuário delegado exige perfil próprio | Conforme caminho e risco | Interna por padrão corporativo; gestão ARM protegida separadamente (§7.1) | Schema da operação | Ação, decisão e mudança correlacionadas, sem credenciais | Política de governança, riscos e conformidade (GRC) por conjunto, não sete anos universais | Plataforma, segurança e auditoria conforme privilégio |
| **Pública anônima autorizada**, somente leitura | Sem token/subscription key por decisão explícita; A.6 | Não exigido no exemplo | Borda aprovada; limite por IP observado é aproximação, não identidade do consumidor | Sem dados de negócio pessoais ou segredos | Diagnóstico mínimo; avaliar também IP e metadados | Política operacional aprovada, sem piso legal genérico de noventa dias | Plataforma/owner e segurança |

> A composição é **vinculante** para o gate (§9.3): ficha, perfil de identidade, policies, suporte do gateway, retenção aprovada e testes precisam concordar. Obrigações legais e contratuais aplicáveis não são dispensadas por aprovação interna. Padrões corporativos adicionais e suas exceções devem ter responsável, justificativa, prazo e controle compensatório; não substituir essa decisão por um prazo ilustrativo desta tabela.

### 7.8 Evidências de privacidade e requisitos BCB/CMN aplicáveis

APIM contribui para os controles, mas não substitui a responsabilidade do controlador, os processos do domínio nem a análise de enquadramento. A ficha liga cada obrigação a responsável, procedimento e evidência verificável:

| Requisito e alcance | Controle ou processo | Evidência necessária |
|---|---|---|
| LGPD arts. 5º, 7º e 11: categoria, finalidade e base | Intake por operação de tratamento e perfil composto (§2.3.4/§7.7) | Categoria legal separada da corporativa, artigo/inciso, finalidade e decisão do controlador; não apenas campo preenchido [66] |
| LGPD art. 18: atendimento de direitos | Procedimento no sistema-fonte; fachada APIM se for o canal escolhido | Inventário, encaminhamento, decisão e teste de atendimento; objetivo de serviço da integração definido pela empresa, não um endpoint imposto pela lei [66] |
| LGPD arts. 46–47: medidas adequadas | Identidade, transporte, autorização no backend e coleta mínima | Configuração efetiva e testes positivos/negativos; dados e destinos de diagnostics/integradores inventariados; plano habilitado não prova cobertura |
| LGPD art. 38: RIPD | Determinação legal/da ANPD, avaliação de alto risco e política interna separadas | Relatório vinculado à operação/projeto, gatilho e responsável; relatório compartilhado apenas quando cobrir os tratamentos similares [66][76] |
| LGPD art. 48 e ANPD nº 15: incidente | Runbook obrigatório quando aplicável (§2.3.1/§10.6), com controlador, ANPD e titulares | Relógio do conhecimento, canais e representação, comunicação inicial em três dias úteis e complementação em vinte, com as ressalvas aplicáveis; declaração da comunicação aos titulares e registro inclusive de incidentes não comunicados (§10.7) [27] |
| LGPD art. 33 e ANPD nº 19: transferência | Mapear exportador/importador, dados, destinos e acessos internacionais | Hipótese de tratamento, mecanismo válido, instrumento/cláusulas pertinentes e restrições setoriais/contratuais; metadado sem dado sensível não é dispensa [66][67] |
| Conjunta nº 1/2020 e BCB nº 32/2020: participante de Open Finance | Perfil e jornada vigentes, contratos e controles dos sistemas envolvidos | Versões aprovadas dos manuais, certificados e emissor, testes do caminho TLS completo e autorização/consentimento no domínio; conjuntos do art. 49 identificados [28][78] |
| CMN nº 4.893/2021 ou BCB nº 85/2021: entidade abrangida e serviço relevante | Due diligence, contratos, comunicação de contratação, condições de exterior, continuidade, testes e resposta | Enquadramento e relevância documentados; requisitos dos arts. 12–17, comunicação do art. 15 e autorização do art. 16 quando cabível; registros/testes dos arts. 22-A/23. Não usar BCB nº 198/2022 como norma cloud [28b][28c] |

Registrar links reais para evidências na ficha, como aprovação, contrato, configuração, relatório de teste e runbook ensaiado. Alertas e correlação em SIEM (sistema de gestão de eventos de segurança), como Microsoft Sentinel, apoiam a operação; não provam sozinhos comunicação, integridade criptográfica ou conformidade.

---

## 8. Plataforma, landing zone, rede e guardrails

### 8.1 Aderência à Azure Landing Zone

- Subscrições padronizadas (Connectivity, Identity, Platform, Workloads).
- **APIM não fica na subscrição Connectivity por padrão.** Connectivity hospeda o hub de rede, firewall/Azure Virtual WAN, resolução DNS (Domain Name System), ExpressRoute e rede privada virtual (VPN, Virtual Private Network). APIM é uma carga ou plataforma de aplicação, provisionada pelo processo corporativo de criação governada de subscrições (subscription vending), em Platform-APIM ou na subscrição do domínio, com policies, tags e RBAC herdados [33].
- APIM corporativo nas subscrições de Platform; instâncias dedicadas nas Workloads do domínio. Exceção para colocar APIM no hub/Connectivity exige ADR aprovada por plataforma, segurança e rede, com justificativa de risco/blast radius.
- Hub-spoke: hub com firewall central, ExpressRoute/VPN e DNS híbrido; APIM em spoke dedicado ou workload spoke conforme modelo de adoção.
- Naming/tagging conforme padrão corporativo (§8.2).

### 8.2 Nomenclatura, tags e responsabilidades

Exemplo de convenção:

- Convenção local `apim-{corp}-{env}-{regiao}-{seq}`; no exemplo `apim-acme-prd-bs-01`, `prd` significa produção, `bs` Brazil South e `01` o número sequencial. Os códigos pertencem ao padrão corporativo, não são nomenclatura universal Azure.
- Tags de recursos Azure, quando o tipo de recurso as suportar: `owner`, `costcenter`, `costAllocation`, `domain`, `apiProductOwner`, `data-classification`, `regulatory-class`, `slaTier`, `slo`, `exposure` (`internal|partner|public|admin`), `retentionClass`, `oncall`, `criticality`, `mpasMapId`, `env`. Classificação corporativa, categoria legal de dados, regimes, exposição e criticidade são dimensões diferentes, compostas no intake; uma tag não comprova conformidade.
- Em APIs/produtos, registrar **metadados** no API Center, contrato e intake, sem presumir suporte a tags ARM da entidade: `owner`, `apiProductOwner`, `supportContact`, `costAllocation`, `slaTier`, `exposure`, `consumerType`, `criticality`, `dataClassification`, `personalDataCategory`, `regulatoryClass`, `retentionPolicyId`, `dataResidencyDecisionId`, `mpasMapId`. Manter relação com os recursos e pools de custo (§6.5).
- Responsabilidade pelo recurso APIM corporativo: equipe de plataforma; por API/produto: domínio produtor. O responsável pelo produto aprova quota, adesão de consumidores, deprecação e custo alocado.

### 8.3 RBAC

- **API Management Service Contributor** somente para identidades de automação/plataforma cujo escopo de operação o justifique; preferir o menor conjunto de ações e recursos necessário.
- **API Management Service Reader** para leitura do serviço, no escopo autorizado.
- **API Management Service Operator** administra o serviço, mas não escreve entidades de API/policy; não é a role de publicação de APIs [59].
- Para publicação, preferir pipeline com identidade própria e role de workspace ou custom role que permita as ações de API/policy necessárias, no escopo correto. Não ampliar automaticamente a Contributor para resolver uma falta de permissão.
- Papéis específicos de edição de policy, leitura de logs ou publicação de portal são custom roles com ações, escopo e testes documentados, não nomes presumidos de roles built-in. Plano ARM, portal e APIs de dados têm controles distintos [59].

### 8.4 Azure Policy e guardrails

Guardrails corporativos mínimos, implementados por definições concretas e gates com evidência de suporte:

- Bloquear criação fora dos tiers e regiões aprovados para a configuração.
- Exigir tags de recurso e metadados aplicáveis, sem tratar rótulos como prova de controle.
- Exigir telemetria e diagnostic settings suportados pelo runtime; para recursos sem a categoria/sink pretendidos, exigir alternativa operável e testada (§10.1).
- Exigir **matriz de cobertura de detecção por API/gateway**. Para APIs REST elegíveis ao Defender, comprovar plano e onboarding. Workspaces e self-hosted gateway não são cobertos: exigir logging, detecção de abuso/anomalia, integração SOC, responsáveis e testes alternativos, com aceite de risco/exceção aplicável. Não dispensar controle obrigatório externo por aceite interno [25][58].
- Bloquear protocolos TLS antigos e exigir a configuração de proteção de dados aplicável ao recurso, usando definições/aliases existentes e testes, sem presumir uma opção de "desligar criptografia" em todo serviço.
- Exigir rede de backend aprovada pela composição de exposição, dados e regimes. Private Link é uma opção quando suportada; injection/integração de saída e conectividade híbrida podem ser necessárias. Incompatibilidade não autoriza backend público sem decisão e controles adequados [15].

Matriz de responsabilidade dos guardrails:

| Camada | O que controla bem | O que NÃO controla | Exemplo de gate |
|---|---|---|---|
| **Azure Policy** | Propriedades e recursos com definição/alias aplicável: tier/região, tags, TLS, diagnostics, rede e identidade | Não comprova onboarding/cobertura Defender, semântica de contrato, quota efetiva, autorização de negócio ou execução de expressões | Deny para configuração proibida; DeployIfNotExists para diagnostics suportados; verificar conformidade e teste da definição |
| **Pipeline/IaC** | Contrato apropriado ao protocolo, recursos, policies, loggers, RBAC, budgets, destinos e matriz de cobertura | Parse/lint e what-if não executam policies nem provam o tráfego real; mudanças fora do pipeline exigem detecção de drift | Lint/diff do contrato; plan/what-if da configuração; testes de tráfego e casos negativos vinculados ao artefato |
| **Runtime APIM/WAF** | Tokens/certificados no ponto de terminação definido, limites de tráfego, schema quando suportado, proteção de entrada e sinais | Não substitui responsabilidades, base legal ou autorização de objeto/campo no backend; cobertura depende do gateway/policy | Rejeitar token/certificado inválido, testar origem restrita e binding mTLS, limites e consultas GraphQL no caminho completo |

Implantação de Azure Policy:

1. **Audit** por 2–4 semanas para medir impacto e falsos positivos.
2. **Modify** apenas para tags/propriedades com alias modificável e valor seguro, com identidade/permissões de remediação adequadas; não provisiona genericamente diagnostic settings [60].
3. **DeployIfNotExists** para recursos auxiliares previstos na definição, como diagnostics ou vínculo DNS aplicáveis. A definição que habilita um plano Defender não comprova onboarding nem cobertura das APIs [25][58].
4. **Deny** para configuração proibida comprovada, como região/tier não aprovado, exposição pública não permitida pela composição de requisitos ou TLS antigo. Rede pública do gateway, plano de gestão e backend são superfícies distintas.

Toda mudança Audit→Deny/Modify/DeployIfNotExists exige changelog, owner, janela de comunicação e exceção formal (§11.5).

### 8.5 Rede, DNS e exposição

- **Rede privada conforme o requisito e o tier**: Premium clássico oferece injection interna/externa; Premium v2 oferece injection do gateway em região elegível. Standard v2 pode combinar private endpoint de entrada e integração de saída à VNet. Inbound private endpoint não equivale a injection, e nenhuma dessas modalidades seleciona o modelo de adoção sozinha [15].
- **WAF antes da entrada externa**, por política corporativa do guia, com bloqueio de acesso direto à origem.
  - **Front Door com WAF**: entrada global, cache/distribuição de conteúdo e roteamento por saúde das origens; validar efeitos de localização/processamento de borda [69].
  - **Application Gateway WAF v2**: entrada regional, roteamento de camada HTTP e acesso privado ao APIM quando compatível com a rede escolhida [15].
  - **Combinação de ambos**: somente com necessidade demonstrada de controles globais e regionais; registrar cada conexão TLS, probes, cabeçalhos e dependências. TLS recriptografado entre proxies continua sendo conexões distintas.
- **DNS**: zonas/resolução privadas para endpoints internos e domínio corporativo externo para a borda. Testar acesso a dependências, resolução híbrida e rotas do tier.
- **Certificados** no Key Vault quando a integração suportar; renovação, rotação e atualização de bindings precisam de fluxo testado, inclusive nos segmentos proxy–APIM–backend.

**mTLS com proxy L7:** a ADR fixa onde o certificado consumidor é autenticado. Application Gateway termina TLS e abre outra conexão com APIM; o certificado original não é encaminhado automaticamente, e `context.Request.Certificate` observa o certificado da conexão que chega ao APIM, não uma identidade herdada por suposição [19]. Se a borda autenticar o consumidor, transportar apenas metadados verificados em canal/origem restritos, remover/substituir cabeçalhos fornecidos pelo cliente e verificar o vínculo do token com o certificado original quando o perfil exigir. Rejeitar bypass da origem, token vinculado a outro certificado e metadado ausente/forjado. Testar a cadeia completa; não pressupor suporte mTLS de todo produto de borda nem tratar WAF como passthrough TLS [64].

Matriz de exposição e controles de rede complementares. As linhas não são classes exclusivas: um serviço interno máquina a máquina pode tratar dados pessoais ou pertencer a setor regulado; compor os requisitos sem convertê-lo automaticamente em API externa ou fluxo de usuário.

**Legenda:** NSG = Network Security Group, filtro de tráfego; UDR = User-Defined Route, rota definida pelo cliente; Azure Virtual WAN = conectividade/roteamento de rede; L7 = camada de aplicação; PIM = Privileged Identity Management; JIT = acesso temporário just-in-time. DNS resolve nomes; não autentica o consumidor.

| Exposição / requisito adicional | Entrada e identidade | DNS | Controles de tráfego e rota | Acesso público do gateway |
|---|---|---|---|---|
| **Interna** | Injection interna ou private endpoint elegível; identidade conforme consumidor | Zonas privadas adequadas ao endpoint e domínio interno | Origem corporativa autorizada, rotas/DNS/dependências testados; egress inspecionado quando requerido | Desabilitar quando a configuração suportar; se não atender à ausência obrigatória de internet, rejeitar a configuração, não presumir equivalência de allowlist |
| **Externa entre empresas (B2B)** | WAF regional/global, autenticação e eventual mTLS conforme o contrato; origem protegida | Domínio público da borda; resolução privada dos backends quando aplicável | Allowlist quando pertinente, backend em rede aprovada, ausência de bypass do WAF | Público apenas pelo caminho de borda autorizado; proteger o endpoint de origem |
| **Adicional de perfil que exija mTLS/FAPI** | Aplicar ao cenário de exposição correspondente; escolher terminação mTLS e verificar vínculo do token/certificado original | Certificados e nomes de acordo com o perfil/contrato; não presumir exigência genérica de validação estendida (EV) ou organizacional (OV) do certificado | Emissor, cadeia e revogação testados; metadados autenticados quando houver proxy, regras de origem e tráfego restritas | Não se satisfaz apenas com "WAF + mTLS no APIM"; provar o caminho completo descrito acima |
| **Fachada administrativa própria** | Acesso restrito e autenticação/autorização da API; gestão Azure/ARM protegida separadamente por RBAC/PIM | Resolução do endpoint administrativo aprovado | Administração privilegiada e acesso temporário controlados; não confundir policies de API com proteção do ARM | Configuração do gateway não bloqueia automaticamente a API ARM de gestão |
| **Pública sem autenticação** | WAF, limites apropriados ao acesso anônimo e cache somente para respostas elegíveis | Domínio público da borda; origem protegida | Origem restrita ao perímetro, detecção de abuso e coleta conforme a finalidade | Acesso apenas pelo caminho aprovado; quota por subscription key não controla chamadas realmente sem chave |

DNS híbrido obrigatório:
- Resolver do hub encaminha zonas privadas Azure para spokes e, quando necessário, para DNS on-prem via Azure DNS Private Resolver.
- O datacenter próprio deve resolver nomes de domínio totalmente qualificados (FQDNs, Fully Qualified Domain Names) dos endpoints privados APIM/backends pelo caminho DNS aprovado, sem desvio para a internet.
- Validar split-horizon (`api.empresa.com` externo vs interno) em runbook antes do go-live.

### 8.6 FinOps de APIM (dimensionamento, custo, showback/chargeback)

O custo de APIM inclui a licença do SKU e os serviços associados. Quatro componentes principais devem ser considerados:

1. **Oferta e execução APIM**: tier, unidades de capacidade e regiões, chamadas incluídas/excedentes quando o plano as cobrar, workspace gateways e self-hosted gateway conforme os meters contratados. Premium clássico multirregião soma capacidade por região; não confundir workspace lógico com gateway separado [1][3][5].
2. **Observabilidade**: volume ingerido, plano de tabela, retenção, consultas e destinos. Amostragem, transformações e armazenamento em camadas podem reduzir custo; a magnitude depende do perfil de coleta, preços efetivos e retenção. Comparar baseline e cenário calculado, incluindo completude da evidência, sem adotar percentual de economia universal. Retenção/amostragem seguem política aprovada por conjunto e finalidade, não um prazo único por rótulo regulatório [18][31].
3. **Borda e segurança**: Front Door, Application Gateway/WAF e Defender for APIs. Defender usa plano mensal por **subscrição Azure**, franquia de chamadas monitoradas e excedente de tráfego; não cobra simplesmente pela quantidade de APIs cadastradas. Estimar apenas cobertura/onboarding efetivos, sem incluir workspaces/self-hosted como APIs protegidas [25][58].
4. **Tráfego e transporte de evidência**: saída de dados, Private Link, Event Hubs, captura e consumo do SIEM (Security Information and Event Management, gestão e correlação de eventos de segurança). Verificar os meters e evitar contabilizar duas vezes encargos já incluídos no serviço de borda.

#### 8.6.1 Dimensionamento inicial

- Não use número fixo de TPS por unit como premissa de dimensionamento ou orçamento. Throughput publicado ou citado é informativo, não é SLA e não deve ser usado para planejamento de capacidade ou orçamento.
- Dimensione a partir do perfil real esperado: pico de requisições, concorrência, tamanho de payload, latência do backend, reutilização de conexões, TLS, número e custo das policies, cache, transformação, validação JWT/mTLS e logs.
- **Teste de carga representativo é obrigatório** antes de assumir compromissos com FinOps/produto. O teste deve refletir payloads, policies, backends, observabilidade e p95/p99 esperados em produção.
- Para operação e autoscale, use métricas sustentadas de capacidade/CPU/memória, latência, erros e saturação por período prolongado; não tome decisão com base em um pico breve.
- Multi-region, unidades adicionais, calls v2, workspace gateways/packs e self-hosted gateways aumentam custo conforme meter aplicável por SKU/região. Revalidar preços no Azure Pricing Calculator [30].

#### 8.6.2 Componentes e parâmetros do estimador de custo

O orçamento exige extrato verificável por `meterId`, tier/SKU, região e data efetiva ou proposta contratual equivalente. FinOps/compras registra moeda, impostos, câmbio, descontos, horas e volumes considerados antes da aprovação. A tabela abaixo reúne fatores de dimensionamento, **sem valores de cotação**. Preços históricos não devem ser reutilizados sem revalidação; existência de um meter de preço não comprova disponibilidade do recurso na região.

| Componente | Unidade/fator a verificar na oferta | Parâmetro necessário para orçamento |
|---|---|---|
| APIM Basic v2 | Capacidade/unidades e chamadas conforme plano | Região disponível, unidades, horas, franquia e excedente de chamadas |
| APIM Standard v2 | Capacidade primária/adicional e chamadas conforme plano | Unidades, regra de escala, horas, franquia e excedente |
| APIM Premium clássico | Unidades/horas por região | Capacidade de cada região e topologia de multirregião |
| Self-hosted gateway | Meter de gateway contratado + infraestrutura do cliente | Gateway/ambiente elegível, licenciamento, containers, operação e coleta própria |
| APIM Premium v2 | Capacidade primária/adicional e chamadas conforme oferta | Só estimar em região/capacidade confirmadas; não está listado em Brazil South [3] |
| Workspace gateway / pacote de oferta | Meter específico do runtime/oferta, não do workspace lógico | Região primária/subscrição elegíveis, quantidade, escala e limites; não listado em Brazil South [3][5] |
| Log Analytics: ingestão | Volume ingerido e plano de tabela | GB por conjunto, transformação/amostragem aprovadas e preço efetivo |
| Log Analytics: retenção/arquivo | Volume e prazo além da inclusão do plano | Política por tabela/finalidade, volume retido e consultas necessárias |
| Event Hubs para auditoria/SIEM | Capacidade, captura, retenção e transferência conforme oferta | Namespace, pico/volume, consumidores e janela de recuperação |
| Microsoft Sentinel | Plano e ingestão/análise/retenção pertinentes | Volume elegível, compromisso contratado e retenção aprovada |
| Defender for APIs | Plano mensal por subscrição, franquia e chamadas monitoradas excedentes [25] | APIs/gateways cobertos, tráfego monitorado agregado da subscrição e plano; workspaces/self-hosted excluídos [58] |
| Application Gateway WAF v2 | Gateway/capacidade e tráfego conforme oferta | Região, capacidade testada, escala, regras e tráfego |
| Front Door | Plano, solicitações e transferência; recursos incluídos variam por oferta | Origem/destino, cache, tier, WAF e conectividade; não duplicar encargos incluídos |
| Private Link | Endpoints/horas e dados processados quando cobrados | Número de endpoints, região, horas e GB |
| Saída de dados | Meter e volume efetivamente faturados | Origem, destino, GB e eventuais inclusões/isenções contratuais |

O Azure Pricing Calculator [30] é uma ferramenta de estimativa, não prova isolada de oferta regional ou preço contratual. Campos sem evidência permanecem pendentes no orçamento; não preencher com os números históricos nem contratar um SKU pela presença de seu meter.

#### 8.6.3 Tagging e showback/chargeback

- Relacionar recursos e metadados de API/produto (`costcenter`, `costAllocation`, `domain`, `apiProductOwner`, `slaTier`, `exposure`, política de retenção e intake) com centros de custo e pools de faturamento; não presumir tags ARM em toda entidade APIM (§8.2).
- Separar gateway logs, atividade administrativa e outras categorias. Atribuição de GB por API/domínio depende das dimensões efetivamente coletadas e da regra de rateio, não apenas de colocar todos os logs em um workspace.
- **Showback** informa consumo e custo alocado sem cobrança interna; **chargeback** aplica cobrança interna aprovada. Fechar ambos por período, pool e responsável, reconciliando com a fatura.
- Para Defender, o pool é o custo real do plano/excedente na subscrição Azure, com franquia e chamadas monitoradas. Ratear por tráfego coberto é uma convenção interna; quantidade de APIs, se adotada como outro critério, não é o meter do fornecedor e precisa de aceite FinOps e reconciliação [25][58].

Rateio mensal por **pool identificado** e período de faturamento. Para cada pool, registrar componentes da fatura, instância/região/subscrição, unidade de alocação (API ou produto), conjunto de beneficiários e driver. Numerador, denominador e custo devem ter o mesmo pool/período; um request ou item de fatura não pode ser integralmente alocado em dois produtos/pools.

```text
custo_i_no_periodo = custo_direto_i
                  + soma_pools(fator_i_pool * custo_compartilhado_pool)
fator_i_pool = driver_i_pool / soma_drivers_dos_beneficiarios_pool
              somente quando soma_drivers_dos_beneficiarios_pool > 0
```

Drivers possíveis: chamadas para o pool de calls APIM; GB e retenção atribuíveis para pools de observabilidade; chamadas **monitoradas e cobertas** para o pool Defender da subscrição; peso de capacidade/criticidade aprovado para capacidade/WAF compartilhados. Custos exclusivos de instância ou gateway dedicado entram como custo direto; não voltam ao pool compartilhado.

Quando o denominador for zero, manter o custo identificado como não alocado ou usar a regra de reserva/capacidade previamente aprovada por FinOps. Não dividir por zero nem redistribuir silenciosamente a outro pool. Pesos internos iniciais Bronze=1, Silver=2, Gold=4 e Crítico=6 são convenção a justificar; rótulo regulatório não mede custo nem determina peso.

Antes do chargeback, comprovar por pool/período: **soma dos custos diretos, alocados e não alocados = custo faturado do escopo**, incluindo créditos/descontos conforme a política. FinOps e responsáveis aprovam drivers, exceções e tratamento de resíduos. O rateio não reproduz necessariamente o meter do fornecedor [25].

#### 8.6.4 Alarmes orçamentários

- Azure Budgets por subscrição APIM + alertas em 50/80/100 %.
- Alertas por componente e pool: ingestão/retenção, borda, Event Hubs/Sentinel e Defender. Para Defender, acompanhar plano, franquia, chamadas monitoradas agregadas da subscrição e tendência de excedente, não apenas quantidade de APIs. Limiar e responsável são aprovados por FinOps [25].
- Revisão trimestral em fórum FinOps.

#### 8.6.5 Otimizações comuns

- **Cache elegível por gateway e finalidade** pode reduzir chamadas ao backend; testar ganho, atualização e isolamento de consumidor. Consumption não tem cache interno, mas admite cache externo. Cache semântico não equivale a cache HTTP e exige backend compatível com RediSearch e policy suportada; proteger contra reutilização indevida entre usuários [1][12][61].
- **Compressão** pode reduzir bytes de saída para conteúdo apropriado; medir custo computacional e compatibilidade.
- **Amostragem/transformação de logs** somente para conjuntos cuja política aprovada permita perda ou redução de detalhe; separar evidência obrigatória de diagnóstico. Não decidir apenas pelo rótulo "não regulada" [18][31].
- **Retenção em camadas e plano de tabela** conforme finalidade, prazo, consulta, recuperação e descarte aprovados; sem pisos universais de logs neste estimador.
- **Compromissos/reservas** apenas nos serviços/ofertas que os disponibilizem e após validação do consumo e contrato; não presumir reserva genérica de APIM.

#### 8.6.6 Estratégia de ambientes não-produtivos

- Desenvolvimento/sandbox pode usar tier e escala menores; homologação (UAT, User Acceptance Testing) replica os controles e o caminho de rede/identidade que serão validados, ainda que com menor capacidade. Se o ambiente for exposto, a proteção de entrada continua exigida pela política aplicável.
- Não presumir que APIM gerenciado possa ser pausado para interromper cobrança. Reduzir unidades somente dentro dos limites suportados, após avaliar testes, disponibilidade e preservação da configuração; alterar gateways e retenção não é um mecanismo universal de desligamento.
- Usar dados sintéticos ou mascarados em desenvolvimento/UAT. Eventual tratamento de dado pessoal real exige hipótese legal, finalidade, minimização, controles e aprovação competentes; aceite do DPO isoladamente não cria autorização legal [66].
- Retenção e destino de logs não produtivos seguem a política por conjunto/finalidade, inclusive sinais de segurança que devam chegar ao SOC. Prazo curto e plano de menor custo são candidatos a testar, não dispensa automática de evidência.
- Testes de carga usam janela controlada, budget temporário e limpeza automática de recursos.

---

## 9. Pipeline de publicação com infraestrutura como código e APIOps

APIOps aplica práticas de versionamento e automação à configuração de APIs. Infraestrutura como código (IaC) provisiona o serviço e suas dependências; as duas camadas podem compartilhar a entrega, mas precisam de responsáveis e evidências próprios [26]. Neste capítulo, PR é a solicitação de revisão de código (pull request), CI é integração contínua, DEV é desenvolvimento, UAT é homologação/aceite e PROD é produção. Merge não autoriza publicação em PROD sem os gates de promoção e aprovação.

### 9.1 Princípios

- **Infraestrutura como código (IaC)** obrigatória. Bicep ou Terraform.
- **Configuração como código**: APIs, operações, schemas, conjuntos de versões (version sets), produtos, grupos/subscriptions, fragments e policies XML, backends, referências a certificados/Key Vault, loggers e diagnostics. Infraestrutura, workspaces/gateways, rede, controle de acesso por papel (RBAC), alertas e budgets ficam no IaC. Conferir a cobertura do publisher escolhido; dois gestores não devem escrever o mesmo recurso [26][95].
- **Named values sem segredos no repositório**: valores sensíveis ficam no Key Vault; o IaC referencia Key Vault/managed identity. Segredo literal em YAML/Bicep/policy/OpenAPI bloqueia o PR.
- **Policies em XML versionado**, com fragmentos reutilizáveis e `<base/>` quando se deve herdar o controle do escopo superior. Parse XML verifica sintaxe; a herança efetiva e expressões exigem testes separados.
- **Contrato versionado por protocolo**: OpenAPI para HTTP/REST, schema e operações para GraphQL, mensagens/handshake para WebSocket e descritores para gRPC quando suportado (§9.3). A descrição de segurança no contrato não instala autenticação no gateway.
- **Build determinístico e entrada imutável**: compilar uma vez e selar um pacote com commit aprovado, template ARM compilado, configuração APIM, parâmetros/overrides de todos os ambientes, regras, scripts, fixtures e manifesto SHA-256. Referências a segredos, não seus valores, integram o pacote.
- **Promoção do mesmo pacote**: DEV → UAT → PROD, verificando o mesmo hash bruto em cada etapa. Apenas selecionar parâmetros/overrides já selados; recompilar, baixar componentes sem versão fixa ou editar parâmetros exige novo pacote e revalidação. Hash do pacote e digest de transporte do GitHub são identificadores diferentes [107].
- **GitOps e desenvolvimento em trunk**: PR → revisão → merge → automação de promoção e gates; evitar branches longas. Feature flags não dispensam validação de contratos ou autorização de exposição.
- **Controle de custos em não produção**: DEV/UAT com SKU, escala e retenção reduzidos, dados sintéticos, remoção de recursos ou redução de escala quando possível e orçamentos próprios (§8.6.6).

### 9.2 Fluxo alvo

```text
[PR] → CI: contratos por protocolo, lint XML/regras, secrets scan e fixtures
  ↓
[revisão] → responsáveis do contrato/controle; decisão única e consultas aplicáveis
  ↓
[merge] → build único; pacote, parâmetros e manifesto com hash
  ↓
[DEV] → implantar esse pacote; requests reais e testes positivos/negativos
  ↓
[UAT] → mesmo hash; contrato, desempenho, segurança e recuperação ensaiada
  ↓
[prévia PROD] → template ARM + parâmetros PROD; preview da configuração APIM
  ↓
[autorização] → provas DEV/UAT + plano PROD + segregação de funções (SoD) + exceções válidas + janela
  ↓
[PROD] → mesmo hash e parâmetros aprovados; smoke e observação contratada
  ↓
[conclusão] → saúde e evidência verificadas; falha bloqueia promoção/conclusão
```

Na autorização de produção crítica/regulada, autores da mudança e iniciadores humanos da implantação devem ser distintos do aprovador, com identidades e controles efetivos verificados conforme §11.6.1. `Prevent self-review` não comprova, isoladamente, toda essa segregação de funções (SoD).

O plano de release aprovado pelo API/Product owner, consultando plataforma e segurança conforme o risco, registra os parâmetros abaixo. Para PROD crítica, a autorização final pertence ao aprovador segregado de §11.6.1.

| Parâmetro | Unidade e significado | Registro e critério de uso |
|---|---|---|
| `janela_observacao_release_h` | Horas de observação após a publicação | Valor positivo por API/jornada no plano de release; não concluir antes do período e da cobertura de tráfego acordados |
| `janela_saude_recuperacao_min` | Minutos de saúde após rollback ou fix-forward | Valor positivo no plano de recuperação, referenciado pelo incidente; não reutilizar a duração de release como se fosse o mesmo parâmetro |
| `limite_latencia_p95_ms`, `limite_erros_5xx_pct` | Milissegundos no percentil 95 e porcentagem de respostas 5xx | Limites derivados do objetivo de nível de serviço (SLO), com consulta, denominador, período e exclusões aprovados |
| `minimo_amostras_observacao` | Quantidade de requisições válidas por jornada durante a janela | Cobertura acordada com os responsáveis; falta de tráfego, de série ou de evidência é inconclusiva e bloqueia o aceite |

Valor, unidade, fonte da métrica, responsável, data de aprovação e versão do registro são obrigatórios. Antes de alterar PROD, uma prova ausente, vencida ou divergente aborta a publicação. Depois de alteração parcial ou falha de smoke, interromper a promoção, acionar o plantão e executar a decisão de recuperação de §9.4.3; upload de logs não transforma falha em sucesso.

### 9.3 Validações obrigatórias em PR/pipeline

- **Contratos por protocolo**: a ficha de intake declara o artefato principal, a fachada adicional quando houver e a evidência de suporte do gateway/tier/região (§6.6).

| Protocolo | Artefato versionado | Validação e evidência antes de PROD |
|---|---|---|
| HTTP/REST (APIs de recursos sobre HTTP) | OpenAPI 3.0.3 e schemas das operações; 3.1 somente por gate de compatibilidade | Spectral com ruleset selado, diff do contrato anterior aprovado, importação sem perdas relevantes e requests reais por operação (§6.1) |
| OData | Modelo de dados em metadata/EDMX e contrato da fachada HTTP, se existente | Compatibilidade do modelo e requests positivos/negativos; não atribuir validação OData nativa ao simples passthrough |
| GraphQL | Schema na linguagem de definição SDL e operações/queries de teste; fachada HTTP adicional se necessária | Compatibilidade de schema, queries/mutations negativas, limites e comportamento do modo GraphQL/gateway selecionado |
| WebSocket | Contrato de negociação inicial (handshake), mensagens e encerramento da conexão | Upgrade/recusa, identidade quando exigida, mensagens inválidas e reconexão; evidência dos controles realmente suportados |
| gRPC, quando confirmado | Contrato Protobuf em `.proto`/descritores e métodos de teste | Compatibilidade e chamadas reais, erros e identidade no gateway com suporte confirmado; suporte pendente bloqueia compromisso de publicação |

- **Mudança incompatível**: comparação com o contrato efetivamente suportado, versão major e migração/comunicação (§6.4 e §11.4.2). XML/OpenAPI bem formado não prova compatibilidade funcional; definições de segurança ignoradas na importação não substituem policies.
- **Policies**: `xmllint` verifica XML estrito, incluindo escaping; linter corporativo verifica regras e herança `<base/>` aplicável. Testes de expressões em harness com mocks, se adotados, devem declarar a ferramenta e seus limites [85].
- **Infraestrutura e configuração**: build Bicep/validação de template e `az deployment ... what-if`, ou plano Terraform, são prévias de mudanças, não testes unitários de policies [14]. Para PROD, usar o template do pacote, os parâmetros efetivos de PROD e, se houver publisher APIOps, a prévia com o mesmo target/override. Revisar mudanças esperadas, deletes, recursos ignorados, ruído e expressões não resolvidas; sucesso do comando não significa “nenhuma mudança”.
- **Comportamento real em DEV/UAT**: runner HTTP/protocolo contra o gateway implantado, com fixtures sintéticas, backend controlado e asserts de status, headers, corpo permitido, efeitos de policy e telemetria. Registrar teste, entrada, resultado esperado/observado, endpoint/versão, horário, ambiente e hash do pacote. Cobertura vazia, dummy, skipped ou apenas parse bloqueia a promoção.
- **Casos negativos conforme o controle aplicável**: credencial ausente/expirada; issuer, audience, scope ou identidade indevidos; certificado ausente, inválido ou vínculo com token incorreto quando exigido; payload inválido; limite/quota com retorno esperado; origem/preflight CORS recusado; tentativa de bypass e falha de backend. Um controle não aplicável precisa de justificativa aprovada, não de teste fictício.
- **SKU, gateway e região**: conferir capacidade, rede, APIM/workspace gateway, firewall de aplicação web (WAF), Key Vault e todos os destinos de dados/telemetria aprovados (§3.4, §8.4 e §10.2.3).
- **Secrets scan**: contratos, policies, named values, YAML, Bicep/Terraform, scripts, exemplos e evidências; referências ao Key Vault/identidade gerenciada, nunca segredo literal.
- **CORS**: origens de produção explícitas, métodos/headers mínimos e testes de recusa. A política corporativa deste guia admite `*` somente no sandbox, mesmo para uma API pública sem credenciais de navegador; CORS não autentica a chamada. Qualquer desvio desse padrão exige decisão registrada e exceção válida e aplicável de §11.5, refletida na ficha, na policy e no gate, antes de publicar (§7.5).
- **Produtos e limites**: produto/rota de consumo explícitos, sem acesso “All APIs/Unlimited” não autorizado. Definir a chave e o âmbito dos contadores por consumidor/jornada, inclusive para acesso anônimo quando admitido; testes não podem presumir que limites por subscription se aplicam sem subscription.
- **Dados e observabilidade**: diagnostics, correlação e mascaramento de informações pessoais (PII), com campos/finalidade e retenção por conjunto de evidência aprovados. Examinar também coletas independentes, como Defender for APIs e backend; desativar body logging no APIM não desativa essas coletas (§7.7, §10.7).
- **Aplicabilidade regulatória e risco**: combinar exposição, dados legais/corporativos, setor, consumidor e criticidade. Exigir mTLS/FAPI somente quando o perfil aplicável ou padrão corporativo explicitamente aprovado exigir; não inferir de LGPD/PCI. Registrar conjuntos de auditoria, dispositivos/prazos e gatilhos de Relatório de Impacto à Proteção de Dados Pessoais (RIPD), sem piso universal de logs [66][79].
- **Intake, threat model e exceções**: ficha atualizada (§6.2), análise de ameaças por fluxo (§7.6), autoridade e validade das exceções (§11.5). Sem obrigação dispensável comprovada, o gate não aceita uma exceção.
- **Pós-deploy**: smoke, contrato e negativos seguros, health probes WAF/APIM/backend, evidência de logs e observação de §9.2. Testes destrutivos de quota/carga usam ambiente/consumidor de teste e limites aprovados; não saturar jornadas reais em PROD.

### 9.4 Drift, exceções e rollback

#### 9.4.1 Drift detection

Alterações de policy ou configuração feitas no portal durante incidentes podem gerar divergência em relação ao IaC (drift). A equipe de plataforma precisa detectar e reconciliar essas divergências.

Ferramentas de referência e limites [13][14][26][95]:

- **APIOps** extrai/publica configuração APIM; não cria automaticamente um relatório confiável de drift. Na consulta de 09/10/2026, o README de `Azure/apiops` anuncia arquivamento e indica APIOps CLI. Registrar a ferramenta/versão escolhida, recursos cobertos e qualificação de migração; não trocar extractor silenciosamente.
- **Baseline aprovado**: configuração desejada do último release aceito no ambiente, com hash, manifesto, target e inventário. Extração e baseline precisam do mesmo layout, filtro/cobertura e normalizador versionado. Não comparar a pasta de contratos `apis` com uma árvore que também contém produtos, backends e named values.
- **Normalização**: resolver apenas diferenças ambientais e campos voláteis explicitamente aprovados, sem descartar policies, permissões ou valores relevantes. Overrides de filhos de workspace não aplicados pelo publisher escolhido exigem alternativa ensaiada ou bloqueio [26].
- **Prévia de infraestrutura**: ARM/Bicep what-if ou Terraform plan com os parâmetros reais do ambiente. Resultado incompleto, `Ignore` ou expressão não resolvida precisa de análise; não pode ser rotulado automaticamente como limpo.
- **Verificação agendada**: publicar resultado e evidência sanitizada. Exigir sucesso do extractor, inventário completo e layout validado antes do diff; baseline ausente ou extração vazia inesperada é erro. Um inventário legitimamente vazio só pode passar se aprovado e comprovado pelo manifesto.

No GNU `diff`, código 0 significa sem diferenças, 1 significa drift e qualquer código maior que 1 é tratado como erro [98]. Capturar stdout, stderr e o código; tamanho do arquivo não substitui o código. Usar `shell: bash`/`pipefail` para que falha do extractor não seja ocultada por `tee` [97]. Drift abre registro para reconciliar via PR/IaC; erro abre incidente de verificação e bloqueia o aceite. Mudanças emergenciais de portal precisam de autorização e reconciliação posterior rastreável, não de baseline reconstruído para ocultar o desvio.

#### 9.4.2 Exceções

Ver §11.5: modelo formal com ciclo de vida.

#### 9.4.3 Rollback

- **Pré-requisitos de recuperação**: antes do merge, registrar release anterior aceito (hash e parâmetros), recursos/estado abrangidos, dependências, plantão, permissões, comandos/procedimento validado, riscos e limites de aborto. O plano identifica o que é reversível e o que exige fix-forward; não prometer reversão imediata de SKU, rede ou migração de dados.
- **Revisions e versions**: em mudança compatível, testar uma revision não corrente e registrar como restaurar a revision anterior como corrente. Mudança incompatível mantém versão anterior e plano de migração do consumidor; redirecionamento só é executado se previsto e testado [26].
- **Blue/green ou canary**: definir dois alvos/revisions e o mecanismo de roteamento efetivo, com compatibilidade de rede, identidade e estado. Produtos no Developer Portal organizam acesso, mas não implementam divisão percentual de tráfego.
- **Falha antes de alterar PROD**: abortar, registrar causa e não publicar. Falha de deploy parcial, smoke ou observação: bloquear promoção, preservar operação/evidência e acionar o comandante de incidente e os plantões de API/plataforma. O mecanismo de recuperação só executa rollback automatizado se o cenário reversível tiver sido aprovado; nos demais casos aplica contenção e escalona uma correção para frente (fix-forward) autorizada. Falha da própria recuperação mantém o incidente aberto e o release reprovado.
- **Ensaio em UAT**: cobrir resolução de nomes (DNS), cadeia/certificado e autenticação aplicável, health probes, endpoint privado, grupos de segurança de rede (NSG), rotas definidas pelo usuário (UDR), firewall, WAF e jornada do consumidor, conforme recursos alterados. Incluir falha de failover/failback e dependências externas no contrato do runbook (§10.6).
- **Verificação e evidência**: vincular incidente/PR ao hash anterior/novo, parâmetros, operação ARM, diff de configuração/rede, resultados dos testes e consultas de telemetria. Após recuperação, cumprir `janela_saude_recuperacao_min`, limites e amostras de §9.2. Ausência de dados não prova saúde; registrar objetivos de tempo/ponto de recuperação (RTO/RPO) e medições por objeto, sem tratá-los como garantia do tier.

### 9.5 Multi-instância: repositórios e sincronização

Para corporações com várias instâncias APIM (modelo C), padronizar:

#### 9.5.1 Estrutura de repositórios

Duas estratégias comuns:

- **Monorepo**: um único repositório para todas as instâncias, organizado por pasta `/instances/{instancia}/...`. Vantagem: facilita o compartilhamento de policy fragments. Desvantagem: CI/CD mais complexo e maior abrangência do impacto de um PR.
- **Polirepo**: um repositório por instância + um repositório `apim-shared` com fragmentos. Vantagem: responsabilidades claras. Desvantagem: sincronização exige ferramentas específicas.

Recomendação: para 2-4 instâncias, monorepo com pipelines paralelos por instância. Para 5+, polirepo + repositório central de componentes compartilhados.

#### 9.5.2 Sincronização de policy fragments

- Fragmentos comuns (JWT validation corporativa, correlation ID, cors padrão) ficam em `apim-shared`.
- Publicação via release tag; instâncias consomem por versão (`apim-shared@v1.4.2`).
- Mudança incompatível exige incremento da versão major e plano de migração.

#### 9.5.3 Drift entre instâncias

- O relatório compara cada instância com a versão de fragmentos aprovada para seu release; versão nova ainda não adotada não é drift de configuração.
- **Atualização interna compatível**: alvo corporativo de adoção em até 60 dias da disponibilização do componente aprovado, com testes e recuperação. O prazo não obriga a instalar uma major sem análise de impacto.
- **Quebra para consumidores**: se a major alterar contrato, autenticação ou comportamento consumido, preservar a comunicação mínima de 90 dias de §11.4.2. Conciliar os prazos por anúncio antecipado antes da release do componente ou por versões em paralelo; não retirar o contrato anterior aos 60 dias só para cumprir a meta interna.
- **Correção urgente de segurança**: priorizar a contenção conforme risco, obrigação externa e contrato. Registrar comandante, comunicação, testes e recuperação; somente prazos de política interna dispensáveis admitem exceção válida (§11.5). A urgência não torna requisito legal dispensável nem permite alteração incompatível silenciosa.

#### 9.5.4 Catálogo unificado

- Cada instância publica seu inventário no **API Center** corporativo [10].
- Sincronização automática (cron) ou via webhook.

---

## 10. Operação, observabilidade e objetivos e acordos de nível de serviço

### 10.1 Sinais obrigatórios

Sinais mínimos por API, com fonte e cobertura registrados no dashboard/workbook. Percentis p50/p95/p99 representam a distribuição de latências; RPS são requisições por segundo. Não são todos métricas nativas universais: dimensões, logs e instrumentação dependem de recurso/tier/gateway [31][36]. Em workspaces, não depender da métrica de serviço `Requests`; obter os sinais por logs/instrumentação suportados. Consumption não oferece gateway request logs em Azure Monitor/Log Analytics, mas admite logs de requisição no Application Insights; self-hosted exige coleta/exportação própria. O aceite registra sinal alternativo e teste quando faltar o sinal pretendido [1][18][31].

- Latência gateway-only e end-to-end (p50, p95, p99).
- Taxa de erro separando **gateway** e **backend**: HTTP 4xx, 5xx, `GatewayResponseCodeCategory`, `BackendResponseCodeCategory`.
- Throughput/RPS por API, operação, produto, subscription e região.
- Latência do backend (`BackendDuration`) e duração total no gateway (`Duration`).
- Latência e falhas de validação de tokens (OAuth/OIDC/FAPI) quando aplicável.
- Cache hit/miss e uso de quota/rate-limit quando aplicável.

Sinais de capacidade por modelo de execução; **não misturar no mesmo alerta**:

| Modelo APIM | Métrica oficial | Uso recomendado |
|---|---|---|
| **Tiers clássicos** (Developer/Basic/Standard/Premium) | `Capacity` / **Capacity** | Tendência de saturação da instância clássica; média para escala e investigação de picos persistentes [6][36]. |
| **Tiers v2** (Basic v2/Standard v2/Premium v2; recurso de serviço APIM) | `CpuPercent_Gateway` e `MemoryPercent_Gateway`, percentuais de CPU/memória do gateway | Escala e diagnóstico do gateway v2, em vez de `Capacity`; confirmar origem e agregação [36]. |
| **Workspace gateway** (`Microsoft.ApiManagement/gateways`) | `CpuPercent_Gateway` e `MemoryPercent_Gateway`, percentuais de utilização do gateway | Consultar o recurso do gateway separado e a referência específica [63], não atribuir suas métricas a todo workspace lógico. No gateway padrão compartilhado, capacidade do serviço não isola o consumo de cada equipe. |

Sinais adicionais obrigatórios:

- **Rede/conectividade**: usar `NetworkConnectivity` por `ResourceType` somente quando disponível no recurso/modelo escolhido. Complementar com probes de DNS, entrada WAF, Key Vault, Event Hubs e backends; um private endpoint criado não prova conectividade ponta a ponta [36].
- **Mudança/drift**: coletar `AzureActivity` para alterações administrativas, incluindo APIs/policies, chaves, named values, diagnostics/loggers, certificados, RBAC e rede; correlacionar com identidade e mudança autorizadas.
- **Segurança**: 401/403, 429, anomalias por consumidor, falhas de tokens/certificados, alterações privilegiadas e tentativa administrativa indevida. Alertas Defender somente para APIs/gateways cobertos e integrados; manter regras SOC/telemetria alternativa para os demais [25][58].
- **Entrega de telemetria**: métricas de eventos Event Hubs expostas no recurso APIM, como drops/rejeições/throttling, são usadas apenas onde suportadas; **não estão disponíveis nos tiers v2** [2][36]. Em v2, monitorar o namespace Event Hubs, consumidor/checkpoints, atraso e ingestão do destino, com canário/heartbeat correlacionado ao envio. Registrar fonte, frequência, atraso tolerado, responsável e teste de falha; ausência de métrica não equivale a ausência de perda.

Logs mínimos:

- Gateway logs sem corpos ou cabeçalhos com dados pessoais por padrão. Não registrar credenciais, `Authorization`, refresh tokens, segredos ou subscription keys. Eventual campo pessoal necessário exige finalidade, hipótese legal, minimização, acesso e retenção aprovados, inclusive pseudônimos e identificadores de consumidor [66].
- Logs de atividade administrativa e de portal somente nas fontes/categorias suportadas; separar configuração, eventos de segurança e diagnóstico operacional.
- Diagnostic settings e coletores alternativos versionados em IaC, com teste de entrega e detecção de mudança manual. A política de logging do gateway não substitui a análise dos dados coletados por Application Insights, Defender, borda ou consumidores de eventos (§10.2.3).

### 10.2 Logs, métricas e alertas

#### 10.2.1 Categorias e tabelas oficiais

Habilitar categorias e destinos **suportados e necessários à finalidade**, não todas as opções indistintamente: Log Analytics em região aprovada, Sentinel quando aplicável, armazenamento imutável quando exigido e Event Hubs quando houver consumidor. Preferir tabelas específicas do recurso; logs de workspaces usam `AzureDiagnostics` conforme a referência atual [31][36][37]. Consumption e self-hosted seguem os caminhos alternativos de §10.1; a tabela abaixo não promete disponibilidade de todas as categorias em todos os runtimes.

| Origem | Categoria oficial | Tabela Log Analytics recomendada | Uso |
|---|---|---|---|
| APIM gateway | `GatewayLogs` | `ApiManagementGatewayLogs` | Requisições HTTP, latência, códigos, erro, cache, API/operação/produto/subscription. |
| WebSocket | `WebSocketConnectionLogs` | `ApiManagementWebSocketConnectionLogs` | Handshake, abertura/fechamento e falhas de conexões WebSocket. |
| Developer Portal | `DeveloperPortalAuditLogs` | `APIMDevPortalAuditDiagnosticLog` | Login, navegação, teste de API e ações no portal do desenvolvedor. |
| Gateway de modelos de linguagem (LLM, Large Language Model) | `GatewayLlmLogs` quando suportada | `ApiManagementGatewayLlmLog` | Tokens/modelo e sinais necessários; prompt/resposta somente com finalidade, hipótese legal quando aplicável, minimização e aprovação de dados/segurança. |
| MCP gateway | `GatewayMCPLogs` quando suportada | `ApiManagementGatewayMCPLog` | Uso de ferramentas MCP, sem presumir coleta integral ou categoria em todo gateway. |
| Plano de controle Azure | Activity Log | `AzureActivity` | Alterações administrativas: recursos, RBAC, diagnostics, certificados, tier e rede. |
| Métricas exportadas | `AllMetrics` quando a métrica admitir exportação | `AzureMetrics` | Somente métricas/dimensões expostas no recurso/tier. Não cria `Requests` em workspaces nem métricas de eventos Event Hubs no APIM v2; usar fontes alternativas. CPU/memória de workspace gateway têm referência própria [36][63]. |
| Legado/compatibilidade | categorias antigas/serviços sem tabela dedicada | `AzureDiagnostics` | Usar só quando resource-specific não estiver disponível. |

Application Insights por API é opcional e deve ser do tipo workspace-based, com connection string/Managed Identity sempre que possível, para correlação fim-a-fim do produtor [32]. Métricas customizadas via policy (`emit-metric`) só para sinais agregados; não usar para PII.

#### 10.2.2 Alertas mínimos de produção

A tabela adota o seguinte modelo operacional inicial: **Sev1** identifica erro com resposta urgente ao impacto confirmado; **Sev2**, aviso que exige investigação ou correção preventiva; **Sev3**, informação para acompanhamento e planejamento. Esses critérios de resposta são propostas deste guia, não uma classificação universal de incidentes nem um SLA. Os responsáveis aprovam impacto, limiar, janela, destinatários e escalonamento por jornada. O campo de severidade do Azure Monitor admite valores de 0 a 4; a tabela usa os três níveis acima [109].

| Alerta | Fonte/KQL ou métrica | Severidade inicial | Ação obrigatória |
|---|---|---|---|
| 401/403 anômalo | `ApiManagementGatewayLogs` por `ResponseCode in (401,403)` e baseline por API/client-id/IP | Sev3; Sev2 se API regulada | Verificar token issuer/audience/scope, mTLS, client bloqueado, possível credential stuffing. |
| 429/rate limit | `ApiManagementGatewayLogs` `ResponseCode == 429` + dimensão subscription/API | Sev3 | Validar quota contratada, abuso, retry/backoff do consumidor, necessidade de escala. |
| 5xx gateway/backend | `Requests` com dimensões suportadas; em workspaces ou fonte ausente, logs/instrumentação do runtime | Sev2 inicial, ajustada por impacto | Acionar responsáveis API/backend; consultar runbooks #1/#2. Não depender de `Requests` no workspace [31][36]. |
| Conectividade de backend | Duração/erro em fonte suportada, `NetworkConnectivity` quando disponível e probe sintético | Sev2 inicial; Sev1 se impacto crítico | Validar DNS, rotas, endpoint, firewall, TLS e esgotamento de conexões/portas; distinguir métrica de evento em log. |
| Capacidade clássica | `Capacity` no recurso clássico e limiar/janela aprovados; 15 min como ponto inicial de avaliação | Sev3/Sev2 | Escala e investigação sustentadas por teste e tendência; não usar esse sinal para Consumption ou v2 [36]. |
| Capacidade v2 / gateway de workspace | CPU/memória no recurso correspondente, limiar/janela aprovados | Sev3/Sev2 | Escalar serviço ou gateway conforme associação, revisar policies/cache/backend e confirmar disponibilidade das métricas [36][63]. |
| Certificado: 30 e 7 dias antes da expiração | Key Vault certificate expiring ou inventário de custom domains/certificados | Sev3 aos 30 dias e Sev2 aos 7 dias antes da expiração | Rotação via runbook 10.6 #3; evidência de validação em DEV/PROD. |
| Ausência/atraso de evidência | Fonte esperada para o runtime: tabela específica, `AzureDiagnostics`, Application Insights ou sink do self-hosted; canário com ID e atraso tolerado aprovados | Sev2 inicial | Distinguir ausência de tráfego de falha de entrega; verificar configuração, sampling, ingestão, consumidor e mudanças. Janela de 15–30 min é candidata, não requisito universal. |
| Entrega via Event Hubs em v2 | Métricas do namespace, progresso/checkpoints do consumidor, ingestão do destino e canário ponta a ponta | Conforme criticidade da evidência | Acionar responsável da coleta, medir lacuna e recuperar pelo procedimento aprovado; não usar métricas Event Hubs do recurso APIM ausentes em v2 [36]. |
| Mudança manual | `AzureActivity` correlacionado com identidade e mudança autorizada | Sev2; Sev1 se impacto crítico | Abrir incidente de drift, preservar evidência e reconciliar; contenção de permissões segue procedimento controlado. |
| Evento de segurança | Defender apenas no escopo coberto/onboarded; regras SOC/Sentinel e controles alternativos | Conforme regra SOC | Triagem, preservação e fluxo de incidente aplicável; DPO apoia quando houver dado pessoal, sem substituir o controlador. |

#### 10.2.3 Gate de residência regional da observabilidade

Antes de habilitar coleta, catálogo ou novo destino, registrar na ADR conjuntos/campos e finalidade, classificação corporativa e legal, regiões de armazenamento/processamento, acessos/suporte, responsáveis, retenção e mecanismos contratuais/legais pertinentes. Aplicar também às cópias coletadas por borda, Defender e integrações, não apenas ao Log Analytics (§3.4/Apêndice C).

| Recurso | Decisão de localização e coleta | Evidência mínima |
|---|---|---|
| **Log Analytics** | Separar workspaces quando geografia, responsabilidade, acesso ou política de dados o exigir | ADR com região, categorias/tabelas, conjunto/finalidade, responsável, política de retenção versionada e acesso [38][40]. |
| **Application Insights** | Aprovar região e workspace vinculado, IPs/URLs/propriedades e eventual conteúdo coletado | Configuração e teste de minimização/mascaramento, acesso e aprovação do responsável pelos dados; DPO/jurídico quando aplicável [32][43]. |
| **Event Hubs** | Para diagnostic settings de recurso regional, namespace regional compatível; distinguir DR de metadados de replicação de eventos | Destino/consumidor aprovados, captura/retenção, recuperação de checkpoints/eventos e canário. Geo-DR de metadados não recupera os eventos por si só [37][44]. |
| **Microsoft Sentinel** | Região do Log Analytics associado e segregação exigida pela operação/controles | Aceite SOC e política de retenção por conjunto/finalidade; prazo incluído na oferta não é mínimo normativo universal [40][42]. |
| **Azure API Center** | Região permitida para metadados, contratos, anexos, exemplos e acessos | Inventário dos campos e aprovação regional. Não replica dados entre regiões nem oferece failover inter-regional; prever inventário recuperável autorizado [10][41]. |
| **Arquivo/auditoria** | País/geografia e imutabilidade quando exigidos pela finalidade/norma/contrato | Política de retenção por conjunto, prazo/marco inicial, acesso, chave, lifecycle e configuração locked/legal hold quando requerida; WORM não cria base legal [39][66]. |

**Aprovação:** exposição, dados pessoais comuns/sensíveis legais, regimes setoriais, consumidor e criticidade se compõem. Dado financeiro ou rótulo corporativo "restrito" não equivale automaticamente a sensível LGPD. Para transferência internacional de dado pessoal, identificar exportador/importador, países e acessos, hipótese legal e mecanismo válido; aprovação interna e aditivo de proteção de dados (DPA) genérico não demonstram, sozinhos, atendimento à LGPD/ANPD [66][67]. Plataforma e responsável pelos dados registram a decisão; DPO/jurídico/compliance participam conforme a obrigação aplicável. Sem destino e política aprovados para o conjunto, o gate bloqueia sua coleta/publicação. Um catálogo opcional pode ser faseado com inventário autorizado, sem dispensar os controles de uma API em produção.

### 10.3 Dashboards

A operação deve usar **Azure Monitor Workbooks** e o dashboard do APIM baseado no Azure Monitor, não o analytics clássico. O analytics clássico integrado aos tiers clássicos será aposentado em **15/03/2027**; até lá, mantê-lo em paralelo apenas para comparação e migrar consultas, alertas e relatórios para Log Analytics/Workbooks [11][31][53].

Mínimo por instância APIM:

- Dashboard/Workbook "Plataforma": saúde geral, `Requests`, latência, erros, capacidade correta por modelo (Capacity clássico vs CPU/memória v2/workspace), top APIs por volume/erro.
- Dashboard "Segurança": falhas de tokens/certificados, 401/403 e anomalias de consumidor; mostrar cobertura/onboarding Defender por API/gateway e a fonte alternativa de detecção para workspaces/self-hosted ou outro cenário excluído [25][58].
- Dashboard "FinOps": ingestion Log Analytics, Defender for APIs, egress (§8.6).
- Dashboard por produto/domínio: APIs do domínio com SLO/erro/latência, consumo por consumer owner e quota.
- Dashboard "Mudança e drift": AzureActivity, PR/deploy correlacionado, mudanças manuais, owner, rollback/evidência.
- Dashboard "Residência e retenção": workspace/destino por região, tabelas com retenção, WORM/Storage, exceções vencidas.

Plano de migração do analytics clássico:

| Data limite | Entrega | Owner |
|---|---|---|
| D+30 após aprovação do guia | Inventário de usuários/relatórios que ainda usam Analytics (classic) | Plataforma + equipe responsável pela experiência do desenvolvedor (DevEx, neste guia) |
| D+60 | Workbooks equivalentes publicados e validados com owners | Plataforma + Observabilidade |
| D+90 | Alertas migrados para métricas/KQL oficiais; classic só read-only | Plataforma + SOC |
| Até 2027-01-15 | Comunicação final e treinamento; dashboard clássico removido de rotinas | DevEx |
| Até 2027-03-15 | Zero dependência operacional do analytics clássico | Plataforma |

### 10.4 SLO, SLA e capacidade

- **SLO**, objetivo operacional do serviço, definido pelo responsável pelo produto e validado pela plataforma: indicador, fronteira gateway/serviço completo, janela, exclusões e método de medição. Disponibilidade de 99,9% e p99 inferior a 500 ms são exemplos a testar, não garantias do tier.
- **SLA**, acordo com o consumidor, formalizado com escopo e dependências explícitos. Como critério corporativo inicial, exigir três meses de evidência do SLO antes de assumir um SLA externo; esse histórico não amplia o SLA do fornecedor nem prova todos os modos de falha.
- **Orçamento de erro (error budget)** calculado para a janela e indicador acordados, com divulgação ao público autorizado e regra de restrição de mudanças. Definir tratamento de correções urgentes sem usar a meta como dispensa de controle.
- **Planejamento de capacidade** trimestral; teste de carga obrigatório a cada upgrade/downgrade de SKU.
- Alerta de escala deve usar o sinal correto do runtime: Capacity para clássico; CPU/memória para v2 e workspace gateway (§10.1).

#### 10.4.1 Metas ilustrativas de disponibilidade e latência

Os cenários abaixo preservam referências iniciais para discussão, **não requisitos derivados de LGPD/Open Finance nem classes exclusivas**. Exposição, regime de dados e criticidade se compõem; o responsável aprova a meta pelo impacto de negócio e pelas evidências do serviço. p99 é o percentil 99 da latência, não o tempo máximo de toda requisição.

| Cenário ilustrativo | Disponibilidade proposta | p99 somente do gateway | Orçamento de indisponibilidade em 730 h |
|---|---|---|---|
| Operação financeira crítica, se o negócio adotar essa meta | 99,95% | ≤ 800 ms | 21,9 min |
| Tratamento de dados pessoais, conforme criticidade definida | 99,9% | ≤ 800 ms | 43,8 min |
| Serviço externo entre empresas | 99,9% | ≤ 1.000 ms | 43,8 min |
| Serviço interno de criticidade compatível | 99,5% | ≤ 1.500 ms | 219 min (3 h 39 min) |
| Administração em janela comercial | 99,5% na janela acordada | ≤ 2.000 ms | Calcular sobre as horas da janela |
| Serviço público sem autenticação | 99,9% | ≤ 1.500 ms | 43,8 min |

A coluna de orçamento usa `730 h × (1 − disponibilidade)`, apenas para um indicador temporal de disponibilidade. Para SLO por proporção de requisições, o orçamento é expresso em requisições, não inferido desses minutos. Latência de gateway exclui backend; o SLO ao consumidor mede a jornada completa com dependências e janela reais. As metas precisam de teste, operação e aprovação; rótulo regulatório ou tier não comprova atendimento.

### 10.5 Alta disponibilidade, recuperação de desastre e estado recuperado

**Legenda:** HA (High Availability) = alta disponibilidade; DR (Disaster Recovery) = recuperação de desastre; AZ = zona de disponibilidade; RTO (Recovery Time Objective) = tempo-alvo de retomada; RPO (Recovery Point Objective) = perda máxima-alvo de estado/dados. As faixas abaixo são candidatas de planejamento por criticidade, não resultados garantidos por tier ou requisito automático de um setor.

A ADR fixa o início/fim da medição de RTO e o critério de serviço recuperado. Definir RPO **separadamente** para configuração APIM, chaves/certificados, transações do backend e evidências de auditoria/eventos. A meta do serviço de negócio inclui rede, identidade, backend e destinos críticos; uma métrica de recuperação do gateway não comprova essa meta. Valores próximos de zero precisam de replicação/procedimento e ensaio do objeto correspondente, não apenas multirregião APIM [7][9][56].

| Criticidade de negócio | Configuração candidata, após gate | HA regional | Recuperação regional | RTO candidato do serviço | RPO candidato, a aprovar por objeto | Prova de recuperação |
|---|---|---|---|---|---|---|
| **Crítica** | Premium clássico se multirregião nativa/self-hosted/injection elegível forem exigidos; outras topologias somente se comprovarem os requisitos | AZ elegível e capacidade após falha testadas | Multirregião clássica ou instâncias independentes com roteamento e dependências preparados | < 5 min | Próximo de zero somente para objetos cuja recuperação sem perda tenha sido demonstrada; não inferido do tier | Prova antes do go-live crítico; ensaio anual inicial e após mudança material |
| **Alta** | Tier e topologia que comprovem capacidade/continuidade; Premium v2 somente em região elegível | Premium clássico/Premium v2 conforme suporte; Standard v2 condicionado ao conflito abaixo | Clássico nativo ou segunda instância independente; v2 sem multirregião nativa | < 30 min | < 5 min como candidato por objeto, incluindo backend/evidência quando no escopo | Ensaio semestral inicial, com limites de sucesso/aborto |
| **Média** | Standard v2 se atender rede/carga e requisitos de falha | Redundância de unidades; AZ Standard v2 pendente de confirmação documental/regional | IaC e artefatos versionados + recuperação de dependências, ou instância preparada conforme a meta | < 4 h | < 1 h como candidato por objeto | Ensaio anual inicial e recuperação medida |
| **Baixa** | Basic v2/Standard v2 quando elegíveis | Basic v2 sem AZ; Standard v2 condicionado abaixo | Reconstrução por IaC/artefatos e recuperação do estado externo | < 24 h | < 24 h como candidato por objeto | Reconstrução, teste de saúde e registro de perdas |
| **Sandbox/desenvolvimento** | Developer | Sem SLA de produção | Recuperação conforme necessidade de testes | Não aplicável à produção | Não aplicável à produção | Recriação/testes do ambiente |

**HA por runtime:** as capacidades regionais de um gateway gerenciado não são herdadas automaticamente por um workspace gateway ou self-hosted gateway. A matriz de gateways não oferece AZ ou multirregião nativas no workspace gateway [54]. No self-hosted, o cliente projeta e comprova disponibilidade, distribuição e operação de containers; associar o runtime a Premium não transfere a redundância Azure a esses containers. Registrar HA/DR por gateway e por dependência, não apenas por tier [54][56].

**Suporte zonal pendente em Standard v2:** a comparação de features e a matriz de gateways negam AZ [1][54], enquanto as páginas de confiabilidade e habilitação incluem Standard v2 [56][57]. As últimas exigem região elegível e habilitação na criação para Standard v2/Premium v2, sem ativação em instância existente. Registrar o conflito na ADR e obter confirmação do tier, região, API/configuração e capacidade antes de compromisso; não concluir suporte em Brazil South nem resolver a divergência por inferência. Basic v2 não oferece AZ nas fontes consultadas.

**Multirregião e escopo:** o recurso nativo é Premium clássico e replica o gateway, não o plano de gestão/Developer Portal para a região secundária nem o backend. Premium v2 não oferece multirregião nativa, mas isso não torna Premium clássico obrigatório apenas por criticidade ou regulação [7].

**Backup/restore:** elegível nos tiers clássicos Developer/Basic/Standard/Premium, mas não em v2/Consumption nem em instância com workspace gateways associados. Backup expira em 30 dias e não inclui certificados, rede, identidade gerenciada, diagnostics ou conteúdo do portal; não substitui IaC, artefatos de APIs/policies, inventário e recuperação de dependências [9]. Reconstruir/testar também DNS, rotas, endpoints, WAF, segredos, RBAC, catálogo, loggers, evidência/eventos e backend conforme o escopo. Se o desenho depender de workspace gateway, o caminho de recuperação precisa ser validado sem backup/restore nativo.

**Localização e aceite:** a recuperação só usa países/destinos permitidos para cada conjunto, incluindo telemetria, metadados, acesso e dados transacionais (§10.2.3). Requisitos de continuidade e residência precedem o compromisso de tier/topologia. Antes de go-live crítico, comprovar recuperação acordada ou registrar apenas o aceite de risco permitido pelas obrigações aplicáveis; nenhum prazo de roadmap substitui a prova [66][67].

### 10.6 Runbooks/playbooks operacionais mínimos

Cinco runbooks obrigatórios, mantidos no repositório de operação, ensaiados periodicamente:

1. **Aumento de erros 5xx em uma API**: consultar dashboard → identificar backend → acionar circuit breaker → comunicar consumidores → escalar para o responsável pela API e o plantão do backend.
2. **Indisponibilidade total do backend**: confirmar via alerta + dashboard → ativar fallback (se houver) → comunicar → atualizar página de status → coordenar com o responsável pelo backend.
3. **Certificado próximo de expirar/expirado**: alertar 30 e 7 dias antes como referência inicial → renovar/rotacionar conforme emissor → validar em desenvolvimento/UAT → trocar bindings de borda, APIM e backend na janela autorizada → verificar todos os segmentos TLS e mTLS. Em perfil FAPI/Open Finance, atender à infraestrutura de chaves públicas (ICP) e ao perfil efetivamente aplicável [19][64].
4. **Rotação de chave/segredo**: preparar novo valor e versões, atualizar consumidores/policies por procedimento autorizado, verificar uso e revogar o anterior conforme risco/janela. Em incidente, containment e evidência precedem a carência normal. Recuperação não pode restaurar automaticamente uma credencial revogada.
5. **Failover/failback de DR**: confirmar o modo de falha e autoridade de ativação → verificar capacidade, configuração, credenciais, rede, backend e telemetria na região permitida → mudar o roteamento conforme o desenho → validar serviço e estado recuperados dentro dos limites acordados → comunicar. Se falhar a validação, aplicar critério de aborto/contingência e preservar evidência; não alternar DNS repetidamente sem controle. No retorno, reconciliar estado e executar failback autorizado com nova verificação. Medir RTO e RPO por objeto (§10.5) e anexar a prova definida em §9.

Fluxos adicionais, obrigatórios quando o cenário/ato aplicável os exigir:
6. **Incidente com dados pessoais**: o controlador, com responsável/substituto e apoio do DPO, avalia risco ou dano relevante e comunica à ANPD **e aos titulares** quando cabível, em três dias úteis do conhecimento de que o incidente afetou dados pessoais, ressalvada legislação específica. Não esperar a investigação completa: emitir comunicação inicial e, quando necessária, complementação fundamentada em vinte dias úteis. Registrar relógio, decisão, canais, campos e evidência, inclusive incidente não comunicado; a retenção do registro segue a obrigação/política específica, não todo gateway log [27].
7. **Incidente cibernético setorial**: compliance identifica entidade/serviço abrangidos, ato vigente, destinatários, gatilhos e prazos do fluxo aplicável. Resolução CMN nº 4.893/2021 tem escopo próprio; não é regra universal de toda API financeira nem se substitui por Resolução BCB nº 198/2022 [28c].
8. **Relatório mensal FinOps**: fechamento de custo por componente e alocação por domínio.
9. **Falha/ausência de telemetria**: verificar a fonte suportada, envio, namespace/consumidor Event Hubs, checkpoints, ingestão e canário, além de configuração, sampling e RBAC. Medir lacuna, acionar responsável e recuperar evidência quando previsto; em v2, não esperar métricas Event Hubs do recurso APIM [36].
10. **Drift/mudança manual em produção**: conter mudanças de forma autorizada, preservar AzureActivity, reconciliar IaC e conduzir análise de causa raiz (RCA, Root Cause Analysis), com evidência e ação corretiva.

Cada runbook externo registra responsável e substituto, escopo/objetos, gatilho, pré-requisitos, dependências, permissões, sequência executável, limites numéricos de validação acordados, critérios de sucesso e aborto, contingência/rollback/recuperação, comunicação, evidência, frequência e data do último ensaio. A duração de verificação e atraso tolerado são parâmetros com unidade e valor aprovados por serviço, não símbolos genéricos "N". Falha de ensaio gera pendência ou aceite de risco permitido, não aprovação automática.

### 10.7 Retenção, finalidade e imutabilidade de auditoria

A política aprovada de retenção identifica, para cada conjunto e destino, os campos necessários, finalidade, base do tratamento, obrigação legal/contratual ou padrão interno, responsável, prazo e evento inicial, além de eliminação e eventual suspensão por obrigação de conservação. O gate usa essa política, não um número associado à classe da API. Normas e contratos aplicáveis não são dispensados por aprovação interna; padrões corporativos adicionais precisam de justificativa compatível com a LGPD e de processo próprio de mudança. Os intervalos operacionais abaixo são exemplos a aprovar, não pisos normativos [66]:

| Conjunto de registros | Finalidade | Origem da obrigação ou política | Prazo e marco inicial | Proteção/imutabilidade |
|---|---|---|---|---|
| `ApiManagementGatewayLogs` de diagnóstico, inclusive de API regulada | Diagnóstico, objetivos de serviço, capacidade e suporte | Política operacional aprovada; base LGPD analisada quando houver dados pessoais | Exemplo: 30–90 dias a partir do evento. Não existe aqui um mínimo legal geral de 1–5 anos para gateway logs | Acesso restrito e coleta mínima; preservar os eventos selecionados para investigação, sem manter todo payload |
| Consentimentos, revogações e demais informações/documentos Open Finance enumerados no art. 49 | Evidenciar o tratamento e os controles das participantes abrangidas | Resolução Conjunta nº 1/2020, art. 49, versão 9 [78] | Cinco anos. Contratos do inciso VIII: a partir da extinção; mecanismos dos incisos IX e XI: a partir da implementação. Para os demais itens, formalizar o marco por conjunto com jurídico/compliance, respeitando a obrigação, sem inventar início único pela requisição do gateway. O antigo inciso III está revogado | Evidência no sistema de domínio/repositório apropriado; não presumir que um gateway log contém consentimento, contrato ou controle necessário |
| Política, planos, relatórios, due diligence, contratos e controles de entidade abrangida | Provar requisitos de cibersegurança/contratação relevantes | CMN nº 4.893/2021 ou BCB nº 85/2021, art. 23, conforme entidade [28b][28c] | Cinco anos para os itens enumerados; contratos: extinção; mecanismos de controle: implementação; testes de intrusão e planos de correção: execução dos testes. Registrar os marcos dos demais documentos na política aplicável | Repositório de evidências com integridade e acesso demonstrados; não confundir com duração de toda telemetria APIM |
| Registro de incidente, comunicado ou não | Provar avaliação de risco, medidas e comunicação ou justificativa da ausência | Resolução CD/ANPD nº 15/2024, art. 10 [27] | Pelo menos cinco anos a partir da data do registro; maior se obrigação adicional exigir. Para entidades do art. 23 da LGPD, observar a ressalva de guarda permanente/tabela de temporalidade do art. 10, §2º | Campos mínimos do art. 10, acesso restrito e preservação da evidência pertinente |
| Eventos de negócio/transação não cobertos pelas linhas anteriores | Disputa, prestação de contas e obrigação setorial/contratual identificada | Norma, inciso e contrato efetivamente aplicáveis; não usar apenas o rótulo financeiro | Prazo e marco fixados por conjunto na política aprovada. Não presumir cinco anos para toda transação de qualquer empresa | Sistema de domínio registra o evento e seu resultado; gateway contribui com correlação minimizada |
| `AzureActivity`, mudanças APIM/IaC e decisões de implantação | Auditoria de mudança, segregação de funções e investigação | Política GRC (governança, riscos e conformidade); obrigações específicas identificadas | Prazo aprovado por conjunto e marco do evento/encerramento pertinente; não há piso geral de sete anos neste guia | Integridade e retenção apropriadas à evidência crítica; correlacionar aprovação e artefato |
| Auditoria do Developer Portal | Acesso e gestão de consumidores | Política de segurança/suporte e base do tratamento quando aplicável | Prazo aprovado a partir do evento ou encerramento da relação, conforme conjunto | Não impor WORM a todo registro apenas pelo rótulo regulado; definir a evidência que precisa de preservação |
| Application Insights traces/dependencies | Diagnóstico da aplicação produtora | Política operacional e base do tratamento pertinente | Exemplo: 30–90 dias a partir do evento; extensão justificada | Sem tokens, headers de credencial, segredos ou corpo com dados pessoais |
| Alertas e eventos de segurança no Sentinel | Detecção, resposta e investigação | Política do centro de operações de segurança e obrigações aplicáveis | Janelas de consulta e arquivo aprovadas por conjunto; incidentes seguem a linha específica acima | Preservar evidência selecionada; não duplicar indistintamente logs por cinco anos |
| Exportações de auditoria, análise de causa e litígio | Prova de controle ou obrigação de conservação | Política GRC, obrigação aplicável e eventual legal hold documentado | Prazo e marco do conjunto; registrar a suspensão da eliminação, seu responsável e sua revisão | Storage WORM/time-based retention locked ou legal hold quando requerido; testar acesso e recuperação |

Para auditoria crítica, enviar a evidência necessária para Azure Storage com **WORM** (Write Once, Read Many), por time-based retention locked ou legal hold conforme a política aprovada. Testar a configuração antes do bloqueio: no estado locked, a retenção não pode ser reduzida/removida dentro do período [39]. Log Analytics oferece analytics/long-term retention até 12 anos, mas essa duração de armazenamento não equivale a imutabilidade [38][39]. WORM não cria base legal, não determina por si o prazo e não impede leitura indevida; controlar acesso, recuperação, conservação adicional e eliminação ao fim da obrigação, respeitando a LGPD [66].

---

## 11. Governança corporativa, responsabilidades e experiência do desenvolvedor

### 11.1 Princípios de governança

- **Governança via guardrails técnicos**: Azure Policy, gates de pipeline, lint e validação dos requisitos aplicáveis. Documentos registram decisão, autoridade e evidência; não substituem o controle técnico.
- **Responsabilidades explícitas**: cada API identifica pessoa, substituto e plantão para produto, implementação, dados e plataforma. O encarregado de proteção de dados (DPO) orienta e atua nos casos aplicáveis, sem substituir a responsabilidade legal do controlador [66][105].
- **Registro de decisão arquitetural (ADR, Architecture Decision Record)**: documentar separadamente modelo de adoção, tier, gateway/rede/região e configuração, com decisor único, consultas e condições de revalidação.
- **Catálogo único** (API Center [10]) como visão executiva.
- **Autoatendimento para mudanças padronizadas**, aprovação específica para exceções (§11.6).

### 11.2 Responsabilidades mínimas

Cada API publicada precisa registrar na ficha (§6.2) os papéis abaixo, com responsável nominal e grupo substituto:

- **API/Product owner** (squad produtora): responsável final por valor, objetivo de nível de serviço (SLO), backlog e comunicação de mudança.
- **Tech lead produtor**: executa e responde tecnicamente pela implementação e compatibilidade do contrato e das policies.
- **Consumer owner**: responsável pelo sistema consumidor/parceiro; recebe mudanças, acompanha limites e incidentes de consumo.
- **Data owner**: decide a classificação corporativa e os usos autorizados sob delegação interna, registrando finalidade, enquadramento legal, retenção e mascaramento com jurídico/compliance. Essa delegação não transfere a responsabilidade legal do controlador.
- **Platform owner APIM**: responsável final pela instância, runtime, diagnostics, rede e pipeline comum; mantém executor e plantão da plataforma.
- **Security owner**: responsável pelos controles de identidade aplicáveis, análise de ameaças e resposta de segurança. Autenticação mútua TLS (mTLS) e FAPI dependem do fluxo/perfil aplicável, não de rótulo genérico de dados.
- **Controlador e representante autorizado**: quando há dados pessoais, registrar a entidade controladora, quem decide/responde em seu nome, substituto e canal de escalonamento, inclusive para comunicação de incidentes.
- **DPO, compliance e jurídico**: orientação e consultas nos fluxos de dados/regulação, inclusive RIPD, residência/transferência e incidentes. O DPO não é o aprovador universal de releases ou de desvios legais [66][105].
- **Escala de plantão**: produtor, plataforma e segurança, com contato primário/substituto, comandante de incidente e procedimentos disponíveis.
- **Aprovador final de release crítica/regulada**: pessoa autorizada e segregada dos autores reais da mudança e dos iniciadores humanos da implantação (§11.6.1).

Nenhuma API entra em PROD sem pessoa ou papel operacional nominal, substituto e escalonamento. Cada atividade tem um único responsável final (Accountable); executor, consultado e aprovador são funções distintas, mesmo quando a matriz admite delegação formal.

### 11.3 RACI resumido

Legenda RACI: **R** executa (Responsible); **A** responde pela decisão e aceite (Accountable); **C** é consultado (Consulted); **I** é informado (Informed). Cada linha abaixo tem um único A nominal e substituto registrado. Comitês são consultados; uma aprovação coletiva não substitui o decisor.

| Atividade | A único | Executores (R) | Consultados (C) | Informados (I) |
|---|---|---|---|---|
| Design e contrato da API | API/Product owner | Tech lead produtor | Consumer owner, dados, plataforma e segurança; compliance/DPO quando aplicável | Auditoria |
| Classificação corporativa e registro dos dados (§7.7/§10.7) | Data owner | Tech lead e Data owner | Segurança, jurídico/compliance/DPO e plataforma | API/Product owner e consumidor |
| Modelo de adoção, tier e topologia na ADR | Responsável de arquitetura designado | Plataforma | Segurança, dados, continuidade e FinOps | Produtores, consumidores e auditoria |
| Publicação em DEV/UAT | API/Product owner | Tech lead e plataforma | Dados e segurança | Consumidores |
| Publicação em PROD e aceite da jornada | API/Product owner | Tech lead e plataforma | Segurança, dados e compliance quando aplicável | Consumidores e auditoria |
| Autorização final PROD crítica/regulada (SoD) | Aprovador de release designado e segregado | Plataforma/identidade de publicação | API/Product owner, segurança e compliance quando aplicável | Auditoria |
| Policies padrão de plataforma | Platform owner APIM | Plataforma | Segurança e produtores | Consumidores |
| Policy custom de domínio | API/Product owner | Tech lead | Plataforma, segurança e dados; compliance quando aplicável | Consumidores |
| Monitoramento e operação da API | API/Product owner | Tech lead/plantão produtor | Plataforma e segurança | Consumidor |
| Monitoramento e operação da plataforma | Platform owner APIM | Plantão da plataforma | Produtor e segurança | API/Product owner |
| Incidente de backend/API | API/Product owner | Plantão produtor | Plataforma, consumidor, segurança e dados quando afetados | Auditoria |
| Incidente de segurança técnica | Security owner | Segurança e plantões produtor/plataforma | Dados, jurídico/compliance/DPO quando aplicável | API/Product owner e auditoria |
| Avaliação e comunicação de incidente com dados pessoais | Representante autorizado do controlador | Segurança e plantões, com jurídico/compliance | DPO, Data owner e responsáveis pelas APIs | Auditoria; destinatários externos conforme obrigação aplicável |
| Residência/destinos de telemetria (§10.2.3) | Data owner | Plataforma | Jurídico/compliance/DPO e segurança | Produtor e auditoria |
| Retenção e armazenamento imutável (WORM: gravação única, múltiplas leituras), §10.7 | Data owner | Plataforma | Jurídico/compliance/DPO, segurança e auditoria | API/Product owner |
| Exceção operacional interna | Platform owner APIM | Requisitante e plataforma | API/Product owner | Auditoria |
| Exceção a padrão técnico interno | Responsável de arquitetura designado | Requisitante e plataforma | Tech lead | Auditoria |
| Exceção de segurança interna dispensável | Autoridade de segurança designada | Requisitante e plataforma | Dados, jurídico/compliance/DPO quando aplicável | API/Product owner e auditoria |
| Exceção interna sobre dados/residência/retenção | Data owner delegado | Requisitante e plataforma | Jurídico/compliance/DPO e segurança | API/Product owner e auditoria |
| Determinação de aplicabilidade/impedimento legal | Representante autorizado do controlador ou da entidade obrigada, identificado no registro | Jurídico/compliance | Dados, segurança e DPO quando aplicável | Plataforma, produtores e auditoria |

Na última linha, o registro escolhe exatamente um representante da entidade obrigada; não há dois aprovadores alternativos ativos. O Data owner decide controles internos sob delegação e consulta jurídica, mas o controlador conserva suas obrigações. Exceções seguem §11.5.3; nenhum A interno pode afastar obrigação externa.

### 11.4 Experiência do produtor e consumidor

- **Self-service** no Developer Portal: registro, subscription, gestão de keys, exemplos, sandbox.
- **Ferramentas do produtor**: templates Bicep, fragmentos de policy, esqueleto OpenAPI (Apêndice B), pipeline de referência (Apêndice D).
- **Documentação atualizada**: contrato do protocolo, exemplos e changelog derivados da fonte versionada; mudanças visíveis no portal. OpenAPI cobre HTTP/REST e eventuais fachadas HTTP, não substitui schemas/descritores de outros protocolos.
- **Acordos de nível de serviço (SLAs) do canal de suporte** definidos para consumidores externos, sem confundir tempo de resposta do suporte com o SLO técnico da API.

#### 11.4.1 Jornada completa do consumidor

| Etapa | Experiência esperada | Evidência/controle |
|---|---|---|
| 1. Descobrir API | Consumidor encontra a API no API Center/Developer Portal por domínio, versão, protocolo, ambiente, owner e suporte | Metadados obrigatórios, contrato aplicável, exemplos e changelog |
| 2. Entender contrato | Lê identidade/scopes aplicáveis, limites, mensagens/payloads, erros, SLO, uso e dados tratados | Contrato + guia rápido; finalidade e contato responsável por dados quando aplicável, sem publicar dados pessoais ou detalhes sigilosos |
| 3. Solicitar acesso | Informa justificativa, ambiente, owner consumidor e volumes; subscription/app registration quando o mecanismo exigir | Workflow com API/Product owner e consumer owner; autorização de uso de dados e consulta jurídica quando necessária |
| 4. Aprovar | Autoatendimento somente para acesso elegível; aprovação governada conforme exposição, dados, setor e risco | A único da atividade (§11.3), consultas e registro nominal; não atribuir toda aprovação ao DPO |
| 5. Testar em sandbox | Usa endpoint, identidade e fixtures sintéticas do protocolo; erros não expõem PII | Sandbox isolado, testes positivos/negativos e suporte de onboarding |
| 6. Promover para produção | Cumpre contrato, identidade, limites, plantão consumidor e plano de recuperação/janela | Gates §9.2–9.3, provas DEV/UAT do mesmo artefato, aprovação aplicável e contato operacional |
| 7. Acompanhar quota/consumo | Consumidor vê consumo, 429, latência, erros e orçamento; recebe alertas antes de atingir a quota | Workbook/portal por subscription; relatório mensal dos maiores consumidores. |
| 8. Suporte e mudança | Canais claros para incidente, dúvida, aumento de quota, deprecação e breaking change | SLA de suporte, comunicação aos 90, 30 e 7 dias antes da retirada, headers `Deprecation`/`Sunset`. |

#### 11.4.2 Comunicação de sunset / deprecação a consumidores

Deprecação anuncia que uma versão não deve receber novas integrações; retirada (sunset) encerra seu atendimento. A comunicação identifica as duas datas, o contrato substituto, impactos, plano de migração e responsável. As RFCs especificam headers, não uma janela universal de suporte [70][71].

| Canal | Quando | Conteúdo mínimo |
|---|---|---|
| **E-mail ao subscriber/consumer owner** | Aviso inicial pelo menos 90 dias antes da retirada; lembretes 30 e 7 dias antes | API/versão, datas de deprecação e retirada, nova versão, impactos e suporte |
| **Banner no Developer Portal** | Desde o aviso inicial até a retirada | Contrato deprecado, prazo, nova versão e migração |
| **Headers HTTP** | Nas respostas HTTP da versão deprecada durante a janela | `Deprecation` conforme RFC 9745, com structured date (`@` seguido de segundos Unix); `Sunset` conforme RFC 8594, com HTTP-date em GMT [70][71] |
| **Webhook opcional para parceiros B2B** | Anúncio, lembretes e execução da retirada | Eventos de deprecação/retirada, payload autenticado e contrato de verificação pelo parceiro |
| **Status page corporativa** | Na janela de retirada e em incidentes relacionados | Versão afetada, estado e contato |
| **Tickets proativos para maiores consumidores** | Pelo menos 60 dias antes; escalonar ausência de confirmação até 30 dias antes | Plano de migração, prazo e consumer owner |

A política corporativa proposta conserva aviso inicial de pelo menos 90 dias e versão anterior em paralelo, com banner/headers disponíveis durante a janela e confirmação de ausência de tráfego por 14 dias consecutivos antes da retirada. Ausência de logs ou consumidores não identificados não prova ausência de uso. Headers se aplicam a respostas HTTP; outros protocolos exigem comunicação equivalente no contrato e nos canais.

Uma major interna de componente não encurta esse compromisso (§9.5.3). Contenção urgente de segurança exige avaliação de impacto, comunicação imediata e plano de continuidade/recuperação; alteração de prazo interno só com exceção válida (§11.5), preservando obrigações legais e contratuais não dispensáveis.

### 11.5 Modelo de exceção (ciclo de vida formal)

Uma exceção pode afastar temporariamente um padrão interno dispensável, sob autoridade delegada. Não pode afastar obrigação legal, determinação de autoridade ou compromisso externo que o aprovador interno não tenha competência para alterar. Postergar mTLS, por exemplo, só é elegível se não descumprir o perfil obrigatório do fluxo; aprovação interna não torna conforme uma implementação que o descumpra.

#### 11.5.1 Formulário e preflight de elegibilidade

Toda solicitação de desvio registra:

- ID, requisitante nominal, data e responsável pela remediação/substituto.
- Regra exata, origem (lei, regulador, contrato ou política interna), versão e evidência de que a dispensa é permitida. Jurídico/compliance confirmam impedimentos externos; não cadastrar obrigação não dispensável como “exceção regulatória”.
- Escopo fechado: APIs/operações, dados/finalidade, consumidor, ambientes, regiões e controles afetados. Permissão para DEV ou uma API não autoriza PROD ou outras APIs.
- Justificativa, impacto/probabilidade, alternativas avaliadas e mitigações compensatórias verificáveis, com teste e responsável.
- Um aprovador competente, registrado por tipo (§11.5.3), consultas obrigatórias e aprovação autenticada vinculada ao escopo.
- Início e expiração com data/hora/fuso, janela de revisão, critérios de saída, condições de suspensão e plano de contenção/continuidade.
- Autoridade para renovação, provas exigidas e novo registro/validade; renovação não é tácita. Data owner e consumer owner registram o impacto quando há dados/consumo.

#### 11.5.2 Ciclo de vida e efeito operacional

```text
solicitada → elegibilidade verificada → análise → aprovada com escopo e validade
          → ativa → revisão antes de expirar
          → encerrada | renovada por nova decisão | expirada/suspensa e escalada
```

O gate consulta o inventário autorizado em cada promoção: só aceita registro aprovado, vigente, não suspenso, com escopo/regra compatíveis e mitigações comprovadas. Registro ausente, vencido, revogado ou sem evidência bloqueia novas publicações e o aceite da mudança.

Na expiração, notificar requisitante, A e plantão; executar o plano de contenção aprovado, que pode restringir acesso/exposição, preservar o estado conhecido ou suspender apenas a operação afetada. O comandante decide a continuidade controlada e escalona com dados/segurança/jurídico quando necessário. Não desligar indiscriminadamente todos os serviços, nem continuar com o desvio por prazo indefinido. Suspensão ou continuidade não dispensam obrigação externa.

#### 11.5.3 Aprovador único por tipo de padrão interno

| Desvio elegível | A único | Consultados obrigatórios |
|---|---|---|
| Operação/configuração menor | Platform owner APIM | API/Product owner |
| Padrão técnico interno | Responsável de arquitetura designado | Tech lead e plataforma |
| Segurança interna dispensável | Autoridade de segurança designada | Plataforma, API owner e dados; jurídico/compliance/DPO quando aplicável |
| Dados/residência/retenção de política interna dispensável | Data owner formalmente delegado | Jurídico/compliance/DPO, segurança e plataforma |
| Obrigação externa não dispensável | Não elegível a este processo | Representante da entidade obrigada, jurídico/compliance e responsáveis técnicos definem remediação/impedimento |

O inventário mantém apenas um A nominal por decisão. Mudança contratual ou autorização de órgão externo segue instrumento próprio e prova da autoridade competente; não é substituída pela ficha de exceção.

#### 11.5.4 Inventário

Inventário controlado em sistema de tickets (Jira/Azure Boards), com leitura para auditoria e acesso por necessidade. O API Center pode expor indicadores agregados e status aprovado, não vulnerabilidades, justificativas sigilosas ou dados pessoais. O gate usa o registro autorizado e sua versão, não uma planilha/exportação desatualizada.

#### 11.5.5 Métricas

- Total de exceções ativas.
- Exceções vencidas (indicador-chave de desempenho (KPI) principal; meta: zero).
- Tempo médio de exceção (dias).
- Exceções renovadas (>1 ciclo).
- Backlog de exceções por classe e por owner.

### 11.6 Autoatendimento e mudanças sujeitas a aprovação

O caminho depende do tipo de mudança e dos requisitos compostos da API. “Self-service” significa execução dentro de controles pré-aprovados; não dispensa CI, evidências ou responsabilidade final.

| Tipo de mudança | Caminho | A único | Gate mínimo |
|---|---|---|---|
| Revision compatível DEV/UAT | Autoatendimento | API/Product owner | CI e testes do ambiente |
| Revision compatível PROD não crítica | Autoatendimento controlado | API/Product owner | CI, CODEOWNERS aplicável, provas DEV/UAT do mesmo pacote e plano efetivo PROD |
| Versão incompatível | Autoatendimento com migração ou governado conforme risco | API/Product owner | Contrato, comunicação, coexistência e recuperação |
| Controle obrigatório de identidade | Governado | Security owner | Análise de ameaças, casos negativos e consultas de dados/regulação aplicáveis |
| Novo produto no catálogo | Autoatendimento se elegível | API/Product owner | Intake e controles de acesso/consumo |
| Nova API com obrigação setorial | Governado | API/Product owner | Aplicabilidade e dados aprovados, controles do fluxo, ADR e consultas de segurança/jurídico/compliance |
| Instância dedicada | Governado | Responsável de arquitetura designado | §4.5, capacidade operacional, continuidade e ADR |
| Região APIM/telemetria | Governado | Data owner | Decisão de residência/transferência, disponibilidade e consultas jurídica/plataforma (§10.2.3) |
| Exceção interna dispensável | Governado | A do tipo em §11.5.3 | Elegibilidade, escopo, mitigações e validade verificados em cada promoção |
| Mudança de tier | Governado | Platform owner APIM | Capacidade, limites de migração, carga, recuperação e ADR |
| Autorização final PROD crítica/regulada | Governado com segregação de funções (SoD) | Aprovador de release designado e segregado | Todos os gates anteriores, plano PROD, SoD e aprovação vinculados ao pacote/target [50][51][52] |

#### 11.6.1 Segregação de funções (SoD) para PROD crítica/regulada

O registro da mudança identifica os autores reais do PR e das alterações promovidas, contribuidores relevantes, iniciadores humanos do deployment, executor técnico e aprovador. Não presumir que autor original do PR, pessoa que fez merge, `github.actor` e solicitante da implantação sejam a mesma pessoa. Identidade desconhecida ou mapeamento não demonstrado bloqueia o gate.

- **PR e branches**: CODEOWNERS por contratos/policies/IaC, scripts, regras e workflows, protegendo o próprio CODEOWNERS. Branch protection/ruleset exige PR, checks de origem confiável e aprovação no conteúdo atual, invalida reviews obsoletos, impede force-push/delete e controla bypass administrativo. Múltiplos owners em um padrão também são OR, não aprovação de todos [50][51].
- **Environment `production`**: required reviewers libera o job com **uma** aprovação da lista. Prevent self-review impede o iniciador do deployment de aprová-lo; sozinho não demonstra segregação em relação aos autores originais. O A final é escolhido no registro, e o gate verifica que não pertence ao conjunto de autores da mudança nem de iniciadores humanos [52].
- **Aprovações cumulativas, se exigidas**: manter no sistema de mudança protegido aprovações autenticadas de plataforma, API owner e segurança/compliance aplicável, cada qual por pessoa distinta e fora do conjunto de autores/iniciadores. Vincular papéis, identidades, decisão, hash do pacote, parâmetros, target, digest do plano e validade. Um gate bloqueante integrado por GitHub App/protection rule, quando disponível, ou por broker externo de publicação, valida todas as aprovações e libera a identidade de escrita somente depois. Uma lista de required reviewers ou uma quantidade de reviews não implementa esse AND de papéis.
- **Preflight de habilitação**: registrar produto/plano GitHub e visibilidade do repositório, capacidades de required reviewers/prevent self-review/protection rules, restrições de branch e bypass, fonte dos checks e teste dos caminhos proibidos. Em Free/Pro/Team, required reviewers de environments estão documentados apenas para repositórios públicos; não presumir controle no repositório privado. Sem recurso compatível comprovado, usar integração externa equivalente verificada ou bloquear PROD crítica; não converter ausência de controle em aceite manual informal [52].
- **Execução**: identidade federada/gerenciada de publicação com privilégios mínimos por ambiente e target; nenhum segredo no pacote. A autorização de escrita deve ser condicionada ao gate de mudança, não apenas à posse de credenciais no runner. Portal normalmente somente leitura; emergência usa acesso temporário autorizado, auditado e reconciliado.
- **Evidência**: PR/commit, autores e identidades autenticadas, reviews/checks, plano, autorização, exceções aplicáveis, operação Azure, hash bruto do pacote e provas de saúde. Mudança de pacote, parâmetros, plano/target, validade ou aprovações invalida a autorização; não reutilizar uma aprovação de outro release.

#### 11.6.2 Métricas DX e governança

As métricas de experiência do desenvolvedor (DX) abaixo têm metas iniciais de política interna, sujeitas a baseline, capacidade e aprovação dos owners; não são benchmarks demonstrados. Registrar período mensal, fontes, exclusões e responsável pelo cálculo. Percentil 50 (P50) é a mediana; P90 cobre 90% dos casos.

| Métrica | Definição e denominador | Meta inicial proposta |
|---|---|---|
| Lead time de mudança | PR aberto até primeira implantação PROD aceita, por mudança identificada; estratificar por caminho/risco | P50 ≤ 2 dias elegíveis; P90 ≤ 10 dias governadas |
| Onboarding consumidor | Solicitação aprovada até primeiro uso funcional aceito no protocolo, separado por sandbox/PROD | Sandbox ≤ 3 dias; PROD ≤ 10 dias |
| Taxa de falha de implantação | Implantações PROD que tentaram alterar o target e falharam, exigiram rollback/fix-forward ou causaram incidente em 24 h, divididas por todas as tentativas PROD do período | < 10%; críticas < 5%, após baseline |
| Satisfação do consumidor | Pesquisa trimestral; Net Promoter Score (NPS), com população e taxa de resposta publicadas | NPS ≥ 40 ou tendência positiva |
| Exceções | Ativas, vencidas, suspensas e renovadas por owner/risco; medir quantidade e idade | Vencidas = 0; remediação por plano |
| Adoção de self-service entre elegíveis | Mudanças PROD aceitas pelo caminho self-service / total de mudanças PROD aceitas classificadas previamente como elegíveis no período | ≥ 70% como objetivo interno de piloto |
| Mix de caminhos | Mudanças self-service / todas as mudanças PROD aceitas, e governadas por motivo/risco | Publicar proporções; sem piso único que reduza controles obrigatórios |
| Tempo de aprovação governada | Entrada completa até decisão, por autoridade; separar espera por informação e por aprovador | P90 ≤ 5 dias úteis, após baseline |

Uma mudança tem ID de release/mudança e pode agregar vários PRs; PRs fechados, documentação sem publicação e execuções repetidas não são automaticamente novas mudanças aceitas. Não misturar esse denominador com a taxa de tentativas de implantação. Se não houver elegíveis no mês, registrar “não aplicável”, não zero. Queda do índice exige examinar mix de risco, lead time, espera e causas observadas; não prova gargalo nem autoriza retirar gates para atingir 70%. Esses diagnósticos orientam as práticas de §12.

---

## 12. Práticas inadequadas e boas práticas

### 12.1 Lista detalhada

#### 1) APIM por aplicação sem critério
A criação de instâncias APIM por cada squad, sem critérios de adoção, aumenta custos e fragmenta o catálogo e a governança. Correção: gate via §4.5; instância dedicada só por exceção justificada.

#### 2) APIM apenas como proxy
Usar APIM apenas para encaminhar chamadas, sem policies, versionamento e catálogo, deixa de aplicar os controles de governança previstos. Correção: padrões mínimos (§6, §7) obrigatórios.

#### 3) Centralização sem segmentação
Concentrar todas as APIs em um produto dificulta regras distintas de consumo e responsabilidades. Correção: produtos segmentam acesso, não isolam runtime. Avaliar workspaces no gateway padrão dos tiers v2 ou com workspace gateway, conforme suporte e região, ou instâncias separadas quando o alcance de falha/administração exigir (§3, §4 e §11.2); Premium clássico não oferece a opção de workspace no gateway padrão [5].

#### 4) Misturar exposições e dados sem estratégia
Não registrar exposição, dados, setor, consumidor e criticidade impede derivar controles adequados. Correção: aplicar a composição de requisitos de §7.7; dados pessoais/cartões não impõem FAPI universal. Instância dedicada depende de isolamento, capacidade e operação comprovados, não apenas do rótulo “sensível”.

#### 5) Sem responsável definido
A API não tem responsáveis identificados para operação e decisões. Correção: §11.2 obrigatório.

#### 6) Publicação manual em produção
Alterações diretas no portal podem gerar drift e perder a trilha de release. Correção: APIOps (§9), portal normalmente somente leitura em PROD; emergência com acesso temporário, autorização, evidência e reconciliação por PR, sem ocultar o desvio.

#### 7) Sem versionamento
Mudanças incompatíveis afetam consumidores sem alternativa. Correção: contrato e versionamento de §6.4, coexistência e política corporativa de comunicação/retirada de §11.4.2, incluindo o tratamento controlado de urgências; não impor a major mais nova de componente aos consumidores apenas pelo prazo interno.

#### 8) Lógica de negócio demais nas policies
Implementar lógica de negócio nas policies acopla a aplicação ao gateway. Correção: policies só executam funções transversais (auth, throttling, transformação leve, correlação, validação de schema); a lógica de negócio fica no backend.

#### 9) Sem observabilidade
A ausência de telemetria impede diagnosticar a latência e outras falhas. Correção: §10.1 obrigatório.

#### 10) Subscription keys como segurança principal
Uma chave estática usada como única identidade de uma API protegida não demonstra quem chama nem autorização adequada. Correção: mecanismo de identidade/autorização conforme consumidor, exposição, dados e perfil aplicável (§7.7); mTLS/FAPI só quando exigido. API pública verdadeiramente anônima precisa de decisão explícita, limites e abuso controlado, não de autenticação fictícia.

#### 11) Expor backends diretamente, contornando o APIM
Bypass via rede pública ao backend. Correção: §7.6 + restrição de rede (Private Link, allowlist).

#### 12) Não separar ambientes
DEV/UAT/PROD na mesma instância. Correção: instâncias separadas, IaC parametrizado.

#### 13) Escolha tardia de SKU/região
Adiar requisitos de rede, gateway, continuidade e região pode comprometer uma arquitetura inviável. Correção: registrar as dimensões de §1.1 e §4.5 antes de comprometer tier/topologia, com RTO/RPO por objeto e suporte regional atual (§14). Premium v2 suporta VNet injection onde disponível; residência no Brasil ou autonomia de domínio, isoladamente, não impõem Premium clássico nem instância dedicada.

#### 14) Governança central sem autoatendimento elegível
Exigir aprovação central até para mudanças padronizadas de baixo risco aumenta espera. Correção: §11.6 com caminhos elegíveis pré-aprovados e governados por risco; medir a adoção entre elegíveis e o mix. A meta interna de 70% não permite retirar controles obrigatórios.

### 12.2 Matriz resumida

| Prática inadequada | Sintoma | Correção | Onde |
|---|---|---|---|
| Proxy puro | Sem policies, sem catálogo | Padrões §6, §7 | §12.1 #2 |
| Sem responsável definido | Incidente ou decisão sem pessoa/substituto para escalonar | Registrar API owner, plataforma, dados, plantão e A da atividade | §11.2–11.3 |
| Publicação manual | Estado divergente do último release aceito | APIOps, autorização de emergência e reconciliação rastreável | §9 |
| Backend exposto | Bypass do APIM | Restrição rede | §7.6 |
| Tier/região tardios | Rede ou continuidade inviáveis após o compromisso | Requisitos compostos, objetivos por objeto e disponibilidade antes do desenho | §1.1, §4.5, §14 |
| Baixa adoção de self-service elegível | Espera observada em mudanças padronizadas, após separar o mix de risco | Diagnosticar causas e automatizar o caminho elegível sem remover gates | §11.6 |

---

## 13. Recomendações finais e plano de implantação sugerido

### 13.1 Recomendações finais

1. Antes do desenho, registrar **exposição, dados legais/corporativos, setor, consumidor e criticidade**, além de rede, operação e objetivos de recuperação por objeto (§1.1 e §14). As perguntas orientam a análise; não são condições automáticas para escolher um tier.
2. Comparar **adoção centralizada (A), distribuída (B) e híbrida governada (C)** (§4). O modelo C é candidato quando compartilhamento e domínios dedicados se justificam juntos; autonomia/SLO próprio não exige, sozinho, instância dedicada. Produtos, workspace, gateway e instância têm limites distintos; equipe/plantão e alcance de falha entram no aceite.
3. Decidir **tier, gateway, rede virtual (VNet) e região separadamente**. Data-base documental: **09/10/2026**. Premium v2 suporta VNet injection onde disponível; Brazil South lista Basic v2/Standard v2, não Premium v2/workspace gateway [3]. Residência Brasil não impõe Premium clássico. Multi-region nativo ou self-hosted podem justificar Premium clássico quando exigidos e disponíveis; confirmar capacidade/região antes do compromisso e revalidar a ADR.
4. Fazer o gate aplicar **requisitos compostos** (§7.7): identidade por fluxo/consumidor, autenticação TLS mútua (mTLS) e perfil Financial-grade API (FAPI) quando aplicáveis, uso/classificação de dados e obrigações setoriais. Dados sensíveis conforme a Lei Geral de Proteção de Dados Pessoais (LGPD), padrão de cartões PCI DSS e participação Open Finance não são a mesma categoria. Relatório de Impacto à Proteção de Dados Pessoais (RIPD) depende de determinação aplicável, alto risco ou política interna explícita; não é universal por endpoint [66][76][79].
5. **APIOps/IaC com promoção rastreável** (§9): mesmo pacote selado em DEV/UAT/PROD, planos/overrides reais de PROD, autorização segregada e recuperação. Scripts e integrações ausentes bloqueiam a habilitação; portal normalmente somente leitura, com emergência controlada.
6. Usar **API Center como catálogo** [10] somente com destinos/acessos aprovados. Residency/transferência abrange também especificações, metadados e coletas de serviços associados, não apenas o gateway; alternativa de catálogo é necessária se o destino não atender ao requisito.
7. Manter **análise de ameaças por fluxo** (§7.6), revisão por mudança relevante e periodicidade acordada. Defender for APIs exige elegibilidade, onboarding e consideração de sua coleta independente; habilitar o plano não prova cobertura de todas as APIs.
8. **Gestão financeira da nuvem (FinOps)** (§8.6): showback, demonstrativo de custos por domínio, com pool/instância/região e período definidos, denominadores coerentes, custos diretos/não alocados e reconciliação da soma. Chargeback, cobrança interna, só após aceite de FinOps/owners; pesos de alocação são convenção, não preço ou consumo medido.
9. **Self-service** (§11.6): objetivo interno de 70% entre mudanças elegíveis, com período/denominador definidos e mix governado publicado. Resultado inferior não prova causa nem autoriza retirar controles.
10. **Exceções válidas antes da promoção** (§11.5): autoridade, regra dispensável, escopo, evidência, mitigações e expiração. Obrigação externa não é dispensada por aceite interno; expiração bloqueia nova publicação e aciona contenção controlada.
11. **Recuperação de desastre (DR)** (§10.5): fixar objetos, dependências e metas de tempo/ponto de recuperação (RTO/RPO) antes de comprometer tier/topologia. Ensaio representativo comprova a retomada acordada, incluindo backend/dados quando abrangidos; tier ou multi-region não garantem essas metas.
12. **Runbooks e aprovação anteriores ao go-live** (§10.6 e §13.2): pré-requisitos, execução, verificação, aborto, recuperação, owner/substituto e evidência. Incluir incidente de API, segurança e dados pessoais com controlador, ANPD e titulares, nos prazos aplicáveis [27]; não restringir o aceite a cinco documentos. Retenção é decidida por conjunto/finalidade/norma, não por um número universal de logs.

### 13.2 Plano de implantação sugerido, dependências e aceite

Os horizontes são estimativas de planejamento, não autorização de publicação. Cada entrega tem um A nominal, executores e consultas. Se o go-live anteceder a data sugerida, antecipar os pré-requisitos; não publicar para depois implantar seus controles.

| Horizonte/marco | Entrega | A único; executores/consultados | Dependência e critério de aceite |
|---|---|---|---|
| **0–30 dias; antes de comprometer arquitetura** | Requisitos compostos, residência/transferências, rede e continuidade por objeto; ADR de adoção/tier/topologia | Responsável de arquitetura; plataforma, API owner, dados, continuidade, segurança, jurídico/compliance/DPO e FinOps | Objetos e dependências com RTO/RPO, restrições e fonte datada de suporte; A de dados aprova seus usos/destinos. ADR tem decisor único, consultas e alternativas; premissa faltante bloqueia compromisso |
| **0–30 dias** | Inventário de APIs, contratos/protocolos, produtores, consumidores e dados | API/Product owner de cada domínio; squads e plataforma | Fichas completas com exposição, dados, setor, consumidor, criticidade e obrigações; catálogo em destino aprovado, inclusive metadados |
| **30–60 dias; inicialmente DEV/UAT** | Provisionamento IaC e guardrails selecionados | Platform owner APIM; plataforma, segurança e dados consultados | Template/parametrização versionados, mudanças esperadas revisadas no what-if/plan e testes reais; preview não é smoke. Defender apenas em escopo elegível com onboarding comprovado. Provisionar não autoriza publicar PROD |
| **Antes da primeira API PROD** | Pipeline, contratos dos scripts, baselines, planos e autorização | Platform owner APIM; DevEx e aprovador de release segregado | Pacote único validado DEV/UAT, plano efetivo PROD, checks bloqueantes, baseline/layout qualificados, artefato/evidência presentes, plano/visibilidade GitHub e SoD comprovados (§11.6.1); teste de bloqueio para falso, ausente, vencido ou divergente |
| **Antes da primeira API PROD** | Procedimento de exceções e análise de ameaças | Security owner; arquitetura, dados, plataforma e jurídico/compliance | Aplicabilidade e gates coerentes; inventário autorizado, aprovadores por tipo, expiração/mitigações executáveis. Sem exceção válida, requisito interno dispensável não é ignorado; obrigação externa nunca é afastada por aceite interno |
| **Antes do primeiro PROD aplicável** | Runbooks de API, plataforma, segurança, incidente de dados e recuperação | API/Product owner para API; cada atividade especializada tem o A de §11.3 | Owner/substituto/plantão, pré-requisitos, sequência executável, sucesso/aborto, recuperação e evidência. Cobrir todos os cenários aplicáveis de §10.6, não só cinco runbooks. Incidente pessoal identifica controlador/representante, substituto, canais, ANPD e titulares: avaliação de risco/dano relevante e comunicação em três dias úteis do conhecimento de que afetou dados pessoais quando aplicável, ressalvada legislação específica; comunicação inicial, complementação fundamentada em até vinte dias úteis nos termos do regulamento e registro do incidente inclusive não comunicado por pelo menos cinco anos da data do registro, não de toda telemetria [27][105] |
| **Antes do go-live crítico/alta criticidade; ensaios periódicos depois** | Prova da continuidade acordada | Responsável de continuidade/DR; plantões, produtores, plataforma e dados | Ensaio representativo dos objetos e dependências, RTO/RPO medidos, failover/failback e falhas de recuperação; limites/ações corretivas. Metas não demonstradas bloqueiam o aceite ou exigem aceite de risco interno permitido e formal, nunca dispensa de obrigação externa |
| **90–120 dias; antes de chargeback** | Showback e eventual piloto de alocação | FinOps lead; plataforma e owners | Telemetria estável, pools/períodos/denominadores, ausência de tráfego e não alocados definidos; soma reconciliada ao custo do pool antes de cobrar domínios |
| **180–240 dias, após piloto** | Adoção de self-service elegível | DevEx lead; plataforma e API owners | Publicar unidade/período, elegíveis, mix de risco, lead time e causas. Meta proposta ≥70% entre elegíveis; não medir PRs como se fossem mudanças implantadas |
| **Contínuo** | Revalidação de capacidades, dados, ameaças e recuperação | A da decisão original | Revisão trimestral de tier/região e revisão de ameaças/DR na periodicidade aprovada ou após mudança relevante; atualizar ADR/contratos e repetir as provas afetadas |

Para cada publicação, o checklist integrado reúne decisões de dados, contrato/segurança, provas DEV/UAT, plano PROD, SoD, exceções, runbooks e recuperação. Qualquer gate falso ou inconclusivo bloqueia o aceite; armazenar evidência não substitui cumprir esses requisitos.

---

## 14. Decisões pendentes que dependem do cliente

As decisões são priorizadas pelo marco que bloqueiam. Cada registro identifica um A nominal, prazo, substituto, consultados e ADR/ticket. Preparar em paralelo não permite ultrapassar o marco sem aceite.

| Marco | Decisão pendente | A sugerido; consultas | Gate / critério de aceite |
|---|---|---|---|
| **Antes do compromisso de arquitetura** | Exposição, dados legais/corporativos, setor, consumidores e criticidade das APIs | API/Product owner; dados, segurança e jurídico/compliance | Ficha de intake com dimensões compostas e normas aplicáveis. Data owner decide classificação/uso sob delegação; representante da entidade obrigada confirma obrigações. LGPD/PCI não implicam FAPI automaticamente |
| **Antes do compromisso de arquitetura** | Objetos, dependências e objetivos de continuidade | Responsável de continuidade/DR; plataforma, API owner e dados | RTO em unidade de tempo e RPO por estado/conjunto: gateway, configuração, identidade/certificados, auditoria, backend/transações e demais dependências incluídas. Escopo e fonte aprovados antes de tier/topologia; metas a comprovar, não garantia do SKU |
| **Antes do compromisso de arquitetura** | Residência/acesso/transferência: Brasil ou destinos específicos | Data owner; jurídico/compliance/DPO, segurança e plataforma | Finalidade, conjunto de dados, entidade, obrigação e destinos registrados, incluindo catálogo/metadados e coletores. Não escolher Premium clássico só por residência |
| **Antes do compromisso de arquitetura** | Modelo de adoção A/B/C e tier/gateway/rede/região | Responsável de arquitetura; plataforma, continuidade, dados, segurança e FinOps | ADR separa centralização/distribuição/híbrido de tier e configuração; avalia limites de isolamento, operação, capacidade e disponibilidade datada [3]. Autonomia de domínio não basta para instância dedicada |
| **Antes de habilitar coleta ou publicar API afetada** | Destinos de Log Analytics, Application Insights, Event Hubs, Sentinel, API Center e serviços com coleta própria | Data owner; plataforma, jurídico/compliance/DPO e segurança | ADR por conjunto/finalidade com regiões, acessos e mecanismo aplicável; sem destino autorizado, bloquear essa coleta/publicação ou escolher alternativa que cumpra o requisito |
| **Antes de habilitar coleta ou publicar API afetada** | Campos, minimização, retenção e WORM por conjunto | Data owner; jurídico/compliance/DPO, plataforma e auditoria | Contrato de §10.7 com prazo/marco, fonte normativa/contratual ou política interna fundamentada e conservação/eliminação. Sem piso universal de gateway logs; armazenamento imutável só quando justificado |
| **Antes da primeira API PROD** | Autoridades, elegibilidade, validade e contenção de exceções | Platform owner APIM para habilitar o processo; A por tipo em §11.5.3 | Nomes/delegações e integração do inventário; gate rejeita falso, fora de escopo, vencido ou obrigação não dispensável; expiração aciona plano controlado |
| **Antes da primeira publicação crítica/regulada** | SoD, autorizações e disponibilidade dos controles GitHub privados | Aprovador de release designado; DevEx, segurança e auditoria | Produto/plano e visibilidade comprovados, autores/iniciadores segregados, regras e checks ativos, aprovações OR/AND conforme política e integração bloqueante. Preparação pode ocorrer em paralelo; o go-live não |
| **Antes do primeiro PROD aplicável** | Contratos dos scripts e runbooks operacionais/normativos | Platform owner APIM para automação; A de cada procedimento em §11.3 | Entradas/saídas/exit codes e evidência qualificados; fluxo API/segurança/dados, controlador/substituto e relógio de três dias úteis quando aplicável [27]. Não aguardar investigação completa para iniciar comunicação devida |
| **Antes do go-live crítico/alta criticidade** | Prova de recuperação por objeto e dependência | Responsável de continuidade/DR; plantões e produtores | Ensaio representativo cumpre o contrato, incluindo aborto/falha de failover/failback. Déficit exige remediação ou aceite interno de risco permitido, sem afastar obrigação externa |
| **Em paralelo; antes de alocar/cobrar custos** | Owner FinOps e pools de showback/chargeback | FinOps lead; plataforma e API owners | Papel nominal, período/pool e denominadores, não alocados e reconciliação definidos; cobrança só após aceite |
| **Depois de requisitos atuais atendidos** | Migração futura de tier e novos destinos fora do desenho aprovado | Responsável de arquitetura; dados, segurança e plataforma | ADR datada e gatilhos de capacidade/região/rede/continuidade. Se o destino já integrar o desenho atual, sua disponibilidade/residência não pode esperar; revalidar [3] |
| **Após piloto; sem condicionar redução de controles** | Adoção self-service e métricas DX | DevEx lead; API owners e plataforma | Unidade de mudança PROD, elegíveis e mix coerentes com §11.6.2; nenhuma inferência causal só do índice de 70% |

---

## 15. Referências consolidadas

> Formato: **instituição; título; URL; data de acesso**. Todas as referências devem ser revalidadas em datas críticas (regulatório, SKU/região). Data desta edição: **2026-10-09**; datas de acesso indicadas por fonte.

### Nota contratual e responsabilidade compartilhada

O desenho APIM deve ser lido junto com os **Online Services Terms (OST)** e o **Microsoft Products and Services Data Protection Addendum (DPA)** aplicáveis ao contrato do cliente. Esses documentos não substituem a decisão arquitetural: no modelo de responsabilidade compartilhada, Microsoft opera a infraestrutura/plataforma do serviço em nuvem, enquanto o cliente continua responsável por dados, identidades, configurações, acesso, classificação, retenção, chaves, policies e conformidade do uso da API [45][46][47].

### Azure API Management: conceitos, tiers, regiões e recursos

[1] Microsoft Learn; Feature-based comparison of Azure API Management tiers; <https://learn.microsoft.com/en-us/azure/api-management/api-management-features>; Acesso em: 2026-10-09.

[2] Microsoft Learn; Azure API Management: v2 tiers; <https://learn.microsoft.com/en-us/azure/api-management/v2-service-tiers-overview>; Acesso em: 2026-10-09.

[3] Microsoft Learn; Azure API Management: availability of v2 tiers and workspace gateways; <https://learn.microsoft.com/en-us/azure/api-management/api-management-region-availability>; Acesso em: 2026-10-09.

[4] Microsoft Learn, documentação histórica; Migrate an Azure API Management instance to the stv2 platform; <https://learn.microsoft.com/en-us/previous-versions/azure/api-management/migrate-stv1-to-stv2>; Acesso em: 2026-10-09.

[5] Microsoft Learn; Workspaces in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/workspaces-overview>; Acesso em: 2026-10-09.

[6] Microsoft Learn; Capacity Metrics in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/api-management-capacity>; Acesso em: 2026-06-18.

[7] Microsoft Learn; Deploy an Azure API Management instance to multiple Azure regions; <https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-deploy-multi-region>; Acesso em: 2026-10-09.

[8] Microsoft Learn; Self-hosted gateway overview; <https://learn.microsoft.com/en-us/azure/api-management/self-hosted-gateway-overview>; Acesso em: 2026-06-18.

[9] Microsoft Learn; Backup and restore your Azure API Management instance for disaster recovery; <https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-disaster-recovery-backup-restore>; Acesso em: 2026-10-09.

[10] Microsoft Learn; What is Azure API Center?; <https://learn.microsoft.com/en-us/azure/api-center/overview>; Acesso em: 2026-10-09.

[11] Microsoft Learn; Azure API Management analytics dashboard retirement (March 2027); <https://learn.microsoft.com/en-us/azure/api-management/breaking-changes/analytics-dashboard-retirement-march-2027>; Acesso em: 2026-06-18.

[12] Microsoft Learn; AI gateway capabilities in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/genai-gateway-capabilities>; Acesso em: 2026-10-09.

### Rede, conectividade, resiliência e arquitetura

[13] Microsoft Azure / GitHub; Azure/apiops: toolkit legado, aviso de arquivamento e transição para APIOps CLI; <https://github.com/Azure/apiops>; Acesso em: 2026-10-09.

[14] Microsoft Learn; ARM template deployment what-if operation; <https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-what-if>; Acesso em: 2026-10-09.

[15] Microsoft Learn; Azure API Management with an Azure virtual network; <https://learn.microsoft.com/en-us/azure/api-management/virtual-network-concepts>; Acesso em: 2026-10-09.

### Policies, segurança, identidade e padrões externos

[16] Microsoft Learn; API Management policy reference: validate-azure-ad-token; <https://learn.microsoft.com/en-us/azure/api-management/validate-azure-ad-token-policy>; Acesso em: 2026-10-09.

[17] Microsoft Learn; Microsoft Defender for APIs; <https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-apis-introduction>; Acesso em: 2026-06-18.

[18] Microsoft Learn; Observability in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/observability>; Acesso em: 2026-10-09.

[19] Microsoft Learn; Secure APIs using client certificate authentication in API Management; <https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-mutual-certificates-for-clients>; Acesso em: 2026-10-09.

[20] OWASP Foundation; OWASP API Security Top 10 2023; <https://owasp.org/API-Security/editions/2023/en/0x00-header/>; Acesso em: 2026-06-18.

[21] IETF / RFC Editor; RFC 9700: Best Current Practice for OAuth 2.0 Security (2025); <https://www.rfc-editor.org/rfc/rfc9700.txt>; Acesso em: 2026-10-09.

[22] OpenID Foundation; Financial-grade API (FAPI) 1.0 Part 1; <https://openid.net/specs/openid-financial-api-part-1-1_0.html>; Acesso em: 2026-06-18.

[23] OpenID Foundation; FAPI 2.0 Security Profile; <https://openid.net/specs/fapi-security-profile-2_0.html>; Acesso em: 2026-06-18.

### Desenho de APIs, catálogo, produtos e experiência do desenvolvedor

[24] Microsoft Learn; Azure API Center key concepts / registry and metadata; <https://learn.microsoft.com/en-us/azure/api-center/key-concepts>; Acesso em: 2026-06-18.

[25] Microsoft Learn; Protect your APIs with Defender for APIs: onboarding, plans and billing; <https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-apis-deploy>; Acesso em: 2026-10-09.

### DevOps, APIOps, IaC e qualidade

[26] Microsoft Learn / Azure Architecture Center; Automate API Management configuration by using APIOps CLI; <https://learn.microsoft.com/en-us/azure/architecture/example-scenario/devops/automated-api-deployments-apiops>; Acesso em: 2026-10-09.

### Regulatório (Brasil)

[27] ANPD / Biblioteca Digital do Ministério da Justiça; Resolução CD/ANPD nº 15, de 24 de abril de 2024: Regulamento de Comunicação de Incidente de Segurança; <https://bibliotecadigital.mj.gov.br/bitstream/1/12879/2/RES_ANPD_2024_15.html>; Acesso em: 2026-10-09.

[28] Banco Central do Brasil; Resolução BCB nº 32/2020: requisitos técnicos e procedimentos operacionais do Open Finance, versão vigente 6; <https://www.bcb.gov.br/api/conteudo/app/normativos/exibeversao?p1=51200&p2=6&p3=Vigente>; Acesso em: 2026-10-09.

[28b] Banco Central do Brasil; Resolução BCB nº 85/2021: segurança cibernética e contratação de processamento, armazenamento e nuvem pelas entidades do art. 1º, versão vigente 4; <https://www.bcb.gov.br/api/conteudo/app/normativos/exibeversao?p1=51353&p2=4&p3=Vigente>; Acesso em: 2026-10-09.

[28c] Conselho Monetário Nacional / Banco Central do Brasil; Resolução CMN nº 4.893/2021: segurança cibernética e contratação de processamento, armazenamento e nuvem, versão vigente 3; <https://www.bcb.gov.br/api/conteudo/app/normativos/exibeversao?p1=51323&p2=3&p3=Vigente>; Acesso em: 2026-10-09.

[29] Open Finance Brasil; Portal oficial; <https://openfinancebrasil.org.br/>; Acesso em: 2026-06-18.

[29b] Open Finance Brasil; Padrões Técnicos / Segurança; <https://openfinancebrasil.atlassian.net/wiki/spaces/OF/pages/17378602/Padr+es+T+cnicos>; Acesso em: 2026-06-18.

> **Nota sobre URLs regulatórias:** as URLs listadas foram a referência ao tempo desta consolidação. Atos normativos podem ser republicados; **antes de uso em auditoria, a equipe de compliance confirma o ato vigente** (número, ano, redação) na fonte oficial.

### Operações, observabilidade, retenção e monitoramento

[30] Microsoft Azure; Pricing Calculator (APIM, Log Analytics, Defender for APIs, Front Door); <https://azure.microsoft.com/pricing/calculator/>; Acesso em: 2026-06-18.

[31] Microsoft Learn; Tutorial: Monitor APIs in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor>; Acesso em: 2026-10-09.

[32] Microsoft Learn; Integrate Azure API Management with Azure Application Insights; <https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights>; Acesso em: 2026-06-18.

[36] Microsoft Learn; Monitoring data reference for Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/monitor-api-management-reference>; Acesso em: 2026-10-09.

[37] Microsoft Learn; Diagnostic settings in Azure Monitor; <https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings>; Acesso em: 2026-06-18.

[38] Microsoft Learn; Manage data retention in a Log Analytics workspace; <https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure>; Acesso em: 2026-06-18.

[39] Microsoft Learn; Immutable storage for blob data; <https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview>; Acesso em: 2026-06-18.

[40] Microsoft Learn; Design a Log Analytics workspace architecture; <https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design>; Acesso em: 2026-06-18.

[41] Microsoft Learn; Reliability in Azure API Center; <https://learn.microsoft.com/en-us/azure/reliability/reliability-api-center>; Acesso em: 2026-10-09.

[42] Microsoft Learn; Onboard to Microsoft Sentinel; <https://learn.microsoft.com/en-us/azure/sentinel/quickstart-onboard>; Acesso em: 2026-06-18.

[43] Microsoft Learn; Application Insights FAQ: data collection, retention, storage, and privacy; <https://learn.microsoft.com/en-us/azure/azure-monitor/app/application-insights-faq#how-does-application-insights-handle-data-collection--retention--storage--and-privacy>; Acesso em: 2026-06-18.

[44] Microsoft Learn; Geo-disaster recovery for Azure Event Hubs; <https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-geo-dr>; Acesso em: 2026-06-18.

[53] Microsoft Learn; Azure Workbooks overview; <https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview>; Acesso em: 2026-06-18.

### Landing zone, governança e Azure Policy

[33] Microsoft Learn; Azure landing zones design areas; <https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas>; Acesso em: 2026-06-18.

[34] Microsoft Learn; API Management policy reference; <https://learn.microsoft.com/en-us/azure/api-management/policy-reference>; Acesso em: 2026-06-18.

[35] Microsoft Learn; Azure Policy guest configuration and remediation; <https://learn.microsoft.com/en-us/azure/governance/policy/concepts/guest-configuration>; Acesso em: 2026-06-18.

### Contratos, responsabilidade compartilhada e frameworks de controle

[45] Microsoft Learn; Shared responsibility in the cloud; <https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility>; Acesso em: 2026-06-18.

[46] Microsoft Licensing; Online Services Terms (OST); <https://www.microsoft.com/licensing/docs/view/Online-Services-Terms>; Acesso em: 2026-06-18.

[47] Microsoft Licensing; Microsoft Products and Services Data Protection Addendum (DPA); <https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA>; Acesso em: 2026-06-18.

[48] ISACA; The Framework to Manage Frameworks: COBIT and the Governance Gap (2026); orientação pública, não texto integral dos objetivos COBIT 2019; <https://www.isaca.org/resources/news-and-trends/isaca-now-blog/2026/the-framework-to-manage-frameworks-cobit-and-the-governance-gap>; Acesso em: 2026-10-09.

[49] COSO; The 2013 COSO Framework & SOX Compliance: One Approach to an Effective Transition (2013); <https://www.coso.org/_files/ugd/3059fc_c98a93b420a34d28a4c79f57db0d2c93.pdf>; Acesso em: 2026-10-09.

[50] GitHub Docs; About code owners; <https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners>; Acesso em: 2026-10-09.

[51] GitHub Docs; About protected branches; <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches>; Acesso em: 2026-10-09.

[52] GitHub Docs; Deployments and environments; <https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments>; Acesso em: 2026-10-09.


### Fontes da atualização corretiva

[109] Microsoft Learn; Microsoft.Insights/metricAlerts 2018-03-01: campo severity; <https://learn.microsoft.com/en-us/azure/templates/microsoft.insights/2018-03-01/metricalerts>; Acesso em: 2026-10-09.

[110] Agência Nacional de Proteção de Dados; Institucional: denominação e Lei nº 15.352/2026; <https://www.gov.br/anpd/pt-br/acesso-a-informacao/institucional>; Acesso em: 2026-10-09.

[54] Microsoft Learn; API gateway overview and feature comparison; <https://learn.microsoft.com/en-us/azure/api-management/api-management-gateways-overview>; Acesso em: 2026-10-09.

[55] Microsoft Learn; Restrictions and details of API formats support; <https://learn.microsoft.com/en-us/azure/api-management/api-management-api-import-restrictions>; Acesso em: 2026-10-09.

[56] Microsoft Learn; Reliability in Azure API Management; <https://learn.microsoft.com/en-us/azure/reliability/reliability-api-management>; Acesso em: 2026-10-09.

[57] Microsoft Learn; Enable availability zones for Azure API Management instances; <https://learn.microsoft.com/en-us/azure/api-management/enable-availability-zone-support>; Acesso em: 2026-10-09.

[58] Microsoft Learn; Support and prerequisites for deploying the Defender for APIs plan; <https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-apis-prepare>; Acesso em: 2026-10-09.

[59] Microsoft Learn; How to use role-based access control in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/api-management-role-based-access-control>; Acesso em: 2026-10-09.

[60] Microsoft Learn; Azure Policy definitions: Modify effect; <https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-modify>; Acesso em: 2026-10-09.

[61] Microsoft Learn; API Management policy reference: llm-semantic-cache-lookup; <https://learn.microsoft.com/en-us/azure/api-management/llm-semantic-cache-lookup-policy>; Acesso em: 2026-10-09.

[62] OpenAPI Initiative; OpenAPI Specification 3.0.3; <https://spec.openapis.org/oas/v3.0.3>; Acesso em: 2026-10-09.

[63] Microsoft Learn; Supported metrics for Microsoft.ApiManagement/gateways; <https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-apimanagement-gateways-metrics>; Acesso em: 2026-10-09.

[64] IETF / RFC Editor; RFC 8705: OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens; <https://www.rfc-editor.org/rfc/rfc8705.txt>; Acesso em: 2026-10-09.

[65] IETF / RFC Editor; RFC 6749: The OAuth 2.0 Authorization Framework, sections 1.3.1 and 4.4; <https://www.rfc-editor.org/rfc/rfc6749.html>; Acesso em: 2026-10-09.

[66] Câmara dos Deputados; Lei nº 13.709/2018: Lei Geral de Proteção de Dados Pessoais, texto atualizado; <https://www2.camara.leg.br/legin/fed/lei/2018/lei-13709-14-agosto-2018-787077-normaatualizada-pl.html>; Acesso em: 2026-10-09.

[67] ANPD; Resolução CD/ANPD nº 19/2024: transferência internacional de dados e cláusulas-padrão; <https://www.gov.br/anpd/pt-br/acesso-a-informacao/institucional/atos-normativos/regulamentacoes_anpd/resolucao-cd-anpd-no-19-de-23-de-agosto-de-2024>; Acesso em: 2026-10-09.

[68] NIST; CSWP 29: The NIST Cybersecurity Framework 2.0; <https://nvlpubs.nist.gov/nistpubs/CSWP/NIST.CSWP.29.pdf>; Acesso em: 2026-10-09.

[69] Microsoft Learn; Azure Front Door overview; <https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview>; Acesso em: 2026-10-09.

[70] IETF / RFC Editor; RFC 9745: The Deprecation HTTP Response Header Field; <https://www.rfc-editor.org/rfc/rfc9745.html>; Acesso em: 2026-10-09.

[71] IETF / RFC Editor; RFC 8594: The Sunset HTTP Header Field; <https://www.rfc-editor.org/rfc/rfc8594.html>; Acesso em: 2026-10-09.

[72] Microsoft Learn; Common questions about Defender for APIs; <https://learn.microsoft.com/en-us/azure/defender-for-cloud/faq-defender-for-apis>; Acesso em: 2026-10-09.

[73] Microsoft Learn; Support for GraphQL APIs in Azure API Management; <https://learn.microsoft.com/en-us/azure/api-management/graphql-apis-overview>; Acesso em: 2026-10-09.

[74] Microsoft Learn; API Management policy reference: validate-graphql-request; <https://learn.microsoft.com/en-us/azure/api-management/validate-graphql-request-policy>; Acesso em: 2026-10-09.


[76] ANPD; Relatório de Impacto à Proteção de Dados Pessoais (RIPD): perguntas e respostas, atualização de 23/09/2026; <https://www.gov.br/anpd/pt-br/canais_atendimento/agente-de-tratamento/relatorio-de-impacto-a-protecao-de-dados-pessoais-ripd>; Acesso em: 2026-10-09.


[78] Banco Central do Brasil / Conselho Monetário Nacional; Resolução Conjunta nº 1/2020: Open Finance, PDF oficial vigente versão 9; <https://normativos.bcb.gov.br/Lists/Normativos/Attachments/51028/Res_Conj_0001_v9_P.pdf>; Acesso em: 2026-10-09.

[79] OpenID Foundation; Financial-grade API Security Profile 1.0: Part 2 Advanced; <https://openid.net/specs/openid-financial-api-part-2-1_0.html>; Acesso em: 2026-10-09.


[81] PCI Security Standards Council; FAQ 1086: How does encrypted cardholder data impact PCI DSS scope? (atualizada em março de 2026); <https://www.pcisecuritystandards.org/faqs/1086/>; Acesso em: 2026-10-09.

[82] PCI Security Standards Council; FAQ 1042: Should cardholder data be encrypted while in memory?; <https://www.pcisecuritystandards.org/faqs/1042/>; Acesso em: 2026-10-09.


[84] Microsoft Learn; API Management policy reference: validate-jwt; <https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy>; Acesso em: 2026-10-09.

[85] Microsoft Learn; API Management policy expressions: tipos permitidos e verificação limitada; <https://learn.microsoft.com/en-us/azure/api-management/api-management-policy-expressions>; Acesso em: 2026-10-09.

[86] Microsoft Learn; API Management policy reference: validate-client-certificate; <https://learn.microsoft.com/en-us/azure/api-management/validate-client-certificate-policy>; Acesso em: 2026-10-09.

[87] Microsoft Learn; Microsoft identity platform: Access token claims reference; <https://learn.microsoft.com/en-us/entra/identity-platform/access-token-claims-reference>; Acesso em: 2026-10-09.

[88] Microsoft Learn; API Management policy reference: rate-limit-by-key; <https://learn.microsoft.com/en-us/azure/api-management/rate-limit-by-key-policy>; Acesso em: 2026-10-09.

[89] Microsoft Learn; API Management policy reference: quota-by-key; <https://learn.microsoft.com/en-us/azure/api-management/quota-by-key-policy>; Acesso em: 2026-10-09.

[90] Microsoft Learn; API Management policy reference: validate-content; <https://learn.microsoft.com/en-us/azure/api-management/validate-content-policy>; Acesso em: 2026-10-09.

[91] Microsoft Learn; API Management policy reference: cors; <https://learn.microsoft.com/en-us/azure/api-management/cors-policy>; Acesso em: 2026-10-09.


[93] Microsoft Learn; Microsoft identity platform: OAuth 2.0 client credentials flow, application permissions e recurso/.default; <https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow>; Acesso em: 2026-10-09.


[95] Microsoft Azure / GitHub; APIOps CLI: configuration-as-code for Azure API Management; <https://github.com/Azure/apiops-cli>; Acesso em: 2026-10-09.


[97] GitHub Docs; Workflow syntax for GitHub Actions; <https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax>; Acesso em: 2026-10-09.

[98] GNU Project; GNU Diffutils manual: Invoking diff; <https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html>; Acesso em: 2026-10-09.







[105] ANPD / Biblioteca Digital do Ministério da Justiça; Resolução CD/ANPD nº 18/2024: atuação do encarregado pelo tratamento de dados pessoais; <https://bibliotecadigital.mj.gov.br/bitstream/1/13151/1/RES_ANPD_2024_18.html>; Acesso em: 2026-10-09.


[107] GitHub Docs; Store and share data with workflow artifacts; <https://docs.github.com/en/actions/tutorials/store-and-share-data>; Acesso em: 2026-10-09.

[108] GitHub Docs; Secure use reference; <https://docs.github.com/en/actions/reference/security/secure-use>; Acesso em: 2026-10-09.

---

## Apêndice A: exemplos de policies XML por perfil de consumidor (§7.7)

> Exemplos de referência para composições específicas, não certificação ou implantação executada. A.1, A.2 e A.5 usam M2M com permissões de aplicação; A.3 usa acesso delegado; A.4 cobre somente o caminho direto de mTLS descrito; A.6 é anônimo autorizado. Os marcadores de dados e setor se somam ao perfil, sem determinar sozinhos o fluxo OAuth.

**Termos dos exemplos:** JWT (JSON Web Token) é o formato do token; TLS protege o transporte e mTLS autentica as duas pontas do segmento. JWKS é o conjunto de chaves públicas de assinatura do emissor. CA é autoridade certificadora; CRL é lista de certificados revogados; OCSP é o protocolo de consulta ao estado do certificado. HMAC é autenticação de mensagem baseada em hash com chave; neste apêndice usa SHA-256. WAF é firewall de aplicações web e L7 indica a camada de aplicação, não transporte TLS transparente.

**Pré-requisitos comuns:** registrar SKU, runtime/gateway e escopo da policy. Os exemplos por chave pressupõem suporte documentado, como gateways gerenciados clássicos ou v2; não assumir suporte em Consumption. Conferir suporte e configuração próprios de workspace/self-hosted antes de adaptar. Criar os named values referenciados, sem publicar valores de segredos; ligar chaves a referências protegidas no Key Vault, com permissões e versões controladas. Cadastrar os schemas de A.2/A.3 e os loggers de A.3/A.4 com destino e acesso aprovados. Os limites numéricos são exemplos de contrato, não capacidade medida ou mínimo regulatório [85][88][89][90].

**Identidade e herança:** A.1–A.3 e A.5 fixam tokens Entra v2.0, audience GUID da API, `azp` e clientes permitidos; A.1/A.2/A.5 exigem permissões exclusivamente de aplicação em `roles`, sem conceder o mesmo papel a usuários, e A.3 exige `scp` separado por espaço. Verificar esse contrato nas configurações de emissão/atribuição e testar que token delegado não recebe a permissão M2M. O recurso de destino precisa emitir a versão esperada; o uso de um token endpoint v2.0, isoladamente, não comprova esse formato. A migração para v1.0 exige revisar audience, `appid`, versão e emissão, não substituir uma única claim. Revisar global/produto/API/operação e `<base />`, inclusive CORS, autenticação, forwarding e captura de dados. O token de auditoria é somente a variável de saída da validação [16][84][87][93].

**Coleta e verificação:** não registrar tokens, Authorization, cookies, subscription keys, segredos ou corpos com dados pessoais em loggers, diagnostics ou traces. Avaliar também URLs/parâmetros: pseudonimizar um evento não elimina dados pessoais de outras coletas. Suspender coleta que não possa atender à minimização aprovada. Antes de produção, executar parse XML, validação de schema/configuração APIM e testes positivos/negativos no gateway escolhido, em etapas separadas. Exigir token expirado, emissor/audience/cliente/permissão incorretos, um e vários escopos, token app-only sem role e token delegado nos exemplos M2M, schema/tamanho inválidos, herança indevida e falhas dos destinos. Parser XML não executa C#, não verifica named values/CA/logger e não comprova runtime; os casos abaixo são testes requeridos, não resultados alegados [85].

### A.1 Perfil interno M2M com token Entra v2.0

Aplicável a chamada de aplicação, com client credentials, tenant, audience e cliente aprovados. `internal-api-client-id` é o GUID da API de destino; `internal-allowed-client-id` é a aplicação consumidora; `internal-required-app-role` é a permissão de aplicação exposta e concedida para a operação. O contador é por API e aplicação validada, sem depender de subscription key. A rede interna e a autorização de objeto no backend são controles separados. Testar também aplicação válida sem a role exigida e ausência de subscription key [16][87][88].

```xml
<policies>
  <inbound>
    <base />
    <set-header name="x-correlation-id" exists-action="override">
      <value>@(context.RequestId.ToString())</value>
    </set-header>
    <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization" failed-validation-httpcode="401" output-token-variable-name="validatedToken">
      <client-application-ids>
        <application-id>{{internal-allowed-client-id}}</application-id>
      </client-application-ids>
      <audiences>
        <audience>{{internal-api-client-id}}</audience>
      </audiences>
      <required-claims>
        <claim name="ver" match="all"><value>2.0</value></claim>
        <claim name="azp" match="any"><value>{{internal-allowed-client-id}}</value></claim>
        <claim name="roles" match="any"><value>{{internal-required-app-role}}</value></claim>
      </required-claims>
    </validate-azure-ad-token>
    <set-variable name="validatedClientId" value="@(((Jwt)context.Variables[&quot;validatedToken&quot;]).Claims[&quot;azp&quot;][0])" />
    <rate-limit-by-key calls="600" renewal-period="60" counter-key="@(&quot;rate:internal:&quot; + context.Api.Id + &quot;:&quot; + (string)context.Variables[&quot;validatedClientId&quot;])" />
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

### A.2 Perfil externo B2B M2M com corpo JSON contratado

Exemplo para operação com corpo JSON e cliente Entra v2.0 autorizado. Cadastrar o schema referenciado por `b2b-request-schema-id`; alternativamente, usar o schema efetivo da operação após verificar a importação. A faixa `b2b-ingress-ip-start/end` representa o endereço observado pelo APIM: atrás de proxy, não é automaticamente o IP do parceiro. A identificação do parceiro vem do token, e o caminho de borda/bypass precisa estar aprovado. Quota e rate limit usam namespaces distintos por API/aplicação, não uma subscription ausente. Testar schema, tamanho, aplicação/role, faixas observadas e contrato de contagem [16][88][89][90].

```xml
<policies>
  <inbound>
    <base />
    <ip-filter action="allow">
      <address-range from="{{b2b-ingress-ip-start}}" to="{{b2b-ingress-ip-end}}" />
    </ip-filter>
    <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization" failed-validation-httpcode="401" output-token-variable-name="validatedToken">
      <client-application-ids>
        <application-id>{{b2b-allowed-client-id}}</application-id>
      </client-application-ids>
      <audiences><audience>{{b2b-api-client-id}}</audience></audiences>
      <required-claims>
        <claim name="ver" match="all"><value>2.0</value></claim>
        <claim name="azp" match="any"><value>{{b2b-allowed-client-id}}</value></claim>
        <claim name="roles" match="any"><value>{{b2b-required-app-role}}</value></claim>
      </required-claims>
    </validate-azure-ad-token>
    <set-variable name="validatedClientId" value="@(((Jwt)context.Variables[&quot;validatedToken&quot;]).Claims[&quot;azp&quot;][0])" />
    <validate-content unspecified-content-type-action="prevent" max-size="262144" size-exceeded-action="prevent">
      <content type="application/json" validate-as="json" action="prevent" schema-id="{{b2b-request-schema-id}}" />
    </validate-content>
    <rate-limit-by-key calls="120" renewal-period="60" counter-key="@(&quot;rate:b2b:&quot; + context.Api.Id + &quot;:&quot; + (string)context.Variables[&quot;validatedClientId&quot;])" />
    <quota-by-key calls="100000" renewal-period="86400" counter-key="@(&quot;quota:b2b:&quot; + context.Api.Id + &quot;:&quot; + (string)context.Variables[&quot;validatedClientId&quot;])" />
    <set-header name="x-correlation-id" exists-action="override"><value>@(context.RequestId.ToString())</value></set-header>
  </inbound>
  <backend><base /></backend>
  <outbound>
    <set-header name="Server" exists-action="delete" />
    <base />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

### A.3 Acesso delegado a dados pessoais com auditoria pseudonimizada

Este exemplo usa access token Entra v2.0 com usuário delegado e `scp`. Para consumidor M2M, compor os controles de privacidade com A.1/A.2 e `roles`, sem impor Authorization Code pelo dado tratado. A categoria sensível legal e a hipótese do art. 11 são decisões de tratamento (§2.3), não propriedades inferidas da policy. O contrato da operação contém corpo JSON e parâmetro de rota `subjectId`; o backend verifica a autorização sobre o objeto solicitado [66].

**Auditoria:** cadastrar `audit-lgpd` e o schema; autorizar os campos e a retenção do evento. `audit-hmac-key-base64` é named value secreto referenciado ao Key Vault, contendo chave HMAC de alta entropia, com versão controlada. `audit-hmac-key-version` identifica a versão efetiva dessa chave, e `audit-pseudonym-domain` delimita controlador/ambiente/finalidade. Publicar chave/versão de modo consistente entre gateways e registrar rotação, acessos e conservação das versões conforme a finalidade. A entrada do HMAC preserva a representação canônica do identificador contratada pela API; não alterar caixa ou normalização sem decisão documentada [85].

O pseudônimo é do **objeto solicitado**, não uma prova de titular autenticado ou de autorização bem-sucedida. HMAC-SHA-256 permite correlação dentro do domínio e da versão da chave; rotação muda essa correlação. Não é anonimização: finalidade, base, acesso, risco de reidentificação e retenção continuam sujeitos à LGPD. O evento registra apenas metadados aprovados, sem token, headers, segredo, corpo ou identificador bruto. Testar um/vários/nenhum scope, cliente incorreto, objeto inexistente/não autorizado, consistência entre reinícios/gateways, rotação e ausência de dados indevidos em todas as coletas [66][85][87].

```xml
<policies>
  <inbound>
    <base />
    <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization" failed-validation-httpcode="401" output-token-variable-name="validatedToken">
      <client-application-ids><application-id>{{personal-allowed-client-id}}</application-id></client-application-ids>
      <audiences><audience>{{personal-api-client-id}}</audience></audiences>
      <required-claims>
        <claim name="ver" match="all"><value>2.0</value></claim>
        <claim name="azp" match="any"><value>{{personal-allowed-client-id}}</value></claim>
        <claim name="scp" match="any" separator=" "><value>{{personal-required-scope}}</value></claim>
      </required-claims>
    </validate-azure-ad-token>
    <set-variable name="validatedClientId" value="@(((Jwt)context.Variables[&quot;validatedToken&quot;]).Claims[&quot;azp&quot;][0])" />
    <validate-content unspecified-content-type-action="prevent" max-size="524288" size-exceeded-action="prevent">
      <content type="application/json" validate-as="json" action="prevent" schema-id="{{personal-request-schema-id}}" />
    </validate-content>
    <rate-limit-by-key calls="60" renewal-period="60" counter-key="@(&quot;rate:personal:&quot; + context.Api.Id + &quot;:&quot; + (string)context.Variables[&quot;validatedClientId&quot;])" />
    <choose>
      <when condition="@(String.IsNullOrEmpty(context.Request.MatchedParameters.GetValueOrDefault(&quot;subjectId&quot;, &quot;&quot;)))">
        <return-response><set-status code="400" reason="Missing object identifier" /></return-response>
      </when>
    </choose>
    <set-variable name="requestedObjectPseudonym" value="@{
        var objectId = context.Request.MatchedParameters.GetValueOrDefault(&quot;subjectId&quot;, &quot;&quot;);
        var input = new JArray(&quot;{{audit-pseudonym-domain}}&quot;, context.Api.Id, objectId).ToString(Newtonsoft.Json.Formatting.None);
        var key = Convert.FromBase64String(&quot;{{audit-hmac-key-base64}}&quot;);
        using (var hmac = new System.Security.Cryptography.HMACSHA256(key)) {
          return Convert.ToBase64String(hmac.ComputeHash(System.Text.Encoding.UTF8.GetBytes(input)));
        }
    }" />
    <set-header name="x-correlation-id" exists-action="override"><value>@(context.RequestId.ToString())</value></set-header>
    <log-to-eventhub logger-id="audit-lgpd"><![CDATA[
      @{
        var token = (Jwt)context.Variables["validatedToken"];
        return new JObject(
          new JProperty("ts", DateTime.UtcNow.ToString("o")),
          new JProperty("event", "acceptedAtGateway"),
          new JProperty("correlationId", context.RequestId.ToString()),
          new JProperty("operationId", context.Operation.Id),
          new JProperty("requestedObjectPseudonym", context.Variables.GetValueOrDefault<string>("requestedObjectPseudonym", "")),
          new JProperty("pseudonymAlgorithm", "HMAC-SHA-256"),
          new JProperty("pseudonymKeyVersion", "{{audit-hmac-key-version}}"),
          new JProperty("scope", String.Join(" ", token.Claims["scp"])),
          new JProperty("clientApplication", (string)context.Variables["validatedClientId"])
        ).ToString();
      }
    ]]></log-to-eventhub>
  </inbound>
  <backend><base /></backend>
  <outbound>
    <base />
    <set-header name="Cache-Control" exists-action="override"><value>no-store</value></set-header>
  </outbound>
  <on-error><base /></on-error>
</policies>
```

### A.4 Token JWT vinculado a certificado no caminho direto cliente → APIM

**Limite arquitetural:** neste exemplo, TLS mútuo termina no APIM e `context.Request.Certificate` é o certificado do cliente dessa conexão direta. A rota não contém Application Gateway ou outro proxy L7 terminando TLS antes do APIM. Application Gateway estabelece outra conexão e não encaminha automaticamente o certificado original; portanto este XML não implementa o binding do consumidor nesse caminho [19][64]. O ingresso direto requer aprovação de arquitetura e Segurança para a exposição e os controles de borda/antiabuso. Não é recomendação automática de remover WAF: se WAF L7 for requisito, este exemplo não atende ao desenho, que precisa de implementação própria comprovada para autenticação na borda e transporte/verificação autenticados dos metadados, com bloqueio de spoofing e bypass. Nenhum header de certificado é tratado como confiável por este exemplo.

**Pré-requisitos:** confirmar a versão do perfil/jornada brasileiro [28][29b][78] e seu contrato de emissão. O exemplo aceita apenas JWT assinado com `scope` separado por espaços, audience e issuer explícitos, claim de cliente definida por `openfinance-client-id-claim` e `cnf.x5t#S256` conforme RFC 8705; token opaco exige outro desenho com introspecção. Cadastrar discovery/JWKS aprovados, audiência, cliente e scope, habilitar recebimento de certificado no domínio/gateway pelo mecanismo de seu runtime e verificar trust store, CAs e revogação. `openfinance-client-issuer-certificate-id` identifica a chave pública do emissor de certificado efetivamente cadastrado e aprovado; não presumir que um ID de raiz valida todo emissor intermediário. Não adaptar automaticamente para workspace, que não oferece o mesmo suporte a CAs [64][84][86].

Cadastrar `audit-openfinance` e sua política de campos/retenção. O evento abaixo comprova aceitação no gateway quando executado, não conclusão da transação nem conformidade FAPI. `x-correlation-id`/`gatewayRequestId` são correlação técnica própria: não substituem os cabeçalhos de interação, assinaturas e demais comportamentos exigidos pela jornada Open Finance, a implementar e testar conforme o perfil vigente. Exigir testes do **caminho efetivo completo**: sem certificado; certificado expirado, revogado ou de emissor não autorizado; token inválido, de outro destinatário/cliente ou com scope ausente; `cnf` ausente/malformado; token de outro certificado; renovação e falha fechada da verificação. Um teste direto não valida uma rota por proxy. Nada disso é declarado executado neste guia.

```xml
<policies>
  <inbound>
    <base />
    <validate-client-certificate validate-revocation="true" validate-trust="true" validate-not-before="true" validate-not-after="true" ignore-error="false">
      <identities>
        <identity issuer-certificate-id="{{openfinance-client-issuer-certificate-id}}" />
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
    <validate-jwt header-name="Authorization" require-scheme="Bearer" require-signed-tokens="true" require-expiration-time="true" failed-validation-httpcode="401" failed-validation-error-message="Invalid access token" output-token-variable-name="validatedToken">
      <openid-config url="{{openfinance-issuer-discovery}}" />
      <audiences><audience>{{openfinance-audience}}</audience></audiences>
      <issuers><issuer>{{openfinance-token-issuer}}</issuer></issuers>
      <required-claims>
        <claim name="{{openfinance-client-id-claim}}" match="any"><value>{{openfinance-allowed-client-id}}</value></claim>
        <claim name="scope" match="any" separator=" "><value>{{openfinance-required-scope}}</value></claim>
      </required-claims>
    </validate-jwt>
    <set-variable name="validatedClientId" value="@(((Jwt)context.Variables[&quot;validatedToken&quot;]).Claims[&quot;{{openfinance-client-id-claim}}&quot;][0])" />
    <set-variable name="tokenCnfX5tS256" value="@{
        var token = (Jwt)context.Variables[&quot;validatedToken&quot;];
        string[] values;
        if (!token.Claims.TryGetValue(&quot;cnf&quot;, out values) || values == null || values.Length != 1) { return &quot;&quot;; }
        try {
          return (string)JObject.Parse(values[0])[&quot;x5t#S256&quot;] ?? &quot;&quot;;
        } catch { return &quot;&quot;; }
    }" />
    <choose>
      <when condition="@(String.IsNullOrEmpty((string)context.Variables[&quot;clientCertX5tS256&quot;]) || String.IsNullOrEmpty((string)context.Variables[&quot;tokenCnfX5tS256&quot;]) || (string)context.Variables[&quot;clientCertX5tS256&quot;] != (string)context.Variables[&quot;tokenCnfX5tS256&quot;])">
        <return-response><set-status code="401" reason="Certificate bound token mismatch" /></return-response>
      </when>
    </choose>
    <set-header name="x-correlation-id" exists-action="override"><value>@(context.RequestId.ToString())</value></set-header>
    <rate-limit-by-key calls="300" renewal-period="60" counter-key="@(&quot;rate:certificate-bound:&quot; + context.Api.Id + &quot;:&quot; + (string)context.Variables[&quot;validatedClientId&quot;])" />
    <log-to-eventhub logger-id="audit-openfinance"><![CDATA[
      @{
        return new JObject(
          new JProperty("ts", DateTime.UtcNow.ToString("o")),
          new JProperty("event", "acceptedAtGateway"),
          new JProperty("gatewayRequestId", context.RequestId.ToString()),
          new JProperty("operationId", context.Operation.Id),
          new JProperty("clientApplication", (string)context.Variables["validatedClientId"])
        ).ToString();
      }
    ]]></log-to-eventhub>
  </inbound>
  <backend><forward-request fail-on-error-status-code="true" /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
```

> A validação do token, a extração de `cnf` do objeto validado e a comparação com o certificado da conexão direta precisam passar pelos testes do gateway selecionado. O backend continua responsável por consentimento, autorização de objeto e regras da jornada. Parse XML e opções de revogação declaradas não comprovam disponibilidade de CRL/OCSP, trust store, comportamento de falha ou conformidade do perfil.

### A.5 Fachada administrativa de aplicação, com automação M2M

A fachada é uma API customizada publicada no **data plane**, não a API ARM de gestão do APIM. Fixar audience da fachada, aplicação consumidora e application role autorizada para a operação; a faixa de ingresso é a observada no salto e deve compor o caminho interno aprovado. Um console de usuário requer outro perfil delegado, não o client credentials deste exemplo. Se o backend chamar ARM, sua identidade e permissões RBAC são configuradas separadamente; PIM/aprovação de acesso humano e de mudança continuam em §7.1/§9. A role do JWT não concede RBAC nem ativa PIM. Testar token de outra audience/aplicação, role ausente e tentativa de acesso direto à gestão [16][87].

```xml
<policies>
  <inbound>
    <base />
    <ip-filter action="allow">
      <address-range from="{{admin-ingress-ip-start}}" to="{{admin-ingress-ip-end}}" />
    </ip-filter>
    <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization" failed-validation-httpcode="401" output-token-variable-name="validatedToken">
      <client-application-ids><application-id>{{admin-allowed-client-id}}</application-id></client-application-ids>
      <audiences><audience>{{admin-facade-client-id}}</audience></audiences>
      <required-claims>
        <claim name="ver" match="all"><value>2.0</value></claim>
        <claim name="azp" match="any"><value>{{admin-allowed-client-id}}</value></claim>
        <claim name="roles" match="any"><value>{{admin-required-app-role}}</value></claim>
      </required-claims>
    </validate-azure-ad-token>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
```

### A.6 API pública anônima autorizada, somente GET

Aplicar no escopo **API/inbound** de gateway com `rate-limit-by-key` e `quota-by-key` suportadas (Usage documenta classic/v2; confirmar o runtime escolhido). Não usar este XML em Consumption sem outro mecanismo suportado. Desabilitar exigência de subscription para a API deliberadamente pública e conferir autenticação/CORS herdadas, sem enfraquecer APIs vizinhas. O exemplo não usa `rate-limit` ou `quota` por subscription, que não limitam chamadas sem chave [88][89].

A chave contém API, finalidade e `context.Request.IpAddress`, o **IP observado pelo APIM**. NAT e proxies podem agregar consumidores; endereços distintos não são identidades. Não ler X-Forwarded-For ou outro header arbitrário para declarar o IP original. Atrás de proxy, o IP observado pode ser o da borda: implementar proteção de abuso também nela, com origem/bypass restritos, conforme o desenho aprovado. Rate limits são aproximados e independentes por gateway/região; quota pode admitir chamadas durante reinícios de recursos. Isso não é limite financeiro global exato, que exige contabilização própria [88][89].

`public-prod-origin` é uma origem explícita aprovada de produção; `*` só pode ser usado no sandbox segundo §7.5/§9.3. CORS não protege contra clientes fora do navegador. Publicar somente operações GET com dados públicos sem corpo pessoal/segredo e verificar o cache permitido; IP/metadados também entram na análise de privacidade e retenção. Testar GET sem chave/token, métodos negados, preflight/origem não permitida, carga distribuída/NAT/proxy e limites sem depender de subscription. Os números abaixo são ilustrativos, a calibrar por contrato [91].

```xml
<policies>
  <inbound>
    <base />
    <cors allow-credentials="false">
      <allowed-origins><origin>{{public-prod-origin}}</origin></allowed-origins>
      <allowed-methods><method>GET</method></allowed-methods>
    </cors>
    <choose>
      <when condition="@(context.Request.Method != &quot;GET&quot;)">
        <return-response>
          <set-status code="405" reason="Method Not Allowed" />
          <set-header name="Allow" exists-action="override"><value>GET</value></set-header>
        </return-response>
      </when>
    </choose>
    <rate-limit-by-key calls="60" renewal-period="60" counter-key="@(&quot;rate:public:&quot; + context.Api.Id + &quot;:&quot; + context.Request.IpAddress)" />
    <quota-by-key calls="100000" renewal-period="86400" counter-key="@(&quot;quota:public:&quot; + context.Api.Id + &quot;:&quot; + context.Request.IpAddress)" />
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

## Apêndice B: modelo OpenAPI para contrato HTTP/REST e metadados de governança

Modelo **a renderizar**, não contrato pronto para importação: substituir os placeholders por valores aprovados, validar referências, schemas, operationIds e URLs e impedir publicação com valores pendentes. OpenAPI 3.0.3 descreve o contrato HTTP/REST; GraphQL usa seu schema e verificações de consultas, WebSocket precisa de contrato de conexão/mensagens e outros protocolos precisam de seus artefatos próprios, conforme §6.6. Uma fachada HTTP adicional não substitui o contrato principal do protocolo [62][55].

`x-governance` é uma convenção deste guia, não padrão externo nem mecanismo de enforcement APIM. Suas dimensões, bases dos arts. 7º/11, conjuntos de retenção e transferências alimentam o intake e o gate; a pipeline precisa conferir os valores e as evidências reais. O exemplo de `security` escolhe M2M; para usuário delegado escolher `oauthDelegated` com o scope contratado e PKCE no cliente público. Para uma operação anônima autorizada, declarar a dispensa explicitamente e aplicar o perfil A.6. Declarar OAuth/security no OpenAPI não instala validação de token: definitions podem ser ignoradas na importação APIM. Conferir e testar as policies resultantes [21][66][62][55].

```yaml
openapi: 3.0.3
info:
  title: "{nome-da-api}"
  version: "1.0.0"
  description: |
    {descrição breve da API}
    - Responsável: {squad / líder técnico / plantão}
    - Perfis compostos e evidências: x-governance e ficha de intake (§2.3.4/§7.7)
    - Bases LGPD por tratamento: artigo 7º ou 11, inciso e finalidade aplicáveis
    - Retenção: política aprovada por conjunto, obrigação, prazo e evento inicial (§10.7)
  contact:
    name: "Equipe {squad}"
    email: "{squad}@empresa.com"
  license:
    name: "Uso interno corporativo"

servers:
  - url: "https://api.empresa.com/{produto}/v1"
    description: Produção

x-governance:
  schema-version: 1
  protocol: "http-rest"
  intake-evidence: "{referência real da ficha aprovada}"
  exposure: "{interna ou externa, conforme desenho aprovado}"
  data-categories:
    personal-common: "{booleano a renderizar}"
    sensitive-art-5-II: ["{categorias efetivamente tratadas, ou lista vazia}"]
    financial: "{booleano a renderizar; não equivale a sensível legal}"
    corporate-classification: "{classificação corporativa aprovada}"
  sector-regimes: ["{atos/perfis aplicáveis e entidade abrangida, ou lista vazia}"]
  consumer-type: "{M2M, usuário delegado ou anônimo autorizado}"
  criticality: "{criticidade e critérios aprovados}"
  legal-bases:
    - processing-purpose: "{finalidade da operação de tratamento}"
      article: "{7 ou 11, quando aplicável}"
      inciso: "{inciso e hipótese efetivamente aplicáveis}"
      decision-evidence: "{referência real da decisão do controlador}"
  impact-assessment:
    trigger: "{determinação legal/ANPD, recomendação por alto risco ou política interna}"
    treatment-evidence: "{operação/projeto e relatório que a cobre, quando aplicável}"
  identity-profile:
    token-version: "{Entra v1.0, v2.0 ou contrato de outro emissor}"
    allowed-audiences: ["{destinatário confirmado}"]
    allowed-client-applications: ["{aplicações aprovadas}"]
    required-permissions: ["{roles para M2M ou scopes delegados, conforme perfil}"]
    policy-evidence: "{configuração e relatório real de testes no gateway escolhido}"
  retention-policy:
    policy-id: "{política aprovada e responsável}"
    records:
      - dataset: "{conjunto de registros, distinto de todo log do gateway}"
        purpose: "{finalidade necessária}"
        obligation: "{ato/artigo/inciso, contrato ou política interna identificada}"
        duration: "{prazo aprovado por conjunto}"
        start-event: "{marco inicial aplicável}"
        disposal-or-hold: "{eliminação ou conservação adicional justificada}"
  international-transfers:
    data-and-accesses: "{dados, exportador/importador, destinos e acessos mapeados}"
    art-33-mechanism: "{mecanismo válido ou conclusão documentada de não incidência}"
    instrument-evidence: "{instrumento/cláusulas efetivamente aplicáveis conforme ANPD nº 19}"

security:
  - oauthM2M: []

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
    oauthM2M:
      type: oauth2
      description: |
        Exemplo Entra M2M: application permissions em roles, cliente e audience
        verificados pela policy. O token endpoint usa o recurso/.default
        aprovado; não solicitar um scope delegado em seu lugar.
      x-required-application-roles: ["{application role aprovada}"]
      x-token-request-scope: "{URI do recurso}/.default"
      flows:
        clientCredentials:
          tokenUrl: "https://{authority-host}/{tenant}/oauth2/v2.0/token"
          scopes: {}
    oauthDelegated:
      type: oauth2
      description: "Acesso de usuário; clientes públicos usam Authorization Code com PKCE."
      flows:
        authorizationCode:
          authorizationUrl: "https://{authority-host}/{tenant}/oauth2/v2.0/authorize"
          tokenUrl: "https://{authority-host}/{tenant}/oauth2/v2.0/token"
          scopes:
            "{scope delegado contratado}": "Permissão de leitura do recurso"
          x-pkce-required: true
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

## Apêndice C: modelo de registro de decisão arquitetural (ADR)

ADR significa Architecture Decision Record. O código `ADR-APIM-{NNN}` abaixo é uma convenção local de identificação sequencial, não uma norma externa; substituir `{NNN}` pelo ID do registro aprovado. O template conserva a autoridade, alternativas, consequências e critérios de revalidação da decisão.

```markdown
# ADR-APIM-{NNN}: {Título da decisão}

- **Status:** Proposta | Aceita | Substituída por ADR-APIM-{MMM} | Obsoleta
- **Data:** AAAA-MM-DD
- **Decisor único (A):** {nome, papel, delegação/autoridade e substituto}
- **Consultados:** {nomes/papéis: arquitetura, plataforma, segurança, dados, jurídico/compliance/DPO, continuidade e FinOps conforme assunto}
- **Informados:** {produtores, consumidores e auditoria}

## 1. Contexto
{Problema, prazo, exposição, dados legais/corporativos, setor, consumidor e criticidade. Separar modelo de adoção, tier/gateway e configuração de rede/região. Registrar obrigações de §2.3, objetos/dependências de recuperação, RTO/RPO com unidades/fonte, requisitos de dados e evidências de disponibilidade. Premissas não verificadas permanecem pendentes; não escrever como fato do cliente.}

## 2. Opções avaliadas
- Opção A: {descrição}; vantagens e desvantagens
- Opção B: {descrição}; vantagens e desvantagens
- Opção C: {descrição}; vantagens e desvantagens

## 3. Decisão
{Opção escolhida e justificativa concisa.}

## 4. Consequências
- Positivas
- Negativas
- Riscos aceitos
- Riscos mitigados

## 5. Critérios de revalidação
- {Mudança de obrigação, dado/finalidade, exposição, capacidade regional, tier/gateway, dependência ou resultado de ensaio. Nomear métrica, unidade, limite e registro aprovado que dispara revisão.}
- {Periodicidade aprovada, data de próxima revisão, owner e efeito de requisito/evidência vencidos.}

## 6. Mapeamento RACI
Legenda: R executa; A responde pela decisão/aceite; C é consultado; I é informado. Nomear uma pessoa e substituto para cada A, conforme §11.3; não somar accountables.

| Atividade | A único | Executores (R) | Consultados (C) | Informados (I) |
|---|---|---|---|---|
| Implementação da API/policy de domínio | API/Product owner | Tech lead produtor | Plataforma, segurança e dados; jurídico/compliance quando aplicável | Consumidor |
| Implementação da plataforma/policy padrão | Platform owner APIM | Plataforma | Produtor, segurança e dados | Consumidores |
| Monitoramento da API/jornada | API/Product owner | Plantão produtor | Plataforma e segurança | Consumidor |
| Monitoramento da plataforma | Platform owner APIM | Plantão da plataforma | Produtor e segurança | API/Product owner |

{Se a decisão abranger incidente/dados, adicionar a atividade de avaliação/comunicação com A no representante autorizado do controlador e DPO consultado. Se abranger autorização crítica, adicionar A segregado de release. Seguir §11.3 e §11.6.1, sem transformar o DPO em A legal universal.}

## 7. Referências
- §X do guia APIM corporativo
- [n] da §15
- Documentos internos
```

---

## Apêndice D: fragmento de promoção e verificação de drift (GitHub Actions)

Este é um **contrato de integração ilustrativo**, não uma automação entregue ou implantada. Os caminhos Unix abaixo valem somente para os runners Ubuntu deste exemplo. DEV/UAT/PROD são rótulos funcionais; os environments são `development`, `uat` e `production`, e os arquivos de parâmetros usam esses mesmos nomes.

### Pré-requisitos para habilitar o fragmento

- Entregar e qualificar scripts, workflow reutilizável `apim-stage.yml`, schemas de evidência, Bicep/ARM, parâmetros, overrides, contratos/testes, ruleset Spectral, inventários e política de autorização. A ausência de um deles bloqueia a execução; um arquivo com exit 0 sem trabalho/evidência não satisfaz o contrato.
- `preflight.sh` verifica arquivos, schemas, ferramentas/versões fixadas, cobertura do publisher e controles §11.6.1. Runners têm Bash, GNU diff/tar, `jq`, SHA-256, Azure CLI/Bicep, linter XML/Spectral e runner de testes do protocolo; APIOps CLI, se escolhido, exige Node.js ≥22 [95]. Instalações/locks são revisados e reproduzíveis, não downloads de “latest”.
- As tags de actions abaixo são rótulos de exemplo. Antes de uso real, fixar SHAs completos verificados e manter o workflow/scripts/rulesets sob revisão protegida [108]. PR não recebe identidade Azure/segredos de publicação. O pacote promovido só nasce do commit revisto em `main`.
- Confirmar plano/visibilidade GitHub privado, branch/ruleset/environment, aprovador segregado e integração cumulativa quando exigida (§11.6.1). O environment não implementa por lista o AND entre plataforma, API owner e segurança/compliance [52].
- Identidades distintas por ambiente/target; `production-plan` controla a identidade da prévia e `production` a publicação. ARM what-if requer permissões de deployment, portanto não chamar essa identidade de “somente leitura” [14]. Gate externo/broker impede liberação de escrita sem autorização; ausência de integração comprovada bloqueia PROD crítica.
- Definir lock de escrita por instância entre workflows/repositórios. `concurrency` deste exemplo serializa somente suas próprias execuções. A baseline de drift é o último release aceito no target, não o último commit ainda não implantado.

### Contratos dos componentes não fornecidos

| Componente | Entradas/saídas obrigatórias | Exit code e evidência de aceite |
|---|---|---|
| `preflight.sh`, `validate-source.sh` | Commit, contratos por protocolo, policies, intake, schemas, rulesets/locks e configuração do workflow; relatório de arquivos/ferramentas e cobertura | 0 somente com todos os pré-requisitos/verificações completos; qualquer falha ou resultado false/ausente é não zero. CI cobre Spectral, diff de contrato, XML estrito, herança aplicável, secrets scan e gates compostos §9.3; parse não é teste em APIM |
| `build-release.sh` | Fonte aprovada e dependências fixas; gera `release-bundle.tgz` e manifesto de arquivos SHA-256, commit, ARM compilado, parâmetros `development/uat/production`, configuração/overrides, scripts, regras, fixtures, normalizador e política de recuperação | 0 só com pacote completo/reproduzível e sem segredos, paths absolutos/traversal ou links que escapem da raiz de extração; nenhuma etapa posterior recompila ou altera conteúdo. Arquivo bruto tem hash diferente do digest de transporte [107] |
| Workflow `apim-stage.yml` | Environment, pacote/nome/hash/commit e prova anterior; declara `evidence_artifact`, `evidence_sha256`, `release_sha256` como outputs | Só aceita `development` ou `uat`, usa a identidade do ambiente, verifica pacote, prévia/implanta e faz requests reais positivos/negativos §9.3. Só retorna sucesso após provas completas e saúde/recuperação necessárias; falha impede próximo job |
| `guard-release.sh` | Subcomandos `preview`, `authorise`, `completion`; pacote, target, hashes de parâmetros/overrides, planos ARM/APIM, provas DEV/UAT, identidade/autorização e registros de exceção | 0 só com schema/assinatura e vínculos válidos, guard verdadeiro, cobertura completa e prazo vigente. Provas contêm testes esperados/executados, resultados observados, target/versão, horário e hash; dummy/vazio/skipped/inconclusivo bloqueia. `authorise` verifica autores/iniciadores, A e AND de papéis quando requerido no registro protegido; `completion` verifica observação §9.2 |
| `preview-apim.sh`, `deploy-apim.sh`, `post-deploy-tests.sh` | Mesmo pacote/target/overrides; prévia de configuração APIM separada da ARM, operação de deploy e resultados reais sanitizados | Preview não executa policy; deploy exige autorização vinculada ao pacote/plano/target. Testes usam runner HTTP/protocolo, identidades/fixtures sintéticas e asserts §9.3; 0 somente com resultados e cobertura aprovados, não só arquivo de log |
| `recover-release.sh` | Operação parcial/falha, incidente, plano de recuperação e pacote anterior aceito com hash/parametrização | Aborta sem escrita se não houve alteração; após alteração só faz rollback automatizado em cenário aprovado/reversível. Senão contém/escalona fix-forward autorizado. Registra decisão, plantão, erros, hashes e saúde por `janela_saude_recuperacao_min`; falha deixa incidente aberto. Mesmo recuperado, o release que falhou não fica verde |
| `fetch-approved-baseline.sh`, extractor, `normalise-apiops.sh`, `validate-layout.sh` | Baseline assinada/selada do target, inventário/filtro e versões de extractor/normalizador; geram árvores `expected`/`actual` comparáveis e manifesto da extração | 0 somente com target/cobertura/layout completos; inexistência, erro de extração, override não aplicado ou vazio inesperado bloqueia. Vazio legítimo precisa de inventário aprovado. Alterar normalização exige revisão; não remover diferenças de controle para “limpar” drift |
| `sanitise-evidence.sh` | Evidência bruta restrita e política de campos/segredos; saída publicável com status, códigos, hashes e diferenças preservadas sem valores sensíveis | 0 somente com inspeção de sanitização e arquivos obrigatórios; erro impede upload público. Logs/relatórios não podem vazar tokens, certificados privados, corpos ou dados pessoais |

O plano de autorização registra hash bruto, commit, target, parâmetros/overrides, digest de ambas as prévias, provas DEV/UAT, aprovadores e `valid_until` em UTC. Target alterado, plano obsoleto ou novo pacote exige nova prévia e autorização; o guard revalida o estado sob o lock antes de escrever. Não há garantia de atomicidade frente a alterações externas não controladas.

```yaml
name: apim-ci

on:
  pull_request:
  push:
    branches: ["main"]
    paths:
      - "apis/**"
      - "policies/**"
      - "bicep/**"
      - "config/**"
      - "overrides/**"
      - "scripts/**"
      - "tools/**"
      - "tests/**"
      - "contracts/**"
      - "rules/**"
      - "governance/**"
      - ".spectral*"
      - "package*.json"
      - ".github/**"
  schedule:
    - cron: "17 3 * * 1"
  workflow_dispatch:

permissions:
  contents: read

defaults:
  run:
    shell: bash

concurrency:
  group: apim-production
  cancel-in-progress: false

jobs:
  ci:
    if: github.event_name == 'pull_request' || github.event_name == 'push'
    runs-on: ubuntu-latest
    outputs:
      release_sha256: ${{ steps.seal.outputs.release_sha256 }}
    steps:
      - uses: actions/checkout@v4
      - name: Preflight e validacao estatica por protocolo
        run: |
          ./scripts/preflight.sh source
          ./scripts/validate-source.sh --output artifacts/ci
      - name: Build unico e hash bruto
        id: seal
        run: |
          ./scripts/build-release.sh --output release-bundle.tgz
          test -s release-bundle.tgz
          digest=$(sha256sum release-bundle.tgz | cut -d ' ' -f 1)
          echo "release_sha256=$digest" >> "$GITHUB_OUTPUT"
      - uses: actions/upload-artifact@v4
        with:
          name: apim-release-${{ github.sha }}
          path: release-bundle.tgz
          if-no-files-found: error

  development:
    needs: [ci]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      id-token: write
    uses: ./.github/workflows/apim-stage.yml
    with:
      environment: development
      release_artifact: apim-release-${{ github.sha }}
      release_sha256: ${{ needs.ci.outputs.release_sha256 }}
      expected_commit: ${{ github.sha }}
    secrets:
      azure_client_id: ${{ secrets.DEV_AZURE_CLIENT_ID }}
      azure_tenant_id: ${{ secrets.DEV_AZURE_TENANT_ID }}
      azure_subscription_id: ${{ secrets.DEV_AZURE_SUBSCRIPTION_ID }}

  uat:
    needs: [ci, development]
    permissions:
      contents: read
      id-token: write
    uses: ./.github/workflows/apim-stage.yml
    with:
      environment: uat
      release_artifact: apim-release-${{ github.sha }}
      release_sha256: ${{ needs.ci.outputs.release_sha256 }}
      expected_commit: ${{ github.sha }}
      previous_evidence: ${{ needs.development.outputs.evidence_artifact }}
      previous_evidence_sha256: ${{ needs.development.outputs.evidence_sha256 }}
    secrets:
      azure_client_id: ${{ secrets.UAT_AZURE_CLIENT_ID }}
      azure_tenant_id: ${{ secrets.UAT_AZURE_TENANT_ID }}
      azure_subscription_id: ${{ secrets.UAT_AZURE_SUBSCRIPTION_ID }}

  plan-production:
    needs: [ci, uat]
    runs-on: ubuntu-latest
    environment: production-plan
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: apim-release-${{ github.sha }}
      - uses: actions/download-artifact@v4
        with:
          name: ${{ needs.uat.outputs.evidence_artifact }}
          path: proof
      - name: Verifica pacote antes de extrair/executar
        env:
          RELEASE_SHA: ${{ needs.ci.outputs.release_sha256 }}
          PROOF_SHA: ${{ needs.uat.outputs.evidence_sha256 }}
        run: |
          printf '%s  %s\n' "$RELEASE_SHA" release-bundle.tgz | sha256sum --check --strict
          printf '%s  %s\n' "$PROOF_SHA" proof/evidence.json | sha256sum --check --strict
          mkdir release
          tar -xzf release-bundle.tgz -C release
          ./release/scripts/preflight.sh release
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Previa com parametros efetivos de production
        env:
          RESOURCE_GROUP: ${{ vars.AZURE_RESOURCE_GROUP }}
        run: |
          mkdir -p artifacts/plan
          az deployment group what-if --resource-group "$RESOURCE_GROUP" \
            --template-file release/bicep/main.json \
            --parameters @release/bicep/parameters/production.json \
            --no-pretty-print > artifacts/plan/arm.json
          ./release/scripts/preview-apim.sh --environment production \
            --release release --output artifacts/plan/apim.json
          ./release/scripts/guard-release.sh preview --release release \
            --environment production --proof proof/evidence.json \
            --arm-plan artifacts/plan/arm.json --apim-plan artifacts/plan/apim.json \
            --output artifacts/plan/preview.json
          jq -e '.eligible == true' artifacts/plan/preview.json
      - uses: actions/upload-artifact@v4
        with:
          name: production-plan-${{ github.sha }}
          path: artifacts/plan
          if-no-files-found: error

  deploy-production:
    needs: [ci, plan-production]
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: apim-release-${{ github.sha }}
      - uses: actions/download-artifact@v4
        with:
          name: production-plan-${{ github.sha }}
          path: plan
      - name: Verifica pacote e autorizacao antes de escrita
        env:
          RELEASE_SHA: ${{ needs.ci.outputs.release_sha256 }}
          CHANGE_CONTROL_READ_TOKEN: ${{ secrets.CHANGE_CONTROL_READ_TOKEN }}
        run: |
          printf '%s  %s\n' "$RELEASE_SHA" release-bundle.tgz | sha256sum --check --strict
          mkdir release
          tar -xzf release-bundle.tgz -C release
          mkdir -p artifacts/production
          ./release/scripts/preflight.sh release
          ./release/scripts/guard-release.sh authorise --release release \
            --environment production --plan plan \
            --output artifacts/production/authorisation.json
          jq -e '.authorised == true' artifacts/production/authorisation.json
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Publica, verifica e recupera sem ocultar falha
        run: |
          rc=0
          ./release/scripts/deploy-apim.sh --environment production --release release \
            --authorisation artifacts/production/authorisation.json \
            --output artifacts/production/deployment.json || rc=$?
          if [ "$rc" -eq 0 ]; then
            ./release/scripts/post-deploy-tests.sh --environment production \
              --release release --output artifacts/production/tests.json || rc=$?
          fi
          if [ "$rc" -eq 0 ]; then
            ./release/scripts/guard-release.sh completion --release release \
              --environment production --evidence artifacts/production \
              --output artifacts/production/completion.json || rc=$?
          fi
          if [ "$rc" -eq 0 ]; then
            jq -e '.accepted == true' artifacts/production/completion.json || rc=$?
          fi
          if [ "$rc" -ne 0 ]; then
            if ./release/scripts/recover-release.sh --environment production \
              --release release --evidence artifacts/production; then
              echo "Recuperacao registrada; o release continua reprovado."
            else
              echo "Falha de recuperacao; incidente aberto e promocao bloqueada." >&2
            fi
            exit 1
          fi
      - name: Sanitiza evidencia antes de upload
        id: safe
        if: always()
        run: ./release/scripts/sanitise-evidence.sh --input artifacts/production --output artifacts/production-safe
      - uses: actions/upload-artifact@v4
        if: always() && steps.safe.outcome == 'success'
        with:
          name: production-evidence-${{ github.sha }}
          path: artifacts/production-safe
          if-no-files-found: error

  drift-report:
    if: (github.event_name == 'schedule' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production-drift
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - name: Inicializa estado de erro
        run: |
          mkdir -p artifacts/drift-raw
          echo "error" > artifacts/drift-raw/status.txt
          echo "DRIFT_STATUS=error" >> "$GITHUB_ENV"
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Extrai e compara layouts qualificados
        run: |
          mkdir -p artifacts/drift-raw
          ./scripts/preflight.sh drift
          ./scripts/fetch-approved-baseline.sh --environment production \
            --output artifacts/baseline
          ./tools/apiops-extractor.sh --environment production \
            --output artifacts/extracted-raw --manifest artifacts/extraction.json \
            2>&1 | tee artifacts/drift-raw/extractor.log
          ./scripts/normalise-apiops.sh --baseline artifacts/baseline \
            --extracted artifacts/extracted-raw --manifest artifacts/extraction.json \
            --output artifacts/comparable
          ./scripts/validate-layout.sh --expected artifacts/comparable/expected \
            --actual artifacts/comparable/actual --manifest artifacts/extraction.json
          rc=0
          diff -ru artifacts/comparable/expected artifacts/comparable/actual \
            > artifacts/drift-raw/report.diff 2> artifacts/drift-raw/diff.stderr || rc=$?
          echo "$rc" > artifacts/drift-raw/diff.exit
          case "$rc" in
            0) status=clean ;;
            1) status=drift ;;
            *) echo "Erro de comparacao, nao evidencia de limpeza." >&2; exit "$rc" ;;
          esac
          echo "$status" > artifacts/drift-raw/status.txt
          echo "DRIFT_STATUS=$status" >> "$GITHUB_ENV"
      - name: Sanitiza resultado inclusive em falha
        id: safe
        if: always()
        run: ./scripts/sanitise-evidence.sh --input artifacts/drift-raw --output artifacts/drift-safe
      - uses: actions/upload-artifact@v4
        if: always() && steps.safe.outcome == 'success'
        with:
          name: drift-report-${{ github.run_id }}
          path: artifacts/drift-safe
          if-no-files-found: error
      - name: Bloqueia drift ou verificacao inconclusiva
        if: always() && env.DRIFT_STATUS != 'clean'
        run: |
          echo "Drift ou erro: registrar e reconciliar; sem aceite de limpeza." >&2
          exit 1
```

Os filtros de push abrangem fontes, parâmetros, scripts, ferramentas, testes, schemas, rulesets e workflows; dependência nova exige revisão dos filtros. PR não usa filtro para que o check obrigatório não desapareça silenciosamente. Branch/ruleset, environments, broker/registro de aprovação e retenção de evidência são configurações externas obrigatórias, não criadas pelo YAML.

A qualificação anterior ao uso real deve demonstrar: pacote alterado, guard false, prova ausente/vencida, scripts/workflow faltantes, aprovação de autor, plano PROD divergente, extractor que falha antes do `tee`, baseline/layout inválidos e `diff` que retorna 2. Nenhum desses casos pode emitir aceite ou `DRIFT_STATUS=clean`. Demonstrar também o caso 0 limpo, 1 drift e a falha de smoke/recuperação. São critérios de teste, não testes executados neste guia.

---

## Apêndice E: glossário

| Termo/sigla | Nome ou definição | Uso e limites |
|---|---|---|
| ADR | Architecture Decision Record | Registro de decisão arquitetural; ver Apêndice C |
| ANPD | Agência Nacional de Proteção de Dados | Denominação vigente após a Lei nº 15.352/2026; normas e documentos históricos podem usar Autoridade Nacional de Proteção de Dados [110] |
| AKS | Azure Kubernetes Service | Para self-hosted gateway |
| API | Application Programming Interface | Interface de programação de aplicações |
| BACEN | Banco Central do Brasil | Autoridade monetária brasileira |
| BCP, em continuidade | Business Continuity Plan | Plano de continuidade de negócio; não é o mesmo uso de BCP em RFCs |
| BCP, em RFCs | Best Current Practice | Categoria de boas práticas da IETF; RFC 9700 trata de segurança OAuth, não de plano de continuidade |
| CIDR | Classless Inter-Domain Routing | Notação de bloco IP |
| CMN | Conselho Monetário Nacional | Emite resoluções complementares ao BACEN |
| COBIT | Control Objectives for Information and Related Technologies | Framework ISACA para governança de informação e tecnologia |
| COSO | Committee of Sponsoring Organizations of the Treadway Commission | Framework de controles internos e gestão de risco |
| CORS | Cross-Origin Resource Sharing | Política HTTP |
| DCR | Dynamic Client Registration | Registro dinâmico de cliente OAuth (RFC 7591) |
| DNS | Domain Name System | Sistema de nomes de domínio |
| DPA | Data Protection Addendum | Aditivo contratual Microsoft para proteção de dados |
| DPO / encarregado | Data Protection Officer / encarregado pelo tratamento de dados pessoais | Papel conforme a LGPD e regulamentação aplicável, com hipóteses/exceções próprias; apoia o controlador, não substitui sua responsabilidade legal [66] |
| DR | Disaster Recovery | Recuperação após desastre com escopo, dependências, RTO/RPO e provas definidos; não é garantia automática do tier |
| DX | Developer Experience | Experiência do desenvolvedor |
| FAPI | Financial-grade API | Perfis de segurança da OpenID Foundation; identificar edição/perfil e requisito aplicável. LGPD ou PCI não exigem FAPI genericamente |
| FAPI 1.0 | Família de perfis FAPI 1.0 | Não equivale a FAPI 2.0. Para Open Finance Brasil, conferir o manual/perfil brasileiro vigente e suas provas de conformidade; citar FAPI genérico não demonstra atendimento integral |
| FinOps | Cloud Financial Operations | Disciplina de governança de custo em nuvem |
| HMAC | Hash-based Message Authentication Code | Para integridade |
| IaC | Infrastructure as Code | Bicep, Terraform, etc. |
| ICP | Infraestrutura de Chaves Públicas | Cadeia ICP-Brasil ou ICP do Open Finance |
| JAR | JWT-Secured Authorization Request | RFC 9101 |
| JWS | JSON Web Signature | RFC 7515 |
| JWT | JSON Web Token | RFC 7519 |
| LGPD | Lei nº 13.709/2018, Lei Geral de Proteção de Dados Pessoais | Hipótese legal e finalidade por operação de tratamento; art. 7º ou art. 11 quando aplicável, não residência Brasil universal [66] |
| Dado pessoal | Informação relacionada a pessoa natural identificada ou identificável | Definição do art. 5º, I; metadados/identificadores também podem ser pessoais [66] |
| Dado pessoal sensível | Categorias enumeradas no art. 5º, II, da LGPD | Dado financeiro não é sensível apenas por ser financeiro; pode revelar categoria sensível conforme o contexto. Hipóteses do art. 11 quando aplicáveis [66] |
| Restrito corporativo | Classificação interna de confidencialidade/risco | Não substitui a categoria legal do dado nem define, sozinha, regime setorial ou fluxo de identidade |
| M2M | Machine-to-Machine | Comunicação entre cargas; OAuth client credentials para cliente confidencial nos cenários pertinentes, sem autorização delegada de usuário. Outros mecanismos exigem justificativa e suporte [65] |
| MAP | Microsoft Architecture Plan, rótulo usado no guia | Template de intake citado em §6.2, a formalizar pela organização; não norma/certificação Microsoft |
| MAU | Monthly Active Users | Usuários ativos mensais; não equivale a requisições por segundo ou capacidade |
| MCP | Model Context Protocol | Contrato de integração de ferramentas com agentes; versão/autenticação e gateway precisam de validação |
| MPAS | Microsoft Partner Architectural Solution, rótulo usado no guia | Modelo citado de avaliação inicial; a organização formaliza conteúdo e aprovação, sem presumir programa oficial obrigatório |
| mTLS | Mutual TLS | Autenticação cliente+servidor por certificado |
| NIST CSF 2.0 | National Institute of Standards and Technology, Cybersecurity Framework 2.0 | Framework de resultados de cibersegurança; não prescreve um único mecanismo nem certifica a organização. Correspondências usam edição, código válido, resultado e evidência do Core 2.0 (Apêndice F) [68] |
| OAuth 2.0 | OAuth 2.0 Authorization Framework | Autorização conforme RFC 6749, com práticas atuais de segurança da RFC 9700; BCP nesse contexto significa Best Current Practice [65] |
| OIDC | OpenID Connect | Camada de identidade sobre OAuth 2.0 para cenários de usuário; não transforma toda chamada M2M em fluxo de usuário |
| OST | Online Services Terms | Termos contratuais Microsoft para serviços online |
| OWASP | Open Web Application Security Project | Referência em segurança de aplicações |
| PCI-DSS | Payment Card Industry Data Security Standard | Padrão de segurança de dados de cartões de pagamento |
| PII | Personally Identifiable Information | Expressão técnica inglesa; para enquadramento brasileiro, usar a definição de dado pessoal da LGPD, incluindo contexto de identificabilidade [66] |
| PIM | Privileged Identity Management | Gestão de acesso privilegiado no Microsoft Entra ID; não substitui permissão RBAC nem autorização da API |
| PKCE | Proof Key for Code Exchange | Proteção da troca do código de autorização, relevante ao fluxo Authorization Code delegado por usuário; não é o fluxo client credentials M2M |
| RACI | Responsible / Accountable / Consulted / Informed | Matriz de responsabilidades; A deve ser único |
| RBAC | Role-Based Access Control | Azure RBAC |
| RIPD | Relatório de Impacto à Proteção de Dados Pessoais | Obrigatório quando determinado pela ANPD ou por requisito aplicável; avaliação de risco/requisito corporativo adicional têm fundamento próprio. Vincular a operações de tratamento, sem presumir um relatório legalmente obrigatório por API [66] |
| RPO | Recovery Point Objective | Perda máxima-alvo de estado/dados, definida e medida por objeto: configuração, credenciais, transações ou evidência; não inferida do tier |
| RTO | Recovery Time Objective | Tempo-alvo até retomada com critério de serviço/objeto, início e fim de medição, dependências e prova acordados |
| SIEM | Security Information and Event Management | Ex.: Microsoft Sentinel |
| SKU | Stock Keeping Unit | Identificador da oferta comercial; tier, runtime, região e modelo de adoção são decisões relacionadas, mas distintas |
| SLA | Service Level Agreement | Acordo de nível de serviço com escopo e compromissos; SLA do fornecedor não cobre automaticamente a jornada de negócio |
| SLO | Service Level Objective | Objetivo operacional medido por indicador/janela e fronteira do serviço, não garantia do tier |
| SOC | Security Operations Center | Centro de operações de segurança, com fontes cobertas e alternativas de detecção definidas |
| SoD | Segregation of Duties | Separação de funções e impedimento de autoaprovação em mudança crítica, comprovados para os atores reais antes da primeira implantação crítica |
| TLS | Transport Layer Security | Segurança da comunicação em trânsito |
| TPS | Transactions Per Second | Transações por segundo |
| VNet | Virtual Network | Azure |
| WAF | Web Application Firewall | Application Gateway / Front Door |
| WORM | Write Once, Read Many | Proteção imutável configurada por objeto/prazo ou legal hold, quando exigida. Não cria hipótese legal, prazo universal de retenção ou permissão para coleta excessiva [39][66] |

---

## Apêndice F: resultados NIST CSF 2.0, práticas APIM e evidências de controle

O **NIST Cybersecurity Framework (CSF) 2.0**, publicado no CSWP 29 de 26/02/2024, descreve resultados, não prescreve uma implementação única. A tabela explicita o resultado do Core e a evidência necessária para a contribuição do APIM; um serviço habilitado não prova que o resultado foi alcançado [68].

ISO/IEC 27001:2022 trata do sistema de gestão de segurança da informação; COBIT 2019, de governança e gestão de tecnologia; COSO 2013, de controle interno. Os identificadores ISO e COBIT abaixo são **candidatos do inventário de controle**, não títulos/requisitos integralmente validados nem equivalências automáticas. O Anexo A ISO não foi integralmente acessível nesta verificação; não se deduz erro ou inexistência de controle a partir desse limite. A orientação pública ISACA [48] não substitui os objetivos licenciados completos. Os componentes COSO [49] organizam a contribuição, mas uma avaliação de efetividade exige princípios, escopo e evidências próprios. Confirmar essas correspondências com os exemplares aplicáveis antes de usá-las como prova de auditoria.

| Prática e limite de escopo | Identificador ISO candidato | Resultado NIST CSF 2.0 [68] | COBIT 2019 candidato | Componente COSO 2013 | Evidência necessária |
|---|---|---|---|---|---|
| Receber inteligência de ameaças e incorporá-la ao cenário | A.5.7 | ID.RA-02: inteligência recebida de fóruns/fontes | APO12 / DSS05 | Avaliação de riscos | Fonte, data, avaliação e ação no modelo de §7.6; mera habilitação de Defender não demonstra recepção/uso |
| Definir e revisar acesso ao gateway e ao domínio | A.5.15 | PR.AA-05: permissões definidas, aplicadas e revisadas, com menor privilégio e segregação | DSS05 / APO13 | Atividades de controle | Perfis, decisões e testes de cliente/permissão/objeto; incluir autorização no backend (§7.3) |
| Provisionar, manter e revogar identidades e direitos | A.5.18 | PR.AA-01: identidades/credenciais geridas; PR.AA-05: direitos geridos/revisados | DSS06 / DSS05 | Atividades de controle | Inventário, concessão/revogação e revisão RBAC/consumidores, não somente lista de roles |
| Gerir fornecedores de nuvem e requisitos contratuais | A.5.23 | GV.SC-04: fornecedores conhecidos/priorizados; GV.SC-05: requisitos de risco integrados aos contratos | APO10 / APO13 | Ambiente de controle | Criticidade do fornecedor, due diligence, cláusulas e acesso/saída; aplicar o ato BCB/CMN pertinente (§2.3) |
| Preparar mecanismos de continuidade | A.5.30 | PR.IR-03: mecanismos para requisitos de resiliência em situações normais/adversas | DSS04 | Avaliação de riscos | Dependências, objetivos e ensaio de recuperação (§10.5/§10.6); prontidão não é execução da recuperação de incidente |
| Verificar identidade em acesso remoto, contribuição parcial | A.6.7, parcial | PR.AA-04: asserções de identidade protegidas, transmitidas e verificadas | DSS05 | Atividades de controle | Proteção do token e transporte; testes de issuer/audience/expiração. PIM/Entra não demonstram sozinhos controles do posto remoto |
| Administrar privilégio e segregação de funções | A.8.2 | PR.AA-05: menor privilégio e segregação na gestão de permissões | DSS05 | Atividades de controle | Escopo RBAC, ativação PIM quando aplicável, aprovador e revisão; app role não equivale a ARM/PIM |
| Autenticar usuários e serviços | A.8.5 | PR.AA-03: usuários, serviços e hardware autenticados | DSS05 | Atividades de controle | Perfil por consumidor, token/certificado e testes negativos (§7.2/apêndice A); não exigir o mesmo fluxo de todos |
| Gerir configuração | A.8.9 | PR.PS-01: práticas de gestão de configuração estabelecidas/aplicadas | BAI10 | Atividades de controle | Baseline versionada, parâmetros efetivos, implantação e comparação de drift com falhas distinguíveis (§9) |
| Gerar logs disponíveis para monitoramento | A.8.15 | PR.PS-04: registros gerados e disponibilizados para monitoramento contínuo | DSS01 / MEA03 | Informação e comunicação | Fonte/campos/destino, teste de emissão/consulta, coleta mínima e retenção; correlação multifuente é outra contribuição, DE.AE-03 |
| Monitorar rede e runtime da API | A.8.16 | DE.CM-01: rede/serviços de rede monitorados; DE.CM-09: software/runtime/dados monitorados | DSS01 / DSS05 | Monitoramento | Consultas, alertas, cobertura de APIs/gateways e teste de detecção; documentar exclusões de workspace/self-hosted do Defender |
| Restringir acesso lógico à rede | A.8.20 | PR.IR-01: redes/ambientes protegidos de acesso lógico e uso não autorizados | DSS05 | Atividades de controle | Rotas/DNS/regras, identidade e testes de bypass; nome de private endpoint não comprova a restrição |
| Proteger os serviços de rede no caminho aprovado | A.8.21 | PR.IR-01: proteção de redes/ambientes contra acesso/uso não autorizados | DSS05 | Atividades de controle | Configuração e testes de ingress/egress e origem; indicar cada terminação TLS |
| Proteger o ingresso HTTP por WAF; distinto de filtragem de navegação | A.8.23: correspondência não demonstrada | PR.IR-01: contribuição do ingresso contra uso lógico não autorizado; não equivalência com todo controle de web filtering | DSS05 | Atividades de controle | Regras, requisições permitidas/negadas e restrição de origem. WAF de entrada não comprova filtragem de navegação nem preservação do certificado original |
| Proteger dados em trânsito | A.8.24, contribuição parcial | PR.DS-02: confidencialidade, integridade e disponibilidade de dados em trânsito protegidas | DSS05 | Atividades de controle | TLS por salto, gestão de chaves/certificados e teste do vínculo quando aplicável; citar Key Vault não demonstra o resultado |
| Integrar requisitos de segurança ao desenvolvimento | A.8.26 | PR.PS-06: práticas de desenvolvimento seguro integradas e monitoradas ao longo do ciclo | BAI03 / APO13 | Atividades de controle | Requisitos, responsáveis, threat model e evidência dos gates por perfil (§7.5/§7.7/§9) |
| Aplicar práticas de implementação segura | A.8.28 | PR.PS-06: práticas de desenvolvimento seguro no ciclo de vida | BAI03 | Atividades de controle | Revisão, verificações estáticas e testes das expressões/contratos; lint XML sozinho não comprova execução |
| Testar segurança antes da aceitação | A.8.29 | PR.PS-06: desenvolvimento seguro integrado/monitorado | BAI07 | Atividades de controle e monitoramento | Fixtures e resultados positivos/negativos do gateway/backend escolhido; what-if não executa policy |
| Separar configurações e acessos de ambientes | A.8.31 | PR.PS-01: configuração gerida; PR.AA-05: permissões/segregação | BAI07 / DSS06 | Atividades de controle | Identidades, segredos, rede e parâmetros por ambiente; teste de impossibilidade de acesso/implantação não autorizados |
| Controlar mudanças de configuração | A.8.32 | PR.PS-01: gestão de configuração estabelecida/aplicada | BAI06 | Atividades de controle; informação e comunicação | PR, revisão, controles efetivos de aprovação, artefato e registro da mudança; validar semântica real do mecanismo usado |
| Proteger a execução de auditoria/testes | A.8.34: correspondência parcial a validar | Sem equivalência direta proposta; PR.AA-05 contribui para permissões e PR.PS-04 para a trilha, não para todo o escopo de auditoria | MEA03 | Monitoramento | Autorização, escopo, contas, janela, limites, proteção dos dados e recuperação; PR.PS-02 é manutenção/substituição/remoção de software, não este controle |
| Segregar autor e aprovador de mudança crítica | Sem identificador adicional proposto | GV.RR-02: papéis/autoridades estabelecidos e aplicados; PR.AA-05: segregação e menor privilégio | DSS06.03 / BAI06 | Atividades de controle | Responsabilidade definida e prova dos impedimentos/aprovações efetivos, não apenas uma lista de revisores |
| Rastrear mudanças e revisar desempenho do controle | Sem identificador adicional proposto | PR.PS-01: mudança de configuração; GV.OV-03: desempenho de gestão de risco avaliado/revisto para ajustes | BAI06 / MEA02 | Informação e comunicação | Trilha PR–artefato–implantação; para GV.OV-03, avaliação periódica e decisões de ajuste, não apenas log de deploy |
| Preservar evidência de auditoria necessária | Sem identificador adicional proposto | PR.DS-01: dados em repouso protegidos; PR.PS-04: logs disponíveis para monitoramento | DSS06 / MEA03 | Monitoramento | WORM/hold, acesso, recuperação e política por conjunto/finalidade (§10.7); imutabilidade não cria base legal ou conformidade integral |

> No sistema de governança, riscos e conformidade (GRC), registrar edição, resultado, alcance, responsável, evidência e data do teste/revisão. Para correspondências parciais, identificar a parcela coberta e as dependências fora do APIM. O Core CSF 2.0 não contém a categoria PR.PT; PR.PS-02 trata do ciclo de manutenção de software. Códigos válidos sem relação demonstrada, nomes de serviços ou uma tabela de associação não comprovam implementação nem certificação [68].

---

## Apêndice G: síntese executiva de adoção, controles e prontidão

Data-base documental: **09/10/2026**. As decisões abaixo dependem dos requisitos e evidências do cliente; não são autorização de implantação.

### Objetivo do APIM corporativo

Azure API Management (APIM) governa a entrada e o consumo de APIs, com autenticação, autorização de perímetro, limites de tráfego e operação. Residência e conformidade dependem também de backend, identidade, catálogo, telemetria, acessos e contrato. O modelo de adoção, o tier/SKU (oferta comercial), o gateway de execução e a região são escolhidos separadamente. Neste resumo, WAF é firewall de aplicações web; SLO é objetivo operacional; mTLS é autenticação TLS mútua; FAPI é um perfil de segurança aplicável quando exigido ou explicitamente adotado, não obrigação genérica da LGPD ou do PCI DSS.

### Critérios de decisão (5 perguntas, §1.1)

1. **Localização restrita ao Brasil?** Confirmar o conjunto de dados/metadados e as regiões permitidas. Brazil South lista Basic v2/Standard v2; não lista Premium v2 ou workspace gateway na consulta de 09/10/2026. Residência isolada não exige Premium clássico. Standard v2 tem integração de saída à rede virtual (VNet) e suporte ao private endpoint; injection obrigatória e região podem justificar Premium clássico [2][3][15].
2. **Dados/regimes e consumidor?** Compor exposição, classificação legal/corporativa, setor e criticidade. Dado financeiro não é automaticamente sensível LGPD. Para acesso máquina a máquina pertinente, OAuth client credentials; para usuário delegado, Authorization Code com proteção da troca do código por Proof Key for Code Exchange (PKCE). FAPI/mTLS seguem o perfil aplicável, sem equivalência genérica entre LGPD, PCI e Open Finance. Retenção segue política aprovada por conjunto/finalidade [65][66].
3. **Entrada pela internet?** WAF e origem protegida são política corporativa. Selecionar borda global/regional e o ponto de autenticação mTLS; o certificado consumidor não atravessa automaticamente Application Gateway até APIM [19].
4. **Separação de operação, falhas ou SLO necessária?** Avaliar workspace, gateway separado ou instância. Dedicada somente se o isolamento exigido e equipe/runbook/plantão a justificarem; autonomia isolada não impõe instância (§4.5).
5. **Multirregião nativa/self-hosted de produção necessários?** Premium clássico é a opção documentada. Alto volume requer teste e escala, não seleção automática. Recuperação pode exigir outras instâncias e dependências; metas precisam de prova (§10.5) [1][7][54].

### Recomendação

**Modelo C, híbrido governado**, é a hipótese para plataforma central madura com exceções operáveis, não escolha universal. Comparar configurações permitidas e calibrar custos, isolamento, equipes e continuidade. A matriz ilustrativa resulta em 47 pontos para o modelo corporativo compartilhado (A), 37 para o modelo por domínio (B) e 50 para o híbrido governado (C), de um máximo de 63. A vantagem de três pontos do híbrido sobre o corporativo compartilhado muda sob prioridades plausíveis. Registrar requisitos, evidências e condições de mudança em Registro de Decisão Arquitetural (ADR), sem usar o ranking como prova do contexto do cliente (§4.5).

### Entregas e condições de prontidão

Prazos iniciais de planejamento, a ajustar pelo cliente; datas não autorizam publicação nem substituem requisitos anteriores:

1. ADR de adoção, tier, execução, região/rede e continuidade, com responsáveis e evidências antes do compromisso arquitetural (referência: 30 dias).
2. Infraestrutura como código (IaC), guardrails e fontes de observabilidade/detecção compatíveis com os gateways (referência: 60 dias); provisionar ambiente não significa autorizar APIs produtivas.
3. Automação do ciclo de APIs (APIOps) com contratos por protocolo, classificação composta, controles e promoção comprovada. Separação de funções (SoD) e impedimento de autoaprovação ativos **antes da primeira implantação crítica**, ainda que o plano de automação tenha referência de 90 dias.
4. Modelo de ameaças e aceite dos controles para a composição dados/setor/exposição/consumidor, antes da API em produção (referência de programa: 90 dias).
5. Recuperação acordada comprovada antes do go-live crítico, ou apenas aceite de risco permitido e registrado. O marco de 180 dias refere-se ao programa de ensaios continuados, não à primeira prova necessária.
6. Meta inicial de autosserviço ≥70% das **mudanças elegíveis implantadas**, no período aprovado, estratificadas por risco, com experiência do desenvolvedor e tempos de aprovação medidos (referência: 240 dias). Não medir PRs como sinônimo de mudança nem reduzir controle obrigatório para atingir a meta.

### Principais custos e riscos

- **Custo:** dimensionar tier/unidades/regiões, borda, telemetria, recuperação e tráfego com oferta/preço efetivos. Defender usa plano/franquia e chamadas monitoradas por subscrição, não cobrança simples por API. Sem extrato/proposta, orçamento permanece pendente (§8.6).
- **Segurança e dados:** workspaces e self-hosted não têm cobertura Defender; comprovar alternativas. Validar mTLS no ponto de terminação e binding quando o perfil exigir. Aprovar catálogo/telemetria/acessos internacionais e política de retenção por conjunto/finalidade, sem piso universal ou dispensa baseada só em "não sensível".
- **Execução:** plataforma e domínios precisam de autosserviço governado e operação compatível; percentual inferior à meta exige analisar mix de risco, tempos e causas, não presumir governança inadequada.
- **Auditoria:** impedir autoaprovação pelos atores reais. CODEOWNERS, proteção de branches e aprovação de environment têm papéis distintos; uma lista de required reviewers do GitHub não implementa, sozinha, aprovação cumulativa de todos os papéis (§11.6.1).

### Decisões pendentes do cliente (§14)

Antes de comprometer tier/topologia, aprovar requisitos de localização, exposição, dados/regimes, capacidade e continuidade. Antes da primeira API em produção, aprovar contrato/identidade, destinos e coleta, política de retenção por conjunto/finalidade, cobertura ou detecção alternativa, gates e autoridade de exceção. Para serviço crítico, comprovar recuperação com tempo-alvo de retomada (RTO) e perda máxima-alvo (RPO) por objeto e ativar SoD antes da primeira implantação crítica; aceite interno não afasta obrigação externa. FinOps, operação e provas podem ser preparados em paralelo, mas precisam estar prontos no marco pertinente. Estratégia futura de Premium v2 e chargeback pode ser faseada se não bloquear requisito atual; migração depende de suporte funcional, região, continuidade, custo e testes, não só de oferta regional.
