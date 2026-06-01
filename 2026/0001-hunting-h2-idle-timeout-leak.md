# ตามล่า Memory Leak ที่ไม่ได้อยู่ใน Code เรา

> บทความนี้มาเล่าว่า reverse proxy ตัวนึงของผม memory ค่อย ๆ ไต่ขึ้นจน OOM แล้วผมหาอยู่นานกว่าจะเจอ เลยลองหยิบ heap profiler มาช่วยดู สุดท้ายปรากฏว่ามันไม่ได้อยู่ใน code เรา

## เล่าที่มา

ช่วงหลังผมเขียน reverse proxy ตัวใหม่ด้วย Rust (อยู่บน [pingora](https://github.com/cloudflare/pingora)) แล้วเอาขึ้น production

อยู่มาวันนึงก็สังเกตว่า RSS ของ pod มันค่อย ๆ ไต่ขึ้นเรื่อย ๆ ไม่ค่อยยอมลง ไต่ไปประมาณ **2 GB ต่อชั่วโมง** สุดท้ายก็ OOM แล้วโดน kill

วิธีแก้เฉพาะหน้าตอนนั้นก็คือ... ตั้ง cronjob roll pod ทิ้งเป็นระยะไปก่อน 😅 ซึ่งก็คงไม่ใช่วิธีแก้ที่ถูกต้องสักเท่าไร

## หาเท่าไรก็ไม่เจอ

สิ่งที่ลองทำไปแล้ว

1. ไล่อ่าน code ตัวเองใหม่หมด หา `Vec` หรือ `HashMap` ที่ลืม clear, หา task ที่ลืม join — ไม่เจออะไรเลย
1. ลอง reproduce ใน local แบบ application-level (ยิง request เยอะ ๆ, ยิง request ใหญ่ ๆ) — memory ก็ขึ้นแล้วก็ลงปกติ
1. เปิด jemalloc มาดู stats — เห็นว่า memory ขึ้นจริง แต่ก็ยังไม่รู้ว่าใครกิน
1. ไปไล่แก้เรื่อง connection pool ที่สงสัยว่าจะ leak — แก้แล้วก็ยังไม่หาย

จุดที่ทำให้ผมพอจะมั่นใจว่ามันน่าจะเป็น leak จริง ๆ ไม่ใช่แค่ allocator ไม่คืน memory ให้ OS ก็คือ ค่า `jemalloc_allocated_bytes` (live heap ที่ยังถูกใช้อยู่) มันไต่ขึ้นเรื่อย ๆ ตามกันไปด้วย

> ถ้าเป็นแค่ allocator ไม่คืน OS ค่า RSS จะขึ้น แต่ `allocated_bytes` น่าจะนิ่ง — แต่อันนี้มันขึ้นทั้งคู่ ก็เลยเดาว่าน่าจะมีคนถือ memory ไว้แล้วไม่ปล่อย

## งัด Heap Profiler

พอหาด้วยตาไม่เจอ ก็ต้องให้เครื่องมือมันบอกเราว่าใครเป็นคน allocate memory ที่ยังถือค้างอยู่

jemalloc มันมี heap profiling มาให้อยู่แล้ว เปิดได้ตอน runtime แล้วใช้ `jeprof` ดู

trick ที่ผมใช้คือ `jeprof --base` เอา profile ของ pod ที่กำลัง leak มาลบกัน 2 ช่วงเวลาห่างกันไม่กี่นาที — ผลที่ได้คือเฉพาะส่วนที่ **โตขึ้น** ในช่วงนั้น ซึ่งก็น่าจะเป็นตัว leak นั่นเอง

ผลออกมาก็ค่อนข้างชัด

| สัดส่วน | frame |
|---|---|
| 54% | `pingora_timeout::Timeout<T,F>::poll` |
| 12% | `pingora_core::protocols::http::v2::server::handshake` |
| ~4% | `bytes::BytesMut::reserve_inner` (อยู่ใต้ 2 อันบน) |

อ่านแล้วแปลได้ว่า → **state ของ downstream HTTP/2 connection มันสะสมขึ้นเรื่อย ๆ** ทั้ง read buffer (`BytesMut`) และ timeout future ของแต่ละ connection

ที่น่าสนใจคือ มันอยู่ใน `pingora_*` แทบทั้งหมด ไม่ใช่ code เรา 🤯 มิน่าหาอยู่ตั้งนานถึงไม่เจอ

## Root Cause

พอรู้ว่าอยู่ใน HTTP/2 serve loop ของ pingora ก็ไปเปิด code มันอ่าน

ปรากฏว่า downstream H2 serve loop ของ pingora 0.8 (`apps/mod.rs`) มัน **ยังไม่มี idle timeout** — มี comment ของมันเองเขียนไว้ว่า `// TODO: add a timeout?`

```rust
loop {
    // รอ stream ถัดไปจาก connection เดิม... ตลอดกาล
    let h2_stream = server::HttpSession::from_h2_conn(&mut h2_conn, digest.clone()).await;
    // ...
}
```

ปัญหาคือ พอ client ทำ H2 handshake เสร็จแล้วเงียบ (ไม่ส่ง stream อะไรมาต่อ) หรือ TCP มัน half-open ไป (หลุดโดยไม่ส่ง RST มาบอก) — loop นี้ก็จะไป block อยู่ที่ `from_h2_conn` รอ stream ถัดไปไปเรื่อย ๆ โดยไม่มีกำหนด

แล้วก็ไม่มีอะไรไปปิด connection นั้นให้ ทำให้ทุกอย่างที่ผูกกับ connection นั้น — h2 state, read buffer, timeout future ของแต่ละ read — ค้างอยู่ใน heap ไปเรื่อย ๆ

แล้วทำไมมันถึงสะสมเยอะขนาดนั้น ?

ก็เพราะ access log ของผมมันเต็มไปด้วย **scanner** ที่ยิงตรงเข้ามา — random SNI บ้าง, `connection closed before reading preface` บ้าง, ยิงตรงเข้า IP โดยไม่ผ่าน Cloudflare บ้าง (อันนี้เคยเล่าไปแล้วใน [0003 - อย่าเปิด Reverse Proxy เองถ้าไม่มีระบบป้องกัน DDoS](../2022/0003-no-reverse-proxy-ddos.md))

พวก scanner เหล่านี้แหละที่ต่อ H2 เข้ามาแล้วทิ้งไว้เฉย ๆ connection ละนิดละหน่อย แต่พอสะสมเยอะ ๆ เข้าก็เลยกลายเป็น leak ที่ค่อย ๆ โตไปเรื่อย ๆ

> นี่ก็น่าจะเป็นเหตุผลว่าทำไม reproduce แบบ application-level ถึงไม่เจอ — เพราะมันไม่ได้เกี่ยวกับ request แต่ไปเกี่ยวกับ **connection** ที่ค้างไว้เฉย ๆ แล้วก็อยู่ลึกใน library ไม่ใช่ code เรา

## วิธีแก้

เนื่องจากมันอยู่ใน pingora แต่ผมยังไม่อยากรอ upstream แก้ ก็เลย vendor `pingora-core 0.8.0` มาแล้ว patch ผ่าน `[patch.crates-io]` เอา โดยแก้แค่ `apps/mod.rs` ไฟล์เดียว

หลักการแก้คือ เพิ่ม idle timeout ให้ serve loop แต่มีเงื่อนไขนิดนึง — **connection ที่กำลังมี stream แบบ long-lived อยู่ (เช่น SSE, long-poll, WebSocket-over-h2) ต้องไม่โดนปิด**

เพราะฉะนั้นแค่ตั้ง timeout เฉย ๆ ไม่ได้ ต้องนับด้วยว่ายังมี stream ที่ทำงานอยู่ไหม

1. นับ in-flight stream ด้วย atomic counter แล้วใช้ drop guard เคลียร์ตอน task จบ

    ```rust
    let active = Arc::new(AtomicUsize::new(0));
    // ...ตอน spawn stream ใหม่:
    active.fetch_add(1, Ordering::Relaxed);
    pingora_runtime::current_handle().spawn(async move {
        struct ActiveGuard(Arc<AtomicUsize>);
        impl Drop for ActiveGuard {
            fn drop(&mut self) {
                self.0.fetch_sub(1, Ordering::Relaxed);
            }
        }
        let _active = ActiveGuard(active); // เคลียร์เองตอน task จบ
        app.process_new_http(ServerSession::new_http2(h2_stream), &shutdown).await;
    });
    ```

1. เพิ่ม branch idle timeout เข้าไปใน serve loop ด้วย `tokio::select!`

    ```rust
    let h2_stream = tokio::select! {
        biased; // ถ้ามี stream พร้อมเสิร์ฟ ให้เสิร์ฟก่อนเสมอ
        h2_stream = server::HttpSession::from_h2_conn(&mut h2_conn, digest.clone()) => h2_stream,
        _ = shutdown.changed() => { /* graceful shutdown */ }
        _ = pingora_timeout::fast_timeout::fast_sleep(h2_idle_timeout()) => {
            // ไม่มี stream ใหม่เข้ามาเลยในช่วง idle window
            // ถ้าไม่มี stream ที่ active อยู่ด้วย แปลว่า connection นี้ idle/half-open จริง ๆ
            // return None เพื่อ drop connection ทิ้ง (ปิด socket ทันที)
            if active.load(Ordering::Relaxed) == 0 {
                return None;
            }
            continue; // ยังมี stream ทำงานอยู่ ก็เสิร์ฟต่อไป
        }
    };
    ```

จุดที่ตั้งใจคือ ตอน timeout แล้วจะ drop connection ทิ้งไปเลย ไม่รอ graceful GOAWAY — เพราะถ้า peer มันตายไปแล้ว (half-open) การรอ GOAWAY ก็อาจจะไปค้างอีก กลายเป็นไม่ได้ช่วยอะไร

ค่า default ตั้งไว้ที่ **60 วินาที** (ให้ตรงกับ H1 keepalive default) แล้วเปิดให้ override ผ่าน env `PINGORA_H2_IDLE_TIMEOUT_SECS`

## ลองของ

verify บน build ที่ patch แล้ว

- **idle H2 connection** → โดนปิดตรงตาม timeout พอดี (ตั้ง `PINGORA_H2_IDLE_TIMEOUT_SECS=3` แล้วมันปิดที่ราว ๆ 3.03 วินาที)
- **SSE stream ที่ถือค้างไว้** → อยู่รอดตลอด ไม่โดนปิดตาม idle timeout (ไม่ regression)
- test ทั้ง 105 ตัวก็ยังผ่านหมด

พอ deploy จริง ค่า `jemalloc_allocated_bytes` ก็ดูแบนราบลง ไม่ค่อยไต่ขึ้นอีก 🎉

## สรุป

- leak ไม่จำเป็นต้องอยู่ใน code เราเสมอไป บางทีมันก็อยู่ใน library/dependency ที่เราใช้ — ไล่อ่าน code ตัวเองอย่างเดียวบางทีก็ยังไม่พอ
- ถ้าหา leak ด้วยตาไม่ค่อยเจอ ลองหยิบ **heap profiler** มาช่วยดู (jemalloc + `jeprof --base` เทียบ 2 ช่วงเวลา) มันพอจะชี้ตัวคนกิน memory ให้ได้ ไม่ต้องเดาเอาเอง
- connection ที่รับตรงจาก internet ก็ควรจะมี idle timeout เผื่อไว้ ไม่งั้นพวก scanner ที่ต่อแล้วทิ้งไว้เฉย ๆ ก็อาจจะค่อย ๆ สะสมจนกลายเป็น leak ได้
- เรื่อง idle/half-open connection กับการกิน memory นี่ก็เป็นธีมเดิม ๆ ที่เคยเล่าไปหลายรอบแล้ว ([0001](../2022/0001-concurrent-queue-ratelimit.md), [0002](../2022/0002-max-http-header.md), [0003](../2022/0003-no-reverse-proxy-ddos.md)) — เปลี่ยนภาษาจาก Go มาเป็น Rust ก็ดูเหมือนจะยังหนีไม่พ้น 😆

---

_Written by Claude_
