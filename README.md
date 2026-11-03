# speedtest

Tests download and upload speed to GitHub. The test data is random bytes, so compression can't inflate the result.

## Use

Paste this into your shell to try it, or add it to `~/.bashrc` to keep it:

```bash
# Download: no login needed. Stops after 20 s.
ghdown() {
  echo "Testing download from GitHub (up to 20 s)..."
  curl -fsL -o /dev/null --max-time 20 -w '%{speed_download} %{size_download} %{time_total}\n' \
    https://github.com/iotistic/speedtest/releases/download/v1/500MB.bin \
    | awk '$2>0 {printf "\033[1;31m%.1f Mbit/s down (%.1f MiB in %.1fs)\033[0m\n", $1*8/1e6, $2/1048576, $3; next} {print "download failed"}'
}

# Upload: needs an SSH key with write access (e.g. iotistic-mnvr). Stops after 20 s.
ghup() (
  echo "Testing upload to GitHub (up to 20 s)..."
  d=$(mktemp -d) && cd $d && git init -q && head -c 50M /dev/urandom > up.bin && git add up.bin && rm up.bin \
    && git -c user.name=mnvr -c user.email=mnvr@local commit -qm up
  b=up/$(hostname)-$(date +%s); r=git@github.com:iotistic/speedtest.git
  s=$(date +%s.%N); timeout 20 git -c pack.compression=0 push --progress $r HEAD:refs/heads/$b 2> log; e=$(date +%s.%N)
  tr '\r' '\n' < log | grep 'Writing objects' | tail -1 | sed -n 's/.*), \([0-9.]*\) \([KMG]\)iB.*/\1 \2/p' \
    | awk -v s=$s -v e=$e '{m=$2=="K"?1024:$2=="M"?1048576:1073741824; printf "\033[1;31m%.1f Mbit/s up (%.1f MiB in %.1fs)\033[0m\n", $1*m*8/(e-s)/1e6, $1*m/1048576, e-s} END{exit !NR}' \
    || { echo "upload failed:"; tail -3 log; }
  git push -q $r --delete $b 2>/dev/null; cd /; rm -rf $d
)

ghspeed() { ghdown; ghup; }
```

Then run `ghdown`, `ghup`, or `ghspeed` for both.

## How it works

**Download**
- Fetches a 500 MB release file and throws the data away. `--max-time 20` stops it after 20 s; the speed is the average up to then.
- Release files are served from `release-assets.githubusercontent.com`, the same as every other GitHub release download.

**Upload**
- Pushes up to 50 MiB to a throwaway `up/` branch, then deletes the branch. If the 20 s limit cuts it off, the speed comes from the bytes git reports it sent before stopping.
- git's output goes to a file, not a pipe, because `timeout` would kill a pipe reader and lose what it was holding.
- `ghup` runs in a subshell (its body is in `( … )`), so it doesn't change your folder or leave variables behind.
- The time includes about 1.3 s for the SSH connection and GitHub's processing, so the result reads a little low.
- `pack.compression=0` stops git from compressing the data, so a slow device CPU doesn't drag the result down.
- Rulesets only allow pushes to `up/*` branches; `main` and tags are locked to repo admins.
- Each run uploads up to 50 MiB (less if the 20 s limit cuts it off), so mind the cost on cellular links.
