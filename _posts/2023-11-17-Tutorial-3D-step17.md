# 3D Tutorial - Step 17 - Head Bobbing
by Monstrous Software


# Step 17 - Head Bobbing

Somewhere along the way we lost the head bobbing effect that we added in step 2 of this tutorial.
Let us reintroduce this.  We add a method to the GameView class that calculates a bob height value which gets
added to the camera height in the render method.  The bob height value is calculated with a sine function.
If the player speed is close to zero the effect is disabled to avoid head bobbing when the player is standing
still.  The `render()` method now needs a new parameter to know the player speed. 

GameView:

```java
    public void render(float delta, float speed ) {
        if(!isOverlay)
            camController.update(world.getPlayer().getPosition(), world.getPlayerController().getViewingDirection());
        addHeadBob(delta, speed);
        cam.update();
        ...
    }

    private void addHeadBob(float deltaTime, float speed ) {
        if( speed > 0.1f ) {
            bobAngle += speed * deltaTime * Math.PI / Settings.headBobDuration;
            // move the head up and down in a sine wave
            cam.position.y +=  bobScale *  Settings.headBobHeight * (float)Math.sin(bobAngle);
        }
    }
```

In the GameScreen class we now have to calculate the player speed to pass it to the game view `render()` method.
The gun view will also apply a camera bobbing motion to simulate the player's hand going up and down while running.


GameScreen:

```java
    @Override
    public void render(float delta) {
        ...
        world.update(delta);

        float moveSpeed = world.getPlayer().body.getVelocity().len();
        gameView.render(delta, moveSpeed);
        if(debugRender) {
            gridView.render(gameView.getCamera());
            physicsView.render(gameView.getCamera());
        }

        if(!thirdPersonView && world.weaponState.currentWeaponType == WeaponType.GUN && !lookThroughScope) {
            gunView.render(delta, moveSpeed);
        }
        ...
    }
```

To calculate the player's velocity we add a new method to the PhysicsBody class which converts the linear velocity 
of the rigid body to a Vector3.

PhysicsBody:

```java
    public Vector3 getVelocity() {
        DVector3C v = geom.getBody().getLinearVel();
        linearVelocity.set((float)v.get0(), (float)v.get1(), (float)v.get2());
        return linearVelocity;
    }
```

This concludes step 17.