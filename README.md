# shullhammer-mod-for-minecraft-26.xx-or-27.xx
Hitboxes. Code. Data packs. for wild radical idealists like me.

---

Subject: Shulhammer – Technical Implementation Overview (Base Damage: 4.5 + Fire DoT + Clash + Heavy Weapon Category + Slot-Specific Overlay + 5 Enchants + PvP Clear + Shared Enchants)

---

1. New Item & Attributes

· Item ID: minecraft:shulhammer
· Creative Tab: Combat
· Attribute Modifiers:
· generic.attack_damage = +8.0 (9 HP = 4.5 Hearts)
· generic.attack_speed = -2.4 (1.2s cooldown)
· generic.movement_speed = -0.05 (5% slowdown)
· generic.knockback_resistance = +0.10 (10% resistance)
· Durability: 500
· Repair Ingredient: minecraft:shulker_shell

---

2. Recipe (Data Pack JSON)

```json
{
"type": "minecraft:crafting_shapeless",
"ingredients": [
{ "item": "minecraft:shulker_shell", "count": 6 },
{ "item": "minecraft:blaze_rod", "count": 3 },
{ "item": "minecraft:heavy_core", "count": 1 }
],
"result": {
"id": "minecraft:shulhammer",
"count": 1
}
}
```

---

3. Positional Damage + Fire Application (Server-Side)

```java
public float calculateShulhammerDamage(LivingEntity attacker, LivingEntity target) {
float base = 4.5f;
int heightDiff = (int) Math.floor(target.getY()) - (int) Math.floor(attacker.getY());

if (target.hasEffect(MobEffects.LEVITATION)) {
return base * 3.0f; // 13.5 Hearts
}
if (heightDiff > 0) {
float bonus = Math.min(heightDiff * 0.25f, 1.5f);
return base * (1.0f + bonus);
} else if (heightDiff < 0) {
float penalty = Math.min(Math.abs(heightDiff) * 0.10f, 0.5f);
return base * (1.0f - penalty);
}
return base;
}

public void onShulhammerHit(LivingEntity attacker, LivingEntity target) {
float damage = calculateShulhammerDamage(attacker, target);
target.hurt(DamageSource.mobAttack(attacker), damage);
target.setFireTicks(100); // 5 seconds
spawnFlameParticles(target.getPosition(), 20);
}
```

---

4. The Clash System (Server-Side)

```java
public void onHeavyWeaponAttack(LivingEntity attacker, LivingEntity target, float shulhammerDamage) {
long currentTime = System.currentTimeMillis();

if (target.getMainHandItem().getItem() instanceof Shulhammer || target.getMainHandItem().getItem() instanceof Mace) {
long targetLastAttack = target.getPersistentData().getLong("lastHeavyAttackTime");
if (currentTime - targetLastAttack <= 400) {
triggerClash(attacker, target, shulhammerDamage);
return;
}
}

onShulhammerHit(attacker, target);
}

public void triggerClash(LivingEntity attacker, LivingEntity target, float shulhammerDamage) {
float maceDamage = 10.0f;
float durabilityDamage = ((shulhammerDamage + maceDamage) / 2) * 1.25f;

attacker.getMainHandItem().hurtAndBreak((int) durabilityDamage, attacker, (p) -> {});
target.getMainHandItem().hurtAndBreak((int) durabilityDamage, target, (p) -> {});

// Sound 1: CLANG
playClashSound(attacker.getPosition(), "minecraft:item.shulhammer.clash");

// Particle Convergence
Vector3d center = attacker.getPosition().add(target.getPosition()).multiply(0.5);
for (int i = 0; i < 50; i++) {
spawnConvergingParticle(getRandomPositionAround(attacker.getPosition(), 3.0), center, 4);
}

// Schedule Explosion (0.2s delay)
Bukkit.getScheduler().runTaskLater(plugin, () -> {
// Sound 2: BOOM
playClashSound(center, "minecraft:item.shulhammer.boom");
spawnExplosionParticles(center, 80);
spawnShockwaveRing(center, 6.0);
applyKnockbackAndStun(attacker, target, center);
}, 4L);

attacker.getPersistentData().putLong("lastHeavyAttackTime", System.currentTimeMillis());
target.getPersistentData().putLong("lastHeavyAttackTime", System.currentTimeMillis());
}
```

---

5. THE HEAVY WEAPON CATEGORY (New System)

Definition: Heavy Weapons are powerful items that require commitment.

Heavy Weapons: Mace, Shulhammer, Shield.
IMPORTANT: Offhand slot is EXEMPT from the restriction. You can always hold a Shield there.

Implementation:

```java
public class HeavyWeaponManager {
// List of Heavy Weapon IDs
private static final Set<Item> HEAVY_WEAPONS = Set.of(
Items.MACE,
Items.SHULHAMMER,
Items.SHIELD
);

// Check if an item is a Heavy Weapon
public static boolean isHeavyWeapon(ItemStack item) {
return HEAVY_WEAPONS.contains(item.getItem());
}

// Check if an item is a Heavy Weapon that can be placed in offhand
// ONLY Shields can be placed in offhand. Mace and Shulhammer cannot.
public static boolean canBeInOffhand(ItemStack item) {
return item.getItem() == Items.SHIELD;
}

// Enforce One Heavy Weapon per Hotbar (main-hand slots 0-8 only)
// Offhand (slot 40) is EXEMPT
public static void enforceHeavyWeaponLimit(Player player) {
Inventory inv = player.getInventory();
int heavyWeaponCount = 0;
int firstHeavySlot = -1;
int secondHeavySlot = -1;

// ONLY check hotbar slots (0-8) - NOT offhand (40)
for (int i = 0; i < 9; i++) {
ItemStack item = inv.getItem(i);
if (isHeavyWeapon(item)) {
heavyWeaponCount++;
if (firstHeavySlot == -1) firstHeavySlot = i;
else secondHeavySlot = i;
}
}

if (heavyWeaponCount > 1) {
// Move the second Heavy Weapon to inventory (or drop)
ItemStack secondWeapon = inv.getItem(secondHeavySlot);
if (!inv.addItem(secondWeapon).isEmpty()) {
// Inventory full: drop on ground
player.getWorld().dropItem(player.getPosition(), secondWeapon);
}
inv.setItem(secondHeavySlot, null);
player.sendMessage("You can only carry one Heavy Weapon in your hotbar!");
}
}

// 0.6-Second Swap Animation with Slot-Specific White Overlay
public static void startHeavySwap(Player player, int slot) {
// 1. Send white overlay packet for the SPECIFIC slot (not the whole screen)
player.sendSlotOverlayPacket(slot); // Custom packet

// 2. Prevent attacks/blocks for 0.6 seconds (12 ticks)
player.addEffect(new MobEffectInstance(MobEffects.WEAKNESS, 12, 255));

// 3. Schedule overlay removal after 0.6s
Bukkit.getScheduler().runTaskLater(plugin, () -> {
player.removeSlotOverlay(slot);
}, 12L);
}

// Check if player is holding a Heavy Weapon in main hand
public static boolean isHoldingHeavyWeapon(Player player) {
return isHeavyWeapon(player.getInventory().getItemInMainHand());
}

// Shield is ALWAYS allowed in offhand
public static boolean isShieldInOffhand(Player player) {
return player.getInventory().getItemInOffHand().getItem() == Items.SHIELD;
}

// Prevent placing Mace/Shulhammer in offhand
public static boolean canPlaceInOffhand(ItemStack item) {
if (item == null) return true;
if (item.getItem() == Items.MACE || item.getItem() == Items.SHULHAMMER) {
return false; // Cannot place Heavy Weapons in offhand
}
return true;
}
}
```

---

6. Client-Side Slot-Specific White Overlay (Visual Feedback)

```java
// Client-side rendering (simplified)
public void renderSlotOverlay(int slot) {
// A white gradient that slides down over the SPECIFIC hotbar slot
// Duration: 0.6 seconds (12 ticks)
// Opacity: Starts at 100%, ends at 0%
// Visual: A brief "flash" on the slot itself, not the whole screen
// The slot's item icon is briefly obscured by the white overlay
// The overlay slides down from the top of the slot and disappears
}
```

---

7. Clear Enchantment: PvP Implementation

```java
public void triggerClear(LivingEntity attacker, Level world, BlockPos centerPos, int radius) {
if (!world.getGameRules().getBoolean(GameRules.SHULHAMMER_CLEAR_PVP)) {
return;
}
if (isProtectedArea(world, centerPos)) {
return;
}
for (int x = -radius; x <= radius; x++) {
for (int z = -radius; z <= radius; z++) {
BlockPos targetPos = centerPos.add(x, 0, z);
BlockState state = world.getBlockState(targetPos);
if (state.getBlock() == Blocks.BEDROCK ||
state.getBlock() == Blocks.OBSIDIAN ||
state.getBlock() == Blocks.REINFORCED_DEEPSLATE) {
continue;
}
world.destroyBlock(targetPos, true, attacker);
attacker.getMainHandItem().hurtAndBreak(1, attacker, (p) -> {});
}
}
}
```

---

8. Enchantment Compatibility Setup

Exclusive Enchantments (Shulhammer Only):

· dig (III)
· breakthrough (IV)
· clear (IV)
· mayhem (III)
· force (II)

Shared Enchantments (Mace & Shulhammer):

· mending (I)
· unbreaking (III)
· smite (V)
· bane_of_arthropods (V)

NOT Compatible:

· fire_aspect (redundant—fire is already applied on every hit)
· curse_of_vanishing (intentionally excluded)

---

9. Gamerules (For Server Administration)

Gamerule Type Default Description
shulhammerEnabled Boolean true Master toggle.
shulhammerDamageCap Float 11.25 Caps instant damage.
shulhammerFireDuration Integer 100 Fire duration in ticks.
shulhammerClearPvP Boolean true Enables Clear in PvP.
shulhammerClearDurabilityCost Integer 1 Durability cost per block broken.
shulhammerClashEnabled Boolean true Toggles Clash System.
shulhammerClashWindow Integer 400 Clash window in milliseconds.
shulhammerSwapDuration Integer 12 Swap animation duration in ticks.
shulhammerHeavyHotbarLimit Integer 1 Max Heavy Weapons in hotbar.

---

10. Summary for Implementation

This weapon requires:

1. 1 New Item: Shulhammer.
2. 1 New Recipe: 6 Shells + 3 Rods + 1 Core.
3. 1 New Damage Calculation Method (positional scaling).
4. Fire Application on every direct hit (5 seconds).
5. 1 New Clash System (particles + sounds + stun).
6. 1 New Heavy Weapon Category (hotbar restriction + slot-specific overlay).
7. 5 Exclusive Enchantments + 4 Shared Enchantments.
8. 8 New Gamerules.

Estimated Development Effort: 3–4 sprints. The shared enchantments make the Shulhammer feel like a natural rival to the Mace, not a random addition.
