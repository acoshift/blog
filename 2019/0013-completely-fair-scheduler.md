# Completely Fair Scheduler (CFS)

ถ้าเรามีงาน 3 งานที่จะต้องแบ่งเวลากันทำ
โดยที่ งานที่ 1 มีความสำคัญมากที่สุด งานที่ 2 และ 3 สำคัญเท่ากัน

| งาน | เวลา |
|---|---|
| 1 | 3 วัน |
| 2 | 2 วัน |
| 3 | 2 วัน |

แล้วเราจัดตารางเวลาไว้แบบนี้

| งาน | อา. | จ. | อ. | พ. | พฤ​. | ศ.​ | ส. |
|---|---|---|---|---|---|---|---|
| 1 | x |   | x |   | x |   |   |
| 2 |   | x |   |   |   | x |   |
| 3 |   |   |   | x |   |   | x |

ถ้าเกิดวัน ศ. มีงานด่วนของงานที่ 1 มาหล่ะ จะเกิดอะไรขึ้น ?

แน่นอนว่าเราไม่สามารถทำได้ เพราะหมด quota ของงานที่ 1 ไปแล้ว

แต่ถ้าเราเอางานที่ 1 มาทำในเวลาของงานที่ 2 มันก็จะเป็นการไม่แฟร์กับงานที่ 2

จะแก้ปัญหานี้ยังไงดี ?

---

ก็แตกเวลาให้เล็กลงสิ

| งาน | 8-10 น. | 10-12 น. | 12-14 น. | 14-16 น. | 16-18 น. | 18-20 น. | 20-22 น. |
|---|---|---|---|---|---|---|---|
| 1 | x |   |   | x |   |   | x |
| 2 |   | x |   |   | x |   |   |
| 3 |   |   | x |   |   | x |   |

ถ้ามีงานแทรง เราก็สามารถทำในเวลาถัดไปได้เลย

วิธีนี้เป็นวิธีการแบ่งการทำงานที่เรียกว่า Completely Fair Scheduler

---

## ลองดูตัวอย่างกันดีกว่า

ถ้าเรารัน nginx โดยที่

- แบ่งช่วงเวลาออกเป็นช่องละ 1 วินาที
- ให้ nginx ใช้ cpu ในแต่ละช่องได้แค่ 200ms

แสดงว่าเราให้ nginx ใช้ cpu อยู่ที่ 20% (ใช้ 200ms จาก 1s)

```sh
$ docker run --name=nginx -d --cpu-quota=200000 --cpu-period=1000000 -p 3000:80 nginx
```

ลอง load test ดู

```sh
$ hey -n 10000 http://127.0.0.1:3000

Summary:
  Total:	3.8350 secs
  Slowest:	0.7919 secs
  Fastest:	0.0005 secs
  Average:	0.0191 secs
  Requests/sec:	2607.5883

  Total data:	6120000 bytes
  Size/request:	612 bytes

Response time histogram:
  0.001 [1]	|
  0.080 [9799]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.159 [0]	|
  0.238 [0]	|
  0.317 [0]	|
  0.396 [0]	|
  0.475 [0]	|
  0.554 [0]	|
  0.634 [50]	|
  0.713 [50]	|
  0.792 [100]	|


Latency distribution:
  10% in 0.0026 secs
  25% in 0.0031 secs
  50% in 0.0039 secs
  75% in 0.0052 secs
  90% in 0.0074 secs
  95% in 0.0096 secs
  99% in 0.7717 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0000 secs, 0.0005 secs, 0.7919 secs
  DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:	0.0001 secs, 0.0000 secs, 0.0041 secs
  resp wait:	0.0184 secs, 0.0005 secs, 0.7918 secs
  resp read:	0.0005 secs, 0.0000 secs, 0.7712 secs

Status code distribution:
  [200]	10000 responses
```

ลองเปลี่ยนใหม่

- แบ่งช่วงเวลาออกเป็นช่องละ 10ms
- ให้ nginx ใช้ cpu ในแต่ละช่องได้แค่ 2ms

แสดงว่าเราให้ nginx ใช้ cpu อยู่ที่ 20% เหมือนเดิม (ใช้ 2ms จาก 10ms)

```sh
$ docker run --name=nginx -d --cpu-quota=2000 --cpu-period=10000 -p 3000:80 nginx
```

ลอง load test ดูอีกที

```sh
$ hey -n 10000 http://127.0.0.1:3000

Summary:
  Total:	4.2118 secs
  Slowest:	0.1562 secs
  Fastest:	0.0006 secs
  Average:	0.0207 secs
  Requests/sec:	2374.2984

  Total data:	6120000 bytes
  Size/request:	612 bytes

Response time histogram:
  0.001 [1]	|
  0.016 [2339]	|■■■■■■■■■■■■■
  0.032 [6990]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.047 [590]	|■■■
  0.063 [41]	|
  0.078 [7]	|
  0.094 [4]	|
  0.110 [1]	|
  0.125 [2]	|
  0.141 [3]	|
  0.156 [22]	|


Latency distribution:
  10% in 0.0084 secs
  25% in 0.0178 secs
  50% in 0.0204 secs
  75% in 0.0266 secs
  90% in 0.0303 secs
  95% in 0.0354 secs
  99% in 0.0449 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0000 secs, 0.0006 secs, 0.1562 secs
  DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0021 secs
  resp wait:	0.0204 secs, 0.0005 secs, 0.1560 secs
  resp read:	0.0002 secs, 0.0000 secs, 0.0304 secs

Status code distribution:
  [200]	10000 responses
```

เห็นความต่างอะไรไหม ?

ลองเอามาเทียบกันชัด ๆ

```text
# 200ms/1s
Latency distribution:
  10% in 0.0026 secs
  25% in 0.0031 secs
  50% in 0.0039 secs
  75% in 0.0052 secs
  90% in 0.0074 secs
  95% in 0.0096 secs
  99% in 0.7717 secs

# 2ms/10ms
Latency distribution:
  10% in 0.0084 secs
  25% in 0.0178 secs
  50% in 0.0204 secs
  75% in 0.0266 secs
  90% in 0.0303 secs
  95% in 0.0354 secs
  99% in 0.0449 secs
```

จะเห็นว่า ที่ 99th percentile (tail latency) ของ 2ms/10ms จะน้อยกว่า 200ms/1s

ทำไมถึงเป็นแบบนี้ ?

ถ้าเรามี CPU 2 cores แล้วให้ nginx รัน 1s

หมายความว่า nginx **สามารถ** ใช้ CPU ทั้ง 2 cores ได้ แต่ถ้าใช้ทั้ง 2 cores พร้อมกันจะใช้ได้แค่ 500ms จึงจะหมด quota

แสดงว่า ถ้าเราให้ nginx ใช้ CPU ที่ 200ms/1s แล้วมี load เข้ามา nginx ใช้ quota หมดแล้ว จะต้องรอจนกว่าจะถึงช่องเวลาหน้าถึงจะทำงานต่อได้
เช่น nginx ใช้ CPU หมดไปตั้งแต่ 100ms แรก nginx จะต้องรออีก 900ms ถึงจะทำงานต่อได้
ทำให้ tail latency พุ่งขึ้น

ในขณะที่เรากำหนด nginx ให้ใช้ CPU ที่ 2ms/10ms แล้ว nginx ใช้ quota หมดภายใน 1ms
nginx จะรอแค่ 9ms ถึงจะได้ทำงานต่อ

---

## สรุป

ดังนั้น ถ้าระบบเราต้องการ tail latency ต่ำ ๆ เราจะต้องแบ่งช่วงเวลาออกไปให้สั้นลง (ปกติ default อยู่ที่ 100ms) หรืออาจจะไม่สามารถ limit CPU ได้เลย ถ้าเราไม่สามารถกำหนดช่วงเวลาเองได้ (เช่นใน GKE ณ วันที่เขียนบทความ เราไม่สามารกำหนดได้)
