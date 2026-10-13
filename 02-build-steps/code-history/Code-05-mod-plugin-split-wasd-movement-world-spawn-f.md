---
title: "05 — mod : plugin split, WASD movement, world_spawn fix"
date: 2026-09-28
sha: 7ec3f1f
type: code-history
tags: [phoenix, code-history, bevy, rust]
---

# 05 — mod : plugin split, WASD movement, world_spawn fix

`7ec3f1f` · 2026-09-28 · generated from git, not written by hand.

## Files changed

```
 src/camera.rs   |  28 +++++++
 src/combat.rs   |  81 +++++++++++++++++++
 src/enemy.rs    |  52 ++++++++++++
 src/main.rs     | 242 ++++----------------------------------------------------
 src/movement.rs |  39 +++++++++
 src/player.rs   |  98 +++++++++++++++++++++++
 src/world.rs    |  27 +++++++
 7 files changed, 339 insertions(+), 228 deletions(-)
```

## Diff

```diff
diff --git a/src/camera.rs b/src/camera.rs
new file mode 100644
index 0000000..356a7a7
--- /dev/null
+++ b/src/camera.rs
@@ -0,0 +1,28 @@
+use crate::player::Player;
+use bevy::prelude::*;
+
+const CAMERA_OFFSET: Vec3 = Vec3::new(0.0, 12.0, 12.0);
+
+pub struct CameraPlugin;
+
+impl Plugin for CameraPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Startup, spawn_camera)
+            .add_systems(Update, camera_follow);
+    }
+}
+
+fn camera_follow(
+    player: Single<&Transform, With<Player>>,
+    mut camera: Single<&mut Transform, (With<Camera3d>, Without<Player>)>,
+) {
+    camera.translation = player.translation + CAMERA_OFFSET;
+    camera.look_at(player.translation, Vec3::Y);
+}
+
+fn spawn_camera(mut commands: Commands) {
+    commands.spawn((
+        Camera3d::default(),
+        Transform::from_xyz(0.0, 7.0, 14.0).looking_at(Vec3::new(0.0, 1.0, 0.0), Vec3::Y),
+    ));
+}
diff --git a/src/combat.rs b/src/combat.rs
new file mode 100644
index 0000000..fa682a8
--- /dev/null
+++ b/src/combat.rs
@@ -0,0 +1,81 @@
+use bevy::prelude::*;
+
+use crate::enemy::Enemy;
+use crate::player::Player;
+
+pub const ATTACK_RANGE: f32 = 1.5;
+pub struct CombatPlugin;
+
+impl Plugin for CombatPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Update, (player_attack, enemy_attack, death));
+    }
+}
+
+#[derive(Component)]
+pub struct Health(pub f32);
+#[derive(Component)]
+pub struct Damage(pub f32);
+#[derive(Component)]
+pub struct AttackTimer(pub Timer);
+
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
diff --git a/src/enemy.rs b/src/enemy.rs
new file mode 100644
index 0000000..bec5899
--- /dev/null
+++ b/src/enemy.rs
@@ -0,0 +1,52 @@
+use crate::combat::{ATTACK_RANGE, AttackTimer, Damage, Health};
+use crate::movement::{MoveTarget, Speed};
+use crate::player::Player;
+use bevy::prelude::*;
+
+pub struct EnemyPlugin;
+
+impl Plugin for EnemyPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Startup, enemy_spawn)
+            .add_systems(Update, enemy_chase);
+    }
+}
+
+#[derive(Component)]
+pub struct Enemy;
+
+fn enemy_spawn(
+    mut commands: Commands,
+    mut meshes: ResMut<Assets<Mesh>>,
+    mut materials: ResMut<Assets<StandardMaterial>>,
+) {
+    commands.spawn((
+        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
+        MeshMaterial3d(materials.add(Color::srgb_u8(220, 80, 80))),
+        Transform::from_xyz(8.0, 0.5, -6.0),
+        Enemy,
+        Speed(3.0),
+        Health(60.0),
+        Damage(8.0),
+        AttackTimer(Timer::from_seconds(1.2, TimerMode::Repeating)),
+    ));
+}
+
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
diff --git a/src/main.rs b/src/main.rs
index 4adacb7..229e986 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -1,235 +1,21 @@
 use bevy::prelude::*;
+mod camera;
+mod combat;
+mod enemy;
+mod movement;
+mod player;
+mod world;
 
 fn main() {
     App::new()
-        .add_systems(Startup, setup)
         .add_plugins(DefaultPlugins)
-        .add_systems(
-            Update,
-            (
-                move_to_target,
-                click_to_point,
-                camera_follow,
-                enemy_chase,
-                player_attack,
-                enemy_attack,
-                death,
-            ),
-        )
+        .add_plugins((
+            camera::CameraPlugin,
+            movement::MovementPlugin,
+            player::PlayerPlugin,
+            world::WorldPlugin,
+            combat::CombatPlugin,
+            enemy::EnemyPlugin,
+        ))
         .run();
 }
-
-#[derive(Component)]
-struct Player;
-
-#[derive(Component)]
-struct MoveTarget(Vec3);
-
-fn setup(
-    mut commands: Commands,
-    mut meshes: ResMut<Assets<Mesh>>,
-    mut materials: ResMut<Assets<StandardMaterial>>,
-) {
-    commands.spawn((
-        Mesh3d(meshes.add(Plane3d::default().mesh().size(50.0, 50.0))),
-        MeshMaterial3d(materials.add(Color::WHITE)),
-    ));
-    commands.spawn((
-        PointLight {
-            shadow_maps_enabled: true,
-            ..default()
-        },
-        Transform::from_xyz(8.0, 16.0, 8.0),
-    ));
-    commands.spawn((
-        Camera3d::default(),
-        Transform::from_xyz(0.0, 7.0, 14.0).looking_at(Vec3::new(0.0, 1.0, 0.0), Vec3::Y),
-    ));
-    commands.spawn((
-        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
-        MeshMaterial3d(materials.add(Color::srgb_u8(124, 144, 255))),
-        Transform::from_xyz(0.0, 0.5, 0.0),
-        Player,
-        children![(
-            Mesh3d(meshes.add(Cuboid::new(0.3, 0.3, 0.6))),
-            MeshMaterial3d(materials.add(Color::srgb_u8(255, 200, 80))),
-            Transform::from_xyz(0.0, 0.0, -0.7),
-        )],
-        Speed(5.0),
-        Health(100.0),
-        Damage(25.0),
-        AttackTimer(Timer::from_seconds(0.8, TimerMode::Repeating)),
-    ));
-
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
-}
-
-const SPEED: f32 = 5.0;
-
-fn player_attack(
-    time: Res<Time>,
-    player: Single<(&Transform, &Damage, &mut AttackTimer), With<Player>>,
-    mut enemies: Query<(&Transform, &mut Health), With<Enemy>>,
-) {
-    let (player_transform, damage, mut timer) = player.into_inner();
-    timer.0.tick(time.delta());
-
-    if !timer.0.just_finished() {
-        return;
-    }
-
-    for (enemy_transform, mut health) in &mut enemies {
-        let mut to_enemy = enemy_transform.translation - player_transform.translation;
-        to_enemy.y = 0.0;
-
-        if to_enemy.length() > ATTACK_RANGE {
-            continue;
-        }
-
-        health.0 -= damage.0;
-        info!("hit enemy -> {} HP", health.0);
-    }
-}
-
-fn enemy_attack(
-    time: Res<Time>,
-    player: Single<(&Transform, &mut Health), With<Player>>,
-    mut enemies: Query<(&Transform, &Damage, &mut AttackTimer), With<Enemy>>,
-) {
-    let (player_transform, mut health) = player.into_inner();
-
-    for (enemy_transform, damage, mut timer) in &mut enemies {
-        timer.0.tick(time.delta());
-
-        let mut to_player = player_transform.translation - enemy_transform.translation;
-        to_player.y = 0.0;
-
-        if to_player.length() > ATTACK_RANGE || !timer.0.just_finished() {
-            continue;
-        }
-
-        health.0 -= damage.0;
-        info!("enemy hits you -> {} HP", health.0);
-    }
-}
-
-fn death(mut commands: Commands, query: Query<(Entity, &Health, Option<&Player>)>) {
-    for (entity, health, player) in query {
-        if health.0 > 0.0 {
-            continue;
-        }
-        if player.is_some() {
-            info!("You Died");
-            continue;
-        }
-
-        commands.entity(entity).despawn();
-    }
-}
-
-fn move_to_target(
-    mut commands: Commands,
-    mut query: Query<(Entity, &mut Transform, &MoveTarget, &Speed)>,
-    time: Res<Time>,
-) {
-    for (entity, mut transform, target, speed) in &mut query {
-        let mut to_target = target.0 - transform.translation;
-        to_target.y = 0.0;
-
-        let distance = to_target.length();
-        let step = speed.0 * time.delta_secs();
-
-        if distance <= step {
-            transform.translation.x = target.0.x;
-            transform.translation.z = target.0.z;
-            commands.entity(entity).remove::<MoveTarget>();
-            continue;
-        }
-
-        transform.look_to(to_target, Vec3::Y);
-        transform.translation += to_target / distance * step;
-    }
-}
-
-fn enemy_chase(
-    mut commands: Commands,
-    player: Single<&Transform, With<Player>>,
-    enemies: Query<(Entity, &Transform), With<Enemy>>,
-) {
-    for (entity, transform) in &enemies {
-        let mut to_player = player.translation - transform.translation;
-        to_player.y = 0.0;
-
-        if to_player.length() <= ATTACK_RANGE {
-            commands.entity(entity).remove::<MoveTarget>();
-            continue;
-        }
-        commands
-            .entity(entity)
-            .insert(MoveTarget(player.translation));
-    }
-}
-
-const CAMERA_OFFSET: Vec3 = Vec3::new(0.0, 12.0, 12.0);
-
-fn camera_follow(
-    player: Single<&Transform, With<Player>>,
-    mut camera: Single<&mut Transform, (With<Camera3d>, Without<Player>)>,
-) {
-    camera.translation = player.translation + CAMERA_OFFSET;
-    camera.look_at(player.translation, Vec3::Y);
-}
-
-fn click_to_point(
-    mouse: Res<ButtonInput<MouseButton>>,
-    camera: Single<(&Camera, &GlobalTransform)>,
-    window: Single<&Window>,
-    mut commands: Commands,
-    player: Single<Entity, With<Player>>,
-) {
-    if !mouse.just_pressed(MouseButton::Left) {
-        return;
-    }
-
-    let (camera, camera_transform) = *camera;
-
-    let Some(cursor) = window.cursor_position() else {
-        return; // Pointer is outside the window
-    };
-
-    let Ok(ray) = camera.viewport_to_world(camera_transform, cursor) else {
-        return; // camera coudn't build a ray
-    };
-
-    let Some(point) = ray.plane_intersection_point(Vec3::ZERO, InfinitePlane3d::new(Vec3::Y))
-    else {
-        return; // ray never meets the ground
-    };
-
-    commands.entity(*player).insert(MoveTarget(point));
-    info!("Clicked at {point:?}");
-}
-
-#[derive(Component)]
-struct Enemy;
-
-#[derive(Component)]
-struct Speed(f32);
-
-const ATTACK_RANGE: f32 = 1.5;
-
-#[derive(Component)]
-struct Health(f32);
-#[derive(Component)]
-struct Damage(f32);
-#[derive(Component)]
-struct AttackTimer(Timer);
diff --git a/src/movement.rs b/src/movement.rs
new file mode 100644
index 0000000..a6384b6
--- /dev/null
+++ b/src/movement.rs
@@ -0,0 +1,39 @@
+use bevy::prelude::*;
+
+pub struct MovementPlugin;
+
+impl Plugin for MovementPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Update, move_to_target);
+    }
+}
+
+#[derive(Component)]
+pub struct MoveTarget(pub Vec3);
+
+#[derive(Component)]
+pub struct Speed(pub f32);
+
+fn move_to_target(
+    mut commands: Commands,
+    mut query: Query<(Entity, &mut Transform, &MoveTarget, &Speed)>,
+    time: Res<Time>,
+) {
+    for (entity, mut transform, target, speed) in &mut query {
+        let mut to_target = target.0 - transform.translation;
+        to_target.y = 0.0;
+
+        let distance = to_target.length();
+        let step = speed.0 * time.delta_secs();
+
+        if distance <= step {
+            transform.translation.x = target.0.x;
+            transform.translation.z = target.0.z;
+            commands.entity(entity).remove::<MoveTarget>();
+            continue;
+        }
+
+        transform.look_to(to_target, Vec3::Y);
+        transform.translation += to_target / distance * step;
+    }
+}
diff --git a/src/player.rs b/src/player.rs
new file mode 100644
index 0000000..df9eeef
--- /dev/null
+++ b/src/player.rs
@@ -0,0 +1,98 @@
+use crate::combat::{AttackTimer, Damage, Health};
+use crate::movement::{MoveTarget, Speed};
+use bevy::prelude::*;
+
+pub struct PlayerPlugin;
+
+impl Plugin for PlayerPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Startup, player_spawn)
+            .add_systems(Update, (click_to_point, keyboard_move));
+    }
+}
+
+#[derive(Component)]
+pub struct Player;
+
+fn player_spawn(
+    mut commands: Commands,
+    mut meshes: ResMut<Assets<Mesh>>,
+    mut materials: ResMut<Assets<StandardMaterial>>,
+) {
+    commands.spawn((
+        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
+        MeshMaterial3d(materials.add(Color::srgb_u8(124, 144, 255))),
+        Transform::from_xyz(0.0, 0.5, 0.0),
+        Player,
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
+}
+
+fn keyboard_move(
+    keys: Res<ButtonInput<KeyCode>>,
+    time: Res<Time>,
+    mut commands: Commands,
+    player: Single<(Entity, &mut Transform, &Speed), With<Player>>,
+) {
+    let (entity, mut transform, speed) = player.into_inner();
+
+    let mut direction = Vec3::ZERO;
+    if keys.pressed(KeyCode::KeyW) {
+        direction.z -= 1.0;
+    }
+    if keys.pressed(KeyCode::KeyS) {
+        direction.z += 1.0;
+    }
+    if keys.pressed(KeyCode::KeyA) {
+        direction.x -= 1.0;
+    }
+    if keys.pressed(KeyCode::KeyD) {
+        direction.x += 1.0;
+    }
+
+    let Ok(direction) = Dir3::new(direction) else {
+        return;
+    };
+
+    commands.entity(entity).remove::<MoveTarget>();
+    transform.look_to(direction, Vec3::Y);
+    transform.translation += direction * speed.0 * time.delta_secs();
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
diff --git a/src/world.rs b/src/world.rs
new file mode 100644
index 0000000..a548246
--- /dev/null
+++ b/src/world.rs
@@ -0,0 +1,27 @@
+use bevy::prelude::*;
+
+pub struct WorldPlugin;
+
+impl Plugin for WorldPlugin {
+    fn build(&self, app: &mut App) {
+        app.add_systems(Startup, world_spawn);
+    }
+}
+
+fn world_spawn(
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
+}
```

## Related

- [[Build-Log]] — what this step was for and what broke
- [[Bevy-ECS-Architecture]] — the model these pieces assemble into
