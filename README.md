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

Pushes up to 50 MiB to a throwaway `up/` branch for at most 20 s, then deletes the branch. If the time runs out, the speed is worked out from the bytes git reports it sent before stopping.

```bash
( d=$(mktemp -d) && cd $d && git init -q && head -c 50M /dev/urandom > up.bin && git add up.bin && rm up.bin \
  && git -c user.name=mnvr -c user.email=mnvr@local commit -qm up
b=up/$(hostname)-$(date +%s); r=git@github.com:iotistic/speedtest.git
s=$(date +%s.%N); timeout 20 git -c pack.compression=0 push --progress $r HEAD:refs/heads/$b 2> log; e=$(date +%s.%N)
tr '\r' '\n' < log | grep 'Writing objects' | tail -1 | sed -n 's/.*), \([0-9.]*\) \([KMG]\)iB.*/\1 \2/p' \
  | awk -v s=$s -v e=$e '{m=$2=="K"?1024:$2=="M"?1048576:1073741824; printf "%.1f Mbit/s up (%.1f MiB in %.1fs)\n", $1*m*8/(e-s)/1e6, $1*m/1048576, e-s}' \
  | grep . || { echo "push failed:"; tail -3 log; }
git push -q $r --delete $b 2>/dev/null; cd /; rm -rf $d )
```

- Wrapped in `( … )` so it runs in a subshell: your shell stays in the same folder and gets no leftover variables.
- `timeout 20` caps the test at 20 s; change it to suit. git's output goes to a file, not a pipe, because `timeout` would kill a pipe reader and lose what it was holding.
- The time includes about 1.3 s for the SSH connection and GitHub's processing, so the result reads a little low.
- `pack.compression=0` stops git from compressing the data, so a slow device CPU doesn't drag the result down.
- Rulesets only allow pushes to `up/*` branches; `main` and tags are locked to repo admins.
- Each run uploads up to 50 MiB (less if the 20 s limit cuts it off), so mind the cost on cellular links.
