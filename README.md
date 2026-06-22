# shullhammer-mod-for-minecraft-26.xx-or-27.xx
Hitboxes. Code. Data packs. for wild radical idealists like me.
look at other links at the bottom! also swedish translation at bottom
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







SWEIDSH TRANSLATION











Hitboxes. Kod. Datapaket. För vilda radikala idealister som jag.
---

Ämne: Shulhammer – Teknisk implementeringsöversikt (Basskada: 4,5 + Eld DoT + Sammandrabbning + Tung vapenkategori + Slotspecifik överlägg + 5 förtrollningar + PvP-klarering + Delade förtrollningar)

---

1. Nytt föremål och attribut

· Föremåls-ID: minecraft:shulhammer
· Kreativ flik: Strid
· Attributmodifierare:
· generic.attack_damage = +8,0 (9 HP = 4,5 hjärtor)
· generic.attack_speed = -2,4 (1,2s nedkylning)
· generic.movement_speed = -0,05 (5 % nedbromsning)
· generic.knockback_resistance = +0,10 (10 % motstånd)
· Hållbarhet: 500
· Reparationsingrediens: minecraft:shulker_shell

---

2. Recept (Data Pack JSON)

```json
{
"type": "minecraft:crafting_shapeless",
"ingredienser": [
{ "item": "minecraft:shulker_shell", "count": 6 },
{ "item": "minecraft:blaze_rod", "count": 3 },
{ "item": "minecraft:heavy_core", "count": 1 }
],
"resultat": {
"id": "minecraft:shulhammer",
"count": 1
}
}
```

---

3. Positionsskada + Eldapplikation (Serversidan)

```java
public float calculateShulhammerDamage(LivingEntity attacker, LivingEntity target) {
float base = 4.5f;
int heightDiff = (int) Math.floor(target.getY()) - (int) Math.floor(attacker.getY());

om (target.hasEffect(MobEffects.LEVITATION)) {
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

public void onShulhammerHit(LivingEntity angripare, LivingEntity mål) {
float damage = calculateShulhammerDamage(angripare, mål);
target.hurt(DamageSource.mobAttack(angripare), damage);
target.setFireTicks(100); // 5 sekunder
spawnFlameParticles(target.getPosition(), 20);
}
```

---

4. Clash-systemet (serversidan)

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

onShulhammerHit(angripare, mål);
}

public void triggerClash(LivingEntity angripare, LivingEntity mål, float shulhammerSkada) {
float maceSkada = 10.0f;
float durabilitySkada = ((shulhammerSkada + maceSkada) / 2) * 1.25f;

angripare.getMainHandItem().hurtAndBreak((int) durabilitySkada, angripare, (p) -> {});
mål.getMainHandItem().hurtAndBreak((int) durabilitySkada, mål, (p) -> {});

// Ljud 1: KLANG
playClashSound(angripare.getPosition(), "minecraft:item.shulhammer.clash");

// Partikelkonvergens
Vector3d center = attacker.getPosition().add(target.getPosition()).multiply(0.5);
for (int i = 0; i < 50; i++) {
spawnConvergingParticle(getRandomPositionAround(attacker.getPosition(), 3.0), center, 4);
}

// Schemalägg explosion (0.2s fördröjning)
Bukkit.getScheduler().runTaskLater(plugin, () -> {
// Ljud 2: BOOM
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

5. KATEGORIN FÖR TUNGA VAPEN (Nytt system)

Definition: Tunga vapen är kraftfulla föremål som kräver engagemang.

Tunga vapen: Spårvagn, Shulhammer, Sköld.
VIKTIGT: Offhand-platsen är UNDANTAGET från begränsningen. Du kan alltid hålla en sköld där.

Implementering:

```java
public class HeavyWeaponManager {
// Lista över tunga vapen-ID:n
private static final Set<Item> HEAVY_WEAPONS = Set.of(
Items.MACE,
Items.SHULHAMMER,
Items.SHIELD
);

// Kontrollera om ett föremål är ett tungt vapen
public static boolean isHeavyWeapon(ItemStack item) {
return HEAVY_WEAPONS.contains(item.getItem());
}

// Kontrollera om ett föremål är ett tungt vapen som kan placeras i offhand
// ENDAST sköldar kan placeras i offhand. Spetsklubba och Shulhammer kan inte.
public static boolean canBeInOffhand(ItemStack item) {
return item.getItem() == Items.SHIELD;
}

// Framtvinga ett tungt vapen per Hotbar (endast huvudhandplatser 0-8)
// Offhand (plats 40) är UNDANTAGET
public static void enforceHeavyWeaponLimit(Player player) {
Inventory inv = player.getInventory();
int heavyWeaponCount = 0;
int firstHeavySlot = -1;
int secondHeavySlot = -1;

// Kontrollera ENDAST hotbar-platser (0-8) - INTE direkt (40)
for (int i = 0; i < 9; i++) {
ItemStack item = inv.getItem(i);
if (isHeavyWeapon(item)) {
heavyWeaponCount++;
if (firstHeavySlot == -1) firstHeavySlot = i;
else secondHeavySlot = i;
}
}

if (heavyWeaponCount > 1) {
// Flytta s
andra tunga vapen till inventariet (eller släpp)
ItemStack secondWeapon = inv.getItem(secondHeavySlot);
if (!inv.addItem(secondWeapon).isEmpty()) {
// Inventariet fullt: släpp på marken
player.getWorld().dropItem(player.getPosition(), secondWeapon);
}
inv.setItem(secondHeavySlot, null);
player.sendMessage("Du kan bara bära ett tungt vapen i din hotbar!");
}
}

// 0,6-sekunders swap-animation med slotspecifik vit överlagring
public static void startHeavySwap(Player player, int slot) {
// 1. Skicka vitt overlay-paket för den SPECIFIKA platsen (inte hela skärmen)
player.sendSlotOverlayPacket(slot); // Anpassat paket

// 2. Förhindra attacker/blockeringar i 0,6 sekunder (12 tick)
player.addEffect(new MobEffectInstance(MobEffects.WEAKNESS, 12, 255));

// 3. Schemalägg borttagning av overlay efter 0,6 s
Bukkit.getScheduler().runTaskLater(plugin, () -> {
player.removeSlotOverlay(slot);
}, 12L);
}

// Kontrollera om spelaren håller ett tungt vapen i huvudhanden
public static boolean isHoldingHeavyWeapon(Player player) {
return isHeavyWeapon(player.getInventory().getItemInMainHand());
}

// Sköld är ALLTID tillåten i offhand
public static boolean isShieldInOffhand(Player player) {
return player.getInventory().getItemInOffHand().getItem() == Items.SHIELD;
}

// Förhindra placering av Mace/Shulhammer i offhand
public static boolean canPlaceInOffhand(ItemStack item) {
if (item == null) return true;
if (item.getItem() == Items.MACE || item.getItem() == Items.SHULHAMMER) {
return false; // Kan inte placera tunga vapen i offhand
}
return true;
}
}
```

---

6. Klientsidans slotspecifika vita överlägg (visuell feedback)

```java
// Klientsidans rendering (förenklad)
public void renderSlotOverlay(int slot) {
// En vit gradient som glider ner över den SPECIFIKA hotbar-platsen
// Varaktighet: 0,6 sekunder (12 tick)
// Opacitet: Börjar vid 100 %, slutar vid 0 %
// Visuell: En kort "blixt" på själva platsen, inte hela skärmen
// Platsens objektikon skyms kort av det vita överlägget
// Överlägget glider ner från toppen av platsen och försvinner
}
```

---

7. Rensa förtrollning: PvP-implementering

```java
public void triggerClear(LivingEntity attacker, Level world, BlockPos centerPos, int radius) {
if (!world.getGameRules().getBoolean(GameRules.SHULHAMMER_CLEAR_PVP)) {
return;
}
if (isProtectedArea(world, centerPos)) {
return;
}
for (int x = -radie; x <= radie; x++) {
for (int z = -radie; z <= radie; z++) {
BlockPos targetPos = centerPos.add(x, 0, z);
BlockState state = world.getBlockState(targetPos);
if (state.getBlock() == Blocks.BREDROCK ||
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

8. Inställning av förtrollningskompatibilitet

Exklusiva förtrollningar (endast Shulhammer):

· dig (III)
· breakthrough (IV)
· clear (IV)
· mayhem (III)
· force (II)

Delade förtrollningar (Mace & Shulhammer):

· mending (I)
· unbreaking (III)
· smite (V)
· bane_of_arthropods (V)

INTE kompatibel:

· fire_aspect (redundant—eld appliceras redan på varje träff)
· curse_of_vanishing (avsiktligt utesluten)

---

9. Spelregler (för serveradministration)

Spelregeltyp Standard Beskrivning
shulhammerAktiverad Boolean sant Master-växling.
shulhammerSkadaCap Float 11.25 Begränsar omedelbar skada.
shulhammerFireDuration Heltal 100 Eldtid i ticks.
shulhammerClearPvP Boolean true Aktiverar Clear i PvP.
shulhammerClearDurabilityCost Heltal 1 Hållbarhetskostnad per block som brutits.
shulhammerClashEnabled Boolean true Växlar Clash System.
shulhammerClashWindow Heltal 400 Clash-fönster i millisekunder.
shulhammerSwapDuration Heltal 12 Växla animationstid i ticks.
shulhammerHeavyHotbarLimit Heltal 1 Max tunga vapen i hotbar.

--

10. Sammanfattning för implementering

Detta vapen kräver:

1. 1 nytt föremål: Shulhammer.
2. 1 nytt recept: 6 granater + 3 stavar + 1 kärna.
3. 1 Ny metod för skadeberäkning (positionsskalning).
4. Eldtillämpning vid varje direktträff (5 sekunder).
5. 1 Nytt konfliktsystem (partiklar + ljud + bedövning).
6. 1 Ny kategori för tunga vapen (restriktion för hotbar + platsspecifikt överlägg).
7. 5 exklusiva förtrollningar + 4 delade förtrollningar.
8. 8 nya spelregler.

Beräknad utvecklingsinsats: 3–4 spurter. De delade förtrollningarna gör att Shulhammern känns som en naturlig rival till klubban, inte ett slumpmässigt tillägg.













LINKS:
GOOGLE DOCS:
PASTE BIN:
