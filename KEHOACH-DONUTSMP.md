# KẾ HOẠCH PHÁT TRIỂN DONUTSMP — TĂNG NGƯỜI CHƠI & TRỞ THÀNH SERVER QUỐC GIA

> Vai trò: Dev có toàn quyền điều khiển server, kiểm soát toàn bộ logic & code.
> Mục tiêu: Tăng lượng người chơi đột biến, hệ thống game mạnh mẽ, được công nhận là **server quốc gia** (top server của một quốc gia / khu vực).

---

## 0. Tóm tắt định hướng (TL;DR)

Để DonutSMP "lên đỉnh", phải đánh đồng thời 5 trục:

1. **Hạ tầng & hiệu năng** — không lag, không downtime, scale tới 5k+ CCU.
2. **Gameplay khác biệt** — có "bản sắc" riêng, không clone Hypixel/2b2t.
3. **Onboarding & retention** — người mới ở lại, người cũ không bỏ.
4. **Kinh tế & nội dung sống** — kinh tế cân bằng, sự kiện liên tục.
5. **Cộng đồng & marketing** — content creator, mạng xã hội, hợp tác chính thức để được công nhận.

Roadmap chia 4 giai đoạn: **Ổn định (M1–M2) → Bùng nổ (M3–M5) → Mở rộng (M6–M9) → Công nhận quốc gia (M10–M12)**.

---

## 1. Hạ tầng kỹ thuật & hiệu năng (Backbone)

### 1.1. Core server
- Chuyển sang **Paper / Folia** (Folia cho khu vực hub & sự kiện đông người — region-based threading, scale CCU gấp 3–5 lần Paper).
- Bật **`view-distance` thấp (4–6)** + **`simulation-distance` 3–4** ở hub/event, cao hơn (8–10) ở SMP world.
- **Async chunk I/O** (đã có sẵn trong Paper), ưu tiên SSD NVMe, RAID10.
- **JVM flags**: ZGC (Java 21+) thay G1GC cho heap >32GB; tuning `-XX:+UseZGC -XX:+ZGenerational` để giảm GC pause < 5ms.
- Tách **proxy (Velocity)** ra trước cụm backend để load-balance + bảo vệ DDoS.

### 1.2. Kiến trúc multi-server
```
                 ┌──────────────┐
                 │  Velocity    │  ← proxy duy nhất, chống DDoS L7
                 └──────┬───────┘
       ┌────────┬───────┼───────┬────────┐
       ▼        ▼       ▼       ▼        ▼
     Hub     SMP-1    SMP-2   Event    Minigame
   (Folia)  (Paper) (Paper) (Folia)   (Paper)
       │        │       │       │        │
       └────────┴───┬───┴───────┴────────┘
                   ▼
           Redis  +  MariaDB/Postgres   (shared state, economy, players)
                   ▼
              S3/MinIO (world backups, logs)
```
- **Cross-server data**: Redis cho session/economy cache, Postgres cho persistent.
- **World sharding**: chia SMP world thành nhiều shard theo toạ độ → mỗi shard 1 process.

### 1.3. Bảo mật & chống DDoS
- **TCPShield / OVH GAME / path.net** trước proxy.
- **Whitelist IP backend**, chỉ proxy mới được kết nối.
- **Rate-limit packet** (PacketEvents) chống bot-join, exploit packet.
- **Anti-crash**: Patch các exploit chunk-ban, book-crash, NBT-crash (NoChatReports, ProtocolLib filter).

### 1.4. Monitoring & SRE
- **Prometheus + Grafana**: TPS, MSPT, CCU, GC time, DB latency mỗi server.
- **Loki / ELK**: log tập trung.
- **Sentry / GlitchTip**: bắt exception plugin tự viết.
- **Alerting** (PagerDuty/Discord webhook): TPS < 18 trong 1 phút → alert; CCU drop > 30% → alert.
- **SLO**: 99.9% uptime, MSPT < 40ms p95.

### 1.5. Backup & DR
- Snapshot world mỗi 30 phút, giữ 7 ngày. Backup full mỗi đêm lên S3/MinIO offsite.
- Test **restore drill** mỗi tháng — chưa test khôi phục thì coi như chưa có backup.
- Có **standby region** (cold standby) ở DC khác để failover khi sự cố lớn.

---

## 2. Gameplay & nội dung — "Bản sắc DonutSMP"

### 2.1. Định vị (positioning)
Tránh làm "server tổng hợp như Hypixel". Chọn **1 USP rõ ràng**, ví dụ:
- **PvP-SMP hardcore có kinh tế mở** (đặc trưng DonutSMP gốc): cướp được, kill drop head bán lấy tiền thật trong game.
- HOẶC **SMP cốt truyện + sự kiện theo mùa** (Lore-driven seasonal SMP).
- HOẶC **SMP quốc gia hoá**: bản đồ mô phỏng địa lý Việt Nam, các "tỉnh" là vùng lãnh thổ người chơi tranh giành.

→ Chọn **1**, all-in vào đó. Mọi feature khác phục vụ USP này.

### 2.2. Hệ thống cốt lõi cần có
| Hệ thống | Mục đích | Ghi chú |
|---|---|---|
| Land claim (GriefPrevention/Lands) | Chống griefing người mới | Free 1 claim nhỏ, trả phí mở rộng |
| Economy (Vault + custom) | Đồng tiền in-game ổn định | Có sink (thuế, decay) chống lạm phát |
| Auction House / Bazaar | Giao dịch an toàn | Có phí giao dịch → sink |
| Player Shops | Người chơi tự bán | Giới hạn theo rank để chống spam |
| Quest / Battle Pass theo mùa | Retention | 90 ngày/season, reward cosmetic |
| Skills / Leveling (mcMMO/AuraSkills) | Long-term progression | Cap mềm, không pay-to-win |
| Job system | Người mới có thu nhập | Cân với economy chính |
| Dungeon / Boss raid | End-game PvE | Loot table có rotate |
| Clan / Faction / Nation | Xã hội hoá | Quan trọng cho USP "server quốc gia" |
| Cosmetic (cloak, particle, pet) | Doanh thu, không P2W | **Không bán power** |

### 2.3. Cập nhật & sự kiện
- **Season-based** (3 tháng/season). Mỗi season:
  - Map mới hoặc reset 1 phần thế giới.
  - Battle Pass mới (free + premium track).
  - 1 boss/dungeon mới + 1 cơ chế gameplay mới.
- **Sự kiện hàng tuần**: PvP tournament, treasure hunt, build contest.
- **Sự kiện hàng ngày**: double XP hour, mob invasion ngẫu nhiên.
- **Sự kiện quốc gia**: lễ Tết, 30/4, 2/9, Trung thu — map trang trí, item giới hạn.

### 2.4. Cân bằng (balance)
- **Không bán power** (no pay-to-win). Doanh thu từ cosmetic, queue priority, extra claim slot.
- **Economy có sink rõ ràng**: thuế nhà đất, phí auction, repair cost, decay khi offline lâu.
- **Telemetry kinh tế**: log mọi giao dịch → dashboard Grafana theo dõi cung tiền M2, lạm phát, tỷ giá item ↔ tiền.

---

## 3. Onboarding & Retention — Giữ chân người chơi

### 3.1. Trải nghiệm 30 phút đầu là sống còn
- **Tutorial bắt buộc 5 phút** dạy: kiếm tiền đầu, claim đất, mở chat, tham gia clan.
- **New player bonus**: 1 starter kit, 1 vùng đất bảo vệ, 1 mentor ngẫu nhiên (player rank cao tự nguyện).
- **Friendly zone 24h đầu**: không bị PvP/grief ở vùng newbie.
- **Daily login reward** 7 ngày liên tục, đỉnh điểm ngày 7 reward to.

### 3.2. Hệ thống mentor & guild
- Player rank cao đăng ký làm **Mentor**, dắt newbie → reward khi mentee đạt mốc.
- Clan/Guild có **alliance system**, war system → tạo lý do quay lại mỗi ngày.

### 3.3. Anti-toxic
- **Auto-mod chat** (AI filter + danh sách từ khoá tiếng Việt).
- **Report system in-game** (`/report`) → ticket Discord tự động.
- **Restorative justice**: warn → mute ngắn → mute dài → ban; có appeal flow rõ ràng.

### 3.4. Đo lường retention
- D1, D7, D30 retention dashboard.
- Funnel: join → tutorial done → first kill/build → first transaction → first clan → D7.
- A/B test mọi thay đổi onboarding.

---

## 4. Cộng đồng & Marketing — Vũ khí tăng trưởng

### 4.1. Content creator program
- Mời **streamer/YouTuber Việt** (Minecraft, gaming) làm "Đại sứ":
  - Free rank cao nhất, kit độc quyền, NPC tên họ trong server.
  - Doanh thu chia sẻ qua creator code (mua rank dùng code → creator nhận %).
  - **Tier**: Partner (>10k sub) / Affiliate (>1k sub) / Verified (đăng ký).
- Tổ chức **giải đấu PvP** có giải thưởng tiền mặt → kéo viewership.

### 4.2. Mạng xã hội
- TikTok: clip 15–60s highlight PvP, trolling, build đẹp — **đăng 2–3 clip/ngày**.
- YouTube Shorts + video dài (event recap).
- Facebook Page + Group cộng đồng VN.
- Discord là **trục chính**: >50k members là target năm 1.

### 4.3. Đối tác chiến lược
- **Hợp tác với trường ĐH** (CLB game, CLB IT) → giải đấu nội bộ.
- **Hợp tác hãng tai nghe / gaming gear** → tài trợ sự kiện.
- **Hợp tác Mojang Partner** (chính thức): đăng ký Minecraft Partner Program nếu đủ điều kiện CCU.

### 4.4. SEO & Server List
- Đăng ký **minecraft-mp.com, minecraftservers.org, top-mc.com, namemc.com**.
- Tối ưu MOTD, banner, icon → CTR cao.
- Vote reward (vote → in-game reward) để đẩy rank trên các site này.
- Website chính thức tối ưu SEO tiếng Việt: "server minecraft việt nam", "minecraft smp việt", "donut smp".

---

## 5. Bảo mật, Anti-cheat & Pháp lý

### 5.1. Anti-cheat
- **Vulcan / Matrix / Grim** (chọn 1 chính, Grim hiện top OSS).
- Server-side check: reach, killaura, scaffold, autoclick, fly, speed.
- **Replay system** (ReplayMod server-side) để admin review report.
- Ban wave định kỳ thay vì ban ngay → khó reverse-engineer cheat.

### 5.2. Bảo mật tài khoản
- **2FA** bắt buộc cho rank VIP+ (AuthMe + TOTP plugin).
- **IP lock** tuỳ chọn cho người chơi.
- **Log đầy đủ** mọi action quan trọng (transaction, claim, OP command).

### 5.3. Pháp lý — quan trọng để được "công nhận quốc gia"
- **Đăng ký kinh doanh** pháp nhân vận hành server (công ty TNHH).
- **Tuân thủ luật VN**:
  - Luật An ninh mạng — log lưu trữ tối thiểu theo quy định.
  - Nghị định 147/2024 (game online): nếu thu phí → cần giấy phép G1 (game có sự tương tác giữa nhiều người chơi). DonutSMP có PvP & economy → **cần giấy phép G1**.
  - Bảo vệ dữ liệu cá nhân (Nghị định 13/2023): có Privacy Policy, ToS rõ ràng.
- **Phân loại độ tuổi**: gắn nhãn 12+ hoặc 16+ tuỳ mức độ PvP.
- **Thanh toán**: tích hợp **MoMo, ZaloPay, VNPay, thẻ cào** qua cổng có giấy phép trung gian thanh toán (1Pay, Appota, Payoo).
  - **Không** dùng thẻ cào trực tiếp nếu chưa có giấy phép — rủi ro pháp lý cao.

→ Đây là chìa khoá để được công nhận "server quốc gia hợp pháp", phân biệt với server lậu.

---

## 6. Kiến trúc kỹ thuật & code — Cải tiến cụ thể

### 6.1. Plugin & code structure
- **Mono-repo** cho toàn bộ plugin nội bộ, build bằng **Gradle multi-module**.
- **Shared library**: economy API, player data API, event bus — các plugin khác phụ thuộc vào đây.
- **Versioning** semver, CI/CD (GitHub Actions) build → deploy staging → prod.
- **Test**: MockBukkit cho unit test plugin; integration test trên server staging trước mỗi release.

### 6.2. Database
- **Postgres 16** primary, read replica cho analytics.
- **Connection pool** HikariCP, không mở connection mỗi query.
- **Migration**: Flyway/Liquibase, không sửa schema bằng tay.
- **Index** cho mọi query hot path (player UUID, transaction time).
- **Partitioning** bảng transaction theo tháng.

### 6.3. Caching
- **Redis** cho:
  - Session (player online status across servers).
  - Leaderboard (sorted set).
  - Rate limiting.
  - Pub/sub cho cross-server message.
- **Caffeine** cache in-process cho lookup nóng.

### 6.4. Async & non-blocking
- Mọi I/O (DB, HTTP, file) phải **async** — không bao giờ block main thread.
- Plugin custom dùng **CompletableFuture** hoặc Kotlin coroutines.
- Folia: hiểu rõ region scheduler, global scheduler, entity scheduler — viết plugin compatible.

### 6.5. CI/CD & Deploy
- **GitHub Actions**: lint → test → build → docker image → push registry.
- **Docker + Docker Compose** hoặc **Kubernetes** (nếu scale lớn) cho backend.
- **Blue-green deploy** cho hub/lobby (zero downtime). SMP world thì restart có thông báo trước 10 phút + countdown.
- **Feature flag** (Unleash/Flagsmith) để bật/tắt feature mà không cần restart.

### 6.6. Observability
- **OpenTelemetry** trace request từ proxy → backend → DB.
- **Custom metric**: số giao dịch/giây, số kill/giây, GDP server (tổng tiền in-game), Gini coefficient (chống top 1% nắm hết tiền).

---

## 7. KPI & Mục tiêu đo lường

| KPI | Hiện tại (giả định) | Mục tiêu 6 tháng | Mục tiêu 12 tháng |
|---|---|---|---|
| CCU peak | ? | 1,500 | 5,000 |
| DAU | ? | 5,000 | 20,000 |
| MAU | ? | 30,000 | 150,000 |
| D7 retention | ? | 25% | 35% |
| D30 retention | ? | 10% | 18% |
| Doanh thu/tháng | ? | 200tr VND | 1 tỷ VND |
| TPS p95 | ? | ≥ 19.5 | ≥ 19.8 |
| Uptime | ? | 99.5% | 99.9% |
| Discord members | ? | 20,000 | 80,000 |
| Server list rank VN | ? | Top 5 | #1 |

---

## 8. Roadmap 12 tháng

### Giai đoạn 1 — Ổn định (Tháng 1–2)
- [ ] Audit toàn bộ plugin, gỡ plugin rác, chuẩn hoá Paper/Folia.
- [ ] Dựng monitoring (Prometheus + Grafana + Loki).
- [ ] Setup CI/CD, staging environment.
- [ ] Anti-cheat Grim + anti-DDoS TCPShield.
- [ ] Backup automation + DR drill lần 1.
- [ ] Đăng ký pháp lý (công ty + bắt đầu hồ sơ G1).

### Giai đoạn 2 — Bùng nổ (Tháng 3–5)
- [ ] Ra mắt **Season 1** với USP rõ ràng + Battle Pass.
- [ ] Mentor program + onboarding mới.
- [ ] Creator program v1, ký 5 streamer top VN.
- [ ] Giải đấu PvP đầu tiên (giải thưởng 50–100tr).
- [ ] Tích hợp MoMo/ZaloPay qua cổng trung gian có phép.
- [ ] Mục tiêu: CCU peak 1,500.

### Giai đoạn 3 — Mở rộng (Tháng 6–9)
- [ ] Folia hoá hub & event server.
- [ ] Cross-server world sharding.
- [ ] Season 2 + boss raid mới.
- [ ] Mở rộng minigame phụ (BedWars/SkyWars phiên bản DonutSMP).
- [ ] Hợp tác trường ĐH, CLB game.
- [ ] Mục tiêu: CCU 3,000, doanh thu 500tr/tháng.

### Giai đoạn 4 — Công nhận quốc gia (Tháng 10–12)
- [ ] Hoàn tất giấy phép G1.
- [ ] Đăng ký Minecraft Partner Program (nếu đạt CCU).
- [ ] Sự kiện quốc gia lớn (Tết, 2/9) PR rộng rãi báo chí.
- [ ] Press kit + làm việc với báo (GameK, Genk, Kenh14).
- [ ] Top 1 minecraft-mp.com region SEA.
- [ ] Mục tiêu: CCU 5,000, MAU 150k, được nhắc đến như **"server Minecraft Việt Nam số 1"**.

---

## 9. Rủi ro & Phương án giảm thiểu

| Rủi ro | Mức độ | Phương án |
|---|---|---|
| DDoS lớn | Cao | TCPShield + path.net + standby region |
| Cheat lan rộng | Cao | Grim + ban wave + replay review |
| Dup item / exploit kinh tế | Rất cao | Telemetry economy realtime, freeze tài khoản nghi ngờ, audit log |
| Pháp lý (chưa có G1) | Cao | Làm hồ sơ ngay từ tháng 1, dùng cổng thanh toán có phép |
| Toxic community | Trung | Auto-mod + report + appeal |
| Mất data | Rất cao | Backup 30p + offsite + DR drill hàng tháng |
| Phụ thuộc 1 streamer | Trung | Đa dạng hoá creator pool, không để 1 người chiếm >20% traffic |
| Burn-out team | Trung | On-call rotation, không deploy thứ 6 chiều |

---

## 10. Checklist hành động ngay tuần này

1. Chạy audit plugin hiện tại, list plugin nào ngốn TPS nhất (Spark profiler).
2. Bật Spark + timings, lưu báo cáo baseline.
3. Setup Prometheus exporter (mc-prometheus-exporter) → Grafana dashboard.
4. Bật TCPShield trial, đo packet loss & latency.
5. Viết Privacy Policy + ToS sơ bộ, đăng lên website.
6. Liên hệ luật sư về hồ sơ G1 (chi phí ~30–80tr VND, thời gian 3–6 tháng).
7. Họp team chốt **USP** của DonutSMP — không quá 1 câu.
8. Lập Discord server chính thức nếu chưa có, mở kênh #feedback.

---

## 11. Triết lý vận hành

> **"Người chơi đến vì gameplay, ở lại vì cộng đồng, quay lại vì sự kiện, trả tiền vì cosmetic."**

- Không bao giờ pay-to-win.
- Không bao giờ lừa người chơi (loot box mù mờ, quảng cáo sai).
- Minh bạch patch note, minh bạch ban list, minh bạch doanh thu (nếu có thể).
- Lắng nghe community nhưng không chiều theo mọi yêu cầu — giữ vision.

Server lớn không phải vì code giỏi nhất, mà vì **vận hành kỷ luật + cộng đồng tin tưởng + nội dung sống**. Code chỉ là công cụ.

---

*Tài liệu sống — cập nhật mỗi tháng theo dữ liệu thực tế.*
