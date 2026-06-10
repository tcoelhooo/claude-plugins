# PTA Plugins — Marketplace interno do Claude

Repositório de plugins internos da Personal Trainer Academy para o Claude (Claude Code / Cowork).

Hoje contém **1 plugin**:

| Plugin | O que faz |
|--------|-----------|
| `contratos-pta` | Analista de contratos: varre a Autentique e gera o painel de assinaturas (macro mensal, semanal e turma a turma). |

---

## Estrutura

```
plugin-marketplace/                      ← este repositório (a raiz do marketplace)
├── .claude-plugin/
│   └── marketplace.json                 ← catálogo de plugins
└── plugins/
    └── contratos-pta/
        ├── .claude-plugin/
        │   └── plugin.json       ← manifesto + pedido do token
        └── skills/
            └── analista-contratos/
                ├── SKILL.md      ← a inteligência: como falar com a Autentique e montar o painel
                └── painel.html   ← layout do painel (opção 1)
```

> O plugin é uma **skill** (instruções) + o layout do painel. Não há código embutido —
> o próprio Claude executa a análise conectando na API da Autentique com o token configurado.

---

## 1. Testar localmente (antes de publicar)

No Claude (Code ou Cowork), aponte o marketplace para esta pasta e instale:

```
/plugin marketplace add ./plugin-marketplace
/plugin install contratos-pta@pta-plugins
```

Ao habilitar, o Claude pede o **Token da Autentique** — cole e ele guarda cifrado no keychain da máquina.
Depois é só pedir em linguagem natural, ex.: *"gera o painel de contratos"*.

> Para teste por linha de comando (sem instalar o plugin), defina o token no ambiente e rode o script direto:
> ```
> AUTENTIQUE_TOKEN=seu_token node plugins/contratos-pta/skills/analista-contratos/lib/gerarPainel.js
> ```

---

## 2. Publicar no GitHub (privado)

1. Crie um repositório **privado** na organização da PTA, ex.: `pta-digital/claude-plugins`.
2. Suba o conteúdo desta pasta (`plugin-marketplace/`) como raiz do repositório:
   ```
   cd plugin-marketplace
   git init
   git add .
   git commit -m "feat: plugin contratos-pta"
   git branch -M main
   git remote add origin git@github.com:pta-digital/claude-plugins.git
   git push -u origin main
   ```
3. Dê acesso de **leitura** ao repositório para os funcionários (membros da org no GitHub).

---

## 3. Como os funcionários instalam

Cada pessoa, uma única vez:

```
/plugin marketplace add pta-digital/claude-plugins
/plugin install contratos-pta@pta-plugins
```

O Git usa o login GitHub de cada um para autenticar — quem não tem acesso ao repo privado não consegue instalar. Na ativação, o Claude pede o token da Autentique de cada um.

Pré-requisito na máquina do funcionário: **Node.js 18+**.

---

## 4. (Opcional) Pré-instalar para toda a empresa

Para o plugin já aparecer instalado sem ninguém digitar comando, um admin adiciona ao `managed-settings.json` distribuído às máquinas:

```json
{
  "extraKnownMarketplaces": {
    "pta-plugins": {
      "source": { "source": "github", "repo": "pta-digital/claude-plugins" }
    }
  },
  "enabledPlugins": {
    "contratos-pta@pta-plugins": true
  }
}
```

Cada funcionário ainda informa o **próprio token** na primeira ativação (a credencial nunca vai no repositório).

---

## 5. Atualizar o plugin

Ao mudar o código, suba uma nova versão:

1. Aumente o `version` em `plugins/contratos-pta/.claude-plugin/plugin.json` (ex.: `1.0.0` → `1.1.0`).
2. `git commit` + `git push`.
3. Os funcionários atualizam com:
   ```
   /plugin marketplace update pta-plugins
   /plugin update contratos-pta@pta-plugins
   ```

> Se você **não** subir a versão, os funcionários não recebem a atualização — o número em `version` é o gatilho.
