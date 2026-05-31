# 开发规范 (Development Guidelines)

## 项目结构

```
Rogue/
├── assets/bgm/             # BGM 音频（当前禁用，文件保留）
├── abyss-labyrinth.html    # 游戏主文件（~5400行，HTML+CSS+JS）
├── README.md               # 项目说明
├── CHANGELOG.md            # 版本日志
├── DEVELOPMENT.md          # 开发规范（本文件）
└── temp/                   # 临时开发文件（脚本/生成器）
```

---

## 代码架构（8 大模块）

```
MODULE 1: 核心配置与状态 (CONFIG + gameState)
  - CONFIG: 地图/敌人/装备/品质/词缀/套装/职业/天赋/精英/升级选项
  - gameState: 初始状态 + restartGame 重置

MODULE 2: 音频引擎
  - SFX 18种音效
  - BGM 已禁用（代码注释保留，assets/bgm/ 文件保留）

MODULE 3: 地图系统
  │  - 城堡大厅 / 地城生成 / 特殊房间（宝藏室/竞技场/祭坛/图书馆） / 渲染 / 缩放

MODULE 4: 玩家与战斗
  - 状态栏(updateStatusBar) / 楼层信息 / 移动(movePlayer) / 特效
  - 战斗(combat): Boss单回合 / 普通敌人自动/手动
  - hasEliteAbility / handleDotEffects / applyCombatEffects
  - 新机制: 穿透/连击/减伤/护盾/冰冻/眩晕/点燃/脆弱

MODULE 5: 装备与商店
  - 升级(checkLevelUp/applyLevelUpChoice) / 装备生成(generateEquipment)
  - 宝箱(openChest) / 装备对比弹窗(showEquipCompare)
  - 商店(openShop/buyItem)

MODULE 6: 天赋系统
  - 18天赋 + Q键主动技能 + tickSkillTurns/tickSkillCooldown

MODULE 7: UI 与生命周期
  - 游戏结束(gameOver) / 重新开始(backToStart/restartGame)
  - 角色面板(openCharSheet/toggleCharSheet)
  - 升级弹窗(showLevelUpModal) / 装备对比弹窗

MODULE 8: 入口与初始化
  - 键盘控制(keydown handler) / 开场动画(playIntroAnimation)
  - 职业选择 / 启动(initGame/startGame)
```

---

## 开发流程

### 每次改动必须同步
1. **更新代码** — 修改 HTML/CSS/JS
2. **语法验证** — `node -e "..."` 检查
3. **更新 3 份文档**:
   - CHANGELOG.md: 新版本号 + 分类记录改动
   - README.md: 版本号 + 修改历史 + 相关功能说明
   - DEVELOPMENT.md: 版本号 + 架构/陷阱更新

### 新增功能标准流程
1. CONFIG 区 — 添加配置常量
2. gameState + restartGame — 两处添加字段
3. CSS — 按功能块添加样式
4. HTML — 添加 UI 结构
5. JS 逻辑 — 对应 MODULE 区添加函数
6. 集成 — 挂接到现有系统
7. 验证 — node 语法检查

---

## 常见陷阱

### 1. gameState 两处声明
新字段必须在 `let gameState = {...}` 和 `restartGame()` 两处添加。

### 2. 变量作用域：else 块内的变量外部不可访问
`generateMap()` 中 `enemyPool` 在 `else` 块内定义。精英/商店/药水等代码若需访问 `enemyPool`，必须放在同一个 else 块内，不能放在 `}` 之后。

### 3. 战斗函数多分支
`combat()` 内 Boss(单回合) 和普通敌人(auto/manual) 有三段平行代码。改攻击/防御/死亡逻辑时需全部同步。

### 4. 手动模式(✋)下普通敌人走 Boss 路径
当 `autoAttack=false` 时，普通敌人进入 Boss 单回合代码块。必须在死亡检查处用 `if (!enemy.isBoss)` 分支处理普通敌人（含精英怪装备掉落）。

### 5. 弹窗回调异步
`showEquipCompare` 和 `showLevelUpModal` 是异步的（用户点击才回调）。传送门揭示/afterKillEffects/tickSkillTurns 必须放在回调内，不能放在弹窗调用之后。

### 6. 装备 HP 与治疗上限
所有治疗/恢复上限用 `player.maxHp + sumEquipHp()`，包括：药水/商店药水/楼层恢复/嗜血/吸血/升级满血。

### 8. TALENT_TREES 结构：树对象 ≠ 天赋数组
`CONFIG.TALENT_TREES` 的值是 `{name, icon, talents:[...]}` 对象，不是直接数组。遍历天赋时用 `tree.talents` 或 `tree.talents || []`，不可直接 `for (const t of tree)`。

### 9. 城堡层 theme='castle' 不在 CONFIG.THEMES 中
城堡使用独立的 `gameState.castleTheme` 对象。进入地城前必须重置 `gameState.theme = 'abyss'`，否则 `generateMap()` 访问 `CONFIG.THEMES['castle']` 返回 undefined 导致崩溃。同理，`openCharSheet` 等处取主题需用 `CONFIG.THEMES[theme] || gameState.castleTheme` 兜底。

### 10. 特殊事件 triggerEventNPC 需手动追踪 action
随机事件NPC的特殊事件（楼层跳转、精英怪生成等）需用 `specialAction` 变量追踪，在函数末尾统一处理地图重生成（楼层变更）和渲染刷新。

### 12. 成就系统 localStorage 命名空间
成就数据存储在 `localStorage['rogue_achievements']`，包含 `achievements`（已解锁id映射）和 `stats`（计数器）。重启游戏时 `initAchievements()` 恢复。避免与其他 localStorage key 冲突。

### 11. 精灵模式 inline style 与 CSS `!important` 的优先级
地图格子有内联 `style="color:#xxx"`，精灵模式的 `.sprite-cell { color:transparent !important }` 无法覆盖。需在 `spriteStyle` 末尾追加 `color:transparent;`。另外 `font-size:0` 会让 `width:1.3em` 变0，需改固定像素。

### 7. DOM null 检查
keydown 处理器中 `introEl` 和 `startScr` 必须 null-check，否则元素缺失时整个按键系统崩溃。

### 13. 特殊房间状态管理
特殊房间使用 `gameState.specialRoom = { type, state: {...} }`。新增特殊房间类型时：
- `generateMap()` 中判定后调用专用生成器并 return
- `renderMap()` 中渲染房间标记（金币/祭坛/图书馆/竞技场出口）
- `movePlayer()` 中添加交互逻辑（金币拾取/祭坛弹窗/图书馆弹窗）
- 竞技场通过 `afterKillEffects()` 中的 `checkArenaClear()` 检查波次
- 祭坛/图书馆弹窗需同步在 keyboard 守卫 + gameOver 清理 + ESC 处理三处添加

### 14. 环境效果范围
环境效果（毒雾/黑暗/魔力紊乱）仅出现在普通层。Boss层/商店层/特殊房间不会触发。
- 毒雾：每步伤害必须放在 `movePlayer()` 成功移动之后、`sfx.play('walk')` 之前，且需检查 HP≤0 → gameOver
- 黑暗：`renderMap()` 中曼哈顿距离>4的格子设为 dark-fog。注意保留墙壁可见
- 魔力紊乱：`tickSkillCooldown()` 中用 `Math.max(0, cd - ticks)` 防止负数
- 环境效果在 `generateMap()` 的 `isShopFloor` 判定之后设置，在下一层时 `generateMap()` 会重新判定并覆盖

### 15. 城堡装饰物碰撞
装饰物（火炬🔥/石柱🏛️/旗帜🚩/王座👑/吊灯🕯️）存储在 `gameState.castleDecorations` 数组中。`movePlayer()` 城堡段在 NPC 检查之前遍历装饰物数组做碰撞检测。注意此检查必须在墙壁检查之后、NPC交互之前，因为装饰物虽不可穿越但不影响 NPC 交互的优先级。红毯 ◆ 走 `castleCarpet` 数组，仅影响渲染不阻挡移动。

---

## 调试技巧

### 语法验证
```bash
node -e "const m=require('fs').readFileSync('/Users/xiaoyu/Desktop/Rogue/abyss-labyrinth.html','utf8').match(/<script>([\\s\\S]*?)<\/script>/);new Function(m[1]);console.log('OK')"
```

### 浏览器控制台
```javascript
console.log(gameState.player);
console.log('总攻击:', gameState.player.atk + sumEquipAtk());
// 查看 9 槽装备
Object.entries(gameState.player.equipment).forEach(([k,v]) => {
    if (v) console.log(k, v.name, v.quality, v.atk, v.def);
});
```

---

## 版本发布

- **主版本**: 重大重构或破坏性变更
- **次版本**: 新功能或较大改进
- **修订号**: Bug 修复或小调整

发布步骤:
1. 确认所有修改测试通过
2. 运行语法验证
3. 更新 3 份文档（CHANGELOG / README / DEVELOPMENT）
4. 浏览器 Ctrl+F5 强制刷新测试

---

## 维护信息

**开发者**: 傻妞
**项目起始**: 2026-05-27
**当前版本**: v3.17.0
**项目路径**: `/Users/xiaoyu/Desktop/Rogue/`