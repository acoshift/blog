# สรุป CORS แบบสั้น ๆ

CORS ย่อมาจาก Cross-Origin Resource Sharing

Origin คือ header ตัวนึง ที่บอกว่า request ถูกเรียกจากที่ไหน
จะต่างกับ Referer โดยที่ Origin จะไม่มี path มาด้วย

Origin ประกอบด้วย 3 ส่วน คือ

1. Scheme เช่น http, https
1. Hostname เช่น example.com, www.example.com
1. Port เช่น 3000, 8000, 8080, 9000

ถ้าเข้าเว็บจาก `file://` Origin จะเป็น `null`

```text
Origin: null
Origin: <scheme>://<hostname>[:<port>]
```

## Origin ที่ต่างกัน

- `http://example.com` กับ `https://example.com` **เป็นคนละ** Origin กัน เพราะ scheme ไม่เหมือนกัน คือ `http` กับ `https`
- `http://example.com` กับ `http://www.example.com` **เป็นคนละ** Origin กัน เพราะ hostname ไม่เหมือนกัน คือ `example.com` กับ `www.example.com`
- `http://example.com:3000` กับ `http://example.com:8080` **เป็นคนละ** Origin กัน เพราะ port ไม่เหมือนกัน คือ `3000` กับ `8000`
- `http://example.com` กับ `http://example.com:80` **เป็น** Origin เดียวกัน เพราะ `http` ถ้าไม่มี port จะถือว่าเป็น port `80` [RFC 6454 3.2.1](https://tools.ietf.org/html/rfc6454#section-3.2.1)

## เมื่อเกิด CORS

เมื่อเราต้องการ fetch resource จาก origin นึง ไปอีก origin นึง จะเกิดการเรียก resource ข้าม origin กัน

Browser จะส่ง `OPTIONS` request ไปถาม server อีก origin ก่อน ว่าจะอนุญาตให้ origin ต้นทางสามารถเข้าไปเรียก resource ได้ไหม

> หมายความว่า ถ้า browser ไม่ support CORS ก็จะสามารถเรียกได้เลย โดยไม่ติด CORS หรือ request เกิดจาก Server-Server คุยกัน ก็จะไม่มี CORS เพราะ คนส่ง CORS คือ Browser

ตัวอย่างเช่น

ถ้าเราอยู่ที่ `https://example.com` แล้วต้องการส่ง request ไปที่ `https://api.example.com`

```js
fetch('https://api.example.com/api1', {
    method: 'POST',
    credentials: 'include',
    headers: {
        'Content-Type': 'application/json',
        'X-Api-Key': 'key-1'
    },
    body: JSON.stringify(data)
})
```

สิ่งที่เกิดขึ้นคือ Browser จะส่ง preflight request ไปถามที่ `api.example.com` ก่อนว่า

- จะให้ Origin `https://example.com` ส่ง request ไปไหม ?
- จะให้ส่ง Method `POST` ไปไหม ?
- จะให้ส่ง Cookie (`credentials: 'include'`) ไปไหม ?
- จะให้ส่ง Header `Content-Type` กับ `X-Api-Key` ไปไหม ?

```http
OPTIONS /api1 HTTP/1.1
Host: api.example.com
Origin: https://example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, X-Api-Key

```

ถ้า Server เราจะต้องการจะตอบ Browser ว่า

- ที่ path นี้ ให้ Origin `https://example.com` ส่ง request เข้ามาได้
- ที่ path นี้ ให้ส่ง Method `GET` กับ `POST` เข้ามาได้
- ที่ path นี้ ให้ส่ง Header `Content-Type` กับ `X-Api-Key` เข้ามาได้
- ที่ path นี้ ให้ส่ง Cookie เข้ามาได้
- ที่ path นี้ ให้ JavaScript อ่าน Header `X-Request-Id` ได้, ถ้าเราไม่บอก Browser ถึงแม้เราจะส่ง header นี้กลับไปใน response แต่เราจะไม่สามารถเขียน JavaScript เข้าไปอ่าน header ตัวนี้ได้
- ที่ path นี้ ให้ Browser จำคำตอบข้างบนไว้ 3600 วินาที จะได้ไม่ต้องมาถามซ้ำ

Server จะต้องตอบ Browser กลับไปว่า

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type, X-Api-Key
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: X-Request-Id
Access-Control-Max-Age: 3600
Vary: Origin

```

> Vary ให้ในการบอก Reverse Proxy (Cache, CDN) ว่า ถ้าจะ cache response นี้ ให้ cache แยกตาม Origin เช่นเราอนุญาตให้ `example.com` เข้าได้ แต่ไม่ให้ `evil.com` เข้า แต่ถ้าเราไม่บอก CDN ว่าถ้าจะ cache ให้ cache แยกตาม Origin ตอน `evil.com` เข้ามา อาจจะได้ผลลัพท์ของ `example.com` กลับไป

ถ้าเราไม่อยากให้ Browser ส่ง request เข้ามาหล่ะ ?

ก็แค่ตอบ error อะไรกลับไปก็ได้ เช่น

```http
HTTP/1.1 403 Forbidden
Content-Type: text/plain
Content-Length: 9

Forbidden
```

## สำคัญมาก ๆ กับการ Allow all origin

บางครั้งเราต้องการให้ทุก Origin เข้ามาได้ เราสามารถใส่

```http
Access-Control-Allow-Origin: *
```

แต่เราจะ **ไม่** สามารถทำแบบนี้ได้

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

เพราะถ้าเรา login ที่เว็บ `https://bank.com` แล้วทุกเว็บจะสามารถส่ง request มาโอนเงินออกจากบัญชีเราได้หมดเลย เพราะทุกเว็บที่ส่ง request มาที่ api ของ `bank.com` browser จะแนบ Cookie ที่ได้ตอน login ที่เว็บ `bank.com` มาด้วย

บางคนอยากให้ทุกเว็บส่ง Cookie เข้ามาได้ ก็ต้องลักไก่ เอา Origin ที่มาจาก request ใส่เข้าไปใน Allow Origin แทน แต่ต้องคิดดี ๆ ก่อนทำ ไม่ฉะนั้นทุกเว็บจะได้ Cookie ไปใช้ได้เลย
