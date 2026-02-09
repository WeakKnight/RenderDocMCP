# RenderDoc MCP Improvement Proposal

## Background

When analyzing RenderDoc captures from Unity projects, the following challenges exist:

1. **UI Noise Problem**: When capturing from Unity Editor, large amounts of Editor UI drawing such as `GUI.Repaint` and `UIR.DrawChain` are included, making it difficult to find actual game rendering (under `Camera.Render`)
2. **Response Size Problem**: Results from `get_draw_calls(include_children=true)` exceed 70KB, consuming LLM context
3. **Exploration Inefficiency**: To find draw calls using specific shaders or textures, each draw call must be checked one by one

## Improvement Proposals

### 1. Marker Filtering (Priority: High)

Functionality to retrieve only actions under specific markers or exclude specific markers.

```python
get_draw_calls(
    include_children=True,
    marker_filter="Camera.Render",  # Get only actions under this marker
    exclude_markers=["GUI.Repaint", "UIR.DrawChain", "UGUI.Rendering"]
)
```

**Use Cases**:
- Extract only game rendering from Unity Editor captures
- Investigate specific rendering passes (Shadows, PostProcess, etc.)

**Expected Results**:
- Reduce response size to 10-20%
- Fits within size directly analyzable by LLM

---

### 2. event_id Range Specification (Priority: High)

Functionality to retrieve only specific event_id ranges.

```python
get_draw_calls(
    event_id_min=7372,
    event_id_max=7600,
    include_children=True
)
```

**Use Cases**:
- When `Camera.Render` event_id is known, retrieve only that surrounding area
- Investigate details around problematic draw calls

**Expected Results**:
- Fast retrieval of only necessary parts
- Enables step-by-step exploration

---

### 3. Shader/Texture/Resource Reverse Lookup (Priority: Medium)

Functionality to search for draw calls using specific resources.

```python
# Search by shader name (partial match)
find_draws_by_shader(shader_name="Toon")

# Search by texture name (partial match)
find_draws_by_texture(texture_name="CharacterSkin")

# Search by resource ID (exact match)
find_draws_by_resource(resource_id="ResourceId::12345")
```

**Return Value Example**:
```json
{
  "matches": [
    {"event_id": 7538, "name": "DrawIndexed", "match_reason": "pixel_shader contains 'Toon'"},
    {"event_id": 7620, "name": "DrawIndexed", "match_reason": "pixel_shader contains 'Toon'"}
  ],
  "total_matches": 2
}
```

**Use Cases**:
- Directly answer the most common question: "Which draws use this shader?"
- Track where specific textures are used
- Identify impact scope of shader bugs

---

### 4. Frame Summary Retrieval (Priority: Medium)

Functionality to get an overview of the entire frame.

```python
get_frame_summary()
```

**Return Value Example**:
```json
{
  "api": "D3D11",
  "total_events": 7763,
  "statistics": {
    "draw_calls": 64,
    "dispatches": 193,
    "clears": 5,
    "copies": 8
  },
  "top_level_markers": [
    {"name": "WaitForRenderJobs", "event_id": 118},
    {"name": "CustomRenderTextures.Update", "event_id": 6451},
    {"name": "Camera.Render", "event_id": 7372},
    {"name": "UIR.DrawChain", "event_id": 6484}
  ],
  "render_targets": [
    {"resource_id": "ResourceId::22573", "name": "MainRT", "resolution": "1920x1080"},
    {"resource_id": "ResourceId::22585", "name": "ShadowMap", "resolution": "2048x2048"}
  ],
  "unique_shaders": {
    "vertex": 12,
    "pixel": 15,
    "compute": 8
  }
}
```

**Use Cases**:
- Understand overall picture as starting point for exploration
- Decide which markers to examine in detail
- Understand performance overview

---

### 5. Draw Calls Only Mode (Priority: Medium)

Functionality to exclude markers (PushMarker/PopMarker) and retrieve only actual draw calls.

```python
get_draw_calls(
    only_actions=True,  # Exclude markers
    flags_filter=["Drawcall", "Dispatch"]  # Only specific flags
)
```

**Use Cases**:
- When only the total count and list of draw calls is needed
- When investigating only Compute Shaders (Dispatch)

---

### 6. Batch Pipeline State Retrieval (Priority: Low)

Functionality to retrieve pipeline states for multiple event_ids at once.

```python
get_multiple_pipeline_states(event_ids=[7538, 7558, 7450, 7458])
```

**Return Value Example**:
```json
{
  "states": {
    "7538": { /* pipeline state */ },
    "7558": { /* pipeline state */ },
    "7450": { /* pipeline state */ },
    "7458": { /* pipeline state */ }
  }
}
```

**Use Cases**:
- Comparative analysis of multiple draw calls
- Difference investigation (comparing normal and abnormal draws)

---

## Priority Summary

| Priority | Feature | Implementation Difficulty | Effect |
|----------|---------|--------------------------|--------|
| **High** | Marker Filtering | Medium | Dramatic improvement by removing UI noise |
| **High** | event_id Range Specification | Low | Speed up with partial retrieval |
| **Medium** | Shader/Texture Reverse Lookup | High | Directly supports most common use case |
| **Medium** | Frame Summary | Medium | Useful as exploration starting point |
| **Medium** | Draw Calls Only Retrieval | Low | Simple filtering |
| **Low** | Batch Retrieval | Low | Efficiency improvement but not essential |

## Unity-Specific Filtering Presets (Optional)

Unity-specific presets would be convenient:

```python
get_draw_calls(
    preset="unity_game_rendering"
)
```

**Preset Contents**:
- `marker_filter`: "Camera.Render"
- `exclude_markers`: ["GUI.Repaint", "UIR.DrawChain", "GUITexture.Draw", "UGUI.Rendering.RenderOverlays", "PlayerEndOfFrame", "EditorLoop"]

---

## Implementation Reference: Current Workflow Issues

### Current Flow

```
1. get_draw_calls(include_children=true)
   → Returns 76KB JSON (saved to file)

2. Analyze file with external tool (Python, etc.)
   → Identify Camera.Render event_id (e.g.: 7372)

3. Manually specify event_id range for detailed investigation
   → get_pipeline_state(7538), get_shader_info(7538, "pixel"), ...
```

### Ideal Improved Flow

```
1. get_frame_summary()
   → Learn that Camera.Render is at event_id: 7372

2. get_draw_calls(marker_filter="Camera.Render", exclude_markers=[...])
   → Get only necessary draw calls (few KB)

3. find_draws_by_shader(shader_name="MyShader")
   → Directly returns relevant event_ids

4. get_pipeline_state(event_id) for detailed confirmation
```

---

## Appendix: Unity Markers to Skip

Markers to exclude from Unity Editor captures:

| Marker Name | Description |
|-------------|-------------|
| `GUI.Repaint` | IMGUI drawing |
| `UIR.DrawChain` | UI Toolkit drawing |
| `GUITexture.Draw` | GUI texture drawing |
| `UGUI.Rendering.RenderOverlays` | uGUI overlay |
| `PlayerEndOfFrame` | End of frame processing |
| `EditorLoop` | Editor loop processing |

Conversely, important markers:

| Marker Name | Description |
|-------------|-------------|
| `Camera.Render` | Main camera rendering start point |
| `Drawing` | Drawing phase |
| `Render.OpaqueGeometry` | Opaque object rendering |
| `Render.TransparentGeometry` | Transparent object rendering |
| `RenderForward.RenderLoopJob` | Forward rendering draw calls |
| `Camera.RenderSkybox` | Skybox rendering |
| `Camera.ImageEffects` | Post-processing |
| `Shadows.RenderShadowMap` | Shadow map generation |
