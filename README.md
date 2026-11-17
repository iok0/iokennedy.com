# iokennedy.com

Source for [iokennedy.com](https://iokennedy.com/): Hugo with no external
theme (the layouts and one CSS file live in this repo). A GitHub Actions
workflow deploys it to GitHub Pages.

## Preview locally

The host doesn't need a Hugo install; it runs in a pinned container:

```sh
docker run --rm -u $(id -u):$(id -g) -v "$PWD":/project -w /project \
  -p 1313:1313 ghcr.io/gohugoio/hugo:v0.147.9 server -D --bind 0.0.0.0
```

Then open http://localhost:1313. `-D` includes drafts.

## Writing

- Posts in the *Engineering with AI* section live in `content/ai/`, listed
  newest first. `draft: true` keeps a post out of the published site.
- Diagrams go in `assets/diagrams/` and are included with the `diagram`
  shortcode, which inlines the SVG so it picks up the site's light and dark
  colours.

## Going live (one-time)

1. Create the GitHub repo and push. Under Settings → Pages, set the source to
   **GitHub Actions**.
2. Settings → Pages → verify `iokennedy.com`. Verification adds a TXT record
   and prevents domain takeover.
3. DNS at the registrar. Write down the existing MX, SPF, DKIM and DMARC
   records first, and **don't change them**; email runs on this domain.
   - `A` records for `iokennedy.com`: `185.199.108.153`, `185.199.109.153`,
     `185.199.110.153`, `185.199.111.153`
   - `AAAA` records: `2606:50c0:8000::153`, `2606:50c0:8001::153`,
     `2606:50c0:8002::153`, `2606:50c0:8003::153`
   - `CNAME` `www` → `iok0.github.io`
4. Set the custom domain to `iokennedy.com` and tick **Enforce HTTPS** once
   the certificate is issued.
5. Check that email still sends and receives.
