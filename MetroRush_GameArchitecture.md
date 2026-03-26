# 🎮 METRO RUSH — Complete Game Architecture & Implementation Guide

> Endless Runner · 3D Mobile · Unity (C#) · Production-Ready Blueprint

---

## 📁 PROJECT FOLDER STRUCTURE

```
MetroRush/
├── Assets/
│   ├── _Game/
│   │   ├── Scripts/
│   │   │   ├── Core/
│   │   │   │   ├── GameManager.cs
│   │   │   │   ├── GameState.cs
│   │   │   │   ├── SaveSystem.cs
│   │   │   │   └── EventBus.cs
│   │   │   ├── Player/
│   │   │   │   ├── PlayerController.cs
│   │   │   │   ├── PlayerAnimator.cs
│   │   │   │   ├── PlayerCollision.cs
│   │   │   │   └── SwipeInput.cs
│   │   │   ├── World/
│   │   │   │   ├── WorldGenerator.cs
│   │   │   │   ├── ChunkSpawner.cs
│   │   │   │   ├── ObstacleSpawner.cs
│   │   │   │   ├── CoinSpawner.cs
│   │   │   │   └── EnvironmentTheme.cs
│   │   │   ├── Gameplay/
│   │   │   │   ├── PowerUpSystem.cs
│   │   │   │   ├── ScoreManager.cs
│   │   │   │   ├── MissionSystem.cs
│   │   │   │   ├── DifficultyScaler.cs
│   │   │   │   └── ComboSystem.cs
│   │   │   ├── Economy/
│   │   │   │   ├── CurrencyManager.cs
│   │   │   │   ├── ShopManager.cs
│   │   │   │   ├── DailyRewardSystem.cs
│   │   │   │   └── IAPManager.cs
│   │   │   ├── UI/
│   │   │   │   ├── HUDController.cs
│   │   │   │   ├── MainMenuUI.cs
│   │   │   │   ├── GameOverUI.cs
│   │   │   │   ├── ShopUI.cs
│   │   │   │   ├── MissionsUI.cs
│   │   │   │   └── PauseMenuUI.cs
│   │   │   ├── Audio/
│   │   │   │   ├── AudioManager.cs
│   │   │   │   └── DynamicMusicController.cs
│   │   │   └── Ads/
│   │   │       ├── AdsManager.cs
│   │   │       └── RewardedAdHandler.cs
│   │   ├── Prefabs/
│   │   │   ├── Characters/
│   │   │   ├── Obstacles/
│   │   │   │   ├── Barrier.prefab
│   │   │   │   ├── Train.prefab
│   │   │   │   ├── LowBeam.prefab
│   │   │   │   ├── Gap.prefab
│   │   │   │   └── MovingCrate.prefab
│   │   │   ├── Chunks/
│   │   │   │   ├── Chunk_Straight.prefab
│   │   │   │   ├── Chunk_Bridge.prefab
│   │   │   │   ├── Chunk_Tunnel.prefab
│   │   │   │   └── Chunk_Turn.prefab
│   │   │   ├── Collectibles/
│   │   │   │   ├── Coin.prefab
│   │   │   │   └── PowerUp_[type].prefab
│   │   │   └── VFX/
│   │   ├── ScriptableObjects/
│   │   │   ├── Characters/    (CharacterData.asset)
│   │   │   ├── PowerUps/      (PowerUpData.asset)
│   │   │   ├── Themes/        (ThemeData.asset)
│   │   │   └── Missions/      (MissionData.asset)
│   │   ├── Scenes/
│   │   │   ├── Boot.unity
│   │   │   ├── MainMenu.unity
│   │   │   └── Game.unity
│   │   └── UI/
│   │       ├── Sprites/
│   │       └── Fonts/
│   ├── Plugins/
│   │   ├── GoogleMobileAds/
│   │   └── UnityPurchasing/
│   └── StreamingAssets/
└── ProjectSettings/
```

---

## 🎮 CORE SCRIPTS

### 1. GameManager.cs
```csharp
using UnityEngine;
using System;

public enum GameState { Menu, Playing, Paused, GameOver, Reviving }

public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }

    [Header("References")]
    public PlayerController player;
    public WorldGenerator world;
    public ScoreManager scoreManager;
    public HUDController hud;

    public GameState CurrentState { get; private set; }

    public event Action<GameState> OnStateChanged;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    void Start() => ChangeState(GameState.Menu);

    public void ChangeState(GameState newState)
    {
        CurrentState = newState;
        OnStateChanged?.Invoke(newState);

        switch (newState)
        {
            case GameState.Playing:
                Time.timeScale = 1f;
                world.StartGeneration();
                player.enabled = true;
                break;
            case GameState.Paused:
                Time.timeScale = 0f;
                break;
            case GameState.GameOver:
                Time.timeScale = 0f;
                SaveSystem.Instance.SaveProgress(scoreManager.CurrentScore, scoreManager.CoinsCollected);
                hud.ShowGameOver();
                break;
        }
    }

    public void StartGame()
    {
        ResetGame();
        ChangeState(GameState.Playing);
    }

    public void PauseGame() => ChangeState(GameState.Paused);
    public void ResumeGame() => ChangeState(GameState.Playing);
    public void EndGame() => ChangeState(GameState.GameOver);

    public void RevivePlayer()
    {
        int gemCost = 1;
        if (CurrencyManager.Instance.SpendGems(gemCost))
        {
            player.Revive();
            ChangeState(GameState.Playing);
        }
    }

    void ResetGame()
    {
        Time.timeScale = 1f;
        world.ResetWorld();
        scoreManager.Reset();
        player.Reset();
        PowerUpSystem.Instance.ClearAll();
    }
}
```

---

### 2. PlayerController.cs
```csharp
using UnityEngine;
using DG.Tweening;

[RequireComponent(typeof(CharacterController))]
public class PlayerController : MonoBehaviour
{
    [Header("Lane Settings")]
    public float laneWidth = 2.5f;
    public float laneChangeSpeed = 12f;
    public int totalLanes = 3;

    [Header("Jump")]
    public float jumpForce = 12f;
    public float gravity = -25f;
    public float doubleJumpForce = 10f;

    [Header("Slide")]
    public float slideDuration = 0.7f;
    public float slideCooldown = 0.2f;

    [Header("Invincibility")]
    public float invincibleDuration = 2f;

    // State
    public int CurrentLane { get; private set; } = 1; // 0=left, 1=center, 2=right
    private float targetX;
    private float verticalVelocity;
    private bool isJumping;
    private bool canDoubleJump;
    private bool isSliding;
    private bool isInvincible;
    private bool isAlive = true;
    private float slideTimer;

    private CharacterController cc;
    private PlayerAnimator anim;
    private SwipeInput swipe;

    void Awake()
    {
        cc = GetComponent<CharacterController>();
        anim = GetComponent<PlayerAnimator>();
        swipe = GetComponent<SwipeInput>();
    }

    void OnEnable()
    {
        swipe.OnSwipeLeft  += MoveLeft;
        swipe.OnSwipeRight += MoveRight;
        swipe.OnSwipeUp    += Jump;
        swipe.OnSwipeDown  += Slide;
    }

    void OnDisable()
    {
        swipe.OnSwipeLeft  -= MoveLeft;
        swipe.OnSwipeRight -= MoveRight;
        swipe.OnSwipeUp    -= Jump;
        swipe.OnSwipeDown  -= Slide;
    }

    void Update()
    {
        if (!isAlive) return;

        HandleHorizontal();
        HandleVertical();
        HandleSlide();
    }

    void HandleHorizontal()
    {
        float xPos = (CurrentLane - 1) * laneWidth;
        Vector3 pos = transform.position;
        pos.x = Mathf.Lerp(pos.x, xPos, Time.deltaTime * laneChangeSpeed);
        transform.position = pos;
    }

    void HandleVertical()
    {
        if (cc.isGrounded)
        {
            verticalVelocity = -2f;
            isJumping = false;
            canDoubleJump = true;
        }

        verticalVelocity += gravity * Time.deltaTime;

        Vector3 move = new Vector3(0, verticalVelocity, 0);
        cc.Move(move * Time.deltaTime);
    }

    void HandleSlide()
    {
        if (!isSliding) return;
        slideTimer -= Time.deltaTime;
        if (slideTimer <= 0)
        {
            isSliding = false;
            anim.StopSlide();
            GetComponent<CapsuleCollider>().height = 2f;
            GetComponent<CapsuleCollider>().center = Vector3.up;
        }
    }

    public void MoveLeft()
    {
        if (CurrentLane <= 0) return;
        CurrentLane--;
        anim.TriggerLeanLeft();
        VFXManager.Instance.SpawnLaneChangeDust(transform.position);
    }

    public void MoveRight()
    {
        if (CurrentLane >= totalLanes - 1) return;
        CurrentLane++;
        anim.TriggerLeanRight();
        VFXManager.Instance.SpawnLaneChangeDust(transform.position);
    }

    public void Jump()
    {
        if (PowerUpSystem.Instance.HasPowerUp(PowerUpType.Jetpack)) { JetpackBoost(); return; }
        if (cc.isGrounded) { verticalVelocity = jumpForce; isJumping = true; anim.TriggerJump(); }
        else if (canDoubleJump) { verticalVelocity = doubleJumpForce; canDoubleJump = false; anim.TriggerDoubleJump(); }
    }

    void JetpackBoost()
    {
        verticalVelocity = jumpForce * 1.5f;
        VFXManager.Instance.SpawnJetpackFlame(transform.position);
    }

    public void Slide()
    {
        if (!cc.isGrounded) { verticalVelocity = -jumpForce; return; } // fast fall
        if (isSliding) return;
        isSliding = true;
        slideTimer = slideDuration;
        anim.TriggerSlide();
        // Shrink collider
        GetComponent<CapsuleCollider>().height = 1f;
        GetComponent<CapsuleCollider>().center = new Vector3(0, 0.5f, 0);
    }

    public void TakeDamage()
    {
        if (isInvincible) return;
        if (PowerUpSystem.Instance.HasPowerUp(PowerUpType.Hoverboard))
        {
            PowerUpSystem.Instance.RemovePowerUp(PowerUpType.Hoverboard);
            StartCoroutine(InvincibilityCoroutine(invincibleDuration));
            VFXManager.Instance.SpawnShieldBreak(transform.position);
            return;
        }
        GameManager.Instance.EndGame();
        isAlive = false;
        anim.TriggerDeath();
        CameraController.Instance.TriggerShake(0.5f, 0.3f);
        VFXManager.Instance.SpawnCrashEffect(transform.position);
        AudioManager.Instance.PlaySFX("Crash");
    }

    System.Collections.IEnumerator InvincibilityCoroutine(float duration)
    {
        isInvincible = true;
        anim.TriggerInvincible(true);
        yield return new WaitForSeconds(duration);
        isInvincible = false;
        anim.TriggerInvincible(false);
    }

    public void Revive()
    {
        isAlive = true;
        StartCoroutine(InvincibilityCoroutine(3f));
        transform.position = new Vector3((CurrentLane - 1) * laneWidth, 1f, transform.position.z);
        anim.TriggerRevive();
    }

    public void Reset()
    {
        isAlive = true;
        CurrentLane = 1;
        verticalVelocity = 0;
        isJumping = isSliding = isInvincible = false;
        transform.position = new Vector3(0, 1f, 0);
        anim.TriggerIdle();
    }
}
```

---

### 3. SwipeInput.cs
```csharp
using UnityEngine;
using System;

public class SwipeInput : MonoBehaviour
{
    [Header("Thresholds")]
    public float minSwipeDistance = 50f;
    public float maxSwipeTime = 0.5f;
    public float tapThreshold = 10f;

    public event Action OnSwipeLeft, OnSwipeRight, OnSwipeUp, OnSwipeDown;

    private Vector2 startPos;
    private float startTime;
    private bool isSwiping;

    void Update()
    {
        HandleKeyboard(); // Editor testing
        HandleTouch();
    }

    void HandleTouch()
    {
        if (Input.touchCount == 0) return;
        Touch touch = Input.GetTouch(0);

        switch (touch.phase)
        {
            case TouchPhase.Began:
                startPos = touch.position;
                startTime = Time.unscaledTime;
                isSwiping = true;
                break;

            case TouchPhase.Ended:
                if (!isSwiping) break;
                isSwiping = false;
                float elapsed = Time.unscaledTime - startTime;
                if (elapsed > maxSwipeTime) break;
                Vector2 delta = touch.position - startPos;
                if (delta.magnitude < minSwipeDistance) break;
                FireSwipe(delta);
                break;
        }
    }

    void FireSwipe(Vector2 delta)
    {
        if (Mathf.Abs(delta.x) > Mathf.Abs(delta.y))
        {
            if (delta.x > 0) OnSwipeRight?.Invoke();
            else OnSwipeLeft?.Invoke();
        }
        else
        {
            if (delta.y > 0) OnSwipeUp?.Invoke();
            else OnSwipeDown?.Invoke();
        }
    }

    void HandleKeyboard()
    {
        if (Input.GetKeyDown(KeyCode.LeftArrow))  OnSwipeLeft?.Invoke();
        if (Input.GetKeyDown(KeyCode.RightArrow)) OnSwipeRight?.Invoke();
        if (Input.GetKeyDown(KeyCode.UpArrow) || Input.GetKeyDown(KeyCode.Space)) OnSwipeUp?.Invoke();
        if (Input.GetKeyDown(KeyCode.DownArrow))  OnSwipeDown?.Invoke();
    }
}
```

---

### 4. WorldGenerator.cs
```csharp
using UnityEngine;
using System.Collections.Generic;

public class WorldGenerator : MonoBehaviour
{
    [Header("Chunk Prefabs")]
    public GameObject[] chunkPrefabs;
    public GameObject[] bridgePrefabs;
    public GameObject[] tunnelPrefabs;

    [Header("Settings")]
    public int chunksAhead = 5;
    public float chunkLength = 30f;
    public Transform playerTransform;

    [Header("Theming")]
    public ThemeData[] themes;
    private int currentThemeIndex;
    private float themeDistance = 500f; // Change theme every 500m

    private Queue<GameObject> activeChunks = new Queue<GameObject>();
    private float nextChunkZ = 0f;
    private bool isGenerating;

    public void StartGeneration()
    {
        isGenerating = true;
        nextChunkZ = 0f;
        ClearChunks();
        for (int i = 0; i < chunksAhead + 2; i++) SpawnChunk();
    }

    public void ResetWorld()
    {
        isGenerating = false;
        ClearChunks();
        nextChunkZ = 0f;
        currentThemeIndex = 0;
        ApplyTheme(themes[0]);
    }

    void Update()
    {
        if (!isGenerating) return;
        
        // Spawn new chunks ahead
        while (nextChunkZ < playerTransform.position.z + chunksAhead * chunkLength)
            SpawnChunk();
        
        // Remove chunks that are behind the player
        while (activeChunks.Count > 0 &&
               activeChunks.Peek().transform.position.z < playerTransform.position.z - chunkLength * 2)
        {
            Destroy(activeChunks.Dequeue());
        }

        // Theme transitions
        int targetTheme = Mathf.FloorToInt(playerTransform.position.z / themeDistance) % themes.Length;
        if (targetTheme != currentThemeIndex) TransitionTheme(targetTheme);
    }

    void SpawnChunk()
    {
        // Select chunk type with weighted randomness
        GameObject prefab = SelectChunkPrefab();
        GameObject chunk = Instantiate(prefab, new Vector3(0, 0, nextChunkZ), Quaternion.identity);
        chunk.GetComponent<ChunkSpawner>()?.Initialize(currentThemeIndex);
        activeChunks.Enqueue(chunk);
        nextChunkZ += chunkLength;
    }

    GameObject SelectChunkPrefab()
    {
        float rand = Random.value;
        if (rand < 0.7f) return chunkPrefabs[Random.Range(0, chunkPrefabs.Length)];
        if (rand < 0.85f) return bridgePrefabs[Random.Range(0, bridgePrefabs.Length)];
        return tunnelPrefabs[Random.Range(0, tunnelPrefabs.Length)];
    }

    void TransitionTheme(int newIndex)
    {
        currentThemeIndex = newIndex;
        // Use DOTween or coroutine to blend material properties
        ApplyTheme(themes[newIndex]);
        HUDController.Instance.ShowLocationBanner(themes[newIndex].locationName);
    }

    void ApplyTheme(ThemeData theme)
    {
        RenderSettings.fogColor = theme.fogColor;
        RenderSettings.fogDensity = theme.fogDensity;
        RenderSettings.ambientLight = theme.ambientColor;
        Camera.main.backgroundColor = theme.skyColor;
    }

    void ClearChunks()
    {
        while (activeChunks.Count > 0)
        {
            var c = activeChunks.Dequeue();
            if (c) Destroy(c);
        }
    }
}
```

---

### 5. ObstacleSpawner.cs
```csharp
using UnityEngine;
using System.Collections.Generic;

public class ObstacleSpawner : MonoBehaviour
{
    [System.Serializable]
    public class ObstaclePattern
    {
        public string name;
        public int[] blockedLanes;   // 0=left, 1=center, 2=right
        public float minDifficulty;
        public float weight = 1f;
        public bool requiresJump;
        public bool requiresSlide;
        public bool requiresLaneChange;
    }

    public ObstaclePattern[] patterns;
    public GameObject[] obstaclePrefabs;
    public float spawnLookAhead = 40f;

    private DifficultyScaler difficulty;
    private float lastSpawnZ;

    // Smart spawn: ensure at least one safe path exists
    public void SpawnObstaclesForChunk(float chunkStartZ, float chunkLength)
    {
        float difficultyLevel = DifficultyScaler.Instance.CurrentDifficulty;
        
        var validPatterns = System.Array.FindAll(patterns, p => p.minDifficulty <= difficultyLevel);
        if (validPatterns.Length == 0) return;

        int numObstacleGroups = Mathf.RoundToInt(1 + difficultyLevel * 2);
        float spacing = chunkLength / (numObstacleGroups + 1);

        for (int i = 0; i < numObstacleGroups; i++)
        {
            float z = chunkStartZ + spacing * (i + 1);
            ObstaclePattern pattern = SelectWeightedPattern(validPatterns);
            SpawnPattern(pattern, z);
        }
    }

    void SpawnPattern(ObstaclePattern pattern, float z)
    {
        // Verify pattern is beatable
        List<int> safeLanes = new List<int> { 0, 1, 2 };
        foreach (int blocked in pattern.blockedLanes) safeLanes.Remove(blocked);
        if (safeLanes.Count == 0) return; // Safety check - always need a path

        foreach (int lane in pattern.blockedLanes)
        {
            float xPos = (lane - 1) * 2.5f;
            // Select appropriate prefab for pattern
            GameObject prefab = obstaclePrefabs[Random.Range(0, obstaclePrefabs.Length)];
            GameObject obs = Instantiate(prefab, new Vector3(xPos, 0, z), Quaternion.identity, transform);
        }
    }

    ObstaclePattern SelectWeightedPattern(ObstaclePattern[] candidates)
    {
        float total = 0;
        foreach (var p in candidates) total += p.weight;
        float rand = Random.Range(0, total);
        foreach (var p in candidates)
        {
            rand -= p.weight;
            if (rand <= 0) return p;
        }
        return candidates[0];
    }
}
```

---

### 6. PowerUpSystem.cs
```csharp
using UnityEngine;
using System.Collections.Generic;
using System;

public enum PowerUpType { CoinMagnet, Jetpack, ScoreMultiplier, Hoverboard, SpeedBoost }

[System.Serializable]
public class PowerUpConfig
{
    public PowerUpType type;
    public float duration;
    public GameObject vfxPrefab;
    public Sprite icon;
    public Color color;
    [TextArea] public string description;
}

public class PowerUpSystem : MonoBehaviour
{
    public static PowerUpSystem Instance { get; private set; }

    public PowerUpConfig[] configs;

    private Dictionary<PowerUpType, float> activePowerUps = new Dictionary<PowerUpType, float>();
    private Dictionary<PowerUpType, GameObject> activeVFX = new Dictionary<PowerUpType, GameObject>();

    public event Action<PowerUpType, float> OnPowerUpActivated;
    public event Action<PowerUpType> OnPowerUpExpired;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
    }

    void Update()
    {
        List<PowerUpType> toRemove = new List<PowerUpType>();
        foreach (var kv in activePowerUps)
        {
            activePowerUps[kv.Key] -= Time.deltaTime;
            HUDController.Instance.UpdatePowerUpTimer(kv.Key, kv.Value / GetConfig(kv.Key).duration);
            if (kv.Value <= 0) toRemove.Add(kv.Key);
        }
        foreach (var t in toRemove) RemovePowerUp(t);
    }

    public void ActivatePowerUp(PowerUpType type)
    {
        PowerUpConfig cfg = GetConfig(type);
        if (cfg == null) return;

        // Refresh if already active
        activePowerUps[type] = cfg.duration;

        // Apply effects
        switch (type)
        {
            case PowerUpType.CoinMagnet:
                CoinSpawner.Instance.SetMagnetRange(8f);
                break;
            case PowerUpType.Jetpack:
                GameManager.Instance.player.EnableJetpack(true);
                break;
            case PowerUpType.ScoreMultiplier:
                ScoreManager.Instance.SetMultiplier(2);
                break;
            case PowerUpType.Hoverboard:
                HUDController.Instance.ShowHoverboard(true);
                break;
            case PowerUpType.SpeedBoost:
                DifficultyScaler.Instance.ApplyTemporarySpeedBoost(1.5f, cfg.duration);
                break;
        }

        // VFX
        if (cfg.vfxPrefab)
        {
            if (activeVFX.TryGetValue(type, out var existing)) Destroy(existing);
            activeVFX[type] = Instantiate(cfg.vfxPrefab, GameManager.Instance.player.transform);
        }

        OnPowerUpActivated?.Invoke(type, cfg.duration);
        AudioManager.Instance.PlaySFX("PowerUp_" + type.ToString());
        spawnFloatingText("+" + type.ToString().ToUpper(), Color.yellow);
    }

    public void RemovePowerUp(PowerUpType type)
    {
        activePowerUps.Remove(type);
        if (activeVFX.TryGetValue(type, out var vfx)) { Destroy(vfx); activeVFX.Remove(type); }

        switch (type)
        {
            case PowerUpType.CoinMagnet: CoinSpawner.Instance.SetMagnetRange(0.5f); break;
            case PowerUpType.Jetpack: GameManager.Instance.player.EnableJetpack(false); break;
            case PowerUpType.ScoreMultiplier: ScoreManager.Instance.SetMultiplier(1); break;
            case PowerUpType.Hoverboard: HUDController.Instance.ShowHoverboard(false); break;
        }

        OnPowerUpExpired?.Invoke(type);
    }

    public bool HasPowerUp(PowerUpType type) => activePowerUps.ContainsKey(type) && activePowerUps[type] > 0;
    public void ClearAll() { foreach (var t in new List<PowerUpType>(activePowerUps.Keys)) RemovePowerUp(t); }

    PowerUpConfig GetConfig(PowerUpType type) => System.Array.Find(configs, c => c.type == type);

    void spawnFloatingText(string text, Color color)
    {
        FloatingTextManager.Instance?.Spawn(text, GameManager.Instance.player.transform.position + Vector3.up * 2, color);
    }
}
```

---

### 7. ScoreManager.cs
```csharp
using UnityEngine;
using System;

public class ScoreManager : MonoBehaviour
{
    public static ScoreManager Instance { get; private set; }

    public int CurrentScore { get; private set; }
    public int CoinsCollected { get; private set; }
    public float DistanceTravelled { get; private set; }
    public int Multiplier { get; private set; } = 1;

    public event Action<int> OnScoreChanged;
    public event Action<int> OnCoinCollected;

    private float scoreAccumulator;
    private const float SCORE_PER_METER = 1f;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
    }

    void Update()
    {
        if (GameManager.Instance?.CurrentState != GameState.Playing) return;

        float delta = DifficultyScaler.Instance.CurrentSpeed * Time.deltaTime;
        DistanceTravelled += delta;
        scoreAccumulator += SCORE_PER_METER * Multiplier * (1 + DifficultyScaler.Instance.CurrentDifficulty);

        if (scoreAccumulator >= 1)
        {
            int toAdd = Mathf.FloorToInt(scoreAccumulator);
            CurrentScore += toAdd;
            scoreAccumulator -= toAdd;
            OnScoreChanged?.Invoke(CurrentScore);
        }
    }

    public void AddCoin(int value = 1)
    {
        int bonus = value * Multiplier;
        CoinsCollected += bonus;
        CurrentScore += bonus * 10;
        OnScoreChanged?.Invoke(CurrentScore);
        OnCoinCollected?.Invoke(CoinsCollected);
        MissionSystem.Instance?.ReportEvent(MissionEventType.CoinCollected, bonus);
    }

    public void SetMultiplier(int mult)
    {
        Multiplier = mult;
        HUDController.Instance?.SetMultiplierDisplay(mult);
    }

    public void Reset()
    {
        CurrentScore = 0; CoinsCollected = 0;
        DistanceTravelled = 0; Multiplier = 1;
        scoreAccumulator = 0;
    }
}
```

---

### 8. DifficultyScaler.cs
```csharp
using UnityEngine;

public class DifficultyScaler : MonoBehaviour
{
    public static DifficultyScaler Instance { get; private set; }

    [Header("Speed Curve")]
    public float baseSpeed = 8f;
    public float maxSpeed = 25f;
    public float speedRampDuration = 300f; // seconds to reach max speed

    [Header("Difficulty Curve")]
    public AnimationCurve difficultyCurve; // 0-1 over time

    public float CurrentSpeed { get; private set; }
    public float CurrentDifficulty { get; private set; }

    private float gameElapsed;
    private float speedBoostMultiplier = 1f;
    private float speedBoostTimer;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
    }

    void Update()
    {
        if (GameManager.Instance?.CurrentState != GameState.Playing) return;

        gameElapsed += Time.deltaTime;

        // Speed ramp (smooth logarithmic curve)
        float t = Mathf.Clamp01(gameElapsed / speedRampDuration);
        float smoothT = 1f - Mathf.Pow(1f - t, 2f); // ease-in quad
        CurrentSpeed = Mathf.Lerp(baseSpeed, maxSpeed, smoothT) * speedBoostMultiplier;
        CurrentDifficulty = difficultyCurve.Evaluate(t);

        // Speed boost timer
        if (speedBoostTimer > 0)
        {
            speedBoostTimer -= Time.deltaTime;
            if (speedBoostTimer <= 0) speedBoostMultiplier = 1f;
        }
    }

    public void ApplyTemporarySpeedBoost(float multiplier, float duration)
    {
        speedBoostMultiplier = multiplier;
        speedBoostTimer = duration;
    }

    public void Reset()
    {
        gameElapsed = 0;
        speedBoostMultiplier = 1f;
        speedBoostTimer = 0;
        CurrentSpeed = baseSpeed;
        CurrentDifficulty = 0;
    }
}
```

---

### 9. MissionSystem.cs
```csharp
using UnityEngine;
using System.Collections.Generic;

public enum MissionEventType
{
    CoinCollected, DistanceTravelled, PowerUpCollected, ObstacleJumped,
    ObstacleSlid, NearMiss, RunCompleted, CharacterUsed
}

[CreateAssetMenu(fileName = "MissionData", menuName = "MetroRush/Mission")]
public class MissionData : ScriptableObject
{
    public string title;
    public string description;
    public MissionEventType eventType;
    public int targetCount;
    public int coinReward;
    public int gemReward;
    public Sprite icon;
    public bool isDaily;
}

public class MissionSystem : MonoBehaviour
{
    public static MissionSystem Instance { get; private set; }

    public MissionData[] allMissions;
    private List<MissionData> activeMissions = new List<MissionData>();
    private Dictionary<string, int> missionProgress = new Dictionary<string, int>();

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        LoadMissions();
    }

    void LoadMissions()
    {
        // Select 3 daily missions
        activeMissions.Clear();
        var shuffled = new List<MissionData>(allMissions);
        shuffled.Sort((a, b) => Random.value.CompareTo(0.5f));
        activeMissions.AddRange(shuffled.GetRange(0, Mathf.Min(3, shuffled.Count)));
        foreach (var m in activeMissions)
            missionProgress[m.name] = SaveSystem.Instance.GetMissionProgress(m.name);
    }

    public void ReportEvent(MissionEventType type, int amount = 1)
    {
        foreach (var mission in activeMissions)
        {
            if (mission.eventType != type) continue;
            if (!missionProgress.ContainsKey(mission.name)) missionProgress[mission.name] = 0;
            missionProgress[mission.name] += amount;
            SaveSystem.Instance.SetMissionProgress(mission.name, missionProgress[mission.name]);
            
            HUDController.Instance?.UpdateMissionProgress(mission, missionProgress[mission.name]);

            if (missionProgress[mission.name] >= mission.targetCount)
                CompleteMission(mission);
        }
    }

    void CompleteMission(MissionData mission)
    {
        CurrencyManager.Instance.AddCoins(mission.coinReward);
        CurrencyManager.Instance.AddGems(mission.gemReward);
        HUDController.Instance?.ShowMissionComplete(mission);
        activeMissions.Remove(mission);
        AudioManager.Instance.PlaySFX("MissionComplete");
    }
}
```

---

### 10. SaveSystem.cs
```csharp
using UnityEngine;
using System.Collections.Generic;

[System.Serializable]
public class PlayerSaveData
{
    public int coins;
    public int gems;
    public int highScore;
    public float totalDistance;
    public List<string> unlockedCharacters = new List<string>();
    public string equippedCharacter;
    public int level;
    public int xp;
    public string lastDailyReward;
    public Dictionary<string, int> missionProgress = new Dictionary<string, int>();
}

public class SaveSystem : MonoBehaviour
{
    public static SaveSystem Instance { get; private set; }

    private PlayerSaveData data;
    private const string SAVE_KEY = "PlayerSaveData";

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
        Load();
    }

    public void Load()
    {
        string json = PlayerPrefs.GetString(SAVE_KEY, "");
        data = string.IsNullOrEmpty(json) ? new PlayerSaveData() : JsonUtility.FromJson<PlayerSaveData>(json);
        // TODO: Add cloud save via Unity Gaming Services
    }

    public void Save()
    {
        PlayerPrefs.SetString(SAVE_KEY, JsonUtility.ToJson(data));
        PlayerPrefs.Save();
    }

    public void SaveProgress(int score, int runCoins)
    {
        if (score > data.highScore) data.highScore = score;
        data.coins += runCoins;
        data.totalDistance += ScoreManager.Instance?.DistanceTravelled ?? 0;
        AddXP(Mathf.FloorToInt(score / 100));
        Save();
    }

    void AddXP(int amount)
    {
        data.xp += amount;
        int xpNeeded = 100 * (data.level + 1);
        if (data.xp >= xpNeeded) { data.level++; data.xp -= xpNeeded; HUDController.Instance?.ShowLevelUp(data.level); }
    }

    // Getters / Setters
    public int GetCoins() => data.coins;
    public int GetGems() => data.gems;
    public int GetHighScore() => data.highScore;
    public int GetLevel() => data.level;
    public bool IsCharacterUnlocked(string id) => data.unlockedCharacters.Contains(id);
    public string GetEquippedCharacter() => data.equippedCharacter ?? "default";
    public int GetMissionProgress(string id) => data.missionProgress.TryGetValue(id, out int v) ? v : 0;
    public void SetMissionProgress(string id, int val) { data.missionProgress[id] = val; Save(); }
    public void UnlockCharacter(string id) { if (!data.unlockedCharacters.Contains(id)) { data.unlockedCharacters.Add(id); Save(); } }
    public void SetEquippedCharacter(string id) { data.equippedCharacter = id; Save(); }
    public bool CanClaimDailyReward() => data.lastDailyReward != System.DateTime.Now.Date.ToString("yyyy-MM-dd");
    public void ClaimDailyReward() { data.lastDailyReward = System.DateTime.Now.Date.ToString("yyyy-MM-dd"); Save(); }
}
```

---

### 11. CurrencyManager.cs
```csharp
using UnityEngine;

public class CurrencyManager : MonoBehaviour
{
    public static CurrencyManager Instance { get; private set; }

    public int Coins { get; private set; }
    public int Gems { get; private set; }

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        Coins = SaveSystem.Instance.GetCoins();
        Gems = SaveSystem.Instance.GetGems();
    }

    public void AddCoins(int amount)
    {
        Coins += amount;
        HUDController.Instance?.UpdateCurrency(Coins, Gems);
        SaveSystem.Instance.Save();
    }

    public bool SpendCoins(int amount)
    {
        if (Coins < amount) return false;
        Coins -= amount;
        HUDController.Instance?.UpdateCurrency(Coins, Gems);
        SaveSystem.Instance.Save();
        return true;
    }

    public void AddGems(int amount) { Gems += amount; HUDController.Instance?.UpdateCurrency(Coins, Gems); SaveSystem.Instance.Save(); }
    public bool SpendGems(int amount)
    {
        if (Gems < amount) return false;
        Gems -= amount;
        HUDController.Instance?.UpdateCurrency(Coins, Gems);
        SaveSystem.Instance.Save();
        return true;
    }
}
```

---

### 12. AudioManager.cs
```csharp
using UnityEngine;
using System.Collections.Generic;

public class AudioManager : MonoBehaviour
{
    public static AudioManager Instance { get; private set; }

    [System.Serializable]
    public class SoundClip { public string name; public AudioClip clip; [Range(0,1)] public float volume = 1f; public bool loop; }

    public SoundClip[] sounds;
    public AudioSource musicSource;
    public AudioSource sfxSource;
    public AudioClip[] musicTracks;

    private Dictionary<string, SoundClip> soundMap = new Dictionary<string, SoundClip>();
    private int currentTrack;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
        foreach (var s in sounds) soundMap[s.name] = s;
    }

    void Start() => PlayMusic(0);

    public void PlaySFX(string name)
    {
        if (!soundMap.TryGetValue(name, out var clip)) return;
        sfxSource.PlayOneShot(clip.clip, clip.volume);
    }

    public void PlayMusic(int index)
    {
        currentTrack = index % musicTracks.Length;
        musicSource.clip = musicTracks[currentTrack];
        musicSource.loop = true;
        musicSource.Play();
    }

    // Dynamic music: pitch up with speed
    public void SetMusicIntensity(float normalized)
    {
        musicSource.pitch = Mathf.Lerp(1f, 1.25f, normalized);
        musicSource.volume = Mathf.Lerp(0.7f, 1f, normalized);
    }

    public void SetMuted(bool muted) { musicSource.mute = muted; sfxSource.mute = muted; }
}
```

---

### 13. HUDController.cs
```csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class HUDController : MonoBehaviour
{
    public static HUDController Instance { get; private set; }

    [Header("In-Game HUD")]
    public TextMeshProUGUI scoreText;
    public TextMeshProUGUI coinsText;
    public TextMeshProUGUI multiplierText;
    public TextMeshProUGUI distanceText;
    public Slider[] powerUpTimers;
    public GameObject hoverboardIndicator;
    public GameObject missionProgressPanel;
    public TextMeshProUGUI missionProgressText;

    [Header("Screens")]
    public GameObject gameOverPanel;
    public GameObject pausePanel;
    public TextMeshProUGUI finalScoreText;
    public TextMeshProUGUI finalCoinsText;
    public TextMeshProUGUI finalDistText;
    public TextMeshProUGUI hiScoreText;

    [Header("Notifications")]
    public GameObject missionCompletePanel;
    public TextMeshProUGUI missionCompleteText;
    public GameObject levelUpPanel;
    public TextMeshProUGUI levelUpText;
    public TextMeshProUGUI locationBannerText;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
    }

    void Update()
    {
        if (GameManager.Instance?.CurrentState != GameState.Playing) return;
        scoreText.text = ScoreManager.Instance.CurrentScore.ToString("N0");
        distanceText.text = Mathf.FloorToInt(ScoreManager.Instance.DistanceTravelled) + "m";
        // Dynamic music intensity
        float speedNorm = Mathf.InverseLerp(8, 25, DifficultyScaler.Instance.CurrentSpeed);
        AudioManager.Instance.SetMusicIntensity(speedNorm);
    }

    public void SetMultiplierDisplay(int mult)
    {
        multiplierText.text = mult > 1 ? $"×{mult}" : "";
        multiplierText.gameObject.SetActive(mult > 1);
    }

    public void UpdatePowerUpTimer(PowerUpType type, float normalized)
    {
        // Update relevant slider
    }

    public void ShowHoverboard(bool show) => hoverboardIndicator.SetActive(show);

    public void ShowGameOver()
    {
        gameOverPanel.SetActive(true);
        finalScoreText.text = ScoreManager.Instance.CurrentScore.ToString("N0");
        finalCoinsText.text = ScoreManager.Instance.CoinsCollected.ToString();
        finalDistText.text = Mathf.FloorToInt(ScoreManager.Instance.DistanceTravelled) + "m";
        hiScoreText.text = "Best: " + SaveSystem.Instance.GetHighScore().ToString("N0");
    }

    public void ShowMissionComplete(MissionData m) => StartCoroutine(ShowBannerCoroutine(missionCompletePanel, missionCompleteText, "✓ " + m.title, 2.5f));
    public void ShowLevelUp(int level) => StartCoroutine(ShowBannerCoroutine(levelUpPanel, levelUpText, "LEVEL " + level + "!", 2f));
    public void ShowLocationBanner(string loc) => StartCoroutine(ShowBannerCoroutine(null, locationBannerText, loc, 3f));
    public void UpdateCurrency(int coins, int gems) { coinsText.text = coins.ToString("N0"); }
    public void UpdateMissionProgress(MissionData m, int prog) { }

    System.Collections.IEnumerator ShowBannerCoroutine(GameObject panel, TextMeshProUGUI label, string text, float duration)
    {
        if (label) label.text = text;
        if (panel) panel.SetActive(true);
        yield return new WaitForSeconds(duration);
        if (panel) panel.SetActive(false);
    }
}
```

---

## 🎨 SCRIPTABLE OBJECTS

### CharacterData.cs
```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "CharacterData", menuName = "MetroRush/Character")]
public class CharacterData : ScriptableObject
{
    public string characterId;
    public string displayName;
    public Sprite icon;
    public GameObject prefab;
    public int coinCost;
    public int gemCost;
    public bool isStartCharacter;
    [TextArea] public string backstory;
}
```

### ThemeData.cs
```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "ThemeData", menuName = "MetroRush/Theme")]
public class ThemeData : ScriptableObject
{
    public string locationName;
    public Color skyColor;
    public Color fogColor;
    public float fogDensity;
    public Color ambientColor;
    public Material roadMaterial;
    public Material buildingMaterial;
    public GameObject[] exclusiveObstacles;
}
```

---

## 🛒 SHOP & IAP

### ShopManager.cs
```csharp
using UnityEngine;
using System.Collections.Generic;

[System.Serializable]
public class ShopItem
{
    public string itemId;
    public string displayName;
    public Sprite icon;
    public int coinPrice;
    public int gemPrice;
    public string productId; // IAP product ID
    public ShopItemType itemType;
}

public enum ShopItemType { Character, Skin, PowerUpUpgrade, CoinPack, GemPack, RemoveAds }

public class ShopManager : MonoBehaviour
{
    public static ShopManager Instance { get; private set; }
    public ShopItem[] shopItems;

    public bool PurchaseWithCoins(ShopItem item)
    {
        if (!CurrencyManager.Instance.SpendCoins(item.coinPrice)) return false;
        GrantItem(item);
        return true;
    }

    public bool PurchaseWithGems(ShopItem item)
    {
        if (!CurrencyManager.Instance.SpendGems(item.gemPrice)) return false;
        GrantItem(item);
        return true;
    }

    void GrantItem(ShopItem item)
    {
        switch (item.itemType)
        {
            case ShopItemType.Character:
                SaveSystem.Instance.UnlockCharacter(item.itemId);
                break;
            case ShopItemType.CoinPack:
                CurrencyManager.Instance.AddCoins(GetCoinPackAmount(item.itemId));
                break;
            case ShopItemType.GemPack:
                CurrencyManager.Instance.AddGems(GetGemPackAmount(item.itemId));
                break;
        }
        AudioManager.Instance.PlaySFX("Purchase");
    }

    int GetCoinPackAmount(string id) => id switch { "coins_small" => 500, "coins_medium" => 1500, "coins_large" => 5000, _ => 0 };
    int GetGemPackAmount(string id) => id switch { "gems_small" => 20, "gems_medium" => 75, "gems_large" => 200, _ => 0 };
}
```

---

## 📈 MONETIZATION CONFIG

### IAP Products
```
Product ID                  Price       Type
com.metrorush.gems.small    $0.99       Consumable (20 gems)
com.metrorush.gems.medium   $2.99       Consumable (75 gems)
com.metrorush.gems.large    $9.99       Consumable (200 gems)
com.metrorush.noads         $3.99       Non-consumable
com.metrorush.starter_pack  $1.99       Non-consumable (exclusive skin + 50 gems)
```

### AdsManager.cs (Unity Ads Integration)
```csharp
using UnityEngine;
using UnityEngine.Advertisements;

public class AdsManager : MonoBehaviour, IUnityAdsLoadListener, IUnityAdsShowListener
{
    public static AdsManager Instance { get; private set; }

    const string ANDROID_GAME_ID = "YOUR_ANDROID_ID";
    const string IOS_GAME_ID = "YOUR_IOS_ID";
    const string REWARDED_AD_ID = "Rewarded_Android";

    private bool adsEnabled = true;
    private System.Action<bool> onRewardCallback;

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
        adsEnabled = !PlayerPrefs.HasKey("NoAds");
    }

    void Start()
    {
        if (!adsEnabled) return;
        string gameId = Application.platform == RuntimePlatform.IPhonePlayer ? IOS_GAME_ID : ANDROID_GAME_ID;
        Advertisement.Initialize(gameId, false); // false = production
        LoadRewardedAd();
    }

    public void LoadRewardedAd() => Advertisement.Load(REWARDED_AD_ID, this);

    public void ShowRewardedAd(System.Action<bool> callback)
    {
        onRewardCallback = callback;
        Advertisement.Show(REWARDED_AD_ID, this);
    }

    public void OnUnityAdsShowComplete(string adUnitId, UnityAdsShowCompletionState state)
    {
        bool rewarded = state == UnityAdsShowCompletionState.COMPLETED;
        onRewardCallback?.Invoke(rewarded);
        LoadRewardedAd(); // Pre-load next ad
    }

    public void RemoveAds() { adsEnabled = false; PlayerPrefs.SetInt("NoAds", 1); }

    // IUnityAdsLoadListener
    public void OnUnityAdsAdLoaded(string id) { }
    public void OnUnityAdsFailedToLoad(string id, UnityAdsLoadError err, string msg) => Debug.LogWarning($"Ad load failed: {msg}");
    public void OnUnityAdsShowFailure(string id, UnityAdsShowError err, string msg) => onRewardCallback?.Invoke(false);
    public void OnUnityAdsShowStart(string id) { }
    public void OnUnityAdsShowClick(string id) { }
}
```

---

## 🎯 IMPLEMENTATION GUIDE

### Phase 1 — Core Loop (Week 1-2)
1. Set up Unity 2022 LTS project with URP
2. Implement PlayerController with lane switching, jump, slide
3. Create basic WorldGenerator with 3 chunk types
4. Add ObstacleSpawner with 3 obstacle types
5. Basic coin pickup + scoring
6. Game loop: Menu → Play → GameOver → Retry

### Phase 2 — Feel & Polish (Week 3)
1. Add all 5 power-ups with VFX
2. Camera shake, particle effects, floating text
3. Player animations (run cycle, jump, slide, death)
4. DifficultyScaler smooth speed ramp
5. Procedural coin patterns
6. Audio: SFX + dynamic music

### Phase 3 — Progression (Week 4)
1. SaveSystem (local + cloud)
2. MissionSystem (daily + weekly)
3. CurrencyManager (coins + gems)
4. Character unlock system (3+ characters)
5. DailyRewardSystem
6. Level + XP system

### Phase 4 — Monetization (Week 5)
1. Unity IAP integration
2. Unity Ads (rewarded video for revive + coin doubler)
3. ShopUI (characters, skins, upgrades)
4. Remove Ads IAP

### Phase 5 — Retention Features (Week 6)
1. Leaderboards (Unity Gaming Services)
2. Social sharing (screenshots via NativeShare)
3. Haptic feedback (iOS/Android)
4. Tutorial overlay for new players
5. App Store / Play Store submission

---

## 📦 REQUIRED PACKAGES

```
Unity 2022.3 LTS (URP)
├── TextMeshPro
├── DOTween (animations)
├── Unity Gaming Services (leaderboards, cloud save)
├── Unity Ads
├── Unity IAP
├── NativeShare (screenshot sharing)
├── UniTask (async/await)
└── Cinemachine (camera)
```

---

## ⚡ PERFORMANCE TARGETS

| Metric            | Target          |
|-------------------|-----------------|
| Frame Rate        | 60 FPS (all devices) |
| Draw Calls        | < 80 per frame  |
| Memory            | < 300 MB        |
| Load Time         | < 3 seconds     |
| APK Size          | < 80 MB         |
| Battery Drain     | Moderate        |

### Key Optimizations
- Object pooling for chunks, obstacles, coins, particles
- LOD groups on buildings/environment
- Texture atlasing for obstacles
- IL2CPP + ARM64 target
- Occlusion culling on world chunks
- Addressables for theme asset bundles

---

## 🎨 ASSET STYLE GUIDE

- **Art Style**: Cartoon / stylized (not realistic)
- **Color Palette**: Vibrant, high-contrast, 90% saturation
- **Character Ratio**: 1:2 head-body (chibi-ish)
- **Polygon Budget**: ~800 tris per character, ~2000 per chunk
- **Texture Resolution**: 512×512 per character, 1024×1024 for environment
- **Animation FPS**: 24 fps, loop-ready
- **UI Font**: Rounded, bold — e.g. Nunito ExtraBold or Fredoka One

---

*MetroRush Architecture v1.0 | Production Blueprint*
