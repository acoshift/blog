# ดูการใช้งาน Connections ใน http.Client ใน Go

บางทีเราอยากจะรู้ว่าตอนนี้ `http.Client` มี idle connection อยู่ใน pool เท่าไร
เขียน หรืออ่าน ไปกี่ bytes แล้ว

เราสามารถ track connection ได้ง่าย ๆ แค่

1. สร้างตัวแปรที่จะเก็บสิ่งที่เราอยากจะ track (ในตัวอย่างขอเก็บไว้ในตัวแปร global ละกัน)

    ```go
    var (
        currentConns int64
        totalWrite   int64
        totalRead    int64
    )
    ```

1. เพิ่มคำสั่ง print stats มาดูหน่อย

    ```go
    func printStats() {
        currentConns := atomic.LoadInt64(&currentConns)
        totalWrite := atomic.LoadInt64(&totalWrite)
        totalRead := atomic.LoadInt64(&totalRead)

        fmt.Printf("connections: %d\nwrite: %d bytes\nread: %d bytes\n", currentConns, totalWrite, totalRead)
    }
    ```

1. เขียน struct มาครอบ `net.Conn`

    ```go
    type trackConn struct {
        net.Conn
        closed int32
    }
    ```

1. เวลาสร้าง `trackConn` ใหม่ ให้เพิ่ม `currentConns`

    ```go
    func newTrackConn(conn net.Conn) *trackConn {
        atomic.AddInt64(&currentConns, 1)
        return &trackConn{Conn: conn}
    }
    ```

1. ถ้า `trackConn` ถูก `Close` ให้ลด `currentConns`

    สิ่งที่ต้องระวังคือ ถ้าเรียก `Close` มากกว่า 1 ครั้ง จะต้องลด `currentConns` แค่ 1 เท่านั้น

    ```go
    func (conn *trackConn) Close() error {
        if !atomic.CompareAndSwapInt32(&conn.closed, 0, 1) {
            return nil
        }
        atomic.AddInt64(&currentConns, -1)
        return conn.Conn.Close()
    }
    ```

1. ถ้า `trackConn` ถูก `Read` ให้เก็บว่าอ่านไปกี่ bytes

    ```go
    func (conn *trackConn) Read(b []byte) (n int, err error) {
        n, err = conn.Conn.Read(b)
        atomic.AddInt64(&totalRead, int64(n))
        return
    }
    ```

1. เช่นเดียวกัน ถ้า `trackConn` ถูก `Write` ให้เก็บว่าเขียนไปกี่ bytes

    ```go
    func (conn *trackConn) Write(b []byte) (n int, err error) {
        n, err = conn.Conn.Write(b)
        atomic.AddInt64(&totalWrite, int64(n))
        return
    }
    ```

1. ตอนเรียกใช้ เวลาที่ dial ให้เอา `trackConn` ของเราไปครอบ `net.Conn`

    ```go
    dialer := &net.Dialer{}

    client := &http.Client{
        Transport: &http.Transport{
            // DisableKeepAlives: true,
            DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
                conn, err := dialer.DialContext(ctx, network, addr)
                if err != nil {
                    return nil, err
                }
                return newTrackConn(conn), nil
            },
        },
    }
    ```

1. เวลา get ลอง print stats ออกมาดูด้วย

    ```go
    getURLAndPrintStats := func(url string) {
        resp, err := client.Get(url)
        if err != nil {
            log.Println(err)
            return
        }
        io.Copy(ioutil.Discard, resp.Body)
        resp.Body.Close()

        fmt.Println("get:", url)
        printStats()
    }
    ```

1. ลอง get หลาย ๆ เว็บมาดู

    ```go
    getURLAndPrintStats("https://www.google.com")
    getURLAndPrintStats("https://www.facebook.com")
    getURLAndPrintStats("https://www.youtube.com")
    getURLAndPrintStats("https://github.com")
    getURLAndPrintStats("https://www.pixiv.net")
    getURLAndPrintStats("https://www.google.com")
    getURLAndPrintStats("https://github.com")
    getURLAndPrintStats("https://www.pixiv.net")
    getURLAndPrintStats("https://www.google.com")
    ```

    ```sh
    $ go run main.go
    get: https://www.google.com
    connections: 1
    write: 732 bytes
    read: 9596 bytes
    get: https://www.facebook.com
    connections: 2
    write: 1554 bytes
    read: 49719 bytes
    get: https://www.youtube.com
    connections: 3
    write: 2815 bytes
    read: 110209 bytes
    get: https://github.com
    connections: 4
    write: 3260 bytes
    read: 139970 bytes
    get: https://www.pixiv.net
    connections: 5
    write: 3867 bytes
    read: 155766 bytes
    get: https://www.google.com
    connections: 5
    write: 4063 bytes
    read: 162148 bytes
    get: https://github.com
    connections: 5
    write: 4229 bytes
    read: 188395 bytes
    get: https://www.pixiv.net
    connections: 5
    write: 4315 bytes
    read: 199187 bytes
    get: https://www.google.com
    connections: 5
    write: 4557 bytes
    read: 205815 bytes
    ```

1. ลองปิด Keep Alive แล้วรันอีกที

    ```go
    client := &http.Client{
        Transport: &http.Transport{
            DisableKeepAlives: true, // เอา comment ตรงนี้ออก
            DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
                conn, err := dialer.DialContext(ctx, network, addr)
                if err != nil {
                    return nil, err
                }
                return newTrackConn(conn), nil
            },
        },
    }
    ```

    ```sh
    $ go run main.go
    get: https://www.google.com
    connections: 0
    write: 675 bytes
    read: 9386 bytes
    get: https://www.facebook.com
    connections: 0
    write: 1318 bytes
    read: 48831 bytes
    get: https://www.youtube.com
    connections: 0
    write: 2564 bytes
    read: 109332 bytes
    get: https://github.com
    connections: 0
    write: 3059 bytes
    read: 139196 bytes
    get: https://www.pixiv.net
    connections: 0
    write: 3655 bytes
    read: 154994 bytes
    get: https://www.google.com
    connections: 0
    write: 4330 bytes
    read: 164557 bytes
    get: https://github.com
    connections: 0
    write: 4825 bytes
    read: 194327 bytes
    get: https://www.pixiv.net
    connections: 0
    write: 5463 bytes
    read: 210157 bytes
    get: https://www.google.com
    connections: 0
    write: 6138 bytes
    read: 219647 bytes
    ```

    จะเห็นว่า connection จะเป็น 0 ตลอด เพราะพอ get เสร็จก็จะปิด connection ทันที

## Source Code เต็ม ๆ

```go
package main

import (
    "context"
    "fmt"
    "io"
    "io/ioutil"
    "log"
    "net"
    "net/http"
    "sync/atomic"
)

func main() {
    dialer := &net.Dialer{}

    client := &http.Client{
        Transport: &http.Transport{
            // DisableKeepAlives: true,
            DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
                conn, err := dialer.DialContext(ctx, network, addr)
                if err != nil {
                    return nil, err
                }
                return newTrackConn(conn), nil
            },
        },
    }

    getURLAndPrintStats := func(url string) {
        resp, err := client.Get(url)
        if err != nil {
            log.Println(err)
            return
        }
        io.Copy(ioutil.Discard, resp.Body)
        resp.Body.Close()

        fmt.Println("get:", url)
        printStats()
    }

    getURLAndPrintStats("https://www.google.com")
    getURLAndPrintStats("https://www.facebook.com")
    getURLAndPrintStats("https://www.youtube.com")
    getURLAndPrintStats("https://github.com")
    getURLAndPrintStats("https://www.pixiv.net")
    getURLAndPrintStats("https://www.google.com")
    getURLAndPrintStats("https://github.com")
    getURLAndPrintStats("https://www.pixiv.net")
    getURLAndPrintStats("https://www.google.com")
}

var (
    currentConns int64
    totalWrite   int64
    totalRead    int64
)

func printStats() {
    currentConns := atomic.LoadInt64(&currentConns)
    totalWrite := atomic.LoadInt64(&totalWrite)
    totalRead := atomic.LoadInt64(&totalRead)

    fmt.Printf("connections: %d\nwrite: %d bytes\nread: %d bytes\n", currentConns, totalWrite, totalRead)
}

type trackConn struct {
    net.Conn
    closed int32
}

func newTrackConn(conn net.Conn) *trackConn {
    atomic.AddInt64(&currentConns, 1)
    return &trackConn{Conn: conn}
}

func (conn *trackConn) Read(b []byte) (n int, err error) {
    n, err = conn.Conn.Read(b)
    atomic.AddInt64(&totalRead, int64(n))
    return
}

func (conn *trackConn) Write(b []byte) (n int, err error) {
    n, err = conn.Conn.Write(b)
    atomic.AddInt64(&totalWrite, int64(n))
    return
}

func (conn *trackConn) Close() error {
    if !atomic.CompareAndSwapInt32(&conn.closed, 0, 1) {
        return nil
    }
    atomic.AddInt64(&currentConns, -1)
    return conn.Conn.Close()
}

```
