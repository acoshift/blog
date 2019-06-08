# กำหนดความสำคัญของ Pod ใน Kubernetes ด้วย Priority Class

ถ้าเราต้องการให้บาง pod ใน kubernetes มีความสำคัญมากกว่า pod อื่น ๆ เช่น ingress controller หรือ database
เราสามารถกำหนด priority ของ pod ได้ด้วย [Priority Class](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#priorityclass-v1-scheduling-k8s-io)

## หลักการทำงาน

1. ถ้ามี pod ที่อยู่ใน status pending อยู่หลาย pod, kubernetes จะพยายามเอา pod ที่ priority สูงสุดมา schedule ก่อน
ถ้า pod นั้นไม่สามารถ schedule ได้ kubernetes จะ schedule pod ที่ priority ต่ำกว่าตามลงมา

1. ถ้า node ไม่มีที่เหลือให้ schedule pod, kubernetes จะลบ pod ที่ priority ต่ำกว่า เพื่อให้มีที่ที่สามารถ schedule pod ที่มี priority สูงกว่าได้

## สิ่งที่เราต้องทำ

1. สร้าง Priority Class

    ```yaml
    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
    name: high-priority
    value: 1000000
    ```

1. เวลาสร้าง pod ให้กำหนด `priorityClassName` ลงไปใน pod spec

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
      priorityClassName: high-priority
    ```
