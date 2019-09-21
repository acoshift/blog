# ทำระบบ Realtime บน Web

ถ้าต้องทำระบบ chat, dashboard, กราฟ ที่ต้องการให้ข้อมูล update เอง จะทำยังไงดี ?

ระบบ Realtime ที่ส่วนใหญ่นิยมใช้ใน web มี

1. Short Pulling
1. Long Pulling
1. Websocket
1. Server-Sent Event (SSE)

## Short Pulling

คือการที่เราตั้งเวลาให้ไปดึงข้อมูลทุก ๆ n วินาที

### ข้อดีของ Short Pulling

- เป็นวิธีทำระบบ realtime ที่ง่ายที่สุด
- รันที่ไหนก็ได้ เพราะเป็นการเรียก HTTP ธรรมดา

### ข้อเสียของ Short Pulling

- เปลือง resource มาก เพราะถึงข้อมูลไม่เปลี่ยน ก็จะมีการส่งข้อมูลตลอดเวลา

### ตัวอย่างการทำ Short Pulling

ฝั่ง Server ไม่ต้องทำอะไรเลย (ง่ายไหม 🙈)

```go
func getDataShortPulling(w http.ResponseWriter, r *http.Request) {
    data := getData()

    w.Write(data)
}
```

ฝั่ง Browser ทำได้ง่าย ๆ 2 วิธี

1. setInterval - เขียนง่าย แต่ถ้า server ช้าจะโดนยิง

    ```js
    setInterval(fetchData, 3000) // ดึงข้อมูลทุก 3 วินาที
    ```

2. setTimeout - ยังเขียนง่ายอยู่

    ```js
    async function startPulling () {
        await fetchData() // รอให้ดึงข้อมูลเสร็จก่อน ค่อยรออีก 3 วินาที
        setTimeout(startPulling, 3000)
    }

    startPulling()
    ```

## Long Pulling

คล้าย ๆ Short Pulling แบบที่ 2 แต่ไม่ต้องมี setTimeout และให้ Server รอจนกว่าข้อมูลเปลี่ยนค่อยตอบ api กลับมา

### ข้อดีของ Long Pulling

- ประหยัด resource กว่า Short Pulling
- รันที่ไหนก็ได้ เพราะเป็นการเรียก HTTP ธรรมดา

### ข้อเสียของ Long Pulling

- มีโอกาสโดน timeout บ่อย
- อาจจะมีถ้าเขียนแบบง่าย ๆ อาจจะมีช่วงที่ไม่ได้ข้อมูลล่าสุด (api ตอบ response กลับไป แล้วมี data change ก่อนที่ client จะยิง api เข้ามาอีกรอบ)

### ตัวอย่างการทำ Long Pulling

ฝั่ง Server ต้องรอให้ Data เปลี่ยนก่อน ค่อยตอบกลับไป

```go
func getDataLongPulling(w http.ResponseWriter, r *http.Request) {
    if r.URL.Query().Get("long") != "1" {
        getDataShortPulling(w, r)
        return
    }

    <-untilNewDataReceived()

    data := getData()

    w.Write(data)
}
```

ฝั่ง Browser ยังเขียนง่ายอยู่

```js
async function startLongPulling () {
    await fetchDataLongPulling()
    startLongPulling()
}

fetchDataShortPulling() // ดึง data มาแสดงตอนเปิดเว็บก่อน
startLongPulling() // หลังจากนั้นค่อยใช้ long pulling รอ data ใหม่
```

ปัญหาของ Long Pulling แบบนี้คือ ถ้า data เปลี่ยนระหว่าง request เราจะไม่ได้ data นั้น เช่น

```
           pull  pull
Client: ----|-----|-----
Server: -----|--|--|----
             1  2  wait
```

จะเห็นว่าถ้า data เปลี่ยนจาก 1 เป็น 2 ระหว่างที่ไม่ได้ pull อยู่ client จะไม่ได้ data

วิธีแก้คือ อาจจะต้องส่ง token (เช่น timestamp) ของ data ล่าสุดไป แล้วเวลา pull ให้เอา token นั้นส่งกลับไปหา server

## Web Socket

Web Socket เป็น full-duplex communication คือ client กับ server สามารถส่งข้อมูลไปกลับได้ผ่าน connection เดียว

### ข้อดีของ Web Socket

- ประหยัด resource ไม่มี overhead ของ http, เปิด connection ทิ้งไว้ สามารถส่ง data ไปกลับได้
- มี library ครบ แทบทุกภาษา

### ข้อเสียของ Web Socket

- ทำงานบน Layer 4 (TCP) ถ้ามี middleware ที่ไม่รองรับ HTTP Upgrade ก็จะไม่สามารถใช้ได้ (เช่น Cloud Run)
- มีโอกาสโดน timeout ถ้า set middleware ไม่ดี (เช่น reverse proxy, load balancer, cdn)

## Server-Sent Event (SSE)

คล้าย ๆ Web Socket แต่ server push data หา client ได้ทางเดียว

### ข้อดีของ Server-Sent Event

- รันที่ไหนก็ได้ เพราะทำงานบน HTTP ธรรมดา
- ประหยัด resource กว่า Short/Long Pulling
- เขียนง่าย ไม่ต้องมี library

### ข้อเสียของ Server-Sent Event

- IE ใช้ไม่ได้ 🤨
- ถ้าเขียนเองต้องมีความรู้เรื่อง HTTP ในระดับนึง

### ตัวอย่างการทำ Server-Sent Event

หน้าตาของ sse event

```
: บรรทัดที่ขึ้นต้นด้วย : คือ comment

: เราสามารถกำหนดชื่อ event ได้
event: add
data: 1

event: remove
data: 1

: ถ้าไม่กำหนดชื่อ event default คือ event: message
data: hello, sse

: ถ้า data มี 2 บรรทัด
data: hello,
data: sse
```

Server เขียนไม่ยากมาก แต่ต้องรู้ว่าจะส่งข้อมูลยังไง

```go
func getDataWithSSESupport(w http.ResponseWriter, r *http.Request) {
    // ถ้าไม่ส่ง ?sse=1 มา ถือว่ายิงมาแบบ api ธรรมดา
    // จะได้ทำ api เดียว รองรับทั้ง api ธรรมดา ทั้ง sse
    if r.URL.Query().Get("sse") != "1" {
        getDataShortPulling(w, r)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache") // ไม่ให้ cache
    w.WriteHeader(http.StatusOK)

    for {
        data := getData()
        fmt.Fprintf(w, "data: %s\n\n", data)
        w.(http.Flusher).Flush()

        select {
        case <-untilNewDataReceived():
        case <-r.Context().Done():
            return
        }
    }
}
```

Browser ยิ่งง่าย

```js
const source = new EventSource('/data')

source.addEventListener('add', (ev) => {
    //
})

source.addEventListener('remove', (ev) => {
    //
})

source.addEventListener('message', (ev) => {
    //
})

// หรือจะรับ ทุก event มาเลย
source.onmessage = (ev) => {
  console.log(ev)
}
```

ลองเอาไปรันเล่น ๆ

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	http.ListenAndServe(":8080", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path == "/" {
			w.Write([]byte(`<!doctype html>
<div id=app></div>
<script>
const $app = document.getElementById('app')
const source = new EventSource('/data')
source.addEventListener('message', (ev) => {
	console.log(ev)
    app.innerHTML += ev.data + "<br>"
})
</script>`))
		}

		if r.URL.Path == "/data" {
			w.Header().Set("Content-Type", "text/event-stream")
			w.Header().Set("Cache-Control", "no-cache")
			w.WriteHeader(http.StatusOK)

			for {
				data := time.Now().Format(time.RFC3339)
				fmt.Fprintf(w, "data: %s\n\n", data)
				w.(http.Flusher).Flush()

				select {
				case <-time.After(time.Second):
				case <-r.Context().Done():
					return
				}
			}
		}
	}))
}
```

## สรุป

- ใช้ Server-Sent Event 🙈🙈🙈🙈🙈🙈

นอกจาก

- ไม่อยากแก้ code server
- คนใช้ไม่เยอะ
- data ไม่จำเป็นต้อง realtime มาก

> ให้ใช้ **Short Pulling**

### Note

- Web socket ค่อนข้างมีปัญหาเวลา deploy เพราะบางที reverse proxy ไม่ support http upgrade
แต่ library บางตัวจะ fallback ไปใช้ short/long pulling มีโอกาสที่ server จะโดน request ยิงรัว ๆ
ยิ่งถ้าใช้ service ที่คิดตังตามจำนวน request ก็จะโดนค่าตรงนี้เยอะมาก อาจจะหลายล้าน request ต่อวัน

- ถ้าจะ support IE ค่อยใช้ Long Pulling ถ้าไม่ support IE ไปใช้ SSE ดีกว่า

- หรือจะทำ Short Pulling ไปก่อนก็ได้ แล้วคนใช้เยอะค่อยให้ api เดิม support SSE แก้ code เพิ่มไม่เยอะ
