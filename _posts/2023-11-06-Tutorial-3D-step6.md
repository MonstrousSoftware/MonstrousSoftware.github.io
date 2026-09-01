#  3D Tutorial - Step 6 - Object placement
by Monstrous Software


# Step 6 - Object placement

## Reset node position

At the end of the previous step we updated the `spawnObject` method. It is time to make another little adjustment. As we've seen so far, when we read nodes (objects) from the GLTF file, in some cases
we want to leave the node position exactly where it appears in Blender.  This applies for example for walls, for the ground, in short mostly for the static objects that define the game level.
In other cases, we want to control the object position from our code. For example for player and enemy characters, for projectiles, etcetera.  For such objects it is best if they are centred on the world origin.
This will mean that in our modelling software we will have all such objects in the same position, making it very cluttered.

We can make it a bit more convenient by allowing such objects to be placed anywhere in the modelling software.  
As long as we use `apply transforms` with the object centred on the origin, we can then move object out of the way.
We will add an extra boolean parameter to `spawnObject` to indicate if we should use the node's current position from the GLTF file, or if we should reset its position when the transforms were applied.
This parameter will determine if the method `applyNodeTransform` will make use of the node transform or not.
Typically, for static objects we want to keep the node position and for dynamic objects we want to reset the node position.

Make the following changes in the World class to `spawnObject` and `applyNodeTransform`. This adds an extra parameter that indicates if we;
want to spawn the object at the position from the Blender file or at the position provided as another parameter.  This way we can do a lot of the level design
by positioning objects such as walls in the 3d editing tool, but we still can spawn bullets where we like.

```java
        public GameObject spawnObject(boolean isStatic, String name, CollisionShapeType shape, boolean resetPosition, Vector3 position, float mass){        //<-- new param
            Scene scene = new Scene(sceneAsset.scene, name);
            if(scene.modelInstance.nodes.size == 0)
                throw new RuntimeException("Cannot find node in GLTF file: " + name);
    
            applyNodeTransform(resetPosition, scene.modelInstance, scene.modelInstance.nodes.first());         //<-- new param
            ...
        }
    
        private void applyNodeTransform(boolean resetPosition, ModelInstance modelInstance, Node node ){            //<-- new param
            if(!resetPosition)                                                                                      //<-- new
                modelInstance.transform.mul(node.globalTransform);
            node.translation.set(0,0,0);
            node.scale.set(1,1,1);
            node.rotation.idt();
            modelInstance.calculateTransforms();
        }
```

Then we need to update the lines in the Populate class to set this extra parameter. It is set to false for the scenery objects but to true
for the dynamic objects:

```java
        public static void populate(World world) {
            world.clear();
    
            world.spawnObject(true, "brickcube", CollisionShapeType.BOX, false, Vector3.Zero, 1);
            world.spawnObject(true, "groundbox", CollisionShapeType.BOX, false, Vector3.Zero, 1f);
            world.spawnObject(true, "brickcube.001", CollisionShapeType.BOX,false, Vector3.Zero, 1f);
            world.spawnObject(true, "brickcube.002", CollisionShapeType.BOX,false, Vector3.Zero, 1f);
            world.spawnObject(true, "brickcube.003", CollisionShapeType.BOX,false, Vector3.Zero, 1f);
            world.spawnObject(true, "wall", CollisionShapeType.BOX,false, Vector3.Zero, 1f);
            world.spawnObject(false, "ball", CollisionShapeType.SPHERE, true, new Vector3(0,4,-2), Settings.ballMass);
            world.spawnObject(false, "ball", CollisionShapeType.SPHERE, true, new Vector3(-1,5,-2), Settings.ballMass);
            world.spawnObject(false, "ball", CollisionShapeType.SPHERE, true, new Vector3(-2,6,-2), Settings.ballMass);
    
            world.player = world.spawnObject(false, "ducky",CollisionShapeType.CAPSULE, true, new Vector3(0,1,0), Settings.playerMass);
        }
```

And we add new variables to the Settings class:

```java
    static public float ballMass = 0.2f;
    static public float playerMass = 1.0f;
```

## Spawning bullets

First, we'll add some methods to GameObject to get an object's position and forward direction. Add the following to GameObject: 
```java
        public final Vector3 direction;
    
        public GameObject(Scene scene, PhysicsBody body) {
            //...
            direction = new Vector3();
        }
        public Vector3 getPosition() {
            return body.getPosition();
        }
    
        public Vector3 getDirection() {
            direction.set(Vector3.Z);
            direction.mul(body.getBodyOrientation());
            return direction;
        }
```

The game object's direction is obtained by starting from a unit vector along the Z-axis and rotating (multiplying) that vector with the
body's orientation.  (The Z-axis is the original forward axis of our model because we happened to model it this way, if your assets face another direction modify this as needed). This gives us the forward direction of the game object if we take into account the current orientation of the related rigid body.


Also, add the following method to PhysicsBody to apply a force to a dynamic body. 
```java
        public void applyForce( Vector3 force ){
            DBody rigidBody = geom.getBody();
            rigidBody.addForce(force.x, force.y, force.z);  
        }
```
To let the player character fire projectiles, we create a new method in the World class called shoot().  This calculates a spawn position, which is slightly in front of 
the player character.  It spawns a bullet object and applies a force to it to make it move forward.
```java
        private final Vector3 dir = new Vector3();
        private final Vector3 spawnPos = new Vector3();
        private final Vector3 shootDirection = new Vector3();
    
        public void shoot() {
            dir.set( player.getDirection() );
            spawnPos.set(dir);
            spawnPos.add(player.getPosition()); // spawn from 1 unit in front of the player
            GameObject ball = spawnObject(false, "ball", CollisionShapeType.SPHERE, true, spawnPos, Settings.ballMass );
            shootDirection.set(dir);        // shoot forward
            shootDirection.y += 0.5f;       // and slightly up
            shootDirection.scl(Settings.ballForce);   // scale for speed
            ball.body.applyForce(shootDirection);
        }
```

We add the following to the Settings class to determine how hard we throw the ball:
```java
        static public float ballForce = 300f;
```

We call world.shoot() from GameScreen whenever the F key is pressed.
```java
        @Override
        public void render(float delta) {
            //...
            if (Gdx.input.isKeyJustPressed(Input.Keys.F))
                world.shoot();
            //...
        }
```
This will spawn a ball object just in front of the player character and give it a forward, slightly upward, impulse so that it moves up in an arc.

To avoid too much jitter we will set the physics update rate in `PhysicsWorld` from 40Hz to 60 Hz which is also a common monitor refresh frequency.
If your monitor runs at 120 Hz, it will execute one physics time step every second frame. 

```java
   public class PhysicsWorld implements Disposable {
    static final float TIME_STEP = 1f/60f;  // fixed physics time step
    //...
```
Note that changing this value affects the simulation, you may need to change force values or body masses to keep the same behaviour.


Then add the following code to the `render()` method to toggle debug mode with the F1 key:
```java
        if (Gdx.input.isKeyJustPressed(Input.Keys.F1))
            debugRender = !debugRender;
```
And put the gridView and physicsView render calls inside a condition to only be called if we are in debug mode:
```
        if(debugRender) {
            gridView.render(gameView.getCamera());
            physicsView.render(gameView.getCamera());
        }
```
Now we can enable and disable the debug view with the F1 key.

## Show rigid body state

An important performance aspect of the physics library is not to spend too much time on objects which are not moving. If a moving object comes to a standstill, after a while it goes to a sleeping state.
This means it is skipped for further calculations.  We can visualize this by using different colours for the debug shapes.

Add the following constants to PhysicsBody:
```java
        // colours to use for active vs. sleeping geoms
        static private final Color COLOR_ACTIVE = Color.GREEN;
        static private final Color COLOR_SLEEPING = Color.TEAL;
        static private final Color COLOR_STATIC = Color.GRAY;
```
Then add the following lines to the `render()` method of PhysicsBody:
```java
        // use different colour for static/sleeping/active objects and for active ones
        DGeom geom = body.geom;
        Color color = COLOR_STATIC;
        if (geom.getBody() != null) {
            if (geom.getBody().isEnabled())
                color = COLOR_ACTIVE;
            else
                color = COLOR_SLEEPING;
        }
        body.debugInstance.materials.first().set(ColorAttribute.createDiffuse(color));   // set material colour
```
These lines will modify the material colour of the debug ModelInstance depending on the body state:  
- static objects (only a geom, no rigid body) are shown in gray
- active objects are shown in green
- sleeping object are shown in teal

Have a try in debug view (press F1). You will see that some green objects will turn teal after a few seconds. If you look at PhysicsWorld `reset()` you will see a number of 
settings to tune automatic disabling of objects if they don't move or rotate for a short while. 
```java
            // set auto disable parameters to make inactive objects go to sleep
            world.setAutoDisableFlag(true);
            world.setAutoDisableLinearThreshold(0.1);
            world.setAutoDisableAngularThreshold(0.001);
            world.setAutoDisableTime(2);
```
These type of parameters can be tuned for efficiency, especially if you have many objects in your game. For example, the balls that are spawned in the previous code seem to remain active forever even though they are not moving.
Replace the angular threshold with the following, to see the balls falling asleep a short while after they come to rest.
```java
            world.setAutoDisableAngularThreshold(0.1);
```
## PlayerController

In tutorial step 2 we developed a first person camera controller where we move the camera using the wasd keys.
Rather than moving the camera, we're now going to make a controller to move a character in third person view.

The class PlayerController implements a dynamic character controller. JamesTKhan has a YouTube video here [https://www.youtube.com/watch?v=O0Deshj2-KU&ab_channel=JamesTKhan](https://www.youtube.com/watch?v=O0Deshj2-KU&ab_channel=JamesTKhan) describing this.
Instead of changing the position of the player object directly, we will apply an impulse (a momentary force) on the corresponding rigid body.

We use strong damping to make the character stop moving as soon as we let go of the keys. If we're simulating a car or a spaceship,
we could use less damping to give the character more inertia.

For rotation of the player object we take a slightly different approach. We could just let the player controller apply a torque (a "rotation force") on the player's physics body. 
However, it is a well-known issue in the ODE physics library, in fact it is mentioned in the user manual, that rotating a capsule around the vertical
axis will eventually cause the capsule to tilt sideways due to small errors accumulating. And if you try this with debug view on, you will see the player starting to tilt over after moving it around for a while.

There are a number of solutions for this. Ours is relatively simple: from a collision detection point of view there is no need to rotate a capsule around its length axis since it is rotationally symmetric anyway.
To avoid the player's capsule geom to tilt, we will lock it from any rotation by setting the maximum angular speed to zero. We'll add a new method to the PhysicsBody class that sets the physics properties of the player body.  This is also the place to tweak the damping which will affect how quickly the player character will come to a stop in absence of keyboard input. And we also ensure the player object never goes to sleep mode.  
```java
    public void setPlayerCharacteristics() {
        DBody rigidBody = geom.getBody();
        rigidBody.setDamping(Settings.playerLinearDamping, Settings.playerAngularDamping);
        rigidBody.setAutoDisableFlag(false);       // never allow player to get disabled
        rigidBody.setMaxAngularSpeed(0);        // keep capsule upright by not allowing rotations
    }
```
Call this method once on the player body on loading the level, e.g. in the `Populator` class or add a new method `setPlayer` in the `World` class as a convenient place to do this.


The PlayerController class is then quite similar to the CamController class we developed earlier and which we can now use instead (in `GameScreen` 
set the input processor to the player controller instead of the camera controller, i.e. `Gdx.input.setInputProcessor(world.getPlayerController());`).

```java
    public class PlayerController extends InputAdapter  {
        public int forwardKey = Input.Keys.W;
        public int backwardKey = Input.Keys.S;
        public int strafeLeftKey = Input.Keys.A;
        public int strafeRightKey = Input.Keys.D;
        public int turnLeftKey = Input.Keys.Q;
        public int turnRightKey = Input.Keys.E;
        public int jumpKey = Input.Keys.SPACE;
        public int runShiftKey = Input.Keys.SHIFT_LEFT;
    
        private final IntIntMap keys = new IntIntMap();
        private final Vector3 linearForce;
        private final Vector3 forwardDirection;   // direction player is facing, move direction, in XZ plane
        private final Vector3 viewingDirection;   // look direction, is forwardDirection plus Y component
        private float mouseDeltaX;
        private float mouseDeltaY;
        private final Vector3 tmp = new Vector3();
        private final Vector3 tmp2 = new Vector3();
        private final Vector3 tmp3 = new Vector3();
    
        public PlayerController()  {
            linearForce = new Vector3();
            forwardDirection = new Vector3();
            viewingDirection = new Vector3();
            reset();
        }
    
        public void reset() {
            forwardDirection.set(0,0,1);
            viewingDirection.set(forwardDirection);
        }
    
        public Vector3 getViewingDirection() {
            return viewingDirection;
        }
    
        public Vector3 getForwardDirection() {
            return forwardDirection;
        }
    
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
    
        @Override
        public boolean mouseMoved(int screenX, int screenY) {
            // ignore big delta jump on start up or resize
            if(Math.abs(Gdx.input.getDeltaX()) >=100 && Math.abs(Gdx.input.getDeltaY()) >= 100)
                return true;
            mouseDeltaX = -Gdx.input.getDeltaX() * Settings.degreesPerPixel;
            mouseDeltaY = -Gdx.input.getDeltaY() * Settings.degreesPerPixel;
            return true;
        }
    
        private void rotateView( float deltaX, float deltaY ) {
            viewingDirection.rotate(Vector3.Y, deltaX);
    
            if (!Settings.freeLook) {    // keep camera movement in the horizontal plane
                viewingDirection.y = 0;
                return;
            }
            if (Settings.invertLook)
                deltaY = -deltaY;
    
            // avoid gimbal lock when looking straight up or down
            Vector3 oldPitchAxis = tmp.set(viewingDirection).crs(Vector3.Y).nor();
            Vector3 newDirection = tmp2.set(viewingDirection).rotate(tmp, deltaY);
            Vector3 newPitchAxis = tmp3.set(tmp2).crs(Vector3.Y);
            if (!newPitchAxis.hasOppositeDirection(oldPitchAxis))
                viewingDirection.set(newDirection);
        }
    
        public void moveForward( float distance ){
            linearForce.set(forwardDirection).scl(distance);
        }
    
        private void strafe( float distance ){
            // add strafe vector to linear velocity
            // strafe vector is at a right angle to forward direction
            // to move left or right
            tmp.set(forwardDirection).crs(Vector3.Y);   // cross product
            tmp.scl(distance);
            linearForce.add(tmp);
        }
    
        public void update (GameObject player, float deltaTime ) {
            // derive forward direction vector from viewing direction
            forwardDirection.set(viewingDirection);
            forwardDirection.y = 0;
            forwardDirection.nor();
    
            // reset velocities
            linearForce.set(0,0,0);
    
            float moveSpeed = Settings.walkSpeed;
            if(keys.containsKey(runShiftKey))
                moveSpeed *= Settings.runFactor;
    
            // mouse to move view direction
            rotateView(mouseDeltaX*deltaTime*Settings.turnSpeed, mouseDeltaY*deltaTime*Settings.turnSpeed );
            mouseDeltaX = 0;
            mouseDeltaY = 0;
    
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
    
            if (keys.containsKey(jumpKey) )
                linearForce.y =  deltaTime * Settings.jumpForce;
    
            linearForce.scl(500);
            player.body.applyForce(linearForce);
            // note: as the player body is a capsule it is not necessary to rotate it
            // (and in fact it causes problems due to errors building up)
            // so we don't rotate the rigid body, but we rotate the modelInstance in World.syncToPhysics()
        }
    }
```

We add the following values to the `Settings` class:

```java
    static public float jumpForce = 10f;
    static public float degreesPerPixel = 0.1f; // mouse sensitivity
```

and we modify the gravity value:

```java
    static public float gravity = -30f;
```


We define a PlayerController object as field of the World class and update it whenever the `World.update()` method is called.

If you play around with this, you will notice that the player doesn't turn.  You may recall that the physics is based on a capsule and we decided not to rotate
the corresponding rigid body.
A downside of not rotating the physics body is that the player character is visually always facing the same direction. To solve this we make an exception in method `update` in the World class.
For all the other game objects that are controlled by a rigid body, the ModelInstance transform is derived from the physics body.  For the player object, the ModelInstance rotation is taken from the PlayerController.

```java
    private void update() {
        playerController.update(player, deltaTime);
        physicsWorld.update(deltaTime);
        for(GameObject go : gameObjects){
            if( go.body.geom.getBody() != null) {
                go.scene.modelInstance.transform.set(go.body.getPosition(), go.body.getOrientation());
            }
        }
        // the player model is an exception, use information from the player controller, since the rigid body is not rotated.
        player.scene.modelInstance.transform.setToRotation(Vector3.Z, playerController.getForwardDirection());
        player.scene.modelInstance.transform.setTranslation(player.body.getPosition());
    }
```
Recall you can rotate the player with the Q and E keys, whereas A and D are for strafing.


It is also important to update the `shoot` method in `World` to make use of the viewingDirection to shoot the balls in the direction that the player is looking.

```java
    public void shoot() {
        dir.set(playerController.getViewingDirection());
        spawnPos.set(dir);
        spawnPos.add(player.getPosition()); // spawn from 1 unit in front of the player
        GameObject ball = spawnObject(false, "ball", CollisionShapeType.SPHERE, true, spawnPos, Settings.ballMass);
        shootDirection.set(dir);        // shoot forward
        shootDirection.y += 0.5f;       // and slightly up
        shootDirection.scl(Settings.ballForce);   // scale for speed
        ball.body.applyForce(shootDirection);
    }
```


```java
    static public float ballMass = 0.2f;
    static public float ballForce = 300f;
```


## New Camera Controller

If we play around with the game so far, we'll quickly get annoyed by the fixed camera. The keyboard and mouse input are not used to control the player character in third person, but the view is now static.

Next we will change the camera controller to a more simplified version one.  We don't control the camera anymore with keyboard and mouse, except that we can zoom with the scroll wheel of the mouse.
Since we originally started our project with a first person view, and somehow we arrived at a third person view, we will allow both options with our new camera controller: In first person view the camera will be 
positioned at the player position and pointed along the player's view direction.  In third person view, the camera will be placed some distance behind and above the player and follow the player around Lara Croft style.

```java
    public class CamController extends InputAdapter {
    
        private final Camera camera;
        private boolean thirdPersonMode = true;
        private final Vector3 offset = new Vector3();
        private float distance = 5f;
    
        public CameraController(Camera camera ) {
            this.camera = camera;
            offset.set(0, 2, -3);
        }
    
        public void setThirdPersonMode(boolean mode){
            thirdPersonMode = mode;
        }
    
        public boolean getThirdPersonMode() { return thirdPersonMode; }
    
        public void update ( Vector3 playerPosition, Vector3 viewDirection ) {
    
            camera.position.set(playerPosition);
    
            if(thirdPersonMode) {
                // offset of camera from player position
                offset.set(viewDirection).scl(-1);      // invert view direction
                offset.y = Math.max(0, offset.y);             // but don't go below player
                offset.nor().scl(distance);                   // scale for camera distance
                camera.position.add(offset);
    
                camera.lookAt(playerPosition);
                camera.up.set(Vector3.Y);
            }
            else {
                camera.direction.set(viewDirection);
            }
            camera.update(true);
        }
    
        @Override
        public boolean scrolled (float amountX, float amountY) {
            return zoom(amountY );
        }
    
        private boolean zoom (float amount) {
            if(amount < 0 && distance < 5f)
                return false;
            if(amount > 0 && distance > 30f)
                return false;
            distance += amount;
            return true;
        }
    }
```

Since the camera controller only controls the view of the game, not any objects in the world space, we can define the CamController object as a field of the GameView class.  We add a getter for this object so that we can 
define both the camera controller and the player controller as input processors in `GameScreen.show()`:

```java
        InputMultiplexer im = new InputMultiplexer();
        Gdx.input.setInputProcessor(im);
        im.addProcessor(gameView.getCameraController());
        im.addProcessor(world.getPlayerController());
```

And we call the `CamController` update method from the `GameView` render method:

```java
        camController.update(world.getPlayer().getPosition(), world.getPlayerController().getViewingDirection());
```

Then in GameView.render() we allow the view to toggle from first to third person using the F2 key.  This method now looks as follows:

```java
    public void render(float delta ) {
        if (Gdx.input.isKeyJustPressed(Input.Keys.F2)) {
            camController.setThirdPersonMode(!camController.getThirdPersonMode());
        }

        camController.update(world.getPlayer().getPosition(), world.getPlayerController().getViewingDirection());
        cam.update();
        if(world.isDirty())
            refresh();
        sceneManager.update(delta);
        
        ScreenUtils.clear(Color.PURPLE, true);  // note clear color will be hidden by skybox anyway
        sceneManager.render();
    }
```


One thing we'll quickly notice is when we switch to first person view, we get a view from inside the duck.

![inside duck](/assets/images/inside-duck.png)

Obviously, we should not render the player character in first person view. Let's add a boolean field called `visible` to GameObject to allow us to hide game objects
and update `GameView.refresh()` to include only scenes from visible game objects. We will make the player visible only in third person mode:

```java    
    public void refresh() {
        world.player.visible = camController.getThirdPersonMode(); //<---------- new
        sceneManager.getRenderableProviders().clear();        // remove all scenes

        // add scene for each game object
        int num = world.getNumGameObjects();
        for(int i = 0; i < num; i++){
            Scene scene = world.getGameObject(i).scene;
            if(world.getGameObject(i).visible)                          //<---------- new
                sceneManager.addScene(scene, false);
        }
    }
```

In PhysicsView we make a similar change, so that we don't draw the player capsule from inside.

Then we modify the code we just added to toggle between first person and third person view to call `refresh` when the viewing mode is toggled:

```java

    public void render(float delta ) {
        if (Gdx.input.isKeyJustPressed(Input.Keys.F2)) {
            camController.setThirdPersonMode(!camController.getThirdPersonMode());
            refresh();
        }
        //...
    }


```      
        
This concludes step 6.
