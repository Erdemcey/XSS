# DOM XSS in document.write sink using source location.search inside a select element

## Lab Açıklaması
Bu laboratuvarda, `location.search` kaynağından gelen değerin bir `<select>` elementinin içine `document.write()` ile nasıl gömüldüğünü ve bunun DOM tabanlı XSS yolunu açtığını inceliyoruz.

---

## Analiz

### Kaynak Kod İncelemesi
```javascript
var stores = ["London","Paris","Milan"];
var store = (new URLSearchParams(window.location.search)).get('storeId');
document.write('<select name="storeId">');
if(store) {
    document.write('<option selected>'+store+'</option>');
}
for(var i=0;i<stores.length;i++) {
    if(stores[i] === store) {
        continue;
    }
    document.write('<option>'+stores[i]+'</option>');
}
document.write('</select>');
```

#### Kodun Mantığı
1. `stores` dizisi sabit mağaza isimlerini içerir.
2. URL'den `storeId` parametresi alınır; örneğin `?storeId=Paris`.
3. `<select>` etiketinin açılışı `document.write()` ile yazılır.
4. Eğer `store` değeri varsa, bu değer `<option selected>` olarak eklenir (kullanıcının seçili gördüğü öğe).
5. Döngü ile `stores` dizisindeki diğer şehirler `<option>` olarak eklenir, fakat seçili olan şehir tekrarlanmamak üzere atlanır.
6. Son olarak `</select>` yazılarak seçim kutusu kapatılır.

### Zafiyet Noktası
- `store` değişkeni URL'den doğrudan çekiliyor ve hiçbir filtreleme ya da encoding yapılmadan HTML'e iletiliyor.
- `document.write()` kullanıldığı için saldırgan değerleri doğrudan HTML markup olarak enjekte edebiliyor.
- Context bir `<option>` elementinin içinde olmakla beraber, `selected` kısmındaki veri herhangi bir HTML içeriğe dönüştürülebilir.

---

## İstismar

### Attack Vektörü
- Amaç, `<select>` elementinin dışına çıkıp ek HTML/JavaScript enjekte etmek.
- İçeriğin birden fazla satırda uçmasını önlemek için `">` ile kapanış yapılmalı ve ardından yeni bir element oluşturulmalı.

### Payload
```
product?productId=1&storeId=\">\</select><img%20src=1%20onerror=alert(1)>
```

> Not: payload URL içinde gönderilir; `"` karakteri escape edilmelidir (örnekte yüzde kodlamasıyla gösterilmiştir).

#### Payload Açıklaması
1. `">` → `<option>` etiketini kapatır ve `<select>` içinde başlayıp bitmemesini sağlar.
2. `</select>` → seçme kutusunu kapatır, böylece geri kalan HTML dışarı çıkar.
3. `<img src=1 onerror=alert(1)>` → geçersiz bir resim kaynağı verilir, `onerror` ile alert() çalıştırılır.

### Saldırı Adımları
1. Tarayıcı adres çubuğuna laboratuvar URL'sini `?storeId=` parametresiyle birlikte payload olarak yaz.
2. Sayfayı yükle.
3. Seçim kutusu açıldığında veya herhangi bir yere tıklamada `alert(1)` pop-up gösterir.
4. Lab böylece tamamlanır.

---

## Sonuç

✅ **Lab Başarıyla Tamamlandı**

Payload, `<select>` elemanının dışına çıkarak bir `<img>` etiketi yerleştirdi ve `onerror` olayını tetikletti; bu da DOM XSS zafiyetinin başarılı bir şekilde istismar edildiğini gösterir.

---

## Zafiyet Özeti

| Unsur | Değer |
|-------|-------|
| **Kaynak (Source)** | `location.search` → `storeId` parametresi |
| **Sink** | `document.write()` (select içinde) |
| **Zafiyet Türü** | DOM-based XSS (option injection) |
| **Payload** | `"> </select><img src=1 onerror=alert(1)>` |
| **Tetikleme Yolu** | Sayfa yükleme/parametre değişikliği |

---

## Önleme Önerileri

### 1. Output Encoding
```javascript
function htmlEncode(str) {
    return str.replace(/&/g, '&amp;')
              .replace(/</g, '&lt;')
              .replace(/>/g, '&gt;')
              .replace(/"/g, '&quot;')
              .replace(/'/g, '&#39;');
}
// Kullanım
if(store) {
    document.write('<option selected>' + htmlEncode(store) + '</option>');
}
```

### 2. DOM API ile Element Yazma
```javascript
var select = document.createElement('select');
select.name = 'storeId';
if (store) {
    var option = document.createElement('option');
    option.selected = true;
    option.textContent = store; // otomatik escape
    select.appendChild(option);
}
stores.forEach(function(s) {
    if (s === store) return;
    var opt = document.createElement('option');
    opt.textContent = s;
    select.appendChild(opt);
});
document.body.appendChild(select);
```

### 3. URL Validasyonu / Whitelisting
```javascript
var allowed = ['London','Paris','Milan'];
if (!allowed.includes(store)) {
    store = '';
}
```

### 4. Content Security Policy (CSP)
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self';">
```

### 5. Server-side Filtering
```javascript
// Node.js örneği
function sanitizeStoreId(val) {
    return val.replace(/[^a-zA-Z0-9]/g, '');
}
```

---

## Genel Güvenlik Öğütleri
- `document.write()` gibi doğrudan yazma fonksiyonlarından kaçının.
- Kullanıcının gönderdiği tüm değerleri **encode** edin ve/veya **whitelist** ile sınırlayın.
- DOM elementlerini programlı olarak oluşturun (`createElement`, `textContent`).
- Tekrar kullanılabilir kaçış fonksiyonları yazın veya güvenilir kütüphaneler kullanın.
- CSP ve diğer header'lar ile tarayıcının yüklediği içeriği kısıtlayın.

---

