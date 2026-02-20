# 65jar（buxin 3.4.6）单人模式崩溃 NPE 分析与修复建议

## 结论（先看）
你这次崩溃是 **0 (65).jar 里的空指针**，不是 JVM 内存或显卡问题。

- 报错点：`SteveNbDangShiTiGengXinKeShiProcedure.java:248`
- 异常：`Cannot invoke "net.minecraft.world.entity.Entity.m_20185_()"`
- 调用链：`BuxinMod.tick -> lambda -> SteveNbDangShiTiGengXinKeShiProcedure`

`m_20185_()` 是实体坐标读取方法（X）。这说明当时代码里把一个 **null 实体** 当成了目标实体去读坐标。

## 代码级定位证据（基于 class 反汇编）
对 `net.mcreator.buxin.procedures.SteveNbDangShiTiGengXinKeShiProcedure` 反汇编后，
在与行号 `248` 对应的字节码里，可以看到如下逻辑：

1. 先判断 `entity instanceof Mob`
2. 然后取 `((Mob) entity).getTarget()`（映射名 `m_5448_()`）
3. **未做 null 判断**，直接调用目标实体坐标 `m_20185_()/m_20186_()/m_20189_()`

当 `getTarget()` 返回空（例如目标丢失、离开范围、实体切换状态）就会触发 NPE。

## 为什么在“单人模式”也会出现
单人模式本质是“集成服务端 + 客户端”。
你的报错是 `Exception in server tick loop`，所以是在 **服务端 tick 逻辑** 崩溃，和多人/单人无关；
只要触发到这段 AI/过程逻辑就会炸。

## 你这个整合包里会放大触发概率的因素
从你贴的模组列表看，有较多战斗与 AI 改写相关模组（如 EpicFight 系列、Indestructible、Player Mobs、各类战斗拓展）。
这些会让目标切换更频繁，导致 `Mob#getTarget()` 在某些 tick 变为空的概率更高，进而触发 65jar 的空指针。

## 临时规避（不改源码也能做）
优先级从高到低：

1. **先把旧版同系列包 `0 (64).jar` 暂时移出 mods**（避免双版本内容并存引发额外行为叠加）。
2. 新建世界做 A/B 测试：只保留 `buxin(65)` + Forge + 必需前置，确认是否仍崩。
3. 如果只在大型整合包崩，先禁用高风险联动：
   - `epicfight_improve`
   - `epic_fight_battle_styles`
   - 与实体战斗 AI 强相关的附加包
4. 崩溃前如果总是出现在某个实体附近（如 steve/村民侦察兵/特定 Herobrine 变体），可临时关闭该实体生成数据包或对应生物群系刷怪配置。

## 根修方案（给模组作者/二次开发者）
在 `SteveNbDangShiTiGengXinKeShiProcedure` 第 248 行附近，把所有 `((Mob)entity).getTarget().getX/Y/Z()` 改为安全写法：

```java
if (entity instanceof Mob mob) {
    LivingEntity target = mob.getTarget();
    if (target != null && target.isAlive()) {
        double tx = target.getX();
        double ty = target.getY();
        double tz = target.getZ();
        // 原逻辑
    }
}
```

并建议把后续用到 target 的分支统一收敛到一次判空，避免同方法里重复出现同类 NPE。

## 我建议你下一步这样做（最快）
1. 先执行临时规避第 1 条（移除 `0 (64).jar`）。
2. 复现后把**最新一次完整崩溃报告 + latest.log 最后 300 行**发我。
3. 我可以继续给你做“按模组冲突优先级排序”的二分排查清单，尽量在最少重启次数内锁定冲突组合。
