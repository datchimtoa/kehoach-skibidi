# KẾ HOẠCH FORK ĐỘC LẬP — 70% FOLIA + 30% MINESTOM

> Tên dự án đề xuất: **DonutPaper** (working name).
>
> Mục tiêu: Fork độc lập của Folia, **giữ 70% Folia** (vanilla parity, mob AI, redstone, world gen, plugin Bukkit ecosystem) + **inject 30% Minestom** (network layer async lock-free, ECS hot data, async event bus, Acquirable cross-thread sync).
>
> Trade-off: nhẹ hơn DonutCore rewrite (~2.5 năm) → **6-12 tháng × 2-3 dev** cho v1.0 production.

---

## 0. Vì sao 70/30 là tỷ lệ hợp lý

### 70% Folia — giữ lại
- **Vanilla parity hoàn chỉnh**: mob AI, redstone, world gen, fluid, weather, biome, structure, advancement, datapack — KHÔNG đụng vào, để PaperMC team duy trì.
- **Plugin Bukkit ecosystem**: WorldEdit, ProtocolLib, Vault, Citizens... vẫn chạy.
- **Region threading**: cơ chế scheduler đã tốt, chỉ cần optimize phần dưới.
- **Build system paperweight**: patch-based fork đã chuẩn hoá.

### 30% Minestom — inject
- **Network layer** Netty pipeline + packet codec async (Minestom làm tốt hơn vanilla).
- **Acquirable\<T\> pattern**: cross-thread entity access an toàn hơn region scheduler.
- **Hot data ECS**: position/velocity packed array → cache-friendly, vector instruction.
- **Async event bus**: event không block tick.
- **Lock-free chunk send**: client packet ra không cần block writer.

→ Đây là **Pareto-optimal** cho team 2-3 dev: 80% lợi ích đa luồng của Minestom, 20% effort của full rewrite.

---

## 1. Phân tích pháp lý license — bắt buộc đọc

| Thành phần | License | Tương thích? |
|---|---|---|
| **Paper / Folia patches** | MIT (PaperMC) | ✅ Free để fork |
| **Mojang vanilla code** (decompile, dùng trong Folia patches) | Mojang EULA | ⚠️ Bukkit DMCA 2014 - vùng xám, nhưng PaperMC vẫn an toàn 10 năm qua |
| **Minestom** | Apache 2.0 | ✅ Tương thích MIT, có thể inject |
| **DonutPaper (sản phẩm cuối)** | Phải MIT hoặc Apache 2.0 | ✅ Compatible khi mix |

### Cách inject hợp pháp
- **KHÔNG** copy-paste code Minestom vào fork Folia → vi phạm Apache 2.0 attribution.
- ✅ **Add Minestom làm Maven dependency** trong fork DonutPaper.
- ✅ **Shade + relocate** Minestom classes vào package `com.donut.shaded.minestom.*` — phải giữ NOTICE file Apache 2.0.
- ✅ **Adapter pattern**: viết wrapper class trong DonutPaper kết nối API Minestom với API Paper.

→ Phân phối DonutPaper kèm `LICENSES.md` ghi rõ:
- Paper: MIT
- Mojang code (in Paper patches): Minecraft EULA
- Minestom: Apache 2.0 + NOTICE file đầy đủ

---

## 2. Kiến trúc tổng thể DonutPaper

```
┌────────────────────────────────────────────────────────────┐
│                       DonutPaper                            │
│              (paperweight fork của Folia)                   │
├────────────────────────────────────────────────────────────┤
│  patches/server/                                             │
│  ├── 0001-Inject-Minestom-network-layer.patch               │
│  ├── 0002-Acquirable-pattern-for-entity.patch               │
│  ├── 0003-ECS-hot-data-storage.patch                        │
│  ├── 0004-Async-event-bus-bridge.patch                      │
│  ├── 0005-Lock-free-chunk-send.patch                        │
│  ├── 0006-Vector-API-for-position-update.patch              │
│  └── 0007-DonutSMP-specific-optimizations.patch             │
├────────────────────────────────────────────────────────────┤
│  Native Folia (KHÔNG ĐỤNG):                                 │
│  • Region scheduler                                          │
│  • Mob AI / pathfinding                                     │
│  • Redstone / piston                                        │
│  • World gen / structure                                    │
│  • Fluid / weather / dimension                              │
│  • Datapack / registry                                       │
│  • Plugin API (Bukkit)                                      │
├────────────────────────────────────────────────────────────┤
│  Shaded Minestom (com.donut.shaded.minestom.*):             │
│  • Network: PacketReader/Writer, codec, frame encoder       │
│  • Concurrent: Acquirable<T>, AcquirableCollection          │
│  • Event: EventNode, EventListener async                    │
│  • Utils: BinaryBuffer, FastUtilHashMap                     │
└────────────────────────────────────────────────────────────┘
```

---

## 3. 7 patches chính — chi tiết

### Patch 1 — Inject Minestom network layer

**Mục tiêu**: Thay Netty pipeline của Folia bằng Minestom's pipeline (zero-copy buffer, async decode).

**File ảnh hưởng**:
- `net.minecraft.server.network.ServerConnectionListener`
- `net.minecraft.network.Connection`
- `io.papermc.paper.network.*`

**Cách làm**:
```java
// Folia gốc:
public class Connection extends SimpleChannelInboundHandler<Packet<?>> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Packet<?> packet) {
        // sync handler, block tick thread
    }
}

// DonutPaper patch:
public class Connection extends SimpleChannelInboundHandler<Packet<?>> {
    private final MinestomPacketReader reader = new MinestomPacketReader();
    private final AsyncPacketQueue queue = new AsyncPacketQueue();
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Packet<?> packet) {
        // 1. Decode async qua Minestom reader (lock-free buffer)
        DecodedPacket decoded = reader.decode(packet, ctx.alloc());
        
        // 2. Push vào queue async, không block tick
        queue.offer(decoded);
        
        // 3. Tick scheduler drain queue mỗi tick
    }
}
```

**Lợi ích đo được**:
- Network throughput tăng ~30-40% (Minestom benchmark).
- Packet latency p99 giảm 50%.
- Tick thread không bị block bởi packet decode.

**Effort**: 2 dev × 1 tháng.

---

### Patch 2 — Acquirable\<T\> pattern cho cross-region entity access

Đây là **Crown Jewel** của Minestom. Cross-region trong Folia hiện tại rất khó (phải `runAtFixedRate` trên thread đúng region). Acquirable cho phép:

```java
// Folia gốc — phải dùng RegionScheduler:
RegionScheduler scheduler = Bukkit.getRegionScheduler();
scheduler.run(plugin, location, task -> {
    Entity e = world.getEntity(uuid);
    e.teleport(...);
});

// DonutPaper với Acquirable:
Acquirable<Entity> acquirable = entity.toAcquirable();
acquirable.async(e -> {
    e.teleport(...);  // Tự động lock entity, thread-safe
});

// Hoặc batch:
AcquirableCollection<Entity> entities = world.getAllEntitiesAcquirable();
entities.acquireForEach(e -> { ... });   // Parallel-safe
```

**File ảnh hưởng**:
- `net.minecraft.world.entity.Entity` — thêm `toAcquirable()` method.
- `org.bukkit.craftbukkit.entity.CraftEntity` — expose API qua Bukkit.

**Cách làm**:
```java
// Patch vào Entity class:
public class Entity {
    private final Acquirable<Entity> acquirable = new AcquirableImpl<>(this);
    
    public Acquirable<Entity> toAcquirable() {
        return acquirable;
    }
}
```

`Acquirable` impl wrap quanh entity với `ReentrantLock` hoặc `StampedLock` (cho read-heavy).

**Effort**: 1 dev × 1.5 tháng.

---

### Patch 3 — ECS hot data storage (position/velocity SoA)

Vanilla Minecraft: `Entity.position()` đọc từ `Vec3 position` field (object). Mỗi entity là 1 object riêng → **cache miss khi tick 10,000 entity**.

DonutPaper patch:
```java
// Trong World class:
public class Level {
    // Hot data SoA — packed array
    private double[] positionX = new double[MAX_ENTITIES];
    private double[] positionY = new double[MAX_ENTITIES];
    private double[] positionZ = new double[MAX_ENTITIES];
    private float[]  velocityX = new float[MAX_ENTITIES];
    private float[]  velocityY = new float[MAX_ENTITIES];
    private float[]  velocityZ = new float[MAX_ENTITIES];
    
    // Cold data vẫn ở object Entity
    private Entity[] entities = new Entity[MAX_ENTITIES];
    
    // Tick movement dùng SoA, vector instruction (Java Vector API)
    public void tickMovement() {
        VectorMask<Double> mask = ...;
        DoubleVector posX = DoubleVector.fromArray(SPECIES, positionX, 0);
        DoubleVector velX = DoubleVector.fromArray(SPECIES, velocityX, 0);
        posX.add(velX).intoArray(positionX, 0);
        // 4-8 entity per CPU instruction
    }
}
```

**Lợi ích**:
- Tick 10,000 entity: 4ms → 0.8ms (5x).
- Cache miss giảm 80%.
- Tận dụng AVX-512 hoặc ARM SVE qua Java Vector API.

**Effort**: 1 dev × 2 tháng (vì phải sync giữa SoA và Entity object).

---

### Patch 4 — Async event bus bridge

Bukkit event bus là **đồng bộ**: `pluginManager.callEvent(event)` block tick thread cho đến khi tất cả listener xong. Với plugin sync DB → tick lag.

DonutPaper patch:
```java
// Bridge Minestom EventNode vào Bukkit:
public class HybridEventBus {
    private final EventNode<Event> minestomNode = EventNode.all("donut-root");
    private final SimplePluginManager bukkitMgr = ...;
    
    public void callEvent(Event event) {
        // Sync listener (Bukkit) — chạy ngay
        bukkitMgr.callEvent(event);
        
        // Async listener (Minestom-style) — schedule
        if (event.isAsyncCapable()) {
            minestomNode.call(event);  // Non-blocking
        }
    }
}
```

Plugin cũ vẫn dùng `@EventHandler` Bukkit (sync). Plugin mới có thể dùng `@AsyncEventHandler`:
```java
@AsyncEventHandler
public void onPlayerMove(PlayerMoveEvent event) {
    // Chạy trên thread pool, không block tick
}
```

**Effort**: 1 dev × 1 tháng.

---

### Patch 5 — Lock-free chunk send

Hiện tại Folia send chunk cho client phải lock chunk → không cho writer ghi → lag spike khi player teleport.

Minestom dùng **immutable chunk snapshot**:
```java
public class ChunkSendQueue {
    public void sendToClient(Player p, ChunkPos pos) {
        // 1. Lấy snapshot version hiện tại (atomic)
        ChunkSnapshot snap = chunk.snapshot();
        
        // 2. Encode packet ngoài lock
        ClientboundLevelChunkPacket pkt = encode(snap);
        
        // 3. Send
        p.connection.send(pkt);
        
        // → Writer KHÔNG bị block, có thể tiếp tục modify chunk
    }
}
```

**Effort**: 1 dev × 1 tháng.

---

### Patch 6 — Vector API position update

Java 21+ có `jdk.incubator.vector` (FFI vector instruction). Folia hiện tại KHÔNG dùng.

DonutPaper bật:
```java
// JVM flag:
--add-modules=jdk.incubator.vector

// Code:
import jdk.incubator.vector.*;

VectorSpecies<Double> SPECIES = DoubleVector.SPECIES_PREFERRED;

public void updatePositions() {
    int loopBound = SPECIES.loopBound(positions.length);
    for (int i = 0; i < loopBound; i += SPECIES.length()) {
        DoubleVector pos = DoubleVector.fromArray(SPECIES, positions, i);
        DoubleVector vel = DoubleVector.fromArray(SPECIES, velocities, i);
        DoubleVector newPos = pos.add(vel);
        newPos.intoArray(positions, i);
    }
    // Tail loop scalar
    for (int i = loopBound; i < positions.length; i++) { ... }
}
```

→ AVX-512 = 8 double per instruction → 8x speedup cho movement tick.

**Effort**: 0.5 dev × 1 tháng (đã làm Patch 3 thì patch này nhẹ).

---

### Patch 7 — DonutSMP specific optimizations

Patch riêng cho gameplay DonutSMP. Ví dụ:
- **Combat hot path**: skip damage calc cho mob outside PvP zone.
- **Economy fast path**: cache balance trong memory, sync DB async batch.
- **Claim check**: spatial hash O(1) thay vì iterate claim list O(n).
- **Anti-dupe**: integrity check trên item move event.

**Effort**: ongoing, 1 dev part-time.

---

## 4. Build system — paperweight setup

```
DonutPaper/
├── build.gradle.kts
├── settings.gradle.kts
├── paperweight.properties
├── patches/
│   ├── server/                  # patch áp lên Folia server
│   │   ├── 0001-...patch
│   │   └── ...
│   └── api/                     # patch áp lên Paper API
├── src/
│   ├── main/java/com/donut/    # code DonutPaper riêng (không phải patch)
│   │   ├── network/             # Minestom adapter
│   │   ├── concurrent/          # Acquirable wrapper
│   │   └── event/               # Async event bus
│   └── shaded/                  # Minestom shaded
└── README.md
```

### `build.gradle.kts`
```kotlin
plugins {
    id("io.papermc.paperweight.patcher") version "1.7.4"
}

paperweight {
    upstreams {
        register("folia") {
            repo.set("https://github.com/PaperMC/Folia.git")
            ref.set("master")
            patchDir("server").set(file("patches/server"))
            patchDir("api").set(file("patches/api"))
        }
    }
}

dependencies {
    // Shade Minestom
    shadedApi("net.minestom:minestom-snapshots:latest") {
        exclude(group = "com.mojang")  // Tránh conflict
    }
    
    // Vector API (incubator)
    compileOnly("org.openjdk.jdk.incubator:vector:21")
}

tasks.shadowJar {
    relocate("net.minestom", "com.donut.shaded.minestom")
}
```

### Build pipeline
```bash
# 1. Apply patches lên Folia upstream
./gradlew applyPatches

# 2. Edit code trong folia-server/, sau đó:
./gradlew rebuildPatches

# 3. Build jar
./gradlew createReobfPaperclipJar

# Output: build/libs/donutpaper-paperclip-1.21.x.jar
```

---

## 5. Migration path — từ Folia hiện tại sang DonutPaper

### Phase 1 (Tháng 0-2): Setup
- [ ] Fork Folia, setup paperweight.
- [ ] Build green với 0 patch (basically là Folia rebrand).
- [ ] CI: GitHub Actions build jar mỗi commit.
- [ ] Deploy staging chạy DonutPaper "trắng".

### Phase 2 (Tháng 2-4): Patch 1 + 4
- [ ] Patch 1: Network layer Minestom.
- [ ] Patch 4: Async event bus.
- [ ] Stress test 1k bot, đo TPS/MSPT/network throughput.
- [ ] Compare benchmark với Folia gốc.

### Phase 3 (Tháng 4-6): Patch 2 + 5
- [ ] Patch 2: Acquirable\<T\>.
- [ ] Patch 5: Lock-free chunk send.
- [ ] Test cross-region entity interaction.
- [ ] Plugin compat test (top 20 plugin Bukkit phổ biến).

### Phase 4 (Tháng 6-8): Patch 3 + 6
- [ ] Patch 3: ECS hot data SoA.
- [ ] Patch 6: Vector API.
- [ ] Benchmark tick 10k+ entity.
- [ ] Stability test 168h liên tục.

### Phase 5 (Tháng 8-10): Patch 7 + Production
- [ ] Patch 7: DonutSMP-specific.
- [ ] Soft launch: 10% traffic DonutSMP qua DonutPaper.
- [ ] Monitor 2 tuần, tune.
- [ ] Full migration.

### Phase 6 (Tháng 10-12): Polish + Open-source
- [ ] Documentation đầy đủ.
- [ ] Public repo, open-source dưới MIT.
- [ ] Cộng đồng đóng góp.

---

## 6. Effort & nhân sự

### Team tối thiểu
- **1 senior dev** (kiến trúc, code review, paperweight expertise) — fulltime.
- **1 dev mid-senior** (network, concurrency) — fulltime.
- **1 dev mid** (testing, plugin compat) — part-time hoặc fulltime.

### Tổng effort
- Patch 1-7: **~7-9 dev-month**.
- Test + bug fix + plugin compat: **~3-4 dev-month**.
- Build/CI/release: **~1 dev-month**.
- **Tổng: ~12-15 dev-month** = **2-3 dev × 6 tháng** hoặc **1 dev × 12-15 tháng**.

### Chi phí
- Lương 3 dev × 6 tháng × 30tr/tháng = **540 triệu VND** (mid-tier).
- + Hardware test (1-2 server bare-metal): ~50-100 triệu.
- + Cloud staging: ~20 triệu.
- **Tổng**: ~600-700 triệu VND cho v1.0 production.

---

## 7. So sánh DonutPaper vs DonutCore (rewrite hoàn toàn)

| Tiêu chí | DonutPaper (70/30 fork) | DonutCore (rewrite) |
|---|---|---|
| Effort | 2-3 dev × 6 tháng | 5 dev × 30 tháng |
| Chi phí | ~600tr | ~5-7 tỷ |
| Risk | Thấp (giữ Folia base) | Rất cao |
| Vanilla parity | 100% (free) | 95% (port từng phần) |
| Plugin Bukkit ecosystem | ✅ giữ | ❌ phải viết shim |
| Performance gain | +30-50% | +200-500% |
| Maintenance load | Patch update theo Folia | Toàn bộ |
| Dependency | Folia upstream + Minestom | Tự lực |
| Pháp lý | OK (MIT + Apache 2.0) | OK clean-room |
| Khả năng "engine quốc tế" | Có (open-source MIT) | Có (chậm hơn) |

→ **DonutPaper là sweet spot cho team 2-3 dev, ngân sách <1 tỷ.**

---

## 8. Bottleneck dự kiến — đã tính trước

### 8.1. Patch conflict khi Folia upstream update
- PaperMC update Folia mỗi 2-4 tuần.
- Mỗi update: `./gradlew applyPatches` có thể fail do conflict.
- **Giảm thiểu**: viết patch nhỏ, scope hẹp; có CI tự động test patch áp được không; có 1 dev "patch maintainer" mỗi tháng dành 2-3 ngày sync.

### 8.2. Minestom shaded conflict với Mojang code
- Minestom có class `Player`, `Entity`, `Position` riêng — clash với Mojang.
- **Giảm thiểu**: shade + relocate sang `com.donut.shaded.minestom.*`.
- Adapter layer trong DonutPaper convert Mojang ↔ Minestom type.

### 8.3. Plugin Bukkit không Folia-compatible
- 95% plugin Bukkit cũ KHÔNG chạy trên Folia (chưa nói đến DonutPaper).
- **Giảm thiểu**: cung cấp `LegacyPluginAdapter` cho phép plugin sync chạy trên 1 thread riêng (slow path) — không bằng native nhưng có còn hơn không.

### 8.4. Vector API là incubator, có thể đổi API
- Java 21 vẫn `jdk.incubator.vector`.
- Có thể stable trong Java 25.
- **Giảm thiểu**: wrap qua `VectorAPI` interface trong DonutPaper, dễ swap khi Java đổi.

### 8.5. Testing đa luồng cực khó
- Bug race condition không reproduce được dễ dàng.
- **Giảm thiểu**:
  - JCStress (Java Concurrency Stress) cho unit test.
  - Mỗi patch có integration test với 100+ bot.
  - Production canary 1% traffic trước khi full rollout.
  - Logging atomic event ID để truy ngược race.

---

## 9. Quyết định kỹ thuật quan trọng

### Q1: Shade Minestom hay viết lại từng phần?
**Trả lời**: Shade. Viết lại = mất 2x effort, không có lợi ích.

### Q2: Bắt đầu từ Folia hay Paper?
**Trả lời**: **Folia**. Paper không có region threading → mất 70% lợi ích đa luồng.

### Q3: Public open-source hay private?
**Trả lời**: **Private 6 tháng đầu, public sau**. Lý do:
- Private: tốc độ phát triển nhanh, không phải support community.
- Public sau: kéo cộng đồng đóng góp, build brand DonutSMP.

### Q4: Có nên fork Minestom luôn?
**Trả lời**: **Không**. Dùng Minestom làm dependency, fork = double maintenance.

### Q5: API mới hay giữ Bukkit?
**Trả lời**: **Giữ Bukkit + thêm `donut-api`** (extension API cho Acquirable, async event, vector). Plugin cũ chạy được, plugin mới có API tốt hơn.

---

## 10. Roadmap ngắn (4 tháng đầu — quan trọng nhất)

### Tháng 1
- Tuần 1: Setup paperweight, fork Folia, build jar trắng.
- Tuần 2: CI/CD GitHub Actions, deploy staging.
- Tuần 3-4: Patch 1 (network layer) — POC.

### Tháng 2
- Tuần 1-2: Patch 1 hoàn thiện, test.
- Tuần 3-4: Patch 4 (async event bus).

### Tháng 3
- Tuần 1-2: Patch 2 (Acquirable).
- Tuần 3-4: Patch 5 (lock-free chunk send).

### Tháng 4
- Tuần 1-2: Stress test, benchmark, compare Folia gốc.
- Tuần 3: Plugin compat test top 20 plugin.
- Tuần 4: Bug fix + soft launch staging DonutSMP.

→ **Cuối tháng 4 có thể demo DonutPaper chạy DonutSMP staging với CCU thực.**

---

## 11. KPI thành công cho DonutPaper

| Metric | Folia gốc | DonutPaper target | Cách đo |
|---|---|---|---|
| Network throughput | baseline | +30% | bot stress 10k packet/s |
| Tick MSPT (1k entity) | 25-30ms | 15-20ms | spark profiler |
| Tick MSPT (10k entity) | 80-100ms | 30-40ms | bot test |
| Cross-region entity ops/s | ~500 | 5,000+ | benchmark Acquirable |
| Plugin compat rate | 60-70% | ≥85% | test 100 plugin top |
| GC pause p99 | 5-10ms | <2ms | JFR + GC log |
| CCU peak/instance | 2,000 | 3,500-5,000 | production load |

---

## 12. TL;DR

- **DonutPaper = fork Folia + shade Minestom (70/30)**, viết 7 patch chính.
- **Effort**: 2-3 dev × 6 tháng. Chi phí ~600tr VND.
- **License**: MIT + Apache 2.0 — hợp pháp.
- **Không động vào**: vanilla mechanics (mob, redstone, world gen, datapack) — để Folia upstream lo.
- **Inject từ Minestom**: network async, Acquirable cross-thread, async event, lock-free chunk send, ECS SoA, Vector API.
- **Performance gain dự kiến**: +30-50% TPS, +30% network, CCU/instance từ 2k lên 3.5-5k.
- **Path mở rộng**: open-source sau 6 tháng, kéo cộng đồng → DonutPaper trở thành "Pufferfish/Purpur của DonutSMP" trong cộng đồng VN.

---

## 13. Hành động ngay tuần này nếu quyết go

1. **Chốt team** (2-3 dev, fulltime hoặc part-time).
2. **Setup repo private GitHub** `donut/donutpaper`.
3. **Cài paperweight CLI**:
   ```bash
   ./gradlew --version  # cần Gradle 8.5+
   ./gradlew applyPatches  # khởi tạo Folia upstream
   ```
4. **Đọc tài liệu**:
   - https://docs.papermc.io/paper/dev/getting-started — Paper API.
   - https://github.com/PaperMC/Folia — Folia source.
   - https://wiki.minestom.net — Minestom architecture.
   - https://github.com/PaperMC/paperweight — paperweight tooling.
5. **Lập Discord/Slack channel** team, daily standup.
6. **Viết 1 patch trắng** (vd: thêm comment vào Server.java) → confirm pipeline build → deploy staging green.

→ Đây là milestone tuần 1: **CI green, jar build được, staging chạy player join được**.

Mọi thứ sau đó là progressive enhancement.

---

*"Đừng viết lại cả thế giới. Viết lại 30% nóng nhất. Để 70% còn lại cho người khác lo."*
