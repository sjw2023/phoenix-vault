---
title: "09 — add: dying message, enemy separation"
date: 2026-09-29
sha: 825b1bf
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 09 — add: dying message, enemy separation

`825b1bf` · 2026-09-29 · generated from git, not written by hand.

## Files changed

```
 src/combat.rs | 29 +++++++++-----------------
 src/enemy.rs  | 35 ++++++++++++++++++++++++++++++-
 src/main.rs   |  1 +
 src/player.rs | 35 ++++++++++++++++++++++++++++---
 src/ui.rs     | 66 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++--
 5 files changed, 141 insertions(+), 25 deletions(-)
```

## Diff

```diff
diff --git a/src/combat.rs b/src/combat.rs
index 0d4e5b1..27c9f57 100644
--- a/src/combat.rs
+++ b/src/combat.rs
@@ -1,9 +1,7 @@
 use bevy::prelude::*;
 
 use crate::enemy::Enemy;
-use crate::game_state::{AppState, DeathMode};
-use crate::movement::MoveTarget;
-use crate::player::PLAYER_SPAWN;
+use crate::game_state::AppState;
 use crate::player::Player;
 
 pub const ATTACK_RANGE: f32 = 1.5;
@@ -14,11 +12,16 @@ pub struct Health {
     pub current: f32,
     pub max: f32,
 }
+
 #[derive(Component)]
 pub struct Damage(pub f32);
+
 #[derive(Component)]
 pub struct AttackTimer(pub Timer);
 
+#[derive(Message)]
+pub struct PlayerDied;
+
 impl Plugin for CombatPlugin {
     fn build(&self, app: &mut App) {
         app.add_systems(
@@ -104,27 +107,15 @@ fn enemy_attack(
 
 fn death(
     mut commands: Commands,
-    mode: Res<DeathMode>,
-    query: Query<(Entity, &mut Health, &mut Transform, Option<&Player>)>,
-    mut next_state: ResMut<NextState<AppState>>,
+    mut died: MessageWriter<PlayerDied>,
+    query: Query<(Entity, &mut Health, Option<&Player>)>,
 ) {
-    for (entity, mut health, mut transform, player) in query {
+    for (entity, health, player) in &query {
         if health.current > 0.0 {
             continue;
         }
         if player.is_some() {
-            match *mode {
-                DeathMode::Softcore => {
-                    health.current = health.max;
-                    *transform = Transform::from_translation(PLAYER_SPAWN);
-                    commands.entity(entity).remove::<MoveTarget>();
-                    info!("respawned");
-                }
-                DeathMode::Retry | DeathMode::Hardcore => {
-                    next_state.set(AppState::GameOver);
-                    info!("You Died");
-                }
-            }
+            died.write(PlayerDied);
             continue;
         }
         commands.entity(entity).despawn();
diff --git a/src/enemy.rs b/src/enemy.rs
index 82830e0..dddbc5d 100644
--- a/src/enemy.rs
+++ b/src/enemy.rs
@@ -9,13 +9,19 @@ pub struct EnemyPlugin;
 impl Plugin for EnemyPlugin {
     fn build(&self, app: &mut App) {
         app.add_systems(OnEnter(AppState::Playing), enemy_spawn)
-            .add_systems(Update, enemy_chase.run_if(in_state(AppState::Playing)));
+            .add_systems(
+                Update,
+                (enemy_chase, enemy_separation).run_if(in_state(AppState::Playing)),
+            );
     }
 }
 
 #[derive(Component)]
 pub struct Enemy;
 
+const SEPARATION_STRENGTH: f32 = 1.2;
+const SEPARATION_RADIUS: f32 = 4.0;
+
 const ENEMY_SPAWNS: [Vec3; 5] = [
     Vec3::new(8.0, 0.5, -6.0),
     Vec3::new(-7.0, 0.5, -9.0),
@@ -67,3 +73,30 @@ fn enemy_chase(
             .insert(MoveTarget(player.translation));
     }
 }
+
+fn enemy_separation(mut enemies: Query<(Entity, &mut Transform), With<Enemy>>, time: Res<Time>) {
+    let positions: Vec<(Entity, Vec3)> = enemies
+        .iter()
+        .map(|(entity, transform)| (entity, transform.translation))
+        .collect();
+
+    for (entity, mut transform) in &mut enemies {
+        let mut push = Vec3::ZERO;
+        for (other_entity, other_position) in &positions {
+            if *other_entity == entity {
+                continue;
+            }
+            let mut away = transform.translation - other_position;
+            away.y = 0.0;
+            let distance = away.length();
+
+            if distance >= SEPARATION_RADIUS || distance == 0.0 {
+                continue;
+            }
+
+            push += away / distance * (SEPARATION_RADIUS - distance);
+        }
+
+        transform.translation += push * SEPARATION_STRENGTH * time.delta_secs();
+    }
+}
diff --git a/src/main.rs b/src/main.rs
index 270c37f..0ae06fa 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -11,6 +11,7 @@ mod world;
 fn main() {
     App::new()
         .add_plugins(DefaultPlugins)
+        .add_message::<combat::PlayerDied>()
         .add_plugins((
             camera::CameraPlugin,
             movement::MovementPlugin,
diff --git a/src/player.rs b/src/player.rs
index d6543df..db4ae64 100644
--- a/src/player.rs
+++ b/src/player.rs
@@ -1,5 +1,5 @@
-use crate::combat::{AttackTimer, Damage, Health};
-use crate::game_state::AppState;
+use crate::combat::{AttackTimer, Damage, Health, PlayerDied};
+use crate::game_state::{AppState, DeathMode};
 use crate::movement::{MoveTarget, Speed};
 use bevy::prelude::*;
 
@@ -10,7 +10,8 @@ impl Plugin for PlayerPlugin {
         app.add_systems(OnEnter(AppState::Playing), player_spawn)
             .add_systems(
                 Update,
-                (click_to_point, keyboard_move).run_if(in_state(AppState::Playing)),
+                (click_to_point, keyboard_move, handle_player_death)
+                    .run_if(in_state(AppState::Playing)),
             );
     }
 }
@@ -19,6 +20,34 @@ pub const PLAYER_SPAWN: Vec3 = Vec3::new(0.0, 0.5, 0.0);
 #[derive(Component)]
 pub struct Player;
 
+fn handle_player_death(
+    mut commands: Commands,
+    mut died: MessageReader<PlayerDied>,
+    mode: Res<DeathMode>,
+    mut next_state: ResMut<NextState<AppState>>,
+    player: Single<(Entity, &mut Health, &mut Transform), With<Player>>,
+) {
+    if died.is_empty() {
+        return;
+    }
+    died.clear();
+
+    let (entity, mut health, mut transform) = player.into_inner();
+
+    match *mode {
+        DeathMode::Softcore => {
+            health.current = health.max;
+            *transform = Transform::from_translation(PLAYER_SPAWN);
+            commands.entity(entity).remove::<MoveTarget>();
+            info!("Respawned");
+        }
+        DeathMode::Hardcore | DeathMode::Retry => {
+            next_state.set(AppState::GameOver);
+            info!("You Died")
+        }
+    }
+}
+
 fn player_spawn(
     mut commands: Commands,
     mut meshes: ResMut<Assets<Mesh>>,
diff --git a/src/ui.rs b/src/ui.rs
index b1ef136..ac1bb03 100644
--- a/src/ui.rs
+++ b/src/ui.rs
@@ -1,6 +1,8 @@
 use bevy::prelude::*;
 
-use crate::combat::Health;
+use crate::combat::{Health, PlayerDied};
+use crate::game_state::AppState;
+use crate::game_state::DeathMode;
 use crate::player::Player;
 
 pub struct UIPlugin;
@@ -8,13 +10,62 @@ pub struct UIPlugin;
 impl Plugin for UIPlugin {
     fn build(&self, app: &mut App) {
         app.add_systems(Startup, spawn_health_bar)
-            .add_systems(Update, update_health_bar);
+            .add_systems(Update, (update_health_bar, clear_health_bar))
+            .add_systems(OnEnter(AppState::GameOver), spawn_game_over)
+            .add_systems(OnExit(AppState::GameOver), despawn_game_over);
     }
 }
 
 #[derive(Component)]
 struct HealthBarFill;
 
+#[derive(Component)]
+struct GameOverScreen;
+
+fn spawn_game_over(mut commands: Commands, mode: Res<DeathMode>) {
+    let hint = match *mode {
+        DeathMode::Hardcore => "This run is over",
+        _ => "press R to try again",
+    };
+
+    commands.spawn((
+        Node {
+            position_type: PositionType::Absolute,
+            width: percent(100),
+            height: percent(100),
+            flex_direction: FlexDirection::Column,
+            align_items: AlignItems::Center,
+            justify_content: JustifyContent::Center,
+            ..default()
+        },
+        GameOverScreen,
+        children![
+            (
+                Text::new("YOU DIED"),
+                TextFont {
+                    font_size: FontSize::Px(72.0),
+                    ..default()
+                },
+                TextColor(Color::srgb(0.85, 0.15, 0.15)),
+            ),
+            (
+                Text::new(hint),
+                TextFont {
+                    font_size: FontSize::Px(24.0),
+                    ..default()
+                },
+                TextColor(Color::srgb(0.9, 0.9, 0.9)),
+            ),
+        ],
+    ));
+}
+
+fn despawn_game_over(mut commands: Commands, screen: Query<Entity, With<GameOverScreen>>) {
+    for entity in &screen {
+        commands.entity(entity).despawn();
+    }
+}
+
 fn spawn_health_bar(mut commands: Commands) {
     commands.spawn((
         Node {
@@ -45,3 +96,14 @@ fn update_health_bar(
     let fraction = (health.current / health.max).clamp(0.0, 1.0);
     fill.width = percent(fraction * 100.0);
 }
+
+fn clear_health_bar(
+    mut died: MessageReader<PlayerDied>,
+    mut fill: Single<&mut Node, With<HealthBarFill>>,
+) {
+    if died.is_empty() {
+        return;
+    }
+    died.clear();
+    fill.width = percent(0.0);
+}
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
