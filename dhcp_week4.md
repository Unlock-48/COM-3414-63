# คู่มือแนวทางการติดตั้งและตั้งค่า DHCP Server บน FreeBSD

เอกสารสรุปขั้นตอนการติดตั้งโปรแกรม ISC DHCP Server 2.0 จาก Source Code บนระบบปฏิบัติการ FreeBSD การกำหนดค่าแจก IP Address อัตโนมัติผ่านการ์ดแลนวงใน (`em1`) การตั้งค่า Auto-start ตอนบูตระบบ และการทดสอบรับ IP ฝั่ง Windows 8 Client
ภาพ ตย
https://docs.google.com/document/d/1eMaK8RTVKLTDIDztfA0GxDT6UXPL8KpeEJGeRnT7GFg/edit?usp=sharing

---

## ภาพรวมระบบ (Environment)
* **Server OS:** FreeBSD (การ์ดแลน `em0` เชื่อมต่ออินเทอร์เน็ต, การ์ดแลน `em1` วงแลนภายใน IP `192.168.2.1`)
* **Client OS:** Windows 8 (เชื่อมต่อวงแลน `VMnet1` เพื่อรับ IP อัตโนมัติจาก FreeBSD)
* **บัญชีผู้ใช้งาน:** `kantinan` (สิทธิ์กลุ่ม `wheel` สามารถใช้ `su -` สลับเป็น `root` ได้)
* **ไฟล์ติดตั้งที่ใช้:** `DHCP-2.0.tar.gz` และไฟล์คอนฟิกตัวอย่าง `DHCPD.CONF` (หรือ `dhcpd.conf`)

---

## ส่วนที่ 1: การโอนไฟล์และติดตั้งโปรแกรมบน FreeBSD (สไลด์ที่ 2–5)

### ขั้นตอนที่ 1: โอนไฟล์ติดตั้งเข้าสู่ Server ผ่าน FTP / SFTP
1. เปิดโปรแกรมโอนไฟล์จากฝั่ง Windows (เช่น WinSCP, SSH Secure File Transfer หรือคำสั่ง `ftp 192.168.2.1` ใน CMD)
2. ล็อกอินด้วยบัญชี `kantinan`
3. อัปโหลดไฟล์ `DHCP-2.0.tar.gz` และ `dhcpd.conf` (หรือ `DHCPD.CONF`) เข้าไปไว้ในโฟลเดอร์ `/usr/home/kantinan`
4. ออกจากโปรแกรมโอนไฟล์ (Logout)

### ขั้นตอนที่ 2: แตกไฟล์และคอมไพล์ติดตั้งโปรแกรม (Compile & Install)
1. ล็อกอินเข้าเครื่อง FreeBSD และสลับสิทธิ์เป็น `root` (สังเกตเครื่องหมาย Prompt เป็น `#`)
2. ย้ายไปยังโฟลเดอร์ที่เก็บไฟล์อัปโหลดไว้ และก๊อปปี้ไฟล์คอนฟิกตัวอย่างไปไว้ที่ `/etc/`:
   ```sh
   cd /usr/home/kantinan
   cp dhcpd.conf /etc/dhcpd.conf
   ```
   *(หมายเหตุ: เช็กชื่อไฟล์ด้วยคำสั่ง `ls` ก่อน หากต้นฉบับเป็นตัวพิมพ์ใหญ่ให้ใช้คำสั่ง `cp DHCPD.CONF /etc/dhcpd.conf`)*
3. แตกไฟล์ Source Code ของ DHCP Server:
   ```sh
   gzip -cd dhcp-2.0.tar.gz | tar xvf -
   ```
   *(ข้อควรระวัง: ตรวจสอบจุด `.` ระหว่าง `2.0` และ `tar.gz` ให้ถูกต้อง สามารถกดปุ่ม `Tab` เพื่อเติมชื่อไฟล์อัตโนมัติ)*
4. เข้าไปในโฟลเดอร์ที่แตกออกมา แล้วรันคำสั่งสร้างและติดตั้งโปรแกรมทีละบรรทัด:
   ```sh
   cd dhcp-2.0
   ./configure
   make
   make install
   ```

---

## ส่วนที่ 2: การกำหนดค่าแจก IP ใน `/etc/dhcpd.conf` (สไลด์ที่ 6)

1. เปิดแก้ไขไฟล์คอนฟิกหลักของ DHCP Server ด้วยโปรแกรม `ee`:
   ```sh
   ee /etc/dhcpd.conf
   ```
2. ปรับแก้ค่าในส่วนประกาศ `subnet` ด้านล่างไฟล์ให้ตรงกับวงแลนของการ์ดแลน `em1` (`192.168.2.0/24`) ดังนี้:
   ```text
   default-lease-time 600;
   max-lease-time 7200;
   option subnet-mask 255.255.255.128;
   option broadcast-address 10.80.104.127;
   option routers 10.80.104.1;
   option domain-name-servers 8.8.8.8;
   option domain-name "com";
   option host-name "kantinan.com";

   subnet 192.168.2.0 netmask 255.255.255.0 {
       range 192.168.2.50 192.168.2.100;
       option domain-name "com";
       option domain-name-servers 8.8.8.8;
       option host-name "kantinan.com";
       option routers 192.168.2.1;
       option subnet-mask 255.255.255.0;
   }
   ```
   *(หมายเหตุ: ค่าภายในบล็อก `subnet 192.168.2.0 { ... }` จะเข้าไปเขียนทับค่าส่วนกลางด้านบนให้อัตโนมัติ)*
3. **จุดที่ต้องตรวจสอบก่อนบันทึก:**
   * ทุกบรรทัดที่ขึ้นต้นด้วย `option ...` ต้องลงท้ายด้วยเครื่องหมายเซมิโคลอน **`;`** ครบทุกบรรทัด (หากลืมจะขึ้น Error `semicolon expected` ตอนสตาร์ท Service)
   * ตรวจสอบเครื่องหมายปีกกาเปิด `{` และปิด `}` ของบล็อก `subnet` ให้ครบคู่ ไม่มีตัวเกิน
4. **บันทึกและออกจาก `ee`:** กดปุ่ม **Esc** $\rightarrow$ กด **Enter** (leave editor) $\rightarrow$ กด **Enter** (save changes)

---

## ส่วนที่ 3: การสร้างฐานข้อมูลและเปิดใช้งาน Service (สไลด์ที่ 7 & 9)

### ขั้นตอนที่ 1: สร้างไฟล์ `dhcpd.leases` และกำหนดสิทธิ์
สร้างไฟล์สำหรับบันทึกประวัติการเช่า IP ของเครื่องลูกข่าย และกำหนดสิทธิ์การเข้าถึงเป็น `777`:
```sh
touch /var/db/dhcpd.leases
chmod 777 /var/db/dhcpd.leases
```
* **ตรวจสอบสิทธิ์ไฟล์:**
  ```sh
  ls -l /var/db/dhcpd.leases
  ```
  *(ต้องแสดงสิทธิ์ด้านหน้าเป็น `-rwxrwxrwx`)*

### ขั้นตอนที่ 2: สั่งรันบริการ DHCP Server
สั่งเปิดใช้งาน `dhcpd` โดยระบุชื่อการ์ดแลนวงใน (`em1`) และใส่เครื่องหมาย `&` เพื่อรันเบื้องหลัง (อย่าลืม `/` ตัวหน้าสุด):
```sh
/usr/sbin/dhcpd em1 &
```
* **ตรวจสอบการทำงานของโปรเซส:**
  ```sh
  ps -ax | grep dhcpd
  ```
  *(ต้องเห็นบรรทัด `/usr/sbin/dhcpd em1` ทำงานอยู่)*

### ขั้นตอนที่ 3: ตั้งค่าให้เปิดทำงานอัตโนมัติทุกครั้งที่บูตเครื่อง (`/etc/rc.local`)
1. เปิดแก้ไขไฟล์ `/etc/rc.local`:
   ```sh
   ee /etc/rc.local
   ```
2. พิมพ์คำสั่งรัน Service ลงไปในไฟล์:
   ```sh
   /usr/sbin/dhcpd em1 &
   ```
3. กด **Esc** $\rightarrow$ **Enter** $\rightarrow$ **Enter** เพื่อบันทึกและออก
4. ทดสอบรีสตาร์ทระบบ 1 รอบเพื่อให้มั่นใจว่า Auto-start ทำงานถูกต้อง:
   ```sh
   reboot
   ```
   *(หลังเครื่องบูตเสร็จ ล็อกอินด้วย `root` แล้วพิมพ์ `ps -ax | grep dhcpd` เช็กอีกครั้ง)*

---

## ส่วนที่ 4: การทดสอบรับ IP อัตโนมัติฝั่ง Windows 8 (สไลด์ที่ 8)

1. บนเครื่อง **Windows 8** กดปุ่ม **Windows + R** พิมพ์ `ncpa.cpl` แล้วกด **Enter**
2. คลิกขวาที่การ์ดแลนวงใน (**Local Area Connection 2** ที่เชื่อมต่อกับ `VMnet1`) $\rightarrow$ **Properties** $\rightarrow$ ดับเบิลคลิก **Internet Protocol Version 4 (TCP/IPv4)**
3. เปลี่ยนจากการกำหนด IP ด้วยมือเป็น:
   * เลือก **Obtain an IP address automatically**
   * เลือก **Obtain DNS server address automatically**
   * กด **OK** ทั้งหมด
4. เปิด **Command Prompt (CMD)** แล้วพิมพ์คำสั่งตรวจสอบค่า IP ที่ได้รับ:
   ```cmd
   ipconfig
   ```
   **ผลลัพธ์ที่ถูกต้อง (สำเร็จ):**
   * **Connection-specific DNS Suffix:** `com`
   * **IPv4 Address:** `192.168.2.50` *(อยู่ในช่วง `192.168.2.50` – `192.168.2.100` ตามที่ตั้งค่าไว้)*
   * **Subnet Mask:** `255.255.255.0`
   * **Default Gateway:** `192.168.2.1`
5. *(เพิ่มเติม)* สามารถตรวจสอบประวัติการแจก IP ฝั่ง FreeBSD ได้ด้วยคำสั่ง:
   ```sh
   cat /var/db/dhcpd.leases
   ```

---

## ตารางสรุปข้อผิดพลาดที่พบบ่อยและวิธีแก้ไข (Troubleshooting)

| อาการ / ข้อความแจ้งเตือน | สาเหตุ | วิธีแก้ไข |
| :--- | :--- | :--- |
| `No such file or directory` ตอนแตกไฟล์ | พิมพ์ชื่อไฟล์ผิด เช่น ลืมจุด `.` ก่อน `tar.gz` | ใช้คำสั่ง `ls` เช็กชื่อไฟล์จริง และกดปุ่ม **Tab** ช่วยเติมชื่อไฟล์ |
| `new file "/etc/dhcpd.conf"` ตอนเปิด `ee` | ลืมก๊อปปี้ไฟล์ตัวอย่าง `dhcpd.conf` มาไว้ที่ `/etc/` | กด **Esc** $\rightarrow$ leave editor $\rightarrow$ no changes แล้วใช้คำสั่ง `cp` ก๊อปปี้ไฟล์มาก่อน |
| `line 10: semicolon expected` | ลืมใส่เครื่องหมาย `;` ปิดท้ายบรรทัดก่อนหน้า `subnet` หรือมี `}` เกิน | เปิด `ee /etc/dhcpd.conf` เติม `;` ท้ายคำสั่ง `option` ให้ครบทุกบรรทัด |
| `usr/sbin/dhcpd: Command not found` | ลืมใส่เครื่องหมาย `/` ตัวหน้าสุดของ Path | พิมพ์ Path ให้ครบเป็น `/usr/sbin/dhcpd em1 &` |
| `Address already in use` | Service `dhcpd` เดิมกำลังทำงานอยู่แล้ว | ไม่ต้องรันซ้ำ (หากแก้คอนฟิกใหม่ให้พิมพ์ `killall dhcpd` ก่อนแล้วค่อยสตาร์ทใหม่) |
