---
title: "02 — add: click-to-move - ray pick ground, walk to MoveTarget"
date: 2026-09-27
sha: 6bdc2a9
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 02 — add: click-to-move - ray pick ground, walk to MoveTarget

`6bdc2a9` · 2026-09-27 · generated from git, not written by hand.

## Files changed

```
 src/main.rs | 65 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++-
 1 file changed, 64 insertions(+), 1 deletion(-)
```

## Diff

```diff
diff --git a/src/main.rs b/src/main.rs
index 506b0b2..ea91ef4 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -1,12 +1,19 @@
-use bevy::{ecs::system::command, prelude::*};
+use bevy::prelude::*;
 
 fn main() {
     App::new()
         .add_systems(Startup, setup)
         .add_plugins(DefaultPlugins)
+        .add_systems(Update, (move_player, click_to_point))
         .run();
 }
 
+#[derive(Component)]
+struct Player;
+
+#[derive(Component)]
+struct MoveTarget(Vec3);
+
 fn setup(
     mut commands: Commands,
     mut meshes: ResMut<Assets<Mesh>>,
@@ -31,5 +38,61 @@ fn setup(
         Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
         MeshMaterial3d(materials.add(Color::srgb_u8(124, 144, 255))),
         Transform::from_xyz(0.0, 0.5, 0.0),
+        Player,
     ));
 }
+
+const SPEED: f32 = 5.0;
+
+fn move_player(
+    mut commands: Commands,
+    mut query: Query<(Entity, &mut Transform, &MoveTarget)>,
+    time: Res<Time>,
+) {
+    for (entity, mut transform, target) in &mut query {
+        let mut to_target = target.0 - transform.translation;
+        to_target.y = 0.0;
+
+        let distance = to_target.length();
+        let step = SPEED * time.delta_secs();
+
+        if distance <= step {
+            transform.translation.x = target.0.x;
+            transform.translation.z = target.0.z;
+            commands.entity(entity).remove::<MoveTarget>();
+            continue;
+        }
+
+        transform.translation += to_target / distance * step;
+    }
+}
+
+fn click_to_point(
+    mouse: Res<ButtonInput<MouseButton>>,
+    camera: Single<(&Camera, &GlobalTransform)>,
+    window: Single<&Window>,
+    mut commands: Commands,
+    player: Single<Entity, With<Player>>,
+) {
+    if !mouse.just_pressed(MouseButton::Left) {
+        return;
+    }
+
+    let (camera, camera_transform) = *camera;
+
+    let Some(cursor) = window.cursor_position() else {
+        return; // Pointer is outside the window
+    };
+
+    let Ok(ray) = camera.viewport_to_world(camera_transform, cursor) else {
+        return; // camera coudn't build a ray
+    };
+
+    let Some(point) = ray.plane_intersection_point(Vec3::ZERO, InfinitePlane3d::new(Vec3::Y))
+    else {
+        return; // ray never meets the ground
+    };
+
+    commands.entity(*player).insert(MoveTarget(point));
+    info!("Clicked at {point:?}");
+}
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
