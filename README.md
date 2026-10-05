# Ben's Claude Code Plugin Marketplace

A Claude Code plugin marketplace.

| Plugin | Stack |
| --- | --- |
| `nextjs-supabase-kit` | Next.js · TypeScript · Tailwind · Supabase · GitHub · Vercel |
| `react-dotnet-k8s-kit` | React · Vite · .NET · GitLab · Kubernetes · ArgoCD |

## Install

```
/plugin marketplace add BenFielder1/claude-kits
/plugin install nextjs-supabase-kit@bens-claude-kits
/plugin install react-dotnet-k8s-kit@bens-claude-kits
```

## Releasing a change

1. Edit the plugin under `plugins/<name>/`.
2. Bump `version` in `plugins/<name>/.claude-plugin/plugin.json`. Users don't receive changes until it changes.
3. Run `claude plugin validate .`, then commit and push.
4. Users run `/plugin marketplace update bens-claude-kits`, or turn on auto-update under Marketplaces in `/plugin`.
