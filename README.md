# APIM Playbook: guia de Azure API Management

[![ORCID](https://img.shields.io/badge/ORCID-0009--0006--0765--4201-A6CE39?logo=orcid&logoColor=white)](https://orcid.org/0009-0006-0765-4201)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](LICENSE)
[![Azure](https://img.shields.io/badge/Cloud-Azure-0078D4?logo=microsoftazure&logoColor=white)](#)
[![Last commit](https://img.shields.io/github/last-commit/EdneiMonteiro/apim-playbook)](https://github.com/EdneiMonteiro/apim-playbook/commits)

Guia técnico e agnóstico de **Azure API Management (APIM)** para decisões de arquitetura, segurança, conformidade, implantação, operação e governança de uma plataforma corporativa de APIs.

> Este repositório reúne material de referência aberto para estudo, desenho arquitetural e governança técnica. Os exemplos e modelos são ilustrativos, não contêm dados de organizações ou clientes e não estão prontos para uso direto em produção sem validação prévia.
> Consulte [`DISCLAIMER.md`](./DISCLAIMER.md) e [`SUPPORT.md`](./SUPPORT.md).

---

## Documentos do repositório

| Arquivo | Descrição |
| --- | --- |
| [`playbook.md`](./playbook.md) | Guia técnico completo de decisão, arquitetura, segurança, APIOps, operação e governança de APIM |
| [`DISCLAIMER.md`](./DISCLAIMER.md) | Aviso legal, limites de uso, isenção de garantias e recomendações para ambientes corporativos |
| [`SUPPORT.md`](./SUPPORT.md) | Orientações para abertura de *issues*, relato de problemas e sugestões de melhoria |
| [`CITATION.cff`](./CITATION.cff) | Metadados padronizados para citação técnica e acadêmica do material |
| [`LICENSE`](./LICENSE) | Licença Creative Commons Attribution 4.0 International (CC BY 4.0) |

---

## Estrutura do [`playbook.md`](./playbook.md)

### Capítulos principais

1. [Sumário executivo](./playbook.md#1-sumário-executivo) — quadro de decisão em 5 perguntas e disponibilidade regional no Brasil
2. [Contexto e problema](./playbook.md#2-contexto-e-problema) — premissas, perfis orientados e requisitos regulatórios (LGPD, BCB/CMN, Open Finance, PCI DSS)
3. [Planos, tiers clássicos, Consumption, v2 e regiões](./playbook.md#3-azure-api-management-planos-tiers-clássicos-consumption-v2-e-regiões) — comparação de SKUs, *workspaces*, *gateways* e oferta em `Brazil South`
4. [Modelos de adoção com matriz de decisão](./playbook.md#4-modelos-de-adoção-corporativo-por-subscriçãodomínio-híbrido-com-matriz-de-decisão) — compartilhado (A), por domínio (B), híbrido governado (C) e *self-hosted gateway*
5. [Arquitetura de referência](./playbook.md#5-arquitetura-de-referência) — integração com WAF (Front Door / Application Gateway), Entra ID, Key Vault, Defender for APIs e API Center
6. [Padrões de API, ciclo de vida, versionamento, catálogo e consumo](./playbook.md#6-padrões-de-api-ciclo-de-vida-versionamento-catálogo-e-consumo) — intake, OpenAPI, GraphQL, WebSocket, gRPC e AI/MCP Gateway
7. [Segurança, identidade e políticas](./playbook.md#7-segurança-identidade-e-políticas) — OAuth 2.0/OIDC, mTLS, BOLA/BFLA, modelo de ameaças STRIDE + OWASP API Top 10 2023 e composição de perfis
8. [Plataforma, landing zone, rede e guardrails](./playbook.md#8-plataforma-landing-zone-rede-e-guardrails) — Azure Policy, topologia de rede/DNS, mTLS com proxy L7 e FinOps (showback/chargeback)
9. [Pipeline de publicação com IaC e APIOps](./playbook.md#9-pipeline-de-publicação-com-infraestrutura-como-código-e-apiops) — promoção imutável, validações de PR, detecção de *drift* e *rollback*
10. [Operação, observabilidade e SLO/SLA](./playbook.md#10-operação-observabilidade-e-objetivos-e-acordos-de-nível-de-serviço) — métricas por geração de gateway, alertas, HA/DR, *runbooks* e retenção/WORM
11. [Governança corporativa, responsabilidades e DX](./playbook.md#11-governança-corporativa-responsabilidades-e-experiência-do-desenvolvedor) — papéis, matriz RACI, jornada do consumidor, exceções e Segregação de Funções (SoD)
12. [Práticas inadequadas e boas práticas](./playbook.md#12-práticas-inadequadas-e-boas-práticas) — antipadrões recorrentes e correções recomendadas
13. [Recomendações finais e plano de implantação](./playbook.md#13-recomendações-finais-e-plano-de-implantação-sugerido) — roteiro sugerido por marcos e critérios de aceite
14. [Decisões pendentes que dependem da organização](./playbook.md#14-decisões-pendentes-que-dependem-do-cliente) — *checklist* decisório pré-arquitetura e pré-produção
15. [Referências consolidadas](./playbook.md#15-referências-consolidadas) — documentação oficial Microsoft Learn, RFCs IETF, OWASP, NIST e normativos

### Apêndices práticos

- **Apêndice A:** [Exemplos de *policies* XML por perfil de consumidor](./playbook.md#apêndice-a-exemplos-de-policies-xml-por-perfil-de-consumidor-77) (M2M interno, B2B, dados pessoais/LGPD com HMAC, mTLS/RFC 8705, fachada administrativa e GET público)
- **Apêndice B:** [Modelo OpenAPI 3.0.3 com extensão `x-governance`](./playbook.md#apêndice-b-modelo-openapi-para-contrato-httprest-e-metadados-de-governança)
- **Apêndice C:** [Modelo de Registro de Decisão Arquitetural (ADR)](./playbook.md#apêndice-c-modelo-de-registro-de-decisão-arquitetural-adr)
- **Apêndice D:** [Fragmento de promoção e verificação de *drift* (GitHub Actions)](./playbook.md#apêndice-d-fragmento-de-promoção-e-verificação-de-drift-github-actions)
- **Apêndice E:** [Glossário técnico e regulatório](./playbook.md#apêndice-e-glossário)
- **Apêndice F:** [Mapeamento NIST CSF 2.0, ISO/IEC 27001, COBIT 2019 e COSO 2013](./playbook.md#apêndice-f-resultados-nist-csf-20-práticas-apim-e-evidências-de-controle)
- **Apêndice G:** [Síntese executiva de adoção, controles e prontidão](./playbook.md#apêndice-g-síntese-executiva-de-adoção-controles-e-prontidão)

