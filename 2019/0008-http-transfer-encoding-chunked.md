# Transfer-Encoding: chunked ใน HTTP คืออะไร

เวลาที่เราส่ง HTTP Request ไป เราจะต้องรู้ว่า content ที่เราจะส่งมีขนาดเท่าไร
ไม่อย่างนั้น server ปลายทางจะไม่รู้ว่าเมื่อไรจบ 1 request

เพราะว่าปกติแล้ว เราจะ reuse connection กัน, 1 connection รับ/ส่ง กันหลาย HTTP requests

เช่น

```text
POST / HTTP/1.1
Host: example.com
Accept: */*
Content-Type: text/plain
Content-Length: 4

textPOST / HTTP/1.1
Host: example.com
Accept: */*
Content-Type: application/json
Content-Length: 15

{"name":"test"}GET / HTTP/1.1
Host: example.com
Accept: */*

```

เห็นไหมว่า ถ้าเราไม่ส่ง `Content-Length` ไป server ปลายทางจะรู้ได้ยังไงว่ามันจบ 1 request แล้ว ?

แล้วถ้าเราไม่รู้หล่ะ ว่า content มันขนาดเท่าไร ?

ใน HTTP/1.1 มี header ตัวนึงสำหรับ stream data คือ

```text
Transfer-Encoding: chunked
```

> ใน HTTP/2 ไม่ support chunked เพราะมีวิธี stream data ที่ดีกว่า

โดยปกติถ้าเราไม่ได้ส่ง `Transfer-Encoding` ไปใน header, เราจะรู้กันว่ามันคือ

```text
Transfer-Encoding: identity
```

## Chunked Transfer Encoding ทำงานยังไง

หลักการทำงานของ chunked transfer encoding ก็คือ

ส่งจำนวน bytes ที่จะส่งแต่ละครั้ง ตามด้วย `\r\n` ตามด้วย data และตามด้วย `\r\n`
ถ้าต้องการจบ stream ก็แค่บอกว่าจะส่ง chunked 0 byte

เช่น

> `\r\n` คือการขึ้นบรรทัดใหม่ เพราะฉะนั้นในตัวอย่างจะใช้การขึ้นบรรทัดใหม่แทน `\r\n`

```text
POST / HTTP/1.1
Host: example.com
Accept: */*
Content-Type: text/plain
Transfer-Encoding: chunked

4
test
5
hello
15
acoshift's blog
0


```

ถ้าเราใช้ `\r\n` แทนการขึ้นบรรทัดใหม่ เราจะได้

```text
4\r\ntest\r\n5\r\nhello\r\n15\r\nacoshift's blog\r\n0\r\n\r\n
```

## ข้อเสียของการใช้ chunked transfer encoding

คือ ไม่รู้ว่า content ขนาดเท่าไร

ตัวอย่าง ถ้าเราทำ api ให้ upload ไฟล์ แต่อยากจะ limit แค่ 100 MiB
ถ้า client ส่ง `Content-Length` มา เรารู้ทันทีว่าไฟล์ขนาดเท่าไร ถ้าเกิน 100 MiB สามารถตอบได้ทันทีว่า Bad Request หรือ Payload Too Large อะไรก็ว่าไป

แต่พอเป็น chunked transfer encoding แล้ว server จะต้องอ่าน content มาก่อน 100 MiB ถึงจะรู้ว่าขนาดมันเกิน

คำถามคือ แล้ว 100 MiB นี่อยู่ที่ไหน ?

ถ้า implement ง่าย ๆ ก็เก็บไว้ในตัวแปรสิ... โดนยิงพร้อม ๆ กัน แน่นอน server ram เต็มแน่ ๆ

ถ้าให้ดีหน่อยก็อาจจะเก็บลง disk ไปก่อน แต่ก็จะ implement ยากขึ้นมา เพราะถ้า body เล็กมาก ๆ เช่น json ไม่กี่ byte แล้วเราต้องเก็บลงไฟล์ เว็บเราจะช้าลงขนาดไหน ?

แต่ถ้าอยากให้เร็ว ๆ และไม่ให้โดนยิงร่วง ก็จะต้องมี buffer ที่รองรับไฟล์เล็ก ๆ เช่น json ถ้าเก็บเกิน buffer ก็ค่อยเอาลง disk แน่นอนว่า implement ยากแน่ ๆ
