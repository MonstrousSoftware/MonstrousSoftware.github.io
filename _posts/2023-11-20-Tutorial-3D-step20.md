# 3D Tutorial - Step 20 - Frame Rate independence
by Monstrous Software


# Step 20 - Frame rate independence

This step is in fact a bug fix, to allow the game to run at different frame rates.  Up to now the game runs fine at 60Hz, but at a higher frame
rate (perhaps because your monitor runs at 120 Hz) some things go too fast. Lesson learnt: always test this before shipping, other people have other computers than you do.

To try this out, you can edit `Lwjgl3Launcher.java` in the lwjgl3 folder.  This is the launcher code used for the "Desktop" version.
The pre-generated comments already give a hint what to do:  change `useVsync(true)` to `useVsync(false)` and comment out the line
with `setForegroundFPS()`.  Now you will get a frame rate that is not tied to your monitor frequency but which depends on your computer and your graphics card.
Maybe now you get 1000 frames per second, because we are rendering quite a simple scene.  Although this is useful for performance testing, you should
normally leave vsync on; uncapping the frame rate just stresses the hardware and runs down laptop batteries for no visible benefit.

Lwjgl3Launcher.java:

```java
    private static Lwjgl3ApplicationConfiguration getDefaultConfiguration() {
        Lwjgl3ApplicationConfiguration configuration = new Lwjgl3ApplicationConfiguration();
        configuration.setTitle("Tut3D");
        configuration.useVsync(true);                                                                       <--- change to false
        //// Limits FPS to the refresh rate of the currently active monitor.
        configuration.setForegroundFPS(Lwjgl3ApplicationConfiguration.getDisplayMode().refreshRate);        <--- comment out
        //// If you remove the above line and set Vsync to false, you can get unlimited FPS, which can be
        //// useful for testing performance, but can also be very stressful to some hardware.
        //// You may also need to configure GPU drivers to fully disable Vsync; this can cause screen tearing.

        configuration.setWindowedMode(1280, 720);
        configuration.setBackBufferConfig(8, 8, 8, 8, 16, 0, 4);

        configuration.setWindowIcon("libgdx128.png", "libgdx64.png", "libgdx32.png", "libgdx16.png");
        return configuration;
    }
```

Although in many places, we already used deltaTime to make game behaviour independent of frame rate, there were in fact still a few places
where it was not done properly.

Some changes are needed to the PlayerController, the view rotation with the mouse is too slow.
The mouse movement was scaled by deltaTime but in fact it shouldn't be. The amount
of rotation is directly determined by how much the mouse has moved.


PlayerController:

```java
    public void update (GameObject player, float deltaTime ) {
        ...

        // mouse to move view direction
        rotateView(mouseDeltaX*Settings.turnSpeed/60f, mouseDeltaY*Settings.turnSpeed/60f );    //<--- removed deltaTime
        mouseDeltaX = 0;
        mouseDeltaY = 0;
    
        // controller stick inputs
        moveForward(stickMove.y*deltaTime * moveSpeed);
        strafe(stickMove.x * deltaTime * Settings.walkSpeed);
        float delta = 0;
        float speedFactor;
        if(world.weaponState.scopeMode) {
            speedFactor = 0.2f;
            delta = (stickLook.y * 30f );
        }
        else {
            speedFactor = 1f;
            delta = (stickLook.y * 90f - stickViewAngle);
        }
        delta *= deltaTime*4f*speedFactor;
        stickViewAngle += delta;
        rotateView(stickLook.x * deltaTime * Settings.turnSpeed*speedFactor,  delta );
        
        // note: most of the following is only valid when on ground, but we leave it to allow some fun cheating
        if (keys.containsKey(forwardKey))
            moveForward(deltaTime * moveSpeed);
        if (keys.containsKey(backwardKey))
            moveForward(-deltaTime * moveSpeed);
        if (keys.containsKey(strafeLeftKey))
            strafe(-deltaTime * Settings.walkSpeed);
        if (keys.containsKey(strafeRightKey))
            strafe(deltaTime * Settings.walkSpeed);
        if (keys.containsKey(turnLeftKey))
            rotateView(deltaTime * Settings.turnSpeed, 0);
        if (keys.containsKey(turnRightKey))
            rotateView(-deltaTime * Settings.turnSpeed, 0);
    
        if (isOnGround && keys.containsKey(jumpKey) )
            linearForce.y =  deltaTime * Settings.jumpForce;
    
        linearForce.scl(500);
        player.body.applyForce(linearForce);
        // note: as the player body is a capsule it is not necessary to rotate it
        // (and in fact it causes problems due to errors building up)
        // so we don't rotate the rigid body, but we rotate the modelInstance in World.syncToPhysics()
    }

```

Then in GameView the `render()` method needs a small update.  This is to make sure that the game view that is used for an overlay (i.e. the gun view), the camera is reset
to the same Y position each frame before adding the bobbing effect.  Otherwise, the gun will just drift away.  It is an effect that only becomes noticeable with very small
deltaTime values.

```java
    @Override
    public void render(float delta) {
            //...

            if(!thirdPersonView && world.weaponState.currentWeaponType == WeaponType.GUN &&!lookThroughScope) {
                gunView.getCamera().position.y = Settings.eyeHeight;    // reset camera height
                gunView.render(delta, moveSpeed);
            }
            //...
    }
```

As the gun's bobbing motion is now barely visible, adjust the scale of it when creating the gun view in `GameScreen#show`:

```java
    @Override
    public void show() {
        //...
    
        // create an overlay view and add gun model
        gunView = new GameView(gunWorld, true, 0.01f, 10f, 1.0f); //last param (bobScale) was 0.1f
    }   
```

These are the steps needed to have the game run the same, regardless of which frame rate you are using, so that different users get the same experience.
If you disabled the vsync(), don't forget to re-enable it after testing.


This concludes step 20 where we adapted the code for different frame rates.


