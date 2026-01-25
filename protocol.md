# Zero-Pole-Island 开发协议（Dev Protocol）

目录
1. 原则
2. 结构与命名
3. 生命周期
4. 实体/角色
5. 战斗
6. 状态
7. 交互
8. 物品/背包
9. 技能
10. Boss/AI
11. 撤离
12. Realm（软/硬）
13. 事件总线
14. 存档
15. MVP

============================================================
1. 原则
============================================================
- 组合优先，继承只保留最薄基类（ActorBase）。
- 数据驱动：数值/掉落/技能/Boss 相位配置化，禁止硬编码。
- 跨模块只用接口 + 事件；禁止 FindObjectOfType/GameObject.Find。
- UI 不直接改 Gameplay，只订阅事件/读只读数据。

============================================================
2. 结构与命名
============================================================
目录：
/Scripts/Core /Gameplay /AI /UI /Data

命名：
接口 IThing；组件 ThingComponent；数据 ThingDef；实例 ThingInstance；事件 ThingEvent。

============================================================
3. 生命周期
============================================================
统一驱动：
public interface IInitializable { void Initialize(); }
public interface ITickable { void Tick(float dt); }
public interface IFixedTickable { void FixedTick(float fdt); }

场景唯一 GameRoot 负责驱动上述接口。

============================================================
4. 实体/角色
============================================================
public interface IEntity {
  int EntityId { get; }
  EntityType Type { get; }
  Faction Faction { get; }
  Transform Transform { get; }
}
public enum EntityType { Player, Enemy, Boss, Loot, Interactable, Projectile }
public enum Faction { Player, Neutral, Hostile }

public interface IMover {
  Vector2 Velocity { get; }
  void Move(Vector2 input, float dt);
  void Teleport(Vector2 worldPos);
}

ActorBase 必选组件（必须挂载并缓存引用）：
IMover / IHealth / ICombatant / IInventoryOwner / IAbilityOwner

============================================================
5. 战斗
============================================================
public interface IDamageable {
  bool CanTakeDamage { get; }
  void TakeDamage(in DamageInfo dmg);
}

public readonly struct DamageInfo {
  public readonly int SourceId;
  public readonly float Amount;
  public readonly DamageType Type;
  public readonly Vector2 HitPoint, HitNormal;
  public DamageInfo(int sourceId, float amount, DamageType type, Vector2 hitPoint, Vector2 hitNormal) { ... }
}
public enum DamageType { Physical, Electric, Thermal, NoiseHighFreq, NoiseLowFreq }

public interface IHealth {
  float Current { get; }
  float Max { get; }
  event System.Action<float,float> OnHealthChanged;
  event System.Action OnDied;
  void Heal(float amount);
}

public interface ICombatant {
  float AttackPower { get; }
  bool CanAttack { get; }
  void Attack(in AttackContext ctx);
}
public readonly struct AttackContext {
  public readonly Vector2 Origin, Direction;
  public readonly float Range;
  public AttackContext(Vector2 origin, Vector2 direction, float range) { ... }
}

============================================================
6. 状态
============================================================
public interface IStatusEffect {
  string EffectId { get; }
  float Duration { get; }
  void OnApply(IEntity target);
  void OnTick(IEntity target, float dt);
  void OnRemove(IEntity target);
}
public interface IStatusReceiver {
  void AddEffect(IStatusEffect effect);
  void RemoveEffect(string effectId);
  bool HasEffect(string effectId);
}

============================================================
7. 交互
============================================================
public interface IInteractable {
  string Prompt { get; }
  bool CanInteract(IEntity interactor);
  void Interact(IEntity interactor);
}

============================================================
8. 物品/背包
============================================================
public abstract class ItemDef : ScriptableObject {
  public string itemId, displayName;
  public Sprite icon;
  public int maxStack = 1;
  public int rarity;
  public float weight;
  public abstract ItemType Type { get; }
}
public enum ItemType { Consumable, Weapon, Mod, Quest, Currency }

public sealed class ItemInstance {
  public ItemDef Def { get; }
  public int Stack { get; set; }
  public Dictionary<string,float> Rolls { get; } = new();
  public ItemInstance(ItemDef def, int stack = 1) { ... }
}

public interface IInventory {
  IReadOnlyList<ItemInstance> Items { get; }
  event System.Action OnInventoryChanged;
  bool TryAdd(ItemInstance item);
  bool TryRemove(string itemId, int count);
}
public interface IInventoryOwner { IInventory Inventory { get; } }

============================================================
9. 技能
============================================================
public interface IAbility {
  string AbilityId { get; }
  float Cooldown { get; }
  bool CanCast(IEntity caster);
  void Cast(IEntity caster, in AbilityContext ctx);
}
public readonly struct AbilityContext {
  public readonly Vector2 AimWorldPos;
  public readonly IEntity Target;
  public AbilityContext(Vector2 aimWorldPos, IEntity target = null) { ... }
}
public interface IAbilityOwner {
  bool TryCast(string abilityId, in AbilityContext ctx);
}

============================================================
10. Boss/AI
============================================================
public interface IBoss {
  int Phase { get; }
  event System.Action<int> OnPhaseChanged;
  void EnterPhase(int phase);
}

public interface IAttackPattern {
  string PatternId { get; }
  float Weight { get; }
  bool CanUse(IEntity boss);
  void Telegraph(IEntity boss);
  void Execute(IEntity boss);
}

============================================================
11. 撤离（Extraction）
============================================================
public interface IExtractionZone : IInteractable {
  float ChannelTime { get; }
  bool IsContested { get; }
  bool CanExtract(IEntity player, out string reason);
}

public interface IRunResultService {
  void CommitExtract(IEntity player);
}

============================================================
12. Realm（软/硬）
============================================================
public enum RealmType { Hard, Soft }
public interface IRealmSwitchListener { void OnEnterRealm(RealmType realm); }

============================================================
13. 事件总线
============================================================
public interface IEventBus {
  void Publish<T>(T evt);
  void Subscribe<T>(System.Action<T> handler);
  void Unsubscribe<T>(System.Action<T> handler);
}

public readonly struct PlayerExtractedEvent { public readonly int PlayerId; public PlayerExtractedEvent(int id){PlayerId=id;} }
public readonly struct BossDefeatedEvent { public readonly int BossId; public BossDefeatedEvent(int id){BossId=id;} }

============================================================
14. 存档
============================================================
public interface ISaveable {
  string SaveId { get; }
  object CaptureState();
  void RestoreState(object state);
}



地图：
出生点 + 资源点 + Boss 房 + 撤离点
