# Craft项目数据库存储系统详解

## 1. 存储架构概述

### **数据库表结构**

```sql
-- 方块数据表
CREATE TABLE block (
    p INT NOT NULL,     -- 区块P坐标
    q INT NOT NULL,     -- 区块Q坐标  
    x INT NOT NULL,     -- 方块X坐标
    y INT NOT NULL,     -- 方块Y坐标
    z INT NOT NULL,     -- 方块Z坐标
    w INT NOT NULL      -- 方块类型ID
);

-- 光照数据表  
CREATE TABLE light (
    p INT NOT NULL,     -- 区块P坐标
    q INT NOT NULL,     -- 区块Q坐标
    x INT NOT NULL,     -- 光源X坐标
    y INT NOT NULL,     -- 光源Y坐标
    z INT NOT NULL,     -- 光源Z坐标
    w INT NOT NULL      -- 光照强度(0-15)
);

-- 标牌数据表
CREATE TABLE sign (
    p INT NOT NULL,     -- 区块P坐标
    q INT NOT NULL,     -- 区块Q坐标
    x INT NOT NULL,     -- 标牌X坐标
    y INT NOT NULL,     -- 标牌Y坐标
    z INT NOT NULL,     -- 标牌Z坐标
    face INT NOT NULL,  -- 标牌朝向
    text TEXT NOT NULL  -- 标牌文字内容
);

-- 区块密钥表
CREATE TABLE key (
    p INT NOT NULL,     -- 区块P坐标
    q INT NOT NULL,     -- 区块Q坐标
    key INT NOT NULL    -- 区块密钥
);
```

### **存储策略：完整存储而非增量存储**

**重要澄清：我之前关于"只存储变更"的分析是错误的！**

## 2. 实际的区块加载流程

### **区块加载的完整过程**

```c
void load_chunk(WorkerItem *item) {
    int p = item->p;
    int q = item->q;
    Map *block_map = item->block_maps[1][1];
    Map *light_map = item->light_maps[1][1];
    
    // 1. 先生成默认地形并存储到内存Map中
    create_world(p, q, map_set_func, block_map);
    
    // 2. 从数据库加载数据，覆盖内存中的默认值
    db_load_blocks(block_map, p, q);
    
    // 3. 加载光照数据
    db_load_lights(light_map, p, q);
}
```

### **`map_set_func` 的关键作用**

```c
void map_set_func(int x, int y, int z, int w, void *arg) {
    Map *map = (Map *)arg;
    map_set(map, x, y, z, w);  // 将生成的方块存储到内存Map中
}
```

**关键理解：**
1. **`create_world`** 生成默认地形并**直接存储到内存Map中**
2. **`db_load_blocks`** 从数据库加载数据，**覆盖**内存中的默认值
3. 这意味着**所有非空方块都会被存储**，不只是玩家修改

## 3. 为什么需要存储Light数据？

### **人工光源的持久化需求**

```c
void toggle_light(int x, int y, int z) {
    int p = chunked(x);
    int q = chunked(z);
    Chunk *chunk = find_chunk(p, q);
    if (chunk) {
        Map *map = &chunk->lights;
        int w = map_get(map, x, y, z) ? 0 : 15;  // 切换光源：0或15
        map_set(map, x, y, z, w);
        db_insert_light(p, q, x, y, z, w);       // 存储到数据库
        client_light(x, y, z, w);
        dirty_chunk(chunk);
    }
}
```

**关键原因：**

1. **玩家放置的光源**：
   - 玩家可以通过右键+Ctrl放置光源（强度15）
   - 这些光源不是程序生成的，需要持久化存储
   - 如果不存储，重启游戏后所有玩家光源都会消失

2. **与自然光照的区别**：
   - 天空光照：程序计算，不需要存储
   - 人工光源：玩家创建，必须存储

3. **光照传播计算**：
   ```c
   void compute_chunk_lighting(int p, int q) {
       // 需要从数据库加载已存储的光源数据
       // 作为光照传播的起点
   }
   ```

## 4. Key表的作用和用途

### **区块密钥的生成和验证**

```c
// 区块密钥生成
int get_key(int p, int q) {
    // 使用区块坐标生成唯一密钥
    // 用于验证区块数据的完整性
}

// 数据库存储
db_set_key(p, q, key);

// 数据库加载时验证
int stored_key = db_get_key(p, q);
if (stored_key != expected_key) {
    // 区块数据可能损坏，需要重新生成
}
```

### **密钥的用途**

1. **数据完整性验证**：
   - 检测数据库中的区块数据是否与当前种子匹配
   - 防止使用错误的世界种子加载数据

2. **世界种子关联**：
   - 每个世界种子会生成不同的密钥
   - 确保数据与正确的世界对应

3. **缓存失效检测**：
   - 当世界种子改变时，所有缓存数据失效
   - 强制重新生成地形

## 5. 种子(Seed)的存储策略

### **种子不直接存储的原因**

1. **程序级存储**：
   - 种子在程序启动时通过命令行参数传入
   - 存储在全局变量中，不需要数据库持久化

2. **密钥间接存储**：
   - 种子通过密钥表间接存储
   - 每个区块的密钥都基于种子生成

3. **离线模式支持**：
   - 即使没有网络，程序也能使用存储的种子
   - 通过密钥验证确保数据一致性

## 6. 与JSON文件存储的对比

### **SQLite vs JSON的优势**

| 特性 | SQLite | JSON文件 |
|------|--------|----------|
| **查询性能** | 索引优化，快速查询 | 线性扫描，性能差 |
| **并发访问** | 支持多线程安全访问 | 文件锁，并发性差 |
| **数据完整性** | ACID事务，数据安全 | 无事务保护 |
| **存储效率** | 二进制存储，空间小 | 文本存储，空间大 |
| **增量更新** | 支持部分更新 | 需要重写整个文件 |
| **索引支持** | 多字段复合索引 | 无索引支持 |

### **具体性能对比**

**SQLite查询示例：**
```sql
-- 快速查询特定区块的方块
SELECT * FROM block WHERE p = ? AND q = ?;

-- 快速查询特定坐标的方块
SELECT w FROM block WHERE x = ? AND y = ? AND z = ?;
```

**JSON文件查询：**
```python
# 需要加载整个文件到内存
with open('world.json', 'r') as f:
    data = json.load(f)
    
# 线性搜索特定区块
for chunk in data['chunks']:
    if chunk['p'] == target_p and chunk['q'] == target_q:
        # 找到目标区块
        pass
```

## 7. 存储优化策略

### **区块级存储优化**

1. **批量操作**：
   ```c
   // 批量插入方块数据
   db_begin_transaction();
   for (int i = 0; i < count; i++) {
       db_insert_block(p, q, x[i], y[i], z[i], w[i]);
   }
   db_commit_transaction();
   ```

2. **延迟写入**：
   - 方块修改先存储在内存中
   - 区块卸载时才批量写入数据库
   - 减少频繁的磁盘I/O操作

3. **压缩存储**：
   - 使用整数坐标而非浮点数
   - 区块坐标作为主键，减少重复存储

### **内存管理策略**

```c
// 区块加载时的内存管理
void load_chunk(WorkerItem *item) {
    // 1. 预分配内存
    Map *block_map = item->block_maps[1][1];
    
    // 2. 生成默认地形
    create_world(p, q, map_set_func, block_map);
    
    // 3. 加载数据库数据
    db_load_blocks(block_map, p, q);
    
    // 4. 标记区块为已加载
    item->chunk->loaded = 1;
}
```

## 8. 数据一致性保证

### **事务处理**

```c
// 数据库操作的事务保护
void db_begin_transaction() {
    sqlite3_exec(db, "BEGIN TRANSACTION", 0, 0, 0);
}

void db_commit_transaction() {
    sqlite3_exec(db, "COMMIT", 0, 0, 0);
}

void db_rollback_transaction() {
    sqlite3_exec(db, "ROLLBACK", 0, 0, 0);
}
```

### **错误恢复机制**

1. **密钥验证失败**：
   - 删除损坏的区块数据
   - 重新生成地形

2. **数据库损坏**：
   - 备份损坏的数据库
   - 重新创建数据库结构
   - 从备份恢复可用数据

3. **数据不一致**：
   - 使用密钥检测不一致
   - 强制重新生成问题区块

## 9. 总结

### **存储策略的核心特点**

1. **完整存储**：所有非空方块都会存储到数据库，不只是玩家修改
2. **分层存储**：方块、光照、标牌分别存储，支持独立查询
3. **密钥验证**：通过区块密钥确保数据与世界种子的一致性
4. **性能优化**：使用SQLite的索引和事务特性，支持高效查询和并发访问

### **设计优势**

- **数据持久性**：游戏重启后所有数据完整恢复
- **性能优秀**：SQLite的索引查询比JSON文件扫描快得多
- **并发安全**：支持多线程安全的数据访问
- **扩展性强**：易于添加新的数据类型和查询需求

### **适用场景**

这种存储策略特别适合：
- 大型开放世界游戏
- 需要频繁数据查询的场景
- 多用户并发访问
- 对数据完整性要求高的应用

Craft项目通过这种精心设计的存储系统，实现了高效、可靠、可扩展的世界数据管理，为体素游戏提供了坚实的数据基础。
