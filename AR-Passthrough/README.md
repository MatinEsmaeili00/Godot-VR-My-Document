# 🥽 Single Player AR Passthrough Setup

### 🎮 Built with Godot 4.6.2 for Meta Quest 3

This document explains a **simple single-player AR passthrough setup** for the VR project.

The goal is simple:

> The player presses `grip_click`, and the game switches between normal VR mode and AR passthrough mode.

There is **no server/client logic** in this version.  
There is **no RPC**.  
There is **no multiplayer broadcasting**.

This version is only for testing passthrough locally on one headset.

---

## 🧩 Project Version

- **Godot version:** 4.6.2 stable
- **Target headset:** Meta Quest 3
- **XR system:** Godot XR Tools + OpenXR
- **Vendor plugin:** Godot OpenXR Vendors plugin
- **Main feature:**  
  Pressing `grip_click` toggles AR passthrough ON/OFF locally.

---

## ✅ Current Goal

When the player presses the grip button:

```text
Player presses grip_click
        ↓
Game toggles passthrough state
        ↓
Current headset switches between VR and AR passthrough
```

Important idea:

```text
Passthrough is a local headset/XR setting.
For single-player testing, we only need to change it on the current headset.
```

---

## 🎮 Input Action

The action name from the OpenXR Action Map is:

```text
grip_click
```

Make sure the script checks exactly this name:

```gdscript
"grip_click":
```

Do not use:

```text
grip
grip_button
grip_pressed
```

---

# 🧱 Required Project Settings

## Project Settings

Go to:

```text
Project Settings
→ Advanced Settings: ON
→ XR
→ OpenXR
```

Recommended settings:

```text
Enabled: On
Environment Blend Mode: Alpha
Reference Space: Stage
```

Then go to:

```text
XR
→ OpenXR
→ Extensions
→ Meta
```

Enable:

```text
Passthrough: On
```

---

# 📦 Required Android Export Settings

Go to:

```text
Project
→ Export
→ Android Preset
→ Options
→ Meta XR Features
```

Set:

```text
Passthrough: Optional
Boundary Mode: Enabled
Quest 2 Support: On
Quest 3 Support: On
Quest Pro Support: On
```

For debugging only, you can temporarily set:

```text
Passthrough: Required
```

Use `Required` only to test if the Android build is correctly requesting passthrough support.

---

# ⚠️ Important Build Note

Restarting the Godot editor is not enough by itself.

After changing OpenXR / Meta XR settings, do this:

```text
1. Save the project.
2. Stop the current running game.
3. Re-export / run the Android build again.
4. Install the new build on the Quest.
5. Fully close the old app on the Quest.
6. Test inside the headset.
```

If the old build is still installed/running, the project may still report only:

```text
Supported XR blend modes: OPAQUE
```

That means the running app still only has normal VR mode available.

---

# 🧠 Code Setup

For a clean single-player setup, create a new script:

```text
ar_passthrough_controller.gd
```

Attach this script to a normal `Node` in your main scene.

Example scene setup:

```text
MainScene
├── XROrigin3D
├── WorldEnvironment
├── ARPassthroughController
└── PlayerController
```

---

## `ar_passthrough_controller.gd`

This script is responsible for:

- Toggling passthrough ON/OFF.
- Checking the supported XR blend modes.
- Applying `ALPHA_BLEND` if the runtime supports it.
- Falling back to `ADDITIVE` if available.
- Returning to `OPAQUE` when passthrough is turned off.
- Making the viewport background transparent when needed.
- Printing useful debug logs.

```gdscript
extends Node

@export var world_environment: WorldEnvironment

var _passthrough_enabled: bool = false


func toggle_passthrough() -> void:
	var target_state := not _passthrough_enabled

	print("Trying to set passthrough to: ", target_state)

	var success := _apply_passthrough_locally(target_state)

	if success:
		_passthrough_enabled = target_state
		print("Passthrough state is now: ", _passthrough_enabled)
	else:
		print("Passthrough toggle failed. State was not changed.")


func _apply_passthrough_locally(enable: bool) -> bool:
	var xr_interface: XRInterface = XRServer.primary_interface

	if xr_interface == null:
		xr_interface = XRServer.find_interface("OpenXR")

	if xr_interface == null:
		print("ERROR: No XR interface found. Cannot change passthrough.")
		return false

	var viewport := get_viewport()
	var env_node := _get_world_environment()
	var modes := xr_interface.get_supported_environment_blend_modes()

	print("Supported XR blend modes: ", _blend_modes_to_string(modes))

	if enable:
		if XRInterface.XR_ENV_BLEND_MODE_ALPHA_BLEND in modes:
			xr_interface.environment_blend_mode = XRInterface.XR_ENV_BLEND_MODE_ALPHA_BLEND
			viewport.transparent_bg = true
			print("Using ALPHA_BLEND passthrough mode.")

		elif XRInterface.XR_ENV_BLEND_MODE_ADDITIVE in modes:
			xr_interface.environment_blend_mode = XRInterface.XR_ENV_BLEND_MODE_ADDITIVE
			viewport.transparent_bg = false
			print("Using ADDITIVE passthrough mode.")

		else:
			print("ERROR: No AR passthrough blend mode is supported. Only VR/OPAQUE is available right now.")
			return false

		if env_node != null and env_node.environment != null:
			env_node.environment.background_mode = Environment.BG_COLOR
			env_node.environment.background_color = Color(0.0, 0.0, 0.0, 0.0)
			env_node.environment.ambient_light_source = Environment.AMBIENT_SOURCE_COLOR

		print("AR passthrough ON locally.")
		return true

	else:
		if XRInterface.XR_ENV_BLEND_MODE_OPAQUE in modes:
			xr_interface.environment_blend_mode = XRInterface.XR_ENV_BLEND_MODE_OPAQUE

		viewport.transparent_bg = false

		if env_node != null and env_node.environment != null:
			env_node.environment.background_mode = Environment.BG_SKY
			env_node.environment.ambient_light_source = Environment.AMBIENT_SOURCE_BG

		print("AR passthrough OFF locally.")
		return true


func _get_world_environment() -> WorldEnvironment:
	if world_environment != null:
		return world_environment

	var found := get_tree().root.find_child("WorldEnvironment", true, false)

	if found is WorldEnvironment:
		world_environment = found as WorldEnvironment

	return world_environment


func _blend_modes_to_string(modes: Array) -> String:
	var names: Array[String] = []

	for mode in modes:
		match mode:
			XRInterface.XR_ENV_BLEND_MODE_OPAQUE:
				names.append("OPAQUE")
			XRInterface.XR_ENV_BLEND_MODE_ADDITIVE:
				names.append("ADDITIVE")
			XRInterface.XR_ENV_BLEND_MODE_ALPHA_BLEND:
				names.append("ALPHA_BLEND")
			_:
				names.append("UNKNOWN_%s" % str(mode))

	return ", ".join(names)
```

---

# 🎮 Connecting It To The Grip Button

In your `player_controller.gd`, add an exported variable:

```gdscript
@export var ar_passthrough_controller: Node
```

Then in your right controller button function, add `grip_click`:

```gdscript
func _on_right_button_pressed(button_name: String) -> void:
	print("Right controller button pressed: '%s'" % button_name)

	match button_name:
		"ax_button":
			print("Right Hand")

		"trigger_click":
			print("Right trigger pressed")

		"grip_click":
			print("Right grip click -> Toggle AR passthrough")

			if ar_passthrough_controller != null and ar_passthrough_controller.has_method("toggle_passthrough"):
				ar_passthrough_controller.toggle_passthrough()
			else:
				print("ERROR: ar_passthrough_controller is not assigned or missing toggle_passthrough().")
```

Then in the inspector:

```text
PlayerController
→ ar_passthrough_controller
→ drag your ARPassthroughController node here
```

---

# ✅ Expected Working Logs

When you press `grip_click`, you want to see something like:

```text
Right controller button pressed: 'grip_click'
Right grip click -> Toggle AR passthrough
Trying to set passthrough to: true
Supported XR blend modes: OPAQUE, ALPHA_BLEND
Using ALPHA_BLEND passthrough mode.
AR passthrough ON locally.
Passthrough state is now: true
```

When you press it again:

```text
Right controller button pressed: 'grip_click'
Right grip click -> Toggle AR passthrough
Trying to set passthrough to: false
Supported XR blend modes: OPAQUE, ALPHA_BLEND
AR passthrough OFF locally.
Passthrough state is now: false
```

---

# 🧯 Troubleshooting

## Problem: Only `OPAQUE` is supported

Log:

```text
Supported XR blend modes: OPAQUE
ERROR: No AR passthrough blend mode is supported. Only VR/OPAQUE is available right now.
```

Meaning:

```text
The current running OpenXR runtime does not expose AR passthrough blend mode.
The game is still running as normal VR.
```

Possible causes:

- You are testing from the Windows editor instead of the Quest Android build.
- The Quest is still running an old build.
- The Android export was not rebuilt after changing settings.
- Meta passthrough is not enabled in Project Settings.
- Meta passthrough is not enabled in the Android Export preset.
- The OpenXR Vendors plugin is missing or not the correct version.
- The app is running through a desktop OpenXR runtime that only supports `OPAQUE`.

---

## Problem: Restarting Godot did not fix it

Restarting the editor helps sometimes, but it does not automatically update the app installed on Quest.

You still need to:

```text
Re-export
Reinstall
Run the new build on the headset
```

---

## Problem: The button works, but passthrough does not

If the log shows this:

```text
Right controller button pressed: 'grip_click'
Right grip click -> Toggle AR passthrough
```

then the input is working.

The remaining problem is the XR runtime/export/device setup.

---

# 🧪 Debug Checklist

Before testing passthrough, check these:

- [ ] Project Settings → Advanced Settings is ON.
- [ ] XR → OpenXR → Environment Blend Mode is set to `Alpha`.
- [ ] XR → OpenXR → Extensions → Meta → Passthrough is ON.
- [ ] Export → Android → Meta XR Features → Passthrough is `Optional` or `Required`.
- [ ] The project was saved after changing settings.
- [ ] The Android build was re-exported.
- [ ] The new build was installed on Quest.
- [ ] The old app was fully closed on Quest.
- [ ] Testing is happening inside the Quest headset, not only the PC editor.
- [ ] Debug output shows `ALPHA_BLEND` in supported blend modes.

---

# ⚠️ Current Limitations

- This setup is single-player only.
- It does not use server/client logic.
- It does not use RPC.
- It only changes passthrough on the current headset.
- Passthrough depends on the runtime reporting `ALPHA_BLEND` or `ADDITIVE`.
- If the runtime only reports `OPAQUE`, the code cannot force passthrough.
- This feature must be tested on the actual Quest Android build.

---

# 🚧 Next Steps

- Test with `Passthrough: Required` in the Android export preset.
- Rebuild and reinstall the app on the Quest.
- Confirm that debug output shows `ALPHA_BLEND`.
- After it works, change `Passthrough` back to `Optional` if needed.
- Add a small UI indicator showing whether the game is currently in VR mode or AR passthrough mode.
