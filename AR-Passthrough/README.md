# 🥽 Single Player AR Passthrough Setup

### 🎮 Built with Godot 4.6.2 for Meta Quest 3

This document explains a **simple single-player AR passthrough setup** for the VR project.

The goal is simple:

> The player presses `grip_click`, and the game switches between normal VR mode and AR passthrough mode.



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



---

## 🎮 Input Action

The action name from the OpenXR Action Map is:

```text
grip_click
```

---

# 🧱 Project Settings

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
Environment Blend Mode: Alpha or Addictive (not Opaque)
```

<img src="images/Project Settings2.jpg" alt="Description" width="1300">



Then in the same section go to:

```text
→ Meta
```

Enable:

```text
Passthrough: On
```
<img src="images/Project Settings1.png" alt="Description" width="1300">

---

# 📦 Android Export Settings

Go to:

```text
Project
→ Export
→ Android Preset
→ Options
	→ XR Features
```
Set:
```text
Enable Meta Plugin: On
```
<img src="images/Export Settings2.jpg" alt="Description" width="720">

----

After that go to : 

```text
In the Same seciton:
→ Options
	→ Meta XR Features
```
<img src="images/Export Settings1.png" alt="Description" width="720">


Set:

```text
Passthrough: Optional
Boundary Mode: Enabled
```

For debugging only, you can temporarily set:

```text
Passthrough: Required
```

Use `Required` only to test if the Android build is correctly requesting passthrough support.


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



# 🧪 Debug Checklist

Before testing passthrough, check these:

- [ ] Project Settings → Advanced Settings is ON.
- [ ] XR → OpenXR → Environment Blend Mode is set to `Alpha`.
- [ ] XR → OpenXR → Extensions → Meta → Passthrough is ON.
- [ ] Export → Android → Meta XR Features → Passthrough is `Optional` or `Required`.
