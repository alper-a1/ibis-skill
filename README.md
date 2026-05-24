# ibis-skill

An open-source [Agent Skill](https://agentskills.io) for the [**Ibis Python dataframe library (v12.x)**](https://ibis-project.org/) - a reference to compensate for Ibis being underrepresented in LLM training data.

Designed for Ibis-powered geospatial pipelines using DuckDB and BigQuery at [OnyX](https://onyxcorporations.com/).

Note: This is a WIP skill, and is currently only tailored to my use case. Feel free to fork and improve as needed for your own use cases. Currently only supports the DuckDB and BigQuery backend as that is what I use.

## What it covers

| Reference | Scope |
|---|---|
| **SKILL.md** | Mental model, pandas-to-ibis translation, quick reference for all core operations |
| core-ops.md | Selectors, extended select/mutate/filter, conditional logic, literals |
| joins.md | All join types, predicate forms, `_right` suffix handling, self-joins, production patterns |
| aggregations.md | Aggregate functions, window functions, struct/array aggregation, ibis.array/map |
| spatial.md | Full geospatial API, spatial joins, composite analysis patterns |
| types.md | Type system (dt.*), schemas, casting, type annotations |
| udfs.md | Builtin scalar/aggregate UDFs, custom UDF flavors |
| backends.md | DuckDB and BigQuery connection, read/write, memtable patterns, SQL interop |

## Installation

### Claude Code

```bash
# Clone the repo
git clone https://github.com/alper-a1/ibis-skill.git

# Symlink the skill into Claude's skills directory
ln -s "$(pwd)/ibis-skill/ibis" ~/.claude/skills/ibis
```

Or copy directly:

```bash
cp -r ibis-skill/ibis ~/.claude/skills/ibis
```

The skill activates automatically when Claude detects ibis-related tasks.

### Other agents

Copy the `ibis/` directory into your agent's skill discovery path. The directory must be named `ibis` to match the `name` field in the frontmatter. See the [Agent Skills specification](https://agentskills.io/specification) for details.

## Structure

```
ibis-skill/
├── README.md
├── CHANGELOG.md
├── ibis/                  <- the skill (this is what gets installed)
│   ├── SKILL.md           <- main body (~270 lines)
│   └── references/        <- extended reference files (~2,000 lines total)
│       ├── core-ops.md
│       ├── joins.md
│       ├── aggregations.md
│       ├── spatial.md
│       ├── types.md
│       ├── udfs.md
│       └── backends.md
└── tests/
    └── evals.json         <- trigger evaluation queries (TODO)
```

## Design principles

- **Progressive disclosure**: SKILL.md stays under 500 lines. Detailed reference material lives in `references/` and is loaded on demand.
- **DRY**: Reference files don't repeat what's in the main body. Each section states what it extends.
- **Defaults over menus**: Recommends one approach, mentions alternatives briefly.
- **Real expertise**: All patterns extracted from production code, not generated from generic knowledge.
- **Opinionated**: Tailored to OnyX related usage patterns and practices.

## Versioning

| Skill version | Ibis version |
|---|---|
| 1.0.0 | 12.x |

### Validation

```bash
uv run agentskills validate ./ibis
```

## License

Apache-2.0
