# CORE Artist Tools

> **LiLo-Labs fork** of [NVIDIA/simready-blender-add-on](https://github.com/NVIDIA/simready-blender-add-on)
> (based on upstream `cec4afc`, v0.4.0). It fixes four export bugs found on Blender 5.2.3. See
> [Changes in this fork](#changes-in-this-fork).

## Changes in this fork

Tested on **Blender 5.2.3 LTS (Windows)** with its bundled OpenUSD, and against **Newton 1.7.0** and the OpenUSD
`usdPhysicsValidators`. The test is `tests/t18_simready_fork.py` in our sprite-physics project. It authors a well
(frame, drum hinge with an angular drive, free bucket, rope; optionally a bail on a breakable joint) and a two-gear
winch with the add-on's own operators. It exports through `SimReady_USDHook`, validates the result, and loads and
simulates it in Newton against hand-written USD. Upstream fails on all four points below. The fork passes all four.

Each change is marked `[LiLo-Labs fork]` in the code. Nothing else in the add-on is touched.

| # | Bug in upstream | Fix | Files |
|---|---|---|---|
| 1 | **No joint can reach the world.** `apply_joint_settings` always needs `body_0` and `body_1`, so a hinge has to be pinned to a dummy body. The hook then gives that dummy a `RigidBodyAPI`, so nothing is anchored and Newton refuses to build the model. | A `body_0` (or `body_1`) whose Blender object is a **passive rigid body** (`Physics > Rigid Body > Type: Passive`), or has the custom property `simready:world_anchor = True`, is written as the **world**. That side's body relationship is left empty (UsdPhysics: world frame), `localPos/localRot` are moved into world space, and the object stays a static collider (its `RigidBodyAPI` is removed). | `ot_simready_usdhook.py` (`_lilo_post_joint_pass`, `_is_world_anchor`, `_to_world`) |
| 2 | **Angular drive targets are about 3,300x too small.** `drive_angular_target_velocity/position` are `unit="ROTATION"` properties, so Blender stores **radians** (the panel only displays degrees). The hook then applied `math.radians()` a second time, so -90 deg/s was written as -0.0274 deg/s (a factor of (180/pi)^2 = 3283). | Convert the stored radians to USD **degrees** once (`math.degrees`). Damping and stiffness pass through unchanged. They are now documented as USD per-degree units (enter SI gain x pi/180). | `ot_simready_usdhook.py` (`set_joint_attributes`), `physics_properties.py` (tooltips) |
| 3 | **Gear (and rack-and-pinion) joints crash the export.** Stock OpenUSD has no `UsdPhysics.GearJoint` / `RackPinionJoint`, so `AttributeError` aborts the whole USD export. Separately, the operator wrote the gear onto `body_1`'s reference empty, **overwriting that body's own revolute joint** (the gear's hinge1), and `gear_hinge_1` came back as `None`. | The operator now writes a coupling joint onto **its own** reference empty (`<body>Gear_joint_NN`), and the follower's hinge survives. The hook uses `UsdPhysics.GearJoint` if a future OpenUSD has it. Otherwise it authors a **`PhysxPhysicsGearJoint`** prim (PhysxSchema type: `hinge0`, `hinge1`, `gearRatio`; read by PhysX/Isaac) and adds **`NewtonMimicAPI`** (newton-usd-schemas) on the hinge1 joint, with `newton:mimicJoint -> hinge0`, `coef0 = 0`, `coef1 = -gearRatio`. Convention: q1 = -gearRatio * q0 for parallel, same-direction hinge axes, as for PhysX `PxGearJoint`, where gearRatio = n0/n1. Rack-and-pinion is written as `PhysxPhysicsRackAndPinionJoint` (PhysX only; Newton cannot couple an angular joint to a linear one). | `physics_operators.py` (`_apply_coupling_joint`), `ot_simready_usdhook.py` (`_define_coupling_joint`, `_author_couplings`) |
| 4 | **Newton cannot build the exported model.** (a) Joint `body0/body1` target the body's collider *mesh* (`<obj>/<mesh>`), not the prim carrying `RigidBodyAPI`. The OpenUSD validator rejects this (`JointNoEnabledRigidBody`), and Newton's articulation parser cannot resolve it. (b) No `PhysicsArticulationRootAPI` is ever written, so a floating assembly (e.g. bucket + bail on a fixed joint) fails Newton's `finalize()` with "joint(s) not belonging to any articulation". | (a) Retarget each body relationship to its rigid-body ancestor, moving the joint frame with it. (b) Breakable joints (finite `breakForce`/`breakTorque`) get `physics:excludeFromArticulation = true`: PhysX ignores breakForce on articulation joints, and a loop-closing joint is what Newton's Steel solver needs to measure a break. Each joint-connected group with no joint to the world gets `ArticulationRootAPI` on its root body (a floating base). Groups that reach the world are left alone. | `ot_simready_usdhook.py` (`_rebase_frame`, `_author_articulation_roots`) |

The post-joint pass never raises into the export. If it fails, it prints an error and the upstream output is kept.

**Results** (Blender 5.2.3, Newton 1.7.0):
- `usdPhysicsValidators` and the NVIDIA physics validators report 0 errors on all three exports. Upstream had a joint error on every joint.
- Newton `finalize()` passes for the well, well+bail and winch.
- Drive read back as -4.0 rad/s with kd 500, identical to our hand-written well.
- The well winds the rope at 0.0652 m per drum radian, against 0.0653 for the hand-written asset in the same solver.
- The winch turns at a -0.500 gear ratio.

**Known limits (not changed):**
- Colliders are always `convexDecomposition` meshes, never primitives.
- Mass comes from SimReady's material/fill model unless `pxr:usd:physics_mass` is set.
- No PhysicsScene, rope or deformable authoring.
- `gear_ratio` is capped at 1.0 in the panel.
- Newton (1.7 and main) also rejects a multi-level joint chain *under* a world joint when no articulation root is authored. The fork does not add one there.

**Installing on Blender 5.2:** copy `CORE_ArtistTools` and `CORE_ArtistTools_Resources` into
`%APPDATA%\Blender Foundation\Blender\5.2\scripts\addons`. Enabling the add-on runs its dependency installer
(`pip install --user` of torch, torchvision, transformers, PySide6, ... into `%APPDATA%\Python`), as upstream does.

## Introduction

CORE Artist Tools is a collection of Blender add-ons for creating SimReady Assets, tailored for Blender 5.1.

## Getting Started

For all docs to get you started, please read the [SimReady Blender Addon Landing Page](docs/simready-blender-addon-landing.md).


### Prerequisites

- Blender 5.1 installed on your machine.

### Installation

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd simready-blender-addons
   ```

2. **Build the add-on package**

   ```bash
   python CORE_SysUtils/package/make_core_zip.py
   ```

   Output: `SimReady_Blender_CORE_<version>.zip` in the repo root (version is read from [`VERSION.md`](VERSION.md)).

3.  **Alternative - Download addon directly**

      - [Latest release](https://github.com/NVIDIA/simready-blender-add-on/releases/tag/Latest)

4. **Install in Blender**

   - Open Blender → `Edit` → `Preferences` → `Install...`
   - Select `SimReady_Blender_CORE_<version>.zip` from the repo root.

## Project structure

```
simready-blender-addons/
├── CORE_ArtistTools/             # Main Blender add-on 
├── CORE_ArtistTools_Resources/   # Companion resources 
├── CORE_SysUtils/                # Build/utility scripts used by repo
├── Docs/                         # e2e User Guides, landing doc
```

## License

This project is licensed under the **Apache License 2.0** — see the [LICENSE](LICENSE) file for details.

```text
SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
```
