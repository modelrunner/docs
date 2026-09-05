# Mintlify Starter Kit

Use the starter kit to get your docs deployed and ready to customize.

Click the green **Use this template** button at the top of this repo to copy the Mintlify starter kit. The starter kit contains examples with

- Guide pages
- Navigation
- Customizations
- API reference pages
- Use of popular components

**[Follow the full quickstart guide](https://starter.mintlify.com/quickstart)**

## Development

Install the [Mintlify CLI](https://www.npmjs.com/package/mint) to preview your documentation changes locally. To install, use the following command:

```
npm i -g mint
```

Run the following command at the root of your documentation, where your `docs.json` is located:

```
mint dev
```

View your local preview at `http://localhost:3000`.

## Analytics

PostHog is configured under `integrations.posthog` in `docs.json`, pointed at the
**same project as the web app** — `ModelRunner Web`, project id `518635`, on US
Cloud (`https://us.i.posthog.com`). The `phc_…` key is a publishable project key
and is safe in the browser; Mintlify has no env-var substitution for it, so it
lives in `docs.json` literally, same value as `POSTHOG_KEY` in the app's deploy
env.

Sharing one project is what makes a visitor the *same person* across the product
and the docs, and it works because the docs are served from the **same origin**:
nginx reverse-proxies `modelrunner.ai/docs` to `modelrunner.mintlify.dev` (see
`deploy/nginx/conf.d/web.conf` in `modelrunner/mrun`), so the browser stays on
`modelrunner.ai` the whole time. PostHog's cookie is named after the project key
(`ph_<apiKey>_posthog`), so with an identical key on an identical origin the docs
read the cookie the app already wrote — the `distinct_id`, the session, and the
signed-in identity from the app's `posthog.identify()` all carry over with no
cross-domain linking.

Two consequences worth remembering before changing any of this:

- **Do not move the docs to their own hostname.** Serving them from
  `docs.modelrunner.ai` (retired, now a 301) or any other host would put the
  cookie on a different domain and split one visitor into two people.
- **`apiHost` must stay explicit.** Mintlify defaults to `https://app.posthog.com`;
  the app ingests on `https://us.i.posthog.com`.

The app suppresses analytics for internal accounts (`ANALYTICS_SUPPRESSED_USERNAMES`)
and only enables PostHog in production. Mintlify can do neither — it has no view
of the signed-in user — so docs traffic is tracked unconditionally, internal
browsing included. Filter that out project-side in PostHog if it distorts a
number.

### Every docs event arrives twice

Mintlify's own bundle — not anything in this repo — emits each docs interaction
under two names: the current one, and a legacy alias it keeps for backward
compatibility. The duplicates are exact: same visitor, same path, a millisecond
or two apart. Nineteen pairs exist (the full alias map is quoted in
modelrunner/docs#24); the ones seen so far are:

| current name | legacy alias |
| --- | --- |
| `docs.navitem.click` | `header_nav_item_click` |
| `docs.accordion.open` | `accordion_open` |

Search, code-block copies, API-playground calls and the assistant have their own
pairs (`docs.search.result_click` / `search_result_click`, `docs.code_block.copy` /
`code_block_copy`, `docs.api_playground.request` / `api_playground_call`,
`docs.assistant.*` / `chat_*` and `ai_chat_*`) and will start double-counting the
moment they see traffic.

Page views are the same story by a different mechanism: the bundle's route-change
handler captures `$pageview` and then `$docs.content.view`, unconditionally, so
on every docs path the two counts match exactly. The `$` prefix on
`$docs.content.view` is Mintlify's, not ours — PostHog reserves it, but the bundle
builds the name with a template literal.

Two rules when reading docs numbers out of PostHog:

- **Count one name per pair.** Query the `docs.*` name and ignore its alias, or
  every engagement figure is double.
- **Never add `$docs.content.view` to `$pageview`.** It is the same view, not
  extra traffic. The project also carries the app's own `$pageview`s, so compare
  the two per docs path, never project-wide.

This is lived with rather than fixed. The two alternatives, neither taken: ask
Mintlify (`mintlify/docs`) for a switch that drops the legacy aliases, or drop
the alias names ingestion-side with a PostHog CDP transformation — project
config that lives in no repo and silently diverges what PostHog stores from what
Mintlify sends.

To re-check after a Mintlify release, paired names should still show identical
counts:

```sql
SELECT event, count() AS n, uniq(person_id) AS persons, max(timestamp) AS last_seen
FROM events
WHERE timestamp >= now() - INTERVAL 30 DAY
  AND event IN ('docs.navitem.click', 'header_nav_item_click',
                'docs.accordion.open', 'accordion_open', '$docs.content.view')
GROUP BY event ORDER BY n DESC
```

and the alias map should still be in the bundle: download every `_next/static`
chunk that `https://modelrunner.ai/docs` references (the chunk hashes rotate with
each release, so grep them all rather than a remembered filename) and search for
`header_nav_item_click`. If both come back empty, Mintlify dropped the shim and
the rules above can go.

## Publishing changes

Install our GitHub app from your [dashboard](https://dashboard.mintlify.com/settings/organization/github-app) to propagate changes from your repo to your deployment. Changes are deployed to production automatically after pushing to the default branch.

## Need help?

### Troubleshooting

- If your dev environment isn't running: Run `mint update` to ensure you have the most recent version of the CLI.
- If a page loads as a 404: Make sure you are running in a folder with a valid `docs.json`.

### Resources
- [Mintlify documentation](https://mintlify.com/docs)
- [Mintlify community](https://mintlify.com/community)
