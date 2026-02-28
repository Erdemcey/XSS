# DOM XSS in jQuery selector sink using a hashchange event

## Lab Açıklaması
Bu laboratuvarda jQuery selector sink'inin `hashchange` olayı üzerinden nasıl DOM tabanlı XSS zafiyeti oluşturduğunu inceliyoruz. Saldırı, hash (#) parametresindeki JavaScript kodunun doğrudan jQuery selector'ına geçirilmesiyle gerçekleşir.

---

## Analiz

### Uygulama Davranışı
- Sitede kullanıcı profiline tıklandığında, ilgili kullanıcının sayfasına yönlendirilir.
- URL'nin hash bölümü (#) kullanıcı bilgisini taşır.
- `hashchange` olayı tetiklendiğinde JavaScript çalıştırılır.

### Zafiyet Mekanizması
- Hash parametresi jQuery selector'ına doğrudan geçiriliyor.
- jQuery selector HTML olarak işlenebilen bir sink'tir.
- Validasyon veya sanitasyon yapılmadan hash değeri kullanılıyor.

---

## Saldırı Vektörü

### Saldırı Stratejisi
1. Saldırgan kendi sunucusuna kontrol ettiği bir sayfa yüklüyor.
2. Hedef siteyi bir `iframe` içinde açıyor.
3. `onload` olayını kullanarak URL'ye kötü amaçlı payload ekliyor.
4. Ziyaretçi bu sayfaya gittiğinde payload otomatik olarak çalışıyor.

### Exploitation Payload

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
```

#### Payload Açıklaması
- `iframe` ile hedef siteyi yüklüyoruz.
- `onload` olayında iframe'in src'sine payload ekliyoruz.
- `<img src=x onerror=print()>` - geçersiz kaynakla resim yüklemesi başarısız olur.
- `onerror` olayı tetiklenir ve `print()` (veya `alert()`) çalışır.
- Hash'e eklenen bu payload jQuery selector tarafından işleniyor.

### Saldırı Adımları
1. Saldırgan kendi sunucusunda (Example Server) bu iframe kodunu body'de yer alacak şekilde yerleştiriyor.
2. Ziyaretçi saldırganın sunucusunu ziyaret ediyor.
3. İframe otomatik olarak hedef siteyi açıyor ve payload ekliyor.
4. Hedef sitenin jQuery kodu hash'teki payload'ı işleniyor.
5. JavaScript çalışıyor ve pop-up veya print dialog açılıyor.

---

## Teknik Detaylar

### Hedef Site Kodu (Tahmini)
```javascript
$(function() {
    $(window).on('hashchange', function() {
        var hash = window.location.hash.slice(1);
        $('#user-profile').html($(hash));
    });
});
```

### Payload Çalışması
1. Normal: `https://lab.com/#user-123`
2. Saldırı: `https://lab.com/#<img src=x onerror=print()>`
3. jQuery selector bu HTML'i işliyor ve `onerror` olayı tetikleniyor.

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Payload başarıyla çalışır ve **print dialog** (veya `alert()` popup) görüntülenir. Bu, jQuery selector sink'inin hash parametresi üzerinden DOM XSS zafiyetine maruz olduğunu gösterir.

---

## Zafiyet Özeti

| Unsur | Değer |
|-------|-------|
| **Kaynak (Source)** | `window.location.hash` |
| **Sink** | jQuery `$(hash)` selector |
| **Zafiyetin Türü** | DOM-based XSS (hashchange event) |
| **Payload Tetikleyicisi** | `hashchange` event, `onload` olayı |
| **Saldırı Türü** | Stored/Reflected XSS (iframe ile delivery) |

---

## Önleme Önerileri

### 1. jQuery Selector Yerine textContent Kullanma
```javascript
$(window).on('hashchange', function() {
    var hash = window.location.hash.slice(1);
    document.getElementById('user-profile').textContent = hash;
});
```

### 2. Giriş Validasyonu
```javascript
$(window).on('hashchange', function() {
    var hash = window.location.hash.slice(1);
    
    // Sadece sayısal user ID'lere izin ver
    if (/^\d+$/.test(hash)) {
        // Güvenli işlem yap
        loadUserProfile(hash);
    }
});
```

### 3. InnerHTML Yerine createElement Kullanma
```javascript
$(window).on('hashchange', function() {
    var hash = window.location.hash.slice(1);
    var elem = document.createElement('div');
    elem.textContent = hash;
    document.getElementById('user-profile').appendChild(elem);
});
```

### 4. DOMPurify Kütüphanesi Kullanma
```javascript
$(window).on('hashchange', function() {
    var hash = window.location.hash.slice(1);
    var clean = DOMPurify.sanitize(hash);
    document.getElementById('user-profile').innerHTML = clean;
});
```

### 5. Content Security Policy (CSP)
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self'">
```

---

## Genel Güvenlik Önerileri
- Hash ve diğer URL parametrelerini **her zaman** şüpheli kabul et.
- jQuery selector'ı (`$()`) kullanıcı verisiyle **asla** kullanma.
- `innerHTML` yerine `textContent` tercih et.
- Tüm dinamik içeriği **sanitize** et.
- CSP başlıklarını uygula.
