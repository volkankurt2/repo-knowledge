# Repo Knowledge — Claude Entry Point

## Single Source
Tüm agent ve skill dosyaları `.rk/` klasöründedir:

| Tür | Konum |
|-----|-------|
| Agent'lar | `.rk/agents/` |
| Skill'ler | `.rk/skills/` |

## Routing

**Repo URL'si veya "X'i tara"** → `.rk/agents/repo-scanner.agent.md` dosyasını oku ve uygula

**"Haritayı güncelle" / "hangi repo hangisini kullanıyor"** → `.rk/agents/ecosystem-mapper.agent.md` dosyasını oku ve uygula

**Geliştirici sorusu / Jira task / etki analizi** → `.rk/agents/dev-advisor.agent.md` dosyasını oku ve uygula

## Kullanılabilir Komutlar
- `/rk-scan` — Repo tara
- `/rk-map` — Ecosystem haritası oluştur
- `/rk-advise` — Geliştirici sorusunu yanıtla

## İlerleme
`kb/_progress.json` — tüm agentlar kendi durumlarını burada günceller
