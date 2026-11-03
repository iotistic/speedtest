# speedtest

Tests download and upload speed to GitHub. The file is random bytes, so compression can't inflate the result.

## Download (no login)

```bash
curl -L -o /dev/null --max-time 20 -w '%{speed_download}\n' \
  https://github.com/iotistic/speedtest/releases/download/v1/500MB.bin \
  | awk '{printf "%.1f Mbit/s down\n", $1*8/1e6}'
```

- `--max-time 20` stops after 20 s, so slow links don't download the whole file. curl reports a timeout error (exit code 28); the speed is still correct.
- Release files are served from `release-assets.githubusercontent.com`, the same as every other GitHub release download.

## Upload (SSH key with write access, e.g. `iotistic-mnvr`)

Pushes 20 MiB to a throwaway `up/` branch, times it, then deletes the branch.

```bash
d=$(mktemp -d) && cd $d && git init -q && head -c 20M /dev/urandom > up.bin && git add up.bin \
  && git -c user.name=mnvr -c user.email=mnvr@local commit -qm up
b=up/$(hostname)-$(date +%s); r=git@github.com:iotistic/speedtest.git
s=$(date +%s.%N); git -c pack.compression=0 push -q $r HEAD:refs/heads/$b 2>/dev/null \
  && e=$(date +%s.%N) && awk -v s=$s -v e=$e 'BEGIN{printf "%.1f Mbit/s up\n", 20*1048576*8/(e-s)/1e6}' \
  || echo "push failed (check SSH key / access)"
git push -q $r --delete $b 2>/dev/null; cd /; rm -rf $d
```

- The time includes about 1.3 s for the SSH connection and GitHub's processing, so the result reads a little low. Use `head -c 50M` (and change `20` to `50`) for a cleaner number.
- `pack.compression=0` stops git from compressing the data, so a slow device CPU doesn't drag the result down.
- Rulesets only allow pushes to `up/*` branches; `main` and tags are locked to repo admins.
- Each run uses 20 MiB of upload data, so mind the cost on cellular links.
