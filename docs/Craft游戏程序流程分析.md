# Craft游戏程序流程分析

## 概述
Craft是一个Minecraft风格的体素游戏，使用C语言和OpenGL开发。本文档详细分析了游戏的主要程序流程和架构设计。

## 程序架构

### 核心数据结构
```c
// 游戏主状态
typedef struct {
    GLFWwindow *window;           // 游戏窗口
    Worker workers[WORKERS];      // 工作线程池
    Chunk chunks[MAX_CHUNKS];     // 区块管理
    Player players[MAX_PLAYERS];  // 玩家管理
    Block block0, block1;         // 方块操作记录
    Block copy0, copy1;           // 复制粘贴缓冲区
    int mode;                     // 游戏模式（在线/离线）
    int mode_changed;             // 模式切换标志
    // ... 其他游戏状态
} Model;

// 区块结构
typedef struct {
    Map map;                      // 方块数据映射
    Map lights;                   // 光照数据映射
    SignList signs;               // 告示牌列表
    int p, q;                     // 区块坐标
    int faces;                    // 可见面数量
    int dirty;                    // 是否需要重新生成
    GLuint buffer;                // OpenGL顶点缓冲区
} Chunk;

// 方块结构
typedef struct {
    int x, y, z;                  // 坐标
    int w;                        // 方块类型
} Block;
```

## 程序执行流程

### 1. 程序初始化阶段

#### 1.1 基础库初始化
```c
curl_global_init(CURL_GLOBAL_DEFAULT);  // 网络库初始化
srand(time(NULL));                       // 随机数种子
rand();
```

#### 1.2 窗口和OpenGL初始化
```c
if (!glfwInit()) return -1;              // 初始化GLFW
create_window();                          // 创建游戏窗口
glfwMakeContextCurrent(g->window);       // 设置OpenGL上下文
glfwSetInputMode(g->window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);
// 设置输入回调函数
glfwSetKeyCallback(g->window, on_key);
glfwSetCharCallback(g->window, on_char);
glfwSetMouseButtonCallback(g->window, on_mouse_button);
glfwSetScrollCallback(g->window, on_scroll);
```

#### 1.3 OpenGL状态配置
```c
glEnable(GL_CULL_FACE);                  // 启用面剔除
glEnable(GL_DEPTH_TEST);                 // 启用深度测试
glLogicOp(GL_INVERT);                    // 设置逻辑操作
glClearColor(0, 0, 0, 1);               // 设置清除颜色
```

#### 1.4 纹理资源加载
```c
// 加载4种核心纹理
GLuint texture = load_png_texture("textures/texture.png");    // 方块纹理
GLuint font = load_png_texture("textures/font.png");          // 字体纹理
GLuint sky = load_png_texture("textures/sky.png");            // 天空纹理
GLuint sign = load_png_texture("textures/sign.png");          // 告示牌纹理
```

#### 1.5 着色器程序加载
```c
// 加载4种着色器程序
Attrib block_attrib = load_program("shaders/block_vertex.glsl", "shaders/block_fragment.glsl");
Attrib line_attrib = load_program("shaders/line_vertex.glsl", "shaders/line_fragment.glsl");
Attrib text_attrib = load_program("shaders/text_vertex.glsl", "shaders/text_fragment.glsl");
Attrib sky_attrib = load_program("shaders/sky_vertex.glsl", "shaders/sky_fragment.glsl");
```

#### 1.6 命令行参数处理
```c
if (argc == 2 || argc == 3) {
    g->mode = MODE_ONLINE;               // 在线模式
    strncpy(g->server_addr, argv[1], MAX_ADDR_LENGTH);
    g->server_port = argc == 3 ? atoi(argv[2]) : DEFAULT_PORT;
} else {
    g->mode = MODE_OFFLINE;              // 离线模式
    snprintf(g->db_path, MAX_PATH_LENGTH, "%s", DB_PATH);
}
```

#### 1.7 工作线程初始化
```c
for (int i = 0; i < WORKERS; i++) {
    Worker *worker = g->workers + i;
    worker->index = i;
    worker->state = WORKER_IDLE;
    mtx_init(&worker->mtx, mtx_plain);   // 初始化互斥锁
    cnd_init(&worker->cnd);               // 初始化条件变量
    thrd_create(&worker->thrd, worker_run, worker);  // 创建工作线程
}
```

### 2. 外层循环（游戏模式管理）

```c
int running = 1;
while (running) {
    // 根据当前模式初始化数据库和网络
    if (g->mode == MODE_OFFLINE || USE_CACHE) {
        db_enable();
        db_init(g->db_path);
    }
    
    if (g->mode == MODE_ONLINE) {
        client_enable();
        client_connect(g->server_addr, g->server_port);
        client_start();
        login();
    }
    
    // 进入主游戏循环
    while (1) {
        // 主游戏逻辑...
        
        // 检查模式切换
        if (g->mode_changed) {
            break;  // 跳出内层循环，重新初始化新模式
        }
    }
    
    // 清理当前模式资源
    db_save_state(s->x, s->y, s->z, s->rx, s->ry);
    db_close();
    client_stop();
    delete_all_chunks();
    delete_all_players();
}
```

### 3. 内层循环（主游戏循环）

#### 3.1 循环初始化
```c
reset_model();                           // 重置游戏模型
FPS fps = {0, 0, 0};                    // 初始化帧率统计
double last_commit = glfwGetTime();      // 数据库提交时间
double last_update = glfwGetTime();      // 位置更新时间
GLuint sky_buffer = gen_sky_buffer();    // 生成天空缓冲区

// 初始化玩家状态
Player *me = g->players;
State *s = &g->players->state;
me->id = 0;
g->player_count = 1;

// 从数据库加载玩家状态
int loaded = db_load_state(&s->x, &s->y, &s->z, &s->rx, &s->ry);
force_chunks(me);                       // 强制加载周围区块
if (!loaded) {
    s->y = highest_block(s->x, s->z) + 2;  // 设置默认高度
}
```

#### 3.2 主循环核心
```c
while (1) {
    // 窗口尺寸和缩放处理
    g->scale = get_scale_factor();
    glfwGetFramebufferSize(g->window, &g->width, &g->height);
    glViewport(0, 0, g->width, g->height);
    
    // 帧率计算
    update_fps(&fps);
    double now = glfwGetTime();
    double dt = now - previous;
    dt = MIN(dt, 0.2);
    dt = MAX(dt, 0.0);
    previous = now;
    
    // 输入处理
    handle_mouse_input();
    handle_movement(dt);
    
    // 网络通信
    char *buffer = client_recv();
    if (buffer) {
        parse_buffer(buffer);
        free(buffer);
    }
    
    // 数据库操作
    if (now - last_commit > COMMIT_INTERVAL) {
        last_commit = now;
        db_commit();
    }
    
    // 位置同步
    if (now - last_update > 0.1) {
        last_update = now;
        client_position(s->x, s->y, s->z, s->rx, s->ry);
    }
    
    // 渲染准备
    delete_chunks();                     // 删除远处的区块
    del_buffer(me->buffer);
    me->buffer = gen_player_buffer(s->x, s->y, s->z, s->rx, s->ry);
    
    // 玩家插值
    for (int i = 1; i < g->player_count; i++) {
        interpolate_player(g->players + i);
    }
    
    // 3D场景渲染
    glClear(GL_COLOR_BUFFER_BIT);
    glClear(GL_DEPTH_BUFFER_BIT);
    render_sky(&sky_attrib, player, sky_buffer);
    glClear(GL_DEPTH_BUFFER_BIT);
    int face_count = render_chunks(&block_attrib, player);
    render_signs(&text_attrib, player);
    render_sign(&text_attrib, player);
    render_players(&block_attrib, player);
    
    if (SHOW_WIREFRAME) {
        render_wireframe(&line_attrib, player);
    }
    
    // HUD渲染
    glClear(GL_DEPTH_BUFFER_BIT);
    if (SHOW_CROSSHAIRS) {
        render_crosshairs(&line_attrib);
    }
    if (SHOW_ITEM) {
        render_item(&block_attrib);
    }
    
    // 文本信息渲染
    if (SHOW_INFO_TEXT) {
        // 显示坐标、区块、帧率等信息
    }
    if (SHOW_CHAT_TEXT) {
        // 显示聊天消息
    }
    if (g->typing) {
        // 显示输入提示
    }
    if (SHOW_PLAYER_NAMES) {
        // 显示玩家名称
    }
    
    // 画中画渲染
    if (g->observe2) {
        // 渲染第二个玩家的视角
    }
    
    // 缓冲区交换和事件处理
    glfwSwapBuffers(g->window);
    glfwPollEvents();
    
    // 退出检查
    if (glfwWindowShouldClose(g->window)) {
        running = 0;
        break;
    }
    
    // 模式切换检查
    if (g->mode_changed) {
        g->mode_changed = 0;
        break;
    }
}
```

## 关键系统流程

### 1. 区块管理系统

#### 1.1 区块加载流程
```c
void ensure_chunks(Player *player) {
    check_workers();                     // 检查工作线程状态
    force_chunks(player);                // 强制加载近距离区块
    
    for (int i = 0; i < WORKERS; i++) {
        Worker *worker = g->workers + i;
        if (worker->state == WORKER_IDLE) {
            ensure_chunks_worker(player, worker);  // 分配工作给空闲线程
        }
    }
}
```

#### 1.2 区块生成流程
```c
void compute_chunk(WorkerItem *item) {
    // 1. 分配内存
    char *opaque = calloc(XZ_SIZE * XZ_SIZE * Y_SIZE, sizeof(char));
    char *light = calloc(XZ_SIZE * XZ_SIZE * Y_SIZE, sizeof(char));
    
    // 2. 填充不透明数组
    for (int a = 0; a < 3; a++) {
        for (int b = 0; b < 3; b++) {
            Map *map = item->block_maps[a][b];
            MAP_FOR_EACH(map, ex, ey, ez, ew) {
                opaque[XYZ(x, y, z)] = !is_transparent(ew);
            } END_MAP_FOR_EACH;
        }
    }
    
    // 3. 光照计算
    if (has_light) {
        // 洪水填充光照强度
    }
    
    // 4. 几何生成
    // 计算暴露的面并生成顶点数据
}
```

#### 1.3 区块渲染流程
```c
int render_chunks(Attrib *attrib, Player *player) {
    ensure_chunks(player);               // 确保区块已加载
    
    for (int i = 0; i < g->chunk_count; i++) {
        Chunk *chunk = g->chunks + i;
        
        // 距离检查
        if (chunk_distance(chunk, p, q) > g->render_radius) {
            continue;
        }
        
        // 可见性检查
        if (!chunk_visible(planes, chunk->p, chunk->q, chunk->miny, chunk->maxy)) {
            continue;
        }
        
        draw_chunk(attrib, chunk);       // 绘制区块
        result += chunk->faces;
    }
    return result;
}
```

### 2. 方块操作系统

#### 2.1 方块设置流程
```c
void set_block(int x, int y, int z, int w) {
    int p = chunked(x);
    int q = chunked(z);
    _set_block(p, q, x, y, z, w, 1);   // 设置主方块
    
    // 处理跨区块边界
    for (int dx = -1; dx <= 1; dx++) {
        for (int dz = -1; dz <= 1; dz++) {
            if (dx == 0 && dz == 0) continue;
            if (dx && chunked(x + dx) == p) continue;
            if (dz && chunked(z + dz) == q) continue;
            _set_block(p + dx, q + dz, x, y, z, -w, 1);  // 标记边界方块
        }
    }
    
    client_block(x, y, z, w);           // 通知客户端
}
```

#### 2.2 方块记录系统
```c
void record_block(int x, int y, int z, int w) {
    memcpy(&g->block1, &g->block0, sizeof(Block));  // 保存上一次操作
    g->block0.x = x;                    // 记录当前操作
    g->block0.y = y;
    g->block0.z = z;
    g->block0.w = w;
}
```

### 3. 渲染系统

#### 3.1 渲染管线
1. **天空渲染** - 作为背景
2. **区块渲染** - 世界方块
3. **告示牌渲染** - 文本信息
4. **玩家渲染** - 角色模型
5. **HUD渲染** - 用户界面
6. **文本渲染** - 信息显示

#### 3.2 着色器处理
```glsl
// 方块片段着色器中的透明处理
void main() {
    vec3 color = vec3(texture2D(sampler, fragment_uv));
    if (color == vec3(1.0, 0.0, 1.0)) {
        discard;  // 丢弃洋红色像素（透明效果）
    }
    // ... 其他渲染逻辑
}
```

### 4. 网络通信系统

#### 4.1 客户端初始化
```c
if (g->mode == MODE_ONLINE) {
    client_enable();
    client_connect(g->server_addr, g->server_port);
    client_start();
    client_version(1);
    login();
}
```

#### 4.2 数据同步
```c
// 位置同步
if (now - last_update > 0.1) {
    client_position(s->x, s->y, s->z, s->rx, s->ry);
}

// 接收服务器数据
char *buffer = client_recv();
if (buffer) {
    parse_buffer(buffer);
    free(buffer);
}
```

## 性能优化策略

### 1. 区块管理优化
- **视锥剔除** - 只渲染可见区块
- **距离管理** - 根据距离动态加载/卸载区块
- **工作线程** - 后台异步生成区块几何数据

### 2. 渲染优化
- **面剔除** - 只渲染暴露的方块面
- **顶点缓冲区** - 预生成几何数据
- **纹理图集** - 减少纹理切换

### 3. 内存管理
- **延迟加载** - 按需加载区块数据
- **智能缓存** - 缓存常用数据
- **及时清理** - 释放不需要的资源

## 错误处理和恢复

### 1. 网络异常处理
```c
// 网络连接失败时自动切换到离线模式
if (client_connect_failed) {
    g->mode = MODE_OFFLINE;
    g->mode_changed = 1;
}
```

### 2. 资源加载失败
```c
// 纹理加载失败时使用默认纹理
if (!load_texture_success) {
    use_default_texture();
    log_error("Texture loading failed");
}
```

### 3. 数据库异常
```c
// 数据库操作失败时重试或降级
if (db_operation_failed) {
    retry_operation();
    if (still_failed) {
        fallback_to_memory_storage();
    }
}
```

## 总结

Craft游戏采用了清晰的分层架构设计：

1. **初始化层** - 负责资源加载和系统配置
2. **模式管理层** - 处理在线/离线模式切换
3. **游戏逻辑层** - 处理玩家输入和游戏状态
4. **渲染层** - 负责图形输出和显示
5. **数据层** - 管理世界数据和持久化存储

这种设计具有良好的可维护性和扩展性，支持多种游戏模式，并能够高效地处理大型开放世界。通过工作线程、智能缓存和渲染优化等技术，确保了游戏的流畅运行。
