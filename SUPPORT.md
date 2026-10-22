# Suporte

Este documento orienta como relatar problemas, sugerir melhorias ou esclarecer dúvidas sobre o conteúdo deste repositório.

## Como obter ajuda ou contribuir

Utilize as [GitHub Issues](https://github.com/EdneiMonteiro/apim-playbook/issues) para:

- Relatar erros técnicos, links desatualizados ou divergências em relação à documentação oficial do Azure API Management;
- Sugerir melhorias em seções do [`playbook.md`](./playbook.md), exemplos de *policies* XML, modelos OpenAPI, templates de ADR ou fluxos de APIOps;
- Apontar atualizações em normas técnicas (RFCs, OWASP, NIST) ou regulatórias citadas no guia.

Antes de abrir uma *issue*:

1. Verifique se já existe uma *issue* aberta ou fechada sobre o mesmo tópico;
2. Consulte o [`README.md`](./README.md), o sumário do [`playbook.md`](./playbook.md) e o [`DISCLAIMER.md`](./DISCLAIMER.md).

## O que incluir na solicitação

Para agilizar a análise e manter o padrão de rastreabilidade do guia, informe:

- **Localização exata:** arquivo, seção (`§`) ou apêndice a que a dúvida ou sugestão se refere (por exemplo, `playbook.md` — `§3.4` ou `Apêndice A.4`);
- **Contexto técnico:** tier/SKU (clássico, v2, Consumption), tipo de *gateway* (gerenciado, *workspace gateway*, *self-hosted*), protocolo ou perfil aplicável;
- **Descrição objetiva:** trecho atual, problema identificado e redação ou correção sugerida;
- **Fonte verificável:** quando se tratar de capacidade do serviço, limite técnico ou requisito normativo, inclua a URL da fonte oficial (Microsoft Learn, RFC Editor, OWASP, NIST ou órgão regulador) e a data da consulta, seguindo o padrão da `§15` do [`playbook.md`](./playbook.md).

## Segurança e sanitização em *issues* públicas

Este é um repositório **público**. Ao abrir *issues*, comentar ou enviar *pull requests*, **nunca publique**:

- Tokens de acesso (JWT/OAuth), *subscription keys*, segredos, senhas, chaves privadas ou certificados reais;
- Identificadores reais de ambientes ou clientes (`tenantId`, `subscriptionId`, nomes de organizações, domínios internos, faixas de IP privadas ou topologias corporativas);
- Dados pessoais (LGPD), informações protegidas por sigilo legal/contratual ou *payloads* e logs extraídos de produção;
- Detalhes de vulnerabilidades ativas ou relatórios de auditoria interna de ambientes reais.

Utilize sempre valores fictícios e *placeholders* sanitizados (como `empresa.com`, `apim-acme-prd-bs-01` ou GUIDs zerados) ao exemplificar cenários.

## Nível de suporte

Este projeto é mantido em caráter referencial e **não possui acordo de nível de serviço (SLA)**. Não há garantia de prazo de resposta, análise de cenários específicos de clientes, correção de erros ou publicação de novas versões.

## Suporte oficial e ambientes de produção

Este repositório **não substitui o suporte oficial da Microsoft** nem os canais internos de governança, operações ou resposta a incidentes (SOC/CSIRT/ITSM) da sua organização.

Para incidentes em ambientes produtivos, dúvidas contratuais ou indisponibilidade de recursos Azure, acione diretamente os canais oficiais de suporte no Portal do Azure e os procedimentos operacionais internos da sua instituição.

