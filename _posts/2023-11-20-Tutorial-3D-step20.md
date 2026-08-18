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

In the PhysicsWorld class we update the physics.  The physics library ODE does not work well with a variable time step.  You can however use the delta time,
to ensure you run an update cycle at the desired rate. (Also update `World.update()` to pass the new parameter `deltaTime`).

PhysicsWorld.java:

```java
    static final float TIME_STEP = 0.025f;  // fixed physics time step

    // update the physics
    // time step of quickStep() needs to be fixed size
    //
    public void update(float deltaTime) {
            timeElapsed += deltaTime;
            while(timeElapsed > TIME_STEP) {
                space.collide(null, nearCallback);
                world.quickStep(TIME_STEP);
                contactGroup.empty();
    
                timeElapsed -= TIME_STEP;
            }
        }
```

Then we notice the enemies are moving much too fast. Where we are applying force to the enemy rigid bodies, we need to scale this with the time step.
Change as follows (you could also change `Settings.cookForce` to get rid of this 60f constant) :

CookBehaviour.java:

```java
  @Override
    public void update(World world, float deltaTime ) {
        ...

            if(distance > 5f)   // move unless quite close
                go.body.applyForce(targetDirection.scl(deltaTime * 60f * Settings.cookForce * climbFactor));
        }
```

Then some changes are needed to the PlayerController, because the jumps are too high and the view rotation with the mouse is too slow.
The jump force also needs to be scaled with deltaTime.  The mouse movement was scaled by deltaTime but in fact it shouldn't be. The amount
of rotation is directly determined by how much the mouse has moved.


PlayerController:

```java
    public void update (GameObject player, float deltaTime ) {
        ...

        // mouse to move view direction
        rotateView(mouseDeltaX*Settings.turnSpeed/60f, mouseDeltaY*Settings.turnSpeed/60f ); <---- removed deltaTime
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
        delta *= deltaTime*Settings.verticalReadjustSpeed*speedFactor;
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
            linearForce.y =  Settings.jumpForce * deltaTime * 60f;              <---- change

        linearForce.scl(120);
        player.body.applyForce(linearForce);
        // note: as the player body is a capsule it is not necessary to rotate it
        // (and in fact it causes problems due to errors building up)
        // so we don't rotate the rigid body, but we rotate the modelInstance in World.syncToPhysics()
    }
```

Then in GameView the `render()` method needs a small update.  This is to make sure that when the game view is used for an overlay (i.e. the gun view), the camera is reset
to the same Y position each frame before adding the bobbing effect.  Otherwise, the gun will just drift away.  It is an effect that only becomes noticeable with very small
deltaTime values.

```java
    public void render(float delta, float speed ) {
        if(!isOverlay)
            camController.update(world.getPlayer().getPosition(), world.getPlayerController().getViewingDirection());
        else                                                    <-----   add
            cam.position.y = Settings.eyeHeight;                <------- add
        addHeadBob(delta, speed);
        cam.update();
        refresh();
        sceneManager.update(delta);

        Gdx.gl.glClear(GL20.GL_DEPTH_BUFFER_BIT);   // clear depth buffer only
        sceneManager.render();
    }

    private void addHeadBob(float deltaTime, float speed ) {
        if( speed > 0.1f ) {
        bobAngle += speed * deltaTime * Math.PI / Settings.headBobDuration;

        // move the head up and down in a sine wave
        cam.position.y +=  bobScale * Settings.headBobHeight * (float)Math.sin(bobAngle);
        }
    }
```

These are the steps needed to have the game run the same, regardless of which frame rate you are using, so that different users get the same experience.
If you disabled the vsync(), don't forget to re-enable it after testing.


This concludes step 20 where we adapted the code for different frame rates.


