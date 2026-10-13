---
title: "04 — add combat - health, damage, attack timers, death"
date: 2026-09-27
sha: 59dbea4
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 04 — add combat - health, damage, attack timers, death

`59dbea4` · 2026-09-27 · generated from git, not written by hand.

## Files changed

```
 src/main.rs | 140 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++---
 1 file changed, 134 insertions(+), 6 deletions(-)
```

## Diff

```diff
diff --git a/src/main.rs b/src/main.rs
index 2c6105d..4adacb7 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -4,7 +4,18 @@ fn main() {
     App::new()
         .add_systems(Startup, setup)
         .add_plugins(DefaultPlugins)
-        .add_systems(Update, (move_player, click_to_point, camera_follow))
+        .add_systems(
+            Update,
+            (
+                move_to_target,
+                click_to_point,
+                camera_follow,
+                enemy_chase,
+                player_attack,
+                enemy_attack,
+                death,
+            ),
+        )
         .run();
 }
 
@@ -39,22 +50,103 @@ fn setup(
         MeshMaterial3d(materials.add(Color::srgb_u8(124, 144, 255))),
         Transform::from_xyz(0.0, 0.5, 0.0),
         Player,
+        children![(
+            Mesh3d(meshes.add(Cuboid::new(0.3, 0.3, 0.6))),
+            MeshMaterial3d(materials.add(Color::srgb_u8(255, 200, 80))),
+            Transform::from_xyz(0.0, 0.0, -0.7),
+        )],
+        Speed(5.0),
+        Health(100.0),
+        Damage(25.0),
+        AttackTimer(Timer::from_seconds(0.8, TimerMode::Repeating)),
+    ));
+
+    commands.spawn((
+        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
+        MeshMaterial3d(materials.add(Color::srgb_u8(220, 80, 80))),
+        Transform::from_xyz(8.0, 0.5, -6.0),
+        Enemy,
+        Speed(3.0),
+        Health(60.0),
+        Damage(8.0),
+        AttackTimer(Timer::from_seconds(1.2, TimerMode::Repeating)),
     ));
 }
 
 const SPEED: f32 = 5.0;
 
-fn move_player(
+fn player_attack(
+    time: Res<Time>,
+    player: Single<(&Transform, &Damage, &mut AttackTimer), With<Player>>,
+    mut enemies: Query<(&Transform, &mut Health), With<Enemy>>,
+) {
+    let (player_transform, damage, mut timer) = player.into_inner();
+    timer.0.tick(time.delta());
+
+    if !timer.0.just_finished() {
+        return;
+    }
+
+    for (enemy_transform, mut health) in &mut enemies {
+        let mut to_enemy = enemy_transform.translation - player_transform.translation;
+        to_enemy.y = 0.0;
+
+        if to_enemy.length() > ATTACK_RANGE {
+            continue;
+        }
+
+        health.0 -= damage.0;
+        info!("hit enemy -> {} HP", health.0);
+    }
+}
+
+fn enemy_attack(
+    time: Res<Time>,
+    player: Single<(&Transform, &mut Health), With<Player>>,
+    mut enemies: Query<(&Transform, &Damage, &mut AttackTimer), With<Enemy>>,
+) {
+    let (player_transform, mut health) = player.into_inner();
+
+    for (enemy_transform, damage, mut timer) in &mut enemies {
+        timer.0.tick(time.delta());
+
+        let mut to_player = player_transform.translation - enemy_transform.translation;
+        to_player.y = 0.0;
+
+        if to_player.length() > ATTACK_RANGE || !timer.0.just_finished() {
+            continue;
+        }
+
+        health.0 -= damage.0;
+        info!("enemy hits you -> {} HP", health.0);
+    }
+}
+
+fn death(mut commands: Commands, query: Query<(Entity, &Health, Option<&Player>)>) {
+    for (entity, health, player) in query {
+        if health.0 > 0.0 {
+            continue;
+        }
+        if player.is_some() {
+            info!("You Died");
+            continue;
+        }
+
+        commands.entity(entity).despawn();
+    }
+}
+
+fn move_to_target(
     mut commands: Commands,
-    mut query: Query<(Entity, &mut Transform, &MoveTarget)>,
+    mut query: Query<(Entity, &mut Transform, &MoveTarget, &Speed)>,
     time: Res<Time>,
 ) {
-    for (entity, mut transform, target) in &mut query {
+    for (entity, mut transform, target, speed) in &mut query {
         let mut to_target = target.0 - transform.translation;
         to_target.y = 0.0;
 
         let distance = to_target.length();
-        let step = SPEED * time.delta_secs();
+        let step = speed.0 * time.delta_secs();
 
         if distance <= step {
             transform.translation.x = target.0.x;
@@ -63,19 +155,40 @@ fn move_player(
             continue;
         }
 
+        transform.look_to(to_target, Vec3::Y);
         transform.translation += to_target / distance * step;
     }
 }
 
+fn enemy_chase(
+    mut commands: Commands,
+    player: Single<&Transform, With<Player>>,
+    enemies: Query<(Entity, &Transform), With<Enemy>>,
+) {
+    for (entity, transform) in &enemies {
+        let mut to_player = player.translation - transform.translation;
+        to_player.y = 0.0;
+
+        if to_player.length() <= ATTACK_RANGE {
+            commands.entity(entity).remove::<MoveTarget>();
+            continue;
+        }
+        commands
+            .entity(entity)
+            .insert(MoveTarget(player.translation));
+    }
+}
+
 const CAMERA_OFFSET: Vec3 = Vec3::new(0.0, 12.0, 12.0);
 
 fn camera_follow(
     player: Single<&Transform, With<Player>>,
-    mut camera: Single<&mut Transform, With<Camera3d>>,
+    mut camera: Single<&mut Transform, (With<Camera3d>, Without<Player>)>,
 ) {
     camera.translation = player.translation + CAMERA_OFFSET;
     camera.look_at(player.translation, Vec3::Y);
 }
+
 fn click_to_point(
     mouse: Res<ButtonInput<MouseButton>>,
     camera: Single<(&Camera, &GlobalTransform)>,
@@ -105,3 +218,18 @@ fn click_to_point(
     commands.entity(*player).insert(MoveTarget(point));
     info!("Clicked at {point:?}");
 }
+
+#[derive(Component)]
+struct Enemy;
+
+#[derive(Component)]
+struct Speed(f32);
+
+const ATTACK_RANGE: f32 = 1.5;
+
+#[derive(Component)]
+struct Health(f32);
+#[derive(Component)]
+struct Damage(f32);
+#[derive(Component)]
+struct AttackTimer(Timer);
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
