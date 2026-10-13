---
title: "08 — add: player health bar"
date: 2026-09-29
sha: ed17b02
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 08 — add: player health bar

`ed17b02` · 2026-09-29 · generated from git, not written by hand.

## Files changed

```
 src/main.rs |  2 ++
 src/ui.rs   | 47 +++++++++++++++++++++++++++++++++++++++++++++++
 2 files changed, 49 insertions(+)
```

## Diff

```diff
diff --git a/src/main.rs b/src/main.rs
index 3da4774..270c37f 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -5,6 +5,7 @@ mod enemy;
 mod game_state;
 mod movement;
 mod player;
+mod ui;
 mod world;
 
 fn main() {
@@ -18,6 +19,7 @@ fn main() {
             combat::CombatPlugin,
             enemy::EnemyPlugin,
             game_state::GameStatePlugin,
+            ui::UIPlugin,
         ))
         .run();
 }
diff --git a/src/ui.rs b/src/ui.rs
new file mode 100644
index 0000000..b1ef136
--- /dev/null
+++ b/src/ui.rs
@@ -0,0 +1,47 @@
+use bevy::prelude::*;
+
+use crate::combat::Health;
+use crate::player::Player;
+
+pub struct UIPlugin;
+
+impl Plugin for UIPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Startup, spawn_health_bar)
+            .add_systems(Update, update_health_bar);
+    }
+}
+
+#[derive(Component)]
+struct HealthBarFill;
+
+fn spawn_health_bar(mut commands: Commands) {
+    commands.spawn((
+        Node {
+            position_type: PositionType::Absolute,
+            bottom: px(24),
+            left: px(24),
+            width: px(320),
+            height: px(28),
+            ..default()
+        },
+        BackgroundColor(Color::srgb(0.12, 0.12, 0.12)),
+        children![(
+            Node {
+                width: percent(100),
+                height: percent(100),
+                ..default()
+            },
+            BackgroundColor(Color::srgb(0.8, 0.2, 0.2)),
+            HealthBarFill,
+        )],
+    ));
+}
+
+fn update_health_bar(
+    health: Single<&Health, With<Player>>,
+    mut fill: Single<&mut Node, With<HealthBarFill>>,
+) {
+    let fraction = (health.current / health.max).clamp(0.0, 1.0);
+    fill.width = percent(fraction * 100.0);
+}
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
