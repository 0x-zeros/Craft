# Craft光照与AO系统详解.md

# Craft项目光照与环境光遮蔽(AO)系统详解

## 系统概述

Craft项目采用了一个**非实时光照系统**，结合**环境光遮蔽(AO)**和**天空光照**，营造出自然的光影效果。你观察到的"AO的效果很自然"正是这个精心设计的光照系统的成果。

## 1. 光照系统架构

### **光照类型分层：**
1. **天空光照** (Daylight) - 全局环境光
2. **人工光源** - 玩家放置的光源方块  
3. **环境光遮蔽** (AO) - 局部阴影效果
4. **漫反射光照** - 基于法向量的方向光

## 2. 环境光遮蔽(AO)计算核心

### **AO计算函数 (`occlusion`)**

```c
void occlusion(
    char neighbors[27], char lights[27], float shades[27],
    float ao[6][4], float light[6][4])
{
    static const int lookup3[6][4][3] = {
        {{0, 1, 3}, {2, 1, 5}, {6, 3, 7}, {8, 5, 7}},
        {{18, 19, 21}, {20, 19, 23}, {24, 21, 25}, {26, 23, 25}},
        {{6, 7, 15}, {8, 7, 17}, {24, 15, 25}, {26, 17, 25}},
        {{0, 1, 9}, {2, 1, 11}, {18, 9, 19}, {20, 11, 19}},
        {{0, 3, 9}, {6, 3, 15}, {18, 9, 21}, {24, 15, 21}},
        {{2, 5, 11}, {8, 5, 17}, {20, 11, 23}, {26, 17, 23}}
    };
    // ...
}
```

**关键设计思想：**

1. **3x3x3邻域分析**：检查每个方块周围27个位置
2. **顶点级精度**：每个方块面的4个顶点独立计算AO
3. **多因素综合**：结合邻域遮挡、光照传播、高度阴影

### **AO值计算公式：**

```c
int corner = neighbors[lookup3[i][j][0]];
int side1 = neighbors[lookup3[i][j][1]];
int side2 = neighbors[lookup3[i][j][2]];
int value = side1 && side2 ? 3 : corner + side1 + side2;
```

**AO算法详解：**

```c
// AO遮蔽级别曲线
static const float curve[4] = {0.0, 0.25, 0.5, 0.75};

// 最终AO值计算
float total = curve[value] + shade_sum / 4.0;
ao[i][j] = MIN(total, 1.0);
```

**遮蔽逻辑：**
- `value = 0`: 无遮蔽 → AO = 0.0 (最亮)
- `value = 1`: 轻微遮蔽 → AO = 0.25  
- `value = 2`: 中等遮蔽 → AO = 0.5
- `value = 3`: 完全遮蔽 → AO = 0.75 (最暗)

### **邻域查找表解析**

AO计算的核心是 `lookup3` 查找表，它定义了每个面的每个顶点需要检查的3个关键位置：
```
3x3x3邻域编号 (27个位置)：
层0 (y-1):  [0][1][2]     层1 (y):    [9][10][11]    层2 (y+1): [18][19][20]
           [3][4][5]                  [12][13][14]               [21][22][23]  
           [6][7][8]                  [15][16][17]               [24][25][26]
```

对于每个顶点，检查：
- **corner**: 对角相邻的方块
- **side1**: 边相邻的方块1  
- **side2**: 边相邻的方块2

## 3. 高度阴影系统

### **垂直阴影计算：**

```c
if (y + dy <= highest[XZ(x + dx, z + dz)]) {
    for (int oy = 0; oy < 8; oy++) {
        if (opaque[XYZ(x + dx, y + dy + oy, z + dz)]) {
            shades[index] = 1.0 - oy * 0.125;
            break;
        }
    }
}
```

**阴影渐变设计：**
- 检查上方8个方块
- 距离越近阴影越深：`1.0 - distance * 0.125`
- 创造自然的深度阴影效果

**阴影强度计算：**
```
距离1格: shades = 1.0 - 1*0.125 = 0.875 (深阴影)
距离2格: shades = 1.0 - 2*0.125 = 0.750
距离3格: shades = 1.0 - 3*0.125 = 0.625
...
距离8格: shades = 1.0 - 8*0.125 = 0.000 (无阴影)
```

## 4. 人工光源系统

### **光照传播算法 (`light_fill`)**

```c
void light_fill(
    char *opaque, char *light,
    int x, int y, z, int w, int force)
{
    if (light[XYZ(x, y, z)] >= w) {
        return;  // 已有更强光照
    }
    if (!force && opaque[XYZ(x, y, z)]) {
        return;  // 不透明方块阻挡
    }
    light[XYZ(x, y, z)] = w--;  // 设置光照强度并递减
    
    // 向6个方向递归传播
    light_fill(opaque, light, x - 1, y, z, w, 0);
    light_fill(opaque, light, x + 1, y, z, w, 0);
    light_fill(opaque, light, x, y - 1, z, w, 0);
    light_fill(opaque, light, x, y + 1, z, w, 0);
    light_fill(opaque, light, x, y, z - 1, w, 0);
    light_fill(opaque, light, x, y, z + 1, w, 0);
}
```

**光照传播特点：**
- **递归洪水填充**：从光源向外扩散
- **强度递减**：每扩散一格强度-1
- **阻挡检测**：不透明方块阻止传播
- **最大范围**：光照级别0-15，传播15格

## 5. 着色器渲染流程

### **顶点着色器处理**

```glsl
// 顶点着色器 (block_vertex.glsl)
const vec3 light_direction = normalize(vec3(-1.0, 1.0, -1.0));

void main() {
    gl_Position = matrix * position;
    fragment_uv = uv.xy;
    fragment_ao = 0.3 + (1.0 - uv.z) * 0.7;  // AO值映射
    fragment_light = uv.w;                     // 人工光源强度
    diffuse = max(0.0, dot(normal, light_direction));  // 漫反射
}
```

### **片段着色器合成**

```glsl
// 片段着色器 (block_fragment.glsl)
void main() {
    vec3 color = vec3(texture2D(sampler, fragment_uv));
    
    // 光照混合
    ao = min(1.0, ao + fragment_light);      // AO与人工光混合
    df = min(1.0, df + fragment_light);      // 漫反射与人工光混合
    float value = min(1.0, daylight + fragment_light);  // 天空光计算
    
    // 最终光照合成
    vec3 light_color = vec3(value * 0.3 + 0.2);
    vec3 ambient = vec3(value * 0.3 + 0.2);
    vec3 light = ambient + light_color * df;
    color = clamp(color * light * ao, vec3(0.0), vec3(1.0));
}
```

**最终光照合成公式：**
```
最终颜色 = 纹理颜色 × (环境光 + 方向光×漫反射) × AO值
```

## 6. 为什么效果自然？

### **多层光照混合**
1. **全局环境光**：提供基础亮度，避免完全黑暗
2. **方向光**：模拟太阳光方向性，增强立体感
3. **AO阴影**：增强方块间的立体感和深度
4. **人工光源**：局部照明增强，丰富光照层次

### **平滑过渡算法**
- **顶点插值**：GPU自动插值产生平滑渐变
- **多级AO**：4级遮蔽强度(0, 0.25, 0.5, 0.75)平滑过渡
- **距离衰减**：光照强度按距离自然衰减
- **高度阴影**：8级渐变创造自然深度感

### **物理合理性**
- **遮蔽逻辑**：符合真实阴影形成原理
- **光照传播**：模拟光的散射和衰减
- **环境混合**：多种光源自然叠加
- **表面法向**：基于几何法向量计算漫反射

## 7. 距离优化与渲染策略

### **分层距离剔除系统**

Craft项目采用**粗粒度的距离优化**（整个区块级别的剔除），而不是细粒度的LOD优化：

```c
// 渲染距离检查
for (int i = 0; i < g->chunk_count; i++) {
    Chunk *chunk = g->chunks + i;
    if (chunk_distance(chunk, p, q) > g->render_radius) {
        continue;  // 超出渲染距离的区块直接跳过
    }
    if (!chunk_visible(planes, chunk->p, chunk->q, chunk->miny, chunk->maxy)) {
        continue;  // 不在视野内的区块跳过渲染
    }
    draw_chunk(attrib, chunk);
}
```

### **配置参数设置**

```c
#define CREATE_CHUNK_RADIUS 10  // 区块创建半径（以区块为单位）
#define RENDER_CHUNK_RADIUS 10  // 区块渲染半径（以区块为单位）
#define RENDER_SIGN_RADIUS 4    // 标牌渲染半径（以区块为单位）
#define DELETE_CHUNK_RADIUS 14  // 区块删除半径（以区块为单位）
```

### **分层渲染优化**

```
距离层级管理：
├── 0-4区块：完整渲染（方块+标牌+光照）
├── 4-10区块：方块渲染（无标牌）
├── 10-14区块：不渲染但保持加载
└── 14+区块：完全卸载
```

### **视锥体剔除算法**

```c
int chunk_visible(float planes[6][4], int p, int q, int miny, int maxy) {
    // 检查区块的8个角点是否在视锥体内
    float points[8][3] = {
        {x + 0, miny, z + 0}, {x + d, miny, z + 0},
        {x + 0, miny, z + d}, {x + d, miny, z + d},
        {x + 0, maxy, z + 0}, {x + d, maxy, z + 0},
        {x + 0, maxy, z + d}, {x + d, maxy, z + d}
    };
    // 使用6个平面进行剔除测试
    int n = g->ortho ? 4 : 6;  // 正交模式只需4个平面
    // ... 剔除逻辑
}
```

### **雾化效果实现**

**顶点着色器中的距离计算：**
```glsl
float camera_distance = distance(camera, vec3(position));
fog_factor = pow(clamp(camera_distance / fog_distance, 0.0, 1.0), 4.0);
```

**片段着色器中的雾化混合：**
```glsl
vec3 sky_color = vec3(texture2D(sky_sampler, vec2(timer, fog_height)));
color = mix(color, sky_color, fog_factor);
```

### **为什么不使用传统LOD？**

1. **静态光照系统**：光照数据预计算并存储，不存在实时光照计算开销
2. **区块级优化**：直接跳过远距离区块比简化光照更高效
3. **二进制可见性**：方块要么完全渲染，要么完全不渲染
4. **GPU插值**：顶点着色器的插值已经提供了平滑效果

## 8. 实现要点总结

### **核心算法**
1. **3x3x3邻域AO**：每个顶点检查周围27个方块
2. **洪水填充光照**：递归传播人工光源
3. **高度阴影**：检查上方8格创造深度阴影
4. **多光源混合**：天空光+人工光+AO的自然融合

### **性能优化**
- **区块级计算**：每个区块独立计算光照
- **邻域数据缓存**：3x3区块光照数据预加载
- **多线程处理**：工作线程异步计算
- **顶点级AO**：避免像素级计算
- **距离剔除**：远距离区块跳过渲染
- **视锥体剔除**：视野外区块不参与渲染

这个系统的巧妙之处在于**用静态预计算实现了接近实时光照的视觉效果**，同时保持了出色的性能。虽然不是实时光照，但通过精心设计的AO算法和多层光照混合，创造出了非常自然和美观的光影效果，这正是体素游戏光照系统的经典实现方案。

## 9. 总结与技术亮点

### **主要内容回顾**

1. **系统架构**：4层光照系统的设计思路
2. **AO算法核心**：3x3x3邻域分析和查找表实现
3. **高度阴影**：8级渐变的垂直阴影系统
4. **光照传播**：递归洪水填充算法
5. **着色器管线**：从顶点到片段的光照数据处理
6. **距离优化**：分层剔除和视锥体剔除策略
7. **性能优化**：区块级计算和多线程策略

### **关键发现：为什么效果自然？**

1. **多层光照混合**：天空光+方向光+AO+人工光源的完美结合
2. **顶点级精度**：每个方块面的4个顶点独立计算AO值
3. **平滑过渡**：4级AO强度曲线(0, 0.25, 0.5, 0.75)
4. **物理合理性**：基于真实光照原理的算法设计

### **技术亮点**

- **查找表优化**：用预计算表加速AO计算
- **3x3区块缓存**：解决边界光照连续性问题
- **静态预计算**：避免实时光照的性能开销
- **GPU插值**：产生平滑的光照渐变
- **分层剔除**：粗粒度距离优化策略
- **视锥体剔除**：精确的可见性检测

### **设计哲学**

这个系统体现了体素游戏光照设计的核心哲学：**用简单高效的算法实现复杂自然的视觉效果**。通过巧妙的算法设计和优化策略，在保持出色性能的同时，创造出了非常自然和美观的光影效果，是体素游戏光照系统的经典实现案例。


