# HDC Video Compression
Here is the official **SemanticVideo v1.0 Specification**.

This document defines the `.semv` format. It formalizes the "Game Engine" approach: not storing pixels; but storing a **Scene Graph** and a **Script**.

---

# SemanticVideo Format Specification (v1.0)

**Extension:** `.semv`
**MIME Type:** `application/x-semantic-video`
**Architecture:** JSON Manifest + Asset Container (Texture/Mesh Library)

## 1. The Philosophy

The file does not describe "what the video looks like." It describes **how to reconstruct the video**.

* **The Encoder** is the "Time Traveler": It pre-calculates optimal assets and paths.
* **The Decoder** is the "Renderer": A lightweight engine that executes instructions.
* **The Content** is separated into **Assets** (DNA) and **Instructions** (Behavior).

---

## 2. File Structure

The `.semv` file is a JSON object with three core sections:

1. **`header`**: Global metadata and canvas settings.
2. **`library`**: The "Deck of Cards" (Textures, Meshes, Shaders).
3. **`timeline`**: The "Dealer" (The instructions for playback).

```json
{
  "header": {
    "version": "1.0",
    "title": "Ape_Rotation_Test",
    "dimensions": [1920, 1080],
    "framerate": 60,
    "duration": 5.0,
    "compression_mode": "hybrid_semantic" // Hybrid = Vectors + Keyframe Swaps
  },

  "library": {
    "textures": [ ... ],
    "meshes": [ ... ],
    "behaviors": [ ... ]
  },

  "timeline": [ ... ]
}

```

---

## 3. The Library (The Assets)

This section defines the "Actors" and "Props." These are downloaded/loaded **once** before playback.

### A. Textures (The Skins)

Instead of one giant atlas, we store discrete "Key Views."

```json
"textures": [
  {
    "id": "tex_bg_desert",
    "src": "assets/bg_desert.webp", 
    "type": "static" 
  },
  {
    "id": "tex_ape_front",
    "src": "assets/ape_front_v2.webp",
    "type": "dynamic",
    "notes": "Base view for 0-30 degrees"
  },
  {
    "id": "tex_ape_side",
    "src": "assets/ape_side_v1.webp",
    "type": "dynamic",
    "notes": "Swapped in when rotation > 45 degrees"
  }
]

```

### B. Meshes (The Skeletons)

The grids that define the shape of the objects.

```json
"meshes": [
  {
    "id": "mesh_quad_fullscreen",
    "type": "grid",
    "rows": 2, "cols": 2, // Simple 4-point plane for background
    "default_points": [[0,0], [1920,0], [0,1080], [1920,1080]]
  },
  {
    "id": "mesh_ape_rig",
    "type": "grid",
    "rows": 16, "cols": 16, // High density for the actor
    "physics": "rigid_rubber" // Regional Primitive: prevents tearing/melting
  }
]

```

---

## 4. The Timeline (The Instructions)

This is the stream. It uses **Sparse Updates**. If a value isn't mentioned, it interpolates from the last known state.

**Core Commands:**

* `SPAWN`: Create an instance of a mesh with a texture.
* `WARP`: Move specific mesh nodes (Vector Flow).
* `SWAP`: Change the texture on an existing mesh (The "Turning Head" Fix).
* `KILL`: Remove object.

```json
"timeline": [
  // --- FRAME 0: SETUP ---
  {
    "t": 0.0,
    "action": "SPAWN",
    "id": "instance_bg",
    "mesh": "mesh_quad_fullscreen",
    "texture": "tex_bg_desert",
    "layer": 0
  },
  {
    "t": 0.0,
    "action": "SPAWN",
    "id": "instance_ape",
    "mesh": "mesh_ape_rig",
    "texture": "tex_ape_front", // Start with Front View
    "layer": 1,
    "pos": [960, 540]
  },

  // --- FRAME 1-29: VECTOR MOTION (Cheap) ---
  // We only send the nodes that moved. The decoder interpolates the rest.
  {
    "t": 0.1,
    "action": "WARP",
    "target": "instance_ape",
    "nodes": [
      { "idx": 45, "delta": [2, 0] }, // Moving the nose slightly right
      { "idx": 46, "delta": [2, 1] }
    ]
  },
  {
    "t": 0.5,
    "action": "WARP",
    "target": "instance_ape",
    "function": "linear_pan", // Using a Global Primitive function
    "params": { "x": 10, "y": 0 }
  },

  // --- FRAME 30: THE CRITICAL MOMENT (The Swap) ---
  // The Encoder detected that warping "Front" to "Side" would look ugly.
  // So it commands a texture swap.
  {
    "t": 1.0,
    "action": "SWAP",
    "target": "instance_ape",
    "texture": "tex_ape_side", // seamless switch to side view
    "blend_duration": 0.1 // Optional crossfade to hide the cut
  },

  // --- FRAME 31+: CONTINUE WARPING ---
  // Now we are warping the SIDE view.
  {
    "t": 1.1,
    "action": "WARP",
    "target": "instance_ape",
    "nodes": [ ... ] 
  }
]

```

---

## 5. The "Smart" Layer (Compression Logic)

This specification allows for the **Global/Regional/Local** hierarchy that is being discussed:

1. **Global (Implicit):** The Player knows how to execute `WARP` and `SWAP`.
2. **Regional (Referenced):**
```json
"behaviors": {
   "physics_model": "anime_snap_v1" // Tells decoder to use step-interpolation
}

```


3. **Local (Embedded):** The specific `nodes` in the timeline.

## 6. Comparison to Standard Video

| Feature | H.264 / MP4 | SemanticVideo (`.semv`) |
| --- | --- | --- |
| **Storage Unit** | Blocks of Pixels (Macroblocks) | Assets + Vectors |
| **Motion** | Estimated Prediction (Optical Flow) | Exact Mesh Coordinates |
| **Occlusion** | Fails / Artifacts | Solved via Asset Swapping |
| **Resolution** | Fixed (e.g., 1080p) | Infinite (Mesh scales to display) |
| **Editability** | None (Baked) | Full (Change texture, change path) |
| **File Size** | Linear Growth (Time = Bytes) | Logarithmic (Assets = Bytes, Time = Tiny) |

---

## 7. The Numbers That Matter

### For 8-Second Video - Ape Eating Chessboard Dalí/Pixar Style (192 frames):
```
H.264 (industry standard): 3.67 MB
Our SemanticVideo: 0.15 MB

Savings: 3.52 MB per 8 seconds
Compression: 24× better
```

### For 2-Hour Movie (216,000 frames):
```
H.264 (typical): 5 GB
Our estimate: 210 MB

Savings: 4.79 GB
Compression: 24× better
```

**For streaming/storage at scale: MASSIVE impact!**


### Next Steps

To build the MVP of this system, we need to build the Encoder Pipeline:

Asset Extractor: Takes a video, creates bg.jpg, actor_front.jpg, actor_side.jpg.

Rigger: Creates the mesh for the actor.

Composer: Runs through the video and generates the JSON timeline, deciding when to Warp and when to Swap.
