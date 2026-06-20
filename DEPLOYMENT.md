# Cloudflare Pages Deployment

This project is ready for Cloudflare Pages as a static Astro site.

## Build settings

- Framework preset: `Astro`
- Build command: `npm run build`
- Output directory: `dist`
- Node.js version: use the Cloudflare default current LTS, or set `NODE_VERSION` to `20`

## Deploy from Git

1. Push this project to a GitHub or GitLab repository.
2. In Cloudflare, open **Workers & Pages**.
3. Select **Create application**.
4. Choose **Pages**.
5. Connect the repository.
6. Use these build settings:
   - Build command: `npm run build`
   - Build output directory: `dist`
7. Click **Save and deploy**.

Cloudflare will install dependencies, run the build, and publish the generated static files.

## Custom domain: newsolars.com

1. In the Cloudflare Pages project, open **Custom domains**.
2. Add `newsolars.com`.
3. If you also want `www.newsolars.com`, add it as a second custom domain.
4. Because the domain is already managed in Cloudflare DNS, Cloudflare can create the required records automatically.
5. Confirm SSL/TLS is active after the domain finishes validating.

Recommended DNS behavior:

- `newsolars.com` should point to the Cloudflare Pages project.
- `www.newsolars.com` can point to the same Pages project or redirect to `newsolars.com`.

## Production checks

After deployment, check:

- `https://newsolars.com/`
- `https://newsolars.com/calculator`
- `https://newsolars.com/methodology`
- `https://newsolars.com/robots.txt`
- `https://newsolars.com/sitemap.xml`

Also test on a phone to confirm the mobile menu and calculator wizard work correctly.

## Notes

The calculator stores estimates in the visitor's browser only. No authentication, database, maps, or server API is required for this deployment.
