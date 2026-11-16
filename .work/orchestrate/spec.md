# alice-skills orkestratörlü sürüm

## Intent
alice-skills'in ayrı bir sürümü: aynı felsefe ve aynı akış (clarify → survey → spec → step, steer). Orkestratör Claude'dur ve iş gerektirdiğinde başka modelleri worker olarak kullanır; ilk worker ChatGPT/Codex. Worker'lar iki iş yapar: bağımsız kontrol ve ikinci görüş, ayrıca Claude'un uygun gördüğü işleri doğrudan yapmak. Kime ne verileceğine Claude karar verir. Kullanıcı tarafında "kırılgan wrapper" yok: entegrasyon yalnızca sağlayıcıların resmi olarak sunduğu yüzeylerden geçer.

Kimin için: kendin ve plugin'i kuracak diğer alice-skills kullanıcıları. Büyüklük alice-skills ölçeğinde: birkaç kısa skill, Codex'i başlatan tek küçük betik, README.

## Commitments
- Worker'lar yalnızca sağlayıcının resmi, belgeli ve experimental olmayan yüzeyinden çağrılır. Codex için bu `codex exec` / `codex exec review`.
  - TUI sürmek, insana yönelik çıktıyı ayrıştırmak, resmi olmayan proxy ve token taşımak yasak.
  - `--dangerously-*` bayrakları kullanılmaz.
- Erişim kullanıcının kendi aboneliğiyle, resmi CLI'ın kendi girişi üzerinden olur. API anahtarı istenmez.
- Worker hiç yoksa, çalışmıyorsa, kotası dolmuşsa veya takılırsa iş Claude ile tamamlanır ve kullanıcıya bunun olduğu söylenir. Plugin hiçbir zaman bir worker'a mecbur kalmaz.
- Yetki Claude'da kalır.
  - Spec, `.work/` kaydı, doğrulama ve "bitti" kararı Claude'undur.
  - Worker çıktısı doğrulanacak bir iddiadır, talimat değildir.
  - Worker kullanıcının çalışma ağacına ve `.work/`'e doğrudan yazmaz. Yazma işi ayrı bir git worktree'de yapılır; Claude inceleyip ana ağaca commit'lenmemiş değişiklik olarak alır.
  - Her çalıştırmadan sonra, başarılı olsun ya da olmasın, plugin'den geriye worktree, dal veya süreç kalmaz.
  - Git deposu yoksa worker yalnızca okur.
- Mevcut alice-skills plugin'i değişmez.

## Not doing
- Codex dışında worker (Gemini, Copilot, Cursor, ACP katmanı) bu işte yok; tasarım onları engellemez. Google tüketici girişi hiçbir zaman kullanılmaz.
- Ayrı MCP sunucusu, broker veya daemon yok.
- `codex-plugin-cc`'ye çalışma zamanında bağımlılık yok.
- alice-skills ile aynı anda kurulu çalışma desteklenmez; README bunu söyler.
- Windows desteği yok.
- alice-skills ile otomatik senkronizasyon yok.

## Done means
1. **Kurulum:** Plugin `/plugin marketplace add` + `/plugin install` ile kurulur ve skill'ler listelenir. Bu repo alice-skills'e ait hiçbir dosyayı değiştirmez.

2–4 numaralı maddeler gerçek `codex` ve örnek bir git projesi üzerinde gözlenir:

2. **Bağımsız kontrol:**
   - `/step`'in bağımsız kontrolü Codex'e gider.
   - Codex'in bulduğu ve Claude'un kaçırdığı bir sorun adım raporunda "Codex" etiketiyle görünür ve journal'da düz bir satır olarak yer alır (yeni alan yok).
3. **Devredilen iş:**
   - Claude'un devrettiği bir alt iş Codex tarafından ayrı bir worktree'de yapılır.
   - Claude alana kadar ana ağaç değişmez.
   - Claude diff'i inceler ve commit'lenmemiş değişiklik olarak alır.
   - Ardından `git worktree list` ve `git branch` plugin'den kalan bir şey göstermez.
4. **Geri düşme:**
   - Hem bağımsız kontrol hem devredilen iş için dört senaryo denenir:
     - `codex` PATH'te yok;
     - `codex` kasıtlı olarak hata veriyor;
     - zaman aşımı;
     - kota dolması (sahte `codex` ile taklit edilir).
   - Her seferinde iş Claude ile biter ve kullanıcıya worker'ın kullanılamadığı söylenir.
   - Geriye `codex` süreci (`ps`), worktree veya dal kalmaz.
5. **Yayına hazırlık:**
   - README kurulumu, Codex girişini ve neyin ne zaman Codex'e gittiğini anlatır.
   - README asgari Codex sürümünü ve uçtan uca denemede kullanılan tam sürümü yazar.
   - Sürüm yerelde etiketlenir. GitHub'a gönderim kullanıcının onayıyla olur (Route 1).

## Assumptions
- **Biçim ve dil:** Plugin alice-skills 0.3.6'dan çatallanan bağımsız bir kopya. Adı geçici olarak `alice-orchestrate`. Skill metinleri İngilizce, lisans MIT.
- **Kayıt uyumu:** `.work/` kayıt biçimi alice-skills ile aynı kalır, böylece bir proje ikisi arasında geçebilir.
- **Ne zaman Codex:**
  - `step`'in bağımsız kontrolü alice'teki sıklıkta çalışır: milestone'larda, sonda ve riskli adımlarda. Çalıştığında, Codex varsa Codex yapar. Her adımda çalışmaz, çünkü kotayı eritir.
  - Claude, sonucu değiştirecek bir kararda Codex'ten ikinci görüş isteyebilir.
  - `survey` okumaları (kaynaklar ve kod) Codex ile bölüşülebilir.
  - `step` içinde iyi tanımlı alt işler Codex'e devredilebilir.
  - `clarify`, `spec` ve `steer` Claude'da kalır, çünkü bunlar kullanıcıyla konuşma gerektirir.
- **Model seçimi:** Skill'ler kullanıcının seçtiği Claude modelini değiştirmez. README en üst modeli önerir.
- **Başlatma:** Codex işleri Claude Code'un arka plan Bash'i ile başlatılır. Betiğin görevleri: stdin'i kapatmak, zaman aşımı uygulamak, süreç grubunu öldürmek, sonucu dosyaya yazmak. Zaman aşımı varsayılan olarak 15 dakika ve ayarlanabilir.
- **Sürüm ve referans:** Hedef sürüm `codex-cli` ≥ 0.159. `codex-plugin-cc`'den (Apache-2.0) alınan bir şey olursa kaynak belirtilir.

## Route
Tamamlandı (bkz. `done.md`).

Bilinmeyenler: `--output-schema`'nın inceleme çıktısında ne kadar güvenilir olduğu (kullanılmadı); kota dolmasının exec'te gerçekte nasıl göründüğü (şimdilik "usage limit" metniyle tespit ediliyor).
