# เอกสารข้อกำหนดความต้องการของระบบ (Software Requirements Specification - SRS)
## โครงการ: Backend API Security & Core Hardening MVP (กลุ่ม 1)

---

### 1. บทนำและภาพรวม (Introduction)

เอกสารฉบับนี้ระบุข้อกำหนดความต้องการทั้งเชิงหน้าที่ (Functional Requirements) และเชิงคุณภาพ (Non-Functional Requirements) สำหรับระบบ **Backend API Security & Core Hardening MVP** ของกลุ่ม 1 เพื่อใช้เป็นพิมพ์เขียวในการพัฒนาและการทดสอบระบบ

---

### 2. ข้อกำหนดความต้องการเชิงหน้าที่ (Functional Requirements - FR)

```
+-----------------------------------------------------------------------------------+
|                         Functional Requirements Map                               |
+-----------------------------------------------------------------------------------+
| [FR-01] JWT Authentication           --> ตรวจสอบ Token ในทุก Protected API        |
| [FR-02] RBAC Authorization           --> ตรวจสอบ Role ของผู้ใช้ก่อนอนุญาตเข้าถึง   |
| [FR-03] Data Export Lockdown         --> จำกัด Route Export ให้เฉพาะ ADMIN         |
| [FR-04] CORS Domain Whitelist        --> อนุญาตเฉพาะ Origin ที่อยู่ใน Whitelist    |
| [FR-05] CORS Preflight Handling      --> รองรับ OPTIONS และส่ง Security Headers    |
| [FR-06] Global Error Masking         --> ซ่อน Raw SQL / Stack Trace ตอบกลับ Generic |
| [FR-07] Trace ID Correlation         --> แนบ Unique trace_id ใน Response และ Log  |
| [FR-08] OTP Request Rate Limiting    --> จำกัดการขอ OTP 3 ครั้ง / 15 นาที / เบอร์   |
| [FR-09] OTP Verification Lockout     --> ผิดเกิน 5 ครั้งล็อก Transaction, TTL 5 นาที |
| [FR-10] Swagger Production Shield    --> ปิด/ล็อก Swagger UI และ Spec บน Production|
+-----------------------------------------------------------------------------------+
```

#### รายละเอียดข้อกำหนดเชิงหน้าที่:

| รหัส | หมวดหมู่ | ชื่อข้อกำหนด | รายละเอียด | ระดับความสำคัญ |
| :--- | :--- | :--- | :--- | :---: |
| **FR-01** | Auth | JWT Authentication | ระบบต้องตรวจสอบความถูกต้องของ JWT Access Token ใน HTTP Header `Authorization: Bearer <token>` หาก Token ไม่ถูกต้องหรือหมดอายุ ต้องปฏิเสธด้วย `401 Unauthorized` | Must Have |
| **FR-02** | Auth | Role-Based Access Control (RBAC) | ระบบต้องตรวจสอบสิทธิ์ของผู้ใช้งานตาม Role (เช่น `ADMIN`, `OFFICER`, `USER`) ก่อนอนุญาตให้เรียกใช้ Resource | Must Have |
| **FR-03** | Auth | Data Export Restriction | Endpoint สำหรับ Export รายงานข้อมูล (เช่น `/api/v1/reports/export/*`) ต้องจำกัดสิทธิ์เฉพาะ Role `ADMIN` เท่านั้น หาก Role อื่นเรียกใช้ต้องตอบกลับ `403 Forbidden` | Must Have |
| **FR-04** | Security | Strict CORS Whitelist | ระบบต้องอ่านค่า Origin Header และอนุญาตเฉพาะ Domain ที่ระบุใน Whitelist Config (เช่น Web App Production/Staging) | Must Have |
| **FR-05** | Security | Preflight `OPTIONS` Request | ระบบต้องรองรับ HTTP `OPTIONS` สำหรับ Preflight Requests พร้อมตอบกลับ Header ที่ปลอดภัย เช่น `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers` | Must Have |
| **FR-06** | Exception | Global Error Masking | เมื่อเกิด Error ภายในระบบ (เช่น Database Crash, SQL Error, Null Pointer) ต้องแปลงเป็น Generic Message ห้ามส่ง SQL Query หรือ Table Schema คืนแก่ Client | Must Have |
| **FR-07** | Logging | Trace ID Correlation | ระบบต้องสร้าง Unique `trace_id` (UUID v4) ให้กับทุก Request และแนบไปใน Response JSON และ Server Log เพื่อใช้สอบสวนข้อผิดพลาด | Must Have |
| **FR-08** | Anti-Abuse| OTP Request Throttling | จำกัดการขอ OTP สูงสุด 3 ครั้ง ต่อช่วงเวลา 15 นาที ต่อ 1 เบอร์โทรศัพท์ และต่อ 1 Client IP Address เพื่อป้องกัน SMS Bombing | Must Have |
| **FR-09** | Anti-Abuse| OTP Verification Lockout | ตรวจสอบรหัส OTP หากกรอกผิดเกิน 5 ครั้ง ให้ยกเลิก Transaction นั้นทันที และ OTP แต่ละรหัสต้องมีอายุไม่เกิน 5 นาที (TTL) | Must Have |
| **FR-10** | Hardening | Swagger Shielding | ปิดการแสดงผล Swagger UI (`/swagger`, `/docs`) และ OpenAPI JSON (`/openapi.json`) ใน Production Environment หากจำเป็นต้องเปิดต้องผ่าน Basic Auth | Must Have |

---

### 3. ข้อกำหนดความต้องการเชิงคุณภาพ (Non-Functional Requirements - NFR)

#### 3.1 ด้านความมั่นคงปลอดภัย (Security)
* **NFR-SEC-01 (Password & OTP Hashing):** รหัสผ่านต้องเข้ารหัสด้วย `bcrypt` (Cost $\ge$ 10) หรือ `Argon2` และรหัส OTP ใน Database ต้องจัดเก็บในรูปแบบ Hash (SHA-256 with Salt หรือ bcrypt)
* **NFR-SEC-02 (Secrets Management):** JWT Secret Key, Database Credentials, และ API Keys ต้องจัดเก็บใน Environment Variables และห้าม Commit ลง Git Repository โดยเด็ดขาด
* **NFR-SEC-03 (Token Expiration):** Access Token ต้องมีอายุไม่เกิน 15-30 นาที และ Refresh Token มีอายุไม่เกิน 7 วัน

#### 3.2 ด้านประสิทธิภาพ (Performance & Scalability)
* **NFR-PERF-01 (Middleware Overhead):** Middleware ทั้งหมด (Auth, CORS, Rate Limit, Error Handler) ต้องเพิ่ม Latency ไม่เกิน 10 มิลลิวินาที (ms) ต่อ Request
* **NFR-PERF-02 (Rate Limit Evaluation):** การตรวจสอบ Rate Limiter ผ่าน In-Memory Cache หรือ Redis ต้องใช้เวลาประมวลผลน้อยกว่า 2 มิลลิวินาที (ms)

#### 3.3 ด้านความพร้อมใช้งานและความเสถียร (Reliability & Availability)
* **NFR-REL-01 (Graceful Degradation):** กรณีระบบ Caching หรือ Rate Limiter ภายนอกขัดข้อง ระบบต้องมี Fallback Mechanism ไม่ให้ส่งผลให้ API หลักหยุดทำงาน
* **NFR-REL-02 (Standardized Error Schema):** Error Response ทุกประเภทต้องมีโครงสร้าง JSON ที่สม่ำเสมอ:
  ```json
  {
    "success": false,
    "error": {
      "code": "ERROR_CODE_NAME",
      "message": "User friendly message",
      "trace_id": "c8b4f14a-67a3-481d-8f92-5b967dc8a421"
    },
    "timestamp": "2026-08-16T11:20:00.000Z"
  }
  ```

#### 3.4 ด้านการบำรุงรักษาและการตรวจสอบ (Maintainability & Auditability)
* **NFR-MAINT-01 (Audit Trail):** ทุกการกระทำที่สำคัญ (Security Events, Export Data, Failed Logins) ต้องบันทึกลง Audit Log พร้อม `trace_id` และ `user_id`
* **NFR-MAINT-02 (Code Quality):** โค้ด Middleware และ Security Layer ต้องผ่าน Unit Test Coverage ไม่น้อยกว่า 80%

---

### 4. ตารางความสัมพันธ์ระหว่างข้อกำหนดและโมดูล (Traceability Matrix)

| โมดูล (Module) | Functional Requirements | Non-Functional Requirements |
| :--- | :--- | :--- |
| **Module 1: JWT & RBAC** | FR-01, FR-02, FR-03 | NFR-SEC-01, NFR-SEC-02, NFR-SEC-03, NFR-PERF-01 |
| **Module 2: CORS Policy** | FR-04, FR-05 | NFR-SEC-02, NFR-PERF-01 |
| **Module 3: Error Handler** | FR-06, FR-07 | NFR-REL-02, NFR-MAINT-01 |
| **Module 4: OTP Rate Limiter** | FR-08, FR-09 | NFR-SEC-01, NFR-PERF-02, NFR-REL-01 |
| **Module 5: Swagger Shield** | FR-10 | NFR-SEC-02 |
