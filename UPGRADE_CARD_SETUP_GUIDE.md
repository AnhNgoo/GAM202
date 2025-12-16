# 📚 HƯỚNG DẪN SETUP UPGRADE CARD SYSTEM

## 🎯 Tổng Quan
Hệ thống mới sử dụng **ScriptableObject** để quản lý icons và borders của upgrade cards, thay vì dùng Resources. Đơn giản hơn, dễ quản lý hơn, và linh hoạt hơn!

---

## 📋 BƯỚC 1: Tạo Upgrade Card Database

### 1.1. Tạo ScriptableObject Asset
1. Trong Unity, chuột phải vào thư mục `Assets/_Data/` (hoặc thư mục bạn muốn)
2. Chọn: **Create > Game > Upgrade Card Database**
3. Đặt tên: `UpgradeCardDatabase`

### 1.2. Gán Icons cho Upgrade Types
Trong Inspector của `UpgradeCardDatabase`:

#### **Upgrade Icons List:**
Thêm các item vào list `Upgrade Icons`, mỗi item gồm:
- **Upgrade Type**: Chọn loại upgrade (MaxHealth, ProjectileDamage, v.v.)
- **Icon**: Kéo sprite icon tương ứng vào

**Ví dụ setup:**
```
[0] MaxHealth      → Icon: heart_icon.png
[1] MaxMana        → Icon: mana_icon.png
[2] MoveSpeed      → Icon: speed_icon.png
[3] ProjectileDamage → Icon: damage_icon.png
[4] ProjectileHoming → Icon: homing_icon.png
... (tất cả 13 loại UpgradeType)
```

### 1.3. Gán Borders cho Rarity
Trong Inspector của `UpgradeCardDatabase`:

#### **Common Border:**
- **Border Sprite**: Khung viền màu xám/trắng
- **Border Color**: Màu viền (ví dụ: #AAAAAA)
- **Background Color**: Màu nền (ví dụ: #FFFFFF)

#### **Rare Border:**
- **Border Sprite**: Khung viền màu tím
- **Border Color**: Màu tím (#8A2BE2)
- **Background Color**: Nền tím nhạt

#### **Epic Border:**
- **Border Sprite**: Khung viền màu vàng cam
- **Border Color**: Màu vàng (#FFD700)
- **Background Color**: Nền vàng nhạt

#### **Legendary Border:**
- **Border Sprite**: Khung viền màu đỏ/vàng rực rỡ
- **Border Color**: Màu đỏ (#FF4500)
- **Background Color**: Nền đỏ nhạt

#### **Default Icon:**
- Gán 1 icon mặc định (nếu không tìm thấy icon nào)

---

## 📋 BƯỚC 2: Gán Database vào UpgradeManager

1. Trong Hierarchy, tìm GameObject có script `UpgradeManager`
2. Trong Inspector, tìm field **Card Database**
3. Kéo file `UpgradeCardDatabase` (vừa tạo ở bước 1) vào field này

✅ **XONG!** Hệ thống sẽ tự động load icons từ database.

---

## 📋 BƯỚC 3: Setup Card UI Prefab

Nếu Card Prefab chưa có Border/Background Images:

### 3.1. Mở Card Prefab
Trong Project, tìm prefab của card (thường ở `Prefabs/UI/`)

### 3.2. Thêm Border Image
1. Chuột phải vào Card GameObject > **UI > Image**
2. Đặt tên: `BorderImage`
3. Kéo vào field **Border Image** trong script `CardUI`

### 3.3. Thêm Background Image (optional)
1. Chuột phải vào Card GameObject > **UI > Image**
2. Đặt tên: `BackgroundImage`
3. Đặt phía sau icon (Order in Layer thấp hơn)
4. Kéo vào field **Background Image** trong script `CardUI`

---

## 🎨 THIẾT KẾ SPRITES

### Icon Sprites
- **Kích thước khuyến nghị**: 128x128 hoặc 256x256
- **Format**: PNG với alpha channel
- **Style**: Icon đơn giản, rõ ràng
- Đặt trong thư mục: `Assets/Art/UI/Icons/Upgrades/`

### Border Sprites
- **Kích thước**: Tùy theo thiết kế card UI của bạn
- **Format**: PNG với alpha channel
- **Style**: Khung viền theo rarity (Common → Legendary)
- Đặt trong thư mục: `Assets/Art/UI/Borders/`

---

## ✅ KIỂM TRA HOẠT ĐỘNG

### Test trong Game:
1. Chạy game
2. Khi level up, mở upgrade UI
3. Kiểm tra:
   - ✓ Icons hiển thị đúng cho mỗi skill
   - ✓ Border/khung viền đúng màu theo độ hiếm
   - ✓ Không có warning trong Console

### Console Check:
Nếu thấy warning:
```
[UpgradeManager] Card Database chưa được gán!
```
→ Quay lại Bước 2, gán database vào UpgradeManager

---

## 🔧 SỬA LỖI THƯỜNG GẶP

### Lỗi 1: Icons không hiển thị
**Nguyên nhân**: Chưa gán icon trong Database
**Giải pháp**: Mở UpgradeCardDatabase, thêm đầy đủ 13 UpgradeType icons

### Lỗi 2: Border không đổi màu
**Nguyên nhân**: Chưa gán Border/Background Image trong Card Prefab
**Giải pháp**: Xem lại Bước 3

### Lỗi 3: Tất cả thẻ dùng default icon
**Nguyên nhân**: UpgradeType trong Database không khớp với UpgradeType của card
**Giải pháp**: Kiểm tra lại tên UpgradeType trong Database list

---

## 💡 LỢI ÍCH CỦA SCRIPTABLEOBJECT

✅ **Đơn giản**: Không cần setup thư mục Resources
✅ **Trực quan**: Kéo thả sprite trong Inspector
✅ **Linh hoạt**: Dễ dàng thay đổi icons/borders
✅ **An toàn**: Không lo lỗi đường dẫn Resources
✅ **Hiệu suất**: Load nhanh hơn Resources.Load()

---

## 📝 GHI CHÚ

- **Database có thể tái sử dụng**: Tạo nhiều database khác nhau cho các theme khác nhau
- **Icons có thể share**: Nhiều UpgradeType có thể dùng chung 1 icon
- **Borders có thể customize**: Tùy chỉnh màu sắc cho phù hợp với art style

---

## 🎮 KẾT QUẢ

Sau khi setup xong:
- ✨ Mỗi skill có icon riêng, đẹp mắt
- 🎨 Border đổi màu theo độ hiếm (Common → Legendary)
- 🚀 Hệ thống hoạt động mượt mà, dễ bảo trì
- 💯 Các skill cũ vẫn hoạt động bình thường

**Chúc bạn thành công!** 🎉
