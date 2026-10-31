# VoteSecure: DevSecOps pipeline and hardened container

This repo shows the delivery and security work around VoteSecure, a React, TypeScript and Vite voting app with a Supabase backend (originally built with Lovable). The focus is the pipeline and the container, not the app. The app's original README is in [docs/app-readme.md](docs/app-readme.md).

The pipeline was built as part of a DevSecOps assignment on a self-hosted GitLab. The full write-up, with configuration files and evidence, is here: https://github.com/Phvl-0/devsecops-assignment

## What's in this repo

- **Dockerfile**: a multi-stage build. A `node:22-alpine` stage installs dependencies and builds the app, then only the compiled files are copied into a small `nginx:alpine` image. The final image has no Node, no npm and no source code. Measured size of the published image: about 27 MB compressed, 95.8 MB on disk.
- **nginx.conf**: hides the nginx version, sends `X-Content-Type-Options: nosniff` and `X-Frame-Options: DENY`, and falls back to `index.html` so React routes work.
- **.gitlab-ci.yml**: a five-stage GitLab CI pipeline (below).
- **tests/**: build smoke tests, run with `npm test`.
- **supabase/config.toml**: holds only the Supabase project ID.

## The pipeline

1. **analyze**: ESLint. The job is allowed to fail, because the generated code has existing findings; the report is saved as an artifact.
2. **build**: `npm ci`, then the Vite production build. The output is kept for a week.
3. **test**: `npm test` (three smoke tests: the build produces `index.html`, the page has the app's mount point, and JavaScript files are generated).
4. **security**: a dependency audit (`npm audit` on production dependencies, failing on critical findings) and a secret scan (a pattern search for private keys and common token formats).
5. **deploy**: copies the build into the nginx web root and checks that it is served over HTTPS. It only runs on the branch `devsecops-pipeline`.

A stage only starts if the one before it passed.

## Run the container

    docker build -t votesecure .
    docker run -d -p 8082:80 votesecure

Then open http://localhost:8082.

## Honest limits

- GitLab CI does not run on GitHub, so pipeline runs cannot be viewed here. The pipeline file and commit history are in this repo, and screen recordings of a run are linked from the assignment write-up.
- The deploy rule matches the branch name `devsecops-pipeline`, so it would not trigger on this repo's `main`.
- The secret scan only checks the files as they are now, not Git history, and its patterns do not include Supabase keys.
- Lint has open errors in the existing code, and the dependency audit only fails on critical findings. At the time of the assignment, 7 production vulnerabilities remained, including one high in `zod`, which need breaking upgrades.
- The container image has not been scanned yet. It sets no non-root user, and it serves plain HTTP; TLS would come from a reverse proxy.
- The backend is Supabase, a hosted service outside the scope of this repo.
- Before this repo was published, its full history was searched for secrets: only public Supabase (anon) keys and the project URL were found, and no service-role key or passwords.
