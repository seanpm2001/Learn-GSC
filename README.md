
***

![/GSC_Language_SquareLogo_V1_HighCompression.png](/GSC_Language_SquareLogo_V1_HighCompression.png)

### Learning GSC (programming language)

I know very little about the GSC programming language. This document will go over all of my knowledge of the GSC programming language. Due to the purpose of the language, I don't intend to go very far with it.

This document heavily relies on reference to [`this script`](https://github.com/seanpm2001/EliotDucky_zm_weap_spike_launcher/blob/main/zm_weap_spike_launcher.gsc)

Here is an archived copy of it:

<details><summary><p lang="en"><b>Click/tap here to expand/collapse the original source</b></p></summary>

[`📄️ View this file separately`](/GSC/Sampled/EliotDucky/zm_weap_spike_launcher.gsc)

```gsc
#using scripts\shared\array_shared;
#using scripts\shared\callbacks_shared;
#using scripts\shared\hud_util_shared;
#using scripts\shared\system_shared;
#using scripts\shared\util_shared;

#insert scripts\shared\shared.gsh;

#using scripts\shared\weapons\_weaponobjects;

#using scripts\zm\_zm_utility;

#define	POI_MAX_RADIUS				200
#define	POI_HALF_HEIGHT				200
#define	POI_INNER_SPACING			2
#define	POI_RADIUS_FROM_EDGES		32
#define POI_HEIGHT 					200

#define SPIKE_POI_RANK				800
#define SPIKE_CHARGE_TIME			15

#precache("string", "PRESS ^3[{+melee}]^7 TO DETONATE SPIKE CHARGE");

#namespace zm_weap_spike_launcher;

function autoexec __init__system__(){
	_arr = undefined;
	system::register("zm_weap_spike_launcher", &__init__, &__main__, _arr);
}

function __init__(){
}

function __main__(){
	callback::on_connect(&spikeLauncherTutorialWatcher);
	callback::on_connect(&spikeLauncherWatcher);
	callback::on_connect(&spikeUpgradeWatcher);

	DEFAULT(level.monkey_attract_dist, 1536);
	DEFAULT(level.num_monkey_attractors, 96);
	DEFAULT(level.monkey_attract_dist_diff, 45);
	DEFAULT(level.spike_attract_dist, level.monkey_attract_dist);
	DEFAULT(level.num_spike_attractors, level.num_monkey_attractors);
	DEFAULT(level.spike_attract_dist_diff, level.monkey_attract_dist_diff);
}

function setSpikeAttractDist(dist){
	if(isdefined(dist) && dist >= 0){
		level.spike_attract_dist = dist;
	}
}

//Call On: Player
function spikeLauncherWatcher(){
	self weaponobjects::createSpikeLauncherWatcher("spike_launcher");
	spike_watcher = self weaponobjects::createWeaponObjectWatcher("spike_charge", self.team);
	spike_watcher.onSpawn = &spikeWatcher;
}

//Call On: Player
//Callback on connect
function spikeLauncherTutorialWatcher(){
	wpn_spike_launcher = GetWeapon("spike_launcher");
	self.spike_launcher_tutorial_complete = false;
	w_current = self GetCurrentWeapon();
	while(!self.spike_launcher_tutorial_complete){
		if(w_current == wpn_spike_launcher){
			self detonateWaitTill(wpn_spike_launcher);
		}else{
			self waittill("weapon_change_complete", w_current);
		}
	}
}

//Call On: Player
function detonateWaitTill(wpn_spike_launcher){
	self endon("death");
	self waittill("weapon_fired", w_current);
	if(w_current == wpn_spike_launcher){
		wait(2);
		self thread spikeLauncherTutorialHUD();
		self util::waittill_any("detonate", "last_stand_detonate");
		self.spike_launcher_tutorial_complete = true;
	}
}

//Call On: Player
function spikeLauncherTutorialHUD(){
	self notify("spike_launcher_HUD");
	self endon("spike_launcher_HUD");
	font = "default";
	fontscale = 2;
	if(level.Splitscreen && !level.hidef){
		fontscale = 3;
	}
	txt = self hud::createFontString(font, fontscale);
	txt.vertalign = "bottom";
	txt.y = -100;
	txt.alpha = 0;
	txt SetText("PRESS ^3[{+melee}]^7 TO DETONATE SPIKE CHARGE");
	txt FadeOverTime(0.5);
	txt.alpha = 1;

	self util::waittill_any_timeout(20, "detonate", "last_stand_detonate");

	txt FadeOverTime(0.5);
	txt.alpha = 0;
	wait(0.5);
	txt Destroy();
}

//Call On: Player
//Callback on connect
function spikeUpgradeWatcher(){
	self.spike_pois = [];

	weapon = "spike_launcher_upgraded";
	watcher = self weaponobjects::createUseWeaponObjectWatcher(weapon, self.team);
	watcher.altName = "spike_charge_upgraded";
	watcher.altWeapon = GetWeapon("spike_charge_upgraded");
	watcher.altDetonate = false;
	watcher.watchForFire = true;
	watcher.hackable = true;
	watcher.hackerToolRadius = level.equipmentHackerToolRadius;
	watcher.hackerToolTimeMs = level.equipmentHackerToolTimeMs;
	watcher.headIcon = false;
	watcher.onDetonateCallback = &weaponobjects::spikeDetonate;
	watcher.onStun = &weaponobjects::weaponStun;
	watcher.stunTime = 1;
	watcher.ownerGetsAssist = true;
	watcher.detonateStationary = false;
	watcher.detonationDelay = 0.0;
	watcher.detonationSound = "wpn_claymore_alert";
	watcher.onDetonateHandle = &upgradedSpikesDetonating;
	self thread upgradedSpikeLauncherUpgradedItemCountChanged(watcher);

	upgrade_spike_watcher = self weaponobjects::createWeaponObjectWatcher("spike_charge_upgraded", self.team);
	upgrade_spike_watcher.onSpawn = &upgradedSpikeWatcher;
	upgrade_spike_watcher.onDetonateCallback = &endSpikeAttractionOnDeath;
}

//Call On: Player
function upgradedSpikesDetonating(watcher){
	spike_count = weaponobjects::getSpikeLauncherActiveSpikeCount(watcher);
	if ( spike_count > 0 )
	{
		self SetControllerUIModelValue( "spikeLauncherCounter.blasting", 1 );
		wait 2;
		self SetControllerUIModelValue( "spikeLauncherCounter.blasting", 0 );
	}
}

//Call On: Player
function upgradedSpikeLauncherUpgradedItemCountChanged(watcher){
	self notify("uSLUICC");
	self endon("uSLUICC");
	self endon("death");
	last_item_count = undefined;
	while(true){
		self waittill("weapon_change", weapon);
		while(weapon.name == "spike_launcher_upgraded"){
			current_item_count = weaponobjects::getSpikeLauncherActiveSpikeCount(watcher);
			if(current_item_count !== last_item_count){
				self SetControllerUIModelValue("spikeLauncherCounter.spikesReady", current_item_count);
				last_item_count = current_item_count;
			}
			wait(0.1);
			weapon = self GetCurrentWeapon();
		}
	}
}

//Call On: The spawned bolt
function upgradedSpikeWatcher(watcher, owner){
	IPrintLn("spike_fired");
	self endon("death");
	self thread detonateAfterTime(SPIKE_CHARGE_TIME, owner);
	self util::waitTillNotMoving();
	//the above filters only those spawned on a surface in

	DEFAULT(level.spike_pois, []);
	
	//Get nav mesh position near this spike
	b_valid_poi = zm_utility::check_point_in_enabled_zone(self.origin, undefined, undefined);
	v_valid_poi = self move_valid_poi_to_navmesh(b_valid_poi);
	if(v_valid_poi != (0, 0, 0)){
		spike_poi = Spawn("script_origin", v_valid_poi);
		spike_poi zm_utility::create_zombie_point_of_interest(
			level.spike_attract_dist,
			level.num_spike_attractors,
			10000);
		spike_poi thread zm_utility::create_zombie_point_of_interest_attractor_positions(
			4, level.spike_attract_dist_diff
		);
		//spike_poi thread zm_utility::wait_for_attractor_positions_complete();
		array::add(level.spike_pois, spike_poi);
	}
}

//Call On: The spawned bolt/spike
//Returns a valid poi location
//If no valid location found, returns (0, 0, 0)
function move_valid_poi_to_navmesh(b_valid_poi){
	v_valid_poi = (0, 0, 0);
	if(IsPointOnNavMesh(self.origin)){
		v_valid_poi = self.origin;
	}else{
		//Find results on the nav mesh to make POI
		query_result = PositionQuery_Source_Navigation(
			self.origin, 0,
			POI_MAX_RADIUS,
			POI_HALF_HEIGHT,
			POI_INNER_SPACING,
			POI_RADIUS_FROM_EDGES
		);
		if(query_result.data.size > 0){
			foreach(point in query_result.data){
				//Check not too far off
				height_delta = Abs(self.origin[2] - point.origin[2]);
				if(!(height_delta > POI_HEIGHT)){
					//Do bullet trace to make sure not going between walls
					start = point.origin + (0, 0, 20);
					end = self.origin + (0, 0, 20);
					if(BulletTracePassed(start, end, false, self, undefined, false, false)){
						//make the poi this valid location
						v_valid_poi = point.origin;
						break; //end the loop as point found
					}
				}
			}
		}
	}
	return v_valid_poi;
}

//Call On: The spawned bolt/spike
function endSpikeAttractionOnDeath(attacker, weapon, target){
	self thread weaponobjects::spikeDetonate(attacker, weapon, target);
	foreach(poi in level.spike_pois){
		//NOT WORKING AT THE MOMENT
		poi zm_utility::deactivate_zombie_point_of_interest();
		poi notify("death");
		poi Delete();
	}
	self notify("death");
	level.spike_pois = [];
}

//Call On: The spawned bolt
function spikeWatcher(watcher, owner){
	IPrintLn("spike_fired");
	self thread detonateAfterTime(SPIKE_CHARGE_TIME, owner);
}

//Call On: spawned spike
function detonateAfterTime(time, player){
	player endon("detonate");
	IPrintLn("called");
	self util::waitTillNotMoving();
	wait(time);
	IPrintLn("detonate");
	//self endSpikeAttractionOnDeath();
	player notify("detonate");
}
```

</details>

#### Hello World in GSC

[`View this example separately`](/GSC/HelloWorld/HelloWorld_InGSC.gsc)

This is how you make a Hello World program in GSC:

```gsc
weapon = "Hello, World!";
```

_/!\ This example has not been tested yet, and may not work_ !-->

#### Comments in GSC

Comments in GSC are the same as in languages like C, C++, Java, etc.

##### Single line comments

[`View this example separately`](/GSC/Comments/Single-line/Single-lineComments_InGSC_V1.gsc)

Single line comments in GSC are written like so:

```gsc
// This is a single line comment
```

##### Multi-line comments

[`View this example separately`](/GSC/Comments/Multi-line/Multi-lineComments_InGSC.gsc)

Multi-line comments in GSC are written like so:

```gsc
/* This is
a multi-line
comment
*/
/* This is
* also a
* multi-line
* comment */
```

#### Break keyword in GSC

[`View this example separately`](/GSC/Break-Keyword/BreakKeyword_InGSC.gsc)

GSC supports the `break` keyword:

```gsc
break;
```

To this day, I am still not entirely sure what the `break` keyword does, but most languages support it.

_/!\ This example has not been tested yet, and may not work_

#### Functions in GSC

[`View this example separately`](/GSC/Functions/Functions_InGSC.gsc)

Functions are defined in GSC like so:

```gsc
function myFunction(){
    break;
}
```

_/!\ This example has not been tested yet, and may not work_

#### Returning a function in GSC

[`View this example separately`](/GSC/Functions/ReturningAFunction/Returning_a_Function_InGSC.gsc)

I think this is how you return a function in GSC:

```gsc
function testFunc(){
    break;
    return 1;
}
return testFunc();
```

_/!\ This example has not been tested yet, and may not work_

#### Defining a weapon

[`View this example separately`](/GSC/DefineWeapon/Defining_a_Weapon_InGSC.gsc)

Defining a Call of Duty weapon in GSC can be done like so:

```gsc
function spikeLauncherTutorialHUD(){
	self notify("spike_launcher_HUD");
	self endon("spike_launcher_HUD");
}
return spikeLauncherTutorialHUD();
```

_/!\ This example has not been tested yet, and may not work_

##### Setting an alternative weapon name

[`View this example separately`](/GSC/DefineWeapon/AlternativeName/SettingAnAlternativeWeaponName_InGSC.gsc)

To set an alternative name for a weapon in GSC, you can do this:

```gsc
function spikeLauncherTutorialHUD(){
	self notify("spike_launcher_HUD");
	self endon("spike_launcher_HUD");
	watcher.altName = "spike_charge_upgraded";
}
return spikeLauncherTutorialHUD();
```

_/!\ This example has not been tested yet, and may not work_

##### Setting a shock duration

[`View this example separately`](/GSC/DefineWeapon/ShockTimer/SettingAShockTimer_InGSC.gsc)

Setting a shock duration to a weapons attack in GSC can be done like so:

```gsc
function spikeLauncherTutorialHUD(){
	watcher.stunTime = 1;
}
return spikeLauncherTutorialHUD();
```

_/!\ This example has not been tested yet, and may not work_

##### Fading the description

[`View this example separately`](/GSC/DefineWeapon/FadeDescription/Fade_the_Description_InGSC.gsc)

To fade the description over a duration of time in GSC, one must do it like so:

```gsc
txt FadeOverTime(0.5);
```

_/!\ This example has not been tested yet, and may not work_

##### End at a players death

[`View this example separately`](/GSC/DefineWeapon/EndAtDeath/EndAtDeath_InGSC.gsc)

To end the function after the player character dies, this script can be used:

```gsc
self endon("death");
```

_/!\ This example has not been tested yet, and may not work_

##### Wait until the current weapon is fired

[`View this example separately`](/GSC/DefineWeapon/WaitUntilFired/WaitUntilWeaponIsFired_InGSC.gsc)

To wait until the current weapon is fired before going to the next function, this script can be used:

```gsc
// w_current is the current weapon
self waittill("weapon_fired", w_current);
```

_/!\ This example has not been tested yet, and may not work_

##### Complete the tutorial

[`View this example separately`](/GSC/DefineWeapon/CompleteWeaponTutorial/CompleteTheWeaponTutorial_InGSC.gsc)

To complete the weapon usage tutorial, this script can be used:

```gsc
self.spike_launcher_tutorial_complete = true;
```

_/!\ This example has not been tested yet, and may not work_

#### Wait keyword in GSC

[`View this example separately`](/GSC/Wait-Keyword/WaitKeyword_InGSC.gsc)

GSC supports the `wait` keyword.

```gsc
wait(1.0);
break;
```

_/!\ This example has not been tested yet, and may not work_

#### Setting up the font

[`View this example separately`](/GSC/DefaultFont/SetFontToDefault_InGSC.gsc)

To default the font to the game default, this script is used:

```gsc
font = "default";
```

_/!\ This example has not been tested yet, and may not work_

#### Change the font scale

[`View this example separately`](/GSC/ChangeFontScale/ChangeFontScale_InGSC.gsc)

To change the fontscale of text, this script can be used:

```gsc
fontscale = 2;
```

_/!\ This example has not been tested yet, and may not work_


#### If statements in GSC

[`View this example separately`](/GSC/IfStatements/IfStatements_InGSC.gsc)

`If` statements can be defined in GSC like so:

```gsc
if true {
    fontscale = 2;
    break;
}
```

_/!\ This example has not been tested yet, and may not work_

#### If else statements in GSC

[`View this example separately`](/GSC/IfElseStatements/IfElse_Statements_InGSC.gsc)

`If else` statements can be defined in GSC like so:

```gsc
if true {
    fontscale = 2;
    break;
}else{
    fontscale = 3;
    break;
}
```

_/!\ This example has not been tested yet, and may not work_

#### While statements in GSC

[`View this example separately`](/GSC/WhileStatement/WhileStatement_InGSC.gsc)

`While` statements can be defined in GSC like so:

```gsc
while(true){
    fontscale = 2;   
}
```

_/!\ This example has not been tested yet, and may not work_

#### Other knowledge of the GSC programming language

1. GSC is a language for the Call of Duty series of video games

2. GSC is a semicolon and curly bracket language

3. GSC has syntax very similar to that of C, Java, and/or another language

4. GSC uses the `*.gsc` file extension by default, but also uses the `*.gsh` and `*.csc` file extensions

5. The `*.gsh` file extension can be confused with a Groovy programming language program.

7. GSC is not one of the top 50 programming languages (as of 2022, July 31st, it has never ranked 50 or higher on the TIOBE index)

8. GSC is a language recognized by GitHub, with the color `orange`

9. No other knowledge of the GSC programming language

#### Additional comments

1. I discovered the GSC language by accident when making a Groovy source file with the extension `*.gsh` which according to Wikipedia was a Groovy file extension.

2. I currently don't know what GSC stands for

3. I currently don't know what year GSC was released on

4. No other additional comments available

***

## File info

**File type:** `Markdown document (*.md *.mkd *.mdown *.markdown)`

**File version:** `1 (2022, Wednesday, August 24th at 4:14 pm PST)`

**Line count (including blank lines and compiler line):** `661`

***

## File history

<details><summary><p>Click/tap here to expand/collapse the history for this file</p></summary>

<details><summary><p><b>Version 1 (2022, Wednesday, August 24th at 4:14 pm PST)</b></p></summary>

> Changes:

> * Started the file

> * Added the `title` section

> * Added the `Hello World in GSC` section

> * Added the `Comments in GSC` section

> > * Added the `Single line comments` subsection

> > * Added the `Multi-line comments` subsection

> * Added the `break keyword in GSC` section

> * Added the `Functions in GSC` section

> * Added the `Returning a function in GSC` section

> * Added the `Defining a weapon` section

> > * Added the `Setting an alternative weapon name` subsection

> > * Added the `Setting a shock duration` subsection

> > * Added the `Fading the description` subsection

> > * Added the `End at a players death` subsection

> > * Added the `Complete the tutorial` subsection

> * Added the `Wait keyword in GSC` section

> * Added the `Setting up the font` section

> * Added the `Change the font scale` section

> * Added the `If statements in GSC` section

> * Added the `If else statements in GSC` section

> * Added the `While statements in GSC` section

> * Added the `other knowledge of the GSC programming language` section

> * Added the `Additional comments` section

> * Added the `file info` section

> * Added the `file history` section

> * No other changes in version 1

</details>

</details>

***
