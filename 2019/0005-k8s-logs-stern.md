# ดู Logs จากหลาย ๆ Pods ใน Kubernetes ด้วย stern

ปกติเวลาเราจะดู logs ของ pod ที่อยู่ใน Kubernetes เราจะต้องหาก่อนว่า pod นั้นชื่ออะไร

เช่นเราอยากดู logs ของ pod ที่อยู่ใน deployment `parapet-ingress-controller`

เราจะต้อง get pods มาดูว่ามี pod อะไรบ้าง

```sh
$ kubectl get po
NAME                                          READY   STATUS    RESTARTS   AGE
parapet-ingress-controller-5bd87c7856-55cdp   1/1     Running   0          2d18h
parapet-ingress-controller-5bd87c7856-fvl7x   1/1     Running   0          2d18h
```

เราต้องเลือกว่าจะดู logs ของ pod อันไหน

```sh
$ kubectl logs parapet-ingress-controller-5bd87c7856-55cdp -f --tail=10
```

> -f คือให้ follow log ใหม่ที่จะออกมาด้วย
>
> --tail=10 คือให้ดึง log 10 บรรทัดล่าสุดออกมา

ใน Kubernetes version หลัง ๆ เราสามารถดู logs ผ่าน deployment ได้เลย

```sh
$ kubectl logs deploy/parapet-ingress-controller --tail=10
Found 2 pods, using pod/parapet-ingress-controller-5bd87c7856-55cdp

...
```

โดย Kubernetes จะสุ่มเลือก pod มาให้เรา ไม่ได้เอา logs ของทั้ง 2 pods มารวมกัน

## stern

เราสามารถดู logs ของ pods ทั้งหมดพร้อมกันได้ด้วย [wercker/stern](https://github.com/wercker/stern)

สำหรับคนใช้ macOS สามารถลงได้อย่างง่าย ๆ ด้วย

```sh
$ brew install stern
```

## วิธีใช้

stern รับชื่อ pod เป็น regex เราสามารถใส่ส่วนนึงของชื่อ pod ลงไปได้เลย เช่น

```sh
$ stern ingress --tail=10
```

Pod ไหนที่มีคำว่า ingress อยู่ในชื่อ pod จะถูกดึง logs ออกมาแสดง 10 บรรทัดล่าสุดของแต่ละ pod และจะ follow log ให้ด้วย

## ปัญหาที่เจอ

- ปกติ stern เวลามี pod ใหม่ หรือ pod เก่าถูกลบออก มันจะเพิ่ม/ลด pod ที่ดึง logs ให้เอง

    แต่เหมือนถ้า connection ที่มันใช้ watch pod ถูก disconnect มันจะไม่รู้ว่า pod ถูกเพิ่ม/ลด ทำให้มันไม่รู้ว่ามี pod ใหม่เข้ามา
    ทำให้บางทีเราต้อง kill stern ทิ้ง แล้วเปิดใหม่

- Project stern ไม่ได้ commit มานานแล้ว (วันที่เขียนบทความ commit ล่าสุดวันที่ 17/10/2018)
