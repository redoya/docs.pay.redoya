
# Tam Dökümantasyon
## Ana URL: https://pay.redoya.net
## Sistem nasıl çalışıyor?
Aşağıda belirtildiği gibi bir ödeme linki oluşturuyorsunuz. Ödeme linki şifreli bir şekilde fiyat, başarılı/iptal yönlendirme linki, "Test modunda mı?" gibi veriler taşır.
<br />
``GET /checkout#orderToken#identifier``
<br />
Ödeme linkini oluşturduktan sonra bu linke kullanıcıyı yönlendirirsiniz. Oluşturulan link ödeme sayfasıdır.
Sonra kullanıcı ödeme sayfasında ödemeyi tamamladıktan ve ödeme işlendikten sonra başarılı/iptal URL'sine yönlendirilir. Eğer bir faturayı takip etmeniz gerekiyorsa başarılı URL'sine querystring olarak faturaId ekleyebilirsiniz.

## Endpointler Tablosu
| METOD  | URL                              | BODY PARAMETRELERİ |
|--------|----------------------------------|--------------------|
| GET    | /checkout#orderToken#identifier  | -                  |
| POST   | /api/v1/order/verify/:orderId    | identifier         |

## Değişken Tanımlamaları
### Secret Key/Gizli anahtar
Bu yalnızca pay.redoya.net ve satıcının bildiği rastgele üretilmiş bir tokendir. Sipariş verisini şifrelemek ve deşifre etmek için kullanılır. **BU SECRET KEY SİZDEN HİÇBİR ZAMAN İSTENMEZ VE BUNU KİMSEYLE PAYLAŞMAYINIZ. Eğer çalındıysa, en kısa sürede yeni secret key talep etmek için destek talebi açınız.**
### Identifier/Vendor/Satıcı Tokeni
Bu herkese açık bir tokendir ve satıcıyı belirtmek için kullanılır. Satıcı ID gibi düşünebilirsiniz.
### Order/Sipariş Tokeni
JSON biçminde tutulan sipariş verisinin şifrelenmiş hâlidir. JSON Web Token, HS256 ile şifrelenmelidir.
### {order_id}
Bu değişken sipariş verisi (orderData) oluştururken kullanılır. Bu değişkeni gelecek sipariş ID'sini başarılı/iptal URL'sinde istediğiniz gere koyarak kullanabilirsiniz. Sipariş ID'si ödemenin durumunu kontrol etmek için kullanılır.

## Sipariş tokeni nasıl üretilir?
[PHP örneği için tıklayın](https://github.com/redoya/php-demo.pay.redoya.net)
<br />

Core PHP için HMACSHA256 fonksiyonunu kaynaktaki gibi kullanabilirsiniz. [Kaynak: https://seegatesite.com/php-json-web-token-tutorial-for-beginners/](https://seegatesite.com/php-json-web-token-tutorial-for-beginners/)
<br />

[NodeJS örneği için tıklayın](https://github.com/redoya/node-pay-redoya/tree/main/example)

## API
### GET /checkout#orderToken#identifier
Calculate the order token:
<br />
**orderData:**
```jsonc
{
  "price": priceFloat, // minimum değer: 3, maksimum değer: 1000
  // Ondalık/virgüllü sayı girerek kuruş ayarlayabilirsiniz. (örn 5.56)
  "successfulURL": "http://example.com/order/success/{order_id}?invoiceId=123",
  // tür: string, en kısa uzunluk: 10, en fazla uzunluk: 1200
  "failURL": "https://redoya.net/odeme/iptal/{order_id}?invoiceId=123",
  // tür: string, min length: 10, max length: 1200
  "isInTestMode": 1,
  // tür: number (boolean türü henüz desteklenmiyor), minimum: 0, maksimum: 1
}
```

**orderToken:** ``JWT.sign(orderData, secret, { expiresIn: secondsInt });``
<br />
Değişkenleri birleştirerek linki üretin:
<br />

**identifier: Redoya'ya destek talebi açarak talep edebilirsin.**
<br />
``https://pay.redoya.net/checkout#<orderToken>#<identifier>``

### POST /api/v1/order/verify/:orderId
İSTEK
------------
**Body parametreleri/şeması:**
```jsonc
{
  "identifier": "Redoya'ya destek talebi açarak talep edebilirsin.",
}
```

YANIT
--------------
**Başarılı ödeme gerçekleştiyse:**
```jsonc
{
  "orderId": "16 karakter uzunluğunda bir string.",
  "status": "success",
  "total_amount": "Müşterinin kartından kesilen paranın (banka komisyonu vb.ni de dahil eder) 100 ile çarpılmış string hâli",
  "payment_type": "card",
  "payment_amount": "Siparişte belirtilen ücretin 100 ile çarpılmış string hâli",
  "currency": "TL",
  "testMode": "0 veya 1",
  "timestamp": timestampInSecondsInt, // Ödemenin gerçekleştiği timestamp
  "uses": IntegerTüründeSatıcınınGönderdiğiDoğrulamaİsteğiSayısı,
}
```
**Başarısız ödeme gerçekleşmişse:**
```jsonc
{
  "orderId": "16 karakter uzunluğunda bir string.",
  "status": "failed",
  "failed_reason_code": "0'dan 99'a kadar bir sayı, PayTR'den alınan hata kodudur.",
  "failed_reason_msg": "Ödemenin neden kabul edilmediğini detaylıca açıklayan bir metindir.",
  "total_amount": "0",
  "payment_type": "card",
  "testMode": "0 or 1",
  "timestamp": timestampInSecondsInt, // Ödemede hata oluşan tarihin timestampi
  "uses": IntegerTüründeSatıcınınGönderdiğiDoğrulamaİsteğiSayısı,
}
```
**On error:**
```jsonc
{
  "name":  "MoleculerError",
  "message":  "Hatayı açıklayan bir mesaj.",
  "code":  400,
  "type":  "HATANIN_TÜRÜ" // Her zaman SNAKE_CASE olarak gelir.
}
```
| KODU | Type                  | Message                                   |
|------|-----------------------|-------------------------------------------|
| 400  | NOT_CONFIRMED_YET     | The order is not end up yet.              |
| 400  | ORDER_TIMED_OUT       | It's been a long time since the order was created. |
| 404  | UNKNOWN_ORDER         | Order with specified ID is not found.     |
| 500  | UNKNOWN_ERROR         | Unknown error! Contact the administrator. |


## Örnek istekler
### GET /checkout#orderToken#identifier
**orderData:**
```jsonc
{
  "price": 15,
  "successfulURL": "https://redoya.net/odeme/basarili/{order_id}?invoiceId=123",
  "failURL": "https://redoya.net/odeme/iptal/{order_id}?invoiceId=123",
  "isInTestMode": 1,
}
```
**orderToken:** ``JWT.sign(orderData, secret, { expiresIn: 15 * 60 });``
<br />

**orderToken:** ``eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfa2V5IjoiMTg1ODM3OCIsIl9pZCI6IndlYnNpdGVzLzE4NTgzNzgiLCJfcmV2IjoiX2NRMHNiMi0tLS0iLCJlbWFpbCI6InVuaXR5dGhlbWFrZXJAZ21haWwuY29tIiwiZmlyc3ROYW1lIjoiSGFsaWwiLCJsYXN0TmFtZSI6IktBUkFCVUxVVCIsImFkZHJlc3MiOiJHw7Zrc3UgbWFoLiA1MzI4LiBjYWQuIG5vOjggVmFkaXRlcGUgQmHFn3DEsW5hciBzaXRlc2ksIEEyOCBBbmthcmEsIEV0aW1lc2d1dCwgMDY4MjAgVHVya2V5IiwicGhvbmUiOiIwNTU0MTMxMzQ3NSIsInNlY3JldCI6InNnUHViZEJWZE8zT2lINGRGM2dkWnJPUXQ1cDVXVXQxRENrU1JqWW1BOFRrVUlKV1pZYkVWbExSYzR1Nm1pN3UzU3JrcVVzTyIsIndobWNzX2lkIjoxMzAsImlhdCI6MTYyMDIwNTMxMiwiZXhwIjoxNjI3OTgxMzEyfQ.icnvXiaHBzJkt7Jty17FLuh_Q7HxpFHJgX6iQcnTAA4``
<br />

**identifier/vendorToken:** ``eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfa2V5IjoiMTg1ODM3OCIsIl9pZCI6IndlYnNpdGVzLzE4NTgzNzgiLCJfcmV2IjoiX2NRMHNiMi0tLS0iLCJlbWFpbCI6InVuaXR5dGhlbWFrZXJAZ21haWwuY29tIiwiZmlyc3ROYW1lIjoiSGFsaWwiLCJsYXN0TmFtZSI6IktBUkFCVUxVVCIsImFkZHJlc3MiOiJHw7Zrc3UgbWFoLiA1MzI4LiBjYWQuIG5vOjggVmFkaXRlcGUgQmHFn3DEsW5hciBzaXRlc2ksIEEyOCBBbmthcmEsIEV0aW1lc2d1dCwgMDY4MjAgVHVya2V5IiwicGhvbmUiOiIwNTU0MTMxMzQ3NSIsInNlY3JldCI6InNnUHViZEJWZE8zT2lINGRGM2dkWnJPUXQ1cDVXVXQxRENrU1JqWW1BOFRrVUlKV1pZYkVWbExSYzR1Nm1pN3UzU3JrcVVzTyIsIndobWNzX2lkIjoxMzAsImlhdCI6MTYyMDExNjE3NywiZXhwIjoxNjI3ODkyMTc3fQ.JFN1tDGnYfdz3RWQntGsl9wD8LgVXQ0WpgbverD3SCg``
<br />

**Bu değişkenlerle, link şu olacaktır:** https://pay.redoya.net/checkout#eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwcmljZSI6MTUsInN1Y2Nlc3NmdWxVUkwiOiJodHRwczovL3JlZG95YS5uZXQvb2RlbWUvYmFzYXJpbGkve29yZGVyX2lkfT9pbnZvaWNlSWQ9MTIzIiwiZmFpbFVSTCI6Imh0dHBzOi8vcmVkb3lhLm5ldC9vZGVtZS9pcHRhbC97b3JkZXJfaWR9P2ludm9pY2VJZD0xMjMiLCJpc0luVGVzdE1vZGUiOjEsImlhdCI6MTYyMDIwNTMxMn0.Bs91gdIkOPuvIVe1owxoSrrolSUdgJwHbaYqrmu47gM#eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfa2V5IjoiMTg1ODM3OCIsIl9pZCI6IndlYnNpdGVzLzE4NTgzNzgiLCJfcmV2IjoiX2NRMHNiMi0tLS0iLCJlbWFpbCI6InVuaXR5dGhlbWFrZXJAZ21haWwuY29tIiwiZmlyc3ROYW1lIjoiSGFsaWwiLCJsYXN0TmFtZSI6IktBUkFCVUxVVCIsImFkZHJlc3MiOiJHw7Zrc3UgbWFoLiA1MzI4LiBjYWQuIG5vOjggVmFkaXRlcGUgQmHFn3DEsW5hciBzaXRlc2ksIEEyOCBBbmthcmEsIEV0aW1lc2d1dCwgMDY4MjAgVHVya2V5IiwicGhvbmUiOiIwNTU0MTMxMzQ3NSIsInNlY3JldCI6InNnUHViZEJWZE8zT2lINGRGM2dkWnJPUXQ1cDVXVXQxRENrU1JqWW1BOFRrVUlKV1pZYkVWbExSYzR1Nm1pN3UzU3JrcVVzTyIsIndobWNzX2lkIjoxMzAsImlhdCI6MTYyMDIwNTMxMiwiZXhwIjoxNjI3OTgxMzEyfQ.icnvXiaHBzJkt7Jty17FLuh_Q7HxpFHJgX6iQcnTAA4
