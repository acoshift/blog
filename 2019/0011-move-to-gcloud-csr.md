# ย้ายมาใช้ Cloud Source Repository

ต้นเดือนหน้า (6 กันยา) Github Organization ที่สมัครไว้จะต้องต่อ subscription อีก 1 ปี
พอมาดูราคาแล้วก็ค่อนข้างแพงเหมือนกัน เพราะจริง ๆ ในทีมมี developer กันแค่ 4 คน
ที่เหลือจะเป็น part-time กับ stakeholder ซะส่วนใหญ่ ทำให้ค่าใช้จ่ายสูงกว่าที่ใช้จริงมาก ๆ

## เลยมีความคิดว่าจะย้ายไปไหนดี ?

เนื่องจากว่าตอนนี้ใช้ Google Cloud Build ทำ CI/CD อยู่
ทำให้ตัด choice ไปได้เยอะ เพราะ Cloud Build support git แค่ 3 ตัวคือ

1. Github
1. Bitbucket
1. Cloud Source Repository (CSR)

| Product | Start Price | Free User | Price per User |
| --- | --- | --- | --- |
| Github | $25 | 5 | $5 |
| Bitbucket | $10 | 5 | $2 |
| CSR | $0 | 5 | $1 |

พอมาเปรียบเทียบ Pricing กันแล้วจะเห็นว่า Bitbucket กับ CSR ก็คิดราคา per-user เหมือนกัน

แต่ CSR คิดแค่ active user แสดงว่าถ้าเรามี 10 users
แต่มีแค่ 8 คนที่ active เราจะเสียแค่ $3 เท่านั้นเอง

แต่สิ่งที่ต้องมาดูเพิ่มคือ CSR คือค่า Storage กับ Egress ด้วย ซึ่งเจ้าอื่นจะคิดเป็น Git LFS แทน

| Product | Free Storage/Bandwidth | Price Storage/Bandwidth | Remark |
| --- | --- | --- | --- |
| Github | 1GB/12GB per year | $60 50GB/600GB per year | - |
| Bitbucket | 5GB | $10 100GB per month | ไม่ได้บอกว่ามีค่า Bandwidth |
| CSR | 50GB/50GB | $0.1/$0.1 per GB per month | Storage เป็น pro-rated, b/w คิดเฉพาะค่า Egress |

เนื่องจากว่าส่วนใหญ่แล้วเราไม่น่าจะได้ใช้ git lfs หรอก เฉพาะฉะนั้นตัดของ Github กับ Bitbucket ออกไปได้เลย

CSR ให้ Storage กับ B/W มาฟรีอย่างละ 50GB แต่เราคิดว่าเราจะเก็บไฟล์ 50GB ไม่ได้ เพราะ git จะเก็บทุก commit
แสดงว่า ถ้าเรา push ไฟล์ 1MB ที่ไม่เหมือนกันเลย เข้าไป 100 รอบ เป็นไฟล์ใหม่ทับไฟล์เก่า ถึงใน repo ของเราจะมีแค่ไฟล์ 1MB แต่จริง ๆ แล้วเราใช้พื้นที่ไป 100MB

แต่ข้อดีของ CSR อย่างนึงคือ storage คิดเป็น pro-rated หมายความว่าถ้าเราใช้พื้นที่ 50GB 29 วัน
แล้ววันที่ 30 เราใช้เกิน 1 GB เราจะเสียเงินน้อยมาก ๆ ($0.1/30 = $0.0033)
ไม่เหมือนกับ Github หรือ Bitbucket ที่เราต้องซื้อพื้นที่มาก่อน จะใช้หรือไม่ใช้เรื่องของเรา

## ข้อเสียอีกอย่างนึงของ CSR คือไม่มี issue และ pull request

CSR เก็บ private git repo ได้อย่างเดียว ไม่มี feature ที่สำคัญอย่าง issue และ pull request

### Issue เป็น feature ที่ทางทีมใช้บ่อยมาก ๆ

ใช้ในการ track งานเลยก็ว่าได้ โดยปกติแล้วทีมจะใช้ Github ทำงานอย่างเดียว ไม่ได้ใช้ tool อื่น ๆ การที่ไม่มี Github หมายความว่าเราจะต้องไปซื้อ tool อื่น ๆ ที่คิดเป็น per-user เหมือนกัน (หนีเสือปะจระเข้)

ตอนนี้ที่ดู ๆ ไว้ก็คือ

1. Github public repo
1. Trello แบบฟรี
1. Google Groups

ส่วนตัวค่อนข้างชอบ Google Groups เพราะสามารถทำงานผ่าน email ได้เลย แต่ทีมไม่ชอบ เพราะมันทำให้ inbox รก ซึ่งก็ต้องใช้ skill ในการจัดการ inbox ของใครของมัน (เช่นการใช้ filter, label) และเว็บ Google Groups ค่อนข้างช้า (ระบบเก่ามากแล้ว)

ส่วน Github public repo ไม่ค่อยเห็นด้วยเพราะทุกคนใน Internet สามารถเข้ามาดูได้

ทางที่น่าจะไปมากที่สุดก็คือ Trello แบบฟรี

### ไม่มี Pull Request (PR)

Pull Request ก็เป็นสิ่งที่ใช้บ่อยมาก ๆ อีก feature นึงของ Github พอมาใช้ CSR แล้วไม่มี PR จะทำยังไง ?

แน่นอนว่าทุกอย่างจะต้องจัดการด้วย Git Client ทั้งหมด แต่เนื่องจากปกติใช้ git command line กันอยู่แล้วเลยไม่มีปัญหา

### ปัญหาคือจะ review code ยังไง ?

พอมาเจอปัญหานี้เลยลองย้อนกลับไปดูว่า พอเรามา review code แล้วเกิดอะไรขึ้นบ้าง

- งานช้าลง เพราะ bottleneck ที่คน review บางทีไม่สามารถ review ทันทีได้
- พอไม่ได้ review ทันทีทำให้เกิด conflict ต้องเสียเวลามาแก้ conflict
- พอ conflict มาก ๆ บางทีก็ทิ้ง branch นั้นแล้วทำใหม่

จะเห็นว่าการที่คน review code ไม่พร้อมที่จะ review ได้ทันที (ภายใน 1-3 นาทีเมื่อเปิด PR)
ก็อาจจะทำให้ progress ช้าลงกว่าที่ควรจะเป็น

ตอนนี้เลยกลับมาใช้วิธีบ้าน ๆ แบบเดิม ก่อนที่จะมีการ review code ก็คือ
ให้ทุกคน push เข้า master ไปเลย~~~~~~

เนื่องจากว่าเรามี CI ที่คอยรัน lint, test, build อยู่แล้ว เพราะฉะนั้นถ้ามี error เกิดขึ้นจะรู้ได้ทันที
หรือคนที่ pull มาก็จะเจอ error คนที่ pull ก็จะมาแก้ error ทำให้ branch master จะไม่มี error ค้างนาน

ดังนั้นแทนที่จะเป็นการ code => review => merge กลายมาเป็น code => refactor แทน

[https://trunkbaseddevelopment.com/committing-straight-to-the-trunk/](https://trunkbaseddevelopment.com/committing-straight-to-the-trunk/)

### ไม่มี protection branch

จริง ๆ ตอนที่มี review code ค่อนข้างสำคัญ เพราะเราจะไม่ให้ push เข้า master ตรง ๆ ต้องทำเป็น PR เท่านั้น

แต่ที่สำคัญจริง ๆ คือกันไม่ให้ force push เข้ามาที่ master ซึ่งตอนนี้ไม่รู้ว่า CSR จะทำได้ไหม น่าจะไม่ได้

## OK จะใช้ CSR ละ ต้องทำไง ?

เนื่องจากว่าส่วนใหญ่ทีมใช้ ssh กับ Github อยู่แล้ว เลยย้ายมาใช้ CSR ค่อนข้างง่าย

แต่ถ้าใครที่ไม่เคยใช้ ssh key มาก่อนจะลำบากหน่อย ๆ เพราะเราไม่สามารถใช้ username, password กับ CSR ได้

จริง ๆ แล้ว CSR มีวิธีใข้ง่ายอยู่ 3 แบบ

1. SSH
1. Cloud SDK
1. Manually generated credentials

สามารถดูวิธี setup ได้[ที่นี่](https://cloud.google.com/source-repositories/docs/authentication)

## อยากได้ push noti ?

ตอนเราใช้ Github เราสามารถ set webhook ได้ง่าย ๆ ปกติทีมใช้ Discord กันอยู่
เราสามารถสร้าง bot ใน Discord แล้วเอา url bot ไปแปะลง github ได้เลย แค่เติม `/github` ต่อท้าย url ของ bot

แต่พอมาใช้ CSR แล้วเราทำแบบนั้นไม่ได้ แต่ [CSR สามารถ push event](https://cloud.google.com/source-repositories/docs/quickstart-adding-pubsub-notifications) เข้า Cloud PubSub ได้

เราก็แค่ทำให้ PubSub ส่งไปหา Cloud Run แล้วรันตัวที่รับ event ของ CSR มาแปลงเป็น message ของ Slack แล้วส่งเข้า Discord ก็สิ้นเรื่อง [github.com/moonrhythm/cloudreposlackhook](https://github.com/moonrhythm/cloudreposlackhook)

## พอย้ายมา CSR แล้วมีข้อดีอะไรบ้าง

1. เร็ว เร็ว เร็ว

    ลอง clone project เดียวกัน ระหว่าง Github กับ CSR

    มาดูของ Github กันก่อน

    ```sh
    $ time git clone git@github.com:___/___.git
    Cloning into '___'...
    remote: Enumerating objects: 66, done.
    remote: Counting objects: 100% (66/66), done.
    remote: Compressing objects: 100% (45/45), done.
    remote: Total 62212 (delta 27), reused 37 (delta 18), pack-reused 62146
    Receiving objects: 100% (62212/62212), 73.94 MiB | 4.83 MiB/s, done.
    Resolving deltas: 100% (35220/35220), done.

    real	0m20.429s
    user	0m2.342s
    sys	0m0.683s
    ```

    มาดูของ CSR กัน

    ```sh
    $ time git clone ssh://___@source.developers.google.com:2022/p/___/r/___
    Cloning into '___'...
    remote: Sending approximately 72.94 MiB ...
    remote: Total 61847 (delta 39962), reused 61847 (delta 39962)
    Receiving objects: 100% (61847/61847), 72.94 MiB | 45.32 MiB/s, done.
    Resolving deltas: 100% (39962/39962), done.

    real	0m3.306s
    user	0m2.251s
    sys	0m0.640s
    ```

    จะเห็นว่าของ Github ใช้เวลา 20 วินาที ส่วน CSR ใช้แค่ 3 วินาทีเท่านั้น

1. เปิดดู audit logs ได้ เราสามารถดูได้ว่าใครทำอะไร ตอนไหน แต่จริง ๆ ก็ไม่ได้เปิดหรอก xD

1. ระบบ search ค่อนข้างดี ปกติเวลา search ใน Github บางทีก็เจอ บางทีก็ไม่เจอ แต่พอมา search ผ่าน CSR มันรู้หมดว่าอันไหนคือ struct อันไหนคือตัวแปร

---

ไว้ลองใช้สักเดือนดูก่อน ถ้าไม่ work ก็อาจจะย้ายอีกที :P
