---
name: video-to-unreal
description: Recreate a short reference video (usually an AI-generated clip) as a real, explorable Unreal Engine 5 scene, built 100% procedurally inside Unreal, then match its camera and lighting and make it playable. Adapts gokaygokay's video-to-shader method (breakdown, build, compare at matched times, repair) to a game engine, driven through Unreal's official MCP plugin. Use when asked to turn a video or clip into an Unreal level, scene or playable game slice.
---

# Video to Unreal

**What this skill is.** gokaygokay's `video-to-shader` method ([shader-skills](https://github.com/gokayfem/shader-skills), `prompts/video-to-shader-prompt.md`) treats the video as the spec. It extracts frames, writes a measured breakdown, builds the scene, compares it to the reference at the same times and fixes the biggest differences. This skill keeps that method and swaps the renderer for Unreal Engine 5 (built and tested on 5.8). The engine gives you real meshes, Nanite, Lumen global illumination, real materials, Sequencer and gameplay.

The shader route is fast, light and runs in a browser. This route is slower but looks closer: use it when surface detail, lighting and playability matter more than speed. In our test, the Unreal round took about 3–4x the shader round's time.

The goal is **the closest, most technically impressive replication you can make**. Quality comes before speed, but always save a working state before starting a bigger push.

## Non-negotiable rules

1. **Everything is procedural inside Unreal.**
   - **Meshes:** generate every one with Geometry Script (the `GeometryScripting` plugin, through Python or Editor Utility), PCG graphs, ProceduralMeshComponent, or Modeling Tools operations driven by script.
   - **No external assets:** no Fab, Megascans, Quixel, Starter Content, Marketplace, Blender or imported meshes or textures, and no `/Engine/BasicShapes` stand-ins in the final scene.
   - **Engine systems are allowed:** these are procedural systems, not art assets. That covers SkyAtmosphere, Volumetric Clouds, Exponential Height Fog, the Water plugin, Niagara, Lumen, Nanite, Substrate, material noise nodes and material functions.
   - **Materials and textures:** build these from material graph math (noise, gradients, procedural wood, metal and parchment, world-aligned masks). Rendering a texture *from your own procedural material* into a Render Target or texture is fine.
2. **The reference video is the spec and is never altered.** Never project or play reference frames inside the scene, and never use AI video or image generation for geometry, textures or viewpoints. Inferred content (anything the clip never shows) is allowed but must be logged as inferred.
3. **Keep the project isolated.** Create a new project folder next to your other Unreal projects and don't touch other projects.
4. **No purchases, paid APIs or cloud jobs.**

## Project setup

- **Start from a blank C++-capable project** and enable these plugins:
  - `ModelContextProtocol` and `AllToolsets`: the official Unreal MCP server and its tools (editor only).
  - `PythonScriptPlugin` and `EditorScriptingUtilities`.
  - `GeometryScripting` and `ModelingToolsEditorMode`.
  - `PCG`, `EnhancedInput`, `Water` (if the scene has water), and `MovieRenderPipeline` (Movie Render Queue). Add `ProceduralMeshComponent` if you use it.
- **MCP auto-start:** in `<Project>\Saved\Config\WindowsEditor\EditorPerProjectUserSettings.ini` add:
  ```
  [/Script/ModelContextProtocolEngine.ModelContextProtocolSettings]
  bAutoStartServer=True
  ServerPortNumber=<free port, e.g. 8021>
  ServerUrlPath=/mcp
  ```
  Or launch with `-ModelContextProtocolStartServer -ModelContextProtocolPort=<port>`.
- **Driving the MCP:** if your agent session has no native Unreal MCP tools, talk to `http://127.0.0.1:<port>/mcp` directly with a small Python JSON-RPC client over streamable HTTP. Send `initialize`, keep the `Mcp-Session-Id` header, send `notifications/initialized`, then `tools/call`.
  - With tool search on, the server exposes only `list_toolsets`, `describe_toolset` and `call_tool`. Discover the toolsets, then dispatch through `call_tool` with `toolset_name`, `tool_name` and `arguments`.
  - The official Unreal MCP agent skill has its own safety rules: save before and after bulk changes, wait for compiles, serialize dependent calls, check every result, and mind PIE.
- **Bulk geometry:** for heavy procedural geometry, Python inside the editor is usually faster than hundreds of single-actor calls. Run it through an MCP Python or console tool, Unreal's Python remote execution, or `UnrealEditor-Cmd.exe <uproject> -run=pythonscript -script=<file>`.
  - Keep the generator scripts in the project (`<Project>\Scripts\gen_*.py`) so every mesh can be regenerated and edited. They are the editable source.
- **Launching the editor:** run it in the background and poll the MCP endpoint until it answers. First launches compile shaders, so that's slow. If the Zen storage server hangs at startup, Unreal kills it and relaunches it after its timeout.

## Method

### 1. Reference breakdown (no approval stop; write it and keep going)
Extract frames with ffmpeg: every 0.5 s, finer around fast motion. Look at them closely and upscaled. Write `REFERENCE.md` with:
- **Clip facts:** duration, resolution, fps, cuts, loop.
- **Shot timeline:** time, beat, what the frame shows, and where the camera is.
- **Object inventory:** every object larger than about 2% of the frame. List its parts (an object reduced to its most obvious part fails), its shape language, its size relative to a ruler (door height, desk height, the hero's height) and its position. Measure from several frames and check that they agree.
- **Materials:** for each surface, the base colour sampled as sRGB means from crops, roughness and metal, wear, grain direction and pattern scale.
- **Camera log:** path, height, FOV estimate, speed, easing and tilt per beat. Solve the path into numbers you can key in Sequencer; annotating landmarks across keyframes works well.
- **Light and palette:** every light source (type, colour, rough intensity, flicker), the key/fill direction, shadow softness, fog/haze, bloom and grade.
- **What you can't reproduce exactly,** and the procedural substitute for each.

### 2. Blockout first (target: a running, saved blockout in about 30–60 min)
Build the space at real-world scale in centimetres with simple procedural primitives. Place the camera in Sequencer to match the reference path, and set rough Lumen lighting. Render matched frames and compare the composition. **Composition and camera are locked before any detail work.**

### 3. Detail passes, ordered by what the eye reads first
1. **Silhouettes and big forms:** walls, ceiling beams, windows, furniture, hero props.
2. **Secondary forms:** trims, panels, mouldings, frames, bevels, window grids, book rows, rope.
3. **Surface detail:** procedural materials, edge wear, grain, dirt in crevices, emissive sources. Nanite is fine for dense generated geometry.
4. **Lighting and atmosphere:** Lumen GI and reflections, practical lights with flicker, volumetric fog and light shafts, sky and outdoor elements, the post-process grade matched to the reference.
5. **Motion:** anything that moves in the clip (flames, water, cloth sway, swinging lanterns, particles), using Niagara, material animation or simple Blueprint/C++ motion.

For repeated items (books, planks, rivets, panels, bottles), write one parameterized generator with seeded variation. Hand-placing near-identical copies is not the method.

### 4. Compare at matched times, then repair
- **Render the comparison frames** with Movie Render Queue, or a high-resolution capture from the Sequencer camera, at the reference timeline rows (at least start / ¼ / ½ / ¾ / end). Make side-by-side images (ffmpeg `hstack`) at the reference resolution.
- **Color check:** for key regions, report the mean RGB and spread of the reference versus the render. Don't trust colour alone. A flat frame painted with the right colours would pass, so also judge layout and silhouettes by eye on the side-by-sides.
- **Repair loop:** each pass, write the five biggest visible differences and fix them, then re-render and compare. **Run at most three passes on the cinematic match, then reassess.** If a feature can't be reproduced, simplify honestly and write it down.

### 5. Deliverables (save a working state after each)
1. **Cinematic match:** a Level Sequence that reproduces the reference camera path and timing, rendered to MP4 (1440p60 is a good target) plus the matched comparison stills.
2. **Variants of the same scene:** an off-axis camera that shows parallax and depth, and an unmistakable lighting change (for example a storm, dawn, or a lanterns-only blackout).
3. **Playable mode:** a game mode that runs in Play-in-Editor and standalone.
   - Movement fitting the clip: first-person walk for a walkthrough clip, a driven pawn for a vehicle clip.
   - Collision, Enhanced Input and a few interactions that fit the scene (pick up or inspect an object, open something, light or blow out a lamp).
   - A free or photo camera and a lighting toggle.
   - On-screen fps and resolution.
4. **Package:**
   - **Launchers:** `Open_Editor.bat`, `Play.bat` (standalone: `UnrealEditor.exe <uproject> <map> -game -windowed`), and optionally `Render_Cinematic.bat`.
   - **README.md:** controls, launch steps, and what was matched versus inferred.
   - **Generator scripts.**
   - **An outputs folder:** `stills\` and `clips\` (cinematic, off-axis, lighting variant and a gameplay capture).

### 6. Performance
Measure the frame rate interactively at a stated resolution and scalability. Keep that separate from offline render time. Report the GPU, resolution, Lumen and Nanite settings, and frame time.

## Logging
Keep `BUILD_LOG.md` in the output folder:
- wall-clock timestamps at the start, each milestone and the end
- what was built
- every comparison pass and its five differences
- measured performance and the remaining defects

Don't invent token counts or costs. Measure them from the agent transcript.

## Final report
A concise report covering:
- launch paths and controls
- what works
- matched versus inferred
- the three biggest remaining mismatches
- performance numbers
- elapsed time per milestone
- the output file list

## Credits
The method is adapted from gokaygokay's ([@gokayfem](https://x.com/gokayfem)) [shader-skills](https://github.com/gokayfem/shader-skills) video-to-shader prompt. The Unreal translation was made by MattVidPro with Claude (Opus 5.5) for a MattVidPro video where it rebuilt an AI-generated pirate cabin in Unreal Engine 5.8.
