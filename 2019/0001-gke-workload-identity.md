# สรุปวิธีใช้ Workload Identity ใน GKE

ถ้าเราต้องการให้ permission gcloud กับ container ที่รันใน GKE ปัจจุบันเราสามารถทำได้ง่าย ๆ 2 วิธี คือ

1. ให้ permission กับ node ใน GKE ที่รัน container, container ทุกตัวที่รันใน node นั้นจะมี permission ของ node ที่มันรันอยู่

    ข้อเสียคือถ้าเรารัน container ที่เราไม่รู้จัก container นั้นก็จะมี permission เดียวกับ node ที่รัน

1. สร้าง service account ใน gcloud แล้วเอา private key มาเก็บใน secret ใน kubernetes แล้วก็ mount เข้าไปเป็น volumn ใน container

    ข้อเสียคือ private key มีวันหมดอายุ ถ้าเราลืม rotate key ก่อนหมดอายุ ระบบอาจจะ down ได้เลย
    อีกอย่างคือ container ที่มี permission ในการอ่าน secret ใน kubernetes ก็สามารถอ่าน private key ของเราออกมาได้ เช่น ingress controller ที่ต้องอ่าน tls cert/key จากใน secret จะขอ permission ในการอ่าน secret ทั้งหมดใน cluster เช่น [nginx-ingress-controller](https://github.com/kubernetes/ingress-nginx/blob/28793092e779f7cb66504a0e41db1fce2f93d91e/deploy/cluster-wide/cluster-role.yaml#L13)

[Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
คือ feature ใหม่ที่จะมาช่วย binding service account ใน kubernetes กับ service account ใน gcloud ทำให้เราไม่ต้องใช้ private key

## หลักการทำงาน

เมื่อเราเปิด Workload Identity ใน GKE แล้ว จะมี Daemon Set ถูกรันขึ้นมาเพิ่ม 2 ตัว คือ

1. [netd](https://github.com/GoogleCloudPlatform/netd)
1. gke-metadata-server

เมื่อ container ของเราจะเรียก api ที่ gcloud netd จะ route traffic ไปที่ gke-metadata-server เพื่อให้ metadata-service inject access token (อายุสั้น ๆ) ที่ header ของ request ของเราก่อนส่งไปหา api ที่ gcloud

## สิ่งที่เราต้องทำ

1. สร้าง namespace ใน kubernetes ถ้ายังไม่มี

    ```sh
    $ kubectl create ns my-namespace
    ```

1. สร้าง service account ใน kubernetes (KSA) ถ้ายังไม่มี หรือจะใช้ default ของแต่ละ namespace ก็ได้

    ```sh
    $ kubectl create sa my-ksa
    ```

1. สร้าง service account ใน gcloud (GSA)

    ```sh
    $ gcloud iam service-accounts create my-gsa
    ```

1. ให้ permission กับ gke-metadata-server ให้สามารถสร้าง access token ของ service account ของเราได้

    ```sh
    $ gcloud iam service-accounts add-iam-policy-binding \
        --role roles/iam.workloadIdentityUser \
        --member "serviceAccount:[PROJECT_ID].svc.id.goog[my-namespace/my-ksa]" \
        my-gsa@[PROJECT_ID].iam.gserviceaccount.com
    ```

1. บอก gke-metadata-server ว่า KSA ของเราให้ใช้ GSA อันไหน

    ```sh
    $ kubectl annotate sa \
        --namespace my-namespace \
        my-ksa \
        iam.gke.io/gcp-service-account=my-gsa@[PROJECT_ID].iam.gserviceaccount.com
    ```

1. เวลาสร้าง pod ก็แค่บอก kubernetes ว่าจะใช้ service account (KSA) อันไหน

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    ...
    spec:
        ...
        template:
            ...
            spec:
                ...
                serviceAccountName: my-ksa
    ```

แค่นี้ container ของเรา ก็จะมี permission เดียวกับ GSA ที่เราสร้างแล้ว โดยที่ไม่ต้องคอยกังวลว่า private key จะหมดอายุ หรือใครจะแอบอ่าน private key ของเรา
