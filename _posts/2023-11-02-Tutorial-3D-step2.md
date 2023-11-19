# 3D Tutorial - Step 2 - Camera Control
by Monstrous Software


# Step 2 - Camera Control

In the previous step we rendered a 3d scene consisting of one square block to represent the ground.  We could move the camera around the origin, to view it from different angles, 
but we couldn't walk around.

In this step we're going to add an FPS camera controller that allows us to move around in the game world.

As a first step, let's replace CameraInputController by FirstPersonCameraController which is provided as standard in LibGDX.
Change the camController declaration to the following:
```java
        private FirstPersonCameraController camController;
```
And change the initialisation in show() to the following:
```java
        camController = new FirstPersonCameraController(cam);
```
Add the following to show() to catch the mouse cursor. 
```java
        // hide the mouse cursor and fix it to screen centre, so it doesn't go out the window canvas
        Gdx.input.setCursorCatched(true);
        Gdx.input.setCursorPosition(Gdx.graphics.getWidth() / 2, Gdx.graphics.getHeight() / 2);
```
At this point it is necessary to allow a key press to exit the game, because it 
is now impossible to click on the window's close button. We'll use the Escape key to close the app.
```java
        if (Gdx.input.isKeyJustPressed(Input.Keys.ESCAPE))
            Gdx.app.exit();
```
If you run this now in the desktop version, you'll notice that you can look around with the mouse if you hold the left mouse 
button down to do this.
However, when we move forward (press A), we go in the direction we're looking even if we're looking up into the sky.
We'd like to restrict the movement to the horizontal plane, even if we're looking up or down.
Furthermore, we will add some head bobbing to the camera.

We'll add a Settings class to hold some general game settings via public static members.  This will make it easy to access from anywhere in the game and easy to modify.
For example:
```java
        public class Settings {
            static public float eyeHeight = 1.5f;   // meters
        }
```
Let us change the camera positioning in GameScreen show() to make use of this setting:
```java
        cam.position.set(10f, Settings.eyeHeight, 5f);
        cam.lookAt(0,Settings.eyeHeight,0);
```
## Replace FirstPersonCameraController by our own CamController.

Let us make our own camera controller.  Define a class which extends InputAdapter.
InputAdapter is a subclass from InputProcessor which implements all the methods with 
a default implementation. This means we only need to override the methods we want to change 
and our code can be a bit shorter than if we implement InputProcessor.

```java
        public class CamController extends InputAdapter {
        }
```
Let us define a few fields for the key bindings we will use:
```java
            public int forwardKey = Input.Keys.W;
            public int backwardKey = Input.Keys.S;
            public int strafeLeftKey = Input.Keys.A;
            public int strafeRightKey = Input.Keys.D;
            public int turnLeftKey = Input.Keys.Q;
            public int turnRightKey = Input.Keys.E;
            public int runShiftKey = Input.Keys.SHIFT_LEFT;
```
Because we will use this class to control the camera, we will keep a reference to the camera which
is passed as a parameter to the constructor:
```java
            protected final Camera camera;

            public CamController(Camera camera) {
                this.camera = camera;
            }
```
To keep track of which keys are pressed, we override the keyUp() and keyDown() methods and we store the key 
state in a map, adding the keycode to the map when the key is pressed down and removing it from the map when 
the key is released. We can then look up the key state by checking if the keycode appears in the map.
```java
            protected final IntIntMap keys = new IntIntMap();

            @Override
            public boolean keyDown (int keycode) {
                keys.put(keycode, keycode);
                return true;
            }
        
            @Override
            public boolean keyUp (int keycode) {
                keys.remove(keycode, 0);
                return true;
            }
```
Let's take a first shot at the update() method.  This will be called once per frame from the GameScreen render() 
method and it will move the camera depending on which keys are held down at that moment.

First add the following fields to the Settings class because we will want to tweak these values easily:
```java
            static public float walkSpeed = 5f;    // m/s
            static public float runFactor = 3f;    // m/s
            static public float turnSpeed = 120f;   // degrees/s
```
Then add an update() method. Depending on which keys are down, it calls moveForward(), strafe() or rotateView() 
and then updates the camera. We will write these methods next.
```java
            public void update (float deltaTime) {

                float moveSpeed = Settings.walkSpeed;
                if(keys.containsKey(runShiftKey))       // go faster if SHIFT is held down
                    moveSpeed *= Settings.runFactor;
        
                if (keys.containsKey(forwardKey)) 
                    moveForward(deltaTime * moveSpeed);
                
                if (keys.containsKey(backwardKey)) 
                    moveForward(-deltaTime * moveSpeed);
                
                if (keys.containsKey(strafeLeftKey)) 
                    strafe(-deltaTime * Settings.walkSpeed);
                
                if (keys.containsKey(strafeRightKey)) 
                    strafe(deltaTime * Settings.walkSpeed);
                
                if (keys.containsKey(turnLeftKey)) 
                    rotateView(deltaTime*Settings.turnSpeed );
                else if (keys.containsKey(turnRightKey)) 
                    rotateView(-deltaTime*Settings.turnSpeed );
                    
                camera.update(true);
            }
```
For the moveForward() method, we add a temporary Vector3 variable to hold the movement vector.
It is set to the camera's direction.  This is the vector indicating in three dimensions where 
the camera is pointing. Then we zero the Y component to project the vector onto the horizontal plane. 

> In LibGDX the convention is that the Y-axis points upwards, 
> but this is not a standard convention everywhere.  
> For example, in Blender, Z is the up axis.  
> Also in the physics library ODE4j, which we will use later, we will use Z to point upwards.    
 
Then we normalize it to make it a unit length again and scale it with the distance we want to move in this update step.  
Finally, we add this vector to the camera position to move it forward (or backwards if direction has a negative value).

```java
            protected final Vector3 fwdHorizontal = new Vector3();

            private void moveForward( float distance ){
                fwdHorizontal.set(camera.direction).y = 0;
                fwdHorizontal.nor();
                fwdHorizontal.scl(distance);
                camera.position.add(fwdHorizontal);
            }
```

For the strafe() method we'll need an additional Vector3. We define these temporary vectors as field members
of the class, rather than local variables to save on garbage collection in these methods that will be called very often.
In the strafe() method, we calculate a forward vector in the horizontal plane in the same way as before 
(projecting the camera direction vector to the XZ plane). Then we determine a cross product of that vector with the camera's up vector to produce a vector
which is orthogonal to the forward and up vectors.  
This gives us the sideways vector that we use for strafing.
```java
            protected final Vector3 tmp = new Vector3();

            private void strafe( float distance ){
                fwdHorizontal.set(camera.direction).y = 0;
                fwdHorizontal.nor();
                tmp.set(fwdHorizontal).crs(camera.up).nor().scl(distance);
                camera.position.add(tmp);
            }
```
Note in the code how vector operations can be concatenated easily to perform multiple operations one after the other:
```java
        tmp.set(fwdHorizontal).crs(camera.up).nor().scl(distance);
```
means:
- set `tmp` vector to the value of `fwdHorizontal`
- then calculate the cross-product of that vector with `camera.up`
- normalize the result to a unit vector, i.e. a vector with length 1
- scale it to length `distance`

This technique is called 'method chaining'.

The next method is not for camera movement, but for camera rotation.  We rotate the camera's direction vector around the camera's up vector.
We also set the camera up vector to the Y axis to make sure the camera always remains upright.
```java
            private void rotateView(float deltaX) {
                camera.direction.rotate(camera.up, deltaX);
                camera.up.set(Vector3.Y);
            }
```
The `rotate()` method on a vector that we use here takes a rotation axis (the camera's up vector) and a rotation angle in degrees. This means we turn the vector left or right.

In the GameScreen class replace `FirstPersonCameraController` by our new `CamController` and in the render() method pass the frame delta to the
camera controller's update method:
```java
        // update
        camController.update(Gdx.graphics.getDeltaTime());
```
At this point the code should compile, and you can guide the camera around with the WASD keys, 
and turn with the Q and E keys.

[ Download step-02a from the code repository to compare ]

Next we'll add support to look around with the mouse.

Add the following settings to the Settings class:
```java
        static public boolean invertLook = false;
        static public boolean freeLook = true;
```
Now add a method called mouseMoved() to CamController.
```java
        protected float degreesPerPixel = 0.1f;

        @Override
        public boolean mouseMoved(int screenX, int screenY) {
            // ignore big delta jump on start up
            if(Gdx.input.getDeltaX() == screenX && Gdx.input.getDeltaY() == screenY)        
                return true;
    
            float deltaX = -Gdx.input.getDeltaX() * degreesPerPixel;
            float deltaY = -Gdx.input.getDeltaY() * degreesPerPixel;
            if (Settings.invertLook)
                deltaY = -deltaY;
            if (!Settings.freeLook) {    // keep camera movement in the horizontal plane
                deltaY = 0;
                camera.direction.y = 0;
            }
            rotateView(deltaX, deltaY);
            return true;
        }
```
We will add a method rotateView() that is extended to take two angles: one angle for rotation around the up axis
and another angle to rotate the camera up and down.  We use two fields from the Settings class to allow user preferences on the 
mouse behaviour. Settings.invertLook swaps the up and down direction for people who prefer that. The Settings.freeLook field
determines if we can even look up and down at all.  These type of settings could perhaps be enabled or disabled in an options menu 
in your game.

A first implementation for rotateView() is as follows. First we rotate around the camera up vector. Then we determine a sideways vector using the cross product from up and forward, as we did before for strafing. 
And then we rotate around the sideways vector using the second angle to turn the camera upwards or downwards.
```java
        private void rotateView(float deltaX, float deltaY) {
            camera.direction.rotate(camera.up, deltaX);
            tmp.set(camera.direction).crs(camera.up).nor();
            camera.direction.rotate(tmp, deltaY);
        }
```
It turns out this can give a nasty spinning effect when you're looking straight up or straight down, because of an effect called "gimbal lock".
Because the `camera.up` and `camera.direction` vectors are very close together, the cross product jitters to the right or to the left of the camera.

Below is a version which avoids this gimbal lock by making sure to ignore rotations which would flip this pitch axis.

```java
    private final Vector3 tmp2 = new Vector3();
    private final Vector3 tmp3 = new Vector3();

        private void rotateView(float deltaX, float deltaY) {
            camera.direction.rotate(camera.up, deltaX);

            // avoid gimbal lock when looking straight up or down
            Vector3 oldPitchAxis = tmp.set(camera.direction).crs(camera.up).nor();
            Vector3 newDirection = tmp2.set(camera.direction).rotate(tmp, deltaY);
            Vector3 newPitchAxis = tmp3.set(tmp2).crs(camera.up);
            if (!newPitchAxis.hasOppositeDirection(oldPitchAxis))
                camera.direction.set(newDirection);
        }
```

## Head Bob

To sell the walking effect, it can be nice to add a bit of head bobbing to the camera.
We can simulate the head bobbing by adding some extra height to the camera using a sine wave.
The input to the sine wave is determined by the walking or running speed.  If the player is not moving, there
should be no head bobbing.
```java
            private float bobAngle = 0;

            private float bobHeight(float speed, float deltaTime ) {
                if(Math.abs(speed) < 0.1f )
                    return 0f;
                bobAngle += deltaTime * speed * 0.5f * Math.PI / Settings.headBobDuration;
                // move the head up and down in a sine wave
                return (float) (Settings.headBobHeight * Math.sin(bobAngle));
            }
```
To tweak this effect we add some new fields to the Settings class:
```java
            static public float headBobDuration = 0.6f; // s
            static public float headBobHeight = 0.04f;  // m
```
The bobHeight function is called at the end of the CamController's update() method and is used to add a value to the
camera's Y position. To calculate the head bobbing we need to keep track of the movement speed. 
We'll use a new variable called bobSpeed and set it when moving forward or back or strafing:
```java
            public void update (float deltaTime) {
    
                float bobSpeed= 0;
        
                float moveSpeed = Settings.walkSpeed;
                if(keys.containsKey(runShiftKey))
                    moveSpeed *= Settings.runFactor;
        
                if (keys.containsKey(forwardKey)) {
                    moveForward(deltaTime * moveSpeed);
                    bobSpeed = moveSpeed;
                }
                if (keys.containsKey(backwardKey)) {
                    moveForward(-deltaTime * moveSpeed);
                    bobSpeed = moveSpeed;
                }
                if (keys.containsKey(strafeLeftKey)) {
                    strafe(-deltaTime * Settings.walkSpeed);
                    bobSpeed = Settings.walkSpeed;
                }
                if (keys.containsKey(strafeRightKey)) {
                    strafe(deltaTime * Settings.walkSpeed);
                    bobSpeed = Settings.walkSpeed;
                }
                if (keys.containsKey(turnLeftKey))
                    rotateView(deltaTime*Settings.turnSpeed );
                else if (keys.containsKey(turnRightKey))
                    rotateView(-deltaTime*Settings.turnSpeed );

                camera.position.y = Settings.eyeHeight + bobHeight( bobSpeed, deltaTime); // apply some head bob if we're moving
                camera.update(true);
            }
```

## Jumping

Finally, we'll add a jumping effect when the player pressed the space bar.  
The camera will make a little hop in the air. To simulate the jump we'll use some very basic physics.
We'll add a new class field for the jump height, which will be added to the normal camera height.
We'll add a new class field for the jump velocity which determines how much the jump height changes per step.
The jump velocity will be affected by a gravity constant. 
To make sure the player cannot jump while we are already in the air, we add a boolean variable 'isJumping'.
The jump movement ends when the jump height reaches zero again, as for now we're only jumping on a flat plane.

New Settings field:
```java
        static public float gravity = -9.8f; // meters / s^2
```
New CamController fields:
```java
        public int jumpKey = Input.Keys.SPACE;

        private boolean isJumping = false;  // are we jumping?
        private float jumpH = 0;        // jump height
        private float jumpV = 0;        // jump velocity
```
Additional code in update():
```java
        if (keys.containsKey(jumpKey) && !isJumping) {
            isJumping = true;
            jumpV = 5;                  // initial upward velocity
        }

        if(isJumping){
            bobSpeed = 0;               // no head bob while jumping
            jumpV += deltaTime*Settings.gravity;      // deceleration due to gravity
            jumpH += jumpV*deltaTime;   // update jump height from velocity
            if(jumpH < 0){              // if jump height is zero, stop the jump
                isJumping = false;
                jumpH = 0;
            }
        }
```
And change the line which sets the camera height to include `jumpH`:
```java
       camera.position.y = Settings.eyeHeight + jumpH + bobHeight( bobSpeed, deltaTime); // apply some head bob if we're moving
```


## Conclusion

This concludes step 2. We've added a first person camera controller with forward and backward movement, strafing, mouse look and head bob.
