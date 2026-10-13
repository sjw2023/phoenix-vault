---
title: "03 — add: click-to-move - ray pick ground, walk to MoveTarget"
date: 2026-09-27
sha: 8f1a11e
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 03 — add: click-to-move - ray pick ground, walk to MoveTarget

`8f1a11e` · 2026-09-27 · generated from git, not written by hand.

## Files changed

```
 src/main.rs | 11 ++++++++++-
 1 file changed, 10 insertions(+), 1 deletion(-)
```

## Diff

```diff
diff --git a/src/main.rs b/src/main.rs
index ea91ef4..2c6105d 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -4,7 +4,7 @@ fn main() {
     App::new()
         .add_systems(Startup, setup)
         .add_plugins(DefaultPlugins)
-        .add_systems(Update, (move_player, click_to_point))
+        .add_systems(Update, (move_player, click_to_point, camera_follow))
         .run();
 }
 
@@ -67,6 +67,15 @@ fn move_player(
     }
 }
 
+const CAMERA_OFFSET: Vec3 = Vec3::new(0.0, 12.0, 12.0);
+
+fn camera_follow(
+    player: Single<&Transform, With<Player>>,
+    mut camera: Single<&mut Transform, With<Camera3d>>,
+) {
+    camera.translation = player.translation + CAMERA_OFFSET;
+    camera.look_at(player.translation, Vec3::Y);
+}
 fn click_to_point(
     mouse: Res<ButtonInput<MouseButton>>,
     camera: Single<(&Camera, &GlobalTransform)>,
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
