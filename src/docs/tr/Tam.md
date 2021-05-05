
# Tam Dökümantasyon
## Ana URL: https://pay.redoya.net
## Sistem nasıl çalışıyor?
Aşağıda belirtildiği gibi bir ödeme linki oluşturuyorsunuz. Ödeme linki şifreli bir şekilde fiyat, başarılı/iptal yönlendirme linki, "Test modunda mı?" gibi veriler taşır.
<br />
``GET /checkout#orderToken#identifier``
<br />
Ödeme linkini oluşturduktan sonra bu linke kullanıcıyı yönlendirirsiniz. Oluşturulan link ödeme sayfasıdır.
Sonra kullanıcı ödeme sayfasında ödemeyi tamamladıktan ve ödeme işlendikten sonra başarılı/iptal URL'sine yönlendirilir. Eğer bir faturayı takip etmeniz gerekiyorsa başarılı URL'sine querystring olarak faturaId ekleyebilirsiniz.

## Table of Endpoints
| METHOD | URL                              | BODY PARAMS |
|--------|----------------------------------|-------------|
| GET    | /checkout#orderToken#identifier  | -           |
| POST   | /api/v1/order/verify/:orderId    | identifier  |

## Değişken Tanımlamaları
### Secret Key
Bu yalnızca pay.redoya.net ve satıcının bildiği rastgele üretilmiş bir tokendir. Sipariş verisini şifrelemek ve deşifre etmek için kullanılır. **BU SECRET KEY SİZDEN HİÇBİR ZAMAN İSTENMEZ VE BUNU KİMSEYLE PAYLAŞMAYINIZ. Eğer çalındıysa, en kısa sürede yeni secret key talep etmek için destek talebi açınız.**
### Identifier/Vendor/Satıcı Tokeni
Bu herkese açık bir tokendir ve satıcıyı belirtmek için kullanılır. Satıcı ID gibi düşünebilirsiniz.
### Order/Sipariş Tokeni
JSON biçminde tutulan sipariş verisinin şifrelenmiş hâlidir. JSON Web Token, HS256 ile şifrelenmelidir.
### {order_id}
Bu değişken sipariş verisi (orderData) oluştururken kullanılır. Bu değişkeni gelecek sipariş ID'sini başarılı/iptal URL'sinde istediğiniz gere koyarak kullanabilirsiniz. Sipariş ID'si ödemenin durumunu kontrol etmek için kullanılır.

## How to create an order token?
Click for PHP example (Soon)
<br />
Click for NodeJS example (Soon)

## API
### GET /checkout#orderToken#identifier
Calculate the order token:
<br />
**orderData:**
```jsonc
{
  "price": priceInt, // min value: 3, max value: 1000
  // Float prices will be accepted in the next update.
  "successfulURL": "http://example.com/order/success/{order_id}?invoiceId=123",
  // type: string, min length: 10, max length: 1200
  "failURL": "https://redoya.net/odeme/iptal/{order_id}?invoiceId=123",
  // type: string, min length: 10, max length: 1200
  "isInTestMode": 1,
  // type: number (boolean is not supported yet), min: 0, max: 1
}
```

**orderToken:** ``JWT.sign(orderData, secret, { expiresIn: secondsInt });``
<br />
Combine the variables and create the payment link:
<br />

**identifier: You must request this from Redoya.**
<br />
``https://pay.redoya.net/checkout#<orderToken>#<identifier>``

### POST /api/v1/order/verify/:orderId
REQUEST
------------
**Body parameters/schema:**
```jsonc
{
  "identifier": "You must request this from Redoya.",
}
```

RESPONSE
--------------
**On successful payment:**
```jsonc
{
  "orderId": "A string with 16 characters length.",
  "status": "success",
  "total_amount": "Money amount withheld from the customer (includes fee etc.). String and multiplied by 100",
  "payment_type": "card",
  "payment_amount": "Money amount defined in the order. String and multiplied by 100",
  "currency": "TL",
  "testMode": "0 or 1",
  "timestamp": timestampInSecondsInt, // Timestamp of when the payment made
  "uses": countOfVerifyRequestsTheVendorHaveMadeInt,
}
```
**On failed payment:**
```jsonc
{
  "orderId": "A string with 16 characters length.",
  "status": "failed",
  "failed_reason_code": "From 0 to 99. Recieved from PayTR API.",
  "failed_reason_msg": "Message that contains detailed information explains why the payment not accepted.",
  "total_amount": "0",
  "payment_type": "card",
  "testMode": "0 or 1",
  "timestamp": timestampInSecondsInt, // Timestamp of when the payment failed
  "uses": countOfVerifyRequestsTheVendorHaveMadeInt,
}
```
**On error:**
```jsonc
{
  "name":  "MoleculerError",
  "message":  "A message that explains the error.",
  "code":  400,
  "type":  "TYPEOF_ERROR_WITH_SNAKE_CASE"
}
```
| Code | Type                  | Message                                   |
|------|-----------------------|-------------------------------------------|
| 400  | NOT_CONFIRMED_YET     | The order is not end up yet.              |
| 400  | ORDER_TIMED_OUT       | It's been a long time since the order was created. |
| 404  | UNKNOWN_ORDER         | Order with specified ID is not found.     |
| 500  | UNKNOWN_ERROR         | Unknown error! Contact the administrator. |


## Example Requests
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

**With this variables, the link will be:** https://pay.redoya.net/checkout#eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwcmljZSI6MTUsInN1Y2Nlc3NmdWxVUkwiOiJodHRwczovL3JlZG95YS5uZXQvb2RlbWUvYmFzYXJpbGkve29yZGVyX2lkfT9pbnZvaWNlSWQ9MTIzIiwiZmFpbFVSTCI6Imh0dHBzOi8vcmVkb3lhLm5ldC9vZGVtZS9pcHRhbC97b3JkZXJfaWR9P2ludm9pY2VJZD0xMjMiLCJpc0luVGVzdE1vZGUiOjEsImlhdCI6MTYyMDIwNTMxMn0.Bs91gdIkOPuvIVe1owxoSrrolSUdgJwHbaYqrmu47gM#eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfa2V5IjoiMTg1ODM3OCIsIl9pZCI6IndlYnNpdGVzLzE4NTgzNzgiLCJfcmV2IjoiX2NRMHNiMi0tLS0iLCJlbWFpbCI6InVuaXR5dGhlbWFrZXJAZ21haWwuY29tIiwiZmlyc3ROYW1lIjoiSGFsaWwiLCJsYXN0TmFtZSI6IktBUkFCVUxVVCIsImFkZHJlc3MiOiJHw7Zrc3UgbWFoLiA1MzI4LiBjYWQuIG5vOjggVmFkaXRlcGUgQmHFn3DEsW5hciBzaXRlc2ksIEEyOCBBbmthcmEsIEV0aW1lc2d1dCwgMDY4MjAgVHVya2V5IiwicGhvbmUiOiIwNTU0MTMxMzQ3NSIsInNlY3JldCI6InNnUHViZEJWZE8zT2lINGRGM2dkWnJPUXQ1cDVXVXQxRENrU1JqWW1BOFRrVUlKV1pZYkVWbExSYzR1Nm1pN3UzU3JrcVVzTyIsIndobWNzX2lkIjoxMzAsImlhdCI6MTYyMDIwNTMxMiwiZXhwIjoxNjI3OTgxMzEyfQ.icnvXiaHBzJkt7Jty17FLuh_Q7HxpFHJgX6iQcnTAA4
