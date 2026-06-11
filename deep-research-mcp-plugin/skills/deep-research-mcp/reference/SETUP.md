# Setup — Firecrawl / Exa MCP connectors in Cowork

This skill needs at least one of the **Firecrawl** or **Exa** remote MCP
connectors enabled. Both are remote MCP servers, which Cowork supports.

## Standard Cowork (Free / Pro / Max / Team)

You add the connectors yourself:

1. **Settings → Connectors**.
2. Check the **Connectors Directory** first — if Firecrawl / Exa are listed, click
   to add (handles auth for you).
3. Otherwise **Add custom connector**, paste the remote MCP server URL, and (if
   the server needs it) configure the OAuth Client ID/Secret or API key in
   Advanced settings.
4. Click the connector and **Connect** to authenticate.
5. In a conversation, use the **"+"** button → **Connectors** to enable it per chat.

You'll need a Firecrawl and/or Exa account + API key from their dashboards. Use the
remote MCP server URL each service publishes in its own docs (verify the current
URL there — they change).

## Managed / Enterprise "Cowork on 3P"

End users **cannot** add remote MCP servers. An admin must either:

- provision them via the `managedMcpServers` configuration key (remote HTTP/SSE),
  which pushes them to every device and supports per-tool `allow`/`ask`/`blocked`
  policy locks; or
- distribute an **organization plugin** that bundles the connector reference.

Ask your Cowork administrator to add Firecrawl/Exa if they're not already present.

## Verify it works

In Cowork, ask: "List the tools from my Firecrawl/Exa connector." If you see
`firecrawl_search` / `firecrawl_scrape` or `web_search_exa` / `web_fetch_exa`,
you're ready. If the names are namespaced (prefixed), the skill still works — it
references the logical operations, not exact identifiers.

## Cleaner alternative: ship as a plugin

The most integrated option is a **plugin** that bundles this SKILL.md _and_ the
Firecrawl/Exa remote MCP connector references, so installing the plugin wires up
both at once. Plugins are supported in Cowork and Claude Code. Build one with the
`mcp-server-dev` / `plugin-dev` tooling if you want one-step distribution instead
of "add connector, then upload skill."
