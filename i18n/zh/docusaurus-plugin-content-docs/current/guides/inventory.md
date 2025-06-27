---
sidebar_position: 8
---

# Inventory 库存指南

`Inventory` 类用于管理玩家、容器等实体的物品栏系统。本指南将介绍如何通过Nukkit API操作库存系统。

## Inventory 类概述 \{#inventory-class-overview}

位于 [cn.nukkit.inventory](https://github.com/MemoriesOfTime/Nukkit-MOT/blob/master/src/main/java/cn/nukkit/inventory/Inventory.java) 包中，所有库存类型的基类。

:::tip 常用子类
- **PlayerInventory** 玩家随身物品栏
- **ChestInventory** 箱子容器库存
:::

## 获取库存实例 \{#obtaining-inventory}

通常不需要直接实例化库存对象，而是通过实体获取：

```java
// 获取玩家物品栏
PlayerInventory playerInv = player.getInventory();

// 获取打开的容器库存（需类型检查）
Inventory openInv = player.getWindowById(WindowId.CONTAINER);
if(openInv instanceof ChestInventory) {
    ChestInventory chestInv = (ChestInventory) openInv;
}
```

## 主要操作方法 \{#main-operations}

### 添加物品 \{#add-items}
```java
Item diamond = Item.get(Item.DIAMOND);

// 直接给予玩家（自动堆叠）
player.giveItem(diamond);

// 安全添加（返回未放入的物品）
Item[] leftovers = playerInv.addItem(diamond.clone());  // 必须克隆防止数据污染

// 强制添加到指定槽位
playerInv.setItem(0, diamond); // 0为热键栏第一个格子
```

### 移除物品 \{#remove-items}
```java
// 移除指定数量的物品（0为meta通配符）
playerInv.removeItem(Item.get(Item.DIAMOND, 0, 5)); 

// 清空指定槽位
playerInv.clear(36); // 移除装备栏头盔位置
```

### 物品查询 \{#item-query}
```java
// 获取主手物品
Item mainHand = playerInv.getItemInHand();

// 检查是否有至少64个圆石
boolean hasCobble = playerInv.contains(Item.get(Item.COBBLESTONE, 0, 64));

// 获取所有物品（排除空物品）
Item[] contents = playerInv.getContents().values().stream()
    .filter(item -> !item.isNull())
    .toArray(Item[]::new);
```

## 槽位系统 \{#slot-system}

### 玩家库存槽位对应表
| 槽位编号 | 说明               |
|----------|--------------------|
| 0-8      | 快捷栏             |
| 9-35     | 主物品栏           |
| 36-39    | 装备栏（头盔-靴子）|
| 40       | 副手               |

### 装备操作示例
```java
// 给玩家装备钻石胸甲
Item chestplate = Item.get(Item.DIAMOND_CHESTPLATE);
playerInv.setChestplate(chestplate);  // 使用专用方法

// 获取玩家头盔
Item helmet = playerInv.getHelmet();   // 正确的方法名
```

## 库存事件监听 \{#inventory-events}

### 基础监听示例
```java
@EventHandler
public void onInventoryClick(InventoryClickEvent event) {
    Player player = event.getPlayer();
    Item clicked = event.getItem();  // 正确的API方法
    
    // 取消所有钻石的点击
    if(clicked.getId() == Item.DIAMOND) {
        event.setCancelled();
        player.sendMessage("禁止操作钻石！");
    }
}
```

### 高级交易监听
```java
@EventHandler
public void onTransaction(InventoryTransactionEvent event) {
    for(InventoryAction action : event.getTransaction().getActions()) {
        if(action instanceof SlotChangeAction) {
            SlotChangeAction slotAction = (SlotChangeAction) action;
            // 检测容器第一格被放入钻石
            if(slotAction.getInventory() instanceof ChestInventory 
               && slotAction.getSlot() == 0 
               && slotAction.getTargetItem().getId() == Item.DIAMOND) {
                event.setCancelled();
            }
        }
    }
}
```

## 特殊库存操作 \{#advanced-operations}

### 保存/恢复库存
```java
// 保存全部物品（深拷贝）
Map<Integer, Item> savedItems = new HashMap<>();
playerInv.getContents().forEach((slot, item) -> 
    savedItems.put(slot, item.clone()));

// 清空库存
playerInv.clearAll();

// 恢复库存（避免引用传递）
savedItems.forEach((slot, item) -> 
    playerInv.setItem(slot, item.clone()));
```

### 自定义库存布局
```java
// 创建虚拟库存（正确API用法）
CustomInventory myInv = new CustomInventory(null, 54, "Custom GUI");

// 设置占位符（玻璃板）
Item border = Item.get(Item.STAINED_GLASS_PANE, 14).setCustomName(" ");
for(int slot : new int[]{0,1,7,8,9,17,18,26,27,35,36,44}){
    myInv.setItem(slot, border.clone());
}

// 添加功能按钮
Item infoBtn = Item.get(Item.BOOK).setCustomName("§e点击查看信息");
myInv.setItem(22, infoBtn);
```

:::warning 重要提醒
1. 操作非玩家库存时，必须检查 `inventory.getHolder() != null`
2. 修改容器库存后需调用 `inventory.sendContents(player)` 同步客户端
3. 使用 `InventoryCloseEvent` 保存容器数据
4. **永远克隆Item对象** 防止数据意外修改
:::

## 实用工具方法 \{#utility-methods}

### 快速填充方法
```java
// 填充圆石到主物品栏（9-35）
Item cobble = Item.get(Item.COBBLESTONE);
for(int slot = 9; slot <= 35; slot++) {
    if(playerInv.getItem(slot).isNull()) {
        playerInv.setItem(slot, cobble.clone());
    }
}
```

## 常见问题处理 \{#troubleshooting}

### 物品不同步问题
```java
// 更新整个库存
playerInv.sendContents(player); 

// 更新指定槽位（热键栏0）
playerInv.sendSlot(0, player);  
```

### 处理不可堆叠物品
```java
// 添加唯一标识的武器
Item sword = Item.get(Item.DIAMOND_SWORD);
sword.setNamedTag(new CompoundTag().putString("UniqueID", UUID.randomUUID().toString()));

playerInv.addItem(sword.clone()); 
```
