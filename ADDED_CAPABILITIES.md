# Inventário de Capacidades Adicionadas (Antigravity)

Este arquivo rastreia todas as extensões, agentes e skills adicionados ao ambiente a partir de repositórios externos para garantir rastreabilidade e facilitar a reversão futura.

## 📁 Repositório Global de Skills
- **Caminho:** `C:\Users\Robson\Skills`
- **Configuração Master-Skill:** `C:\Users\Robson\.gemini\antigravity\skills\master-skill\config\settings.json`

## 🚀 Skills Instaladas (tech-leads-club/agent-skills)

As seguintes skills foram selecionadas por serem complementares e não conflitantes com as capacidades nativas:

| Categoria | Skill | Função Principal | Status |
| :--- | :--- | :--- | :--- |
| **(tooling)** | `mermaid-studio` | Criação e renderização de diagramas Mermaid. | ✅ Instalada |
| **(tooling)** | `excalidraw-studio` | Desenho e prototipação com Excalidraw. | ✅ Instalada |
| **(cloud)** | `aws-advisor` | Consultoria especializada em arquitetura AWS. | ✅ Instalada |
| **(cloud)** | `google-cloud-advisor` | Consultoria especializada em Google Cloud. | ✅ Instalada |
| **(cloud)** | `azure-advisor` | Consultoria especializada em Microsoft Azure. | ✅ Instalada |
| **(development)** | `nestjs-modular-monolith` | Padrões para arquitetura modular no NestJS. | ✅ Instalada |
| **(development)** | `shopify-developer` | Desenvolvimento especializado para Shopify. | ✅ Instalada |
| **(tooling)** | `confluence-assistant` | Integração e gestão de docs no Confluence. | ✅ Instalada |
| **(tooling)** | `jira-assistant` | Gestão de tickets e workflow no Jira. | ✅ Instalada |
| **(development)** | `tlc-spec-driven` | Desenvolvimento orientado a especificações. | ✅ Instalada |
| **(architecture)** | `frontend-blueprint` | Blueprints de arquitetura frontend moderna. | ✅ Instalada |
| **(gtm)** | `solo-founder-gtm` | Estratégias de Go-to-Market para fundadores. | ✅ Instalada |

## 🛠️ Como Usar
- As skills estão ativas e podem ser chamadas diretamente ou via `/master-skill`.
- Exemplo: `/master-skill quero a skill de mermaid-studio`.
- Elas estão localizadas fisicamente em `c:\Users\Robson\Projetos\.agent\skills\`.

## 🗑️ Como Desfazer / Remover
Para remover as capacidades adicionadas e retornar ao estado original:
1. **Remover do Projeto:** Exclua as pastas correspondentes em `c:\Users\Robson\Projetos\.agent\skills`.
2. **Remover Globalmente:** Exclua a pasta `C:\Users\Robson\Skills`.
3. **Limpar Registro:** Delete este arquivo (`ADDED_CAPABILITIES.md`).

---
*Assinado: Antigravity AI*
