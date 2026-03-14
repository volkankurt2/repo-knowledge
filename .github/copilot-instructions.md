# Repo Knowledge — GitHub Copilot Instructions

## Single Source
Tüm agent ve skill dosyaları `.rk/` klasöründedir:

| Tür | Konum |
|-----|-------|
| Agent'lar | `.rk/agents/` |
| Skill'ler | `.rk/skills/` |

## Routing

Kullanıcı bir repo tarama talebi getirdiğinde:
→ `.rk/agents/repo-scanner.agent.md` dosyasını oku, içindeki workflow'u adım adım uygula

Kullanıcı "hangi repo hangisini kullanıyor" veya ecosystem haritası istediğinde:
→ `.rk/agents/ecosystem-mapper.agent.md` dosyasını oku ve uygula

Kullanıcı bir geliştirici sorusu, Jira task veya "nereye yazarım" sorusu sorduğunda:
→ `.rk/agents/dev-advisor.agent.md` dosyasını oku ve uygula

## Araç Kullanımı
- Repo kaynak koduna eriş: GitLab extension, Azure DevOps extension veya `@terminal` ile API çağrısı
- Dosya oku/yaz: workspace dosya sistemi erişimi

## İlerleme
`kb/_progress.json` — tarama durumunu takip eder
