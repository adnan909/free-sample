# Backend Skills for Claude Code: free sample

Two production-tested [Claude Code skills](https://docs.claude.com) for **TypeScript + GraphQL + Postgres** backends. Drop them in your repo and Claude Code writes resolvers and migrations the way a senior backend engineer would.

| Skill | What it makes Claude do |
| --- | --- |
| `graphql-n-plus-one` | Find N+1 queries in resolvers, fix them with per-request DataLoaders, and add a query-count test so they don't come back |
| `postgres-safe-migrations` | Write migrations that don't lock live tables: concurrent indexes, `NOT VALID` constraints, batched backfills, expand-and-contract renames |

## Install

Copy the skills folder into your project:

```bash
git clone https://github.com/adnan909/free-sample
mkdir -p your-project/.claude/skills
cp -r free-sample/.claude/skills/* your-project/.claude/skills/
```

Claude Code picks them up on the next session. Or install them for all projects by copying into `~/.claude/skills/` instead.

## Try it

In a TS/GraphQL project, ask Claude Code:

- "Check the Post and Order resolvers for N+1 queries"
- "Add a NOT NULL `status` column to the projects table"

## The full pack

The full **Backend Skills Pack** is coming soon with 10 skills for this stack: schema design, resolver auth, typed errors, integration tests against real Postgres, slow-query profiling, a PR review checklist, a `CLAUDE.md` template, and a starter repo every skill is tested against.

Star this repo to hear when it launches.

## License

MIT
