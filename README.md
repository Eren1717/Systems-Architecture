# Mobile Endless Runner — Technical Architecture & Design Patterns

> **Engine:** Unity 6 (6000.0+) · **Language:** C# · **Platform:** Android / iOS  
> **Patterns:** State Machine · Observer · Object Pool · Agent-Based · Singleton  
> **Input:** Unity New Input System 1.19+

---

## Table of Contents

1. [Project Philosophy](#1-project-philosophy)
2. [Folder Structure & Namespace Map](#2-folder-structure--namespace-map)
3. [Core Architecture](#3-core-architecture)
4. [Procedural World Generation](#4-procedural-world-generation)
5. [Autonomous Traffic System](#5-autonomous-traffic-system)
6. [Avoidance Decision System](#6-avoidance-decision-system)
7. [Player Input & Movement](#7-player-input--movement)
8. [Economy & Progression](#8-economy--progression)
9. [Persistence Layer](#9-persistence-layer)
10. [Design Decisions Log](#10-design-decisions-log)

---

## 1. Project Philosophy

Three questions drive every decision in this codebase:

| Question | What it guards against |
|---|---|
| **What does the player feel?** | Technically correct but emotionally empty systems |
| **How does this talk to other systems?** | Tight coupling, cascade failures |
| **Can this be changed without breaking everything else?** | Monolithic, untestable architecture |

Every system is **independently testable**, **event-driven**, and **config-driven** via ScriptableObjects. No magic numbers in code.

---

## 2. Folder Structure & Namespace Map

```
Assets/Scripts/
├── Core/               [Project].Core
│   ├── GameBootstrapper.cs     Boot sequence orchestrator
│   ├── GameStateManager.cs     State Machine (8 states)
│   ├── GameEvents.cs           Static Event Bus
│   ├── GameplayConfig.cs       ScriptableObject config hub
│   └── ObjectPool.cs           Generic + MonoBehaviour pool
│
├── Gameplay/           [Project].Gameplay
│   ├── PathGenerator.cs        Procedural infinite path
│   ├── TrafficManager.cs       Autonomous traffic orchestrator
│   ├── PlayerController.cs     Velocity / stamina motor
│   ├── InputHandler.cs         New Input System wrapper
│   ├── AutoAvoidance.cs        Slack-metric avoidance AI
│   ├── NearMissDetector.cs     Enter/Exit proximity trigger
│   └── CameraController.cs     Dynamic FOV chase camera
│
├── Managers/           [Project].Managers
│   ├── EconomyManager.cs       AnimationCurve upgrade engine
│   ├── ScoreManager.cs         Score + multiplier system
│   ├── SaveManager.cs          JSON persistence layer
│   ├── BoostManager.cs         Energy lifecycle manager
│   ├── AdsManager.cs           Rewarded ad wrapper
│   ├── ShopManager.cs          IAP + cosmetic store
│   └── ThemeManager.cs         Environment theme switcher
│
├── Data/               [Project].Data
│   ├── PlayerData.cs           JSON-serializable save model
│   ├── ThemeData.cs            ScriptableObject theme config
│   └── Economy/
│       └── EconomyData.cs      ScriptableObject economy config
│
├── UI/                 [Project].UI
│   ├── UIManager.cs            Panel transition controller
│   ├── GameHUD.cs              In-game HUD
│   └── UpgradePanel.cs         Economy upgrade UI
│
└── Editor/             UNITY_EDITOR only — stripped at build
    └── SceneSetupAutomator.cs  One-click scene setup tool
```

### Dependency Direction

```
    UI  →  Managers  →  Core
                ↓            ↑
            Gameplay  →  GameEvents (Bus)
```

**Rule:** Lower layers never reference higher layers. All cross-layer communication flows through `GameEvents`.

---

## 3. Core Architecture

### 3.1 State Machine

**Pattern:** State (GoF) with Dictionary-based dispatch.

```
MainMenu → Playing ↔ Boosting
              ↓
           AdOffer → MegaBoosting
              ↓
           GameOver

Any state → Paused → [resume] → previous
Any state → Shop   → [close]  → MainMenu
```

```csharp
public abstract class BaseGameState
{
    public virtual void OnEnter()  { }
    public virtual void OnUpdate() { }
    public virtual void OnExit()   { }
}

// O(1) dispatch — no switch statements
private Dictionary<GameState, BaseGameState> _stateMap;

public void ChangeState(GameState next)
{
    if (CurrentGameState == next) return;  // Guard: no re-entry
    _currentState?.OnExit();
    CurrentGameState = next;
    _currentState = _stateMap[next];
    _currentState.OnEnter();
    OnStateChanged?.Invoke(previous, next);
}
```

State transitions raise `GameEvents` — no direct method calls to other systems.

---

### 3.2 Event Bus (Observer Pattern)

**Pattern:** Static C# events. Zero MonoBehaviour overhead, no scene dependency.

```csharp
public static class GameEvents
{
    public static event Action              OnGameStarted;
    public static event Action<float>       OnEnergyChanged;    // 0-1 normalized
    public static event Action<int>         OnScoreChanged;
    public static event Action<float>       OnNearMiss;         // stamina bonus ratio
    public static event Action<int,int,int> OnEconomyUpdated;   // spd, sta, inc
    public static event Action<string>      OnUpgradePurchased;
    // 20+ events total

    public static void RaiseNearMiss(float bonus = 0f) => OnNearMiss?.Invoke(bonus);
    public static void ClearAllListeners() { OnGameStarted = null; /* ... */ }
}
```

| Option | Verdict |
|---|---|
| `UnityEvent` | Inspector-wired — breaks on scene reload |
| `ScriptableObject` events | Extra asset per event — overkill at this scale |
| **Static C# events** | Zero allocation, compile-time types, no scene coupling ✓ |

---

### 3.3 Object Pooling

**Pattern:** Generic typed `ObjectPool<T>` + singleton `PoolManager`.

```csharp
public class ObjectPool<T> where T : Component
{
    private readonly Queue<T> _pool = new Queue<T>();

    public T Get(Vector3 pos, Quaternion rot)
    {
        T obj = _pool.Count > 0 ? _pool.Dequeue() : CreateNew();
        obj.transform.SetPositionAndRotation(pos, rot);
        obj.gameObject.SetActive(true);
        return obj;
    }

    public void Return(T obj)
    {
        obj.gameObject.SetActive(false);
        _pool.Enqueue(obj);
    }
}
```

**Zero-allocation contract:** Pools pre-warmed at startup. Zero `Instantiate` / `Destroy` during normal gameplay — no GC spikes on mobile.

---

## 4. Procedural World Generation

### 4.1 Noise-Based Path

The path center is a **stateless pure function** of distance `z`. Any system queries it independently:

```csharp
public static float GetCenterX(float z)
{
    float noise = Mathf.PerlinNoise(z * noiseScale + seed, 0.5f);
    float curve = (noise - 0.5f) * 2f;   // Remap [0,1] → [-1,+1]
    float blend = ComputeBlend(z);
    return curve * amplitude * blend;
}

// Angle from positional derivative — no separate angle storage
public static float GetCenterAngleY(float z)
{
    float x1 = GetCenterX(z - 1f);
    float x2 = GetCenterX(z + 1f);
    return Mathf.Atan((x2 - x1) / 2f) * Mathf.Rad2Deg;
}
```

**Why Perlin Noise over a spline editor?**
- No pre-authored data — no artist tooling dependency
- Infinite, non-repeating, continuous output
- Single `seed` parameter controls entire world character
- Deterministic — AI and replay use identical geometry

---

### 4.2 Smoothstep Continuity

**Problem:** Naïve blend `t²` has a **derivative discontinuity** at the straight-to-curved boundary. Players perceive *rate of change*, not value — a smooth value with a discontinuous derivative still feels like a direction "snap."

```csharp
// t² — value smooth, derivative jumps at t=1  ← produces snap
// blend = t * t;

// Smoothstep — zero derivative at both endpoints
float t = z / straightDistance;
float blend = t * t * (3f - 2f * t);  // f'(0)=0  f'(1)=0  — C¹ continuity
```

`t²(3-2t)` is the minimum-degree polynomial satisfying C¹ continuity at both boundaries.

---

### 4.3 Infinite Recycling

```
Player Z = 500 m.  Active segments: Z = 460 → 750 m

Segment at Z = 458 → below threshold
    → SetActive(false) → returned to pool

New segment at Z = 756
    → pulled from pool → PlaceSegment(idx) → SetActive(true)

Runtime allocations per frame: 0
```

Segments store only their **index**. World position is always recomputed from `GetCenterX(idx × segLen)` — no stale transform state.

---

## 5. Autonomous Traffic System

### 5.1 Agent-Based Design

Each agent is **self-contained**. The orchestrator ticks agents; it does not dictate positions.

```
TrafficManager (Orchestrator)  — WHEN and HOW MANY
    └── TrafficBot × N (Agent) — WHERE exactly

TrafficBot owns:    Speed, LaneOffset, DistanceOnSpline
TrafficBot reads:   path geometry (stateless function)
TrafficBot ignores: other agents, player, score
```

```csharp
public class TrafficBot : MonoBehaviour
{
    public float Speed;
    public float LaneOffset;
    public float DistanceOnSpline;

    public void ApplyTransform()
    {
        float cx  = PathGenerator.GetCenterX(DistanceOnSpline);
        float ang = PathGenerator.GetCenterAngleY(DistanceOnSpline);
        transform.position = new Vector3(cx + LaneOffset, 0f, DistanceOnSpline);
        transform.rotation = Quaternion.Euler(0f, ang, 0f);
    }
}
```

---

### 5.2 Spline-Based Movement

```
Each frame:
  agent.DistanceOnSpline -= agent.Speed × deltaTime   ← linear progress
  agent.ApplyTransform()                              ← world position derived

Result: agents track all curves automatically.
        Changing path geometry = zero agent code changes.
```

---

### 5.3 Spatial Constraint — Clustering Prevention

```csharp
private float ResolveClusteringConflict(TrafficBot newBot, float proposed)
{
    for (int attempt = 0; attempt < MAX_ATTEMPTS; attempt++)
    {
        bool conflict = false;
        foreach (var existing in _bots)
        {
            if (!existing.Active || existing == newBot) continue;
            float gap      = Mathf.Abs(existing.DistanceOnSpline - proposed);
            bool  sameLane = Mathf.Abs(existing.LaneOffset - newBot.LaneOffset) < 0.5f;
            float required = sameLane ? MIN_SAME_LANE_GAP : MIN_CROSS_LANE_GAP;
            if (gap < required) { proposed += MIN_SAME_LANE_GAP; conflict = true; break; }
        }
        if (!conflict) break;
    }
    return proposed;
}
```

**Two thresholds:** Same-lane = head-on threat (larger gap). Cross-lane = oblique threat (smaller gap acceptable).

---

### 5.4 Zero-Allocation Respawn

```
Naive: Destroy() + Instantiate() → GC every cycle → frame spikes on mobile
This:  agent.DistanceOnSpline = playerZ + spawnOffset
       agent.ApplyTransform()  → zero allocation, same object reused
```

---

## 6. Avoidance Decision System

### 6.1 Potential Field / Slack Metric

**Slack** = signed scalar encoding lane safety. Positive = safe margin, negative = danger:

```
slack        = actual_distance − required_safety_gap
required_gap = closing_speed × maneuver_window + static_margin
maneuver_win = lane_width / lateral_speed + reaction_buffer
```

Speed-adaptive: higher speed → larger closing_speed → larger required_gap → earlier decision.

```csharp
private float GetLaneSlack(Vector3 checkPos, Vector3 forward, float mySpeed)
{
    float window = laneWidth / lateralSpeed + reactionBuffer;
    Collider[] hits = Physics.OverlapBox(checkPos, halfExtents, orientation, ...);

    float worstSlack = float.PositiveInfinity;
    foreach (var col in hits)
    {
        float dot          = Vector3.Dot(forward, toAgent.normalized);
        float closingSpeed = dot >= 0 ? (mySpeed - agent.Speed)
                                      : (agent.Speed - mySpeed);
        if (closingSpeed <= 0) continue;  // Diverging — not a threat
        float slack = distance - (closingSpeed * window + staticMargin);
        worstSlack  = Mathf.Min(worstSlack, slack);
    }
    return worstSlack;
}
```

Decision: `currentSlack < 0` triggers evaluation. Switch only when `targetSlack > currentSlack + MIN_MARGIN` AND `targetSlack >= -MAX_DEFICIT`.

---

### 6.2 Debounce + Hysteresis

```
DEBOUNCE  (time-domain): After switch, ignore for T seconds. Prevents rapid re-switching.
HYSTERESIS (value-domain): Target must exceed current by MIN_MARGIN. Prevents marginal oscillation.

Without both: left +0.1 → switch → right +0.1 → switch → (forever)
With both:    left +4.2 > MIN_MARGIN, cooldown elapsed → switch (decisive, human-like)
```

---

## 7. Player Input & Movement

### Input Layer

```csharp
// Auto-created — no .inputactions asset required
_boostAction = new InputAction(type: InputActionType.Button);
_boostAction.AddBinding("<Touchscreen>/primaryTouch/press");
_boostAction.AddBinding("<Mouse>/leftButton");
_boostAction.performed += _ => GameEvents.RaiseBoostPressed();
_boostAction.canceled  += _ => GameEvents.RaiseBoostReleased();
```

### Velocity Motor

```
currentSpeed = Lerp(cruiseSpeed, boostSpeed, stamina / staminaMax)

Stamina states:
  FILLING   (held, stamina < max)   stamina += fillRate × dt
  LOCKED    (held, stamina = max)   stamina -= drainRate × dt  [hold ignored]
  DRAINING  (released)              stamina -= drainRate × dt
  IDLE      (stamina = 0)           speed = cruiseSpeed
```

**Cruise speed invariant across all upgrade levels.** Only `boostSpeed` and stamina parameters scale. Fixed floor = consistent baseline feel at any progression stage.

### Lateral Continuity

```csharp
// Proportional to forward speed → constant steering angle at all velocities
float lateralSpeed = Mathf.Max(currentSpeed * lateralRatio, minLateralSpeed);
```

Lane changes complete in roughly constant time at all speeds — critical for avoidance timing accuracy.

---

## 8. Economy & Progression

### AnimationCurve Tuning

```csharp
[SerializeField]
private AnimationCurve boostSpeedCurve = new AnimationCurve(
    new Keyframe(  0f, BASE_BOOST),
    new Keyframe( 25f, MID_BOOST),
    new Keyframe(100f, MAX_BOOST));

public float GetBoostSpeed(int level) => boostSpeedCurve.Evaluate(level);
```

Inspector-editable without recompilation. Non-uniform progression (steep early, asymptotic late) without complex math. Curve shape communicates design intent visually.

### Cost Formula

```
cost(level) = baseCost × level ^ 1.65
```

| Exponent | Effect |
|---|---|
| 1.0 | Same cost every level — no tension |
| 2.0 | Too steep — frustration |
| **1.65** | S-curve — accessible early, aspirational late |

### Three-Category Design

| Category | Player Perception | Mechanical Effect |
|---|---|---|
| Power | "I go faster" | Boost top speed |
| Stamina | "I can hold longer" | Duration + fill rate |
| Income | "Money arrives passively" | Coins per second |

Three player psychologies: competitive / strategic / idle. Neglecting one still leaves meaningful choices in others.

---

## 9. Persistence Layer

```
SaveManager
├── Primary:   persistentDataPath/save.json  (JsonUtility)
│   Triggers:  OnApplicationPause + OnApplicationQuit
└── Secondary: PlayerPrefs  (coins, high score — fast UI reads)

PlayerData  (pure C# — no Unity imports — independently unit-testable)
    int    HighScore
    int    TotalCoins
    float  TotalDistanceTraveled
    string ActiveSkinId
    List<string>           OwnedSkinIds
    Dictionary<string,int> UpgradeLevels   // upgradeId → level
    bool   IsAdFree / HasStarterPack
    bool   SoundEnabled / MusicEnabled
```

---

## 10. Design Decisions Log

### [Path] Smoothstep vs t²

**Symptom:** Direction "snap" at straight-to-curved boundary.  
**Root cause:** `t²` derivative discontinuous at `t=1`. Players feel rate of change, not value.  
**Fix:** `t²(3-2t)` — C¹ continuity, zero derivative at both endpoints.  
**Lesson:** Always verify the derivative, not just the value.

---

### [Traffic] Two gap thresholds

**Context:** One global gap caused clustering cross-lane or excessive spacing same-lane.  
**Fix:** Independent thresholds proportional to threat geometry (head-on vs oblique).  
**Lesson:** Spatial rules should encode the geometry of the threat, not a scalar distance.

---

### [Avoidance] Debounce alone is insufficient

**Symptom:** Oscillation persisted despite cooldown timer.  
**Root cause:** After cooldown, near-equal slack values caused immediate re-switch.  
**Fix:** `minImprovementMargin` — switch only when improvement is unambiguous.  
**Lesson:** Oscillation needs two filters: temporal (debounce) + magnitude (hysteresis).

---

### [Economy] Cruise speed as invariant

**Decision:** Base speed does not scale with upgrades.  
**Rationale:** Fixed floor = consistent baseline feel. Boost contrast is sharper when the floor never moves.  
**Lesson:** Withholding a feature can create a better experience than adding it.

---

### [Traffic] Teleport vs destroy/instantiate

**Decision:** Teleport existing agent to new spawn position.  
**Rationale:** `Destroy()` + `Instantiate()` allocates every respawn → GC → frame spikes on low-end mobile.  
**Lesson:** On mobile, prefer recycle over recreate for high-frequency objects.

---

### [Near-Miss] Enter/Exit vs continuous range check

**Decision:** Award on *exit* from proximity zone, not while inside it.  
**Rationale:** Continuous check rewards hovering. The mechanic should reward *passing through* danger, not proximity.  
**Lesson:** Model player intent ("passing"), not the physical condition ("being close").

---

*This document describes architectural patterns and design rationale only.  
Project-specific gameplay details are not disclosed.*
