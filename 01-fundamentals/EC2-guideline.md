# EC2 Deployment Guideline

## Environment Variables và Case Sensitivity

### Vấn đề thường gặp

Khi deploy từ Windows Local lên AWS EC2 (Ubuntu/Linux), một lỗi phổ biến là:

```env
# Trong .env
ENVIRONMENT_AUTO=True
```

```js
// Trong code
if (process.env.ENVIRONMENT_AUTO === "true") {
  // Luôn fail vì "True" !== "true"
}
```

**Giải thích kỹ thuật:**

- Environment Variables luôn là `string`, không phụ thuộc OS
- Linux case-sensitive: `"True"`, `"true"`, `"TRUE"` là 3 chuỗi khác nhau
- Lỗi xảy ra do code so sánh case-sensitive, KHÔNG phải Linux "parse" khác Windows

### Best Practice cho Environment Variables

```
# Đúng - lowercase cho boolean
ENVIRONMENT_AUTO=true
ENABLE_CACHE=false
DEBUG_MODE=false

# Đúng - uppercase cho other values
NODE_ENV=production
API_PORT=3000
REGION=ap-southeast-1
```

### Xử lý Boolean đúng cách

```js
// Utility function
function parseBoolean(value) {
  return String(value).toLowerCase() === "true";
}

// Sử dụng
const autoEnabled = parseBoolean(process.env.ENVIRONMENT_AUTO);

// Hoặc dùng thư viện validation (Zod example)
import { z } from 'zod';

const configSchema = z.object({
  ENVIRONMENT_AUTO: z.coerce.boolean().default(false),
});
```

## File Path và Case Sensitivity

### Common Error

```js
// Code
require('./Config');

// File thực tế trên Linux
config.js
```

Windows: chạy được (case-insensitive filesystem)

Linux: `Error: Cannot find module './Config'`

### Checklist khi deploy

- [ ] Tên file trong code = tên file thực tế (chính xác về chữ hoa/thường)
- [ ] Thư mục upload có đúng tên (uploads/ vs Uploads/)
- [ ] Line ending đúng: LF (Linux) thay vì CRLF (Windows)

## Bảo mật và AWS Best Practices

### Tránh lưu secrets trong file .env trên EC2

Thay vì:
```
EC2 -> đọc file .env trực tiếp
```

Nên dùng:
```
EC2 -> IAM Role -> AWS Parameter Store / Secrets Manager
```

### AWS Services phù hợp

| Use Case | Service |
|----------|---------|
| DB Password, API Keys | AWS Secrets Manager |
| Config flags, feature toggles | AWS Systems Manager Parameter Store |
| Static config values | Parameter Store (Secure String) |

## Common Deploy Errors

| Error | Nguyên nhân | Cách fix |
|-------|-------------|----------|
| `Cannot find module` | Sai tên file (case) | Check tên file chính xác |
| `/bin/bash^M: not found` | CRLF line ending | Convert sang LF: `dos2unix` |
| `Permission denied` | Thiếu execute permission | `chmod +x script.sh` |
| Logic sai | String case mismatch | Normalize với `.toLowerCase()` |

## Tổng kết

1. **Environment Variables luôn là string** - không có auto-boolean conversion
2. **Linux case-sensitive** - kiểm tra kỹ tên file và giá trị biến
3. **Không lưu secrets trong .env Production** - dùng Parameter Store/Secrets Manager
4. **Test trên Linux trước khi deploy** - để phát hiện line ending, path issues
