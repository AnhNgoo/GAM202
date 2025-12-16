# HƯỚNG DẪN SETUP HỆ THỐNG UPGRADE

## 📋 TÓM TẮT HỆ THỐNG

### **Đã tạo:**
1. ✅ **UpgradeManager.cs** - Quản lý toàn bộ upgrade logic
2. ✅ **UpgradeUI.cs** - UI hiển thị 3 card khi level up
3. ✅ **Cập nhật PlayerStats** - Tích hợp upgrade UI, EXP = 50/level
4. ✅ **Cập nhật EnemyStats** - Quái mạnh dần + cho nhiều EXP hơn theo thời gian

### **Upgrade Cards có:**
- **Common (Thường)**: +HP, +Mana, +Mana Regen, +Speed, +Damage, +Attack Speed
- **Rare (Hiếm)**: Pierce 1, Lifesteal 5%, Explosion 2m, +15% EXP
- **Epic (Sử thi)**: Pierce 2, Explosion 4m, 15% Crit, Lifesteal 10%
- **Legendary (Huyền thoại)**: Homing, Explosion 6m, 25% Crit x3, Pierce ∞

---

## 🎨 SETUP TRONG UNITY

### **BƯỚC 1: Tạo UpgradeManager GameObject**

1. **Hierarchy** → Create Empty → Đặt tên: `UpgradeManager`
2. **Add Component** → `UpgradeManager`
3. **DontDestroyOnLoad** sẽ tự động

#### **Setup Rarity Curves** (Optional - có default):
- Rare Chance Curve: Level 1 = 5%, Level 20 = 30%
- Epic Chance Curve: Level 1 = 0%, Level 20 = 15%
- Legendary Chance Curve: Level 1 = 0%, Level 20 = 5%

---

### **BƯỚC 2: Tạo Upgrade UI Canvas**

#### **2.1. Tạo Canvas**
```
Hierarchy → UI → Canvas
Tên: UpgradeCanvas
Canvas Scaler: Scale With Screen Size (1920x1080)
Render Mode: Screen Space - Overlay
```

#### **2.2. Tạo Background Panel**
```
UpgradeCanvas → UI → Panel
Tên: UpgradePanel
Anchor: Stretch Full
Color: Semi-transparent black (0,0,0,180)
Add Component: UpgradeUI script
```

#### **2.3. Tạo Cards Container**
```
UpgradePanel → Create Empty
Tên: CardsContainer
Add Component: Horizontal Layout Group
  - Spacing: 30
  - Child Alignment: Middle Center
  - Control Child Size: Width + Height
  - Child Force Expand: Width + Height
```

---

### **BƯỚC 3: Tạo Card Prefab**

#### **3.1. Tạo Card**
```
Hierarchy → UI → Image
Tên: UpgradeCard
Size: 300x400
```

#### **3.2. Thêm các UI elements vào Card:**

**A. Background Image**
- Color: Sẽ đổi theo rarity (script tự động)

**B. Icon (Image)**
```
Position: Top (Y: 120)
Size: 100x100
```

**C. Card Name (TextMeshPro)**
```
Position: (Y: 40)
Font Size: 28
Alignment: Center
Color: White
```

**D. Description (TextMeshPro)**
```
Position: (Y: -30)
Size: 260x150
Font Size: 18
Alignment: Center + Top
Wrapping: Enabled
Color: Light Gray
```

**E. Select Button**
```
UpgradeCard → UI → Button
Tên: SelectButton
Anchor: Bottom (Y: -150)
Size: 250x60
Text: "CHỌN"
```

#### **3.3. Gắn Script vào Card**
- Add Component: `CardUI`
- Kéo các UI elements vào:
  - Name Text → Card Name
  - Description Text → Description
  - Icon Image → Icon
  - Select Button → Select Button

#### **3.4. Tạo Prefab**
- Kéo UpgradeCard vào folder Prefabs
- Xóa khỏi Hierarchy

---

### **BƯỚC 4: Setup UpgradeUI**

**Chọn UpgradePanel**, kéo references vào:
- **Canvas Group**: Add Component → Canvas Group (nếu chưa có)
- **Card Prefab**: Kéo UpgradeCard prefab vào
- **Cards Container**: Kéo CardsContainer vào

**Setup Rarity Colors:**
- Common: (180, 180, 180) - Xám
- Rare: (128, 77, 204) - Tím
- Epic: (204, 153, 51) - Vàng
- Legendary: (230, 51, 51) - Đỏ

---

### **BƯỚC 5: Kết nối với PlayerStats**

1. **Mở Scene Game** (GameScene)
2. **Chọn Player GameObject**
3. **PlayerStats component** → Kéo `UpgradePanel` vào field **Upgrade UI**

---

## 🎮 TEST HỆ THỐNG

### **Test nhanh:**
1. Play game
2. Bấm `K` để tự giết (hoặc giết 5-6 quái để lên level)
3. Sẽ hiện 3 card → Chọn 1 card
4. Kiểm tra upgrade có áp dụng không

### **Test độ hiếm:**
- Level 1: 100% Common
- Level 5: ~10% Rare
- Level 10: ~20% Rare, ~5% Epic
- Level 20: ~30% Rare, ~15% Epic, ~5% Legendary

---

## 📊 CÔNG THỨC MỚI

### **Player:**
- **EXP/Level**: 50 EXP (cố định)
- **Level up**: Hồi đầy HP + Mana + Chọn 1 trong 3 card

### **Enemy:**
- **HP**: `30 + (gameTime * 0.5)` → Sau 60s: 60 HP
- **EXP**: `8-12 + (gameTime * 0.3)` → Sau 60s: 26-30 EXP
- **Với EXP boost 15%**: 30-35 EXP

---

## 🔧 TIẾP THEO CẦN LÀM

### **Cập nhật MagicProjectile để áp dụng upgrades:**
- Pierce (xuyên quái)
- Explosion (vụ nổ)
- Homing (tự động nhắm)
- Crit chance
- Lifesteal (hút máu khi kill)

### **Cập nhật PlayerController:**
- Move Speed upgrade
- Attack Speed upgrade

### **Thêm Visual Effects:**
- Particle khi chọn card
- Border glow theo rarity
- Animation card appear

---

## 🎨 TÙY CHỈNH

### **Thay đổi upgrade values:**
Mở `UpgradeManager.cs` → `InitializeUpgrades()` → Sửa value

### **Thêm upgrade mới:**
1. Thêm vào enum `UpgradeType`
2. Tạo UpgradeCard trong `InitializeUpgrades()`
3. Xử lý trong `UpgradeUI.ApplyUpgradeToPlayer()`

### **Thay đổi tỷ lệ rarity:**
- Sửa Animation Curves trong UpgradeManager Inspector
- Hoặc sửa code trong `InitializeRarityCurves()`

---

## ❗ LƯU Ý

1. **UpgradeManager phải có trong scene đầu tiên** (MainMenu hoặc GameScene)
2. **Card Prefab cần đủ các UI elements** để CardUI script hoạt động
3. **Test kỹ các upgrade** trước khi deploy
4. **Nếu upgrade không work**, check Console có lỗi không
5. **DontDestroyOnLoad** giúp giữ upgrades khi đổi scene

---

## 🐛 TROUBLESHOOTING

**Không hiện Upgrade UI khi level up?**
- Check PlayerStats có gán UpgradeUI chưa
- Check UpgradeManager có trong scene chưa

**Card không có màu?**
- Check Setup Rarity Colors trong UpgradeUI

**Upgrade không áp dụng?**
- Check UpgradeManager.Instance != null
- Check Console có lỗi không

**Quái không mạnh dần?**
- Check EnemyStats có sử dụng OnObjectSpawn() không
- Check hpScalingPerSecond > 0
