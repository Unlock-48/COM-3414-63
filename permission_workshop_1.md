# สรุปผลการทดลอง: งาน Permission Workshop 1 (FreeBSD)

เอกสารสรุปขั้นตอนการปฏิบัติ ผลการสังเกตการณ์ และเหตุผลทางเทคนิคเกี่ยวกับสิทธิ์การเข้าถึงไฟล์และไดเรกทอรี (File & Directory Permissions) บนระบบปฏิบัติการ FreeBSD

---

## ข้อมูลสภาพแวดล้อมการทดสอบ (Environment)
* **เครื่อง Server:** FreeBSD (`IP: 192.168.2.1`)
* **เครื่อง Client:** Windows 8 (`IP: 192.168.2.2`) เชื่อมต่อผ่านโปรโตคอล `Telnet`
* **บัญชีผู้ใช้งานที่ทดสอบ:**
  1. `root` (ผู้ดูแลระบบสูงสุด สิทธิ์เต็ม)
  2. `kantinan` (ผู้ดูแลระบบ/เจ้าของ Home Directory: `/usr/home/kantinan` อยู่ในกลุ่ม `wheel`)
  3. `Doraemon` (ผู้ใช้งานทั่วไป อยู่นอกกลุ่ม `wheel`)

---

## สไลด์ที่ 1: ข้อที่ 1 — ใช้สิทธิ์ `root` สร้างไฟล์ `testbyroot.txt`

### คำสั่งที่ใช้ปฏิบัติ
```sh
su -
touch /usr/home/kantinan/testbyroot.txt
ls -l /usr/home/kantinan
```

### ผลการสังเกตการณ์
* ระบบสร้างไฟล์ `/usr/home/kantinan/testbyroot.txt` ได้สำเร็จ
* เมื่อตรวจสอบด้วย `ls -l` จะพบค่าสิทธิ์ดังนี้:
  ```text
  -rw-r--r--  1 root  kantinan  0 Nov 16 16:08 testbyroot.txt
  ```
* **คำอธิบายสิทธิ์:** เจ้าของไฟล์ (Owner) คือ `root` มีสิทธิ์อ่านและเขียน (`rw-`) ส่วนกลุ่ม (`kantinan`) และคนทั่วไป (Others) มีสิทธิ์อ่านได้อย่างเดียว (`r--`)

---

## สไลด์ที่ 2: ข้อที่ 2 — ใช้สิทธิ์ `kantinan` แก้ไขหรือลบไฟล์ `testbyroot.txt`

### คำสั่งที่ใช้ปฏิบัติ
```sh
su - kantinan
ee /usr/home/kantinan/testbyroot.txt
rm /usr/home/kantinan/testbyroot.txt
```

### ผลการสังเกตการณ์
1. **การแก้ไขไฟล์:** **ไม่สามารถแก้ไขได้** ระบบแจ้งเตือน `unable to create file "/usr/home/kantinan/testbyroot.txt"` (หรือ `Permission denied`)
2. **การลบไฟล์:** **สามารถลบไฟล์ได้สำเร็จ** (ระบบจะถามยืนยัน `override rw-r--r-- root/kantinan for ...?` เมื่อตอบ `y` ไฟล์จะถูกลบทันที)

### เหตุผลทางเทคนิค
* **แก้ไขไม่ได้:** เพราะ `kantinan` ไม่ใช่เจ้าของไฟล์ (`root`) จึงได้รับสิทธิ์เพียงแค่ `r--` (อ่านได้อย่างเดียว)
* **ลบได้:** ในระบบ Unix/FreeBSD สิทธิ์ในการ **ลบไฟล์** ไม่ได้ขึ้นอยู่กับสิทธิ์ของตัวไฟล์ แต่ขึ้นอยู่กับสิทธิ์การเขียน (`w`) ของ **ไดเรกทอรีที่ไฟล์นั้นตั้งอยู่** ซึ่ง `kantinan` เป็นเจ้าของไดเรกทอรี `/usr/home/kantinan` (`rwxr-xr-x`) จึงมีสิทธิ์ลบไฟล์ใดๆ ภายในบ้านของตัวเองได้

---

## สไลด์ที่ 3: ข้อที่ 3 — ใช้สิทธิ์ `Doraemon` แก้ไขหรือลบไฟล์ `testbyroot.txt`

### คำสั่งที่ใช้ปฏิบัติ
```sh
su - Doraemon
echo "test" >> /usr/home/kantinan/testbyroot.txt
rm /usr/home/kantinan/testbyroot.txt
```

### ผลการสังเกตการณ์
* **การแก้ไขไฟล์:** **ไม่สามารถแก้ไขได้** (`Permission denied`)
* **การลบไฟล์:** **ไม่สามารถลบได้** (`Permission denied`)

### เหตุผลทางเทคนิค
* `Doraemon` จัดอยู่ในกลุ่มบุคคลภายนอก (Others) ของทั้งไฟล์ `testbyroot.txt` (สิทธิ์ `r--`) และไดเรกทอรี `/usr/home/kantinan` (สิทธิ์ `r-x`) จึงไม่มีสิทธิ์เขียน (Write) ทั้งในตัวไฟล์และในโฟลเดอร์ ทำให้ทำไม่ได้ทั้งสองอย่าง

---

## สไลด์ที่ 4: ข้อที่ 4 — ใช้สิทธิ์ `Doraemon` แก้ไขหรือลบไฟล์ `testbyadmin.txt` ที่ `kantinan` สร้าง

### คำสั่งที่ใช้ปฏิบัติ
```sh
# 1. สร้างไฟล์ด้วยสิทธิ์ kantinan
touch /usr/home/kantinan/testbyadmin.txt

# 2. ทดสอบด้วยสิทธิ์ Doraemon
su - Doraemon
echo "test" >> /usr/home/kantinan/testbyadmin.txt
rm /usr/home/kantinan/testbyadmin.txt
```

### ผลการสังเกตการณ์
* **การแก้ไขไฟล์:** **ไม่สามารถแก้ไขได้** (`Permission denied`)
* **การลบไฟล์:** **ไม่สามารถลบได้** (`Permission denied`)

### เหตุผลทางเทคนิค
* ไฟล์ `testbyadmin.txt` มีเจ้าของคือ `kantinan` (`-rw-r--r-- kantinan kantinan`) ทำให้ `Doraemon` ซึ่งเป็นคนนอก (Others) มีสิทธิ์เพียงอ่าน (`r--`) เท่านั้น และไม่มีสิทธิ์เขียนในไดเรกทอรี `/usr/home/kantinan` จึงไม่สามารถแก้ไขหรือลบไฟล์นี้ได้

---

## สไลด์ที่ 5: ข้อที่ 5 — ใช้สิทธิ์ `Doraemon` และ `kantinan` แก้ไขไฟล์ `/etc/rc.conf`

### คำสั่งที่ใช้ปฏิบัติ
```sh
# ทดสอบด้วยสิทธิ์ Doraemon
echo "3test" >> /etc/rc.conf

# ทดสอบด้วยสิทธิ์ kantinan
exit
echo "#test" >> /etc/rc.conf
```

### ผลการสังเกตการณ์
* **ทั้ง `Doraemon` และ `kantinan` ไม่สามารถแก้ไขไฟล์ `/etc/rc.conf` ได้ทั้งคู่** โดยระบบขึ้นแจ้งเตือน `cannot create /etc/rc.conf: Permission denied`

### เหตุผลทางเทคนิค
* ไฟล์ `/etc/rc.conf` เป็นไฟล์กำหนดค่าหลักของระบบปฏิบัติการ มีสิทธิ์เริ่มต้นคือ `-rw-r--r-- root wheel`
* ถึงแม้ `kantinan` จะอยู่ในกลุ่ม `wheel` แต่สิทธิ์ของกลุ่ม `wheel` บนไฟล์นี้ถูกกำหนดไว้เพียง `r--` (อ่านได้อย่างเดียว) ดังนั้นหากต้องการแก้ไขไฟล์ระบบ จะต้องยกระดับสิทธิ์เป็น `root` ด้วยคำสั่ง `su -` เสียก่อน

---

## ตารางสรุปผลการทดลองทั้งหมด

| ข้อ | ผู้ปฏิบัติ (User) | เป้าหมาย (Target) | การแก้ไข (Write) | การลบ (Delete) | เหตุผลสรุป |
| :-: | :--- | :--- | :---: | :---: | :--- |
| **1** | `root` | สร้าง `testbyroot.txt` | สำเร็จ | สำเร็จ | `root` เป็นผู้ดูแลสูงสุด มีสิทธิ์ทำได้ทุกอย่าง |
| **2** | `kantinan` | `testbyroot.txt` (ของ `root`) | ไม่ได้ | **ได้** | แก้ไฟล์ไม่ได้เพราะไม่ใช่เจ้าของ แต่ลบได้เพราะเป็นเจ้าของโฟลเดอร์ |
| **3** | `Doraemon` | `testbyroot.txt` (ของ `root`) | ไม่ได้ | ไม่ได้ | เป็นคนนอก (Others) ไม่มีสิทธิ์เขียนทั้งไฟล์และโฟลเดอร์ |
| **4** | `Doraemon` | `testbyadmin.txt` (ของ `kantinan`) | ไม่ได้ | ไม่ได้ | เป็นคนนอก (Others) มีสิทธิ์แค่อ่าน (`r--`) เท่านั้น |
| **5** | `Doraemon` & `kantinan` | `/etc/rc.conf` (ไฟล์ระบบ) | ไม่ได้ | ไม่ได้ | ไฟล์ระบบสงวนสิทธิ์การแก้ไขไว้ให้ `root` เพียงผู้เดียว |