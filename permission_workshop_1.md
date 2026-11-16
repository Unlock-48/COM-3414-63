# คู่มือวิธีทำฉบับสมบูรณ์: FreeBSD Server & งาน Permission Workshop 1

เอกสารรวบรวมขั้นตอนการปฏิบัติตั้งแต่การเริ่มต้นตั้งค่าระบบ FreeBSD การสร้างผู้ใช้งาน การเชื่อมต่อเครือข่ายกับ Windows 8 ไปจนถึงวิธีทำและสรุปผลการทดลองเรื่องสิทธิ์การเข้าถึงไฟล์ (Permissions) ทั้ง 5 ข้อ

---

## ส่วนที่ 1: การตั้งค่าฝั่ง FreeBSD Server

### ขั้นตอนที่ 1: ล็อกอินเข้าสู่ระบบ (สิทธิ์ root)
1. เปิดเครื่อง FreeBSD ที่หน้า `login:` ให้พิมพ์ `root` แล้วกด **Enter**
2. ใส่รหัสผ่านของ `root` (ขณะพิมพ์จะไม่แสดงตัวอักษรบนหน้าจอ) แล้วกด **Enter**
3. เมื่อสำเร็จ เครื่องหมายพร้อมรับคำสั่งจะแสดงเป็น `#`

### ขั้นตอนที่ 2: สร้างบัญชีผู้ใช้ (`kantinan` และ `Doraemon`)
ใช้คำสั่ง `adduser` เพื่อสร้างบัญชีผู้ใช้งานสำหรับทดสอบ:
```sh
adduser
```
1. **สร้างบัญชีผู้ดูแลระบบ (`kantinan`):**
   * `Username:` พิมพ์ `kantinan`
   * `Invite kantinan into other groups?:` พิมพ์ `wheel` *(สำคัญ: เพื่อให้ใช้คำสั่ง `su -` สลับเป็น `root` ได้)*
   * `Enter password:` ตั้งรหัสผ่าน `1234` (พิมพ์ยืนยัน 2 ครั้ง)
   * `Lock out the account after creation?:` พิมพ์ `no`
   * `OK? (yes/no):` พิมพ์ `yes`
2. **สร้างบัญชีผู้ใช้ทั่วไป (`Doraemon`):**
   * เมื่อระบบถาม `Add another user? (yes/no):` ให้พิมพ์ `yes`
   * `Username:` พิมพ์ `Doraemon`
   * `Invite Doraemon into other groups?:` กด **Enter** ข้าม *(ไม่ต้องใส่ `wheel`)*
   * `Enter password:` ตั้งรหัสผ่าน `1234` (พิมพ์ยืนยัน 2 ครั้ง)
   * `Lock out the account after creation?:` พิมพ์ `no`
   * `OK? (yes/no):` พิมพ์ `yes` และพิมพ์ `no` เมื่อเสร็จสิ้น

*(หมายเหตุ: หากสร้างไปแล้วพบว่าบัญชีถูกล็อก หรือลืมใส่กลุ่ม `wheel` ให้ใช้คำสั่งแก้ไขดังนี้)*
```sh
pw unlock Doraemon
pw groupmod wheel -m kantinan
```

### ขั้นตอนที่ 3: สำรองไฟล์และตั้งค่า IP Address (`rc.conf`)
1. ย้ายไปที่โฟลเดอร์ `/etc` และก๊อปปี้ไฟล์สำรองชื่อ `rcbak.conf`:
   ```sh
   cd /etc
   cp rc.conf rcbak.conf
   ls -l rcbak.conf
   ```
2. เปิดแก้ไขไฟล์ `/etc/rc.conf` เพื่อกำหนด IP และเปิดบริการรีโมท:
   ```sh
   ee /etc/rc.conf
   ```
3. เลื่อนลงมาบรรทัดล่างสุด แล้วพิมพ์ข้อความต่อไปนี้เพิ่มเข้าไป:
   ```sh
   ifconfig_em1="inet 192.168.2.1 netmask 255.255.255.0"
   inetd_enable="YES"
   sshd_enable="YES"
   ```
4. **บันทึกและออก:** กดปุ่ม **Esc** $\rightarrow$ กด **Enter** $\rightarrow$ กด **Enter**

### ขั้นตอนที่ 4: เปิดใช้งานบริการ Telnet, SSH, FTP (`inetd.conf`) และรีสตาร์ท
1. เปิดแก้ไขไฟล์ `/etc/inetd.conf`:
   ```sh
   ee /etc/inetd.conf
   ```
2. ลบเครื่องหมาย `#` ด้านหน้าบรรทัด `ftp`, `ssh` และ `telnet` (ทั้ง `tcp` และ `tcp6`) ออก
3. กด **Esc** $\rightarrow$ **Enter** $\rightarrow$ **Enter** เพื่อบันทึกและออก
4. รีสตาร์ทเครื่องเพื่อให้การตั้งค่าทั้งหมดทำงาน:
   ```sh
   reboot
   ```
5. หลังเครื่องบูตเสร็จ ให้ล็อกอินด้วย `root` และตรวจสอบสถานะ:
   ```sh
   ifconfig em1
   sockstat -4
   ```
   *(ต้องเห็น IP `192.168.2.1` สถานะ `active` และพอร์ต `21`, `22`, `23` เปิดทำงานอยู่)*

---

## ส่วนที่ 2: การตั้งค่าและเชื่อมต่อจากฝั่ง Windows 8 Client

### ขั้นตอนที่ 1: ตั้งค่า IP Address บน Windows 8
1. กดปุ่ม **Windows + R** พิมพ์ `ncpa.cpl` แล้วกด **Enter**
2. คลิกขวาที่การ์ดแลน $\rightarrow$ **Properties** $\rightarrow$ ดับเบิลคลิก **Internet Protocol Version 4 (TCP/IPv4)**
3. เลือก **Use the following IP address** และกรอกค่า:
   * **IP address:** `192.168.2.2`
   * **Subnet mask:** `255.255.255.0`
4. กด **OK** ทั้งหมด

### ขั้นตอนที่ 2: เปิดใช้งาน Telnet Client บน Windows 8
1. กดปุ่ม **Windows + R** พิมพ์ `optionalfeatures` แล้วกด **Enter**
2. ติ๊กถูกหน้าหัวข้อ **Telnet Client** แล้วกด **OK** รอจนติดตั้งเสร็จ

### ขั้นตอนที่ 3: ทดสอบการเชื่อมต่อและรีโมทเข้าสู่ FreeBSD
1. เปิดโปรแกรม **Command Prompt (CMD)** พิมพ์ทดสอบสัญญาณ:
   ```cmd
   ping 192.168.2.1
   ```
2. เชื่อมต่อรีโมทด้วยคำสั่ง Telnet:
   ```cmd
   telnet 192.168.2.1
   ```
3. ล็อกอินด้วยบัญชี `kantinan` รหัสผ่าน `1234` (จากนั้นสามารถพิมพ์ `su -` และใส่รหัส `root` เพื่อสลับเป็นผู้ดูแลระบบสูงสุดได้)
4. *(เพิ่มเติม)* หากต้องการดูไฟล์ในโฟลเดอร์ผ่านหน้าต่างกราฟิก ให้เปิด **Windows Explorer** แล้วพิมพ์ในช่อง Address Bar ว่า `ftp://192.168.2.1` จากนั้นกรอก User: `kantinan` และ Pass: `1234`

---

## ส่วนที่ 3: วิธีทำและสรุปผล "งาน Permission Workshop 1"

### ข้อที่ 1: ใช้สิทธิ์ `root` สร้างไฟล์ `testbyroot.txt` ใน `/usr/home/kantinan`
* **วิธีทำ:**
  ```sh
  su -
  touch /usr/home/kantinan/testbyroot.txt
  ls -l /usr/home/kantinan
  ```
* **ผลลัพธ์:** สร้างไฟล์สำเร็จ สิทธิ์ที่ได้คือ `-rw-r--r-- 1 root kantinan` (เจ้าของคือ `root`)

### ข้อที่ 2: ใช้สิทธิ์ `kantinan` เข้าไปแก้ไข หรือลบไฟล์ `testbyroot.txt`
* **วิธีทำ:**
  ```sh
  exit
  echo "test" >> /usr/home/kantinan/testbyroot.txt
  rm /usr/home/kantinan/testbyroot.txt
  ```
  *(เมื่อลบเสร็จ ให้สลับเป็น `root` ด้วย `su -` แล้วใช้ `touch /usr/home/kantinan/testbyroot.txt` สร้างไฟล์คืนไว้ทำข้อ 3 ต่อ)*
* **ผลลัพธ์:**
  1. **แก้ไขไม่ได้** (`Permission denied` / `unable to create file`) เพราะ `kantinan` ไม่ใช่เจ้าของไฟล์ (`root`) จึงมีสิทธิ์แค่อ่าน (`r--`)
  2. **ลบได้สำเร็จ** (ตอบ `y` เพื่อยืนยัน) เพราะสิทธิ์การลบไฟล์ขึ้นอยู่กับสิทธิ์การเขียน (`w`) ของไดเรกทอรี ซึ่ง `kantinan` เป็นเจ้าของโฟลเดอร์ `/usr/home/kantinan`

### ข้อที่ 3: ใช้สิทธิ์ `Doraemon` เข้าไปแก้ไข หรือลบไฟล์ `testbyroot.txt`
* **วิธีทำ:**
  ```sh
  su - Doraemon
  echo "test" >> /usr/home/kantinan/testbyroot.txt
  rm /usr/home/kantinan/testbyroot.txt
  ```
* **ผลลัพธ์:** **แก้ไขไม่ได้ และลบไม่ได้ทั้งคู่** (`Permission denied`) เพราะ `Doraemon` เป็นคนนอก (Others) ไม่มีสิทธิ์เขียนทั้งในตัวไฟล์และในโฟลเดอร์ `/usr/home/kantinan`

### ข้อที่ 4: ใช้สิทธิ์ `Doraemon` เข้าไปแก้ไข หรือลบไฟล์ `testbyadmin.txt` ที่ `kantinan` สร้าง
* **วิธีทำ:**
  ```sh
  # 1. กลับมาเป็น kantinan และสร้างไฟล์ testbyadmin.txt
  exit
  touch /usr/home/kantinan/testbyadmin.txt

  # 2. สลับเป็น Doraemon เพื่อทดสอบ
  su - Doraemon
  echo "test" >> /usr/home/kantinan/testbyadmin.txt
  rm /usr/home/kantinan/testbyadmin.txt
  ```
* **ผลลัพธ์:** **แก้ไขไม่ได้ และลบไม่ได้ทั้งคู่** (`Permission denied`) เพราะไฟล์นี้มีเจ้าของคือ `kantinan` (`-rw-r--r--`) คนนอกจึงอ่านได้อย่างเดียว

### ข้อที่ 5: ใช้สิทธิ์ `Doraemon` และ `kantinan` แก้ไขไฟล์ `/etc/rc.conf`
* **วิธีทำ:**
  ```sh
  # ทดสอบด้วยสิทธิ์ Doraemon
  echo "3test" >> /etc/rc.conf

  # กลับมาเป็น kantinan แล้วทดสอบ
  exit
  echo "#test" >> /etc/rc.conf
  ```
* **ผลลัพธ์:** **ทั้ง `Doraemon` และ `kantinan` ไม่สามารถแก้ไข `/etc/rc.conf` ได้ทั้งคู่** (`cannot create /etc/rc.conf: Permission denied`) เพราะไฟล์ระบบมีสิทธิ์ `-rw-r--r-- root wheel` สงวนสิทธิ์การแก้ไขไว้ให้ `root` เพียงผู้เดียว

---

## ตารางสรุปผลการทดลองทั้งหมด

| ข้อ | ผู้ปฏิบัติ (User) | เป้าหมาย (Target) | การแก้ไข (Write) | การลบ (Delete) | เหตุผลสรุป |
| :-: | :--- | :--- | :---: | :---: | :--- |
| **1** | `root` | สร้าง `testbyroot.txt` | สำเร็จ | สำเร็จ | `root` เป็นผู้ดูแลสูงสุด มีสิทธิ์ทำได้ทุกอย่าง |
| **2** | `kantinan` | `testbyroot.txt` (ของ `root`) | ไม่ได้ | **ได้** | แก้ไฟล์ไม่ได้เพราะไม่ใช่เจ้าของ แต่ลบได้เพราะเป็นเจ้าของโฟลเดอร์ |
| **3** | `Doraemon` | `testbyroot.txt` (ของ `root`) | ไม่ได้ | ไม่ได้ | เป็นคนนอก (Others) ไม่มีสิทธิ์เขียนทั้งไฟล์และโฟลเดอร์ |
| **4** | `Doraemon` | `testbyadmin.txt` (ของ `kantinan`) | ไม่ได้ | ไม่ได้ | เป็นคนนอก (Others) มีสิทธิ์แค่อ่าน (`r--`) เท่านั้น |
| **5** | `Doraemon` & `kantinan` | `/etc/rc.conf` (ไฟล์ระบบ) | ไม่ได้ | ไม่ได้ | ไฟล์ระบบสงวนสิทธิ์การแก้ไขไว้ให้ `root` เพียงผู้เดียว |
