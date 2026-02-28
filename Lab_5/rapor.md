# DOM XSS in jQuery anchor href attribute sink using location.search source

## Lab Açıklaması
Bu laboratuvarda jQuery kullanılarak yazılmış bir sitede `location.search` kaynağından gelen verinin `href` attribute'ine nasıl DOM tabanlı XSS zafiyeti oluşturduğunu inceliyoruz.

---

## Analiz

### Uygulama Akışı
- Kullanıcı feedback (geri bildirim) yaptıktan sonra geri dön butonuyla orijinal sayfaya döndürülür.
- Site jQuery ile yazılmıştır.

### Kaynak Kod İncelemesi
```javascript
$(function() {
    $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
});
```

#### Kodun Çalışma Mantığı
- `#backLink` adlı butonun `href` attribute'ı, URL parametresinden alınan `returnPath` değeriyle ayarlanır.
- `new URLSearchParams(window.location.search)` ile URL'den `returnPath` parametresi çekiliyor.
- Bu değer herhangi bir validasyon olmadan doğrudan `href` attribute'ine atanıyor.

---

## Zafiyet Noktası
- `href` attribute'ine konulan değer **filtrelenmeden** ve **encode edilmeden** kullanılıyor.
- `href` attribute'i sadece URL değerleri değil, `javascript:` protokolünü de kabul ediyor.
- Bu, saldırganın keyfi JavaScript kodunu çalıştırmasına olanak sağlıyor.

---

## İstismar

### Saldırı Vektörü
Normal kullanım:
```
https://LabID/feedback?returnPath=/
```

Saldırgan tarafından değiştirilen URL:
```
https://LabID/feedback?returnPath=javascript:alert(1)
```

### Payload Açıklaması
- `returnPath` parametresine `javascript:alert(1)` yazılıyor.
- Geri dön butonu tıklandığında `href="javascript:alert(1)"` çalıştırılır.
- Bu, doğrudan JavaScript kodunu browser ortamında çalıştırır.

### Saldırı Adımları
1. Feedback sayfasında URL'yi değiştirerek `returnPath=javascript:alert(1)` ekle.
2. Geri dön butonuna tıkla.
3. JavaScript payload tetiklenir ve pop-up görüntülenir.

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Geri dön butonuna basınca bir **pop-up** görüntülenir. Bu, jQuery üzerinden `href` attribute'ine JavaScript protokolü enjeksiyonunun başarılı olduğunu gösterir.

---

## Zafiyet Özeti

| Unsur | Değer |
|-------|-------|
| **Kaynak (Source)** | `location.search` → `returnPath` parametresi |
| **Sink** | jQuery `.attr("href", ...)` |
| **Zafiyetin Türü** | DOM-based XSS (JavaScript Protocol Injection) |
| **Payload** | `javascript:alert(1)` |
| **Tetikleme Yolu** | Link tıklaması |

---

## Önleme Önerileri

### 1. Whitelist Kontrolü
```javascript
$(function() {
    var returnPath = (new URLSearchParams(window.location.search)).get('returnPath');
    // Sadece '/' veya belirli sayfalar izin ver
    if (returnPath === '/' || returnPath.startsWith('/safe-path')) {
        $('#backLink').attr("href", returnPath);
    } else {
        $('#backLink').attr("href", '/');
    }
});
```

### 2. URL Validasyonu
```javascript
$(function() {
    var returnPath = (new URLSearchParams(window.location.search)).get('returnPath');
    try {
        var url = new URL(returnPath, window.location.origin);
        // Aynı domain kontrol et
        if (url.origin === window.location.origin) {
            $('#backLink').attr("href", returnPath);
        }
    } catch(e) {
        $('#backLink').attr("href", '/');
    }
});
```

### 3. Genel Güvenlik Önlemleri
- Tüm dinamik `href` değerlerini **doğrula ve sanitize** et.
- `javascript:` ve `data:` protokollerini **engelle**.
- Content Security Policy (CSP) uygula.
- Kullanıcı girdilerini **her zaman** şüpheli kabul et.
