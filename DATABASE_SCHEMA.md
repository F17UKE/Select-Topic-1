# DATABASE_SCHEMA_V2.md

**Description for AI Prompts:** 
This document contains the Technical Database Specification for a Hyperlocal Food Delivery Platform integrated with LINE OA. It is designed for a relational database (PostgreSQL). Please read these architectural principles and the schema structure carefully before generating API routes, SQL migrations (e.g., Knex.js/Prisma), or application logic.

---

## 1. Architectural Design Principles

*   **Role & Staff Separation:** แยกข้อมูลบัญชีร้านค้า (`MERCHANTS`) และพนักงาน (`MERCHANT_STAFFS`) ขาดจากกัน โดยพนักงานแต่ละคนมี `role` (เช่น MANAGER, RIDER, KITCHEN) เพื่อจำกัดสิทธิ์การเข้าถึง Web Dashboard และรับงาน
*   **Multiple Addresses:** ลูกค้า 1 คนสามารถมีที่อยู่จัดส่งได้หลายแห่งผ่านตาราง `CUSTOMER_ADDRESSES` พร้อมเก็บ `contact_phone` สำหรับที่อยู่นั้นๆ เพื่อให้ Rider โทรติดต่อผู้รับปลายทางได้ถูกต้อง
*   **Snapshot Pattern (Data Freezing):** แช่แข็งราคา (`unit_price`, `extra_price`) และค่าจัดส่ง (`delivery_fee`) ลงในตาราง Orders เสมอ เพื่อป้องกันปัญหายอดบิลย้อนหลังเพี้ยนเมื่อ Master Data มีการอัปเดตราคา
*   **Delivery Types & Payments:** รองรับออเดอร์ 2 รูปแบบ (`DELIVERY`, `PICKUP`) และช่องทางการชำระเงินที่หลากหลาย (`PROMPTPAY`, `COD`) แยกออกจากสถานะการทำอาหารอย่างชัดเจน
*   **Payment Slip Verification:** บันทึกยอดเงินที่โอนจริง (`amount_transferred`) ควบคู่กับการตรวจสอบสลิป (`is_verified`) เพื่อให้ร้านค้ารีเช็คยอดได้
*   **Order-based In-App Chat:** แชทแยกตามห้อง (`order_id`) พร้อมระบบ 2-Sided Watermark (`customer_last_read_message_id`, `merchant_last_read_message_id`) เพื่อเช็คสถานะการอ่าน
*   **Kitchen Display System (KDS):** ฟิลด์ `is_completed` ใน `ORDER_ITEMS` ช่วยให้หน้าจอในครัวสามารถเช็คสถานะอาหารเสร็จทีละจานได้
*   **LINE Notification Ready:** เก็บ `line_id` สำหรับลูกค้า และ `line_uid` สำหรับฝั่งร้านค้า/พนักงาน เพื่อใช้ยิง Push Message แจ้งเตือนสถานะต่างๆ แบบเรียลไทม์

---

## 2. Entity Relationship Diagram (Mermaid)

```mermaid
erDiagram
    SOIS ||--o{ DORMITORIES : "has"
    SOIS ||--o{ DELIVERY_FEES : "configures"
    MERCHANTS ||--o{ DELIVERY_FEES : "sets"
    MERCHANTS ||--o{ MERCHANT_STAFFS : "employs"
    
    DORMITORIES ||--o{ CUSTOMER_ADDRESSES : "locates"
    CUSTOMERS ||--o{ CUSTOMER_ADDRESSES : "owns"
    CUSTOMER_ADDRESSES ||--o{ ORDERS : "delivers_to"

    CUSTOMERS ||--o{ ORDERS : "places"
    MERCHANTS ||--o{ ORDERS : "fulfills"
    MERCHANTS ||--o{ MENU_ITEMS : "owns"
    MERCHANT_STAFFS ||--o{ ORDERS : "delivers"

    MENU_ITEMS ||--o{ MENU_OPTION_GROUPS : "contains"
    MENU_OPTION_GROUPS ||--o{ MENU_OPTION_CHOICES : "lists"

    ORDERS ||--|{ ORDER_ITEMS : "contains"
    ORDER_ITEMS ||--o{ ORDER_ITEM_CHOICES : "customized_with"
    MENU_OPTION_CHOICES ||--o{ ORDER_ITEM_CHOICES : "references"

    ORDERS ||--o| PAYMENT_SLIPS : "verifies_with"
    ORDERS ||--o{ MESSAGES : "includes"

    SOIS {
        int id PK
        varchar name
    }
    DORMITORIES {
        int id PK
        varchar name
        text location
        int soi_id FK
    }
    CUSTOMERS {
        int id PK
        varchar username UK
        varchar email UK
        varchar password_hash
        varchar google_id UK
        varchar line_id UK
        varchar phone
        varchar profile_image_url
        timestamp created_at
        timestamp updated_at
    }
    CUSTOMER_ADDRESSES {
        int id PK
        int customer_id FK
        int dormitory_id FK
        varchar room_number
        varchar label
        varchar contact_phone
        boolean is_default
        timestamp created_at
        timestamp updated_at
    }
    MERCHANTS {
        int id PK
        varchar username UK
        varchar password_hash
        varchar store_name
        varchar promptpay_id
        varchar prefix
        int last_order_number
        boolean is_open
        text location
        varchar store_image_url
        varchar line_uid
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    MERCHANT_STAFFS {
        int id PK
        varchar username UK
        varchar password_hash
        varchar full_name
        varchar phone
        varchar role
        boolean is_active
        varchar line_uid
        int merchant_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    DELIVERY_FEES {
        int id PK
        decimal fee
        int merchant_id FK
        int soi_id FK
    }
    MENU_ITEMS {
        int id PK
        varchar name
        text description
        decimal price
        boolean is_available
        int stock_quantity
        varchar image_url
        int merchant_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    MENU_OPTION_GROUPS {
        int id PK
        varchar name
        boolean is_required
        int min_choices
        int max_choices
        int menu_item_id FK
    }
    MENU_OPTION_CHOICES {
        int id PK
        varchar name
        decimal extra_price
        int option_group_id FK
    }
    ORDERS {
        int id PK
        varchar order_code UK
        int merchant_order_number
        decimal total_amount
        decimal delivery_fee
        varchar delivery_type
        varchar payment_method
        varchar payment_status
        varchar status
        int customer_last_read_message_id
        int merchant_last_read_message_id
        timestamp created_at
        timestamp updated_at
        int customer_id FK
        int merchant_id FK
        int customer_address_id FK
        int delivery_staff_id FK
    }
    ORDER_ITEMS {
        int id PK
        int quantity
        decimal unit_price
        varchar note
        boolean is_completed
        int order_id FK
        int menu_item_id FK
    }
    ORDER_ITEM_CHOICES {
        int id PK
        varchar choice_name
        decimal extra_price
        int order_item_id FK
        int menu_option_choice_id FK
    }
    PAYMENT_SLIPS {
        int id PK
        varchar image_url
        varchar ref_number UK
        decimal amount_transferred
        boolean is_verified
        timestamp created_at
        timestamp updated_at
        int order_id FK
    }
    MESSAGES {
        int id PK
        boolean is_merchant_sender
        varchar sender_sub_role
        varchar message_type
        text content_text
        int call_duration_seconds
        timestamp created_at
        int order_id FK
    }
