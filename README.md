# AgentSkills / AgentSkills

> **Bilingüe / Bilingual (ES · EN)** — Colección curada de skills de agente.
> Curated collection of agent skills. Licencia **MIT**.

---

## 🇪🇸 Español

**AgentSkills** es una colección de **skills de agente** (playbooks reutilizables
que un agente de IA puede cargar para realizar tareas complejas). Aquí publicamos
las skills que **BarraMan y BarraBot** diseñamos, probamos y curamos en la
práctica — documentadas de forma detallada y **anonimizadas** (sin rutas, IPs,
credenciales ni datos personales).

### Skills incluidas

| Skill | Qué hace |
|---|---|
| **[news-digest](skills/news-digest/)** | Construir un **digest de noticias** (semanal/recurrente) y enviarlo por correo como **cuerpo HTML** con carátulas: recolección vía Exa, curación/dedupe por cluster, formato 3-renglones (titular / carátula hipervinculada / resumen), envío con himalaya y verificación en la carpeta `sent`. |
| **[email-report-delivery](skills/email-report-delivery/)** | Enviar un informe/digest **como cuerpo HTML del email** (no como adjunto ni Markdown), con imágenes embebidas como **CID attachments** (no data-URI), acentos garantizados y links a la noticia completa. |

### Por qué publicamos esto
- **Reutilización**: cada skill es un playbook probado con bugs, "pitfalls" y
  verificación real — no teoría.
- **Seguridad**: los datos sensibles (paths, IPs, tokens) fueron **anonimizados**
  antes de publicar. No hay secretos en el repo.
- **Transparencia**: código y know-how abiertos bajo **MIT**.

### Uso
Cada skill está en `skills/<nombre>/SKILL.md` (contenido principal) más
`references/` (detalles y gotchas). Léelas como guías paso a paso.

---

## 🇬🇧 English

**AgentSkills** is a collection of **agent skills** — reusable playbooks an AI
agent can load to perform complex tasks. We publish the skills **BarraMan and
BarraBot** designed, tested, and curated in real practice — documented in
detail and **anonymized** (no paths, IPs, credentials, or personal data).

### Included skills

| Skill | What it does |
|---|---|
| **[news-digest](skills/news-digest/)** | Build a **news digest** (weekly/recurring) and send it by email as an **HTML body** with cover images: collection via Exa, curation/dedupe by cluster, 3-row format (title / hyperlinked cover / summary), delivery with himalaya, and verification in the `sent` folder. |
| **[email-report-delivery](skills/email-report-delivery/)** | Send a report/digest **as the email HTML body** (not an attachment, not Markdown), with images embedded as **CID attachments** (not data-URIs), guaranteed accents, and full-news links. |

### Why we publish this
- **Reusability**: each skill is a tested playbook with real bugs, "pitfalls",
  and verification — not theory.
- **Security**: sensitive data (paths, IPs, tokens) was **anonymized** before
  publishing. No secrets in the repo.
- **Transparency**: code and know-how open under **MIT**.

### Usage
Each skill lives in `skills/<name>/SKILL.md` (main content) plus `references/`
(details and gotchas). Read them as step-by-step guides.

---

## 📂 Estructura / Structure

```
AgentSkills/
├── README.md                          # este archivo (ES + EN)
├── LICENSE                            # MIT
├── .gitignore                         # bloqueo de secretos (defensa)
└── skills/
    ├── news-digest/
    │   ├── SKILL.md
    │   └── references/{sources,curate,exa-search,formato-casa,build-send-script,himalaya-send}.md
    └── email-report-delivery/
        ├── SKILL.md
        └── references/himalaya-send.md
```

## 🔐 Seguridad / Security
- **Anonimizado**: rutas → `$HOME`/`CATALOG/`, IPs → `<LAN_IP>`, `.exa_key` →
  `<EXA_KEY_FILE>`, correos → `<YOUR_EMAIL>`.
- Sin `.env`, tokens, credenciales ni archivos de media en el repo.

## 📜 License
[MIT](LICENSE) © 2026 BarraMan.

## 🔎 Topic tags
`ai-agents` `skills` `playbook` `automation` `media` `audio` `video`
`metadata` `id3` `ffmpeg` `mutagen` `catalog` `classification` `web`
`email` `html` `reports` `smtp` `himalaya` `python` `best-practices`
`documentation` `bilingual`