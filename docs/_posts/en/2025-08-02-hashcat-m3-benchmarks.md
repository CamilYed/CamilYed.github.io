---
layout: post
title: "First Look: Benchmarking Hashcat v7.0 on Apple M3 Pro"
lang: en
ref: hashcat-m3-benchmarks
date: 2025-08-02
tags: [ hashcat, cybersecurity, benchmarking, m3, apple-silicon ]
permalink: /en/hashcat-m3-benchmarks/
---

With the recent release of [Hashcat v7.0.0](https://hashcat.net/hashcat/), including 58 new algorithms and support for the Assimilation Bridge and Python Bridge, it's time to explore how it performs on Apple's new M3 hardware.

This post documents an experiment I ran using [my own benchmarking harness](https://github.com/CamilYed/hashcat-m3-tests) to compare speeds of selected algorithms (MD5, Argon2, ZIP) on Apple M3 using Python and log-scale graphs.

<!--more-->

## 🔍 Why Hashcat v7.0 Is a Big Deal

According to [Michal Sajdak](https://www.linkedin.com/feed/update/urn:li:activity:7357325556654182400/), the new release brings:

- ✅ Support for **58 new algorithms** (e.g., Argon2, LUKS2, GPG, OpenSSH)
- ✅ `--identify`: automatic detection of hash type based on sample input
- ✅ **20 new tools** for extracting hashes from various sources (e.g., BitLocker)
- ✅ **Assimilation Bridge**: seamless integration of multiple cracking engines (e.g., local CPU/GPU, remote GPUs, custom scripts)
- ✅ **Python Bridge**: define password crackers in Python for niche or unsupported algorithms
- ✅ **Major performance improvements**: up to 54% faster RAR cracking, 320% for scrypt, and broader Apple/AMD optimizations

So I set out to test it on a native Apple M3 Pro system.


---

## 🧪 Project Goals

- Benchmark selected hash algorithms: MD5, Argon2, ZIP, bcrypt, etc.
- Evaluate Apple M3 GPU/CPU performance with Metal/OpenCL backends
- Automate:
  - Test execution
  - Result parsing and CSV export
  - Visual log-scale plots

---

## 📁 Repository Structure

[🔗 GitHub Repo: CamilYed/hashcat-m3-tests](https://github.com/CamilYed/hashcat-m3-tests)

```
hashcat-m3-tests/
├── benchmark.py            # Benchmark orchestration using subprocess
├── config/
│   └── algorithms.yaml     # Declarative list of hash modes to test
├── config_loader.py        # Loads and parses YAML config
├── plot_results.py         # Generates log-scale performance bar charts
├── results.csv             # Raw benchmark results (speed, time)
├── results.png             # Plot exported from matplotlib
└── requirements.txt        # Python dependencies
```

---

## ⚙️ How It Works

When `benchmark.py` is run:

1. It loads supported hash modes from `config/algorithms.yaml`
2. Asks the user which algorithms to benchmark
3. Runs 3 repetitions per algorithm using:
   ```bash
   hashcat -b -m <mode> --force
   ```
4. Captures output speeds and durations
5. Stores the data into `results.csv`
6. You can then run `plot_results.py` to generate graphs from CSV

---

## 📈 Sample Output

Here’s a benchmark comparing MD5 and ZIP (WinZip):

<div style="display: block; text-align: center; margin: 20px auto; max-width: 100%; overflow: hidden;">
  <img src="/assets/hash_cat_benchmark.png" alt="benchmark" style="width: 100%; height: auto;">
</div>

- MD5: ~8.8 billion H/s
- ZIP: ~1.5 million H/s
- Scale is logarithmic due to magnitude differences

---

## 🧠 Observations

- ✅ Hashcat runs natively using Metal/OpenCL on M3 without external GPU
- ✅ Argon2 `mode 70000` runs via Python Bridge with `Assimilation Bridge`
- ❌ Some modes like `20100` may require additional `.so` modules on macOS
- ✅ CSV and plot automation makes it easy to compare across machines/algorithms
- 📉 Huge variance in speed across hash types (MD5 vs Argon2 vs ZIP)

---

## 📦 Supported Features

This project also lays the groundwork for:

- ✅ Custom YAML-based algorithm lists
- ✅ Optional hashcat `--identify` integration (planned)
- ✅ Better M3 support testing for Metal/OpenCL backends
- ✅ Future remote GPU support via Assimilation Bridge

---

## 🔚 Conclusion

This is an early exploration into using Python to benchmark Hashcat on Apple Silicon. It already provides reproducible measurements, customizable algorithm selection, and visual comparisons.

**Next steps:**

- Add automatic `--identify` hash detection integration
- Include device info (CPU/GPU type, backend) in results
- Test more bridged modes (GPG, LUKS2, OpenSSH)
- Compare M1 vs M3 vs AMD/NVIDIA GPUs

🧪 Try it yourself:  
📁 https://github.com/CamilYed/hashcat-m3-tests

---

If you have feedback or ideas (e.g., storing results over time, backend comparisons), feel free to open a GitHub issue or comment below.

Stay secure 💻🔐
