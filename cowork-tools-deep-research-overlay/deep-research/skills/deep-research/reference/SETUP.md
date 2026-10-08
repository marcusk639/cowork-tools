# Setup — Firecrawl / Exa MCP connectors in Cowork

The `deep-research` skill works without any connector, using Cowork's built-in
web search and fetch. Enabling the **Firecrawl** and/or **Exa** remote MCP
connectors improves it: fetching any public URL, PDF parsing, domain filters and
semantic search. Both are remote MCP servers, which Cowork supports.

> **This plugin does not bundle the connectors.** You enable them yourself in
> Cowork (one-time setup below). This is deliberate — you control which provider
> runs and which account is billed, and there are no third-party server settings
> baked into the plugin. The skill checks for an enabled connector at runtime and
> falls back to Cowork's built-in web tools if neither is present.

## Standard Cowork (Free / Pro / Max / Team)

You add the connector(s) yourself. **Exa** is in Cowork's Connectors Directory;
**Firecrawl** is typically added as a custom connector.

### Exa (in the directory)

1. **Settings → Connectors**.
2. Find **Exa** in the directory and click to add — server URL `https://mcp.exa.ai/mcp`.
3. Click **Connect** and complete sign-in (OAuth; no client ID/secret to enter).

### Firecrawl (custom connector)

1. **Settings → Connectors → Add custom connector**.
2. Server URL: `https://mcp.firecrawl.dev/v2/mcp`.
3. Auth — pick one:
   - **Keyless (OAuth):** leave the OAuth Client ID and Client Secret **blank**
     (Firecrawl uses Dynamic Client Registration); a browser sign-in opens on first
     connect — authorize, and the connection completes. All calls bill to the
     Firecrawl account you sign in as.
   - **API key:** select Bearer Token auth and paste a Firecrawl API key from your
     Firecrawl dashboard (firecrawl.dev → API Keys).
4. Click **Connect**.

### Enable per conversation

In a Cowork task, use the **"+"** button → **Connectors** to switch the connector on
for that session.

> Endpoints can change — if a connect attempt fails, verify the current remote MCP
> URL in each service's own docs (Exa: docs.exa.ai; Firecrawl: docs.firecrawl.dev).
> Both providers **bill per call**, Firecrawl especially (a full research run is
> ~5 searches + ~15 scrapes + verification searches), so watch usage in their
> dashboards.

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

## Note: bundling is possible, but intentionally not done here

Cowork plugins *can* bundle a remote MCP connector (via a `.mcp.json` at the plugin
root), so installing the plugin would also register the server in one step. This
plugin intentionally does **not** do that: keeping connector setup explicit means
no third-party server config lives in the marketplace repo, you choose the provider
and the billed account, and there's nothing to go stale if an endpoint URL changes.
The one-time setup above is the trade. (On managed "Cowork on 3P," where users
can't add servers, an admin can still bundle these via an org plugin or
`managedMcpServers` — see the section above.)
