# 愚者
「愚者架构」是一种架构设计思想，结合了Spartan-Programming、KISS原则、领域驱动设计、(C语言的)过程式编程思想、(C#的)面向对象编程思想的优点。它已经过多个项目实战检验。  
其主要用于游戏客户端的架构设计，也同样可用于服务端单体架构设计，或应用于其他领域。  
<i>(注: 由于架构设计需要足够的编程经验, 建议编程超过3000小时以上的程序员阅读.)</i>

<br>
<br>

# 原则
### 1. 要使代码易于静态阅读
<details>
<summary>点击展开</summary>
<summary>- 极力避免继承和多态</summary>
<summary>- 极力避免MVVM</summary>
<summary>- 尽量保持正序控制, 极力避免IoC: 因此不使用依赖注入框架; 不使用事件驱动, 仅在必要时(例如UI、网络)使用事件</summary>
</details>

### 2. 架构是演进的
<details>
<summary>点击展开</summary>
<summary>在研发初期, 易于变化很重要, 因此要减少约束</summary>
<summary>在研发中期, 内容量产很重要, 因此要易于静态阅读, 使新参与者能快速上手</summary>
<summary>在项目上线期, 稳定性很重要, 因此要对架构加以约束</summary>
<summary>重构需要每时每刻都在进行</summary>
</details>

<br>
<br>

# 关键设计
### 1. Entity
> `Entity` = `字段` + `Component`  
> 由很纯粹的字段组成，只有极简的功能(函数)，目的是让设计者关注数据本身。  
> 如果字段过多, 可将一系列字段拆解成`Component`
``` CSharp
// Entity类的示例
public class BulletEntity {

    public EntityType type => EntityType.Bullet;
    public int id;
    public float speed;
    public Vector3 position;

    public AnimationComponent animation;
    ...

    public void Move(Vector3 dir, float deltaTime) {
        // 处理子弹移动逻辑
        position += dir * speed * deltaTime;

        // 更新动画组件
        animation.Play("Fly");
    }
}
```

### 2. Repository
> 是Entity的集合，负责存储所有Entity, 以便于查找  
``` CSharp
// Repository类的示例
public class BulletRepository {

    private Dictionary<int, BulletEntity> bullets = new Dictionary<int, BulletEntity>();

    // 注意: 这里只是添加一个已经生成的实例
    // 绝不在Repository中创建或销毁Entity实例
    public void Add(BulletEntity bullet) {
        bullets.Add(bullet.id, bullet);
    }

    public bool TryGet(int id, out BulletEntity bullet) {
        return bullets.TryGetValue(id, out bullet);
    }

    public void Remove(int id) {
        bullets.Remove(id);
    }

    public void Foreach(Action<BulletEntity> action) {
        foreach (var bullet in bullets.Values) {
            action(bullet);
        }
    }

}
```

### 3. Domain
> `Domain` = `Entity` + `Repository`  
> 是单一`Entity`的功能集合  
> 一定有 `Spawn()` / `Unspawn()` 函数, 最好将这两个主要函数放在前面  
> 整个程序的`Entity`的创建与销毁只发生在`Spawn()`与`Unspawn()`函数中, 因此静态阅读时可以快速定位
``` CSharp
// Domain类的示例
public static class BulletDomain {

    public static BulletEntity Spawn(BulletRepository repo, int id, float speed, Vector3 position) {
        // 创建一个新的子弹实例(或从对象池获取)
        BulletEntity bullet = new BulletEntity();
        ...
        repo.Add(bullet);
        return bullet;
    }

    public static void Unspawn(BulletRepository repo, BulletEntity bullet) {
        // 从Repository中移除子弹
        repo.Remove(bullet.id);

        // 销毁(或放入对象池)
    }

    public static void Move(AudioModule audioModule, BulletEntity bullet, float deltaTime) {
        // 处理子弹移动逻辑
        bullet.Move(Vector3.forward, deltaTime);

        // 播放音效
        audioModule.Play(bullet.type, bullet.id, "bullet_move");
    }
}
```

### 4. Controller
> `Controller` = `Context` + `Domain`  
> 是`Domain`的集合，负责处理不同种类`Entity`的交互逻辑
``` CSharp
// Controller类的示例
public static class BulletController {

    public static void Move(AudioModule audioModule, RoleRepository roleRepo, BulletRepository bulletRepo, BulletEntity bullet, float deltaTime) {
        // 子弹飞行
        BulletDomain.Move(audioModule, bullet, deltaTime);

        // 检测子弹与角色的碰撞

        // 处理碰撞逻辑

    }
}
```

### 5. Context
> `Context` = `Repository` + `Module`  
> 上下文, 即是程序的状态, 一切数据(与模块)都存储在这里  
> 只允许添加、删除、查询，不允许创建与销毁实例  
``` CSharp
// Context类的示例
public class GameContext {

    public SystemStatus systemStatus; // Enum, 表示系统状态

    public BulletRepository bulletRepo;
    public RoleRepository roleRepo;

    public AudioModule audioModule;

    public UIApplication uiApp;

    public void Inject(BulletRepository bulletRepo, RoleRepository roleRepo, AudioModule audioModule, UIApplication uiApp) {
        this.bulletRepo = bulletRepo;
        this.roleRepo = roleRepo;

        this.audioModule = audioModule;

        this.uiApp = uiApp;
    }
}
```

### 6. System
> `System` = `Context` + `Controller`  
> 整合`Controller`的调用顺序  
> 常用系统: 登录系统 / 暂停系统 / 设置系统 / 制作名单系统, 以及最重要的游戏系统
``` CSharp
// System类的示例
public static class LoginSystem {

    public static void Enter(LoginContext ctx) {
        // 打开登录界面
        ctx.systemStatus = SystemStatus.Running;
        ctx.uiApp.Login_Open();
    }

    public static void Tick(LoginContext ctx, float deltaTime) {
        // LoginSystem 的主循环
        if (ctx.inputModule.IsExitDown()) {
            ExitGame(ctx);
        }
    }

    public static void Exit(LoginContext ctx) {
        // 关闭登录界面
        ctx.systemStatus = SystemStatus.Stop;
        ctx.uiApp.Login_Close();
    }

    static void ExitGame(LoginContext ctx) {
        Application.Quit();
    }

}
```

### 7. Main
> 主入口  
> 负责实例化所有`Context`  
> 负责`Context`之间的依赖注入  
> 负责所有IoC的事件绑定  
``` CSharp
// Main类的示例
public class Main {

    LoginContext loginContext; // 登录系统的上下文
    GameContext gameContext; // 游戏系统的上下文
    
    BulletRepository bulletRepo; // 子弹仓库
    RoleRepository roleRepo; // 角色仓库

    AssetModule assetModule; // 资源模块
    AudioModule audioModule; // 音频模块

    UIApplication uiApp; // UI

    public void Start() {

        // ==== 实例化 ====
        loginContext = new LoginContext();
        gameContext = new GameContext();

        bulletRepo = new BulletRepository();
        roleRepo = new RoleRepository();

        assetModule = new AssetModule();
        audioModule = new AudioModule();

        // ==== 注入 ====
        loginContext.Inject(audioModule, uiApp);
        gameContext.Inject(bulletRepo, roleRepo, audioModule, uiApp);

        // ==== 绑定事件 ====
        // 例如: UI事件、网络事件等
        uiApp.OnLoginButtonClick = () => {
            LoginSystem.Exit(loginContext);
            GameSystem.Enter(gameContext);
        };

        // ==== 初始化 ====
        assetModule.Initialize();

        // ==== 开始程序 ====
        LoginSystem.Enter(loginContext);
    }

    void Update(float deltaTime) {
        LoginSystem.Tick(loginContext, deltaTime);
        GameSystem.Tick(gameContext, deltaTime);
    }

    void OnApplicationQuit() {
        LoginSystem.Exit(loginContext);
        GameSystem.Exit(gameContext);
    }
}
```

# 类型的演进
1. 一切围绕: 程序 = 数据 + 函数  
2. 数据 = `Context`  
    - 一开始, `Context`只有一个字段, 它就足以满足需求  
    - 但需求往往都是复杂的, 因此拆出新类型——`Entity`  
    - 当`Entity`字段过多时, 拆分出`Component`  
    - 当`Entity`有多实例时, 独立出`Repository`  
3. 函数 = `Main()`
    - 一开始, 只有`Main()`函数, 在`Main()`中配合`if`和`goto`就能完成整个程序, 即使有一个函数中有十万行代码。但这样写就太难以维护了  
    - 于是把`Main()`函数拆分成多个函数, 其中分为`控制函数`与`行为函数`  
    - 即: `Main()` = `控制函数` + `行为函数`
    - `行为函数`负责具体的行为, 比如`生成子弹`、`移动子弹`等
    - `控制函数`负责处理不同`行为函数`的调用顺序
    - `控制函数` = `System`
        - `System` = `Context` + `Controller`
        - 在实战中, 负责游戏核心的`GameSystem`会很庞大, 因此才增加了`Controller`层. 如果游戏玩法很简单, 则可以不需要`Controller`层
    - `行为函数` = `Domain`
4. 模块 = 小型程序
    - 模块一般不复杂, `Module` = `Context` + `Domain`, 甚至不需要`Domain`也可以

# 联系方式
如果你对愚者架构感兴趣, 可以找我聊天: 杰克有茶, chenwansal1@163.com  
发邮件时不需要正式语气, 我很乐意与你交流。  

# 参考资料
《领域驱动设计》  
《架构整洁之道》  
《重构: 改善既有代码的设计》  
《公正：该如何是好？》（是的，这本书与架构无关，但与思想有关。它使人不陷于“好”与“坏”的极端评价，而是从“公正”的角度看待事物。）