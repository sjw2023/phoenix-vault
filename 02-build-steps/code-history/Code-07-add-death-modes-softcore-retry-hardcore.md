---
title: "07 — add: death modes - softcore, retry, hardcore"
date: 2026-09-29
sha: 3919745
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 07 — add: death modes - softcore, retry, hardcore

`3919745` · 2026-09-29 · generated from git, not written by hand.

## Files changed

```
 src/combat.rs     | 87 +++++++++++++++++++++++++++++++++++++++++++------------
 src/enemy.rs      | 10 +++++--
 src/game_state.rs | 53 +++++++++++++++++++++++++++++++++
 src/main.rs       |  2 ++
 src/movement.rs   |  3 +-
 src/player.rs     | 14 +++++++--
 6 files changed, 144 insertions(+), 25 deletions(-)
```

## Diff

```diff
diff --git a/src/combat.rs b/src/combat.rs
index fa682a8..0d4e5b1 100644
--- a/src/combat.rs
+++ b/src/combat.rs
@@ -1,28 +1,44 @@
 use bevy::prelude::*;
 
 use crate::enemy::Enemy;
+use crate::game_state::{AppState, DeathMode};
+use crate::movement::MoveTarget;
+use crate::player::PLAYER_SPAWN;
 use crate::player::Player;
 
 pub const ATTACK_RANGE: f32 = 1.5;
 pub struct CombatPlugin;
 
-impl Plugin for CombatPlugin {
-    fn build(&self, app: &mut App) {
-        app.add_systems(Update, (player_attack, enemy_attack, death));
-    }
-}
-
 #[derive(Component)]
-pub struct Health(pub f32);
+pub struct Health {
+    pub current: f32,
+    pub max: f32,
+}
 #[derive(Component)]
 pub struct Damage(pub f32);
 #[derive(Component)]
 pub struct AttackTimer(pub Timer);
 
+impl Plugin for CombatPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(
+            Update,
+            (player_attack, enemy_attack, death).run_if(in_state(AppState::Playing)),
+        )
+        .add_systems(OnExit(AppState::Playing), despawn_actors);
+    }
+}
+
+fn despawn_actors(mut commands: Commands, actors: Query<Entity, Or<(With<Player>, With<Enemy>)>>) {
+    for entity in actors {
+        commands.entity(entity).despawn();
+    }
+}
+
 fn player_attack(
     time: Res<Time>,
     player: Single<(&Transform, &Damage, &mut AttackTimer), With<Player>>,
-    mut enemies: Query<(&Transform, &mut Health), With<Enemy>>,
+    mut enemies: Query<(Entity, &Transform, &mut Health), With<Enemy>>,
 ) {
     let (player_transform, damage, mut timer) = player.into_inner();
     timer.0.tick(time.delta());
@@ -31,17 +47,37 @@ fn player_attack(
         return;
     }
 
-    for (enemy_transform, mut health) in &mut enemies {
+    let mut nearest: Option<(f32, Entity)> = None;
+
+    for (entity, enemy_transform, _) in &mut enemies {
         let mut to_enemy = enemy_transform.translation - player_transform.translation;
         to_enemy.y = 0.0;
 
+        let distance = to_enemy.length();
+
         if to_enemy.length() > ATTACK_RANGE {
             continue;
         }
 
-        health.0 -= damage.0;
-        info!("hit enemy -> {} HP", health.0);
+        let closer = match nearest {
+            Some((best, _)) => distance < best,
+            None => true,
+        };
+
+        if closer {
+            nearest = Some((distance, entity));
+        }
     }
+    let Some((_, target)) = nearest else {
+        return;
+    };
+
+    let Ok((_, _, mut health)) = enemies.get_mut(target) else {
+        return;
+    };
+
+    health.current -= damage.0;
+    info!("hit enemy -> {} HP", health.current);
 }
 
 fn enemy_attack(
@@ -61,21 +97,36 @@ fn enemy_attack(
             continue;
         }
 
-        health.0 -= damage.0;
-        info!("enemy hits you -> {} HP", health.0);
+        health.current -= damage.0;
+        info!("enemy hits you -> {} HP", health.current);
     }
 }
 
-fn death(mut commands: Commands, query: Query<(Entity, &Health, Option<&Player>)>) {
-    for (entity, health, player) in query {
-        if health.0 > 0.0 {
+fn death(
+    mut commands: Commands,
+    mode: Res<DeathMode>,
+    query: Query<(Entity, &mut Health, &mut Transform, Option<&Player>)>,
+    mut next_state: ResMut<NextState<AppState>>,
+) {
+    for (entity, mut health, mut transform, player) in query {
+        if health.current > 0.0 {
             continue;
         }
         if player.is_some() {
-            info!("You Died");
+            match *mode {
+                DeathMode::Softcore => {
+                    health.current = health.max;
+                    *transform = Transform::from_translation(PLAYER_SPAWN);
+                    commands.entity(entity).remove::<MoveTarget>();
+                    info!("respawned");
+                }
+                DeathMode::Retry | DeathMode::Hardcore => {
+                    next_state.set(AppState::GameOver);
+                    info!("You Died");
+                }
+            }
             continue;
         }
-
         commands.entity(entity).despawn();
     }
 }
diff --git a/src/enemy.rs b/src/enemy.rs
index 51ec6a8..82830e0 100644
--- a/src/enemy.rs
+++ b/src/enemy.rs
@@ -1,4 +1,5 @@
 use crate::combat::{ATTACK_RANGE, AttackTimer, Damage, Health};
+use crate::game_state::AppState;
 use crate::movement::{MoveTarget, Speed};
 use crate::player::Player;
 use bevy::prelude::*;
@@ -7,8 +8,8 @@ pub struct EnemyPlugin;
 
 impl Plugin for EnemyPlugin {
     fn build(&self, app: &mut App) {
-        app.add_systems(Startup, enemy_spawn)
-            .add_systems(Update, enemy_chase);
+        app.add_systems(OnEnter(AppState::Playing), enemy_spawn)
+            .add_systems(Update, enemy_chase.run_if(in_state(AppState::Playing)));
     }
 }
 
@@ -38,7 +39,10 @@ fn enemy_spawn(
             Transform::from_translation(position),
             Enemy,
             Speed(3.0),
-            Health(60.0),
+            Health {
+                current: 60.0,
+                max: 60.0,
+            },
             Damage(8.0),
             AttackTimer(Timer::from_seconds(1.2, TimerMode::Repeating)),
         ));
diff --git a/src/game_state.rs b/src/game_state.rs
new file mode 100644
index 0000000..5d6f98b
--- /dev/null
+++ b/src/game_state.rs
@@ -0,0 +1,53 @@
+use bevy::prelude::*;
+
+pub struct GameStatePlugin;
+
+impl Plugin for GameStatePlugin {
+    fn build(&self, app: &mut App) {
+        app.init_state::<AppState>()
+            .init_resource::<DeathMode>()
+            .add_systems(Update, select_death_mode)
+            .add_systems(Update, restart.run_if(in_state(AppState::GameOver)));
+    }
+}
+
+#[derive(Resource, Clone, Copy, Debug, Default, Eq, PartialEq)]
+pub enum DeathMode {
+    Softcore,
+    #[default]
+    Retry,
+    Hardcore,
+}
+
+#[derive(Debug, Clone, Copy, Default, Eq, PartialEq, Hash, States)]
+pub enum AppState {
+    #[default]
+    Playing,
+    GameOver,
+}
+
+fn restart(
+    keys: Res<ButtonInput<KeyCode>>,
+    mode: Res<DeathMode>,
+    mut next_state: ResMut<NextState<AppState>>,
+) {
+    if *mode == DeathMode::Hardcore {
+        return;
+    }
+    if keys.just_pressed(KeyCode::KeyR) {
+        next_state.set(AppState::Playing);
+    }
+}
+
+fn select_death_mode(keys: Res<ButtonInput<KeyCode>>, mut mode: ResMut<DeathMode>) {
+    if keys.just_pressed(KeyCode::Digit1) {
+        *mode = DeathMode::Softcore;
+    } else if keys.just_pressed(KeyCode::Digit2) {
+        *mode = DeathMode::Retry;
+    } else if keys.just_pressed(KeyCode::Digit3) {
+        *mode = DeathMode::Hardcore;
+    } else {
+        return;
+    }
+    info!("Death mode: {:?}", *mode);
+}
diff --git a/src/main.rs b/src/main.rs
index 229e986..3da4774 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -2,6 +2,7 @@ use bevy::prelude::*;
 mod camera;
 mod combat;
 mod enemy;
+mod game_state;
 mod movement;
 mod player;
 mod world;
@@ -16,6 +17,7 @@ fn main() {
             world::WorldPlugin,
             combat::CombatPlugin,
             enemy::EnemyPlugin,
+            game_state::GameStatePlugin,
         ))
         .run();
 }
diff --git a/src/movement.rs b/src/movement.rs
index a6384b6..f72fa42 100644
--- a/src/movement.rs
+++ b/src/movement.rs
@@ -1,10 +1,11 @@
+use crate::game_state::AppState;
 use bevy::prelude::*;
 
 pub struct MovementPlugin;
 
 impl Plugin for MovementPlugin {
     fn build(&self, app: &mut App) {
-        app.add_systems(Update, move_to_target);
+        app.add_systems(Update, move_to_target.run_if(in_state(AppState::Playing)));
     }
 }
 
diff --git a/src/player.rs b/src/player.rs
index df9eeef..d6543df 100644
--- a/src/player.rs
+++ b/src/player.rs
@@ -1,4 +1,5 @@
 use crate::combat::{AttackTimer, Damage, Health};
+use crate::game_state::AppState;
 use crate::movement::{MoveTarget, Speed};
 use bevy::prelude::*;
 
@@ -6,11 +7,15 @@ pub struct PlayerPlugin;
 
 impl Plugin for PlayerPlugin {
     fn build(&self, app: &mut App) {
-        app.add_systems(Startup, player_spawn)
-            .add_systems(Update, (click_to_point, keyboard_move));
+        app.add_systems(OnEnter(AppState::Playing), player_spawn)
+            .add_systems(
+                Update,
+                (click_to_point, keyboard_move).run_if(in_state(AppState::Playing)),
+            );
     }
 }
 
+pub const PLAYER_SPAWN: Vec3 = Vec3::new(0.0, 0.5, 0.0);
 #[derive(Component)]
 pub struct Player;
 
@@ -30,7 +35,10 @@ fn player_spawn(
             Transform::from_xyz(0.0, 0.0, -0.7),
         )],
         Speed(5.0),
-        Health(100.0),
+        Health {
+            current: 100.0,
+            max: 100.0,
+        },
         Damage(25.0),
         AttackTimer(Timer::from_seconds(0.8, TimerMode::Repeating)),
     ));
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
