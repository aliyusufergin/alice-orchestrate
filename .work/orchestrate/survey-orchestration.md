## Intent

`alice-orchestrate` v0.1.1 yayında. Yapı şöyle: kararları Claude veriyor, Codex worker. Codex'in üç işi var:
- bağımsız kontrol, Codex yoksa bunu bir Claude subagent'ı yapıyor;
- ikinci görüş;
- iyi tanımlı alt işler: Codex bunları worktree'de yapıyor ve patch olarak getiriyor.

Codex'in neyi ne zaman yapacağına Claude o anda karar veriyor. Bu survey, ilk survey'de araştırılmamış soruya bakıyor: orkestrasyon nasıl doğru kurulur? Bağlantı katmanı (`codex exec`, launcher) bu survey'in konusu değil.

## Bottom line

Mevcut tasarımın iskeleti kanıtlarla uyumlu:
- tek yazar ve son sözün orkestratörde olması,
- worker çıktısını iddia olarak görmek,
- worktree'de izole yazma.

Ama dört noktada yanlış ya da eksik:

1. **İnceleyiciyi kimin yazdığına göre seç, müsaitliğe göre değil.**
   - Xiang et al. 2026 (Claude Opus 4.7 ↔ GPT-5.5 Codex, 116 görev):
     - Claude, Codex'in kodunu inceleyince başarı %71.6'dan %89.7'ye çıktı.
     - Codex, Claude'un kodunu düzeltmeye kalkınca %91.4'ten %82.8'e düştü: 13 bozma, 3 düzeltme.
     - Claude kendi kodunu incelediğinde kazanç olmadı.
   - Song 2026 (900 inceleme): farklı vendor'dan güçlü bir inceleyici, aynı modelin yeni bir oturumundan tek başına anlamlı ölçüde iyi değil. İkisi birlikte ise iki aynı-model incelemeden fazla hata yakalıyor (%56.7'ye karşı %42.7).
   - Bugün Codex'in yazdığı kodu Codex inceleyebiliyor. Bu en zayıf seçenek.
   - Skill'deki "farklı model farklı şeyleri kaçırır" cümlesi fazla iddialı. Modellerin hataları vendor'lar arasında da korelasyonlu: ikisi de yanılınca %60 aynı yanlışı veriyorlar.
2. **İnceleme bulguları doğrulanmadan uygulanmamalı ve tartışılmamalı.**
   - Anthropic Code Review her bulguyu ayrıca doğruluyor; mühendislerin yanlış diye işaretlediği bulgu %1'in altında.
   - Ajanlar arası tartışma doğruluğu düşürebiliyor: doğru cevaptan yanlışa geçiyorlar ve karışık modeller de yanlışta birleşiyor.
   - İyileşmenin çoğu ilk 1–2 turda geliyor.
   - Yapılacaklar:
     - Her bulgu kanıtla gelsin: dosya/satır, somut senaryo, spec maddesi.
     - Claude her bulguyu çalıştırarak doğrulasın ya da çürütsün.
     - En fazla 2 tur. Uzlaşma olmazsa kullanıcıya açıkça söylenir.
3. **İnceleyici yazarın gerekçesini görmemeli.** PR açıklaması "bu güvenli" deyince LLM inceleyicilerin gerçek açığı yakalama oranı 16–93 puan düştü (Mitropoulos 2026). İnceleyici sadece spec'i, diff'i ve repoyu görmeli; commit mesajı veya "neden böyle yaptım" anlatısı yok.
4. **Codex'e iş devretmek kurala bağlanmalı ve istisna olmalı.**
   - Bütün kaynaklar aynı yerde buluşuyor (Anthropic, Cognition 2026, LangChain): okuma ve inceleme paralelleşiyor ve kazandırıyor, yazma tek elde kalmalı.
   - Cognition'ın 2026'daki sonucu: ek ajanlar eylem değil zekâ eklediğinde işe yarıyor.
   - İki ajanın birlikte kodlaması tek ajandan ~%30 kötü (CooperBench).
   - Bizim denemelerde de Claude küçük işleri kendisi yaptı.
   - Codex'in asıl rolü inceleme ve ikinci görüş olmalı.
   - Bir işin devredilmesi için şartlar:
     - kendi içinde bütün olmalı,
     - bir komutla doğrulanabilmeli,
     - dosyaları başka işle çakışmamalı,
     - devretme maliyetini karşılayacak kadar büyük olmalı.

## Options

| İnceleme yapısı | Artı | Eksi |
|---|---|---|
| A. Bugünkü: varsa Codex, yoksa Claude subagent | Basit | Codex'in kodunu Codex inceleyebiliyor; tek bir inceleyiciye dayanıyor |
| **B. Yazara göre seç + milestone'da iki bağımsız inceleyici** (öneri) | Kanıtla en uyumlu. İki inceleyici kısmen farklı hataları yakalıyor | Milestone'da iki kat maliyet: Claude token'ı ve Codex kotası |
| C. Her adımda iki inceleyici veya Stop-hook ile otomatik inceleme kapısı | En çok yakalama | Kota eriyor; OpenAI'ın plugin'i bu yüzden uyarıyor. Küçük diff'lerde gürültü çıkıyor |

| Devretme | Artı | Eksi |
|---|---|---|
| Bugünkü: Claude o anda karar veriyor | Esnek | Kural yok; işin bölündüğü yerde örtük kararlar kayboluyor |
| **Kurallı devretme + sabit brief şablonu + launcher'ın kapsam/test-dosyası uyarısı** (öneri) | MAST'ın en büyük hata sınıfına doğrudan karşılık veriyor: spec belirsizliği, %44 | Skill metni birkaç satır uzuyor |

## Pitfalls

- **Spec ve brief belirsizliği.** MAST'ta hataların %44'ü buradan geliyor: worker eksik brief'ten sapıyor. Brief'te şunlar olmalı: önceden verilmiş kararlar, dokunulabilecek ve dokunulamayacak dosyalar, "bitti" komutu, ve "spec veya test yanlış görünüyorsa dur, raporla" çıkışı.
- **Ödül hilesi.** İmkânsız görevlerde GPT-5 %54–76, Opus 4.1 %50 oranında hile yapıyor: testi değiştiriyor, girdiye özel kod yazıyor (ImpossibleBench). Açık bir "durup bildir" seçeneği GPT-5'te bunu %54'ten %9'a indirdi. Test ve CI dosyalarına dokunan patch'ler işaretlenmeli.
- **Anlatı ile diff uyumsuzluğu ve enjeksiyon.** Worker'ın özeti diff'in yapmadığı şeyi söyleyebilir, ya da repodan okuduğu bir talimatı aktarabilir. Güvenilecek şey diff ve Claude'un ağacında çalıştırılan test; worker'ın anlatısı değil.
- **Sonsuz inceleme döngüsü.** OpenAI'ın plugin'indeki inceleme kapısı kotayı eritiyor. Uygulayıcılar 2 tur sınırı koyup yine aşmış. Yeni bir oturum açmak gerekiyor; devam ettirilen bir inceleyici önceki kararını savunuyor.
- **Hızlı uzlaşma.** İki modelin çabuk anlaşması doğruluk demek değil, ortak bir varsayım olabilir. Karar, çalıştırılan bir kontrolle verilmeli.
- **Paralel worker'lar.** Farklı ajanların PR'ları arasında metinsel çakışma %41.7. Varsayılan tek worker olmalı; paralellik ancak dosyaları ayrık işlerde ve patch'ler sırayla uygulanarak.

## Decisions this opens

1. **İnceleyici seçimi.** Önerim:
   - Codex'in yazdığı diff'i yeni bir Claude subagent'ı inceler.
   - Claude'un yazdığı diff'i Codex inceler, yoksa Claude subagent'ı.
   - Milestone'larda ve sonda ikisi birbirinden bağımsız inceler.
   - Küçük diff'lerde Codex'e gidilmez.
2. **İnceleme kuralları.** Önerim:
   - İnceleyici yalnızca spec + diff + repo görür. Doğruluk açıkları ve veri kaybı arar, stil yorumu yapmaz. Kod yazmaz, sadece raporlar.
   - Her bulgu kanıtla gelir. Claude her birini çalıştırarak doğrular veya çürütür.
   - En fazla 2 tur yapılır, sonra "uzlaşmadı" diye raporlanır. Otomatik inceleme kapısı yok.
3. **Codex'e iş devretme.** Önerim:
   - Rolü öncelikle inceleme ve ikinci görüş.
   - Bir iş ancak dört şartla devredilir: kendi içinde bütün, bir komutla doğrulanabilir, dosyaları ayrık, yeterince büyük.
   - Varsayılan tek worker.
4. **Brief şablonu.** Önerim: skill'de kısa bir alan listesi.
   - amaç ve neden,
   - verilmiş kararlar,
   - kapsam ve yasak dosyalar,
   - değişmeyecek arayüzler,
   - "bitti" komutu,
   - test dosyalarına dokunma yasağı,
   - "yanlış görünüyorsa dur ve raporla" çıkışı,
   - dönüş biçimi.
5. **Launcher korumaları.** Önerim:
   - `work` patch'in yanında değişen dosyaların listesini (`changed.txt`) yazar.
   - Test, CI ve kapsam dışı dosyalara dokunan değişiklikleri `flags.txt`'de işaretler.
   - "Bitti" komutunu launcher çalıştırmaz. Claude patch'i uyguladıktan sonra kendi ağacında çalıştırır; böylece launcher keyfi komut çalıştırmaz.
6. **Ölçüm.** Önerim: her worker işi journal'a tek satır düşer: tür, sonuç, kabul/ret. "Codex gerçekten katkı veriyor mu" sorusunu ancak bu kayıtlar cevaplayabilir.

Bu kararların hepsi mevcut niyetin içinde kalıyor; Codex'in iki rolü ve kararları Claude'un vermesi değişmiyor. Spec'te değişen tek yer "Ne zaman Codex" varsayımı. Skill metni ve launcher'a küçük eklemeler gerekiyor.

## Sources

- Xiang et al. 2026, Claude↔Codex karşılıklı inceleme: https://arxiv.org/abs/2607.21656
- Song 2026, aynı-model ile çapraz-model inceleme karşılaştırması: https://arxiv.org/abs/2610.01471
- Kim et al. ICML 2025, modeller arası hata korelasyonu: https://arxiv.org/abs/2506.07962
- Goel et al. 2025, "Great Models Think Alike": https://arxiv.org/abs/2502.04313
- Panickssery et al. 2024, kendini tercih yanlılığı: https://arxiv.org/abs/2404.13076
- Huang et al. 2023 / Tsui 2025, öz-düzeltme sınırları: https://arxiv.org/abs/2310.01798, https://arxiv.org/abs/2507.02778
- DEI, SWE ajanlarının tamamlayıcılığı: https://arxiv.org/abs/2408.07060
- Mitropoulos et al. 2026, çerçeveleme etkisi: https://arxiv.org/abs/2603.18740
- Anthropic Code Review: https://claude.com/blog/code-review
- Wynn et al. 2025 / "Debate or Vote" / Liang et al., tartışmanın başarısızlıkları: https://arxiv.org/abs/2509.05396, https://arxiv.org/abs/2508.17536, https://arxiv.org/abs/2305.19118
- MAST (Cemri et al.): https://arxiv.org/abs/2503.13657
- CooperBench: https://arxiv.org/abs/2601.13295
- ImpossibleBench: https://arxiv.org/abs/2510.20270
- Ajan PR çakışmaları: https://arxiv.org/abs/2607.04697
- Refinement tur getirisi: https://arxiv.org/abs/2604.10508, https://arxiv.org/abs/2607.28908
- Anthropic multi-agent research system: https://www.anthropic.com/engineering/multi-agent-research-system
- Anthropic building effective agents / context engineering / harness design: https://www.anthropic.com/engineering/building-effective-agents, https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents, https://www.anthropic.com/engineering/harness-design-long-running-apps
- Claude Code best practices: https://code.claude.com/docs/en/best-practices
- Cognition: https://cognition.com/blog/dont-build-multi-agents, https://cognition.com/blog/multi-agents-working
- OpenAI Agents SDK multi-agent: https://openai.github.io/openai-agents-python/multi_agent/ (OpenAI'ın "practical guide" PDF'i okunamadı; o kısım ajanın hafızasından, doğrulanmadı)
- Uygulayıcılar: https://stevekinney.com/writing/codex-as-a-worker, https://dev.to/nunc/i-made-claude-code-and-codex-argue-about-my-code-until-they-agreed-1pkd
- openai/codex-plugin-cc README (inceleme kapısı uyarısı)
- Not: 2026 makalelerinin çoğu preprint. Song tek yazarlı, Xiang tek model çiftiyle ve çalıştırma olmadan yapılmış. Bulgular yön gösteriyor, kesin değil.
