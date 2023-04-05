## Dev Log ‘War of The Pixels’

This game was developed for the LibGDX game jam in March 2023 with as theme ‘pixel wars’.

![wotp](https://user-images.githubusercontent.com/49096535/230102813-61f142fe-dfa5-41d9-9565-cb0f7453378a.png)



It is a very simplified version of a real-time strategy game. The airships were clearly inspired by the Kirovs from Red Alert.

Originally there was going to be a clearer link to the theme of pixel wars, by using the amount of occupied territory to determine how much money you could make.  The minimap was supposed to show squares (“pixels”) occupied by red or by blue.  Moving the flag would allow you to capture more land, but at a risk of endangering the flag.  The money you made could be used to by more units.

It goes without saying that all this had to be cut in the interest of finishing in a week.  The current version gives you a fixed set of units and you’ll just have to make do with that.

### Models

The models were created in Blender. What helped with the work flow is to have all the models in the same blender file, all at the origin. 


![wotp1](https://user-images.githubusercontent.com/49096535/230103288-60aff8b8-4017-42af-9854-7e819f1140e6.png)

> All assets at the same position


In Blender you can hide the models you are not working on.  Then to export as FBX file, just select the whole collection, use scale 0.01 and export the meshes.  The fbx-convert tool converts it to a single g3db file which is loaded by the game.   In LibGDX terminology the different units are nodes of the same model. They are loaded using:

```java 
ModelInstance modelInstance = new ModelInstance(model, "Tank");
```

where “Tank” is the node name and the name given to the object in Blender.

For every unit the first material as defined in Blender is the colour of the army, so blue units can be changes into red units by changing the first material.

```java 
Material mat = modelInstance.materials.get(0);
mat.clear();
mat.set(army.material);
```
 
One thing to keep in mind is that although all units are modelled relative to the same origin, the airships appear some distance above the origin.  For collision detection, we have to calculate a bounding box of the mesh and take into account the centre of the bounding box which is not the same as the origin.  Before this tank bullets would pass underneath an airship and somehow still hit them.

### Controls

Originally the user interface was more like a classic RTS.  The camera hovered over the battlefield.  You could pan across the terrain with WASD keys. Click on a unit and direct it to some position on the terrain by clicking on the terrain. Like a general moving chess pieces.

However, since I wanted this to also be playable from a web browser including from mobile phones, this became problematic as they is no keyboard input and in effect only one mouse button.

So I changed the controls so that the camera always follows the active unit. A mouse click on the terrain directs the unit towards that location. A mouse click on another of your units, switches to that unit. Dragging the mouse (or the finger on touch screens) rotates the camera around the active unit. A pinch gesture or the mouse wheel can be used to scroll in and out.  

It made the game play more hectic because the player doesn’t always have a good overview of the battlefield.  You may need to zoom out or turn the camera around to see what’s happening elsewhere.  Let’s just say it aims to replicate the confusion of the battle field.

### Terrain

The terrain is a simple heightfield based off an image file of Perlin noise.  I originally had the idea to add rivers in the terrain so you would have to place bridges to cross to the other side, but this was quickly abandoned as too fidgety.

I used a low, low resolution texture on the terrain, to remain in the pixel theme. 

I wanted to experiment with some vegetation and props on the terrain, so I randomly place a few hundred trees and stones. For some reason, I modelled three different stones but only a single tree. Yes, they are supposed to be stones, not [rabbit poos][1].

This many model instances really slowed down the HTML version, until I put the terrain and all the scenery in a ModelCache to reduce the number of draw calls to the GPU.  This had the added benefit that the trees and stones are no longer needed in the gameObjects array and therefore invisible to collision detection.  Where we iterate to update all the game objects, we don’t have to waste processing cycles on bits of scenery.

### AI

The enemy’s AI is very basic. After 20 seconds, and every 5 seconds thereafter, it will make an inventory of the friendly and enemy units.  Then it will assign the first unit to defend its flag and the other units to randomly attack an enemy structure.  In either case, the instruction is simply for the unit to go towards the relevant location. All units automatically fire on any enemy within range.

A flaw in the AI is that it doesn’t recognize that once an airship has dropped its bomb, it should return to the tower to pick up a new one. Clever players can exploit this by tempting airships to drop their bomb early and hence becoming harmless.


### Voice responses

The voice responses were generated with an free online [robot voice generator][2].

> 

Some online sites had more realistic voices, but, alas, the ones I checked were not free.



### Particle effects

A 3d particle effect was added for fire and smoke when a unit or structure gets destroyed and for the explosion when a bomb drops.

One thing to look out for when using ParticleSystem is that it doesn’t stop and dispose particle effects once they have finished.  My update code checks for effects.isComplete() and then disposes it and removes it from the particle system. 

First the particle effect had a “blocky” look to it, because the blending was incorrect.  I used the constructor of the PointSpriteParticleBatch to define a better BlendingAttribute `(src_color, 1-src_alpha)` which seems to work.





[1]: https://www.youtube.com/watch?v=em0cy5iPmpg&t=5548s&ab_channel=Raeleus "libGDX Jam March 2023 Review"
[2]: https://lingojam.com/RobotVoiceGenerator "online robot voice generator"
