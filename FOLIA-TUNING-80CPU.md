# CÔNG THỨC & KẾ HOẠCH TỐI ƯU HIỆU NĂNG FOLIA — TẬN DỤNG 80% CPU

> Mục tiêu: Một process Folia chạy ổn định ở **CPU utilization 70-80%** trên toàn bộ core của 1 máy vật lý, với MSPT < 40ms p99, TPS = 20.0 ổn định, KHÔNG có thread bottleneck.

---

## 0. Tại sao "80% là target", không phải 100%

- 100% CPU = **không còn headroom** → spike CCU/event là crash.
- < 50% CPU = lãng phí phần cứng, có thể nén thêm CCU.
- **70-80% là sweet spot**: tận dụng tối đa nhưng còn dư cho GC, OS scheduler, network IRQ, monitoring.

→ Mọi công thức dưới đây đều quy về target này.

---

## 1. Hiểu mô hình thread của Folia (CỰC QUAN TRỌNG)

Folia chia CPU thành 4 nhóm thread pool chính:

| Pool | Chức năng | Tham số config | Default |
|---|---|---|---|
| **Tick Threads** | Tick region (chunk, entity, block update) — đây là thread "nóng" nhất | `global.threaded-regions.threads` | `-1` = số core - 2 |
| **Chunk System Workers** | Load/save/generate chunk async | `chunk-system.worker-threads` | `-1` = `min(cores/2, 4)` |
| **IO Workers** | Disk I/O (region file, NBT) | `chunk-system.io-threads` | `-1` = auto |
| **Common Workers** (Java FJP) | Async task chung | `-Djava.util.concurrent.ForkJoinPool.common.parallelism=N` | `cores - 1` |

**Lỗi sai phổ biến**: để hết default → 4 pool fight nhau cùng core → context switch dày → CPU 100% nhưng TPS tụt.

→ **Quy tắc**: tổng số thread "active đồng thời" KHÔNG được vượt số core vật lý quá 1.5x.

---

## 2. Công thức phân bổ thread theo CPU

Gọi `C` = số **physical core** (KHÔNG phải logical/SMT thread).

### Công thức gốc (DonutPaper recommended):

```
TICK_THREADS         = C - 4
CHUNK_WORKERS        = max(2, C / 8)
IO_WORKERS           = max(2, C / 16)
FJP_PARALLELISM      = C - 2
RESERVED_FOR_OS_GC   = 2-4 core (để OS, GC, network IRQ)
```

### Bảng tra nhanh:

| CPU | C (cores) | tick | chunk | io | fjp | Ghi chú |
|---|---|---|---|---|---|---|
| Ryzen 7 7700X | 8 | 4 | 2 | 2 | 6 | Nhỏ, cho test/staging |
| Ryzen 9 7950X | 16 | 12 | 2 | 2 | 14 | Sweet spot 1k CCU |
| Threadripper 7960X | 24 | 20 | 3 | 2 | 22 | 2k CCU |
| Threadripper 7970X | 32 | 28 | 4 | 2 | 30 | 3k CCU |
| Threadripper 7985WX | 64 | 60 | 8 | 4 | 62 | 5k+ CCU |
| EPYC 9654 | 96 | 92 | 12 | 6 | 94 | 8k+ CCU |

**Tắt SMT/Hyperthreading** trong BIOS với workload Minecraft — SMT gây **TPS variance cao** vì 2 logical thread share L1/L2 cache, mà tick loop rất nhạy với cache miss.

---

## 3. File config Folia chuẩn (template)

### `config/paper-global.yml`
```yaml
chunk-system:
  gen-parallelism: true
  io-threads: -1          # auto theo công thức
  worker-threads: -1       # auto

threaded-regions:
  threads: -1             # auto, hoặc set explicit theo bảng trên

async-chunks:
  threads: -1

# CRITICAL: tắt feature ngốn TPS không cần thiết
unsupported-settings:
  allow-permanent-block-break-exploits: false
  allow-piston-duplication: false
```

### `config/paper-world-defaults.yml` (apply per-world)
```yaml
chunks:
  delay-chunk-unloads-by: 10s        # giảm thrash unload/reload
  prevent-moving-into-unloaded-chunks: true
  
collisions:
  fix-climbing-bypassing-cramming-rule: true

entities:
  spawning:
    despawn-ranges:
      ambient: { hard: 60, soft: 32 }
      axolotls: { hard: 60, soft: 32 }
      creature: { hard: 80, soft: 32 }
      misc:    { hard: 80, soft: 32 }
      monster: { hard: 100, soft: 40 }   # giảm từ 128/32 vanilla
      water_creature: { hard: 60, soft: 32 }
    per-player-mob-spawns: true            # KEY — chia hạn ngạch mob theo người chơi
    spawn-limits:
      monster: 35                          # giảm từ 70 default
      creature: 5
      ambient: 1
      water_animal: 3
      water_ambient: 10
      axolotls: 3
      underground_water_creature: 3
  
  behavior:
    disable-chest-cat-detection: true
    disable-player-crit-particles: true
    spawner-nerfed-mobs-should-jump: false

environment:
  optimize-explosions: true
  treasure-maps:
    enabled: false                          # treasure map gen rất nặng
    
  fire-physics-event-for-redstone: false
  
  
hopper:
  disable-move-event: true                  # KEY — tắt InventoryMoveItem event nếu không plugin nào dùng
  ignore-occluding-blocks: true

tick-rates:
  mob-spawner: 2                            # giảm tần suất spawner check
  container-update: 3                       # update chest/inventory chậm hơn
  grass-spread: 4
  sensor:
    villager: { secondarypoisensor: 80, nearestbedsensor: 80, nearestlivingentitysensor: 40 }
    
  behavior:
    villager:
      validatenearbypoi: 120

unsupported-settings:
  fix-invulnerable-end-crystal-exploit: true
```

### `config/paper-config.yml` (per-world)
```yaml
chunk-loading:
  autoconfig-send-distance: true
  enable-frustum-priority: true
  global-max-chunk-load-rate: -1.0
  global-max-chunk-send-rate: -1.0
  global-max-concurrent-loads: 500.0
  max-concurrent-sends: 2
  min-load-radius: 2
  player-max-chunk-load-rate: 100.0
  player-max-chunk-send-rate: 75.0
  target-player-chunk-send-rate: 100.0
```

### `config/spigot.yml`
```yaml
world-settings:
  default:
    mob-spawn-range: 6                       # giảm từ 8
    entity-activation-range:
      animals: 16
      monsters: 24
      raiders: 48
      misc: 8
      water: 16
      villagers: 16
      flying-monsters: 32
      wake-up-inactive:
        animals-max-per-tick: 4
        animals-for: 100
        monsters-max-per-tick: 8
        monsters-for: 100
    entity-tracking-range:
      players: 48
      animals: 32
      monsters: 32
      misc: 16
      display: 96
      other: 64
    item-despawn-rate: 4000                  # 200s thay vì 6000
    arrow-despawn-rate: 600
```

---

## 4. JVM flags — công thức cho Java 21+ ZGC

```bash
JAVA_OPTS="
  -Xms32G -Xmx32G                              # heap = RAM * 0.5, giữ Xms = Xmx
  
  # === Garbage Collector ===
  -XX:+UseZGC
  -XX:+ZGenerational                           # KEY — Java 21 generational ZGC
  -XX:-ZUncommit                               # giữ heap committed, tránh trả RAM về OS
  -XX:ZAllocationSpikeTolerance=2.0
  -XX:SoftMaxHeapSize=28G                      # ZGC sẽ cố gắng giữ heap dưới mức này
  
  # === Memory ===
  -XX:+AlwaysPreTouch                          # touch toàn bộ heap khi start, tránh page fault runtime
  -XX:+UseLargePages                           # huge page 2MB, cần OS hỗ trợ (xem mục 6)
  -XX:+UseTransparentHugePages
  -XX:LargePageSizeInBytes=2m
  
  # === Performance ===
  -XX:+UnlockExperimentalVMOptions
  -XX:+UnlockDiagnosticVMOptions
  -XX:+DisableExplicitGC                       # System.gc() bị ignore — plugin nào gọi sẽ KHÔNG trigger GC
  -XX:-OmitStackTraceInFastThrow
  -XX:+PerfDisableSharedMem                    # tắt /tmp/hsperfdata, giảm I/O
  
  # === ForkJoinPool ===
  -Djava.util.concurrent.ForkJoinPool.common.parallelism=$((CORES - 2))
  
  # === Vector API (Folia tận dụng) ===
  --add-modules=jdk.incubator.vector
  
  # === Class data sharing ===
  -XX:+UseAppCDS                               # cache class load
  
  # === Mojang/Folia ===
  -Dpaper.playerconnection.keepalive=30
  -Dio.netty.allocator.maxOrder=9
  -Dio.netty.recycler.maxCapacity=0
  -Dio.netty.recycler.maxCapacityPerThread=0
  
  # === Logging ===
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
"
```

### Tại sao ZGC chứ không phải G1GC + Aikar flags?
- Aikar flags được tune cho G1GC heap < 16GB. Heap > 32GB → G1 pause >100ms.
- ZGC generational (Java 21+) có pause **<2ms p99** kể cả heap 128GB.
- Trade-off: ZGC ngốn CPU GC nhiều hơn ~10-15% — nhưng phân tán đều, không spike.

→ Với target "80% CPU tận dụng", ZGC phù hợp hơn vì throughput cao đều, không có pause spike phá MSPT.

### Heap sizing
- **Xmx = 50% RAM máy** (ZGC cần colored pointer + metadata, tốn RAM).
- KHÔNG bao giờ Xmx > 80% RAM — OS/page cache cần phần còn lại.
- Ví dụ máy 128GB → Xmx 64GB là đủ cho mọi server <5k CCU.

---

## 5. CPU affinity & NUMA pinning

### Pin Folia process vào NUMA node 0
Trên Threadripper/EPYC có nhiều NUMA node, **phải pin** process tránh cross-NUMA memory access (tốn 2-3x latency).

```bash
# Check NUMA topology
numactl --hardware

# Run Folia pinned to NUMA node 0 + cores 0-31
numactl --cpunodebind=0 --membind=0 --physcpubind=0-31 \
  java $JAVA_OPTS -jar folia.jar nogui
```

### Tách core OS/network khỏi tick threads
Reserve core 0-1 cho OS + network IRQ, Folia dùng core 2-31:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=2-31 nohz_full=2-31 rcu_nocbs=2-31"
sudo update-grub && sudo reboot

# Run Folia pinned tới isolated cores
taskset -c 2-31 java $JAVA_OPTS -jar folia.jar
```

`isolcpus` loại core khỏi scheduler thường → tick thread KHÔNG bị OS preempt → MSPT variance giảm 50-70%.

### IRQ affinity (network card)
```bash
# Pin IRQ network NIC vào core 0-1
sudo systemctl stop irqbalance
for irq in $(grep eth0 /proc/interrupts | awk -F: '{print $1}'); do
  echo 3 | sudo tee /proc/irq/$irq/smp_affinity   # mask 0b11 = core 0,1
done
```

---

## 6. OS-level tuning (Linux kernel)

### `/etc/sysctl.d/99-minecraft.conf`
```conf
# === Network ===
net.core.somaxconn=65535
net.core.netdev_max_backlog=30000
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.tcp_fin_timeout=15
net.ipv4.tcp_keepalive_time=120
net.ipv4.tcp_keepalive_intvl=30
net.ipv4.tcp_keepalive_probes=5
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_no_metrics_save=1
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 87380 67108864
net.ipv4.tcp_wmem=4096 65536 67108864
net.ipv4.tcp_mtu_probing=1
net.ipv4.tcp_congestion_control=bbr            # BBR > cubic cho game traffic

# === VM / memory ===
vm.swappiness=1
vm.dirty_ratio=10
vm.dirty_background_ratio=5
vm.max_map_count=262144
vm.overcommit_memory=1

# === Huge pages (cho -XX:+UseLargePages) ===
vm.nr_hugepages=16384                          # 16384 * 2MB = 32GB

# === Scheduler ===
kernel.sched_autogroup_enabled=0
kernel.sched_min_granularity_ns=10000000
kernel.sched_wakeup_granularity_ns=15000000
kernel.sched_migration_cost_ns=5000000
```

Apply: `sudo sysctl -p /etc/sysctl.d/99-minecraft.conf`

### Filesystem
- **XFS** hoặc **ext4** với `noatime,nodiratime,nobarrier` mount option.
- KHÔNG dùng btrfs/zfs cho region file — overhead snapshot tốn TPS.
- Region file trên **NVMe Gen4/Gen5**, latency p99 < 100µs.

```bash
# /etc/fstab
/dev/nvme0n1p1  /var/minecraft  xfs  defaults,noatime,nodiratime,nobarrier  0  0
```

### CPU governor
```bash
# Set performance governor (KHÔNG dùng powersave/ondemand)
sudo cpupower frequency-set -g performance

# Disable CPU C-states (tránh CPU "ngủ" giữa tick)
sudo cpupower idle-set -D 1
```

---

## 7. Profiling & đo lường — biết mình ở đâu

### Spark profiler (CHÍNH)
```bash
# In-game
/spark profiler --thread tick --timeout 60       # profile tick thread 60s
/spark profiler --thread * --timeout 60          # profile tất cả thread
/spark health                                     # quick health check
/spark tps                                        # TPS history
```

Spark cho:
- Flame graph hot method.
- Phát hiện plugin nào ngốn TPS.
- Memory allocation hot path.

### async-profiler (sâu hơn)
```bash
java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=profile.html \
  $JAVA_OPTS -jar folia.jar
```

### Metric runtime
```bash
# JFR (Java Flight Recorder) — GC + thread pool stats
-XX:StartFlightRecording=duration=5m,filename=/tmp/folia.jfr

# View với JDK Mission Control
```

### Prometheus exporter
- Plugin: [`mc-monitor`](https://github.com/itzg/mc-monitor) hoặc [`MinecraftPrometheusExporter`](https://github.com/sladkoff/minecraft-prometheus-exporter).
- Metric quan trọng: `mc_tps`, `mc_mspt_p99`, `mc_chunks_loaded`, `mc_entities_count`.

---

## 8. Workload distribution — quan trọng nhất với Folia

Folia scale theo **region**, không theo player. Nếu 200 player tập trung 1 region → 1 thread → bottleneck.

### Chiến thuật "scatter"
- **Hub spawn**: random spawn trong vùng 500x500 block, không cùng 1 chunk.
- **Multi-hub**: 4-8 hub song song, player join random — tránh tập trung.
- **Event arena**: chia map ra nhiều "instance" cách nhau >256 block (= cách nhau >16 region).
- **Warps phổ biến**: spread mỗi warp cách nhau >512 block.

### Force region split
Folia tự split region khi >3 player + chunk active. Có thể "force" bằng cách:
- Đặt portal/teleport khiến player spread ra.
- Tạo "lobby corridor" dài, spawn point rải đều.

### Anti-clustering
Plugin nội bộ phát hiện **mass clustering** (>30 player trong bán kính 50 block):
- Tự động warp 1 phần ra hub khác.
- Spawn "natural barrier" ngăn dồn lại.
- Cảnh báo admin qua webhook.

---

## 9. Plugin tối ưu — không phá hiệu năng Folia

### Quy tắc cứng cho plugin
1. **Mọi I/O async**: DB, HTTP, file.
2. **Dùng đúng scheduler Folia**:
   - `RegionScheduler.run(plugin, location, task)` cho task ở 1 vị trí.
   - `EntityScheduler.run(plugin, entity, task, retired)` cho task gắn entity.
   - `GlobalRegionScheduler.run(plugin, task)` cho task không gắn region.
   - **TUYỆT ĐỐI KHÔNG** dùng `Bukkit.getScheduler().runTask()` (sẽ throw trên Folia).
3. **Tránh `Bukkit.getOnlinePlayers()` trong loop nóng** — O(n).
4. **Cache entity/chunk lookup** thay vì query mỗi tick.
5. **Tránh sync Bukkit event listener** chạy DB/HTTP — phải async + schedule callback.

### Plugin hot-path cần audit (xếp hạng risk)
| Plugin | Risk | Lý do |
|---|---|---|
| Hopper-heavy (sorting machine) | RẤT CAO | InventoryMoveItem event mỗi tick |
| Custom mob (MythicMobs, ModelEngine) | CAO | Tick custom AI |
| Worldedit/FAWE operation lớn | CAO | Block update spam |
| Economy plugin sync DB | RẤT CAO | Block main thread |
| Citizens NPC | CAO | Pathfinding mỗi tick |
| ProtocolLib heavy filter | CAO | Mỗi packet qua filter |
| ChestShop / Shop sign | TRUNG | Sign update event |
| Skript script phức tạp | RẤT CAO | Chạy interpreter trên main thread |

→ **Quy tắc**: plugin nào không có Folia support → không xài. Đừng "hack" `paperweight-userdev` để force compatibility.

---

## 10. Công thức cuối — "80% CPU formula"

```
CCU TARGET = (TICK_THREADS × 60) - (PLUGIN_OVERHEAD_FACTOR × 200)

Trong đó:
  TICK_THREADS         = C - 4
  PLUGIN_OVERHEAD_FACTOR ∈ [0.0, 1.0]
    0.0 = vanilla, 0.3 = SMP có economy/claim, 0.6 = full RPG, 0.9 = stress event
```

### Ví dụ Threadripper 7970X (32 core):
- TICK_THREADS = 28
- Vanilla SMP (overhead 0.2): CCU = 28×60 - 0.2×200 = 1680 - 40 = **1,640 CCU**
- DonutSMP economy + claim (overhead 0.4): CCU = 1680 - 80 = **1,600 CCU**
- PvP event đông (overhead 0.7): CCU = 1680 - 140 = **1,540 CCU**

→ Để CPU đạt 80% utilization với target trên, cần đảm bảo **player phân tán đều khắp region** (mục 8).

### Khi nào CPU < 60%?
- Player tập trung 1 chỗ → vài tick thread idle → CPU usage thấp nhưng MSPT cao. **Đây là trạng thái xấu**.
- → Fix: scatter player + multi-instance, KHÔNG phải mua CPU mạnh hơn.

### Khi nào CPU > 90% kéo dài?
- Tick thread saturate → đang **chạm trần**. Hoặc:
- GC ăn CPU → tune ZGC.
- Plugin sync I/O → audit theo mục 9.
- Mob/entity quá nhiều → tune spawn limit & activation range mục 3.

---

## 11. Roadmap thực thi (4 tuần)

### Tuần 1 — Baseline
- [ ] Cài monitoring: Prometheus + Grafana + node_exporter + mc-monitor.
- [ ] Chạy Spark 7 ngày liên tục, lưu báo cáo.
- [ ] Đo CPU usage hiện tại theo giờ peak.
- [ ] Liệt kê plugin nào ngốn TPS nhất (top 10).

### Tuần 2 — JVM + OS tuning
- [ ] Update Java 21+ (nếu chưa).
- [ ] Apply JVM flags ZGC ở mục 4.
- [ ] Apply sysctl + scheduler tuning mục 6.
- [ ] Setup numactl/taskset/isolcpus.
- [ ] Đo lại: CPU usage có tăng đều, MSPT có giảm variance?

### Tuần 3 — Folia config
- [ ] Áp file config mục 3 (paper-global.yml, paper-world-defaults.yml, spigot.yml).
- [ ] Tune thread pool theo công thức mục 2.
- [ ] Test stress: bot 500 player phân tán → đo TPS/CPU.
- [ ] Test stress: bot 200 player tập trung → xác nhận region split hoạt động.

### Tuần 4 — Plugin audit + workload
- [ ] Refactor plugin nội bộ async hết (theo mục 9).
- [ ] Implement anti-clustering plugin.
- [ ] Spread spawn/warp/hub.
- [ ] Re-test stress, xác nhận CPU 70-80% peak, MSPT < 40ms p99.

---

## 12. Checklist 1 trang (in ra dán cạnh ghế)

```
[ ] Java 21+ với ZGC generational
[ ] Heap = 50% RAM, AlwaysPreTouch, LargePages
[ ] TICK_THREADS = cores - 4
[ ] FJP parallelism = cores - 2
[ ] SMT/Hyperthreading: TẮT
[ ] CPU governor: performance
[ ] CPU C-states: tắt
[ ] numactl pin NUMA node + core
[ ] isolcpus cho tick threads
[ ] IRQ NIC pin core 0-1
[ ] sysctl: BBR, somaxconn 65535, hugepages
[ ] FS: XFS noatime, NVMe Gen4+
[ ] Folia config: per-player-mob-spawns true
[ ] Hopper move-event: disable
[ ] Treasure map: disable
[ ] Activation/tracking range giảm 30-50%
[ ] Spark profiler chạy thường trực
[ ] Prometheus + Grafana monitoring
[ ] Spawn/warp/hub scattered
[ ] Anti-clustering plugin chạy
[ ] Plugin sync I/O: ZERO
[ ] DB connection pool HikariCP
```

---

## 13. Quy luật vàng

1. **Đo trước, tune sau**. Spark + Prometheus là bắt buộc.
2. **80% CPU tận dụng = workload phân tán đều**, không phải "Folia ăn nhiều CPU hơn".
3. **Plugin sync I/O giết Folia nhanh hơn mọi thứ khác**. Audit định kỳ.
4. **Mua CPU 64+ core không cứu được nếu player tập trung 1 region**. Scale ngang trước.
5. **GC pause là kẻ giết MSPT thầm lặng**. ZGC + heap đúng = 80% bài toán giải xong.
6. **Reboot máy mỗi tuần** (rolling restart): JVM + OS state luôn fresh, không bị memory fragmentation.
7. **Chứng minh bằng số liệu**, không "tôi cảm thấy lag". MSPT p50/p95/p99 + CPU per-core histogram là sự thật.

---

*Tài liệu này là sống — cập nhật mỗi quý theo phiên bản Folia mới và kết quả benchmark thực tế của DonutSMP.*
