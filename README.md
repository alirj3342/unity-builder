// OpenWorldMobileDemo.cs
// Single-file Unity C# demo for a mobile open-world prototype.
// Attach to an empty GameObject in a blank scene. Press Play.
// Creates player, camera, simple touch controls, NPC "gang" with patrol/chase,
// and some procedural obstacles/buildings. Self-contained.

using UnityEngine;
using UnityEngine.UI;
using System.Collections.Generic;

public class OpenWorldMobileDemo : MonoBehaviour
{
    // Adjustable parameters
    public int worldSize = 80;
    public int npcCount = 8;
    public int buildingCount = 12;
    public float npcSightRange = 10f;
    public float npcChaseSpeed = 3.2f;
    public float npcPatrolSpeed = 1.5f;
    public float playerSpeed = 4.5f;
    public float playerRunMultiplier = 1.6f;
    public float attackRange = 1.8f;
    public int playerMaxHealth = 100;

    // Internal references
    GameObject player;
    CharacterController playerController;
    Camera mainCam;
    Text debugText;
    Text healthText;
    List<NPC> npcs = new List<NPC>();
    int playerHealth;

    // Touch / virtual joystick state
    Vector2 joystickStart;
    Vector2 joystickDelta;
    bool joystickActive = false;

    void Start()
    {
        Application.targetFrameRate = 60;
        playerHealth = playerMaxHealth;
        // Create flat ground
        CreateGround();

        // Spawn some buildings/obstacles
        for (int i = 0; i < buildingCount; i++) SpawnBuilding();

        // Create player
        CreatePlayer();

        // Create camera
        CreateCamera();

        // Spawn NPCs
        for (int i = 0; i < npcCount; i++) SpawnNPC();

        // Create simple UI
        CreateUI();
    }

    void Update()
    {
        HandleTouchMovement();
        UpdateNPCs();
        UpdateUI();
    }

    #region Scene Setup
    void CreateGround()
    {
        GameObject ground = GameObject.CreatePrimitive(PrimitiveType.Plane);
        ground.transform.localScale = new Vector3(worldSize / 10f, 1, worldSize / 10f);
        ground.name = "Ground";
        var mat = new Material(Shader.Find("Standard"));
        mat.color = new Color(0.6f, 0.9f, 0.6f);
        ground.GetComponent<Renderer>().material = mat;
    }

    void SpawnBuilding()
    {
        Vector3 pos = RandomNavPosition();
        GameObject b = GameObject.CreatePrimitive(PrimitiveType.Cube);
        float sx = Random.Range(3f, 8f);
        float sy = Random.Range(3f, 12f);
        float sz = Random.Range(3f, 8f);
        b.transform.position = new Vector3(pos.x, sy / 2f, pos.z);
        b.transform.localScale = new Vector3(sx, sy, sz);
        b.GetComponent<Renderer>().material.color = new Color(Random.value * 0.6f, Random.value * 0.6f, Random.value * 0.6f);
        b.name = "Building";
        b.AddComponent<BoxCollider>();
    }

    void CreatePlayer()
    {
        player = GameObject.CreatePrimitive(PrimitiveType.Capsule);
        player.name = "Player";
        player.transform.position = Vector3.zero + Vector3.up * 1.0f;
        player.AddComponent<Rigidbody>().isKinematic = true; // we'll use CharacterController
        playerController = player.AddComponent<CharacterController>();
        playerController.height = 1.6f;
        playerController.radius = 0.4f;

        // simple visual
        var mr = player.GetComponent<Renderer>();
        if (mr != null) mr.material.color = Color.cyan;
    }

    void CreateCamera()
    {
        GameObject camObj = new GameObject("Main Camera");
        mainCam = camObj.AddComponent<Camera>();
        mainCam.transform.position = player.transform.position + new Vector3(0f, 6f, -8f);
        mainCam.transform.LookAt(player.transform.position + Vector3.up * 0.9f);
        camObj.AddComponent<CameraSmoothFollow>().target = player.transform;
        camObj.tag = "MainCamera";
    }

    void SpawnNPC()
    {
        Vector3 pos = RandomNavPosition();
        GameObject npcGO = GameObject.CreatePrimitive(PrimitiveType.Capsule);
        npcGO.transform.position = pos + Vector3.up * 0.9f;
        npcGO.name = "NPC";
        var mat = new Material(Shader.Find("Standard"));
        mat.color = Color.red;
        npcGO.GetComponent<Renderer>().material = mat;
        NPC npc = new NPC()
        {
            go = npcGO,
            state = NPCState.Patrol,
            patrolTarget = RandomNavPosition(),
            health = 50,
            speed = npcPatrolSpeed
        };
        npcs.Add(npc);
    }

    Vector3 RandomNavPosition()
    {
        float half = worldSize * 0.45f;
        return new Vector3(Random.Range(-half, half), 0f, Random.Range(-half, half));
    }
    #endregion

    #region UI
    void CreateUI()
    {
        // Canvas
        GameObject canvasObj = new GameObject("Canvas");
        var canvas = canvasObj.AddComponent<Canvas>();
        canvas.renderMode = RenderMode.ScreenSpaceOverlay;
        CanvasScaler cs = canvasObj.AddComponent<CanvasScaler>();
        cs.uiScaleMode = CanvasScaler.ScaleMode.ScaleWithScreenSize;
        canvasObj.AddComponent<GraphicRaycaster>();

        // Health text
        GameObject h = new GameObject("HealthText");
        h.transform.SetParent(canvasObj.transform);
        healthText = h.AddComponent<Text>();
        healthText.font = Resources.GetBuiltinResource<Font>("Arial.ttf");
        healthText.fontSize = 28;
        healthText.alignment = TextAnchor.UpperLeft;
        RectTransform rt = healthText.GetComponent<RectTransform>();
        rt.anchorMin = new Vector2(0f, 1f);
        rt.anchorMax = new Vector2(0f, 1f);
        rt.pivot = new Vector2(0f, 1f);
        rt.anchoredPosition = new Vector2(10, -10);
        rt.sizeDelta = new Vector2(300, 40);

        // Debug text (bottom)
        GameObject d = new GameObject("DebugText");
        d.transform.SetParent(canvasObj.transform);
        debugText = d.AddComponent<Text>();
        debugText.font = Resources.GetBuiltinResource<Font>("Arial.ttf");
        debugText.fontSize = 18;
        debugText.alignment = TextAnchor.LowerLeft;
        RectTransform drt = debugText.GetComponent<RectTransform>();
        drt.anchorMin = new Vector2(0f, 0f);
        drt.anchorMax = new Vector2(0f, 0f);
        drt.pivot = new Vector2(0f, 0f);
        drt.anchoredPosition = new Vector2(10, 10);
        drt.sizeDelta = new Vector2(500, 80);
    }

    void UpdateUI()
    {
        healthText.text = $"Health: {playerHealth}/{playerMaxHealth}";
        debugText.text = $"NPCs: {npcs.Count}  | Pos: {player.transform.position.x:F1}, {player.transform.position.z:F1}";
    }
    #endregion

    #region Player Movement & Touch
    void HandleTouchMovement()
    {
        Vector2 move = Vector2.zero;
        float run = 1f;

        // Touch controls: left-half virtual joystick; right-half tap for attack
        if (Input.touchCount > 0)
        {
            foreach (Touch t in Input.touches)
            {
                if (t.phase == TouchPhase.Began)
                {
                    if (t.position.x < Screen.width * 0.5f)
                    {
                        joystickActive = true;
                        joystickStart = t.position;
                        joystickDelta = Vector2.zero;
                    }
                    else
                    {
                        // right half: attack / interact
                        TryAttack();
                    }
                }
                else if (t.phase == TouchPhase.Moved || t.phase == TouchPhase.Stationary)
                {
                    if (joystickActive && t.position.x < Screen.width * 0.5f)
                    {
                        joystickDelta = t.position - joystickStart;
                        float max = Screen.dpi > 0 ? Screen.dpi * 0.5f : 150f;
                        joystickDelta = Vector2.ClampMagnitude(joystickDelta, max);
                        move = joystickDelta.normalized * (joystickDelta.magnitude / max);
                        // two-finger or long press for run
                        if (Input.touchCount >= 2) run = playerRunMultiplier;
                    }
                }
                else if (t.phase == TouchPhase.Ended || t.phase == TouchPhase.Canceled)
                {
                    if (t.position.x < Screen.width * 0.5f)
                    {
                        joystickActive = false;
                        joystickDelta = Vector2.zero;
                    }
                }
            }
        }
        else
        {
            // allow keyboard for editor testing
            float h = Input.GetAxis("Horizontal");
            float v = Input.GetAxis("Vertical");
            move = new Vector2(h, v);
            if (Input.GetKey(KeyCode.LeftShift)) run = playerRunMultiplier;
            if (Input.GetMouseButtonDown(0))
            {
                TryAttack();
            }
        }

        Vector3 camForward = mainCam.transform.forward;
        camForward.y = 0;
        camForward.Normalize();
        Vector3 camRight = mainCam.transform.right;
        camRight.y = 0;
        camRight.Normalize();

        Vector3 moveWorld = camRight * move.x + camForward * move.y;
        if (moveWorld.magnitude > 0.01f)
        {
            // face movement direction
            player.transform.forward = Vector3.Slerp(player.transform.forward, moveWorld.normalized, Time.deltaTime * 10f);
        }

        Vector3 velocity = moveWorld * playerSpeed * run;
        // simple gravity
        velocity.y = -9.81f * 0.02f;
        playerController.Move(velocity * Time.deltaTime);
    }

    void TryAttack()
    {
        // find closest NPC in range and damage
        NPC target = null;
        float best = attackRange + 0.1f;
        foreach (var n in npcs)
        {
            if (n == null || n.go == null) continue;
            float d = Vector3.Distance(player.transform.position, n.go.transform.position);
            if (d <= attackRange && d < best)
            {
                best = d;
                target = n;
            }
        }
        if (target != null)
        {
            target.health -= 25;
            if (target.health <= 0)
            {
                Destroy(target.go);
                npcs.Remove(target);
            }
        }
    }
    #endregion

    #region NPC Logic
    void UpdateNPCs()
    {
        for (int i = npcs.Count - 1; i >= 0; i--)
        {
            NPC n = npcs[i];
            if (n == null || n.go == null) { npcs.RemoveAt(i); continue; }
            float distToPlayer = Vector3.Distance(n.go.transform.position, player.transform.position);

            // state switching
            if (distToPlayer <= npcSightRange)
            {
                n.state = NPCState.Chase;
                n.speed = npcChaseSpeed;
            }
            else
            {
                if (n.state == NPCState.Chase) // lost player
                {
                    n.state = NPCState.Patrol;
                    n.patrolTarget = RandomNavPosition();
                    n.speed = npcPatrolSpeed;
                }
            }

            if (n.state == NPCState.Patrol)
            {
                Vector3 dir = (n.patrolTarget - n.go.transform.position);
                dir.y = 0;
                if (dir.magnitude < 1f) n.patrolTarget = RandomNavPosition();
                else n.go.transform.position += dir.normalized * n.speed * Time.deltaTime;
                // slow rotation
                if (dir.sqrMagnitude > 0.01f)
                    n.go.transform.forward = Vector3.Slerp(n.go.transform.forward, dir.normalized, Time.deltaTime * 3f);
            }
            else if (n.state == NPCState.Chase)
            {
                Vector3 dir = (player.transform.position - n.go.transform.position);
                dir.y = 0;
                if (dir.magnitude > 0.2f)
                {
                    n.go.transform.position += dir.normalized * n.speed * Time.deltaTime;
                    n.go.transform.forward = Vector3.Slerp(n.go.transform.forward, dir.normalized, Time.deltaTime * 6f);
                }

                // if very close, attack player
                if (dir.magnitude <= 1.2f)
                {
                    // damage player (throttle so not instant-death)
                    if (n.lastAttackTime + 1.0f < Time.time)
                    {
                        playerHealth -= 8;
                        n.lastAttackTime = Time.time;
                        if (playerHealth <= 0)
                        {
                            OnPlayerDead();
                        }
                    }
                }
            }
        }
    }

    void OnPlayerDead()
    {
        // simple respawn
        playerHealth = playerMaxHealth;
        player.transform.position = Vector3.zero + Vector3.up * 1.0f;
        // optional: remove all NPCs or respawn them
        debugText.text = "You Died — Respawning...";
    }
    #endregion

    // simple NPC container
    class NPC
    {
        public GameObject go;
        public NPCState state;
        public Vector3 patrolTarget;
        public float speed = 1.5f;
        public int health = 50;
        public float lastAttackTime = -10f;
    }

    enum NPCState { Patrol, Chase }
}

// Camera follow helper (kept inside same file for single-file requirement)
public class CameraSmoothFollow : MonoBehaviour
{
    public Transform target;
    public Vector3 offset = new Vector3(0f, 6f, -8f);
    public float smooth = 6f;

    void LateUpdate()
    {
        if (target == null) return;
        Vector3 desired = target.position + offset;
        transform.position = Vector3.Lerp(transform.position, desired, Time.deltaTime * smooth);
        transform.LookAt(target.position + Vector3.up * 1.0f);
    }
}
