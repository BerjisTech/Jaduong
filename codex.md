 CODEX TASK — UE5.6 IN-GAME MULTIPLAYER BUILDING SYSTEM (PLUGIN-AWARE)

 Project / Module:
  - Unreal Engine 5.6
  - Module Name: Jaduong

 Goal (short):
  Produce a server-authoritative, multiplayer, persistent in-game building system that
  lets players place walls, windows, doors, floors, and stairs in runtime (not editor).
  Multiple players can co-edit the same building simultaneously. Building data is compactly
  replicated as descriptor USTRUCTs (not vertex buffers). Server persists buildings to disk
  as JSON (Saved/Builds/<Level>_buildings.json) and supports optional SQLite saving via SQLiteSupport.

 Plugins / runtime capabilities to use:
  - ReplicationGraph (efficient replication)
  - OnlineSubsystem* (Steam/EOS/Null) — for player identity (use Null by default)
  - JsonBlueprintUtilities / FJsonSerializer (JSON save/load)
  - SQLiteSupport (optional persistence backend)
  - ProceduralMeshComponent or RuntimeMeshComponent (dynamic geometry on clients)
  - InstancedStaticMeshComponent for repeated modular pieces (frames, doors, windows)
  - Replication-friendly patterns: Server RPCs, NetMulticast for FX, UPROPERTY replication

 Do NOT use editor-only plugins for runtime features. Mention editor plugins only as prototyping aids in comments.

 Files to produce (complete .h + .cpp) — generate exact code for these:
  1. BuildingStructs.h / BuildingStructs.cpp  (USTRUCT definitions, helper functions)
  2. BuildingActor.h / BuildingActor.cpp      (ABuildingActor — replicated authoritative building instance)
  3. BuildingManager.h / BuildingManager.cpp  (ABuildingManager — spawned by GameMode, persistence, save/load/periodic save)
  4. PlayerBuildingController.h / PlayerBuildingController.cpp (APlayerController subclass — BuildMode client UI hooks + RPCs)
  5. Jaduong.Build.cs snippet (module references to ProceduralMeshComponent, ReplicationGraph, SQLiteSupport etc.)

 Required content & behavior (be explicit in code):

 A. BuildingStructs (USTRUCT BlueprintType, ReplicatedWhenUsed)
   - FWallDesc
     - FString WallID
     - FVector Start, End
     - float Height, Thickness
     - TArray<FOpeningDesc> Openings
   - FOpeningDesc
     - FString OpeningID
     - uint8 Type (0=Window,1=Door) or enum
     - float CenterOnWall, Width, Height, SillHeight
   - FStairDesc
     - FString StairID
     - FVector Start, End
     - int32 FromFloor, ToFloor
   - FFloorPatchDesc
     - FString PatchID
     - FVector Center; FVector2D Size; int32 FloorLevel
   - Each USTRUCT must contain helper functions (Length(), Direction(), ToJsonObject(), FromJsonObject())

 B. ABuildingActor (server-authoritative)
   - Replicated properties:
     - FString BuildingID
     - int32 Revision  (increment on each change)
     - TArray<FWallDesc> Walls  (UPROPERTY(ReplicatedUsing=OnRep_BuildingChanged))
     - TArray<FOpeningDesc> Openings or included in Walls as nested arrays
     - TArray<FFloorPatchDesc> FloorPatches
     - TArray<FStairDesc> Stairs
   - Components:
     - UProceduralMeshComponent* ProcMesh  (CreateDefaultSubobject; visible on clients)
     - UInstancedStaticMeshComponent* FrameInstances (for window/door frames)
   - Server RPCs (UFUNCTION(Server, Reliable)):
     - Server_AddWall(const FWallDesc& WallDesc)
     - Server_RemoveWall(const FString& WallID)
     - Server_AddOpening(const FString& WallID, const FOpeningDesc& OpeningDesc)
     - Server_AddStairs(const FStairDesc& StairDesc)
     - Server_RemoveFloorPatch(const FString& PatchID)
     - All server RPCs must validate (basic checks: range, wall length > epsilon, opening fits inside wall, unique IDs)
   - Client notifications:
     - OnRep_BuildingChanged() rebuilds the local ProcMesh from descriptor arrays (client-side mesh builder).
     - Multicast RPC Multicast_PlayPlacementFX(...) for visible effects when server confirms placement.

   - Mesh strategy:
     - Do NOT replicate raw vertex buffers. Replicate descriptor arrays only. After replication, clients reconstruct geometry
       using a deterministic builder: BuildWallMeshes(Walls, Openings, FloorPatches, Stairs).
     - Implement a simple rectangular wall builder in C++ that supports rectangular openings (window/door).
     - Use earcut triangulation algorithm (include a brief in-file implementation or call helper) or simple rectangle-with-hole triangulation sufficient for rectangular openings.

C. ABuildingManager (singleton authoritative manager, server-only write)
   - Spawned by GameMode on server BeginPlay (or placed in level).
   - Holds TMap<FString, ABuildingActor*> ActiveBuildings.
   - Persistence functions (server-only):
     - SaveAllBuildingsToJson(const FString& Path)   writes Saved/Builds/<Level>_buildings.json
     - LoadAllBuildingsFromJson(const FString& Path)  spawns ABuildingActor for each building
     - Optional: SaveToSQLite() / LoadFromSQLite() using SQLiteSupport (wrap in #if WITH_SQLITE support)
   - Periodic autosave (FTimerHandle, default 60s) and Exec commands:
     - Exec: SaveBuildingsNow, LoadBuildingsNow
   - Ensure only server writes disk. Clients can request Save (RPC) but manager must run it on server.

D. APlayerBuildingController (client UI + RPC glue)
   - Extends APlayerController (or your MultiplayerPlayerController).
   - Local preview mode: when player enters BuildMode, the controller provides client-side preview placement (no server effect).
   - When player confirms placement, call Server RPC on ABuildingActor via APlayerBuildingController -> Server_RequestAddWall(BuildingID, FWallDesc).
   - Provide functions for ToggleBuildMode(), Toggle2DView() — implement camera switch to orthographic top-down view for precise placement.
   - Provide input mapping comments, and example BindAction in code to ToggleBuildMode.

E. Replication and concurrency policy
   - Server is authoritative. Clients send requests (Server_* RPCs). Server validates and applies with atomic revision++.
   - After applying changes, server calls ForceNetUpdate() or relies on property replication to push descriptors to clients.
   - Use last-write-wins with Revision ints. Log conflicts using UE_LOG. Optionally implement soft locks (server grants temporary edit locks to players).

F. Persistence JSON schema (example):
  {
    "schemaVersion": 1,
    "buildings": [
      {
        "buildingID":"bld_001","revision":12,
        "walls":[ { "wallID":"w1","start":[0,0,0],"end":[400,0,0],"height":300,"thickness":10, "openings":[ ... ] } ],
        "stairs":[ ... ],
        "floorPatches":[ ... ]
      }
    ]
  }

G. Jaduong.Build.cs snippet
   - Add module dependencies: "Core", "CoreUObject", "Engine", "InputCore", "ProceduralMeshComponent", "ReplicationGraph", "OnlineSubsystem", "Json", "JsonUtilities", "SQLiteSupport" (conditionally).

H. Code style / constraints
   - Provide full, compiling code (include necessary headers).
   - Use GENERATED_BODY() and appropriate macros.
   - Implement GetLifetimeReplicatedProps for all replicated USTRUCT arrays.
   - Provide comments that call out which parts are intentionally simplified and where to extend (e.g., earcut algorithm, advanced navmesh updates).
   - Add author header in each file:
      Author : @BerjisTech <ben@proz.com>
      Date : 2025-10-08

I. Testing instructions (in comments):
   - How to test in PIE with 2 windows (Play -> New Editor Window (PIE) -> 2 players) and verify:
     * Place walls on client -> server validates -> walls replicate to other client.
     * SaveBuildingsNow -> exit PIE -> LoadBuildingsNow -> building reappears.
   - How to test offline resume: run dedicated server, spawn IR, use Save/Load.

J. Performance & production notes (comments):
   - Recommend using InstancedStaticMesh for frames and modular pieces.
   - Recommend moving heavy triangulation to a small static helper or 3rd-party earcut C++ library.
   - For large numbers of buildings, enable ReplicationGraph and compact the descriptor payloads.

Author: @BerjisTech <ben@proz.com>
Date: 2025-10-08

 ---- END PROMPT
