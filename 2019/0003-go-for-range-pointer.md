# ดึง Pointer ใน for range ใน Go ต้องระวัง

มีหลายครั้งที่ function อาจจะส่ง slice ของ struct ออกมา แล้วเราต้องการที่จะ index สิ่งนั้นลง map

```go
package main

import "fmt"

type A struct {
    Name  string
    Value int
}

func listA() []A {
    return []A{
        {"A", 1},
        {"B", 2},
        {"C", 3},
    }
}

func main() {
    list := listA()
    nameToA := make(map[string]*A)
    for _, a := range list {
        nameToA[a.Name] = &a
    }

    fmt.Println("A:", nameToA["A"].Value)
    fmt.Println("B:", nameToA["B"].Value)
    fmt.Println("C:", nameToA["C"].Value)
}
```

ดู code ข้างบนดี ๆ ก็เหมือนจะไม่มีอะไร แต่พอลองรันเท่านั้นแหละ !!!

```sh
$ go run main.go
A: 3
B: 3
C: 3
```

ทำไมถึงเป็นแบบนี้ ?

เพราะว่า `for _, a := range list` คือการ copy ค่าใน list ออกมาใส่ตัวแปร a เพราะว่า list เก็บเป็น slice ของ value (แน่นอนว่าถ้า list เป็น slice ของ pointer จะไม่มีปัญหานี้เกิดขึ้น)

พอวน loop รอบถัดไป ตัวแปร `a` ก็จะถูก replace ด้วยค่าต่อไปใน list ทำให้ค่าของตัวแปร `a` เปลี่ยน แต่ address ของตัวแปร `a` อยู่ที่เดิม

การที่เราเขียน `nameToA[a.Name] = &a` หมายถึงให้เก็บ addess ของตัวแปร `a`

เราสามารถแก้ปัญหานี้ได้ 3 วิธี คือ

1. for range index แทน

    ```go
    for i := range list {
        nameToA[list[i].Name] = &list[i]
    }
    ```

1. copy ค่าออกมาใส่ในตัวแปรที่อยู่ใน scope ของ for

    ```go
    for _, a := range list {
        a := a
        nameToA[a.Name] = &a
    }
    ```

    การเขียน `a := a` คือการที่เรา copy ค่าเข้ามาในตัวแปรใหม่ ที่ชื่อซ้ำกับตัวแปรเดิม (shadow variable)

    ถ้าเราไม่อยาก shadow variable สามารถตั้งเป็นชื่ออื่นได้

    ```go
    for _, a := range list {
        b := a
        nameToA[b.Name] = &b
    }
    ```

1. ไม่ต้องเก็บ pointer ใน map สิ

    ```go
    nameToA := make(map[string]A)
    for _, a := range list {
        nameToA[a.Name] = a
    }
    ```

ดังนั้น เมื่อไรก็ตามที่เราต้องใช้ for range กับ slice ของ value ต้องระวังให้ดี ไม่อย่างนั้นอาจจะเจอปัญหานี้ได้
