# lembaran-site

Public landing page + privacy policy for the Lembaran iOS app, served via GitHub
Pages at https://lembaran.app/. The old project URL,
https://ihzacordova.github.io/lembaran-site/, still resolves — GitHub redirects it
to the custom domain, which is what keeps the privacy URL filed with App Review
working.

The custom domain is set by the `CNAME` file in this repo root. Deleting it
reverts the site to the github.io URL, so leave it.

This repo is public because Pages on a private repo needs a paid plan, and the
privacy policy has to stay reachable — App Review follows it, and the shipping
app links to it from Settings and the paywall (`Sources/Links.swift`).

## Where each file comes from

| File | Source of truth |
| --- | --- |
| `index.html` | **This repo.** Self-contained: fonts and the app icon are inlined, so there is no build step and no external request. Edit it here. |
| `privacy.html` | `ios/Lembaran.swiftpm/privacy.html` in the private `Lembaran` repo. Edit there, copy here. |
| `icon.png` | The 1024×1024 app icon, used as favicon and OG image. |

`index.html` started as a design mockup and was adapted for publication: it got a
`<head>` (title, description, OG tags, favicon), the App Store badges were
pointed at `https://apps.apple.com/app/id6794497692`, and the footer's
`lembar.app` link — a domain that was never registered — became the privacy link.

The paywall panel partway down the page is a **replica of the in-app screen**.
Its "Terms / Privacy / Restore purchase / Maybe later" links are deliberately
inert; they illustrate the app's UI rather than act as site navigation.

`.nojekyll` stops GitHub Pages from running Jekyll over the output. Leave it.

The other marketing site in the private repo (`web/`, headline "Not for
everyone.") is a **separate, unpublished** design. Don't confuse the two.
