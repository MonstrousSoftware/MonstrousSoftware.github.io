# 3D Tutorial - Step 9 - Pick-ups
by Monstrous Software


# Step 9 - Pick-ups

Let's add some pick-ups to our game. Items that are scattered around, like ammo boxes or health packs, and that the player can pick up by walking over them.
This means we need to react to collisions between the player and specific objects.

It also means we need to start distinguishing different types of objects to determine their behaviour.

Let's define a class GameObjectType. We'll add a typeName as that will be handy for debugging and some attributes as boolean values: is it a static object, is it the player, can it be picked up?

```java
        public class GameObjectType {
            public String typeName;
            public boolean isStatic;
            public boolean isPlayer;
            public boolean canPickup;
        
            public GameObjectType(String typeName, boolean isStatic, boolean isPlayer, boolean canPickup) {
                this.typeName = typeName;
                this.isStatic = isStatic;
                this.isPlayer = isPlayer;
                this.canPickup = canPickup;
            }
        }
```

Let's add some type constants to define some generic game object types. We'll use the static type for objects that don't move or interact (apart from collisions) such as the ground and walls,
we'll use the player type for the object that corresponds to the player character. We'll define a pickup type for coins and for health packs that can be picked up and a dynamic type for objects
that may move around, like the balls.

```java
        public class GameObjectType {

            public final static GameObjectType TYPE_STATIC = new GameObjectType("static", true, false, false);
            public final static GameObjectType TYPE_PLAYER = new GameObjectType("player", false, true, false);
            public final static GameObjectType TYPE_PICKUP_COIN = new GameObjectType("coin", false, false, true);
            public final static GameObjectType TYPE_PICKUP_HEALTH = new GameObjectType("health", false, false, true);
            public final static GameObjectType TYPE_DYNAMIC = new GameObjectType("dynamic", false, false, false);

            ...
        }
```

We extend the GameObject class so that every game object refers to a game object type. This will be a new parameter in the constructor. 

```java
        public class GameObject {

            public final GameObjectType type;           // new
            public final Scene scene;
            public final PhysicsBody body;
            public final Vector3 direction;
            public boolean visible;
        
            public GameObject(GameObjectType type, Scene scene, PhysicsBody body) {
                this.type = type;           
                this.scene = scene;
                this.body = body;
                body.geom.setData(this);            // the geom has user data to link back to GameObject for collision handling
                visible = true;
                direction = new Vector3();
            }
            //...
        }
```

And we extend the spawnObject method of the World class to include the object type as input parameter so that it can be passed to the GameObject constructor.  We can remove the boolean `isStatic` parameter
because that is now determined by the game object type.

```java
        public GameObject spawnObject(GameObjectType type, String name, String proxyName, CollisionShapeType shapeType, boolean resetPosition, Vector3 position, float mass){
            Scene scene = loadNode( name, resetPosition, position );
            ModelInstance collisionInstance = scene.modelInstance;
            if(proxyName != null) {
                Scene proxyScene = loadNode( proxyName, resetPosition, position );
                collisionInstance = proxyScene.modelInstance;
            }
            PhysicsBody body = factory.createBody(collisionInstance, shapeType, mass, type.isStatic);
            GameObject go = new GameObject(type, scene, body);
            gameObjects.add(go);
            isDirty = true;         // list of game objects has changed
            return go;
        }
```

Let's add some coins and health packs to the world that can be picked up. These will be of type pick-up.  The ground, walls and fixed blocks are typed as static objects.
The balls are dynamic objects and the player has the player type.

```java
        public static void populate(World world) {
            world.clear();
    
            world.spawnObject(GameObjectType.TYPE_STATIC, "brickcube", null, CollisionShapeType.BOX, false, Vector3.Zero, 1);
            ...    
            world.spawnObject(GameObjectType.TYPE_DYNAMIC, "ball", null, CollisionShapeType.SPHERE, true, new Vector3(0,4,-2), Settings.ballMass);
            world.spawnObject(GameObjectType.TYPE_DYNAMIC, "ball", null, CollisionShapeType.SPHERE, true, new Vector3(-1,5,-2), Settings.ballMass);
            world.spawnObject(GameObjectType.TYPE_DYNAMIC, "ball", null, CollisionShapeType.SPHERE, true, new Vector3(-2,6,-2), Settings.ballMass);
    
            world.spawnObject(GameObjectType.TYPE_PICKUP_COIN, "coin",  null, CollisionShapeType.BOX, true, new Vector3(-5, 1, 0), 1);
            world.spawnObject(GameObjectType.TYPE_PICKUP_COIN, "coin",  null,  CollisionShapeType.BOX, true, new Vector3(5, 1, 15), 1);
            world.spawnObject(GameObjectType.TYPE_PICKUP_COIN, "coin",  null, CollisionShapeType.BOX, true, new Vector3(-12, 1, 13), 1);
            world.spawnObject(GameObjectType.TYPE_PICKUP_HEALTH, "healthpack",null, CollisionShapeType.BOX, true, new Vector3(26, 0.1f, -26), 1);
            world.spawnObject(GameObjectType.TYPE_PICKUP_HEALTH, "healthpack",  null, CollisionShapeType.BOX, true, new Vector3(-26, 0.1f, 26), 1);

    
            GameObject go = world.spawnObject(GameObjectType.TYPE_PLAYER, "ducky",null, CollisionShapeType.CAPSULE, true, new Vector3(0,4,0), Settings.playerMass);
            world.setPlayer(go);
        }
```

In the PhysicsWorld class we add a field for the game world (not to be confused with DWorld) and set it from the constructor.
In the collision callback method we can now add a call to a new method `World.onCollision()` when two game objects collide.
For this we use the data field of the geoms (which we set in the GameObject constructor specifically for this purpose) and cast them back to GameObject.


```java
        public class PhysicsWorld implements Disposable {
            ...
            private final World gameWorld;
        
            public PhysicsWorld(World gameWorld) {
                this.gameWorld = gameWorld;
                OdeHelper.initODE2(0);
                Gdx.app.log("ODE version", OdeHelper.getVersion());
                Gdx.app.log("ODE config", OdeHelper.getConfiguration());
                contactGroup = OdeHelper.createJointGroup();
                reset();
            }
            // ...
            private final DGeom.DNearCallback nearCallback = new DGeom.DNearCallback() {
            
                    @Override
                    public void call(Object data, DGeom o1, DGeom o2) {
                        DBody b1 = o1.getBody();
                        DBody b2 = o2.getBody();
                        if (b1 != null && b2 != null && OdeHelper.areConnected(b1, b2))
                            return;
            
                        final int N = 8;
                        DContactBuffer contacts = new DContactBuffer(N);
            
                        int n = OdeHelper.collide(o1, o2, N, contacts.getGeomBuffer());
                        if (n > 0) {
                            gameWorld.onCollision((GameObject)o1.getData(), (GameObject)o2.getData());        // <-------    new
            
                            for (int i = 0; i < n; i++) {
                                DContact contact = contacts.get(i);
                                //...
                            }
                        }
                    }
                };
        }
```


The callback function checks the types of the collided object to determine if anything special needs to be done.  
If one object is the player and the other object is a pickup, then it calls the pickup() method, which in this case will just delete the pickup object.
Note how we need to test the two input object in either order because we have no guarantee on which order the pair will be provided by the collision logic.

```java
        public class World implements Disposable {
            //...
            
            public void update( float deltaTime ) {
                playerController.update(player, deltaTime);
                physicsWorld.update(this);                                      // new parameter to pass this World object
                syncToPhysics();
            }
    
           public void onCollision(GameObject go1, GameObject go2){             // called on collision
                // try either order
                handleCollision(go1, go2);
                handleCollision(go2, go1);
            }
        
            private void handleCollision(GameObject go1, GameObject go2){
                if(go1.type.isPlayer && go2.type.canPickup){
                     pickup(go1, go2);
                }
            }
    
            private void pickup(GameObject character, GameObject pickup){
                removeObject(pickup);
            }
        }
```

This concludes step 9 where we've seen how to distinguish different types of objects, how to catch collisions between specific object types
and how to implement a pickup mechanism.  When we launch the program we can run around and pick up the three coins and two health packs.