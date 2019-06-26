# ทำไมถึงไม่ใช้ JWT เก็บ Session

ปัจจุบัน JWT ถูกนำไป implement เพื่อมาเก็บ Session กันมากขึ้น วันนี้ผมจะมาเล่าประสบการณ์ว่า
ทำไมผมถึงเลิกใช้ JWT มาเก็บ session แล้วกลับไปใช้ Cookie แล้วเก็บ Session ใน Database (Redis) แทน

## ปัญหา

เริ่มท่ีปัญหา **ที่ผมเจอ** เมื่อ implement ระบบ Session ด้วย JWT

1. ไม่รู้จะเก็บ Token ไว้ที่ไหน

    ถ้าอ่านบทความเกี่ยวกับ security เขาจะแนะนำว่าให้เก็บใน memory (ตัวแปร) เท่านั้น
    ห้ามเก็บลง local storage หรือ session storage เพราะมีโอกาสโดน XSS

    แต่พอเราเก็บ Token ไว้ใน memory ปัญหาคือ user จะไม่สามารถ refresh เว็บเราได้เลย เราจะถูก logout ออกทันที

1. Logout ไม่ได้

    ถ้าเรา implement ผิด เราจะไม่สามารถ logout จากเว็บได้เลย

    หลายคนบอกว่า "ก็ลบ Token ออกจาก browser ไง"

    แต่การลบ Token ออกจาก browser ไม่ใช่การ Logout การ Logout เราจะต้อง invalidate Token ที่ฝั่ง Server ด้วย

1. จากข้างบน เลยต้อง implement Refresh Token และ Access Token ทำให้ช้ากว่าเดิม

    การจะแก้ปัญหา Logout นั้นง่ายมาก ก็ทำ Refresh Token เก็บลง Database
    แล้วใช้ Refresh Token ยิงเข้ามาขอ Access Token ที่เป็น JWT ก็สิ้นเรื่อง

    แล้วตั้งเวลา Access Token ให้หมดอายุเร็ว ๆ เท่านี้ ถ้ามีคนขโมย Access Token ไป ก็ใช้ได้แปปเดียว

    แต่ปัญหาที่ตามมาคือ แทนที่เราจะยิง API แค่รอบเดียว แต่เราต้องยิงถึง 2 รอบ (และ 4 รอบ ถ้าติด CORS)

    - (CORS) แลก Refresh Token เป็น Access Token
    - แลก Refresh Token เป็น Access Token
    - (CORS) API
    - API

1. ไม่ผ่าน Audit

    ผมเคย implement ระบบ Session ด้วย JWT (เพราะว่าไม่มี Database) แต่ไม่ผ่าน Audit ทำให้ต้องแก้ มาเก็บ Session ที่ Database แทนการใช้ JWT

## ความเข้าใจผิดของ JWT

1. เร็ว

    เพราะถ้าเก็บ Session ใน Database เราจะต้องไปดึงมาจาก Database ทุก Request

    ในความเป็นจริง เราแทบไม่ได้ยิง API ติดกันรัว ๆ โดยการ Reuse Access Token บ่อยขนาดนั้น
    ทำให้มันช้ากว่าการเก็บ Session ที่ Database ด้วยซ้ำ เพราะต่้องยิง API เพื่อไปขอ Access Token ก่อน

    นอกจากนี้เรายังสามารถใช้ In-Memory Database เช่น Redis มาเก็บ Session ก็ได้ ซึ่งมันเร็วอยู่แล้ว

1. Scale ง่าย

    ถ้าเราพัฒนา API Service ให้เป็น Stateless อยู่แล้ว ไม่น่าเกี่ยวกับการที่ JWT ทำให้ scale ง่าย

1. Logout = ลบ JWT

    เหมือนกับปัญหาเรื่อง Logout ไม่ได้ เพราะการลบ JWT ไม่ใช่การ Logout

    การ Logout คือ ถึงจะไม่ได้ลบ Token ทิ้ง Token นั้นจะต้องไม่สามารถใช้ได้ทันทีที่ Logout

1. ไม่มีปัญหาเรื่องการใช้ API หลาย Domain

    - Frontend และ API Service อยู่คนละ Origin

        เราแค่เพิ่ม `credentials: 'include'` เข้าไปตอน `fetch`

        ```js
        fetch('https://api.example.com', { credentials: 'include' })
        ```

        และเพิ่ม CORS ฝั่ง Service ให้ตอบ CORS ให้ถูก

    - API Services มีหลาย Origin

        - รวมให้เป็น Origin เดียวกันด้วย API Gateway, Reverse Proxy
        - ให้ Server ที่ ส่ง Frontend จัดการ Cookie ให้ (ให้ Server Frontend เป็น API Gateway)

    - ถ้าไม่อยากรวม Origin จริง ๆ ก็อาจจะจำเป็นต้องใช้ JWT

        - ที่ Auth Service ใช้ Cookie เหมือนเดิม (เราจะได้ไม่ต้องเก็บ Refresh Token ที่ Local/Session Storage)
        - ก่อนจะส่ง Request ไปหา Services อื่น ๆ ให้ส่ง Request ไปขอ Access Token ที่ Auth Service แล้วใช้ JWT (Access Token) จาก Auth Service ส่งไปให้ Services ที่อยู่ที่ Origin อื่น โดยเก็บ JWT ไว้ใน Memory เท่านั้น

        แต่ส่วนใหญ่แล้วเคสนี้ไม่ค่อยจะเกิดขึ้น เพราะเว็บจะช้าลง ต้องคอยยิงไปขอ Access Token บ่อย ๆ

1. ปลอดภัยกว่า Cookie

    เพราะว่า JWT ไม่มีปัญหาเรื่อง CSRF

    ใช่เลย !!! แต่มีข้อแม้ว่า ถ้าเราเก็บ JWT ใน Memory อะนะ

    จริง ๆ แล้ว CSRF (อาจจะ)​ ป้องกันง่ายกว่า XSRF นะ เท่าที่ดูคนที่เขียน Frontend มักจะชอบ เอา user input ยัดลง html ตรง ๆ บ่อย (เช่น Vue ก็มักจะใช้ v-html กัน)

## เมื่อไรเราควรใช้ JWT

1. เราเป็น OAuth Provider ที่ต้องการให้ Service ที่**คนอื่นเขียน** มาดึง resource ของ user ของเรา

    ถ้าเราไม่ได้ต้องการให้ Service ของคนอื่นมาดึง resource ของ user ก็ไม่จำเป็นต้อง implement OAuth

1. Service-Service คุยกัน ไม่ผ่าน Frontend

    จริง ๆ นอกจาก JWT แล้วก็ใช้เป็น API-Key ก็ได้เหมือนกัน

    ลองอ่านเรื่อง [Service Account](https://cloud.google.com/iam/docs/understanding-service-accounts) ของ Google Cloud ได้

> ขอไม่พูดถึงพวก Signed URL เพราะเราไม่ได้เป็นคนทำระบบ Auth เราแค่สร้าง Access Token (JWT) แล้วส่งต่อไปให้ Frontend/Service อื่น เพื่อทำอะไรบางอย่างแทน Service เรา

## ความเข้าใจผิดเกี่ยวกับ Cookie และการเก็บ Session ใน Database

1. ไม่ Cross Platform

    จริง ๆ แล้ว Mobile, Service ก็อ่าน Cookie ได้เหมือนกัน เพราะมันเป็นแค่ HTTP Header ตัวนึงเท่านั้นเอง

    แต่ถ้าไม่อยากอ่านจาก Cookie ก็แค่เพิ่ม Flag บางอย่างให้ API ส่ง Session ID กลับมาทาง Body ก็ได้ โดยที่ถ้าเป็น Mobile ก็ค่อยส่ง Flag นี้เข้ามา ถ้าเป็น Web ก็ไม่ต้องส่ง

1. ช้า เพราะต้องดึง Database ทุก Request

    ไปใช้ In-Memory Database ก็ไม่ช้าแล้ว และอย่าดึง Session ถ้า Request วิ่งไปขอ Assets ไม่ได้วิ่งมาที่ API

1. ไม่ปลอดภัย

    เราต้อง config ให้ถูก เช่นใส่ HttpOnly, Secure flag ให้ครบ
    หรือถ้า Frontend และ API Service อยู่บน Origin เดียวกัน ก็ใส่ SameSite ไปด้วย

    นอกจากนี้ส่วนใหญ่แล้วจะมี library ที่ช่วยจัดการพวก CSRF ให้อยู่แล้ว

## ข้อดีของการใช้ Cookie

1. Web Frontend เขียนง่าย ไม่ต้องทำอะไรเลย ก็สามารถใช้ API ได้เลย

1. Security บางอย่าง Browser ทำให้แล้ว แค่ config ให้ถูก

## ข้อดีของการเก็บ Session ไว้ใน Database

1. สามารถ invalidate session ตอนไหนก็ได้

    เช่น
    - user เปลี่ยนรหัสผ่าน ก็ลบ session ของ user คนนั้นทั้งหมดออก เพื่อให้ login ใหม่
    - สามารถทำปุ่ม Logout จาก Session อื่น ๆ ได้
    - สามารถรู้ได้ว่ามี Session ไหน Active อยู่, Logout session ใน device ที่ไม่ได้ใช้แล้วออกได้

1. สามารถต่ออายุ Session ได้เรื่อย ๆ โดยไม่ต้องสร้าง Session ID ใหม่ (ทำ Idle Timeout ง่าย)

## ข้อเสียของการใช้ Cookie

1. ถ้าเราใช้ Cookie ข้าม Origin แล้ว User ที่ตั้งค่าใน Browser ให้ Block all 3rd party cookies จะไม่สามารถใช้เว็บเราได้ (แก้ได้ด้วยการรวม Origin)

1. ใช้ Bandwidth มากขึ้น และมีโอกาสที่ Cookie จะหลุดมากกว่า เพราะทุก Request จะมี Cookie แนบมาด้วย

    อาจจะต้องมี mechanism อะไรบางอย่างเพื่อตรวจจับว่ามีการใช้ Session ID เดียวกันจาก 2 ที่พร้อมกัน ให้ Logout Session นั้นทิ้งไปเลย

## สิ่งที่ต้องระวังเวลาเก็บ Session ใน Database

1. อย่าลืม Hash Session ID ก่อนเก็บ ไม่อย่างนั้นถ้า Database ที่เก็บ Session หลุด คนที่ได้ไปสามารถเอา Session ID ไปใช้ได้เลย

1. อย่าลืม Rotate Session ID เวลา login เพื่อป้องกัน Session Fixation
