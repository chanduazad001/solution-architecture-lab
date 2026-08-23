# Solution Architecture Lab

Cloud architecture, platform engineering, DevSecOps and AI infrastructure —
documented as deliberate architecture practice: designs, constraints,
trade-offs, and the reasoning behind architectural decisions.

**Published site:** https://chanduazad001.github.io/solution-architecture-lab/

Part of a wider set of labs linked from
[chanduazad001.github.io](https://chanduazad001.github.io/), including
[Cybersecurity Architecture Lab](https://chanduazad001.github.io/cybersecurity-architecture-lab/).

## Structure

Each architecture lives under `docs/architecture/<name>/` and follows the
same set of documents:

- **Overview** — what the architecture is and why
- **Architecture** — detailed component design
- **Data Flow** — request/response paths through the system
- **Security** — trust boundaries and controls
- **Availability** — HA, failure scenarios, DR
- **Decisions** — Architecture Decision Records (ADRs)

Diagrams are written as [Mermaid](https://mermaid.js.org/) inline in the
Markdown, so they stay version-controlled alongside the text.

## Local development

```bash
python -m venv .venv
.venv\Scripts\activate      # Windows
source .venv/bin/activate   # macOS/Linux

pip install -r requirements.txt

mkdocs serve
```

Then open http://127.0.0.1:8000.

## Publishing

Pushes to `master` that touch `docs/`, `mkdocs.yml`, or `requirements.txt`
trigger `.github/workflows/docs.yml`, which builds the site with
`mkdocs build --strict` and deploys it to GitHub Pages.

## License

See [LICENSE](LICENSE).
