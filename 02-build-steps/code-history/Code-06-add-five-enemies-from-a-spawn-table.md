---
title: "06 — add : five enemies from a spawn table"
date: 2026-09-28
sha: 4690905
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 06 — add : five enemies from a spawn table

`4690905` · 2026-09-28 · generated from git, not written by hand.

## Files changed

```
 src/enemy.rs | 33 +++++++++++++++++++++++----------
 1 file changed, 23 insertions(+), 10 deletions(-)
```

## Diff

```diff
diff --git a/src/enemy.rs b/src/enemy.rs
index bec5899..51ec6a8 100644
--- a/src/enemy.rs
+++ b/src/enemy.rs
@@ -15,21 +15,34 @@ impl Plugin for EnemyPlugin {
 #[derive(Component)]
 pub struct Enemy;
 
+const ENEMY_SPAWNS: [Vec3; 5] = [
+    Vec3::new(8.0, 0.5, -6.0),
+    Vec3::new(-7.0, 0.5, -9.0),
+    Vec3::new(12.0, 0.5, 3.0),
+    Vec3::new(-10.0, 0.5, 5.0),
+    Vec3::new(2.0, 0.5, -14.0),
+];
+
 fn enemy_spawn(
     mut commands: Commands,
     mut meshes: ResMut<Assets<Mesh>>,
     mut materials: ResMut<Assets<StandardMaterial>>,
 ) {
-    commands.spawn((
-        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
-        MeshMaterial3d(materials.add(Color::srgb_u8(220, 80, 80))),
-        Transform::from_xyz(8.0, 0.5, -6.0),
-        Enemy,
-        Speed(3.0),
-        Health(60.0),
-        Damage(8.0),
-        AttackTimer(Timer::from_seconds(1.2, TimerMode::Repeating)),
-    ));
+    let mesh = meshes.add(Cuboid::new(1.0, 1.0, 1.0));
+    let material = materials.add(Color::srgb_u8(220, 80, 80));
+
+    for position in ENEMY_SPAWNS {
+        commands.spawn((
+            Mesh3d(mesh.clone()),
+            MeshMaterial3d(material.clone()),
+            Transform::from_translation(position),
+            Enemy,
+            Speed(3.0),
+            Health(60.0),
+            Damage(8.0),
+            AttackTimer(Timer::from_seconds(1.2, TimerMode::Repeating)),
+        ));
+    }
 }
 
 fn enemy_chase(
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
