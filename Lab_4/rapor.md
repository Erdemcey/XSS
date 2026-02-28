# DOM XSS in innerHTML sink using source location.search

## Lab Açıklaması
Bu laboratuvarda `location.search` kaynağından gelen verinin `innerHTML` sink'ine nasıl DOM tabanlı XSS zafiyeti oluşturduğunu inceliyoruz.

---

## Analiz

### Kaynak Kod İncelemesi
```javascript
function doSearchQuery(query) {
    document.getElementById('searchMessage').innerHTML = query;
}
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    doSearchQuery(query);
}
```

#### 1. URL Parametresinin Alınması
- `new URLSearchParams(window.location.search)` ile URL'den `search` parametresi çekiliyor.
- Parametre boş değilse `doSearchQuery()` fonksiyonu çağrılıyor.

#### 2. İçeriğin DOM'a Yazılması
- `doSearchQuery()` fonksiyonu aldığı `query` değerini `searchMessage` id'li elemana `innerHTML` olarak atıyor.
- `innerHTML` olduğu için içerik HTML olarak işleniyor; bu da kullanıcı verisinin tarayıcıda yorumlanmasına izin veriyor.

---

## Zafiyet Noktası
- Kullanıcı kontrolündeki `query` değeri **hiçbir filtreleme veya encoding** yapılmadan DOM'a yazılıyor.
- Bu durum, saldırganın HTML/JavaScript enjeksiyonuna yol açıyor.

---

## İstismar

### Saldırı Stratejisi
- `query` parametresine HTML tagı eklemeliyiz.
- Basit bir `img` etiketi ve `onerror` olayı kullanarak execute edeceğiz.

### Payload

```
<img src=1 onerror=alert(1)>
```

**Açıklama:**
- Kaynak (src) geçersiz olduğundan resim yüklenemez.
- `onerror` olayı tetiklenir ve JavaScript çalışır.

---

## Sonuç

Sayfada bir **pop-up** görüntülenir; bu, DOM XSS zafiyetinin başarılı bir şekilde istismar edildiğini gösterir.
Laboratuvar işi bu şekilde tamamlanır.

---

## Önleme Önerileri
- **innerHTML kullanmaktan kaçının**; bunun yerine `textContent` veya DOM elementleri oluşturun.
- Kullanıcı girdilerini **encode** edin (`encodeURIComponent`, `textContent` vs.).
- İçerik Güvenlik Politikası (CSP) uygulayın.
- Her zaman **sanitize** ve **validate** adımlarını ekleyin.

---

## Güvenli Örnek Kod

```javascript
function doSearchQuery(query) {
    var elem = document.getElementById('searchMessage');
    elem.textContent = query; // HTML olarak yorumlanmaz
}
```
