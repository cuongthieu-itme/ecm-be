# ECM Backend - B2C Ecommerce API

Backend API cho ứng dụng Ecommerce B2C, xây dựng trên NestJS 11 + Prisma 7 + MySQL.

## Tech Stack

- **Framework:** NestJS 11
- **ORM:** Prisma 7 (với `@prisma/adapter-mariadb`)
- **Database:** MySQL
- **Auth:** JWT + Passport
- **Docs:** Swagger (OpenAPI)

## Yêu cầu

- Node.js >= 18
- MySQL >= 8.0
- npm >= 9

## Cài đặt và chạy dự án

### 1. Clone và cài dependencies

```bash
git clone <repo-url>
cd ecm-be
npm install
```

### 2. Cấu hình environment

Copy file `.env.example` thành `.env` và cập nhật thông tin kết nối:

```bash
cp .env.example .env
```

Nội dung `.env`:

```env
DATABASE_URL="mysql://root:password@localhost:3306/ecm_db"

JWT_SECRET="your-jwt-secret-key"
JWT_EXPIRES_IN="15m"
JWT_REFRESH_EXPIRES_IN="7d"

PORT=3000
```

### 3. Tạo database

Tạo database MySQL trước khi chạy migration:

```sql
CREATE DATABASE ecm_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 4. Chạy Prisma migration

```bash
npx prisma migrate dev --name init
```

Lệnh này sẽ:
- Tạo tất cả bảng trong database theo schema
- Generate Prisma Client

### 5. Chạy server

```bash
# Development (hot-reload)
npm run start:dev

# Production
npm run build
npm run start:prod
```

### 6. Truy cập

- **API:** http://localhost:3000/api/v1
- **Swagger Docs:** http://localhost:3000/api/docs

## Cấu trúc dự án

```
src/
├── main.ts                     # Bootstrap, Swagger, ValidationPipe
├── app.module.ts               # Root module
├── prisma/                     # PrismaService (global)
├── common/                     # Decorators, Guards, Filters, Interceptors, DTOs
├── auth/                       # Register, Login, Refresh, Logout
├── user/                       # Profile, Admin user list
├── category/                   # CRUD categories
├── product/                    # CRUD products (soft delete, search, filter)
├── cart/                       # Cart management (upsert items)
└── order/                      # Order creation (transaction), status management
```

## API Endpoints

| Module   | Method | Path                     | Auth  |
|----------|--------|--------------------------|-------|
| Auth     | POST   | /auth/register           | -     |
| Auth     | POST   | /auth/login              | -     |
| Auth     | POST   | /auth/refresh            | -     |
| Auth     | POST   | /auth/logout             | JWT   |
| Users    | GET    | /users/me                | JWT   |
| Users    | GET    | /users                   | ADMIN |
| Category | GET    | /categories              | -     |
| Category | GET    | /categories/:id          | -     |
| Category | POST   | /categories              | ADMIN |
| Category | PATCH  | /categories/:id          | ADMIN |
| Category | DELETE | /categories/:id          | ADMIN |
| Product  | GET    | /products                | -     |
| Product  | GET    | /products/:id            | -     |
| Product  | POST   | /products                | ADMIN |
| Product  | PATCH  | /products/:id            | ADMIN |
| Product  | DELETE | /products/:id            | ADMIN |
| Cart     | GET    | /cart                    | JWT   |
| Cart     | POST   | /cart/items              | JWT   |
| Cart     | PATCH  | /cart/items/:id          | JWT   |
| Cart     | DELETE | /cart/items/:id          | JWT   |
| Cart     | DELETE | /cart                    | JWT   |
| Order    | POST   | /orders                  | JWT   |
| Order    | GET    | /orders                  | JWT   |
| Order    | GET    | /orders/:id              | JWT   |
| Order    | GET    | /orders/admin/all        | ADMIN |
| Order    | PATCH  | /orders/:id/status       | ADMIN |

## Quy trình Prisma

### Prisma 7 - Lưu ý quan trọng

Prisma 7 có thay đổi lớn so với các phiên bản trước:

- **Không dùng `url` trong `schema.prisma`** - URL kết nối database chỉ được cấu hình trong `prisma.config.ts`
- **Bắt buộc dùng Driver Adapter** - PrismaClient cần truyền adapter trong constructor (ví dụ: `@prisma/adapter-mariadb` cho MySQL)
- **Import path thay đổi** - Import từ `generated/prisma/client.js` thay vì `@prisma/client`

### Thêm bảng mới / Cập nhật bảng

**Bước 1:** Sửa file `prisma/schema.prisma`

```prisma
model NewTable {
  id        Int      @id @default(autoincrement())
  name      String   @db.VarChar(255)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("new_tables")  // tên bảng trong DB dùng snake_case
}
```

**Bước 2:** Tạo migration

```bash
# Tạo migration mới (tự đặt tên mô tả thay đổi)
npx prisma migrate dev --name add_new_table
```

**Bước 3:** Generate lại Prisma Client (nếu chỉ cần generate mà không migrate)

```bash
npx prisma generate
```

**Bước 4:** Sử dụng trong code

```typescript
// Import model types
import { PrismaClient, NewTable } from '../../generated/prisma/client.js';

// Sử dụng qua PrismaService (đã inject sẵn)
const items = await this.prisma.newTable.findMany();
```

### Quy ước đặt tên trong schema

| TypeScript (camelCase) | Database (snake_case) | Cách ánh xạ |
|------------------------|-----------------------|-------------|
| `createdAt`            | `created_at`          | `@map("created_at")` |
| `userId`               | `user_id`             | `@map("user_id")` |
| `CartItem` (model)     | `cart_items` (table)  | `@@map("cart_items")` |

### Các lệnh Prisma thường dùng

```bash
# Tạo migration mới
npx prisma migrate dev --name <ten_migration>

# Generate Prisma Client (không migration)
npx prisma generate

# Xem trạng thái migration
npx prisma migrate status

# Reset database (xóa toàn bộ data + chạy lại migration)
npx prisma migrate reset

# Mở Prisma Studio (GUI xem data)
npx prisma studio

# Format schema file
npx prisma format
```

### Sửa bảng đã có

```prisma
model Product {
  // Thêm field mới
  rating Float? @default(0)

  // Thêm index mới
  @@index([rating])
}
```

Sau đó chạy:

```bash
npx prisma migrate dev --name add_rating_to_product
```

### Thêm quan hệ giữa các bảng

```prisma
model Review {
  id        Int     @id @default(autoincrement())
  content   String  @db.Text
  rating    Int
  userId    Int     @map("user_id")
  productId Int     @map("product_id")

  user    User    @relation(fields: [userId], references: [id])
  product Product @relation(fields: [productId], references: [id])

  @@map("reviews")
}

// Nhớ thêm relation ngược ở model User và Product:
// reviews Review[]
```

## Scripts

```bash
npm run build         # Build TypeScript
npm run start:dev     # Chạy dev (hot-reload)
npm run start:prod    # Chạy production
npm run lint          # Lint code
npm run format        # Format code
```
