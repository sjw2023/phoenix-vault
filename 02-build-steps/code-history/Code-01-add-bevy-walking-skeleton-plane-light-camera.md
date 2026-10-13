---
title: "01 — add:bevy walking skeleton - plane, light, camera, cube"
date: 2026-09-25
sha: e900ed7
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 01 — add:bevy walking skeleton - plane, light, camera, cube

`e900ed7` · 2026-09-25 · generated from git, not written by hand.

## Files changed

```
 Cargo.toml  | 14 ++++++++++++++
 src/main.rs | 35 +++++++++++++++++++++++++++++++++++
 2 files changed, 49 insertions(+)
```

## Diff

```diff
diff --git a/Cargo.toml b/Cargo.toml
new file mode 100644
index 0000000..9228f7b
--- /dev/null
+++ b/Cargo.toml
@@ -0,0 +1,14 @@
+[package]
+name = "phoenix"
+version = "0.1.0"
+edition = "2024"
+
+[dependencies]
+bevy = "0.19.1"
+
+[profile.dev]
+opt-level = 1
+
+[profile.dev.package."*"]
+opt-level = 3
+
diff --git a/src/main.rs b/src/main.rs
new file mode 100644
index 0000000..506b0b2
--- /dev/null
+++ b/src/main.rs
@@ -0,0 +1,35 @@
+use bevy::{ecs::system::command, prelude::*};
+
+fn main() {
+    App::new()
+        .add_systems(Startup, setup)
+        .add_plugins(DefaultPlugins)
+        .run();
+}
+
+fn setup(
+    mut commands: Commands,
+    mut meshes: ResMut<Assets<Mesh>>,
+    mut materials: ResMut<Assets<StandardMaterial>>,
+) {
+    commands.spawn((
+        Mesh3d(meshes.add(Plane3d::default().mesh().size(50.0, 50.0))),
+        MeshMaterial3d(materials.add(Color::WHITE)),
+    ));
+    commands.spawn((
+        PointLight {
+            shadow_maps_enabled: true,
+            ..default()
+        },
+        Transform::from_xyz(8.0, 16.0, 8.0),
+    ));
+    commands.spawn((
+        Camera3d::default(),
+        Transform::from_xyz(0.0, 7.0, 14.0).looking_at(Vec3::new(0.0, 1.0, 0.0), Vec3::Y),
+    ));
+    commands.spawn((
+        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
+        MeshMaterial3d(materials.add(Color::srgb_u8(124, 144, 255))),
+        Transform::from_xyz(0.0, 0.5, 0.0),
+    ));
+}
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
