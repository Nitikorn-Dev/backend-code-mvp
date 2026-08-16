# เอกสารเกณฑ์การยอมรับงาน (Acceptance Criteria & Test Scenarios)
## โครงการ: Backend API Security & Core Hardening MVP (กลุ่ม 1)

---

### 1. บทนำ (Introduction)

เอกสารฉบับนี้กำหนดเกณฑ์การยอมรับงาน (Acceptance Criteria) และสถานการณ์การทดสอบ (Test Scenarios) โดยใช้รูปแบบ **Gherkin Syntax (Given-When-Then)** เพื่อให้ทีมพัฒนา (Developers), ผู้ดูแลผลิตภัณฑ์ (Product Owner) และทีมทดสอบ (QA) มีความเข้าใจตรงกันในการตรวจรับงานแต่ละฟังก์ชัน

---

### 2. เกณฑ์การยอมรับงานรายโมดูล (Acceptance Criteria per Module)

#### Module 1: JWT Authentication & Role-Based Access Control (RBAC)

* **AC-01.1: เข้าถึง API เมื่อส่ง Token ที่ถูกต้องและมีสิทธิ์ตาม Role**
  * **Given:** ผู้ใช้งานเข้าสู่ระบบสำเร็จและถือ JWT Token ที่มี Role เป็น `ADMIN`
  * **When:** ผู้ใช้ส่ง Request `GET /api/v1/reports/export/users` พร้อม Header `Authorization: Bearer <valid_admin_token>`
  * **Then:** ระบบต้องส่งกลับ HTTP Status `200 OK` และส่งข้อมูลไฟล์รายงานที่ต้องการ Export

* **AC-01.2: ปฏิเสธการเข้าถึง Endpoint Export เมื่อ Role ไม่ได้รับอนุญาต (Insufficient Role)**
  * **Given:** ผู้ใช้งานเข้าสู่ระบบสำเร็จและถือ JWT Token ที่มี Role เป็น `USER` ทั่วไป
  * **When:** ผู้ใช้พยายามส่ง Request `GET /api/v1/reports/export/users` พร้อม Header `Authorization: Bearer <valid_user_token>`
  * **Then:** ระบบต้องส่งกลับ HTTP Status `403 Forbidden` พร้อมข้อความแจ้งเตือน `"Access denied: Insufficient permissions"` และไม่ส่งข้อมูลไฟล์ออกไป

* **AC-01.3: ปฏิเสธการเข้าถึงเมื่อไม่ส่ง Token หรือส่ง Token ไม่ถูกต้อง/หมดอายุ**
  * **Given:** Client ไม่แนบ Token หรือแนบ Token ที่ถูกปลอมแปลง หรือ Token ที่หมดอายุแล้ว
  * **When:** ส่ง Request ไปยัง Protected Endpoint ใดๆ
  * **Then:** ระบบต้องส่งกลับ HTTP Status `401 Unauthorized` พร้อมข้อความ `"Invalid or expired token"`

---

#### Module 2: CORS Configuration (การควบคุมการเข้าถึงข้ามโดเมน)

* **AC-02.1: อนุญาต Request จาก Origin ที่อยู่ใน Whitelist**
  * **Given:** โดเมน `https://app.company.com` ถูกบันทึกไว้ใน Whitelist Configuration
  * **When:** Client ส่ง HTTP Request มายัง API โดยแนบ Header `Origin: https://app.company.com`
  * **Then:** ระบบต้องประมวลผลคำขอสำเร็จ และแนบ Header `Access-Control-Allow-Origin: https://app.company.com` ใน Response

* **AC-02.2: ปฏิเสธ Request จาก Origin ที่ไม่ได้อยู่ใน Whitelist**
  * **Given:** โดเมน `https://untrusted-site.com` ไม่ได้อยู่ใน Whitelist
  * **When:** Client ส่ง HTTP Request หรือ Preflight `OPTIONS` โดยแนบ Header `Origin: https://untrusted-site.com`
  * **Then:** ระบบต้องไม่แนบ Header `Access-Control-Allow-Origin` หรือส่งกลับ HTTP Status `403 Forbidden` ทำให้ Browser บล็อกการอ่านข้อมูล

* **AC-02.3: การตอบกลับ Preflight `OPTIONS` Request**
  * **Given:** Client ส่ง Preflight Request ด้วย HTTP Method `OPTIONS`
  * **When:** ตรวจสอบ Origin ถูกต้องตาม Whitelist
  * **Then:** ระบบต้องส่งกลับ HTTP Status `204 No Content` หรือ `200 OK` พร้อม Header `Access-Control-Allow-Methods` และ `Access-Control-Allow-Headers` ที่ระบุชัดเจน

---

#### Module 3: Global Error Handler & SQL Masking (การซ่อนข้อผิดพลาดระบบ)

* **AC-03.1: ซ่อนข้อผิดพลาดฐานข้อมูล (SQL Error / DB Exception)**
  * **Given:** ฐานข้อมูลเกิดปัญหา เช่น Syntax Error, Connection Timeout, หรือ Deadlock
  * **When:** Client เรียกใช้งาน Endpoint ใดๆ ที่มีการ Query ฐานข้อมูล
  * **Then:**
    * Client ต้องได้รับ HTTP Status `500 Internal Server Error`
    * Response Body ต้องเป็น Generic JSON Format เช่น:
      ```json
      {
        "success": false,
        "error": {
          "code": "INTERNAL_SERVER_ERROR",
          "message": "เกิดข้อผิดพลาดขึ้นในระบบ กรุณาลองใหม่อีกครั้ง",
          "trace_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
        }
      }
      ```
    * ห้ามมีคำสั่ง SQL, ชื่อตาราง (Table Name), หรือ Stack Trace ปรากฏใน Response แก่ Client โดยเด็ดขาด
    * Full Stack Trace และ SQL Detail ต้องถูกบันทึกลง Server Log ภายในระบบควบคู่กับ `trace_id` เดียวกัน

* **AC-03.2: การจัดการ Input Validation Error**
  * **Given:** Client ส่ง Request Payload ที่ฟิลด์บังคับไม่ครบถ้วน หรือ Data Type ไม่ถูกต้อง
  * **When:** Request ผ่าน Validation Layer
  * **Then:** ระบบต้องส่งกลับ HTTP Status `400 Bad Request` พร้อมแจกแจงรายการฟิลด์ที่ผิดพลาดอย่างปลอดภัย

---

#### Module 4: OTP Rate Limiter & SMS Bombing Protection (การจำกัดความถี่ OTP)

* **AC-04.1: ป้องกันการขอ OTP ถี่เกินกำหนด (Request Rate Limit)**
  * **Given:** เบอร์โทรศัพท์ `089-111-2233` มีการส่งคำขอ OTP ไปแล้วครบ 3 ครั้งภายในรอบ 15 นาที
  * **When:** ผู้ใช้หรือบอทพยายามส่ง Request ขอ OTP ครั้งที่ 4 ภายในช่วงเวลา 15 นาทีเดิม
  * **Then:**
    * ระบบต้องปฏิเสธคำขอและส่งกลับ HTTP Status `429 Too Many Requests`
    * Response ต้องระบุข้อความแจ้งเตือนและ Header `Retry-After`
    * ระบบต้องไม่ส่งคำสั่งยิง SMS ไปยัง SMS Gateway Provider

* **AC-04.2: การล็อกการยืนยันรหัส OTP เมื่อกรอกผิดเกินกำหนด (Verification Lockout)**
  * **Given:** ผู้ใช้มี OTP Transaction ที่ยังไม่หมดอายุ แต่กรอกรหัส OTP ผิดพลาดติดต่อกันครบ 5 ครั้ง
  * **When:** ผู้ใช้พยายามส่งรหัส OTP มายืนยันเป็นครั้งที่ 6
  * **Then:**
    * ระบบต้องตอบกลับ HTTP Status `400 Bad Request` หรือ `423 Locked`
    * สถานะ OTP Transaction นั้นต้องถูกปรับเป็นยกเลิก (Invalidated) ทันที ผู้ใช้ต้องเริ่มขั้นตอนขอ OTP ใหม่ทั้งหมด

* **AC-04.3: การหมดอายุของรหัส OTP (TTL Expiration)**
  * **Given:** รหัส OTP ถูกสร้างขึ้นมาเกินระยะเวลา 5 นาที (TTL Exceeded)
  * **When:** ผู้ใช้ส่งรหัสที่ถูกต้องมายืนยัน
  * **Then:** ระบบต้องส่งกลับ HTTP Status `400 Bad Request` พร้อมข้อความ `"OTP has expired, please request a new code"`

---

#### Module 5: Swagger Docs Protection (การป้องกันเอกสาร API Spec)

* **AC-05.1: ซ่อน Swagger UI บน Production Environment**
  * **Given:** แอปพลิเคชันถูก Deploy บนสภาพแวดล้อม Production (`NODE_ENV=production` หรือ `APP_ENV=production`)
  * **When:** บุคคลภายนอกเปิด URL `/swagger`, `/api-docs` หรือ `/openapi.json`
  * **Then:** ระบบต้องตอบกลับ HTTP Status `404 Not Found` หรือเรียกตรวจ Basic Authentication (HTTP `401`) โดยไม่เปิดเผย API Endpoint Spec สู่สาธารณะ

* **AC-05.2: เปิดใช้งาน Swagger บน Development / Staging Environment**
  * **Given:** แอปพลิเคชันรันบนสภาพแวดล้อม Development / Staging
  * **When:** นักพัฒนาเปิด URL `/api-docs`
  * **Then:** ระบบต้องแสดงผลหน้า Swagger UI พร้อมเรียกดู Interactive Documentation และทดสอบ Endpoint ได้ตามปกติ

---

### 3. ตารางสรุป Test Scenarios Matrix

| Scenario ID | โมดูล | กรณีทดสอบ (Test Case) | Expected Result |
| :--- | :--- | :--- | :--- |
| **TC-SEC-01** | JWT | ส่ง Valid Token + Role Admin ไปที่ Route Export | `200 OK` + Data File |
| **TC-SEC-02** | JWT | ส่ง Valid Token + Role User ไปที่ Route Export | `403 Forbidden` |
| **TC-SEC-03** | JWT | ส่ง Expired Token หรือ Invalid Token | `401 Unauthorized` |
| **TC-CORS-01** | CORS | ยิง Request จาก Whitelisted Domain | `200 OK` + Allow-Origin Header |
| **TC-CORS-02** | CORS | ยิง Request จาก Unapproved Domain | `403 Forbidden` / No Allow-Origin |
| **TC-ERR-01** | Error Handler | จำลอง Database Crash / SQL Error | `500 Server Error` (No SQL Leaked) + Trace ID |
| **TC-ERR-02** | Error Handler | ส่ง JSON Payload ผิดรูปแบบ | `400 Bad Request` (Validation Schema) |
| **TC-OTP-01** | Rate Limit | ขอ OTP ติดกัน 3 ครั้งใน 1 นาที (ผ่าน) | `200 OK` (SMS Triggered) |
| **TC-OTP-02** | Rate Limit | ขอ OTP ครั้งที่ 4 ภายใน 15 นาที | `429 Too Many Requests` (No SMS Sent) |
| **TC-OTP-03** | Rate Limit | กรอก OTP ผิดครบ 5 ครั้ง | `423 Locked` / Invalidated |
| **TC-SWG-01** | Swagger | เปิด `/api-docs` บน Production | `404 Not Found` / Auth Prompt |
| **TC-SWG-02** | Swagger | เปิด `/api-docs` บน Development | `200 OK` (Render Swagger UI) |

---

### 4. เกณฑ์ความพร้อมส่งมอบงาน (Definition of Done Checklist)

- [ ] ทุก Test Scenario ผ่านการทดสอบระดับ Automated Unit/Integration Test (Green 100%)
- [ ] ผ่านการทำ Manual Penetration Test เบื้องต้นตาม Scenario ข้างต้น
- [ ] Code ได้รับการ Review และ Approve จาก Tech Lead
- [ ] อัปเดตเอกสาร API และคู่มือการตั้งค่า Environment Variables ครบถ้วน
