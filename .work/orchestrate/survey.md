## Intent

alice-skills'in ayrı, kardeş bir plugin'i. Felsefe aynı: önce niyet, kısa skill'ler, `.work/` kaydı, yetki sınırları. Orkestratör en üst Claude modeli; işin bir kısmını başka modellere worker olarak verir:
- bağımsız kontrol / ikinci görüş,
- Claude'un uygun gördüğü işleri doğrudan yapma.

Görevlendirmeye her seferinde Claude karar verir. Spec, kayıt, doğrulama ve "bitti" kararı Claude'da kalır.

Erişim kullanıcının mevcut abonelikleriyle, sağlayıcıların resmi CLI'ları üzerinden olur. TUI kazıma, insana yönelik çıktıyı ayrıştırma, resmi olmayan proxy ya da token taşıma yok. Worker yoksa veya kotası dolduysa Claude işi kendisi yapar.

İlk worker Codex (ChatGPT girişi). Diğerleri sonra eklenir, tasarım onlara açık kalır. Ayrı repo olacak (bu klasör); mevcut alice-skills'e dokunulmaz.

## Bottom line

1. **Codex'e bağlanmanın kararlı ve resmi yolu `codex exec`.**
   - Kullanılacak bayraklar: `--json`, `-o`, `--output-schema`, `-s read-only|workspace-write`, `--worktree`, `exec resume`. İnceleme için `codex review` / `codex exec review`.
   - OpenAI bunu script ve otomasyon yolu olarak belgeliyor ve experimental değil.
   - `codex mcp-server` kaldırıldı.
   - `app-server` resmen experimental ve sürüme bağlı.
2. **Ayrı bir MCP sunucusu veya broker yazmaya gerek yok.**
   - Claude Code'un arka plan Bash'i (`run_in_background`) bitince bildirim gönderiyor; bu iş-kuyruğu modelini bedavaya veriyor.
   - Gereken tek şey, `codex exec`'i zaman aşımı, stdin kapatma ve süreç ağacını öldürme ile başlatan küçük bir başlatıcı betik. Gerisi skill metni.
   - OpenAI'ın kendi plugin'indeki hataların çoğu (broker sızıntısı, takılan işler, oturumlar arası iş öldürme) tam da bu ek katmandan çıkıyor.
3. **OpenAI'ın resmi Claude Code plugin'i (`openai/codex-plugin-cc`) var, ama ona bağımlı olunmamalı.**
   - Neyi yapıyor: review, adversarial-review ve rescue'yu karşılıyor.
   - Neden bağımlı olunmamalı: 2026-07-08'den beri güncellenmiyor, ciddi açık hataları cevapsız, alice'in kayıt ve yetki modeli de onda yok.
   - Neyi alacağız: tasarımını ve review JSON şemasını (Apache-2.0) referans alacağız.
   - Sonuç: bu plugin'i yapmaya değer. Mevcut çözüm niyeti karşılamıyor.
4. **Ek worker'lar için asıl standart ACP (Agent Client Protocol).**
   - Copilot, Cursor, Gemini, Mistral Vibe, Qwen ve Grok resmi ACP desteği sunuyor. Codex sunmuyor; Zed'in adaptörü var ama o üçüncü taraf.
   - ACP istemcisi kurmak iş istiyor: plugin içinde bir MCP sunucusu gerekir.
   - Bu yüzden ikinci worker eklenene kadar ertelenmeli. v1'de sadece worker'ı değiştirilebilir bırakan bir dikiş yeterli.
5. **Google (Gemini CLI / Antigravity) tüketici aboneliğiyle kullanılamaz.**
   - Google, tüketici girişiyle üçüncü taraf ajan orkestrasyonuna izin vermiyor; hesap askıya alınabiliyor.
   - Google ancak API anahtarı veya Vertex ile kullanılabilir. Bu, "abonelikle" tercihine bir istisna demek.

## Options

| Yol | Artı | Eksi |
|---|---|---|
| **A. Skill'ler + `codex exec` başlatıcı betiği + arka plan Bash** (öneri) | En az parça. Resmi ve belgeli yüzey. Claude Code'un yerleşik bildirimini kullanır. | Codex'e özel. İkinci worker için bir adaptör eklemek gerekir. |
| B. Plugin içinde MCP sunucusu (iş başlat/durum/sonuç/iptal) | Worker'lar tek arayüzde toplanır, ACP'ye hazır olur. | Kendi süreç yönetimi ve hatalarıyla gelir (OpenAI plugin'inin düştüğü yer). v1 için fazla. |
| C. `codex-plugin-cc`'ye bağımlı olmak | Sıfır entegrasyon kodu. | Bakımsız, hatalı. Kayıt ve yetki modeli yok. Kontrol bizde değil. |
| D. `app-server` / Codex SDK | Kota okuma, akış, steer. | Experimental, sürüme bağlı şema. Node veya Python bağımlılığı. |

## Pitfalls

- **Takılmalar.** `codex exec` stdin açık kalınca, akış durunca ya da alt süreç pipe'ı tutunca takılabiliyor. Gerekenler: kendi duvar saati zaman aşımımız, stdin'i kapatmak, süreç grubunu öldürmek.
- **Onaylar.** Exec'te onay istemi beklenmemeli; MCP araç çağrıları otomatik iptal ediliyor. Okuma işleri `-s read-only` ile, yazma işleri `-s workspace-write` (tercihen `--worktree`) ile çalışmalı. `--dangerously-*` bayrakları kullanılmamalı. `--full-auto` artık yok.
- **Kota.** Plus'ta 5 saatlik pencere ve haftalık limit var. Kota dolunca exec'in ayrı bir çıkış kodu yok; `turn.failed`/`error` olayları ve sıfırdan farklı çıkış kodu kota dolması olarak da yorumlanmalı ve Claude'a geri düşülmeli. Her adımda otomatik inceleme kotayı hızla eritir, bu yüzden açıkça istenmedikçe olmamalı.
- **Sürüm kayması.** Son aylarda `--full-auto` ve `mcp-server` kaldırıldı. Alınacak önlemler:
  - Asgari sürüm kontrolü yapmak.
  - JSONL'de bilinmeyen olay türlerini yok saymak.
  - Mümkünse sadece `-o` ile son mesaja dayanmak.
- **Yapılandırılmış çıktı.** `--output-schema` modele göre bozulabiliyor ve `exec review` bu bayrağı yok sayıyor. Şema kısıtları basit tutulmalı.
- **Aynı ağaçta iki yazar.** Codex aynı çalışma ağacında Claude ile birlikte yazarsa çakışır. Yazma işleri ayrı worktree'de yapılmalı ve sonucu Claude birleştirmeli.
- **Windows.** workspace-write çalışmıyor ve büyük çıktı kayboluyor. v1 Linux/macOS hedeflemeli.

## Decisions this opens

1. **v1'in kapsamı.**
   - Önerim: yalnızca Codex, A yolu.
   - ACP/MCP katmanı ikinci worker eklenince gelir; o zaman ayrı bir karar olur.
2. **Google worker'ları için API anahtarı istisnası.**
   - Önerim: v1'de yok.
   - İleride Google eklenirse, sadece API anahtarı veya Vertex ile, açıkça istisna olarak.
3. **Codex'in yazdığı işin Claude'a dönüş yolu.**
   - Önerim: ayrı git worktree'de çalışsın, Claude diff'i inceleyip ana ağaca alsın.
   - Bu, projenin git deposu olmasını gerektirir. Git yoksa Codex sadece okuma ve inceleme yapar.
4. **Hangi alice skill'leri worker kullanır.**
   - Önerim:
     - `step`'in bağımsız kontrolü, varsa Codex'e gitsin.
     - `survey` okumayı bölüştürebilsin.
     - `step` uygun alt işleri devredebilsin.
     - `clarify`, `spec` ve `steer` kullanıcıyla konuşma olduğu için Claude'da kalsın.

## Sources

- Codex non-interactive mode: https://learn.chatgpt.com/docs/non-interactive-mode (developers.openai.com/codex buraya yönleniyor)
- `codex mcp-server` kaldırıldı: https://learn.chatgpt.com/docs/mcp-server
- app-server: https://learn.chatgpt.com/docs/app-server
- Fiyat/limitler ve kimlik doğrulama: https://learn.chatgpt.com/docs/pricing, https://learn.chatgpt.com/docs/auth
- OpenAI'ın Claude Code plugin'i: https://github.com/openai/codex-plugin-cc (v1.0.6, 2026-07-08; issue #793, #798, #801, #803, #826, #367)
- codex issue'ları: openai/codex #20919, #48716, #50775, #24135, #38545, #54285
- Gemini CLI ToS: https://geminicli.com/docs/resources/tos-privacy
- Antigravity üçüncü taraf kullanımı: https://discuss.ai.google.dev/t/is-using-the-official-agy-cli-through-a-local-mcp-server-with-third-party-ai-agents-permitted/184829
- Copilot CLI ACP: https://docs.github.com/en/copilot/reference/copilot-cli-reference/acp-server
- Cursor CLI: https://cursor.com/docs/cli/headless, https://cursor.com/docs/cli/acp
- ACP registry: https://zed.dev/blog/acp-registry
- PAL MCP: https://github.com/BeehiveInnovations/pal-mcp-server
- Claude Code plugin/MCP/araçlar: https://code.claude.com/docs/en/plugins-reference, https://code.claude.com/docs/en/mcp, https://code.claude.com/docs/en/tools-reference
- Yerel doğrulama: `codex-cli 0.159.2 --help` (exec, review, app-server), Claude Code 2.1.325
