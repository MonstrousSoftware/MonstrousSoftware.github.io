## Health Bars

This coding example is to introduce health bars in a 3d game.

![image](https://user-images.githubusercontent.com/49096535/231277885-d14506f5-dff2-447e-92cd-290317dfc845.png)
> Health bars in action


The following code creates a texture for the health bar: A black frame around a grey box overlapped by coloured health bar extending from the right. 
Different colours are used depending on the remaining health.

![image](https://user-images.githubusercontent.com/49096535/231278616-1d86215d-4c2b-47ed-bbaa-509752d9ca32.png)
> A health bar texture up close

```java
    private Texture makeBarTexture(int width, int height, float health, Color color) {
        Pixmap pixmap = new Pixmap(width+4, height+2, Pixmap.Format.RGBA8888);
        pixmap.setColor(Color.BLACK);
        pixmap.fill();
        pixmap.setColor(Color.GRAY);
        pixmap.fillRectangle(2,1, width, height);
        pixmap.setColor(color);
        int w = MathUtils.round((float)width*health);
        if(w > 0)
            pixmap.fillRectangle(2,1, w, height);
        return new Texture(pixmap);
    }
```

To avoid creating the textures dynamically, an array of texture regions is created on startup for different levels of health. During the render cycle we just have to pick 
the one matching the remaining health.  

One approach is to render the health bars as sprites using SpriteBatch as an overlay after rendering the 3d scene. 
The problem with this approach that the bars are not subject to depth testing.
The bars will appear even when they are supposed to be obscured by something in the foreground.  Another issue is that the health bars don't scale with distance.

A better approach for a 3d game is to use decals.   These are textures that have a location in the 3d environment and that can be rotated to face the camera (also known as billboarding).

```java
   public void show(Array<Creature> creatures) {
        for(Creature creature : creatures) {
            if(creature.healthBarDecal == null) {
                creature.healthBarDecal = Decal.newDecal(selectTexture(creature.health));
                creature.healthBarDecal.setDimensions(2f, 0.4f);                    // world units
            }else
                creature.healthBarDecal.setTextureRegion(selectTexture(creature.health));
        }

        viewDir.set(cam.direction).scl(-1);    // direction that decals should be facing: opposite of camera view vector
        for(Creature creature : creatures) {
            creature.transform.getTranslation(position);
            position.y += Y_OFFSET;     // place health bar above the creature
            creature.healthBarDecal.setPosition(position);
            creature.healthBarDecal.setRotation(viewDir, Vector3.Y);
            decalBatch.add(creature.healthBarDecal);
        }
        decalBatch.flush();
    }
```

Note how we don't point the decals directly at the camera (i.e. we don't use `decal.lookAt(camera.position, camera.up)` as recommended by the LibGDX documentation.)
We get a better effect by pointing the decal towards the camera's near plane. Or in other words, to have the normal vector of the decal in the opposite direction as the camera's view vector.
All the decals are oriented in the same direction.


![image](https://user-images.githubusercontent.com/49096535/231281328-86d8435f-7ce2-40fd-b4c2-abe8893711c9.png)
> Pointing decals directly at the camera, skews the rectangles and causes jagged edges  

As usual, code is available:

[Source code](https://github.com/MonstrousSoftware/HealthBars)

