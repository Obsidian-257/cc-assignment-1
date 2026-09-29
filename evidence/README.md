# Evidence

## Part B — build evidence

| File | Contents |
|:--|:--|
| `b1-sizes.txt` | `docker image ls cc-demo` for all three builds, plus `docker history` for A, B, and C |
| `b2-cache.txt` | Full `--progress=plain` build output after changing one line in `app.py` — shows CACHED on dependency layers |
| `b3-cache.txt` | Same change, built against `Dockerfile.bad` with `COPY . .` above `pip install` — shows dependencies rebuilding |

## Part C — Azure deployment evidence

| File | Contents |
|:--|:--|
| `c1-acr.txt` | `az acr repository show-tags` showing the image in the registry |
| `c1-acr.png` | Portal screenshot of the image in ACR |
| `c2-aci.txt` | `az container show` output with public IP and running state |
| `c2-aci.png` | Portal screenshot of the running container instance |
| `c3-request.txt` | HTTP request to the public IP showing the greeting response |
| `c4-secure-var.txt` | Environment variables showing API_KEY value as null |