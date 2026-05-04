### // TECHNICAL DOCUMENTATION — GAME SECURITY

# UE5 GAME

# SECURITY

## Attack Vectors · Detection Methods · Anti-Cheat Architecture

A complete technical reference for UE5 developers on how cheating tools operate at the memory, driver,
and network layer — and how to detect, prevent, and respond to every category of attack.

```
ENGINE — Unreal Engine 5.3 / 5.4 (latest)
```
```
PLATFORM — Windows 10/11 x64 (primary)
```
```
SECTIONS — 28 sections — attack taxonomy to full AC architecture
```
```
COVERS — Memory · Injection · Aimbot · ESP · DMA · Kernel · Network
```
```
APPROACH — Defender-first: understand the attack, build the shield
```
```
DATE — April 2026
```
### CONFIDENTIAL — FOR AUTHORIZED GAME DEVELOPERS ONLY


```
UE5 Game Security Reference
Attack Vectors, Detection Methods and Anti-Cheat Architecture
```
```
Copyright © 2026 — For authorized game developers only.
```
This document is a technical security reference produced for the purpose of understanding attack
vectors in order to build effective defenses. All techniques described are analysed strictly from the
defender's perspective. Distribution to unauthorized parties is prohibited.

```
Unreal Engine is a trademark of Epic Games, Inc. All other trademarks are the property of their
respective owners.
```
```
First Edition — April 2026
```

## TABLE OF CONTENTS

- 01 The Threat Landscape — 2025/2026 Overview
- 02 Cheat Categories — Full Taxonomy
- 03 How Memory Reading Attacks Work
- 04 UE5 Memory Layout — What Attackers See
- 05 DLL Injection — How It Works and How to Detect It
- 06 External vs Internal Cheats — Architecture Differences
- 07 Aimbot Mechanics — Bone Targeting, FOV, Smoothing
- 08 ESP and Wallhack — World-to-Screen and Visibility
- 09 Radar Cheats — Minimap Data Extraction
- 10 Trigger Bots — Input Automation and Detection
- 11 Speed and Movement Hacks — Physics Manipulation
- 12 Kernel-Mode Cheats — Driver-Level Attacks
- 13 DMA Cheats — Hardware-Level Memory Access
- 14 Hypervisor-Based Cheats — VM Evasion
- 15 Packet Manipulation and Network Cheats
- 16 Anti-Debug and Anti-Detection Techniques
- 17 GAN Aimbots — AI-Powered Undetectable Input
- 18 UE5 Defense Architecture — Server Authority
- 19 Memory Encryption and Secure Variables (SCUE5)
- 20 VTable Sentinel and Integrity Verification
- 21 Process and Module Scanning
- 22 Behavioral Detection — Telemetry and Heuristics
- 23 Input Analysis — Detecting Aimbot via Statistics
- 24 Kernel-Level Anti-Cheat — EAC and BattlEye Architecture
- 25 Network-Side Validation — Server Authoritative UE5
- 26 Detection Response Strategy — Ban Tiers
- 27 Building Your Own Anti-Cheat in UE5 — Full Blueprint
- 28 Resource Reference and Further Reading


## 01 THE THREAT LANDSCAPE — 2025/

As of 2025, anti-cheat systems protect over 338 games. The sophistication of cheating has escalated
from simple memory editing in the 2000s to kernel-mode drivers, hardware DMA cards, hypervisor attacks,
and AI-powered aimbots that mimic human mouse movement. For UE5 developers, understanding this threat
landscape is the mandatory first step before writing a single line of protection code.

```
Attack Category Privilege Level Detection
Difficulty
```
```
2026 Prevalence
```
```
Memory Reading
(external)
```
```
User-mode Medium Very High
```
```
DLL Injection
(internal)
```
```
User-mode Medium High
```
```
Kernel Driver Cheat Ring 0 Hard High
DMA Hardware Cheat Hardware Very Hard Growing
Hypervisor / VM Cheat Ring -1 Extreme Niche
Network Packet Spoof User-mode Medium Medium
GAN / AI Aimbot User-mode Very Hard Growing
Input Device Spoofing HID layer Hard Medium
```
```
KEY INSIGHT FOR UE5 DEVELOPERS
No single protection layer stops all attacks. Defense-in-depth is mandatory.
Client-side protection is always bypassable given enough time. Server authority is
the true floor.
The economics matter: make cheating expensive enough that most cheaters give up.
Delayed bans (1-2 weeks after detection) prevent cheat developers identifying your
detection method.
Black Hat USA 2025: game anti-cheat is now the fastest-moving front-line in
civilian cybersecurity.
```

## 02 Cheat Categories — Full Taxonomy

Every cheat feature maps to one or more attack vectors. Understanding the category tells you exactly
which detection surface to target. The images you provided show a typical commercial cheat menu with
Aim, Trigger, Weapon Config, Visuals/ESP, Radar, World, Colors, and Misc modules — each maps to a
specific attack type.

```
Cheat Feature Attack Vector What It Reads/Does Your Detection Surface
```
```
Aimbot / Memory
Aim
```
```
Memory Read +
Input Write
```
```
Reads enemy bone positions
(GWorld -> Actors -> Mesh ->
Bones)
```
```
Bone access pattern,
mouse input stats
```
```
FOV Circle Overlay Render Draws circle via
DirectX/overlay on top of game
render
```
```
Foreign render hooks,
overlay windows
```
```
Visibility Check Memory Read +
LineTrace
```
```
Reads CanSee flags or calls
LineOfSight via memory
```
```
Abnormal LineOfSight
call rate from client
Triggerbot Memory Read +
Input Write
```
```
Reads CrosshairTarget actor,
fires when non-null
```
```
Sub-human reaction
time distribution
ESP / Box /
Skeleton
```
```
Memory Read +
Overlay
```
```
Reads all PlayerState actors,
projects 3D->2D, draws boxes
```
```
Actor array scan
pattern, overlay
injection
Radar / Minimap Memory Read +
Overlay
```
```
Reads all actor positions,
renders separate radar window
```
```
Unauthorized window
with game data
Wallhack Memory / File
Mod
```
```
Modifies material opacity or
reads positions through
geometry
```
```
Material CRC check,
impossible sight lines
```
```
Speed Hack Memory Write Writes CharacterMovement->MaxW
alkSpeed
```
```
Server velocity
validation
No Recoil Input Write Offsets mouse input to cancel
recoil pattern
```
```
Recoil pattern
deviation stats
Fly Hack Memory Write Writes player Z position
faster than game can correct
```
```
Server position
authority, teleport
detection
Config Save/Load File I/O Stores cheat settings to disk
for persistence
```
```
Suspicious file access
in game dir
Stream Proof API Hook Hooks capture APIs (BitBlt,
DWM) to hide overlay
```
```
API hook detection on
capture interfaces
```

## 03 How Memory Reading Attacks Work

Memory reading is the foundation of almost every UE5 cheat. The attacker's process reads your game's
process memory using Windows APIs to extract game state without modifying the game. Because it is
read-only, it leaves almost no modification footprint. Understanding the exact read chain is critical
for designing your defenses.

The Attack Read Chain

Cheats navigate UE5's object hierarchy in memory using known, publicly-documented offsets. The chain is
consistent across UE5 builds because UE5's engine structure is open source — the offsets just change
with each engine version, which is why cheat makers release 'offset updates'.

```
// C++ — External Read Chain (Attacker Perspective — What You Must Detect)
// How an external cheat reads your player positions
// Step 1: Find base address of your game EXE in target process
HANDLE hProcess = OpenProcess(PROCESS_VM_READ, FALSE, game_pid);
uintptr_t base = GetModuleBaseAddress(game_pid, L"YourGame.exe");
// Step 2: Read GWorld pointer (well-known UE5 global)
uintptr_t gworld = ReadMemory<uintptr_t>(hProcess, base + GWORLD_OFFSET);
// Step 3: Walk UE5 object hierarchy
uintptr_t game_instance = ReadMemory<uintptr_t>(hProcess, gworld + 0x1C0);
uintptr_t local_players = ReadMemory<uintptr_t>(hProcess, game_instance + 0x38);
uintptr_t persistent_lvl = ReadMemory<uintptr_t>(hProcess, gworld + 0xC0);
uintptr_t actor_array = ReadMemory<uintptr_t>(hProcess, persistent_lvl + 0xA0);
int actor_count = ReadMemory<int>(hProcess, persistent_lvl + 0xA8);
// Step 4: Iterate all actors, find APawn/APlayerState subclasses
for (int i = 0; i < actor_count; i++) {
uintptr_t actor = ReadMemory<uintptr_t>(hProcess, actor_array + i * 0x8);
// Read mesh component -> bone array -> head bone transform
uintptr_t mesh = ReadMemory<uintptr_t>(hProcess, actor + MESH_OFFSET);
FTransform head = ReadMemory<FTransform>(hProcess, mesh + BONE_ARRAY + HEAD_IDX*0x60);
}
```
Windows APIs Used by External Cheats

```
API Purpose Detection Approach
OpenProcess(PROCESS_VM_READ) Attach to game process Monitor for cross-process handle
opens targeting your PID
ReadProcessMemory() Read game memory remotely Kernel callback:
PsSetCreateProcessNotifyRoutine
VirtualQueryEx() Enumerate game memory
regions
```
```
Unusual memory query patterns from
foreign process
CreateToolhelp32Snapshot() List processes and modules Detect snapshot of your process
from unsigned process
NtReadVirtualMemory() NT-level memory read
(bypass)
```
```
SSDT hook or ETW monitoring of NT
calls
```
Your Detection Surface

```
Monitor for external processes opening a PROCESS_VM_READ handle to your game PID.
Any process performing ReadProcessMemory on your game should be flagged immediately.
Scan for processes enumerating your module list — legitimate tools rarely do this at runtime.
Watch for unusual process relationships: your game should not have unexpected child or sibling
processes reading it.
```

## 04 UE5 Memory Layout — What Attackers See

Because UE5 is open source, its entire memory layout is documented. Cheats use known offsets to
navigate the object graph. These are the exact structures cheats target in your game.

Critical UE5 Structures Targeted by Cheats

```
UE5 Structure Offset Path Data Extracted Cheat Feature Enabled
UWorld GWorld (global
ptr)
```
```
Game state, actor
registry, level
```
```
All cheats — root entry
point
ULevel::Actors UWorld + 0xC0 +
0xA
```
```
Full actor array with
positions
```
```
ESP, Radar, Wallhack
```
```
APlayerState Actor array
iteration
```
```
Player name, team,
score
```
```
ESP name display, team
filtering
APawn::Location Actor +
RootComponent +
Rel.
```
```
World-space player
position
```
```
ESP, Aimbot target
position
```
```
USkeletalMeshComp APawn + 0x
(approx)
```
```
Bone transform array
(head, chest...)
```
```
Bone Aimbot, Skeleton
ESP
APlayerController GWorld +
LocalPlayers
```
```
Camera rotation, FOV,
ViewTarget
```
```
World-to-screen
projection
UCharacterMovement APawn +
MovementComp
```
```
MaxWalkSpeed,
Velocity
```
```
Speed hack, movement
cheat
AWeapon/CurrentWea
pon
```
```
Pawn +
CurrentWeapon ptr
```
```
Ammo, weapon state,
fire rate
```
```
No recoil, rapid fire
```
```
AGameState UWorld + GameState Match timer, round
info
```
```
Match state manipulation
```
```
How World-to-Screen Projection Works (ESP Foundation)
// C++ — What ESP Does With Your Memory (Defender Reference)
// Every ESP cheat does this to project 3D bone position to 2D screen
// The cheat reads your PlayerController's ProjectionMatrix from memory
FVector2D WorldToScreen(FVector world_pos, FCameraInfo cam) {
// Read from memory: ViewMatrix (ProjectionMatrix * ViewMatrix)
FMatrix vp = cam.view_projection_matrix; // read from ULocalPlayer
// Apply projection
FVector4 clip = vp.TransformFVector4(FVector4(world_pos, 1.0f));
if (clip.W < 0.001f) return {-1, -1}; // behind camera
float inv_w = 1.0f / clip.W;
float ndc_x = clip.X * inv_w;
float ndc_y = clip.Y * inv_w;
// NDC -> Screen
float sx = (screen_w / 2.0f) + (ndc_x * screen_w / 2.0f);
float sy = (screen_h / 2.0f) - (ndc_y * screen_h / 2.0f);
return {sx, sy};
}
// DETECTION: You can detect this by monitoring read access to
// ULocalPlayer::ViewProjectionMatrix — legitimate game code rarely
// reads this from outside the render thread.
```
```
DEFENSE: Encrypt or rotate these key pointers at runtime
UE5 SCUE5 plugin: replace raw UProperty values with FSafeFloat, FSafeVector —
encrypts values in memory.
Randomise struct layout per-session using compiler tricks or XOR padding.
Add canary values adjacent to bone array — reads that hit canaries signal
scanning.
Rotate GWorld pointer indirection: add a random offset layer only your code knows.
```

## 05 DLL Injection — How It Works and How to Detect It

Internal cheats inject a DLL into your game process. Once inside, they have direct memory access — no
ReadProcessMemory needed. They can hook your render pipeline, read and write memory directly, and call
engine functions. This is the most powerful attack vector and the one used by the commercial 'menu'
cheats shown in your screenshots.

Injection Methods — From Simple to Advanced

```
Method How It Works Detection Signature
LoadLibrary
Injection
```
```
CreateRemoteThread calls LoadLibraryA
in target process
```
```
Unsigned DLL in module list,
remote thread creation
Manual Mapping Manually maps DLL sections without
calling LoadLibrary — no module list
entry
```
```
Memory regions with PE headers
but no module entry
```
```
Reflective DLL
Injection
```
```
DLL contains its own loader,
bootstraps itself from shellcode
```
```
Executable anonymous memory
regions (MEM_PRIVATE+EXECUTE)
Process Hollowing Spawns legitimate process, replaces
its memory with cheat code
```
```
Image base mismatch between disk
and memory
Thread Hijacking Suspends existing thread, redirects
RIP to shellcode, resumes
```
```
Thread context modification from
external process
APC Injection Queues APC to alertable thread to
execute shellcode
```
```
NtQueueApcThread from
unauthorized process
Kernel Driver
Injection
```
```
Kernel driver maps DLL directly into
process address space
```
```
Requires kernel detection — see
Section 12
```
```
Detecting Injected DLLs in UE5 — C++ Implementation
// C++ — UE5 Anti-Injection Detection
// Run this check on a background thread every 30-60 seconds
void ScanForInjectedModules() {
// Build whitelist of expected modules at startup
static TSet<FString> KnownModules = BuildWhitelistAtStartup();
HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, GetCurrentProcessId());
MODULEENTRY32W me = { sizeof(me) };
if (Module32FirstW(snap, &me)) {
do {
FString modName = FString(me.szModule).ToLower();
if (!KnownModules.Contains(modName)) {
// Unknown module detected
ReportSecurityEvent(ESecEvent::UnknownModule, modName);
}
// Also check: is DLL signed by a trusted publisher?
if (!VerifyModuleSignature(me.modBaseAddr)) {
ReportSecurityEvent(ESecEvent::UnsignedModule, modName);
}
} while (Module32NextW(snap, &me));
}
CloseHandle(snap);
}
// Also scan for anonymous executable memory (manually mapped DLLs)
void ScanExecutableMemory() {
MEMORY_BASIC_INFORMATION mbi;
uintptr_t addr = 0;
while (VirtualQuery((LPCVOID)addr, &mbi, sizeof(mbi))) {
bool is_exec = mbi.Protect & (PAGE_EXECUTE|PAGE_EXECUTE_READ|PAGE_EXECUTE_READWRITE);
bool is_private = mbi.Type == MEM_PRIVATE;
bool has_pe = CheckForPEHeader((uintptr_t)mbi.BaseAddress);
if (is_exec && is_private && has_pe) {
// PE header in anonymous private executable memory = manually mapped
ReportSecurityEvent(ESecEvent::ManuallyMappedDll, addr);
}
```

addr += mbi.RegionSize;
}
}


## 06 EXTERNAL vs INTERNAL CHEATS — ARCHITECTURE

```
Aspect External Cheat Internal (Injected) Cheat
```
```
Process Separate process, reads game via API Lives inside game process, direct
access
Memory Access ReadProcessMemory — detectable API
calls
```
```
Direct pointer dereference — no API
overhead
Detection
Risk
```
```
Easier: process enumeration catches it Harder: appears as legitimate game code
```
```
Rendering Overlay window (DirectX/OpenGL over
game)
```
```
Hooks game's own render pipeline
(PostRender)
Input SendInput / mouse_event to OS Can hook input at engine level
(UPlayerInput)
Persistence Runs alongside game, dies when game
dies
```
```
Dies when game dies, re-inject each
session
Signature Foreign process with game handle Unknown module in module list,
anomalous memory
Speed Slightly slower due to API overhead Fastest — direct memory access
Example External radar, external ESP Internal menu cheat (your screenshots)
```

## 07 Aimbot Mechanics — Bone Targeting, FOV, Smoothing

Aimbots read your players' skeletal mesh bone transforms and calculate the angle delta needed to
redirect the cheater's aim to a target bone (usually the head). Understanding this mechanism precisely
tells you what statistical signatures to detect.

```
The Aimbot Algorithm — Step by Step
// C++ — Aimbot Core Algorithm (What You Are Detecting)
// This is the core of every memory aimbot — what you need to detect
void AimbotTick() {
APawn* local_pawn = ReadLocalPawn();
FVector local_pos = ReadActorLocation(local_pawn);
FRotator current_aim = ReadControlRotation(local_pawn);
float fov_filter = settings.fov_radius; // only aim within FOV circle
APawn* best_target = nullptr;
float best_dist = FLT_MAX;
for (APawn* enemy : ReadActorArray()) {
if (!IsEnemy(enemy)) continue;
if (!PassesVisibilityCheck(enemy)) continue; // LineOfSight call
FVector head_pos = ReadBonePosition(enemy, HEAD_BONE_INDEX);
FRotator angle_to = CalcAngle(local_pos, head_pos);
// FOV filter: only target if within circular FOV limit
float delta = FMath::Abs(angle_to.Yaw - current_aim.Yaw);
if (delta > fov_filter) continue;
float dist = FVector::Dist(local_pos, head_pos);
if (dist < best_dist) {
best_dist = dist;
best_target = enemy;
}
}
if (best_target) {
FVector target_pos = ReadBonePosition(best_target, settings.target_bone);
FRotator target_rot = CalcAngle(local_pos, target_pos);
// Smoothing: interpolate towards target to look human
float smooth = settings.smoothing; // 1.0 = instant, 10.0 = slow
FRotator new_aim = FMath::RInterpTo(current_aim, target_rot, dt, smooth);
// Aimkey check: only apply if mouse button held
if (GetAsyncKeyState(settings.aimkey)) {
SetMouseAngle(new_aim); // SendInput to Windows
}
}
}
```
Statistical Signatures Your Server Can Detect

```
Metric Normal Player Aimbot Signature Detection Method
```
```
Target switch
time
```
```
200-600ms
average
```
```
< 50ms — instant
switch
```
```
Track time between target
changes server-side
Aim snap angle Gradual delta Large
instantaneous
rotation delta
```
```
Monitor rotation delta per
frame
```
```
Headshot rate 5-20% typical > 60% sustained =
impossible
```
```
Per-session headshot
percentage tracking
Aim deviation Human jitter
~2-5deg
```
```
Near-zero
deviation on
target
```
```
Statistical variance of aim
on enemy actors
```
```
Reaction time > 150ms human
min
```
```
< 50ms — sub-human Time from enemy visible to
first shot fired
Smooth curve
shape
```
```
Random
Brownian
```
```
Mathematically
smooth arc
```
```
Fourier analysis of mouse
movement path
```

```
Metric Normal Player Aimbot Signature Detection Method
```
FOV tracking Variable Perfect angular
lock within FOV

```
Correlation between enemy pos
and aim angle
```

## 08 ESP and Wallhack — World-to-Screen and Visibility

ESP (Extra Sensory Perception) cheats read all player positions from memory and draw boxes, skeletons,
health bars and name tags on screen using a DirectX overlay. They know where every player is at all
times. Wallhacks either use ESP data or modify material transparency.

How ESP Renders

```
Read all APawn actors from persistent level actor array.
For each enemy pawn, read root position and skeletal mesh bone transforms.
Project 3D bone positions to 2D screen using ViewProjectionMatrix (read from ULocalPlayer).
Render boxes, skeletons, health bars via DirectX 11/12 overlay window (positioned over game).
Stream Proof mode: hooks BitBlt/DWM capture APIs so overlay is invisible to OBS/recording.
```
Detecting ESP Overlays

```
Scan for foreign windows with WS_EX_LAYERED + WS_EX_TRANSPARENT style positioned over your game
window.
Detect DirectX device creation from a non-game process that has your game window as a sibling.
Monitor BitBlt and DWM hook tampering — Stream Proof cheats reveal themselves by hooking capture
APIs.
Check for unusual GetWindowRect calls on your game window from external processes.
Detect NVIDIA Freestyle or ReShade hooks — used by some ESP overlays as a vehicle.
```
```
Wallhack via Material Modification
// C++ UE5 — Material Integrity Check
// Some wallhacks modify material opacity or blend mode
// Detection: Hash check all loaded material assets at runtime
void VerifyMaterialIntegrity() {
for (UMaterial* mat : GetAllLoadedMaterials()) {
FString mat_path = mat->GetPathName();
uint32 expected = MaterialHashDB.FindChecked(mat_path);
uint32 actual = HashMaterialParameters(mat);
if (expected != actual) {
ReportSecurityEvent(ESecEvent::MaterialTampered, mat_path);
}
}
}
```
Server-Side ESP Countermeasure — Occlusion Culling

The most effective ESP countermeasure is server-side: don't send enemy data to clients who can't see
it. World of Tanks pioneered this and it fundamentally eliminates ESP by removing the data from the
client.

```
// C++ UE5 — Server-Side Occlusion Culling (Kills ESP at Source)
// UE5: Override NetCullDistanceSquared and custom relevancy
bool AYourGameMode::IsActorRelevantToConnection(
const AActor* Actor, const TArray<FNetViewer>& Viewers) {
for (const FNetViewer& viewer : Viewers) {
APlayerController* PC = viewer.InViewer;
if (!PC) continue;
// Only replicate enemy if visible to this client
bool line_of_sight = PC->LineOfSightTo(Actor);
bool in_range = FVector::Dist(
PC->GetPawn()->GetActorLocation(),
Actor->GetActorLocation()) < MAX_REPLICATE_DIST;
if (line_of_sight && in_range) return true;
}
```

return false; // Don't send this actor's data to this client
}


## 09 RADAR CHEATS, TRIGGERBOTS, MOVEMENT HACKS

09 — Radar Cheats

Radar cheats read all actor positions and render them in a separate minimap window — often a separate
second monitor. They provide full battlefield awareness without touching the game window.

```
What they read All APawn positions, team data, player names, item locations
How rendered Separate WinForms/Qt window, often on second monitor
DMA variant PCILeech DMA card reads memory from hardware — game process untouched
Detection Monitor for cross-process reads of your actor array. DMA needs kernel
driver detection.
Server defense Same as ESP: server-side relevancy culling removes position data from
client
```
10 — Triggerbot Detection

Triggerbots fire automatically when the crosshair overlaps an enemy hitbox. They read the
CrosshairTraceTarget from memory and send a click when non-null. The key detection is statistical:
human reaction times are never sub-50ms.

```
// C++ UE5 — Triggerbot Statistical Detection
// Server-side: track time from 'enemy enters crosshair' to 'shot fired'
// Requires server to know approximate client aim direction (sent with input)
struct FReactionStats {
TArray<float> ReactionTimes; // milliseconds
float GetMedian() { /* sort and median */ }
float GetMin() { return *Algo::MinElement(ReactionTimes); }
};
// Flag: min reaction time < 30ms sustained across 10+ samples = triggerbot
if (stats.GetMin() < 30.0f && stats.ReactionTimes.Num() > 10) {
FlagPlayerForReview(player, ECheatFlag::Triggerbot);
}
```
11 — Speed and Movement Hacks

Movement hacks write directly to CharacterMovementComponent values or teleport the player by writing
position faster than server correction. Server-authoritative movement in UE5 is your primary defense.

```
// C++ UE5 — Server Movement Validation
// UE5 Server-side movement validation
void AYourCharacter::ServerMove_Implementation(
float TimeStamp, FVector_NetQuantize10 InAccel,
FVector_NetQuantize100 ClientLoc, ...) {
// Validate client claimed location against server simulation
FVector server_loc = GetActorLocation();
float max_error = CalculateMaxAllowedError(TimeStamp);
if (FVector::Dist(ClientLoc, server_loc) > max_error) {
// Teleport detected — correct client, log event
ClientAdjustPosition(TimeStamp, server_loc, ...);
ReportSecurityEvent(ESecEvent::PositionCheat, this);
return;
}
// Validate speed
float max_speed = GetCharacterMovement()->MaxWalkSpeed;
float actual_spd = (ClientLoc - LastValidatedLoc).Size() / DeltaTime;
if (actual_spd > max_speed * 1.1f) { // 10% tolerance
ReportSecurityEvent(ESecEvent::SpeedCheat, this);
}
}
```

## 12 Kernel-Mode Cheats — Driver-Level Attacks

Kernel-mode cheats load a Windows driver (Ring 0) giving them the same privilege level as the OS
itself. From here they can read any process memory with zero Windows API calls, hide processes from
enumeration, disable other security drivers, and intercept hardware input. This is the attack tier that
forces EAC and BattlEye to also run at Ring 0.

How Kernel Cheats Load Their Driver

```
Method How it Works Detection
Vulnerable signed
driver (BYOVD)
```
```
Load a legitimate but vulnerable
driver, exploit it to load unsigned code
```
```
Monitor loaded driver list for
known-vulnerable drivers
Test signing mode Boot with testsigning=on, load
self-signed driver
```
```
Detect testsigning boot flag
in your anti-cheat
EV code signing Purchase EV certificate, sign malicious
driver
```
```
Certificate chain validation,
revocation checking
DSE patch Disable Driver Signature Enforcement in
kernel memory
```
```
Monitor PatchGuard, DSE flag
integrity
Secure Boot bypass Use bootkit to disable Secure Boot
protections
```
```
Require Secure Boot enabled to
launch game
```
What Kernel Cheats Do Differently

```
Read/write game memory via MmCopyMemory — bypasses all user-mode monitoring.
Hide their own process and driver from enumeration via DKOM (Direct Kernel Object Manipulation).
Hook SSDT (System Service Descriptor Table) to intercept and spoof API calls.
Disable or crash competing anti-cheat drivers by targeting their IRP handlers.
Intercept raw hardware input before it reaches the OS input stack.
```
Detection at the Kernel Level

```
Load your own kernel driver with PsSetLoadImageNotifyRoutine to detect all driver loads.
Maintain a whitelist of expected drivers; flag any unknown drivers immediately.
Check for BYOVD: monitor for known-vulnerable driver hashes (loldrivers.io database).
Verify DSE is enabled: read the g_CiEnabled flag and validate it has not been patched.
Use Kernel Patch Protection (PatchGuard) awareness: monitor for PG bypasses.
EAC/BattlEye run at Ring 0 for exactly this reason — user-mode cannot detect kernel cheats.
```

## 13 DMA Cheats — Hardware-Level Memory Access

DMA (Direct Memory Access) cheats use a hardware PCILeech-compatible FPGA card installed in a PCIe slot
to read the gaming PC's RAM directly over the hardware bus — completely bypassing the OS, drivers, and
all software detection. The cheat software runs on a separate PC connected via DMA card.

```
Hardware required PCILeech FPGA card (~$150-$500) in PCIe slot of gaming PC
Attack PC Second PC runs the cheat software, reads memory via USB/PCIe from FPGA
OS visibility Zero — no process, no driver, no API call on gaming PC
What it reads Full physical RAM — can find any value in any process
Render method Cheat renders on the second PC's screen (second monitor or phone)
Detection approach IOMMU/VT-d configuration, PCIe device whitelist, timing attacks
```
How to Detect DMA Attacks

```
Enable IOMMU (Intel VT-d / AMD-Vi) and verify it is active at game launch — this restricts PCIe
DMA to authorized regions.
Enumerate PCIe devices at startup and warn/block on unexpected FPGA or DMA-capable devices.
Timing attack: MmCopyMemory read timing differs measurably from normal reads — statistical timing
analysis can detect DMA.
Memory bus fingerprinting: legitimate memory access patterns differ from bulk sequential DMA
reads.
Some anti-cheats monitor PCIe BAR (Base Address Register) mappings for suspicious entries.
Require Secure Boot + IOMMU as launch preconditions for competitive modes.
```
```
DMA cheats are currently the hardest category to detect. Server-side behavioral analysis (aim
statistics, position validation) is your most reliable fallback when client-side detection fails.
```

## 14-16 HYPERVISOR, NETWORK, AND ANTI-DEBUG CHEATS

14 — Hypervisor-Based Cheats (Ring -1)

The most sophisticated attacks run a Type-2 hypervisor beneath Windows, making the OS a guest VM. From
Ring -1 the cheat has full control and can intercept any memory read — even from anti-cheat kernel
drivers.

```
Attack QEMU with memory patching, custom VMX hypervisor
Detection CPUID hypervisor bit check, RDTSC timing anomalies (VM overhead), VMX
leaf check
Defense Require Hyper-V Credential Guard as prerequisite, detect non-Microsoft
hypervisors
Prevalence Niche — requires significant technical expertise and may cause
instability
```
15 — Network Packet Manipulation

Network cheats intercept, modify, or replay UE5 network packets. In UE5, all gameplay state is
transmitted via the UNetConnection pipeline.

```
Attack What It Does UE5 Defense
```
```
Packet delay
manipulation
```
```
Delays own packets to abuse lag
compensation
```
```
Server-side lag compensation caps
(UE5 default 300ms)
Position packet spoof Sends false client position to
server
```
```
Server authority + movement
validation (Section 11)
Packet replay Replays old input packets to
duplicate actions
```
```
Per-packet sequence numbers +
timestamp validation
Traffic analysis ESP Reads unreplicated actor data
from network sniff
```
```
Encrypt all UE5 network traffic
(PacketEncryption plugin)
```
16 — Anti-Debug and Anti-Detection Techniques

Commercial cheats actively try to detect and evade your anti-cheat. Understanding these techniques
helps you make your detection code harder to find.

```
Evasion Technique What It Does Counter-Detection
```
```
IsDebuggerPresent
check
```
```
Cheat checks if debugger
attached, goes dormant
```
```
Patch IsDebuggerPresent return in your
AC process
Process hiding (DKOM) Removes cheat process from
PsActiveProcessHead list
```
```
Kernel-level process enumeration via
direct structure walk
Module hiding Removes DLL from
InLoadOrderModuleList in PEB
```
```
Scan memory regions for PE headers, not
just module list
Timing checks Detects VM/debugger via RDTSC
instruction timing
```
```
Normalize timing in VM detection, use it
as data
AC signature scanning Scans for your AC code
patterns, disables/avoids
```
```
Obfuscate AC code, pack with
polymorphic packer
Thread hiding Hides cheat thread from
NtQuerySystemInformation
```
```
Kernel callback NtSetInformationThread
monitor
Handle table walking Finds AC handles to
terminate/deny access
```
```
Protect AC process handle with
PsProtectProcess
```

## 17 GAN Aimbots — AI-Powered Undetectable Input

Published in 2022 (arxiv 2205.07060) and increasingly deployed in 2025, GAN-based aimbots train a
generative model on real human mouse movement data then replay that model to produce mouse trajectories
that are statistically indistinguishable from human input. They require less than one hour of gameplay
data to train. This is the hardest category to detect at the input layer.

How GAN Aimbots Work

```
Record hours of real human mouse movement while playing the target game.
Train a GAN (Generative Adversarial Network) — generator produces mouse paths, discriminator
validates them against human data.
At runtime: instead of snapping to target, feed target angle into model — model outputs a
humanised path to reach it.
The output mouse trajectory has correct Brownian noise, natural deceleration, and biological
overshoot — passes all simple tests.
Some variants use screen-capture object detection (YOLO) rather than memory reading — fully
external, no process interaction.
```
```
Detection Strategies
Deep temporal analysis: even GAN paths have subtle frequency-domain artifacts detectable with FFT
analysis.
Contextual impossibility: GAN aimbot still achieves reaction times and accuracy distributions
outside human range.
Long-session statistical drift: human players fatigue, warm up, misplay — GAN models are perfectly
consistent over hours.
Input device fingerprinting: GAN output is sent via SendInput — real mice have hardware-specific
jitter signatures.
Computer-vision based YOLO aimbots: detect screen capture process, detect unusually high GPU usage
from object detection.
Anti-cheat academic benchmark: train your own classifier on labelled aimbot vs human sessions — ML
fights ML.
```

## 18 UE5 Defense Architecture — Server Authority

Server authority is the only true floor. Every client-side protection can be bypassed given time and
resources. The server must validate everything that matters. Design security into your UE5 architecture
from day one — retrofitting is expensive and always incomplete.

The Defense-in-Depth Stack

```
Layer 1 — Network Architecture Server authoritative movement, combat, economy. Client sends
input only.
```
```
Layer 2 — Server Validation Validate all client-reported positions, actions, speeds against
server simulation.
```
```
Layer 3 — Data Not Sent Server-side occlusion culling: don't replicate invisible enemies
to clients.
```
```
Layer 4 — Memory Encryption SCUE5 FSafeFloat/FSafeVector: encrypted variables frustrate
memory scanners.
```
```
Layer 5 — Integrity Checking VTable sentinel, module scan, memory region scan, material CRC
checks.
```
```
Layer 6 — Process Monitoring Detect foreign process handles, injected DLLs, overlay windows.
```
```
Layer 7 — Behavioral Telemetry Collect aim stats, reaction times, position history — detect
statistically.
```
```
Layer 8 — Kernel Protection EAC/BattlEye kernel driver or your own: detect Ring 0 cheats.
```
```
Layer 9 — Response Strategy Silent detection, shadow ban, delayed bans, hardware fingerprint
bans.
```

## 19 Memory Encryption and Secure Variables (SCUE5)

SCUE5 (Secure Client UE5) is the leading UE5 anti-cheat plugin on Fab. Its core approach: replace every
sensitive gameplay variable with an encrypted equivalent. Memory scanners see only ciphertext — they
cannot find the player's health value, position, or ammo count.

SCUE5 Secure Variable Types

```
Standard Type SCUE5 Equivalent What Changes in Memory
```
```
bool FSafeBool Stored as XOR-encrypted pair, switches representation on access
int32 FSafeInt Dual-buffer: value + encrypted shadow, key rotated each write
float FSafeFloat IEEE754 bits XOR'd with runtime key, base/shift container
FString FSafeString Characters encrypted individually, length obfuscated
FVector FSafeVector Each component independently encrypted
```
```
Implementing Secure Variables in Your UE5 Game
// C++ UE5 — Encrypted Variable Implementation
// BEFORE — cheat can scan for '100' and find player health
UPROPERTY()
float PlayerHealth = 100.0f;
// AFTER — memory contains encrypted value, never plaintext 100
UPROPERTY()
FSafeFloat PlayerHealth = 100.0f;
// Usage is identical — getter/setter handle encryption transparently
float hp = PlayerHealth.GetValue();
PlayerHealth.SetValue(hp - damage);
// Rolling your own (without SCUE5):
class FEncryptedFloat {
uint32 key;
uint32 encrypted_bits;
public:
FEncryptedFloat(float v) : key(FMath::Rand()) {
encrypted_bits = *reinterpret_cast<uint32*>(&v) ^ key;
}
float Get() const {
uint32 raw = encrypted_bits ^ key;
return *reinterpret_cast<float*>(&raw);
}
void Set(float v) {
key = FMath::Rand(); // re-key on every write
encrypted_bits = *reinterpret_cast<uint32*>(&v) ^ key;
}
};
```
```
Canary Variables — Honeypot Detection
// C++ UE5 — Canary Honeypot Pattern
// Place fake 'health' variables adjacent to real ones
// Any read that touches the canary = memory scanner detected
struct FPlayerStats {
FSafeFloat RealHealth; // actual value
volatile float Canary1; // set to 100.0f — looks like health
FSafeFloat RealArmor; // actual value
volatile float Canary2; // set to 100.0f — looks like armor
};
// Monitor canary modifications
void CheckCanaries() {
if (Stats.Canary1 != 100.0f || Stats.Canary2 != 100.0f) {
ReportSecurityEvent(ESecEvent::CanaryTripped);
// Reset to keep honeypot active
```

Stats.Canary1 = 100.0f;
Stats.Canary2 = 100.0f;
}
}


## 20 VTable Sentinel and Integrity Verification

Internal cheats hook your game's virtual function tables (VTables) to intercept function calls —
particularly PostRender for overlay injection and input functions for aim manipulation. VTable sentinel
monitoring detects these hooks in real-time.

```
// C++ UE5 — VTable Sentinel + Code Integrity
// Store original VTable pointers at startup
static uintptr_t OriginalVTables[NUM_WATCHED_CLASSES];
void CaptureVTableBaselines() {
OriginalVTables[0] = GetVTablePtr(APlayerController::StaticClass());
OriginalVTables[1] = GetVTablePtr(USkeletalMeshComponent::StaticClass());
OriginalVTables[2] = GetVTablePtr(ACharacter::StaticClass());
// ... all security-critical classes
}
// Run every 5 seconds on a dedicated thread
void VTableSentinelTick() {
if (GetVTablePtr(APlayerController::StaticClass()) != OriginalVTables[0]) {
ReportSecurityEvent(ESecEvent::VTableTampered, TEXT("APlayerController"));
}
if (GetVTablePtr(USkeletalMeshComponent::StaticClass()) != OriginalVTables[1]) {
ReportSecurityEvent(ESecEvent::VTableTampered, TEXT("USkeletalMesh"));
}
}
// Also verify .text section hash (code integrity)
void VerifyCodeIntegrity() {
uint32 current_hash = CRC32(game_text_section_start, game_text_section_size);
if (current_hash != stored_hash_at_startup) {
ReportSecurityEvent(ESecEvent::CodePatched);
}
}
```

## 22 Behavioral Detection — Telemetry and Heuristics

Even when client-side detection fails, server-side behavioral analysis catches cheaters through
statistical impossibility. Collect rich telemetry every session and run it through heuristic rules and
ML models.

```
UE5 Telemetry Data Structure
// C++ UE5 — Telemetry Struct
// Collect this per-player per-session, send to your analytics backend
USTRUCT()
struct FPlayerTelemetry {
UPROPERTY() FString PlayerID;
UPROPERTY() float SessionDurationSec;
// Aim telemetry
UPROPERTY() float HeadshotRate; // 0.0-1.0
UPROPERTY() float MedianReactionTimeMs;
UPROPERTY() float MinReactionTimeMs;
UPROPERTY() float AimSnapMaxDeltaDeg; // max single-frame rotation
UPROPERTY() float AimVarianceOnTarget; // should not be near-zero
UPROPERTY() float TargetSwitchTimeMs; // time between targets
// Movement telemetry
UPROPERTY() float MaxSpeedObserved;
UPROPERTY() int32 TeleportEventCount; // server detected
UPROPERTY() float AirTimeRatioPercent;
// Combat telemetry
UPROPERTY() float KDRatio;
UPROPERTY() float FirstShotAccuracy;
UPROPERTY() int32 ImpossibleHitCount; // behind-cover hits
// System telemetry
UPROPERTY() FString HardwareFingerprint;
UPROPERTY() TArray<FString> LoadedModuleHashes;
UPROPERTY() TArray<FString> SecurityEvents;
};
```
Heuristic Detection Rules

```
Rule Threshold Cheat Indicated Confidence
```
```
Headshot rate > 65% sustained over
30+ kills
```
```
Bone aimbot High
```
```
Min reaction time < 30ms over 10+
samples
```
```
Triggerbot Very
High
Aim snap delta > 90 deg in single
frame
```
```
Non-smooth aimbot Very
High
Aim variance on
target
```
```
< 0.1 deg
(near-zero)
```
```
Perfect lock
aimbot
```
```
High
```
```
Target switch
time
```
```
< 50ms average Multi-target
aimbot
```
```
High
```
```
Max speed
observed
```
```
> MaxWalkSpeed * 1.2 Speed hack Very
High
Teleport events > 3 per session Position cheat Very
High
Impossible hit
count
```
```
> 5 behind-cover
hits
```
```
Wallhack + aimbot High
```
```
Unknown module
hash
```
```
Any unknown DLL
loaded
```
```
Injection High
```
```
Security events Any
VTable/canary/code
event
```
```
Active cheat Very
High
```

## 26 Detection Response Strategy — Ban Tiers

How you respond to detected cheating is as important as how you detect it. Immediate bans on first
detection reveal your detection method to cheat makers. A tiered, delayed response maximises
effectiveness.

```
Tier Trigger Action Rationale
```
```
1 — Silent
Flag
```
```
Any anomaly detected Flag account, gather
data, no action yet
```
```
Don't alert cheat maker
that detection works
2 — Shadow
Ban
```
```
Multiple flags +
confidence > 80%
```
```
Move to cheat-only
matchmaking pool
```
```
Cheater thinks they're
playing normally; data
keeps flowing
3 — Delayed
Ban
```
```
High confidence +
pattern confirmed
```
```
Ban 7-14 days after
detection
```
```
Prevents cheat devs
correlating update timing
to detection trigger
4 — Hard Ban Confirmed cheat +
ban evasion attempt
```
```
Account + hardware
fingerprint ban
```
```
HWID ban makes creating
new accounts expensive
5 — Legal Commercial cheat
distribution
```
```
DMCA, civil action,
platform report
```
```
Reserve for organised
cheat sellers causing
significant harm
```
```
Hardware Fingerprinting in UE5
// C++ UE5 — Hardware Fingerprinting
// Collect multiple hardware identifiers and hash them
FString BuildHardwareFingerprint() {
TArray<FString> components;
// CPU serial (if available)
components.Add(GetCPUId());
// Primary drive serial
components.Add(GetDriveSerial());
// NIC MAC address (first physical adapter)
components.Add(GetMacAddress());
// Motherboard serial (WMI query)
components.Add(GetMotherboardSerial());
// GPU device ID
components.Add(GetGPUDeviceId());
// Hash all components together
FString combined = FString::Join(components, TEXT("|"));
return FMD5::HashAnsiString(*combined);
}
// Send fingerprint to your backend on login — ban if fingerprint is flagged
// Note: inform users this data is collected in your Privacy Policy
```

## 27 Building Your Own Anti-Cheat in UE5 — Full Blueprint

Implementation Priority Order

```
Priority Task Effort Impact
```
```
P0 — Day 1 Server authoritative movement + combat. Client
sends input only.
```
```
High Eliminates speed,
fly, position,
teleport cheats
P0 — Day 1 Server-side occlusion culling — don't replicate
invisible enemies.
```
```
Medi
um
```
```
Eliminates ESP and
radar at the
source
P0 — Day 1 Server-side combat validation — damage ranges, line
of sight.
```
```
Medi
um
```
```
Eliminates
wallhack shooting,
damage hacks
P1 — Week
1
```
```
Replace sensitive UProperties with
FSafeFloat/FSafeVector (SCUE5).
```
```
Low Frustrates memory
scanners
significantly
P1 — Week
1
```
```
Module whitelist scan + unsigned DLL detection. Low Detects most
common injection
cheats
P1 — Week
1
```
```
VTable sentinel on PlayerController, SkeletalMesh,
Character.
```
```
Low Detects hooks from
internal cheats
P1 — Week
1
```
```
Canary variables near health, ammo, position
structs.
```
```
Low Detects memory
scanning probes
P2 —
Month 1
```
```
Telemetry collection: aim stats, reaction times,
movement events.
```
```
Medi
um
```
```
Foundation for
behavioral
detection
P2 —
Month 1
```
```
Process handle monitoring for cross-process reads. Medi
um
```
```
Detects external
memory readers
P2 —
Month 1
```
```
Overlay window detection via EnumWindows scan. Low Detects DirectX
overlay ESP cheats
P2 —
Month 1
```
```
Behavioral heuristic rules: headshot rate, snap
angle, reaction time.
```
```
Medi
um
```
```
Catches
statistical
impossibilities
P3 —
Month 3
```
```
Hardware fingerprinting + ban database backend. High Makes ban evasion
expensive
P3 —
Month 3
```
```
EasyAntiCheat or BattlEye integration (if budget
allows).
```
```
Medi
um
```
```
Kernel-level
protection,
outsourced
P3 —
Month 3
```
```
ML model trained on labelled cheat vs legit
sessions.
```
```
High Catches
sophisticated
behavioral cheats
P4 —
Ongoing
```
```
Update module whitelist, add new cheat signatures
monthly.
```
```
Low Keeps detection
current
P4 —
Ongoing
```
```
Review ban appeals — tune false positive rate. Medi
um
```
```
Maintain player
trust
```
Minimum Viable Anti-Cheat — Code Skeleton

// C++ UE5 — Anti-Cheat Subsystem Skeleton
// AntiCheatSubsystem.h — UGameInstanceSubsystem
UCLASS()
class UAntiCheatSubsystem : public UGameInstanceSubsystem {
GENERATED_BODY()
public:
virtual void Initialize(FSubsystemCollectionBase& Collection) override;
virtual void Deinitialize() override;
private:
FTimerHandle ScanTimer;
void RunSecurityScan();


void ScanModules(); // check module whitelist
void ScanVTables(); // check VTable integrity
void ScanMemoryRegions(); // check for PE headers in anon memory
void ScanProcessHandles(); // check for external handle to our PID
void ScanOverlayWindows(); // check for layered windows over game
void CheckCanaries(); // check honeypot variables
void ReportEvent(ESecEvent Event, const FString& Detail);
void SendTelemetryBatch(); // send to your backend every 60s
};
// AntiCheatSubsystem.cpp
void UAntiCheatSubsystem::Initialize(FSubsystemCollectionBase& Collection) {
// Capture baselines at startup
CaptureVTableBaselines();
BuildModuleWhitelist();
CaptureCodeSectionHash();
// Run scans every 30 seconds
GetWorld()->GetTimerManager().SetTimer(
ScanTimer, this,
&UAntiCheatSubsystem::RunSecurityScan,
30.0f, true
);
}
void UAntiCheatSubsystem::RunSecurityScan() {
ScanModules();
ScanVTables();
ScanMemoryRegions();
ScanOverlayWindows();
CheckCanaries();
}


## 28 Resource Reference and Further Reading

UE5 Anti-Cheat Plugins

SCUE5 — Secure Client Anti-Cheat
https://www.fab.com/listings/8824ae87-df1a-4aee-b114-63666a8392fd
Leading UE5 anti-cheat plugin. Encrypted variables, VTable sentinel, process scanner. Fab marketplace.

Hard Anti-Cheat (AI-Powered)
https://hard.ac/
Server-side AI anti-cheat specifically for Unreal Engine. Neural network behavioral detection.

GetGud SDK
https://www.getgud.io/
Player behavior analysis and cheat detection SDK with UE5 integration.

Professional Anti-Cheat SDKs

Easy Anti-Cheat
https://www.easy.ac/
Epic Games owned. Deployed in Fortnite, Apex, 150+ titles. Kernel-level. Free for UE5 titles.

BattlEye
https://www.battleye.com/
Industry-leading kernel anti-cheat. PUBG, Rainbow Six, DayZ. License required.

Valve Anti-Cheat (VAC)
https://store.steampowered.com/anticheat/
Steam-integrated. Delayed ban strategy. Free for Steam titles.

Academic Research

GAN-Aimbots Paper
https://ar5iv.labs.arxiv.org/html/2205.07060
University research: ML-powered aimbots that pass statistical detection. Essential reading.

Anti-Cheat Attacks and Defenses 2024
https://tomchothia.gitlab.io/Papers/AntiCheat2024.pdf
Collins et al. Academic analysis of client-side anti-cheat effectiveness.

Black Hat USA 2025 — Game Security
https://www.darkreading.com/cyberattacks-data-breaches/video-game-anti-cheat-systems-cybersecurity-goldmine
Gaming PC as most-attacked endpoint. Anti-cheat as cybersecurity research frontier.

Technical References

UE5 Networking Deep Dive
https://docs.unrealengine.com/5.4/en-US/networking-overview-for-unreal-engine/
Official UE5 docs: server authority, replication, Net Cull Distance, relevancy.

SCUE5 Architecture Overview
https://thehokkaidohideout.com/2025/08/04/scue5-overview/
Deep dive into SCUE5 design: encrypted types, VTable sentinel, canary guards.

UE5 Memory Protection Guide
https://gist.github.com/gamedev44/28f2af7a8af042dfe64dfb8967cece30
Practical C++ guide: pointer obfuscation, ASLR, anti-debug checks for UE5.

DrCodes UE5 Anti-Cheat
https://drcodes.com/posts/defending-ue5-multiplayer-games-anti-cheat-memory-security
Comprehensive 2026 article: honeypot variables, dynamic allocation, response tiers.

loldrivers.io
https://www.loldrivers.io/
Database of known-vulnerable drivers used in BYOVD attacks. Essential for kernel defense.

PCILeech/DMA Documentation
https://github.com/ufrisk/pcileech
Reference DMA attack framework. Understand it to defend against it.

Awesome Game Security (GitHub)
https://github.com/gmh5225/awesome-game-security
Curated list of game security tools, attack frameworks, and defensive resources.


END OF DOCUMENT
UE5 Game Security — Attack Vectors and Anti-Cheat Architecture



