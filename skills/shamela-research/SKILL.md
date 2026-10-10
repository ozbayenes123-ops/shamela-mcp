---
name: shamela-research
description: "Use when doing source-grounded Islamic research over a local Shamela install through the shamela-mcp tools — finding books, reading pages, searching, verifying quotations, and formatting citations with book_id + page_id."
version: 1.0.0
---

# Shamela Araştırma İş Akışı

Bu beceri, `shamela-mcp` sunucusunun yerel **Mektebetü'ş-Şâmile 4** kurulumu üzerinde
sağladığı MCP araçlarıyla **kaynağa dayalı** bir araştırmanın pratik akışını anlatır.
Tüm iddialar bu depodaki gerçek araç tanımlarına (`src/server/tools/*.ts` içindeki zod
giriş şemaları) dayanır — uydurma argüman yoktur.

Sunucu salt-okunurdur: hiçbir araç Şâmile veritabanını yazmaz. Aramalar **yalnızca
indirilmiş kitaplarda** çalışır; bu yüzden önce neyin gerçekten indirilmiş olduğunu bil.

## 0. En kritik kural: `page_id` ≠ basılı sayfa numarası

`shamela_get_page` ve `shamela_get_pages_range` **dahili `page_id`** alır — bu Şâmile'nin
Lucene/SQLite'taki kendi ardışık sayacıdır, kitabın bastığı sayfa numarası değildir.
Doğrulama (`src/server/tools/getPage.ts:19`):

> `page_id: z.number().int().positive().describe("The page id (Lucene/SQLite internal id, not the printed page number).")`

`getPagesRange.ts:17` de aynı şekilde `start_page_id` için "First page_id (inclusive)"
der. Yani elle taşınan bir "ص ١٧" (basılı 17) değerini `page_id` olarak verme — bu,
nadiren denk gelen yanlış bir sayfayı okur.

**Doğru `page_id` keşif yolu:**

1. `shamela_search_pages` / `shamela_search_phrase` / `shamela_search_exact` /
   `shamela_search_boolean` sonuçları her isabet için `book_id` + `page_id` **ve**
   `printed_page` etiketini birlikte döndürür → `page_id`'yi buradan al.
2. Ya da `shamela_get_toc(book_id, parent_id, depth)` ile bölüm ağacını gez,
   `title_id`'yi al, `shamela_get_book_section(book_id, title_id)` ile o bölümün
   tamamını oku (araç başlığın başlangıç/bitiş sayfasını per-book SQLite'tan çözer).
3. `shamela_verify_quote`, kredilendirilen sayfada alıntıyı bulamazsa **basılı
   numarası** verilen sayfayı da kontrol eder ve `printed_page_confusion` olarak
   raporlar — elle taşınan atıflardaki en yaygın hatayı bu alan yakalar.

`shamela_get_page` çıktısı da `printed_page` etiketini ve `prev_page_id`/`next_page_id`
komşularını döndürür; basılı numarayı göstermek için onu kullan, aramada değil.

## 1. Kitap bulma

| Araç | Giriş (gerçek şema) | Not |
|---|---|---|
| `shamela_search_books` | `query` (string, min 1); `scope` (author_ids, category_ids, period_from, period_to, downloaded_only); `options`; `limit`/`offset` | ~8.500 kitabın kataloğunu arar. **`scope.book_ids` kabul etmez** — katalog zaten evrendir. İndirme olmadan da çalışır. |
| `shamela_resolve` | `query` (min 1), `type` (`"any"\|"book"\|"author"`, varsayılan `"any"`), `limit` (1–20, varsayılan 5) | Bir ismi kitap/müellif `id`'sine çözer, güven skoru döndürür. Latin harfli yazım da kabul edilir ("Ibn Qudama"); o durumda yanıt `transliterated:true` taşır — bunlar **aday**tır, indeks isabeti değil. |
| `shamela_list_downloaded_books` | `category_id` (opsiyonel), `limit`/`offset` | **Bu makinede gerçekten indirilmiş** kitapları listeler. `content_status`: `readable` vs `downloaded_no_pages`. Araştırma kapsamını dürüstçe çizmek için şart. |

Arama öncesi doğru `book_id`/`author_id` için önce `shamela_resolve`, indirilmişleri
görmek için `shamela_list_downloaded_books` kullan.

## 2. Sayfa / aralık okuma

- `shamela_get_page(book_id, page_id)` — tek sayfa. `keep_html` (varsayılan false),
  `around_phrase` + `around_phrase_window` (40–2000, varsayılan 300) ile belirli bir
  ibarenin çevresini okur. Uzun sayfa ~4000 karakterlik parçalara bölünür → `body_part`.
- `shamela_get_pages_range(book_id, start_page_id, count)` — `count` 1–20 (varsayılan 5),
  `keep_html`. Uzun aralık karakter bütçesiyle kısalır → `next_start_page_id`.
- `shamela_get_book_section(book_id, title_id)` — bir başlığın altındaki tüm sayfalar
  (`max_pages` varsayılan 30, en çok 100; `truncated` bayrağı).

Her ikisi de **`book_id` + dahili `page_id`** ister (bkz. §0).

## 3. Arama

| Araç | Öne çıkan giriş | Semantik |
|---|---|---|
| `shamela_search_pages` | `query`; `scope`; `options{morphology, wildcards, search_in:["body"\|"foot"\|"comment"], preserve_*}`; `limit`/`offset` | Gövde + haşiye (varsayılan). Çok kelime AND'lenir. `preserve_*` şu an `OPTION_NOT_SUPPORTED`. `morphology` (AlKhalil kök genişletme) ile `wildcards` (`*`/`?`) **birlikte kullanılamaz**. |
| `shamela_search_phrase` | `query`; `mode` (`"phrase"`→bitişik, `"near"`→yakın, varsayılan `"phrase"`); `distance` (1–50, varsayılan 5); `search_in` (varsayılan `["body","foot"]`); `scope` | `phrase`: kelimeler **bitişik**. `near`: kelimeler herhangi sırada `distance` kelime içinde. |
| `shamela_search_exact` | `query`; `preserve{preserve_diacritics, preserve_hamza, preserve_digits}`; `scope` | **`preserve` içinde en az biri `true` olmalı** (şema `refine` ile zorlar), yoksa normal aramayı kullan. Sorguyu **korunacak haliyle** yaz (tashih otomatik yapılmaz). `preserve_hamza` → «أحمد» ile «احمد» eşleşmez. |
| `shamela_search_boolean` | `all_of[]` (VE), `any_of[]` (VEYA), `none_of[]` (DEĞİL), `search_in`; `scope` | `all_of`/`any_of`'dan **en az biri gerekli**. Sonuç = `(∩ all_of) ∩ (∪ any_of)` − `(∪ none_of)`. |

`options.search_in:["foot"]` yalnız bırakmak, tahkik edenin haşiyesine (basılı tahric ve
kaynak atıflarının yeri) odaklanır ve metni sonuç dışı tutar.

## 4. Alıntı doğrulama — `shamela_verify_quote`

Giriş: `quote` (string, min 4 — **yazdığın gibi**), `book_id` (opsiyonel),
`page_id` (opsiyonel, `book_id` gerektirir), `scope` (yalnızca kitapsız kütüphane
taramasında), `limit` (1–20, varsayılan 5).

Beş hüküm döndürür:

- `verbatim` — yazıldığı gibi (tashih, hemze, rakam sistemleri dahil) mevcut.
- `differs` — tamamı var ama farklı eksenler adlandırılmış.
- `partial` — bir kısmı var, gerisi başka kelimelerle (ezberden aktarılan alıntının izi).
- `not_found` — bu **makinedeki indirilmiş kitaplarda** yok (gelenek hakkında hüküm değil).
- `unverifiable` — kredilendirilen kitap indirilmemiş; hiçbir şey incelenmedi.

**Karşılaştırma "as-typed"dır** (birebir): hiçbir tashih/hemze/rakam normalizasyonu
otomatik yapılmaz. Bu yüzden alıntıyı **önce ham haliyle doğrula**; ancak doğrulama
başarısız olursa ve yalnızca tashih/hemze farkından şüpheleniyorsan normalleştirip
tekrar dene — asla önce normalleştirip sonucu "birebir" sayma. Gövde (`body`) yazarın
metni, haşiye (`foot`) muhakkikin dipnotudur: haşiyeden alınıp yazara nispet edilen bir
alıntı, birebir eşleşse bile **yanlış nispettir**.

## 5. Atıf biçimlendirme — `shamela_get_citation`

Giriş: `book_id` (zorunlu), `page_id` (opsiyonel — verilmezse kitap düzeyinde atıf),
`text` (opsiyonel — verilirse `shamela` stilinde «kitap» (cüz/sayfa): «metin» bloğu
üretilir), `style` (`"shamela"` varsayılan | `"short"` | `"full"`).

- `shamela`: Şâmile'nin "atıfla kopyala" biçiminin aynısı — yayına hazır.
- `short`: tek satır — <müellif>، <kitap>، ص <sayfa>.
- `full`: müellif vefat yılıyla uzun biçim; `notes[]` ile eksik künye alanlarını
  (baskı/yayıncı/şehir/muhakkik) listeler — master.db'de yoktur, **uydurulmaz**.
- Tüm sayılar Arap-Hint rakamlarıyla çıkar. `book_date` yayım/baskı yılı değil,
  Şâmile'nin tarih damgasıdır; atıfta basılmaz.

## 6. Delil güvenilirliği (nispet ve ihtilaf)

- **Nispet (provenance):** Her metin alıntısını araç sonucundan al; haşiye ile metni
  ayır; sayfa numaralandırmasının durumunu ve eksik veriyi açıkça söyle (bkz. §0, §5).
  Master.db'de olmayan künye alanlarını (baskı/yayıncı/yıl) asla uydurma.
- **`shamela_scan_consensus`:** `question` (min 2), `families` (`"ijmaa"`|`"khilaf"`),
  `formulas[]` (sözlükteki haliyle, ör. «لا خلاف», «روايتان»), `distance` (1–50,
  varsayılan 15), `search_in` (varsayılan `["body"]` — haşiye opt-in: muhakkikin
  "icmâ" beyanı yazarınki değildir), `witnesses` (0–5, varsayılan 2), `scope`
  (`madhhab` kabul eder). **Hüküm alanı yoktur** ve iki kolonun toplamı yoktur:
  indeks olumsuzlamayı/nispeti göremez, «لا إجماع» de «ادعى الإجماع» de kalıbı taşır.
  Şahitleri oku; sayılar yalnızca nereye bakılacağını söyler. Bir meseleyi tartışmadan
  **önce** onun tartışmalı olup olmadığını görmek için kullan.
- **`shamela_research_scope(term, synonyms?)`:** her mezhep için `found`/`silent`/
  `cannot_tell` ayrımı döndürür. `silent` (mezhebin kitapları var, hiçbiri demiyor)
  ile `cannot_tell` (hiç indirilmemiş) zıt sonuçlardır ama sıradan arama sonucunda
  aynı görünür. Bir mezhebi **yalnızca `silent` diyen satırdan** "suskun" ilan et.

## Doğrulama kuralları

1. **`book_id` + `page_id` + doğrulanmış alıntı olmadan** hiçbir hadis veya fıkhî nispet
   sunma. `shamela_verify_quote` sonucu `verbatim`/`differs`/`partial` değilse (özellikle
   `not_found`/`unverifiable`), iddiayı nispet etmiş sayma.
2. Alıntı doğrulaması **birebir (exact-match)**tir: **önce ham haliyle** dene;
   normalizasyon (tashih/hemze/rakam) yalnızca doğrulama sonrası, farkın kaynağını
   teşhis etmek için ve açıkça belirtilerek yapılır. Normalize edilmiş metni "birebir
   doğrulandı" diye sunma.
3. `not_found` **bu makinedeki indirilmiş kitaplar** hakkındadır, gelenek hakkında
   değil — yokluğu delil sayma.
4. Metin (matn) ile haşiye (foot) ayrımını koru; haşiyeden alınan bir söz yazara
   nispet edilemez.
5. Künye alanlarını (baskı/yayıncı/şehir/muhakkik) uydurma; eksikse `notes[]`'i
   aktar.
6. Kapsamı dürüst bildir: aramalar yalnızca **indirilmiş** kitapları kapsar;
   mezhep karşılaştırmasında bir mezhebi yalnızca `research_scope` satırı `silent`
   diyorsa "görüşsüz" ilan et.

## Tipik akış (özet)

1. `shamela_list_downloaded_books` — indirilmiş kütüphaneyi gör.
2. `shamela_resolve` / `shamela_search_books` — kitabı ve `book_id`'yi bul.
3. `shamela_search_pages` / `_phrase` / `_exact` / `_boolean` — konumu bul, isabetin
   `page_id`'sini al (elle basılı numara taşıma).
4. `shamela_get_page(book_id, page_id)` veya `shamela_get_book_section` — oku.
5. `shamela_verify_quote` — nispeti birebir doğrula.
6. `shamela_get_citation` — `book_id` (+ `page_id`, + `text`) ile yayına hazır atıf üret.
