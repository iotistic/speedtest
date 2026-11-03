# speedtest

A 500 MB file of random bytes for testing download speed from GitHub. It's random, so compression can't inflate the result.

```bash
curl -L -o /dev/null --max-time 20 -w '%{speed_download}\n' \
  https://github.com/msaad-iotistic/speedtest/releases/download/v1/500MB.bin \
  | awk '{printf "%.1f Mbit/s\n", $1*8/1e6}'
```

- `--max-time 20` stops after 20 s, so slow links don't download the whole file. curl reports a timeout error (exit code 28); the speed is still correct.
- Release files are served from `release-assets.githubusercontent.com`, the same as every other GitHub release download.
- To test `git clone` speed instead: `git clone --depth 1 https://github.com/torvalds/linux` and watch the MiB/s.
