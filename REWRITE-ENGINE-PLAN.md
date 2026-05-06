# KẾ HOẠCH REWRITE ENGINE — "FOLIA + MINESTOM" HYBRID

> Mục tiêu: Engine có **đa luồng cực mạnh như Minestom** (region-free, lock-free) **+ "quy luật tự nhiên" parity vanilla như Folia** (mob AI, redstone, world gen, weather, dimension, advancement, datapack...).
>
> Tên dự án đề xuất: **DonutCore** (working name).

---

## 0. Vấn đề cốt lõi — tại sao hai con đường mâu thuẫn

| Khía cạnh | Folia | Minestom | Cái ta muốn |
|---|---|---|---|
| Parity vanilla | ~100% (fork Mojang code) | 0% (rewrite từ đầu) | ~95% |
| Multi-thread design | Region-based, có giới hạn | Lock-free, ECS-like, scale tuyến tính | Lock-free + đa region |
| Plugin ecosystem | Bukkit (>15 năm) | Tự viết tất cả | API mới + adapter Bukkit cơ bản |
| Effort | 0 (xài sẵn) | 6-12 tháng (nếu chỉ minigame) | **3-7 năm × team 5-10 người** |
| Maintenance | PaperMC team duy trì | Minestom team duy trì | **DonutCore team duy trì 1 mình** |

**Kết luận thẳng**: Cái ông muốn về mặt kỹ thuật là **đúng và làm được**, nhưng nó là dự án 5-10 dev × 3-5 năm. Trước khi viết 1 dòng code, phải hiểu **chi phí thực sự**.

---

## 1. Cách tiếp cận sai — fork Folia rồi "viết lại đa luồng hơn"

Đây là cái bẫy mà ai cũng bước vào:
1. Fork Folia.
2. "Tôi sẽ thay region scheduler bằng ECS lock-free."
3. 6 tháng sau: nhận ra **tất cả 200,000 dòng code Mojang** ngầm giả định single-thread:
   - `ServerLevel.getEntity(UUID)` không thread-safe.
   - Block update đệ quy vào neighbor → cross-region race.
   - `BlockEntity.tick()` truy cập world state trực tiếp.
   - LevelChunk có internal mutable map, không có lock.
4. Mọi method bạn "song song hoá" đều phá vỡ giả định ngầm khác.
5. Bug không reproduce được, world corruption ngẫu nhiên.
6. **Bỏ cuộc**.

→ Folia đã đi 80% con đường này (region threading), thêm 20% còn lại = effort × 10 lần. Lý do PaperMC chưa làm là vì **không ai làm được trong khuôn khổ kế thừa code Mojang**.

---

## 2. Cách tiếp cận đúng — "Minestom + Vanilla Layer"

Đây là kiến trúc thực tế nhất:

```
┌────────────────────────────────────────────────────────┐
│              DonutCore (Java 21+, lock-free ECS)        │
├────────────────────────────────────────────────────────┤
│  Layer 4: Plugin API + Bukkit compat shim (optional)   │
├────────────────────────────────────────────────────────┤
│  Layer 3: Vanilla Mechanics (PORT từng phần)            │
│  ┌──────┬──────┬─────────┬────────┬──────┬──────────┐  │
│  │ Mob  │Redst │ WorldGen│ Weather│Fluid │Advancement│  │
│  │ AI   │ one  │         │        │      │           │  │
│  └──────┴──────┴─────────┴────────┴──────┴──────────┘  │
├────────────────────────────────────────────────────────┤
│  Layer 2: Data-Driven Layer (đọc datapack Mojang)       │
│  ┌────────┬──────┬─────────┬────────┬──────┬────────┐  │
│  │Recipe  │Loot  │Advance  │WorldGen│Tag   │Block   │  │
│  │        │table │ment     │preset  │      │state   │  │
│  └────────┴──────┴─────────┴────────┴──────┴────────┘  │
├────────────────────────────────────────────────────────┤
│  Layer 1: Core Engine (NỀN TẢNG — viết 1 lần)           │
│  ┌────────┬──────┬─────────┬────────┬──────┬────────┐  │
│  │Network │Chunk │Entity   │Block   │Event │Schedule│  │
│  │protocol│store │ECS      │palette │bus   │r       │  │
│  └────────┴──────┴─────────┴────────┴──────┴────────┘  │
└────────────────────────────────────────────────────────┘
```

### Triết lý
- **Layer 1 viết một lần, không chạm lại** (giống Minestom).
- **Layer 2 KHÔNG viết, chỉ đọc** datapack Mojang vanilla → 30% gameplay miễn phí (recipe, loot, advancement, tag, block state, world gen preset).
- **Layer 3 port từng cơ chế** — chia nhỏ, mỗi cơ chế là 1 module độc lập, port song song được.
- **Layer 4 optional** — nếu muốn tương thích Bukkit plugin có sẵn.

### Tại sao đi đường này?
1. **Layer 1 là nơi đa luồng được thiết kế từ đầu** — không có giả định single-thread nào để phá.
2. **Layer 3 mỗi cơ chế độc lập** — port được song song, không block release.
3. **Layer 2 leverage Mojang free** — tiết kiệm 1-2 năm effort.
4. **MVP có thể chạy survival cơ bản trong 6-9 tháng**, parity 95% trong 2-3 năm.

---

## 3. Layer 1 — Core Engine (nền tảng đa luồng)

### 3.1. Concurrency model: ECS lock-free

Bỏ hoàn toàn mô hình **OOP-Bukkit** (`Player extends LivingEntity extends Entity ...`). Dùng **Entity Component System**:

```
World {
  Storage<Position>   positions    // SoA — packed array
  Storage<Velocity>   velocities
  Storage<Health>     healths
  Storage<MobAI>      mob_ai
  Storage<PlayerConn> player_conns
  ...
}

Entity = u64 ID (không phải object)
```

- **System** (logic) đọc/ghi component cụ thể, chạy song song nếu không xung đột.
- **Schedule** giải quyết dependency giữa system → tự động song song hoá.
- Tham khảo: Bevy ECS (Rust), Flecs (C++), JEcs (Java).

### 3.2. Chunk storage

Folia dùng `LevelChunk` của Mojang — đầy mutable HashMap. Ta dùng:
- **Block palette** packed (4-bit khi <16 block type, 8-bit khi <256, full khi >256).
- **Section-based** (16x16x16 = 4096 block) như Anvil format.
- **Chunk = array of Section + lookup table** — không lock toàn chunk, lock per-section nếu cần.
- **Copy-on-write** cho read-heavy: snapshot chunk cho client send, không block writer.

### 3.3. Network layer

- **Netty 4** với epoll/io_uring (Linux) / kqueue (BSD).
- **Packet decode/encode async** — không trên main thread.
- **Per-connection thread** cho heavy packet (chunk send), pool cho light packet.
- **Compression**: zlib với native binding hoặc Zstd (Mojang đã chuyển dần).

### 3.4. Tick model: time-step independent

Vanilla: 20 tick/giây cứng = 50ms/tick. Mọi thứ tick cùng nhịp.

DonutCore proposal:
- **Server tick**: 20 Hz cho game logic chính.
- **Physics tick**: 60 Hz cho entity movement (smoother PvP).
- **Mob AI tick**: 10 Hz (giảm tải, không cần 20 Hz).
- **Block update tick**: event-driven, không tick định kỳ.
- **Weather/lighting**: tick độc lập, async.

→ Mỗi loại tick chạy **thread pool riêng**, schedule không xung đột nhờ ECS dependency graph.

### 3.5. Networking với client vanilla

Đây là **constraint tuyệt đối**: client là Mojang vanilla, không được mod.
- Phải nói **Minecraft Java Edition protocol** đúng (1.21.x hiện tại).
- Phải gửi packet đúng thứ tự, đúng timing.
- Việc tick logic 60 Hz chỉ là server-side — client vẫn nhận snapshot 20 Hz qua `ClientboundMoveEntityPacket` standard.

→ Đây là chỗ phải đọc kỹ [wiki.vg/Protocol](https://minecraft.wiki/w/Java_Edition_protocol). Mọi packet Mojang gửi/nhận phải bit-perfect.

---

## 4. Layer 2 — Data-driven (KHÔNG VIẾT, chỉ đọc Mojang)

Mojang đã chuyển ~30% gameplay sang **data-driven JSON** từ 1.13:

| Data | Path trong vanilla jar | Mô tả |
|---|---|---|
| `recipes/` | `data/minecraft/recipes/*.json` | Tất cả công thức craft |
| `loot_tables/` | `data/minecraft/loot_tables/**/*.json` | Drop khi đập block / kill mob |
| `advancements/` | `data/minecraft/advancements/**/*.json` | Hệ thống achievement |
| `tags/` | `data/minecraft/tags/**/*.json` | Nhóm block/item/entity (e.g. `#wool`) |
| `worldgen/` | `data/minecraft/worldgen/**/*.json` | Biome, structure, noise setting |
| `dimension/` | `data/minecraft/dimension/*.json` | Overworld, Nether, End config |
| `damage_type/` | `data/minecraft/damage_type/*.json` | (1.20+) Loại damage |
| `chat_type/` | `data/minecraft/chat_type/*.json` | Format chat |
| `trim_pattern/` | `data/minecraft/trim_pattern/*.json` | Armor trim (1.20+) |

→ DonutCore **đọc trực tiếp** các JSON này. Khi Mojang update version mới, ta chỉ cần update version data, **không phải viết lại logic**.

**Pháp lý**: data JSON này nằm trong `minecraft-server.jar` chính thức Mojang. Phân phối kèm DonutCore = vi phạm. Cách hợp pháp:
- DonutCore tự download `server.jar` từ Mojang launcher meta API tại runtime.
- Extract data → cache.
- Tương tự cách `Glowstone`, `Minestom-Vanilla` làm.

→ Layer 2 = ~500-1000 dòng code (parser + cache), chứ không phải viết lại 50,000 dòng recipe/loot.

---

## 5. Layer 3 — Vanilla Mechanics (port từng cơ chế)

Đây là phần **tốn thời gian nhất**, nhưng có thể **chia nhỏ** và port song song.

### 5.1. Inventory cơ chế cần port

Xếp hạng theo độ phức tạp & ưu tiên cho SMP:

| # | Cơ chế | Độ khó | Ưu tiên | LOC ước lượng | Tham khảo |
|---|---|---|---|---|---|
| 1 | Block placement/break | Dễ | P0 | ~2k | Mojang `BlockBehaviour` |
| 2 | Inventory + crafting | Trung | P0 | ~3k | Recipe JSON + container UI |
| 3 | Damage + armor | Trung | P0 | ~2k | Damage type JSON |
| 4 | Combat + PvP | Trung | P0 | ~3k | Knockback formula, attribute |
| 5 | Hunger + food | Dễ | P0 | ~1k | Food properties |
| 6 | Sleep + spawn point | Dễ | P0 | ~500 | |
| 7 | World generation | RẤT KHÓ | P0 | ~15k | Noise, biome, structure |
| 8 | Mob spawning | Khó | P0 | ~3k | Per-biome spawn list |
| 9 | Mob AI (passive) | Khó | P1 | ~8k | Goal-based AI per mob |
| 10 | Mob AI (hostile) | Khó | P1 | ~10k | + targeting + pathfinding |
| 11 | Pathfinding (A*) | Khó | P0 | ~3k | NodeEvaluator, Path |
| 12 | Redstone | RẤT KHÓ | P1 | ~6k | Wire signal propagation |
| 13 | Pistons | Khó | P1 | ~2k | Block move event |
| 14 | Fluid (water/lava) | Khó | P1 | ~3k | Flow simulation |
| 15 | Fire spread | Trung | P1 | ~1k | |
| 16 | Weather + lightning | Trung | P1 | ~1k | |
| 17 | Day/night + sky | Dễ | P0 | ~500 | |
| 18 | Crops + farmland | Trung | P1 | ~1k | |
| 19 | Bee + breeding | Trung | P2 | ~2k | |
| 20 | Villager + trading | RẤT KHÓ | P2 | ~5k | Profession + POI |
| 21 | Raid | Khó | P2 | ~2k | |
| 22 | Boss (dragon/wither) | RẤT KHÓ | P2 | ~5k | |
| 23 | Portals (nether/end) | Khó | P1 | ~2k | Coordinate translation |
| 24 | Enchantment | Trung | P0 | ~2k | |
| 25 | Potion/effect | Trung | P0 | ~2k | |
| 26 | Brewing | Trung | P1 | ~1k | |
| 27 | Furnace/smelting | Dễ | P0 | ~1k | |
| 28 | Hopper/dropper | Trung | P0 | ~2k | |
| 29 | Beacon | Trung | P2 | ~1k | |
| 30 | Conduit | Trung | P3 | ~500 | |
| 31 | Sculk + warden | RẤT KHÓ | P3 | ~5k | |
| 32 | Trial chamber (1.21) | Khó | P3 | ~3k | |

**Tổng LOC ước lượng**: ~100,000-150,000 dòng code Java cho parity vanilla 95%.

### 5.2. World generation — phức tạp nhất

Đây là **nửa effort** của cả Layer 3. Có 3 lựa chọn:

#### Option A — Tự rewrite hoàn toàn
- Implement Perlin/Simplex noise, density function, structure placement.
- Match seed-for-seed với Mojang? **Gần như không thể** (Mojang dùng nhiều magic constant).
- Nếu chấp nhận seed khác → effort vừa phải (~1 năm).

#### Option B — Sub-process Mojang vanilla server
- Chạy Mojang `server.jar` ẩn ở chế độ "world gen only".
- DonutCore yêu cầu chunk → Mojang gen → trả về.
- Nhược: chậm, cross-process IPC, vẫn phụ thuộc code Mojang.

#### Option C — Pre-gen + load chunk
- Dùng Mojang server pre-gen world rộng (10k×10k chunk) → lưu vào region file.
- DonutCore chỉ load, không gen.
- Nhược: world có giới hạn, không infinite.
- Phù hợp cho **map-based server** (Hypixel, BedWars), không phù hợp infinite SMP.

→ **DonutCore khuyến nghị: Option A** với chấp nhận seed khác vanilla. World gen của ta sẽ "feel like Minecraft" nhưng không seed-identical. **99% người chơi không quan tâm**.

### 5.3. Mob AI — phức tạp thứ hai

Vanilla dùng **Goal-based AI** (mỗi mob có list goal, mỗi tick chọn goal priority cao nhất khả thi):
```
Zombie goals:
  - FloatGoal (avoid drowning)
  - ZombieAttackGoal (attack target)
  - MoveTowardsTargetGoal
  - MoveThroughVillageGoal
  - WaterAvoidingRandomStrollGoal
  - LookAtPlayerGoal
  - RandomLookAroundGoal
```

→ Port theo từng mob, mỗi mob ~200-500 LOC. Có ~50 mob trong game → ~15-25k LOC.

**Tối ưu khác Mojang**:
- Pathfinding chạy **trên thread pool riêng**, kết quả callback.
- Mob outside view-distance: **freeze** hoàn toàn (Mojang đã làm 1 phần).
- Goal evaluation cache 5-10 tick (không cần đánh giá mỗi tick).

### 5.4. Redstone — phức tạp thứ ba

Redstone vanilla là **single-threaded recursion hell**. Update lan toả O(n) với n = số block neighbor.

→ Có thể tham khảo **MCHPRS** (Minecraft High Performance Redstone Server, viết bằng Rust):
- Compile redstone graph thành DAG.
- Tick toàn graph trong 1 lần (vector instruction).
- Nhanh hơn vanilla 1000x cho mạch lớn.

DonutCore có thể dùng tương tự: mỗi region redstone là 1 graph, tick parallel.

---

## 6. Concurrency model chi tiết

### 6.1. Schedule graph

Mỗi tick, scheduler tạo DAG dependency:

```
[Network IO]──┐
              ├─→ [Player input apply]──┐
[Player tick]─┘                         │
                                        ├─→ [World update]──┐
[Mob AI tick]──┐                        │                   │
               ├─→ [Entity move]────────┘                   │
[Physics tick]─┘                                            │
                                                            ├─→ [Send packet]
[Block update]──┐                                           │
                ├─→ [Block entity tick]────────────────────┘
[Fluid tick]────┘
```

Mỗi node là 1 system → chạy trên thread pool. Dependency edge = phải đợi xong mới chạy.

### 6.2. Lock-free chunk write

- Chunk có **version counter** (atomic).
- Reader chụp snapshot version → đọc.
- Writer increment version → ghi vào copy → atomic swap pointer.
- Reader nào đang dùng version cũ vẫn an toàn.

Tham khảo: **RCU (Read-Copy-Update)** từ Linux kernel.

### 6.3. Cross-region interaction

- Player A region 1 đánh player B region 2.
- DonutCore: **message passing** giữa region (như Erlang actor model).
- Region 1 gửi `DamageEvent{target: B, amount: 5}` vào queue region 2.
- Region 2 tick lần sau xử lý → áp damage.
- Latency = 1 tick (50ms). Chấp nhận được cho PvP.

---

## 7. Plugin API — nên thiết kế thế nào

### Lựa chọn 1 — API mới hoàn toàn (như Minestom)
```java
node.addListener(EntityAttackEvent.class, event -> {
    Entity attacker = event.getEntity();
    Entity target = event.getTarget();
    // ...
});
```
- Pros: clean, async-first, không kế thừa nợ kỹ thuật.
- Cons: plugin Bukkit **không chạy được** → phải tự viết tất cả.

### Lựa chọn 2 — Compat shim Bukkit (như SpongeForge)
- Implement `org.bukkit.*` interface map sang DonutCore API.
- Plugin Bukkit cũ chạy được (mostly).
- Pros: ecosystem có sẵn.
- Cons: kéo theo single-thread assumption Bukkit → mất lợi thế đa luồng.

### Khuyến nghị: Lựa chọn 1 cho năm 1-3, Lựa chọn 2 sau khi core stable
- API mới native từ đầu.
- Năm 4-5 thêm shim cho plugin Bukkit phổ biến (Vault, ProtocolLib, WorldEdit) — không phải để chạy mọi plugin, chỉ những plugin "must-have".

---

## 8. Effort thực tế — số liệu trần trụi

### So sánh với dự án thực
| Dự án | Team | Năm | Trạng thái |
|---|---|---|---|
| Minestom | ~5-10 dev | 2020-nay | Stable, không có vanilla layer |
| Glowstone | ~10 dev (peak) | 2011-nay | Stop dev, lag versions |
| Cuberite | ~5 dev | 2011-nay | Maintenance, không follow vanilla mới |
| Cytonic | ~3 dev | 2022-nay | Minigame only, dùng Minestom |
| HollowCube | ~2 dev | 2023-nay | Vanilla layer trên Minestom — chưa hoàn thiện |
| Mojang vanilla | ~30 dev fulltime | 2009-nay | Reference |

### Estimation cho DonutCore (parity 95% vanilla)
- **Layer 1 (core)**: 2 dev × 8 tháng = 16 dev-month.
- **Layer 2 (data-driven loader)**: 1 dev × 2 tháng = 2 dev-month.
- **Layer 3 (vanilla mechanics)**:
  - World gen: 2 dev × 10 tháng = 20 dev-month.
  - Mob AI (50 mob): 2 dev × 12 tháng = 24 dev-month.
  - Redstone + piston: 1 dev × 6 tháng = 6 dev-month.
  - Combat/PvP/damage: 1 dev × 4 tháng = 4 dev-month.
  - Inventory/craft/furnace: 1 dev × 4 tháng = 4 dev-month.
  - Block update + fluid: 1 dev × 6 tháng = 6 dev-month.
  - Villager/raid/boss/portal: 2 dev × 8 tháng = 16 dev-month.
  - Còn lại (potion, enchant, weather, sleep...): 1 dev × 6 tháng = 6 dev-month.
- **Layer 4 (plugin API + shim)**: 1 dev × 6 tháng = 6 dev-month.
- **Network + protocol upkeep** (mỗi version Mojang ra): 1 dev × ongoing.
- **Test + benchmark + bug fix**: 30% tổng effort = ~30 dev-month.

**Tổng**: ~140 dev-month = **5 dev × 28 tháng** = **~2.5 năm với team 5 người fulltime**.

Với team part-time 3 người → **5-7 năm**.

---

## 9. Lộ trình thực thi — chia milestone

### M1 (Tháng 0-3): Skeleton
- [ ] Network layer + protocol 1.21.x.
- [ ] Login/handshake/keepalive.
- [ ] Player join thấy void world (như Limbo).
- [ ] Có thể chat, /tp, /gamemode.
- **Demo**: 1000 bot ngồi không trong void, CPU < 5%.

### M2 (Tháng 3-6): Block & inventory
- [ ] Chunk storage + palette.
- [ ] Block place/break.
- [ ] Inventory + crafting (đọc recipe JSON).
- [ ] Container (chest, furnace).
- [ ] Pre-gen flat world.
- **Demo**: build creative cơ bản, craft tool.

### M3 (Tháng 6-12): Survival cơ bản
- [ ] Damage + armor + food.
- [ ] Mob AI cho zombie/skeleton/cow/pig (4 mob đại diện).
- [ ] Pathfinding A*.
- [ ] World gen overworld (đơn giản — không structure).
- [ ] Day/night + sleep.
- **Demo**: chơi survival qua 1 đêm, kill mob, kiếm food.

### M4 (Tháng 12-18): Vanilla parity 70%
- [ ] World gen full (biome, cave, structure village/temple).
- [ ] 30 mob common.
- [ ] Redstone cơ bản (wire, torch, repeater).
- [ ] Fluid simulation.
- [ ] Nether + portal.
- **Demo**: chơi survival full overworld + nether.

### M5 (Tháng 18-24): Vanilla parity 90%
- [ ] Tất cả mob (50+).
- [ ] Villager + trade + raid.
- [ ] End dimension + dragon.
- [ ] Redstone full + piston quirk.
- [ ] Enchantment + potion full.
- **Demo**: nhập world Mojang vanilla, chơi như bình thường.

### M6 (Tháng 24-30): Polish + production
- [ ] Plugin API stable v1.0.
- [ ] Bukkit compat shim (Vault, ProtocolLib).
- [ ] DataConverter (load world Mojang version cũ).
- [ ] Anti-cheat hook.
- [ ] Documentation đầy đủ.
- **Production**: DonutSMP chạy thử trên DonutCore với 100 player.

### M7+ (Tháng 30+): Tận dụng
- [ ] Stress test 5,000 CCU 1 instance.
- [ ] Migrate DonutSMP gradually.
- [ ] Open-source Layer 1+2 (giống Minestom) — kéo cộng đồng đóng góp.

---

## 10. Pháp lý & EULA — cần chú ý

### Được phép
- Reverse-engineer protocol → wiki.vg đã làm 14 năm, Mojang chấp nhận.
- Đọc datapack từ `server.jar` mà player đã có (KHÔNG redistribute jar).
- Implement gameplay tương tự — gameplay không phải tài sản trí tuệ riêng.

### Không được phép
- Phân phối DonutCore kèm `server.jar` Mojang.
- Decompile `server.jar` → copy code Mojang → paste vào DonutCore. Phải **clean-room** (đọc spec, viết lại từ đầu).
- Dùng tên "Minecraft" trong tên/branding DonutCore.

### Tham khảo pháp lý
- **Bukkit/Spigot từng có vấn đề** (DMCA 2014) vì dùng quá nhiều code Mojang qua decompile.
- **Glowstone, Minestom, Cuberite không có vấn đề** vì rewrite từ đầu.
- → DonutCore phải đi theo Glowstone/Minestom: **clean-room rewrite**.

---

## 11. Quyết định — đáng làm không?

### Khi NÊN làm
- DonutSMP có doanh thu >2 tỷ/tháng → fund được team 5 dev fulltime.
- Đã chạm trần Folia + scale ngang đa instance phức tạp.
- Có ý định trở thành **platform** (cho server khác dùng), không chỉ vận hành 1 server.
- Có CTO senior từng làm distributed system.

### Khi KHÔNG NÊN làm
- DonutSMP <500 CCU → vô lý, Folia thừa sức.
- Team <3 dev senior → impossible.
- Chỉ vì "muốn cool" → 99% bỏ giữa chừng.
- Mong muốn "Minecraft 2x performance" mà không có gameplay khác biệt.

### Path thay thế khôn ngoan hơn
1. **Fork Folia + custom patches** (như Pufferfish) — 1 dev × năm, 80% lợi ích.
2. **Microservice backend ngoài Minecraft** — economy/social/leaderboard chạy ngoài, MC chỉ lo block & combat.
3. **Multi-instance Velocity + Folia** — scale ngang trước khi scale dọc.
4. **Đợi Folia tiến hoá** — PaperMC team đang làm rất tốt, chờ 1-2 năm có thể có giải pháp.

---

## 12. Quyết định cuối — tôi khuyên gì

**Thẳng thắn:**

DonutCore là **dự án đáng làm về mặt kỹ thuật**, đẹp về kiến trúc. Nhưng với DonutSMP hiện tại:

1. **Không phải bottleneck**. CCU hiện tại của DonutSMP thực tế (vài trăm — vài nghìn) → Folia + tuning + multi-instance là **thừa đủ**.
2. **Cost cực cao**. 5 dev × 2.5 năm = ~5-7 tỷ VND lương + chưa tính cơ hội.
3. **Risk cao**. 60% dự án rewrite engine bỏ giữa chừng (industry stat).
4. **Không phải lý do người chơi đến**. Player đến vì gameplay/cộng đồng, không ai biết server chạy Folia hay DonutCore.

**Nhưng nếu vẫn muốn làm**, cách đúng là:

- **Năm 1**: Skunkworks team 2-3 dev viết MVP DonutCore (Layer 1). Không production.
- **Năm 2**: Demo nội bộ, đo benchmark, quyết định go/no-go.
- **Năm 3-4**: Nếu go — ramp team lên 5-7, port Layer 3.
- **Song song**: DonutSMP production vẫn chạy Folia + scale ngang.
- **Năm 5**: Migrate gradually nếu DonutCore stable.

**Hoặc** — và đây là path tôi nghĩ thực tế nhất:

→ **Đóng góp cho HollowCube / MinestomVanilla** thay vì viết từ đầu.
- Đây là dự án open-source đang làm chính xác việc "vanilla layer trên Minestom".
- Tham gia, đẩy thêm dev → leverage cộng đồng → DonutSMP có engine tự chủ mà không tự cô đơn build từ scratch.
- Effort giảm 60-70%.

Link: https://github.com/hollow-cube/minestom-ce | https://github.com/Project-Cepi (vanilla-on-Minestom community).

---

## 13. TL;DR

- "Folia + Minestom" hybrid = đúng về mặt kiến trúc, gọi là **DonutCore**.
- Không phải fork Folia rồi rewrite — đó là bẫy.
- Đúng đường: **Minestom làm core + port từng vanilla mechanic** thành layer độc lập.
- Effort thực: **5 dev × 2.5 năm** cho parity 95%.
- Datapack Mojang cho 30% gameplay miễn phí (recipe, loot, advancement, world gen preset).
- Plugin API mới native, optional Bukkit shim sau.
- Pháp lý OK nếu clean-room, không redistribute Mojang jar, không dùng tên Minecraft.
- **Khuyến nghị**: chưa làm bây giờ. Đóng góp HollowCube/Project Cepi trước. Khi DonutSMP đạt 5k+ CCU và chạm trần Folia + multi-instance, lúc đó mới nghiêm túc cân nhắc.

*Engine tự viết là vũ khí mạnh nhất — nhưng cũng là cái lỗ chôn xác dự án nhiều nhất trong lịch sử Minecraft server.*
