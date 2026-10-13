---
title: "10 — add: knockback on hit"
date: 2026-09-29
sha: 54b07c9
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 10 — add: knockback on hit

`54b07c9` · 2026-09-29 · generated from git, not written by hand.

## Files changed

```
 src/combat.rs |  23 ++++++++++---
 src/enemy.rs  | 106 +++++++++++++++++++++++++++++++++++++++++++++++++++++-----
 src/main.rs   |   1 +
 3 files changed, 117 insertions(+), 13 deletions(-)
```

## Diff

```diff
diff --git a/src/combat.rs b/src/combat.rs
index 27c9f57..3d6850e 100644
--- a/src/combat.rs
+++ b/src/combat.rs
@@ -22,6 +22,12 @@ pub struct AttackTimer(pub Timer);
 #[derive(Message)]
 pub struct PlayerDied;
 
+#[derive(Message)]
+pub struct Hit {
+    pub target: Entity,
+    pub direction: Dir3,
+}
+
 impl Plugin for CombatPlugin {
     fn build(&self, app: &mut App) {
         app.add_systems(
@@ -42,6 +48,7 @@ fn player_attack(
     time: Res<Time>,
     player: Single<(&Transform, &Damage, &mut AttackTimer), With<Player>>,
     mut enemies: Query<(Entity, &Transform, &mut Health), With<Enemy>>,
+    mut hits: MessageWriter<Hit>,
 ) {
     let (player_transform, damage, mut timer) = player.into_inner();
     timer.0.tick(time.delta());
@@ -53,12 +60,12 @@ fn player_attack(
     let mut nearest: Option<(f32, Entity)> = None;
 
     for (entity, enemy_transform, _) in &mut enemies {
-        let mut to_enemy = enemy_transform.translation - player_transform.translation;
-        to_enemy.y = 0.0;
+        let mut away = enemy_transform.translation - player_transform.translation;
+        away.y = 0.0;
 
-        let distance = to_enemy.length();
+        let distance = away.length();
 
-        if to_enemy.length() > ATTACK_RANGE {
+        if away.length() > ATTACK_RANGE {
             continue;
         }
 
@@ -75,11 +82,17 @@ fn player_attack(
         return;
     };
 
-    let Ok((_, _, mut health)) = enemies.get_mut(target) else {
+    let Ok((_, enemy_transform, mut health)) = enemies.get_mut(target) else {
         return;
     };
 
+    let mut away = enemy_transform.translation - player_transform.translation;
+    away.y = 0.0;
     health.current -= damage.0;
+    hits.write(Hit {
+        target,
+        direction: Dir3::new(away).unwrap_or(Dir3::NEG_Z),
+    });
     info!("hit enemy -> {} HP", health.current);
 }
 
diff --git a/src/enemy.rs b/src/enemy.rs
index dddbc5d..cc3f2f6 100644
--- a/src/enemy.rs
+++ b/src/enemy.rs
@@ -1,4 +1,4 @@
-use crate::combat::{ATTACK_RANGE, AttackTimer, Damage, Health};
+use crate::combat::{ATTACK_RANGE, AttackTimer, Damage, Health, Hit};
 use crate::game_state::AppState;
 use crate::movement::{MoveTarget, Speed};
 use crate::player::Player;
@@ -6,22 +6,56 @@ use bevy::prelude::*;
 
 pub struct EnemyPlugin;
 
+impl FromWorld for EnemyMaterials {
+    fn from_world(world: &mut World) -> Self {
+        let mut materials = world.resource_mut::<Assets<StandardMaterial>>();
+
+        Self {
+            normal: materials.add(Color::srgb_u8(220, 80, 80)),
+            hit: materials.add(Color::WHITE),
+        }
+    }
+}
+
 impl Plugin for EnemyPlugin {
     fn build(&self, app: &mut App) {
         app.add_systems(OnEnter(AppState::Playing), enemy_spawn)
+            .init_resource::<EnemyMaterials>()
             .add_systems(
                 Update,
-                (enemy_chase, enemy_separation).run_if(in_state(AppState::Playing)),
+                (
+                    enemy_chase,
+                    enemy_separation,
+                    flash_on_hit,
+                    end_hit_flash,
+                    apply_knockback,
+                    knockback_on_hit,
+                )
+                    .run_if(in_state(AppState::Playing)),
             );
     }
 }
 
 #[derive(Component)]
 pub struct Enemy;
+#[derive(Resource)]
+struct EnemyMaterials {
+    normal: Handle<StandardMaterial>,
+    hit: Handle<StandardMaterial>,
+}
+#[derive(Component)]
+struct HitFlash(Timer);
+#[derive(Component)]
+struct Knockback {
+    direction: Dir3,
+    timer: Timer,
+}
 
-const SEPARATION_STRENGTH: f32 = 1.2;
-const SEPARATION_RADIUS: f32 = 4.0;
-
+const SEPARATION_STRENGTH: f32 = 4.0;
+const SEPARATION_RADIUS: f32 = 1.2;
+const FLASH_SECONDS: f32 = 0.12;
+const KNOCKBACK_SECONDS: f32 = 0.10;
+const KNOCKBACK_SPEED: f32 = 12.0;
 const ENEMY_SPAWNS: [Vec3; 5] = [
     Vec3::new(8.0, 0.5, -6.0),
     Vec3::new(-7.0, 0.5, -9.0),
@@ -30,18 +64,74 @@ const ENEMY_SPAWNS: [Vec3; 5] = [
     Vec3::new(2.0, 0.5, -14.0),
 ];
 
+fn knockback_on_hit(mut commands: Commands, mut hits: MessageReader<Hit>) {
+    for hit in hits.read() {
+        commands.entity(hit.target).insert(Knockback {
+            direction: hit.direction,
+            timer: Timer::from_seconds(KNOCKBACK_SECONDS, TimerMode::Once),
+        });
+    }
+}
+
+fn apply_knockback(
+    mut commands: Commands,
+    time: Res<Time>,
+    mut knocked: Query<(Entity, &mut Transform, &mut Knockback)>,
+) {
+    for (entity, mut transform, mut knockback) in &mut knocked {
+        if knockback.timer.tick(time.delta()).just_finished() {
+            commands.entity(entity).remove::<Knockback>();
+            continue;
+        }
+        transform.translation += knockback.direction * KNOCKBACK_SPEED * time.delta_secs();
+    }
+}
+
+fn flash_on_hit(
+    mut commands: Commands,
+    mut hits: MessageReader<Hit>,
+    materials: Res<EnemyMaterials>,
+    mut enemies: Query<&mut MeshMaterial3d<StandardMaterial>, With<Enemy>>,
+) {
+    for hit in hits.read() {
+        let Ok(mut material) = enemies.get_mut(hit.target) else {
+            continue;
+        };
+        material.0 = materials.hit.clone();
+        commands
+            .entity(hit.target)
+            .insert(HitFlash(Timer::from_seconds(
+                FLASH_SECONDS,
+                TimerMode::Once,
+            )));
+    }
+}
+
+fn end_hit_flash(
+    mut commands: Commands,
+    time: Res<Time>,
+    materials: Res<EnemyMaterials>,
+    mut flashing: Query<(Entity, &mut HitFlash, &mut MeshMaterial3d<StandardMaterial>)>,
+) {
+    for (entity, mut flash, mut material) in &mut flashing {
+        if flash.0.tick(time.delta()).just_finished() {
+            material.0 = materials.normal.clone();
+            commands.entity(entity).remove::<HitFlash>();
+        }
+    }
+}
+
 fn enemy_spawn(
     mut commands: Commands,
     mut meshes: ResMut<Assets<Mesh>>,
-    mut materials: ResMut<Assets<StandardMaterial>>,
+    materials: Res<EnemyMaterials>,
 ) {
     let mesh = meshes.add(Cuboid::new(1.0, 1.0, 1.0));
-    let material = materials.add(Color::srgb_u8(220, 80, 80));
 
     for position in ENEMY_SPAWNS {
         commands.spawn((
             Mesh3d(mesh.clone()),
-            MeshMaterial3d(material.clone()),
+            MeshMaterial3d(materials.normal.clone()),
             Transform::from_translation(position),
             Enemy,
             Speed(3.0),
diff --git a/src/main.rs b/src/main.rs
index 0ae06fa..1bd99d9 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -12,6 +12,7 @@ fn main() {
     App::new()
         .add_plugins(DefaultPlugins)
         .add_message::<combat::PlayerDied>()
+        .add_message::<combat::Hit>()
         .add_plugins((
             camera::CameraPlugin,
             movement::MovementPlugin,
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
