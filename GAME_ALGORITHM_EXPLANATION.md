# 📘 GIẢI THÍCH THUẬT TOÁN GAME - GAM202

## 🎯 TỔNG QUAN HỆ THỐNG

Game là một **3D Action RPG** với hệ thống:
- **Player**: Điều khiển nhân vật, bắn đạn, nhảy, tránh né
- **Enemy**: Quái vật tự động tìm đường bằng NavMesh AI, tấn công player
- **Upgrade System**: Hệ thống thẻ bài nâng cấp khi level up
- **Object Pooling**: Tái sử dụng objects (đạn, quái) để tối ưu performance
- **Audio System**: Quản lý âm thanh nhạc nền và hiệu ứng

---

## 📂 CẤU TRÚC HỆ THỐNG

### 🎮 **1. PLAYER SYSTEM (Scripts/Player/)**

#### **PlayerController.cs**
**Mục đích**: Điều khiển di chuyển, xoay, nhảy, bắn đạn của nhân vật

**Thuật toán di chuyển (Move)**:
```
1. Đọc input từ bàn phím (WASD hoặc Arrow Keys)
2. Tạo vector di chuyển từ input (X, Z axis)
3. Nhân vector với currentMoveSpeed (baseMoveSpeed * upgrade multiplier)
4. GIỮ NGUYÊN trục Y của Rigidbody.velocity (để gravity hoạt động đúng)
5. Gán velocity mới cho Rigidbody
```

**Thuật toán xoay theo chuột (LookAtCursor)**:
```
1. Lấy vị trí chuột trên màn hình (Mouse.current.position)
2. Bắn 1 tia Ray từ Camera qua vị trí chuột
3. Tia Ray va chạm với Ground Layer → lấy điểm va chạm (hit.point)
4. ÉP điểm này có cùng độ cao Y với Player (tránh nhân vật ngửa lên trời)
5. Tính vector hướng từ Player đến điểm này
6. Dùng Quaternion.RotateTowards xoay Player theo hướng đó với tốc độ turnSpeed
7. QUAN TRỌNG: Dùng RotateTowards thay vì Slerp để xoay tức thì không delay
```

**Thuật toán nhảy (Jump)**:
```
Điều kiện:
- isGrounded == true (chân đang chạm đất)
- !isPlayerDead (chưa chết)
- !isKnockedBack (không đang bị đẩy lùi)

Thực hiện:
1. Reset velocity trục Y = 0 (xóa vận tốc rơi cũ)
2. Tạo vector lực nhảy = Vector3.up * jumpForce
3. AddForce với mode Impulse (lực tức thời)
4. Kích hoạt animation Jump: animator.SetTrigger(jumpHash)
```

**Thuật toán bắn đạn (HandleAttackInput → CastMagic)**:
```
Kiểm tra điều kiện:
- Không chết
- Đã hết cooldown (Time.time - lastAttackTime >= currentAttackCooldown)
- PlayerStats.UseMana(10f) thành công (đủ mana)

Thực hiện:
1. Kích hoạt animation Attack: animator.SetTrigger(attackHash)
2. Phát âm thanh: AudioManager.Instance.Play("Attack_SFX")
3. Spawn đạn từ ObjectPooler tại firePoint.position
4. Đạn bay theo hướng transform.rotation của Player
```

**Thuật toán Knockback (bị đẩy lùi)**:
```
1. Set cờ isKnockedBack = true (tạm thời vô hiệu hóa Move)
2. Reset velocity = Vector3.zero
3. AddForce với vector forceVector (Impulse mode)
4. Sau 0.2 giây: set isKnockedBack = false (cho phép di chuyển lại)
```

---

#### **PlayerStats.cs**
**Mục đích**: Quản lý HP, Mana, EXP, Level, Kill count

**Thuật toán hồi mana tự động (Update)**:
```
Mỗi frame:
- Nếu currentMana < maxMana:
  1. Tính manaRegenRate thực tế = base * (1 + ManaRegen upgrade multiplier)
  2. currentMana += manaRegenRate * Time.deltaTime
  3. Clamp maxMana để không vượt quá
  4. Cập nhật UI: hud.UpdateMana()
```

**Thuật toán thêm kinh nghiệm (AddExp)**:
```
1. currentExp += amount
2. WHILE currentExp >= maxExp:
   a. Gọi LevelUp()
   b. currentExp -= maxExp (EXP thừa chuyển sang level tiếp)
3. Cập nhật UI thanh EXP
```

**Thuật toán tăng cấp (LevelUp)**:
```
1. currentLevel++
2. maxExp += expIncreasePerLevel (mỗi level cần nhiều EXP hơn)
   VD: Level 1→2: 100 EXP, 2→3: 115 EXP, 3→4: 130 EXP...
3. Hồi đầy HP và Mana
4. Phát âm thanh: AudioManager.Play("LevelUp_SFX")
5. Hiện Upgrade UI: upgradeUI.ShowUpgradeUI(currentLevel)
   → Time.timeScale = 0 (pause game để chọn thẻ)
```

**Thuật toán chết (Die)**:
```
1. Set cờ isDead = true
2. Kích hoạt animation: animator.SetTrigger("Die")
3. Gọi PlayerController.OnDeath() để:
   - Disable input
   - Set velocity = 0
   - Set rigidbody.isKinematic = true (không bị vật lý tác động)
4. Hiện UI Game Over: gameUI.ShowDiePanel()
```

---

#### **MagicProjectile.cs**
**Mục đích**: Đạn ma thuật bay, gây sát thương, có hiệu ứng pierce/explosion/homing

**Thuật toán khởi tạo đạn (OnObjectSpawn)**:
```
1. Reset trạng thái:
   - hasHit = false
   - hitEnemies.Clear() (danh sách enemy đã trúng)
   - Enable Collider

2. Tính Pierce Count:
   - pierceCount = (int)UpgradeManager.GetUpgradeValue(ProjectilePierce)
   VD: Upgrade Pierce 2 lần → pierceCount = 2 → xuyên 2 con

3. Tính Final Damage:
   - Gọi CalculateDamage() để tính sát thương cuối

4. Kiểm tra Homing:
   - Nếu có upgrade Homing > 0:
     → Set isHoming = true
     → FindHomingTarget() (tìm enemy gần nhất)

5. Bắt đầu Coroutine: AutoDisableAfterLifetime() (tự động tắt sau X giây)
```

**Thuật toán tính sát thương (CalculateDamage)**:
```
1. finalDamage = baseDamage

2. Áp dụng ProjectileDamage Multiplier:
   damageMultiplier = 1 + UpgradeManager.GetUpgradeValue(ProjectileDamage)
   VD: Upgrade 3 lần 15% → multiplier = 1.45
   finalDamage *= damageMultiplier

3. Kiểm tra Critical Hit:
   critChance = UpgradeManager.GetUpgradeValue(CriticalChance)
   Nếu Random.value <= critChance:
     a. critDamage = UpgradeManager.GetUpgradeValue(CriticalDamage)
     b. finalDamage *= (2.0 + critDamage)
        VD: CritDamage = 0.5 → nhân 2.5x damage
     c. Log: "CRITICAL HIT!"
```

**Thuật toán Homing (tự động bám mục tiêu)**:
```
Trong Update():
1. Nếu isHoming && homingTarget != null:
   a. Tính vector hướng: direction = (target.position - transform.position).normalized
   b. Tính rotation mới: Quaternion.LookRotation(direction)
   c. Xoay dần về hướng đó: Quaternion.Slerp với homingStrength
   → Đạn từ từ uốn cong theo enemy

2. Di chuyển thẳng: transform.Translate(Vector3.forward * speed * Time.deltaTime)
```

**Thuật toán va chạm (OnTriggerEnter)**:
```
NẾU va chạm với Enemy:
  1. Kiểm tra hitEnemies.Contains(enemy) → Nếu có rồi thì return (tránh trúng lại)
  2. Gây sát thương: enemyStats.TakeDamage(finalDamage)
  3. Thêm enemy vào hitEnemies list
  
  4. Kiểm tra Explosion:
     - Nếu có upgrade Explosion > 0:
       → ExplodeArea(vị trí va chạm, bán kính explosion)
       → Tất cả enemies trong bán kính nhận 50% damage
  
  5. Kiểm tra Lifesteal:
     - Nếu có upgrade Lifesteal > 0:
       → Hồi máu cho player = maxHealth * lifestealValue
       VD: Lifesteal 10% → hồi 10% maxHP
  
  6. Kiểm tra Pierce:
     - Nếu pierceCount > 0:
       → pierceCount--
       → RETURN (không destroy, tiếp tục bay)
     - Nếu pierceCount == 0:
       → HitTarget() (destroy đạn)

NẾU va chạm với Ground:
  → HitTarget() (destroy ngay)
```

---

### 👻 **2. ENEMY SYSTEM (Scripts/Enemy/)**

#### **GhostEnemy.cs**
**Mục đích**: AI quái vật tự động đuổi theo player, tấn công khi đến gần

**Thuật toán khởi tạo (OnObjectSpawn)**:
```
1. Reset trạng thái:
   - isDead = false
   - isKnockedBack = false
   - lastAttackTime = -999f
   - lastPathUpdateTime = 0f

2. Reset Components:
   - Enable Collider
   - Rigidbody: set isKinematic = true, clear velocity
   - NavMeshAgent: enable, reset path
   - Kiểm tra agent.isOnNavMesh, nếu không thì Warp về vị trí hợp lệ

3. Tính scaling theo player level:
   playerLevel = PlayerStats.currentLevel
   
   Tốc độ:
   - speed = baseSpeed * (1 + (playerLevel - 1) * speedScalingPerLevel)
   VD: Level 10, baseSpeed=3.5, scaling=10% → speed = 3.5 * 1.9 = 6.65
   
   Sát thương:
   - damage = baseDamage * (1 + (playerLevel - 1) * damageScalingPerLevel)
   VD: Level 10, baseDamage=10, scaling=12% → damage = 10 * 2.08 = 20.8
   
   → Level càng cao, quái càng nhanh và đau!

4. Gán speed cho NavMeshAgent.speed
```

**Thuật toán AI (Update)**:
```
1. Kiểm tra điều kiện:
   - Nếu isDead hoặc isKnockedBack → return (không làm gì)

2. Cập nhật đường đi (mỗi 0.2 giây):
   - Nếu Time.time - lastPathUpdateTime >= pathUpdateInterval:
     a. agent.SetDestination(playerTarget.position)
        → NavMesh tính đường đi mới
     b. lastPathUpdateTime = Time.time
   
   QUAN TRỌNG: Update destination liên tục 0.2s/lần
   → Enemy luôn đuổi theo vị trí MỚI NHẤT của player
   → Không bị "chạy về vị trí cũ" như trước

3. Kiểm tra khoảng cách tấn công:
   distance = Vector3.Distance(transform.position, playerTarget.position)
   
   Nếu distance <= attackRange:
     a. Kiểm tra cooldown: Time.time - lastAttackTime >= attackCooldown
     b. Nếu đủ cooldown:
        - agent.isStopped = true (dừng di chuyển)
        - animator.SetTrigger(attackHash) (chơi animation đấm)
        - lastAttackTime = Time.time
   
   Nếu distance > attackRange:
     - agent.isStopped = false (tiếp tục đuổi)
```

**Thuật toán tấn công (DealDamage - được gọi từ Animation Event)**:
```
1. Tính lại khoảng cách hiện tại
   → Tránh gây damage khi player đã chạy xa

2. Nếu distance <= attackRange:
   a. Phát âm thanh: AudioManager.Play("EnemyAttack_SFX")
   
   b. Gây sát thương: targetStats.TakeDamage(damage)
   
   c. Knockback Player:
      - Tính vector đẩy: direction * pushPlayerForce
      - targetController.ApplyKnockback(pushForce)
      → Player bị hất xa ra
   
   d. Spawn hiệu ứng:
      - Instantiate(playerHitEffectPrefab) tại playerHitPoint
```

**Thuật toán nhận sát thương (TakeDamageFromPlayer)**:
```
1. Nếu isDead → return
2. Chơi animation: animator.SetTrigger(takeDamageHash)
3. StartCoroutine(Knockback):
   a. Set isKnockedBack = true
   b. Disable NavMeshAgent tạm thời
   c. Set rigidbody.isKinematic = false (cho phép vật lý)
   d. AddForce với direction * knockbackForce
   e. Sau knockbackDuration (0.5s):
      - Dừng velocity
      - Set isKinematic = true lại
      - Enable NavMeshAgent
      - isKnockedBack = false
```

**LƯU Ý QUAN TRỌNG - Fix lỗi console warning**:
```
Trước khi set rb.velocity:
- PHẢI kiểm tra !rb.isKinematic
- Kinematic rigidbody KHÔNG thể set velocity (NavMeshAgent điều khiển)
- Chỉ set velocity khi đã chuyển sang non-kinematic mode
```

---

#### **EnemyStats.cs**
**Mục đích**: Quản lý HP, thanh máu, tính EXP thưởng khi chết

**Thuật toán khởi tạo HP (OnObjectSpawn)**:
```
1. Tính maxHP dựa vào 2 yếu tố:
   
   a. Scaling theo thời gian:
      gameTime = Time.timeSinceLevelLoad
      timeScaling = gameTime * hpScalingPerSecond
      VD: 60 giây * 0.5 = +30 HP
   
   b. Scaling theo player level:
      playerLevel = PlayerStats.currentLevel
      levelScaling = baseMaxHP * (1 + (playerLevel - 1) * hpScalingPerPlayerLevel)
      VD: Level 10, base=30, scaling=15% → 30 * 2.35 = 70.5 HP
   
   c. maxHP = levelScaling + timeScaling
   
   → Càng chơi lâu + level cao = quái càng trâu!

2. currentHP = maxHP (hồi đầy máu)
3. Enable Collider
4. Cập nhật UI thanh máu
```

**Thuật toán nhận sát thương (TakeDamage)**:
```
1. Nếu isDead → return
2. currentHP -= amount
3. Cập nhật UI: hpSlider.value = currentHP / maxHP
4. Nếu currentHP <= 0:
   → Gọi Die()
```

**Thuật toán chết (Die)**:
```
1. Set isDead = true
2. Ẩn thanh máu: hpSlider.SetActive(false)

3. Tính EXP thưởng:
   a. gameTime = Time.timeSinceLevelLoad
   b. minExp = baseMinExpReward + (gameTime * expScalingPerSecond)
   c. maxExp = baseMaxExpReward + (gameTime * expScalingPerSecond)
   d. randomExp = Random.Range(minExp, maxExp)
   
   e. Áp dụng EXP Boost upgrade:
      expBoost = UpgradeManager.GetUpgradeValue(ExperienceBoost)
      randomExp *= (1 + expBoost)
   
   f. Cho EXP và Kill cho player:
      - playerStats.AddExp(randomExp)
      - playerStats.AddKill()

4. Invoke("DisableObject", 2f)
   → Sau 2 giây tắt object (trả về pool)
   → Thời gian này để animation chết chạy xong
```

---

#### **EnemySpawner.cs**
**Mục đích**: Tự động spawn quái theo thời gian, khó dần theo level

**Thuật toán spawn rate động (UpdateSpawnInterval)**:
```
Công thức:
currentSpawnInterval = baseSpawnInterval - (playerLevel - 1) * intervalDecreasePerLevel

Clamp giá trị: currentSpawnInterval = Mathf.Max(currentSpawnInterval, minSpawnInterval)

Ví dụ:
- Level 1: 4.0s (chậm, dễ chơi)
- Level 5: 4.0 - (4 * 0.15) = 3.4s
- Level 10: 4.0 - (9 * 0.15) = 2.65s
- Level 20: 4.0 - (19 * 0.15) = 1.15s → Clamp về 1.0s (nhanh nhất)

→ Càng level cao, spawn càng dày, game càng khó!
```

**Thuật toán spawn quái (TrySpawnEnemy)**:
```
1. Chọn điểm spawn ngẫu nhiên:
   a. Lấy hướng random: Random.insideUnitCircle.normalized
   b. Lấy khoảng cách random: Random.Range(minSpawnDistance, maxSpawnDistance)
      VD: Spawn cách player từ 10m đến 20m (không quá gần, không quá xa)
   c. spawnPos = playerPosition + hướng * khoảng cách

2. Kiểm tra điểm có hợp lệ trên NavMesh không:
   NavMesh.SamplePosition(spawnPos, out hit, navMeshCheckRadius, NavMesh.AllAreas)
   
   - Nếu tìm thấy → hit.position là điểm hợp lệ
   - Nếu không → Bỏ qua lần này (tránh spawn trong tường)

3. Spawn cảnh báo:
   a. SpawnWarning(hit.position)
   b. ObjectPooler spawn dấu X cảnh báo
   c. Dấu X nhấp nháy 1.5 giây
   d. Sau đó spawn quái thật tại vị trí đó
```

---

#### **ObjectPooler.cs**
**Mục đích**: Tái sử dụng objects (đạn, quái, effects) thay vì Instantiate/Destroy

**Thuật toán khởi tạo pools (InitializePools)**:
```
1. Clear dictionary cũ
2. Xóa tất cả children cũ của ObjectPooler (nếu có)

3. Với mỗi Pool trong list:
   a. Tạo Queue mới
   b. Loop từ 0 đến size:
      - Instantiate prefab
      - SetActive(false)
      - SetParent vào ObjectPooler
      - Enqueue vào Queue
   c. Thêm Pool vào dictionary với key = tag

VD: Pool "Ghost" với size = 10 → Tạo sẵn 10 con quái ẩn
```

**Thuật toán spawn object (SpawnFromPool)**:
```
1. Kiểm tra dictionary có tag không → Nếu không có thì return null
2. Dequeue object từ Queue
3. SetActive(true)
4. Set position và rotation
5. Gọi IPooledObject.OnObjectSpawn() (nếu có) để reset trạng thái
6. Enqueue lại vào Queue (để tái sử dụng)
7. Return object

→ Không Instantiate mới → Tiết kiệm CPU và RAM!
```

**Thuật toán reset (ResetAllPools)**:
```
Với mỗi pool trong dictionary:
  - Dequeue từng object
  - SetActive(false)
  - Enqueue lại
→ Tắt tất cả objects đang active, trả về pool
```

---

#### **WarningEffect.cs**
**Mục đích**: Hiệu ứng cảnh báo nhấp nháy trước khi quái spawn

**Thuật toán nhấp nháy (WarningRoutine)**:
```
1. timer = 0
2. WHILE timer < duration (1.5 giây):
   a. timer += Time.deltaTime
   
   b. Tính scaleFactor (dùng Sin wave):
      scaleFactor = 0.5 + Abs(Sin(timer * blinkSpeed))
      → Sin dao động từ -1 đến 1
      → Abs → 0 đến 1
      → +0.5 → 0.5 đến 1.5
      
   c. transform.localScale = initialScale * scaleFactor
      → Dấu X phồng to thu nhỏ liên tục (nhấp nháy)
   
   d. yield return null (chờ frame tiếp)

3. Sau khi hết thời gian:
   a. Reset scale về gốc
   b. SpawnEnemy() tại vị trí này
   c. SetActive(false) (trả về pool)
```

---

### 🔊 **3. AUDIO SYSTEM (Scripts/Music/)**

#### **Sound.cs**
**Mục đích**: Data class chứa thông tin 1 âm thanh

**Cấu trúc**:
```
- name: Tên gọi (VD: "BGM_Menu", "Attack_SFX")
- clip: File AudioClip
- type: Music hoặc SFX (để điều chỉnh volume riêng)
- volume: Độ to (0→1)
- pitch: Độ cao (0.1→3, mặc định 1)
- loop: Có lặp lại không (Music = true, SFX = false)
- source: AudioSource component (được tạo runtime)
```

---

#### **AudioManager.cs**
**Mục đích**: Singleton quản lý tất cả âm thanh trong game

**Thuật toán khởi tạo (Awake)**:
```
1. Singleton pattern:
   - Nếu Instance == null → Instance = this, DontDestroyOnLoad
   - Nếu Instance != null → Destroy(this) (tránh trùng)

2. Với mỗi Sound trong list:
   a. Tạo AudioSource component mới
   b. Gán clip, volume, pitch, loop
   c. Lưu vào sound.source
```

**Thuật toán phát âm thanh (Play)**:
```
1. Tìm Sound có name khớp trong list
2. Nếu không tìm thấy → Log error, return
3. Nếu sound.source.isPlaying → return (tránh phát trùng)
4. sound.source.Play()
```

**Thuật toán dừng âm thanh (Stop)**:
```
1. Tìm Sound có name khớp
2. Nếu không tìm thấy → return
3. sound.source.Stop()
```

**Thuật toán điều chỉnh volume (SetMusicVolume / SetSFXVolume)**:
```
1. Lưu multiplier:
   - musicVolumeMultiplier = value (0→1)
   - sfxVolumeMultiplier = value (0→1)

2. UpdateAllVolumes(type):
   a. Với mỗi Sound có type khớp:
      - source.volume = sound.volume * multiplier
      
   VD: Sound có volume = 0.8, slider = 0.5
   → Âm lượng thực tế = 0.8 * 0.5 = 0.4
```

**Cách sử dụng**:
```csharp
// Phát âm thanh
AudioManager.Instance.Play("Attack_SFX");

// Dừng nhạc nền
AudioManager.Instance.Stop("BGM_Game");

// Điều chỉnh volume (từ SettingUI)
AudioManager.Instance.SetMusicVolume(0.5f); // 50%
```

---

### 🎴 **4. UPGRADE SYSTEM (Scripts/UI/)**

#### **UpgradeCardDatabase.cs**
**Mục đích**: ScriptableObject lưu trữ icons và borders của thẻ

**Cấu trúc**:
```
- upgradeIcons: List<UpgradeTypeIcon>
  → Mỗi UpgradeType có 1 icon riêng
  
- commonBorder, rareBorder, epicBorder, legendaryBorder
  → Mỗi rarity có sprite + màu viền + màu nền riêng
  
- defaultIcon: Icon mặc định nếu không tìm thấy
```

**Thuật toán lấy icon (GetIcon)**:
```
1. Loop qua upgradeIcons list
2. Tìm item có upgradeType khớp
3. Return item.icon
4. Nếu không tìm thấy → return defaultIcon
```

**Thuật toán lấy border (GetBorder)**:
```
Switch rarity:
  - Common → return commonBorder
  - Rare → return rareBorder
  - Epic → return epicBorder
  - Legendary → return legendaryBorder
```

---

#### **UpgradeManager.cs**
**Mục đích**: Singleton quản lý tất cả upgrade cards và logic rarity

**Thuật toán khởi tạo thẻ (InitializeUpgrades)**:
```
1. allUpgrades.Clear()

2. Tạo 32 thẻ:
   - 16 Common: MaxHealth, MaxMana, MoveSpeed, Damage, AttackSpeed...
   - 9 Rare: Pierce, Explosion, Critical Chance, Lifesteal...
   - 4 Epic: Homing, Mana Regen Large, EXP Boost...
   - 7 Legendary: Infinite Pierce, Homing Pro, Mana Infinity...
   
3. AssignWeights():
   - Common: weight = 10.0
   - Rare: weight = 4.0
   - Epic: weight = 1.5
   - Legendary: weight = 0.3
   
   ĐẶC BIỆT: Legendary cards có power modifiers:
   - "Infinite Pierce" → weight *= 0.2 (siêu hiếm)
   - "Homing" → weight *= 0.3
   → Đảm bảo skill mạnh xuất hiện cực kỳ ít

4. AssignIconsFromDatabase():
   - Với mỗi card:
     → card.icon = cardDatabase.GetIcon(card.upgradeType)
```

**Thuật toán chọn thẻ ngẫu nhiên (GetRandomCards)**:
```
1. Tạo list kết quả (3 thẻ)

2. Loop 3 lần:
   a. Random rarity dựa vào level:
      - rarity = RandomRarityByLevel(playerLevel)
   
   b. Lọc list thẻ có rarity khớp:
      - filteredCards = allUpgrades.Where(c => c.rarity == rarity)
   
   c. Tính tổng trọng số:
      - totalWeight = Sum(filteredCards.Select(c => c.weight))
      VD: 3 thẻ Common weight 10 → totalWeight = 30
   
   d. Random weighted:
      - random = Random.Range(0, totalWeight)
      - accumulator = 0
      - FOREACH card:
          accumulator += card.weight
          NẾU random <= accumulator:
            → Chọn card này, break
      
      VD: random = 15, weights [10, 10, 10]
      → Loop 1: acc=10, 15>10, continue
      → Loop 2: acc=20, 15<=20, chọn card 2

3. Return list 3 thẻ
```

**Thuật toán random rarity (RandomRarityByLevel)**:
```
1. Tính tỉ lệ % dựa vào level:
   - rareChance = Lerp(rareChanceLevel1, rareChanceLevel20, (level-1)/19)
   - epicChance = Lerp(epicChanceLevel1, epicChanceLevel20, (level-1)/19)
   - legendaryChance = Lerp(legendaryChanceLevel1, legendaryChanceLevel20, (level-1)/19)
   
   VD Level 10:
   - rareChance = Lerp(0.05, 0.25, 9/19) = 0.14 (14%)
   - epicChance = Lerp(0.01, 0.15, 9/19) = 0.07 (7%)
   - legendaryChance = Lerp(0, 0.05, 9/19) = 0.024 (2.4%)

2. Roll xác suất:
   random = Random.value (0→1)
   
   - Nếu random <= legendaryChance → Legendary
   - Else nếu random <= epicChance → Epic
   - Else nếu random <= rareChance → Rare
   - Else → Common

→ Level càng cao, tỉ lệ thẻ hiếm càng tăng!
```

---

#### **UpgradeUI.cs**
**Mục đích**: Hiển thị 3 thẻ lên UI, xử lý chọn thẻ

**Thuật toán hiện UI (ShowUpgradeUI)**:
```
1. ClearCards() (xóa thẻ cũ nếu có)

2. Lấy 3 thẻ ngẫu nhiên:
   cards = UpgradeManager.Instance.GetRandomCards(playerLevel, 3)

3. Với mỗi card:
   - CreateCardUI(card)
   - Instantiate cardPrefab
   - Setup CardUI component
   - Hiển thị tên, mô tả, icon, border

4. Pause game:
   - Time.timeScale = 0
   - Cursor.lockState = None (hiện chuột)
   - canvasGroup.alpha = 1 (hiện UI)

5. Nâng sortingOrder lên cao nhất để UI không bị che
```

**Thuật toán chọn thẻ (OnCardSelected)**:
```
1. UpgradeManager.AddUpgrade(card.upgradeType, card.value)
   → Cộng dồn giá trị upgrade vào dictionary

2. ApplyUpgradeToPlayer(card):
   Switch card.upgradeType:
     - MaxHealth: stats.maxHealth += value, Heal(value)
     - MaxMana: stats.maxMana += value, stats.currentMana += value
     - ManaRegen: stats.manaRegenRate += value (đã được lưu trong Manager)
     - MoveSpeed, AttackSpeed: Được áp dụng tự động trong PlayerController
     - ProjectileDamage, Pierce, Explosion... → Áp dụng trong MagicProjectile
     
3. Phát âm thanh: AudioManager.Play("CardSelect_SFX")

4. Ẩn UI:
   - Time.timeScale = 1 (tiếp tục game)
   - canvasGroup.alpha = 0
   - ClearCards()
```

---

#### **CardUI.cs**
**Mục đích**: Component trên mỗi card prefab, hiển thị thông tin

**Thuật toán setup (Setup)**:
```
1. Lưu tham chiếu: cardData = card, upgradeUI = ui

2. Hiển thị thông tin:
   - nameText.text = card.cardName
   - descriptionText.text = card.description
   - iconImage.sprite = card.icon

3. Gán sự kiện nút:
   - selectButton.onClick.AddListener(OnClick)

4. ApplyRarityVisuals(card.rarity):
   a. Lấy database từ UpgradeManager
   b. rarityData = database.GetBorder(rarity)
   c. borderImage.sprite = rarityData.borderSprite
   d. borderImage.color = rarityData.borderColor
   e. backgroundImage.color = rarityData.backgroundColor
   
   → Thẻ Common = khung xám, Rare = khung tím, Epic = vàng, Legendary = đỏ
```

---

### 🎨 **5. UI SYSTEM (Scripts/UI/)**

#### **Billboard.cs**
**Mục đích**: Làm UI (thanh máu enemy) luôn quay về phía Camera

**Thuật toán (LateUpdate)**:
```
1. Nếu camTransform == null → return
2. Lấy góc Y của camera: camTransform.eulerAngles.y
3. Set rotation của UI:
   - X = 90° (nằm ngửa lên trời, song song với mặt đất)
   - Y = góc Y của camera (xoay theo camera để chữ không bị ngược)
   - Z = 0° (không nghiêng)
   
→ UI luôn "nhìn" về phía player!
```

---

#### **GameUIController.cs**
**Mục đích**: Quản lý UI trong game (Pause, Setting, Die Panel)

**Thuật toán pause (PauseGame)**:
```
1. isPaused = true
2. Time.timeScale = 0 (đóng băng game)
3. Cursor.lockState = None, Cursor.visible = true
4. SetDefaultCursor() (con trỏ mặc định Windows)
5. settingPanel.SetActive(true)
```

**Thuật toán resume (ResumeGame)**:
```
1. isPaused = false
2. Time.timeScale = 1 (game chạy lại)
3. SetCustomCursor() (con trỏ custom của game)
4. settingPanel.SetActive(false)
```

**Thuật toán hiện bảng chết (ShowDiePanel)**:
```
1. Nếu isGameOver → return (tránh gọi nhiều lần)
2. isGameOver = true
3. diePanel.SetActive(true)
4. Gọi DieUI.Show():
   - Chờ delayShow (2 giây) để xem animation chết
   - Sau đó: Time.timeScale = 0, hiện chuột
```

---

#### **DieUI.cs**
**Mục đích**: Panel Game Over với nút Retry và Menu

**Thuật toán (ShowRoutine)**:
```
1. yield WaitForSeconds(delayShow) → Chờ 2 giây
2. Time.timeScale = 0 (pause game)
3. Cursor.lockState = None, visible = true
4. SetCursor(null) (con trỏ mặc định)
```

**Nút Retry**:
```
1. Time.timeScale = 1 (khôi phục thời gian)
2. AudioManager.Play("Button_SFX")
3. diePanel.SetActive(false)
4. LoadingPanel.LoadScene(SceneManager.GetActiveScene().name)
   → Reload scene hiện tại
```

**Nút Menu**:
```
1. Time.timeScale = 1
2. AudioManager: Play("Button_SFX"), Stop("BGM_Game")
3. diePanel.SetActive(false)
4. LoadingPanel.LoadScene("MainMenu")
```

---

#### **LoadingPanel.cs**
**Mục đích**: Màn hình loading với thanh tiến độ mượt mà

**Thuật toán loading (LoadSceneCoroutine)**:
```
1. Hiện loading panel:
   - canvasGroup.alpha = 1
   - canvasGroup.blocksRaycasts = true

2. Bắt đầu load scene:
   operation = SceneManager.LoadSceneAsync(sceneName)
   operation.allowSceneActivation = false (chờ load xong mới chuyển)

3. Loop cho đến khi load xong:
   WHILE !operation.isDone:
     a. targetProgress = operation.progress / 0.9
        (Unity progress chỉ đến 0.9, chia cho 0.9 để thành 0→1)
     
     b. Lerp currentProgress về targetProgress:
        currentProgress = Mathf.MoveTowards(current, target, fillSpeed * Time.deltaTime)
        → Thanh chạy mượt, không giật lag
     
     c. UpdateUI(currentProgress):
        - loadingSlider.value = progress
        - loadingText.text = (progress * 100).ToString("F0") + "%"
     
     d. Khi operation.progress >= 0.9:
        - Nếu useFakeProgress:
          → Chạy fake progress từ 0.99 → 1.0 với tốc độ chậm
          → Tránh loading bar dừng đột ngột ở 90%
        
        - currentProgress >= 0.995:
          → operation.allowSceneActivation = true
          → Chuyển scene
     
     e. yield return null (chờ frame tiếp)

4. Chờ thêm postLoadDelay (0.3s) cho mượt
5. Ẩn loading panel: canvasGroup.alpha = 0
```

---

#### **PlayerHUD.cs**
**Mục đích**: Hiển thị thanh HP, Mana, EXP, Level, Kills

**Các hàm update UI**:
```
UpdateHealth(current, max):
  - healthSlider.maxValue = max
  - healthSlider.value = current

UpdateMana(current, max):
  - manaSlider.maxValue = max
  - manaSlider.value = current

UpdateExp(current, max):
  - expSlider.maxValue = max
  - expSlider.value = current

UpdateLevelInfo(level, kills):
  - levelText.text = "Lv. " + level
  - killText.text = "Kills: " + kills
```

---

#### **SettingUI.cs**
**Mục đích**: Panel Setting với slider điều chỉnh volume

**Thuật toán khởi tạo (Start)**:
```
1. Đọc giá trị đã lưu từ PlayerPrefs:
   - savedMusic = PlayerPrefs.GetFloat("MusicVolume", 1f)
   - savedSFX = PlayerPrefs.GetFloat("SFXVolume", 1f)

2. Gán vào slider:
   - musicSlider.value = savedMusic
   - sfxSlider.value = savedSFX

3. Cập nhật AudioManager ngay:
   - UpdateMusic(savedMusic)
   - UpdateSFX(savedSFX)

4. Đăng ký sự kiện:
   - musicSlider.onValueChanged.AddListener(OnMusicVolumeChanged)
   - sfxSlider.onValueChanged.AddListener(OnSFXVolumeChanged)
```

**Thuật toán thay đổi volume**:
```
OnMusicVolumeChanged(value):
  1. UpdateMusic(value):
     → AudioManager.Instance.SetMusicVolume(value)
  
  2. Lưu vào PlayerPrefs:
     - PlayerPrefs.SetFloat("MusicVolume", value)
     - PlayerPrefs.Save()

OnSFXVolumeChanged(value):
  1. UpdateSFX(value):
     → AudioManager.Instance.SetSFXVolume(value)
  
  2. Lưu vào PlayerPrefs:
     - PlayerPrefs.SetFloat("SFXVolume", value)
     - PlayerPrefs.Save()
```

---

#### **UIController.cs** (Main Menu)
**Mục đích**: Điều khiển UI ở màn hình chính

**Thuật toán khởi tạo (Start)**:
```
1. ShowMainPanel() (hiện Main, ẩn Setting)
2. AudioManager.Instance.Play("BGM_Menu")
   → Phát nhạc nền menu
```

**Nút Setting**:
```
1. AudioManager.Play("Button_SFX")
2. mainPanel.SetActive(false)
3. settingPanel.SetActive(true)
```

**Nút Exit**:
```
1. AudioManager.Play("Button_SFX")
2. Application.Quit() (thoát game)
```

---

#### **PlayUI.cs**
**Mục đích**: Xử lý nút Play với cinematic camera transition

**Thuật toán (PlaySequence)**:
```
1. uiController.HideAllPanels() (ẩn tất cả UI)

2. Kích hoạt intro camera:
   - introCamera.Priority = 100 (cao hơn main camera)
   → Camera bay theo path đẹp mắt

3. Chờ cameraMoveDuration (2 giây)

4. Fade to black:
   - nightPanelCanvasGroup.alpha từ 0 → 1
   - Thời gian: fadeDuration (1 giây)

5. Chờ thêm 0.2s để player thấy fade

6. LoadingPanel.LoadScene("GameScene")
```

---

## 🔄 LƯU ĐỒ TỔNG QUÁT

### **Game Loop**:
```
1. Player di chuyển, bắn đạn, tránh né
2. Enemy spawn liên tục, đuổi theo player
3. Đạn bay, va chạm enemy → Gây sát thương
4. Enemy chết → Cho EXP
5. Player đủ EXP → Level up → Chọn thẻ nâng cấp
6. Game khó dần (spawn nhanh hơn, enemy mạnh hơn)
7. Player chết → Game Over → Retry hoặc về Menu
```

### **Mối quan hệ giữa các class**:
```
PlayerController
  ├─ Điều khiển di chuyển, xoay, nhảy
  ├─ Dùng PlayerStats để check mana, trigger upgrade UI
  ├─ Spawn đạn qua ObjectPooler
  └─ Dùng AudioManager để phát âm thanh

PlayerStats
  ├─ Quản lý HP, Mana, EXP, Level
  ├─ Gọi UpgradeUI khi level up
  ├─ Gọi GameUIController khi chết
  └─ Dùng PlayerHUD để update UI

MagicProjectile
  ├─ Đọc upgrades từ UpgradeManager
  ├─ Tính damage, pierce, explosion, homing
  ├─ Gây sát thương cho EnemyStats
  └─ Dùng AudioManager để phát âm thanh

GhostEnemy
  ├─ AI tìm đường bằng NavMeshAgent
  ├─ Đọc PlayerStats để scale theo level
  ├─ Gây sát thương cho PlayerStats
  └─ Dùng AudioManager để phát âm thanh

EnemyStats
  ├─ Quản lý HP của enemy
  ├─ Cho EXP cho PlayerStats khi chết
  ├─ Scale HP theo level và thời gian
  └─ Kiểm tra EXP boost từ UpgradeManager

EnemySpawner
  ├─ Spawn enemy theo interval
  ├─ Đọc PlayerStats để scale spawn rate
  ├─ Dùng ObjectPooler để spawn Warning + Enemy
  └─ Dùng NavMesh để check điểm spawn hợp lệ

ObjectPooler (Singleton)
  ├─ Quản lý pools cho đạn, quái, effects
  ├─ Tránh Instantiate/Destroy → Tối ưu performance
  └─ Gọi IPooledObject.OnObjectSpawn() để reset

UpgradeManager (Singleton)
  ├─ Quản lý 32 upgrade cards
  ├─ Weight system cho rarity
  ├─ Random thẻ dựa vào level
  ├─ Lưu upgrades trong Dictionary
  └─ Dùng UpgradeCardDatabase để lấy icons/borders

UpgradeCardDatabase (ScriptableObject)
  ├─ Lưu trữ icons cho 13 UpgradeTypes
  ├─ Lưu trữ borders cho 4 rarities
  └─ Được UpgradeManager và CardUI sử dụng

UpgradeUI
  ├─ Hiện 3 thẻ ngẫu nhiên khi level up
  ├─ Pause game (Time.timeScale = 0)
  ├─ Apply upgrade khi chọn thẻ
  └─ Resume game sau khi chọn

AudioManager (Singleton)
  ├─ DontDestroyOnLoad (tồn tại xuyên scenes)
  ├─ Quản lý tất cả âm thanh (Music + SFX)
  ├─ Điều chỉnh volume riêng cho từng loại
  └─ Được mọi script khác gọi để phát âm thanh

Sound (Data Class)
  ├─ Chứa thông tin 1 âm thanh
  ├─ AudioManager tạo AudioSource cho mỗi Sound
  └─ Không có logic, chỉ là container

GameUIController
  ├─ Quản lý Pause/Resume game
  ├─ Hiện Setting Panel
  ├─ Hiện Die Panel khi player chết
  └─ Điều khiển cursor (custom/default)

LoadingPanel (Singleton)
  ├─ DontDestroyOnLoad
  ├─ Loading screen với thanh tiến độ mượt
  ├─ Fake progress để UX đẹp hơn
  └─ Được gọi từ mọi nơi cần chuyển scene
```

---

## 💡 CÁC KỸ THUẬT QUAN TRỌNG

### **1. Singleton Pattern**
```csharp
// Đảm bảo chỉ có 1 instance duy nhất
public static AudioManager Instance;

void Awake() {
    if (Instance == null) {
        Instance = this;
        DontDestroyOnLoad(gameObject); // Tồn tại xuyên scenes
    } else {
        Destroy(gameObject); // Destroy instance trùng
    }
}
```

**Sử dụng**: AudioManager, UpgradeManager, ObjectPooler, LoadingPanel

---

### **2. Object Pooling**
```csharp
// Tái sử dụng objects thay vì Instantiate/Destroy

// Khởi tạo pool
for (int i = 0; i < size; i++) {
    GameObject obj = Instantiate(prefab);
    obj.SetActive(false);
    pool.Enqueue(obj);
}

// Spawn
GameObject obj = pool.Dequeue();
obj.SetActive(true);
pool.Enqueue(obj); // Thêm lại vào cuối hàng đợi

// Disable (trả về pool)
obj.SetActive(false);
```

**Lợi ích**: Tăng FPS, giảm lag, tối ưu memory

---

### **3. NavMesh AI**
```csharp
// AI tự động tìm đường đến player

// Cập nhật destination liên tục
if (Time.time - lastPathUpdateTime >= 0.2f) {
    agent.SetDestination(playerTarget.position);
    lastPathUpdateTime = Time.time;
}

// NavMesh tự động:
// - Tránh chướng ngại vật
// - Đi vòng nếu bị chặn
// - Tính đường ngắn nhất
```

---

### **4. Weighted Random**
```csharp
// Random có trọng số (thẻ hiếm xuất hiện ít hơn)

float totalWeight = cards.Sum(c => c.weight);
float random = Random.Range(0, totalWeight);
float accumulator = 0;

foreach (var card in cards) {
    accumulator += card.weight;
    if (random <= accumulator) {
        return card; // Chọn thẻ này
    }
}
```

**Ví dụ**: Cards [A:10, B:4, C:1] → totalWeight = 15
- Random 0-10 → Chọn A (66.7%)
- Random 10-14 → Chọn B (26.7%)
- Random 14-15 → Chọn C (6.6%)

---

### **5. Lerp/Curve Scaling**
```csharp
// Tăng dần tỉ lệ thẻ hiếm theo level

float t = (playerLevel - 1) / 19f; // Level 1→20 thành 0→1
float rareChance = Mathf.Lerp(0.05f, 0.25f, t);

// Level 1: 5%
// Level 10: 14.5%
// Level 20: 25%
```

---

### **6. Coroutine**
```csharp
// Chạy code theo thời gian không block main thread

IEnumerator MyCoroutine() {
    // Chờ 2 giây
    yield return new WaitForSeconds(2f);
    
    // Chờ đến frame tiếp
    yield return null;
    
    // Chờ đến khi điều kiện đúng
    yield return new WaitUntil(() => isDone);
}

// Gọi
StartCoroutine(MyCoroutine());
```

---

### **7. Animation Events**
```csharp
// Gọi hàm từ Animation (không cần code check frame)

// Trong Animation Clip:
// - Frame 15: Event "DealDamage"

// Trong Script:
public void DealDamage() {
    // Code gây sát thương
}

// Unity tự động gọi hàm này ở frame 15 của animation
```

---

### **8. ScriptableObject Database**
```csharp
// Lưu trữ data dạng asset (không cần Resources folder)

[CreateAssetMenu(fileName = "Database", menuName = "Game/Database")]
public class Database : ScriptableObject {
    public List<Item> items;
}

// Tạo asset: Right Click → Create → Game → Database
// Kéo vào Inspector → Truy cập trong code
```

**Lợi ích**: Dễ quản lý, không lỗi đường dẫn, Inspector-friendly

---

## 🎓 KẾT LUẬN

Hệ thống game được thiết kế theo các nguyên tắc:

1. **Modular**: Mỗi script làm 1 nhiệm vụ rõ ràng
2. **Scalable**: Dễ dàng thêm upgrade, enemy, mechanic mới
3. **Optimized**: Object Pooling, NavMesh, minimal Instantiate/Destroy
4. **User-friendly**: Inspector controls, không cần code để balance
5. **Clean Code**: Singleton, ScriptableObject, Design Pattern

Tất cả hệ thống phối hợp tạo thành gameplay loop:
```
Fight → Kill → EXP → Level Up → Choose Upgrade → Stronger → Harder Enemies → Repeat
```

**Chúc bạn code tốt!** 🚀
