# How to update this branch

```bash
git checkout master # or whatever
# build the tgz file and copy it here
helm package deploy/cert-manager-webhook-hetzner/
git checkout gh-pages
# create or update the index.yaml for repo
helm repo index . 
git add .
git commit -m "New chart version"
```